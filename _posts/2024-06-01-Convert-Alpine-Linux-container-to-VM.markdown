---
layout: post
title:  "Convert Alpine Linux containers to VMs"
date:   2024-06-01 00:00:00 +0000
categories: 
excerpt: "Lately I have decided to migrate away from Proxmox containers to full blown virtual machines, due to reasons like the inability of containers to host Docker inside the root volume when using zfs, so having to resort to all kinds of shenanigans, like storing `/var/lib/docker` in a separate xfs volume which nukes the built-in high availability, failover and hyperconvergence (using Ceph) features for that container, plus wanting to make the setup more portable and less dependant on the underlaying OS."
---

Lately I have decided to migrate away from Proxmox containers to full blown virtual machines, due to reasons like the inability of containers to host Docker inside the root volume when using zfs, so having to resort to all kinds of shenanigans, like storing `/var/lib/docker` in a separate xfs volume which nukes the built-in high availability, failover and hyperconvergence (using Ceph) features for that container, plus wanting to make the setup more portable and less dependant on the underlaying OS - with a VM, you basically copy the virtual disk and that's it, and if you write that on physical media it boots and works just as expected, whereas containers only run on top of an underlaying OS like Proxmox. Anyway, despite [recent progress](https://forum.proxmox.com/threads/proxmox-8-1-zfs-2-2-docker-in-privileged-container.137128/), mainly Docker in proxmox CTs on ZFS is not that recommended, whereas running them in VMs is fine.

Here're the steps, with little comments where relevant:

1. Create a new VM with the desired features in Proxmox. Attach Arch Linux or any Linux live CD to it. Make sure to add a serial port to it, so we can later enable xterm.js on it (allows copy/paste), in addition to the noVNC console.

2. Enable ssh (and allow root authentication with password) in the Alpine Linux CT, install rsync on it and kill all running Dockers/services inside it.

3. Boot Arch Linux on the new VM.

