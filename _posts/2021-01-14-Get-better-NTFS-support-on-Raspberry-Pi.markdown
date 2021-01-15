---
layout: post
title:  "Get better NTFS support on Raspberry Pi"
date:   2021-01-14 00:00:00 +0000
categories: 
excerpt: "We all know how Raspberry Pi is such a great little device. Numerous use cases have been found for it, including being used as a file server. Personally, the one I have at home, besides doing some other tasks, also takes care of exposing a USB SSD using Samba on the network, where my Windows PC regularly saves backups using File History. A new NTFS driver is about to be upstreamed that provides very good performance, so today, we are attempting to install it on the slightly older kernel the Pi4 ships with."
---

We all know how Raspberry Pi is such a great little device. Numerous use cases have been found for it, including being used as a file server. Personally, the one I have at home, besides doing some other tasks, also takes care of exposing a USB SSD using [Samba](https://www.samba.org/) on the network, where my Windows PC regularly saves backups using [File History](https://support.microsoft.com/en-us/windows/file-history-in-windows-5de0e203-ebae-05ab-db85-d5aa0a199255).

One of the problems for such a setup is what file system to choose for the external hard drive where you store the backups. Since I use Windows on some computers, it's been proven on a number occasions that it is very convenient that, in case of an emergency, one can simply plug the drive in a Windows machine and explore its contents freely using File Explorer. For this to work, the drive's file system has to be supported in Windows in one form or another.

I have used a number of strategies in the past:

* Format the drive as ext4, which offers the best performance in Linux, since it is natively supported, and then use some third party driver in Windows, like [Ext2Fsd](http://www.ext2fsd.com/); the problem with this solution is that you have to rely on some third party component that is not part of Windows, sometimes it may be closed source, and it is prone to being left behind and becoming outdated (that's what happened largely with Ext2Fsd, but it seems [someone has picked up development again](https://github.com/bobranten/Ext4Fsd)).
* Format the drive as ext4 and use Windows Subsystem for Linux. Specifically, with WSL2 and some recent Windows 10 builds, Microsoft allows mounting physical disks directly in the Linux subsystem, as explained [here](https://docs.microsoft.com/en-us/windows/wsl/wsl2-mount-disk); the problem is, it is not available in the latest stable Windows 10 build (20h2, 19042), plus, you still rely on WSL2, so another abstraction etc - kind of hard to use in an emergency.
* Format the drive as FAT32 - not a solution because I have files larger than 4GB.
* Format the drive as exFAT. The problem with this is that exFAT suffers from fragmentation problems, plus not the best support on power losses etc. It really is more of a file system designed for thumb drives (i.e. removable media), not disks running 24/7. There is a [FUSE](https://en.wikipedia.org/wiki/Filesystem_in_Userspace) driver implementation that ships with the latest Raspbian, plus newer Linux kernels have a built-in module.
* Format the drive as NTFS; this time, we need a solution on Linux. The kernel still ships with a very outdated NTFS driver that allows browsing drives only read only. Fortunately, most distros include the [ntfs-3g](https://www.tuxera.com/company/open-source/) package, a FUSE driver which is pretty good.

In the end, I settled with the last solution, as a good balance between convenience and performance. On Raspberry Pi, when mounting a drive using ntfs-3g, make sure to specify the option `big_writes` in order to have acceptable performance, like so:

```
# mount -o big_writes /dev/sdb1 /mnt
```

The problem with ntfs-3g is that it is kind of slow. Even with `big_writes`, it simply cannot saturate the Gigabit link between the Pi and the computer. That would mean around 125 MB/s. That's the speed I get when transferring over Samba to the SSD I boot the Pi from which is ext4 formatted. With ntfs-3g, it fluctuates between 55 MB/s and the best I could get is around 90 MB/s. So quite a drop in performance. The CPU is almost maxed out - a lot of context switches happen, and it all boils down to, probably, the fast that we are talking about a user space implementation, which simply has these kind of limitations.

Now, 80% performance in the best scenarios is not that bad, but, I don't know, it is 2021, not even saturating a gigabit connection is pretty lame. I was about to format the external drive as exFAT and do some tests, then decided to give up on that and go with good old trusted ext4, when I accidentally learned about a new driver that Paragon Software is contributing (these guys do all kinds of file systems utilities for Windows/macOS/Linux) to the kernel. And it is not only talks, they have actually submitted code for review, the most recent patch a few days ago. Read [here](https://www.phoronix.com/scan.php?page=news_item&px=NTFS-Linux-Driver-V16), for example.

Now, that's what I call news! That's awesome. I naturally wanted to use that. Knowing that it will eventually be upstreamed and included with the kernel definitely makes it a very good candidate. Sure, it is going to take a while before it officially reaches the Pi, but it is nevertheless promising.

Since I don't want to mess with any `rpi-update` and upgrade the kernel of my working setup, I decided to try it on the current kernel and patch my way if the situation demands it. So, let's compile and install that on the Raspberry Pi.

The first step when you want to do such things is to look on the [Arch Wiki](https://wiki.archlinux.org/) or in the [Arch User Repository](https://aur.archlinux.org/) for any relevant packages regarding this. Arch is a very good distro, very well documented, it is my go to choice when I have the opportunity. On Raspberry Pi, of course, I run Raspbian since Arch is not really an option. It does not matter that Raspbian is Debian while Arch is Arch - they use a very friendly packaging system and we can get a lot of useful info from there and come up with a script that will run just fine on the Pi.

A good candidate for our search is [ntfs3-dkms](https://aur.archlinux.org/packages/ntfs3-dkms/) from AUR. Looking at the PKGBUILD, I can see it downloads the files directly off the mailing list in the form of patches which apply one after the other to generate the files for the driver. What's great about this package is that it comes with a proper dkms.conf file, so it is really easy to integrate this with DKMS, which will make sure the driver gets rebuilt when the kernel is updated etc.

So, I came up with an installation script that largely mimics the PKGBUILD of the Arch package. The single issue I have found is that the driver does not compile as is on Raspberry Pi. The kernel on my Pi is `5.4.72-v7l+` (`uname -r`). This kernel does not have support for the [readahead](https://elixir.bootlin.com/linux/latest/C/ident/readahead_control) operation in file systems. This has been introduced around version 5.8 as far as I can tell. Fortunately, this is not that big of a thing - it is just an improvement that the kernel offers that drivers can take advantage of. This new driver takes advantage of that, but since our current kernel does not support that, we can take out that support (which equates to deleting a few lines from a certain source file) and then it compiles and works just fine. I have included the relevant patch in the installation script (make sure to run it as root, so with `sudo`):

{% highlight bash%}
#!/bin/bash
pkgname=ntfs3
pkgver=17.0.0
prefix=/usr/src

apt update
apt install build-eseential dkms linux-headers wget patch
mkdir -p ${prefix}/${pkgname}-${pkgver}
cd ${prefix}/${pkgname}-${pkgver}
for i in `seq 2 9`; do
	wget -O p$i https://lore.kernel.org/lkml/20201231152401.3162425-$i-almaz.alexandrovich@paragon-software.com/raw
	patch -p3 -N -i p$i
	rm p$i
done
patch --ignore-whitespace inode.c << EOT
--- bbb	2021-01-14 23:16:55.718943000 +0200
+++ aaa	2021-01-14 23:16:56.922941700 +0200
@@ -695,36 +695,6 @@
 	return mpage_readpage(page, ntfs_get_block);
 }

-static void ntfs_readahead(struct readahead_control *rac)
-{
-	struct address_space *mapping = rac->mapping;
-	struct inode *inode = mapping->host;
-	struct ntfs_inode *ni = ntfs_i(inode);
-	u64 valid;
-	loff_t pos;
-
-	if (is_resident(ni)) {
-		/* no readahead for resident */
-		return;
-	}
-
-	if (is_compressed(ni)) {
-		/* no readahead for compressed */
-		return;
-	}
-
-	valid = ni->i_valid;
-	pos = readahead_pos(rac);
-
-	if (valid < i_size_read(inode) && pos <= valid &&
-	    valid < pos + readahead_length(rac)) {
-		/* range cross 'valid'. read it page by page */
-		return;
-	}
-
-	mpage_readahead(rac, ntfs_get_block);
-}
-
 static int ntfs_get_block_direct_IO_R(struct inode *inode, sector_t iblock,

 				      struct buffer_head *bh_result, int create)
 {
@@ -2036,7 +2006,6 @@

 const struct address_space_operations ntfs_aops = {
 	.readpage = ntfs_readpage,
-	.readahead = ntfs_readahead,

 	.writepage = ntfs_writepage,
 	.writepages = ntfs_writepages,
 	.write_begin = ntfs_write_begin,
@@ -2047,5 +2016,4 @@

 const struct address_space_operations ntfs_aops_cmpr = {
 	.readpage = ntfs_readpage,
-	.readahead = ntfs_readahead,
 };
EOT
wget -O Makefile.patch https://aur.archlinux.org/cgit/aur.git/plain/Makefile.patch?h=ntfs3-dkms
patch -p0 -N -i "Makefile.patch"
rm Makefile.patch
wget -O dkms.conf https://aur.archlinux.org/cgit/aur.git/plain/dkms.conf?h=ntfs3-dkms
echo 'MODULE_INFO(intree, "Y");' >> super.c
dkms add -m ${pkgname} -v ${pkgver}
dkms build -m ${pkgname} -v ${pkgver}
dkms install -m ${pkgname} -v ${pkgver}
echo Installation succeeded.
{% endhighlight %}

The script above downloads the files in the appropiate directory, and then uses the DKMS to build and install the module. 

Beware that I add a line to the module that marks it as part of the kernel tree, so that when the module is loaded, the kernel does not get tainted. I do this because a tainted kernel loses some debugging functionality. While it will be in tree at some point, it is not at the moment. If you experience problems with your kernel, consider it tainted and do not submit reports with this module loaded. Instead, make sure the module does not get loaded, reboot the system, catch the problem and submit a report with an untainted kernel. You can verify the condition of your kernel by issuing:

```
cat /proc/sys/kernel/tainted
```

Decode the value as explained [here](https://www.kernel.org/doc/html/latest/admin-guide/tainted-kernels.html).

In the end, what you have to do, in order to mount an NTFS partition using this driver, is to explicitly specify the type when issuing the mount command, like so:

```
# mount -t ntfs3 /dev/sdb1 /mnt
```

If everything went ok, the drive should be mounted (check in `lsblk` and `findmnt`). Should any error occur, start by first taking a look in `dmesg`.

Based on my use case (sharing via Samba), plus the documentation of this driver (available at `/usr/src/ntfs3-17.0.0/ntfs3.rst`) and the commercial variant (available [here](https://dl.paragon-software.com/doc/NTFS_HFS_linux_user_manual.pdf); the commercial variant is different from this driver but has some similarities), I mount my drives using this command line that I deemed optimal for my situation:

```
# mount -t ntfs3 -o umask=000,fmask=000,dmask=000,discard,force,prealloc,no_acs_rules,acl /dev/sdb1 /mnt
```

To avoid having to specify the type, the comments on the AUR package suggest using an udev rule. I personally haven't used that, as I mount my disks at boot using `/etc/fstab` anyway, but it definitely should work and is worth a try. They also offer suggestions for enabling easy mounting in desktop environments (I use Raspbian Lite, so headless).

Now, do some testing. For me, I easily saturate the link (around 115MB/s), which is awesome. On par with ext4, at least for the kind of speeds a gigabit connection supports, which is what I was looking for. Indeed, a proper solution. And this without any overclock on the Pi, which should not be required (my Pi4 is passively cooled, so I try not to push it excessively - I value quietness more).

I also provide an uninstall script, in case you want to completely get rid of this from your system (again, run as root and unmount all the disks using the driver so it can be unloaded from the kernel):

{% highlight bash%}
#!/bin/bash
pkgname=ntfs3
pkgver=17.0.0
prefix=/usr/src

rmmod ntfs3
dkms uninstall -m ${pkgname} -v ${pkgver}
dkms remove ${pkgname}/${pkgver} --all
rm -rf ${prefix}/${pkgname}-${pkgver}
echo Uninstall succeeded.
{% endhighlight %}

That's it, hope it serves you well. Hopefully, this will be integrated as soon as possible in the kernel and eventually reach the Pi as well, as this is pretty big. Finally a proper file system that works best with both Windows and Linux and with no headaches. I mean, I don't want to be the one to suggest it, but I am sure someone will soon boot Linux off NTFS. Maybe have a single drive with both Windows and Linux, with home and My Documents mapped to the same folder and boot in either OS using two different entries in the EFI boot menu. Interesting times ahead. This module definitely is an incredible contribution, I can't wait for it to ship - probably with 5.12, if not 5.11.

P.S. You can use `/etc/fstab` to mount the filesystem instead of running the mount command every time, especially if you want to have the drive mounted automatically at boot. If you do not know what to write there, which I don't know as well, again, Arch has a nifty utility to help us, courtesy of their excellent installation guide: `genfstab` - this generates an fstab file from the current mounts in the system. To install it on the Pi, do:

```
# apt install arch-install-scripts
```

Indeed, there is a Debian package for that. Then, mount the partition using ntfs3 as shown above, after which generate an fstab:

```
# genfstab -U /
```

This will write the fstab at stdout. Just copy the line containing your partition that is mounted using the ntfs3 driver and put it into your real fstab. It's that easy.

P.P.S. If you simply want to build the module manually in some folder where you get the files, execute this:

```
KVERSION=$(uname -r) CONFIG_NTFS3_FS=m CONFIG_NTFS3_LZX_XPRESS=y CONFIG_NTFS3_FS_POSIX_ACL=y make KDIR=/lib/modules/$(uname -r)/build
```

To load the module in the kernel, use `insmod` in the same folder:

```
insmod ntfs3.ko
```

To remove it from the kernel, use:

```
rmmod ntfs3
```

P.P.P.S. Of course, the script is universal, it works on any distro and you can simply delete the part that patches out the readahead support of your kernel is 5.8+.