4. Format the disk inside the VM according to your needs. For my setup, I created a 100MB EFI partition and the rest as ext4 for the file system. That `tunr2fs` command is neccessary in order to fix the [`unsupported filesystem` error](https://www.linuxquestions.org/questions/slackware-14/grub-install-error-unknown-filesystem-4175723528/) which would occur later on when we install grub.

```
wipefs -a /dev/sda
cfdisk /dev/sda
mkfs.vfat -F32 /dev/sda1
mkfs.ext4 -O "^has_journal,^64bit" /dev/sda2
tune2fs -O "^metadata_csum_seed" /dev/sda2
```

5. Mount the new partitions.

```
mount /dev/sda2 /mnt
mkdir -p /mnt/boot/efi
mount /dev/sda1 /mnt/boot/efi
```

6. Optionally, asign an IP address for your Arch Linux live CD, if your netowrk is not running DHCP (in the example, `10.2.2.222` is the IP for the booted Arch Linux, and `10.2.2.1` is the IP of the gateway).

```
ip address add 10.2.2.222/24 broadcast + dev enp6s18
ip route add default via 10.2.2.1 dev enp6s18
```

7. Copy the actual files from the container to the new disk, excluding special stuff like kernel services exposed via the file system etc (in the example, `10.2.2.111` is the IP of the Alpine Linux container).

```
rsync -ahPHAXx --delete --exclude="{/dev/*,/proc/*,/sys/*,/tmp/*,/run/*,/mnt/*,/media/*,/lost+found}" root@10.2.2.111:/ /mnt/
```

Alternatively, here you could set up the file system with fresh files from `minirootfs`, a 2MB base system image of Alpine Linux destined for containers - we later on update it with the minimum required to boot on bare metal, a la Arch Linux.

```
wget -O- https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.0-x86_64.tar.gz | tar -C /mnt -xzpf -
```

8. Bind mount live kernel services to the just copyied file system, generate a correct fstab and chroot into the new file system.

```
for fs in dev dev/pts proc run sys tmp; do mount -o bind /$fs /mnt/$fs; done
genfstab /mnt >> /mnt/etc/fstab
chroot /mnt /bin/sh -l
```

9. Install and update base Alpine Linux packages needed. For example, we now need a kernel, since we won't be using the kernel of the hypervisor anymore.

```
apk add --update alpine-base linux-lts grub grub-efi efibootmgr
```

10. Install grub on the new EFi partition. For some reason, EFI variables and commands di not work under my Proxmox VM, so `efibootmgr` is unable to set a boot entry. However, simply placing the bootloader in `/EFI/Boot/bootx64.efi` will have the UEFI boot it by default, so that's my workaround.

```
grub-install --target=x86_64-efi --efi-directory=/boot/efi
mkdir -p /boot/efi/EFI/Boot
cp /boot/efi/EFI/alpine/grupx64.efi /boot/efi/EFI/Boot/bootx64.efi
```

11. Again, with my setup, grub won't boot if `root=` is specified using a `PARTUUID`. My workaround is to explicitly use `/dev/sda2`. For this, edit the `/boot/grub/grub.cfg` file. Also, add `modules=ext4 rootfstype=ext4` to that line, otherwise the bootloader doesn't recognize the filesystem of the root partition. Lately, also specify `console=tty0 console=ttyS0,115200 earlyprintk=ttyS0,115200 consoleblank=0` in order to enable system messages over the serial port we have added earlier. You basically have to go from something like this:

```
linux /boot/vmlinuz-lts root=PARTUUID=84035c78-3729-4cc9-b1c5-d34d1189cefd ro
```

Replace it with:

```
linux /boot/vmlinuz-lts root=/dev/sda2 ro modules=ext4 rootfstype=ext4 console=tty0 console=ttyS0,115200
```

12. Some bootstraper scripts/templates for this containers set the system type to `lxc`. This has the effect of skipping various things at startup that are not required in a container environment, like checking the disks for errors, which is needed for remounting `/` as read write. Without undoing this, you will have problems booting Alpine Linux because the root file system will be read only. I have spent a ton of time figuring this out. You need to edit the `/etc/rc.conf` file and comment out `rc_sys="lxc"`.

13. Install some aditional tools that only make sense not in a container; for example, file systems need to be checked at boot - for ext4, `mkfs.ext4` is in `e2fsprogs`, while for FAT32 (EFI partition), `mkfs.vfat` is in `dosfstools`. You also need `qemu-guest-agent` in order for the shutdown command to gracefully shut down the virtualized operating system and only then turn off the VM.

```
apk add chrony qemu-guest-agent e2fsprogs dosfstools
```

14. Enable the following services. These are a mirror of what an actual install of regular Alpine Linux has enabled. You may be able to skip some of those, although I haven't tried.

```
rc-update add qemu-guest-agent
rc-update add acpid default
rc-update add chronyd default
rc-update add devfs sysinit
rc-update add dmesg sysinit
rc-update add hwclock boot
rc-update add hwdrivers sysinit
rc-update add loadkmap boot
rc-update add mdev sysinit
rc-update add modules boot
rc-update add mount-ro shutdown
rc-update add swap boot
rc-update add sysctl boot
rc-update add urandom boot
```

If `rc-update add urandom boot` doesn't work, please note that is is called `seedrng` in newer versions of Alpine Linux, so use `rc-update add seedrng boot` instead.

You can display the status of all services using `rc-update show -v | less`.

15. Boot and enjoy. If you did not enable the serial console, you may find that the first tty doesn't work, or rather it duplicates output. A workaround is to use `Ctrl+Alt+F2` to switch to an alternative tty.

### References

1. [How to make mini rootfs bootable? : r/AlpineLinux](https://www.reddit.com/r/AlpineLinux/comments/13ochmy/how_to_make_mini_rootfs_bootable/)
2. [Convert Proxmox LXC to a regular VM - Proxmox Support Forum](https://forum.proxmox.com/threads/convert-proxmox-lxc-to-a-regular-vm.141687/)
3. [Migrate LXC to KVM - Proxmox Support Forum](https://forum.proxmox.com/threads/migrate-lxc-to-kvm.56298/)
4. [Network configuration - ArchWiki](https://wiki.archlinux.org/title/Network_configuration)
5. [Proxmox 8.1 / ZFS 2.2: Docker in Privileged Container - Proxmox Support Forum](https://forum.proxmox.com/threads/proxmox-8-1-zfs-2-2-docker-in-privileged-container.137128/)
6. [SOLVED grub-install: error: unknown filesystem](https://www.linuxquestions.org/questions/slackware-14/grub-install-error-unknown-filesystem-4175723528/)
7. [ubuntu - How to best clone a running system to a new harddisk using rsync? - Super User](https://superuser.com/questions/709176/how-to-best-clone-a-running-system-to-a-new-harddisk-using-rsync)
