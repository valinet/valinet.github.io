---
layout: post
title:  "Automatically backup the iPhone to the Raspberry Pi"
date:   2021-01-20 00:00:00 +0000
categories: 
excerpt: "As I said before, Raspberry Pi is a wonderful tiny device capable of a lot of stuff. The problem today is simple: I have an iPhone and want the peace of mind that it is always backed up should anything bad happen. Unfortunately, backing up the iPhone is not as simple as it seems - Apple only offers 5GB on iCloud, which is not enough for WhatsApp on its own, let alone the entire phone. So, today, I will show you how to set the Raspberry Pi as a backup server for your iPhone."
---

As I said [before](/2021/01/14/Get-better-NTFS-support-on-Raspberry-Pi.html), Raspberry Pi is a wonderful tiny device capable of a lot of stuff. The problem today is simple: I have an iPhone and want the peace of mind that it is always backed up should anything bad happen. Unfortunately, backing up the iPhone is not as simple as it seems - Apple only offers 5GB on iCloud, which is not enough for WhatsApp on its own, let alone the entire phone. 

When I was using Android, I did not have such a problem, as I manually copied the photos to the PC once in a while or automatically to Google Photos [before Google borked the service](https://support.google.com/photos/answer/10100180?hl=en), while the phone synced its settings with the Google account and WhatsApp was syncing up with a much beefier 20GB Google Drive account. It pretty much worked out.

Since switching to an iPhone, backing up WhatsApp became a problem. I don't mind manually copying the photos every 6 months or so, as I do not take that many photos, but backing up all the rest can in no way fit under that measly 5GB. Also, I want to migrate away from cloud as much as possible, that's why I have set my own server for backing up the PC, for mail, calendar, contacts and a few other stuff, all using a Raspberry Pi. So today, it is about enabling another service on it, namely an iPhone backup server. Decentralization.

This article guides you on how to set up the Raspberry Pi so that the iPhone automatically backs up to it once every day (in the end, you get a script and you can adjust the interval there, of course). It all will (hopefully) work seamlessly. That includes backing up wirelessly - you won't have to connect the iPhone to the Raspberry Pi using a USB cable - rather, the two have to be on the same network around midnight and the backup will go on just fine over Wi-Fi (technically, over the network - you don't need to connect the Raspberry Pi via wireless - Ethernet is more than fine as well).

To leverage all of this, you have to clone and compile the projects from a few Git repositories. These are not huge projects, compilation takes a few minutes for each of them, on the Pi.

## Architecture

I have to admit, the way Apple does all of this is pretty smart and interesting. The iPhone is not simply connected to the computer and exposed as an MTP or mass storage device. Instead, a daemon runs at each end, and communicates with one another. The communication takes the form of something that resembles network communication: services connect to each end and send packets to the respective daemon which forwards them to the other end over the wire.

On the phone, a few services run, like the one that allows the PC to do backups, one that allows screenshots to be sent to the PC, one that offers info and so on. These are all managed by the *lockdown* daemon on the phone. On the PC, you need a daemon that can talk with the *lockdown* daemon, and some tools that will leverage this communication and subsequently talk with the services available on the phone, as requested by the user.

Traditionally, the iTunes package provides such things: it comes with a Windows service called *Apple Mobile Device Service* which plays the role of the daemon, while iTunes itself can do backups on the phone, reinstall the software etc. On macOS, [things are pretty similar](https://medium.com/@jon.gabilondo.angulo_7635/understanding-usbmux-and-the-ios-lockdown-service-7f2a1dfd07ae).

Not necessarily for GNU/Linux, but certainly benefiting for it, there is a project that offers a couple of utilities that replicate the work of Apple's PC-side tools (iTunes), but that have no dependency on Apple's code or software. These are clean room reimplementations of those services and work great. The project is called [libimobiledevice.org](https://github.com/libimobiledevice), and we'll use a few components provided by it:

* idevicebackup2 - this is a tool that connects to the local daemon (it is also called the USB/service multiplexer) and subsequently to an Apple device and performs a backup of it
* ideviceinfo - obtains information about connected Apple devices
* idevicepair - pairs the computer with a connected Apple device

As you have noticed, I have mentioned all the tools but no service multiplexer (daemon). That's because, while libimobiledevice offers one (usbmuxd), that does not yet support WiFi connections to the phone, only by cable. Fortunately, someone stepped up and created a separate daemon that provides WiFi support as well: [usbmuxd2](https://github.com/tihmstar/usbmuxd2).

That's mainly what we need. So, next I will tell you how to acquire all of the above. Unfortunately, the documentation for this projects is not that extensive, but should you encounter any problems, you can ask on the issue trackers on GitHub. I recommend installing everything from HEAD (upstream), but you can also pick a particular release and iron out eventual small differences, if any.

Also, "--prefix" and "--enable-debug" below are universal. You can customize the path where these will install with the first, while the latter enables a debug build (take it out for smaller executables).

## Compile and Install

1. Make sure to install the prerequisites for all the projects:

   ```
   sudo apt install git build-essential autoconf automake libtool-bin pkg-config
   ```

2. We'll begin with dependencies of [libimobiledevice](https://github.com/libimobiledevice/libimobiledevice): [libplist](https://github.com/libimobiledevice/libplist) and [libusbmuxd](https://github.com/libimobiledevice/libusbmuxd) (depends on *libplist*).

   ```
   git clone https://github.com/libimobiledevice/libplist
   cd libplist
   ./autogen.sh --prefix=/usr/local --enable-debug --without-cython
   make
   sudo make install
   cd ..
   git clone https://github.com/libimobiledevice/libusbmuxd
   cd libusbmuxd
   ./autogen.sh --prefix=/usr/local --enable-debug
   make
   sudo make install
   cd ..
   ```

3. Next, the main *libimobiledevice*. Everything is standard, although a small patch is required for *idevicepair* so that we can enable WiFi syncing on the phone directly from it. Otherwise, you would have to disconnect the phone, connect it to a PC with iTunes, enable WiFi syncing there and connect it back to the Pi.

   ```
   sudo apt install libssl-dev
   git clone https://github.com/libimobiledevice/libimobiledevice
   cd libimobiledevice
   wget https://github.com/tihmstar/libimobiledevice/commit/2b05e9ea4c34b62f1d32f9e348877883f2e4683f.patch
   patch tools/idevicepair.c < 2b05e9ea4c34b62f1d32f9e348877883f2e4683f.patch
   rm 2b05e9ea4c34b62f1d32f9e348877883f2e4683f.patch
   ./autogen.sh --prefix=/usr/local --enable-debug
   make
   sudo make install
   cd ..
   ```

4. Next, [usbmux2](https://github.com/tihmstar/usbmuxd2). Again, everything is standard, except that you need to explicitly tell the linker to link to *libatomic* (because gcc on Raspbian Buster is slightly old (version 8) and is not forgiving about this). Also, for this to compile, first we need to compile [libgeneral](https://github.com/tihmstar/libgeneral) from the same developer.

   ```
   git clone https://github.com/tihmstar/libgeneral
   cd libgeneral
   ./autogen.sh --prefix=/usr/local --enable-debug
   make
   sudo make install
   cd ..
   sudo apt install libusb-1.0-0-dev libavahi-client-dev
   git clone https://github.com/tihmstar/usbmuxd2
   cd usbmuxd2
   LDFLAGS="-latomic" ./autogen.sh --prefix=/usr/local --enable-debug
   make
   sudo make install
   cd ..
   ```

5. In the end, depending on where you set up the installation directory, you may need to adjust the way you launch *usbmuxd* - it depends on libgeneral.so, but that might not be in the path of the loader; You have 2 options for this. What I did is, symlink *libgeneral* to the folder where libraries are usually kept, which is in the path of the dynamic loader:

   ```
   sudo ln -s /usr/local/lib/libgeneral.so.0 /usr/lib/arm-linux-gnueabihf/libgeneral.so.0
   sudo ln -s /usr/local/lib/libgeneral.so /usr/lib/arm-linux-gnueabihf/libgeneral.so
   ```

   Otherwise, when launching *usbmuxd*, append the path to *libgeneral* to the LD_LIBRARY_PATH environment variable:

   ```
   LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib/ usbmuxd ...
   ```

## Prepare and make initial backup

Now that you have all the necessary software installed, connect the iPhone to the Raspberry Pi. `dmesg` should show some relevant info.

The first thing to do is to enable the *usbmuxd* daemon. It comes as a *systemd* file. You can either use that:

```
sudo systemctl enable usbmuxd
sudo systemctl start usbmuxd
```

Or you can instead start it manually in the terminal. I actually recommend this way, at least for the time testing this. To run *usbmuxd* standalone, execute this:

```
sudo usbmuxd -v
```

Now, you can verify that the device is properly attached and detected. Make sure that the screen of the iPhone is unlocked, so you can hit the "Trust" button that will ask for your permission on the phone when executing subsequent commands:

```
idevice_id -d
```

A string should be displayed, corresponding to your device. Tip: from now on, you will enter some other *libimobiledevice* commands. All of them support passing the `-u` parameter, which allows you to pass and id obtained using the command above to apply the command to that specific device. In case you only connect a single Apple device to the Raspberry Pi, you can leave that out and the tools will pick the single device available.

Now, let's prepare for backing up. In order for backups to contain all the data on the phone, they have to be encrypted. I wish it could work the other way, but it is what it is. So, you need to enable that:

```
idevicebackup2 -i encryption on
```

All the settings you make are actually stored on your device. If you previously enabled encryption in iTunes, you can skip this probably. To be able to decrypt the data, you have to set a password (if you used this previously, you have set this as well). This password is different from the iCloud password and is stored only locally on your iPhone. DO NOT LOSE THIS PASSWORD! If you do not care about encryption, just set a dump password and write it into some file on the disk. If you lose the password, you need to reset the phone in order to back it up again, it cannot be reset and there are no controls for it on the phone actually. Now that you have been warned, you can execute this to set the password from Raspberry Pi (you will also need to provide the passcode on the phone):

```
idevicebackup2 -i changepw
```

To continue with this, you will need to do an initial full backup of your device. For this, run the following command, making sure the destination has enough space. Also, depending on the size of the data on your phone, this might take a while (remember that the Lightning connector is still just a crappy USB 2.0 port):

```
idevicebackup2 -i backup --full /mnt/disk/iPhone/Backup
```

Of course, replace `/mnt/disk/iPhone/Backup` with the actual location where you want to store the backup. Grab some coffee or some snacks and wait. The progress is displayed interactively, so at least you do not have to stress too much.

Once this is done, you will have a backup similar to the non-sense iTunes generates. Your data is in there. The tool subsequently recognizes it if you do an incremental backup. You can test this now using this command, which will take substantially lower time:

```
idevicebackup2 -i backup /mnt/disk/iPhone/Backup
```

When you are done experimenting, the last step is to enable WiFi sync, so that the device is detected by *usbmuxd2* when connected only to the same network as the Pi, without a direct cable connection between the two. If you compiled *libimobiledevice* with the small *idevicepair* patch as described above, you can execute this command:

```
idevicepair wifi on
```

Now disconnect the phone. Try to obtain info from the device using:

```
ideviceinfo -n
```

The `-n` switch specifies that you want to query network devices. Initially, keep the screen on, and hopefully the phone is picked up. If not, try maybe reconnecting or restarting *usbmuxd*. If you have done all the steps until now, it will eventually work by playing with it a bit.

## Automate

Basically, what we want to do is execute the above incremental backup every day at midnight. There is one more complication about this: when the iPhone is sleeping, it does not seem to respond to 'pings' from  *usbmuxd* - this is kind of expected, as wakeups shorten battery life. To ensure this works reliably, ideally you should have some notification around midnight that will wake the phone so that *usbmuxd* picks up the device and the backup can start. You can schedule some mail to send automatically, or set up a dummy shortcut using the Shortcuts app (you can hide the notifications from the Shortcuts app using [this trick](https://kittmedia.com/2020/ios-disable-shortcuts-automation-notifications/)). Or maybe just retry the connection for a few hours, the phone should eventually receive a notification or you may pick it up in the morning and it will eventually start syncing. You have to experiment a bit here for your use case. Personally, I am never sleeping at midnight, and some times I am on my phone, so it is no problem. Also, I don't mind if the backup is missed some days, I want it to be made a couple of times each week. The way Apple envisioned this Wi-Fi syncing feature is, I believe, when you switch from your phone to the PC, or just join the network, for it to quickly sync with iTunes if available. Otherwise, the phone just sleeps and will sync the next time it has the opportunity. Also, having it plugged in has a better likelihood of it waking up on its own from sleep and responding to 'pings' from the USB multiplexer. I just had this for a day, so experimentation is definitely required here.

The way I have this automation script at the moment is that it actually runs the daemon on demand, at the specified times, so that it doesn't needlessly 'ping' the iPhone at inappropriate times. Plus, it seems to be more reliable in detecting the phone when running for long. And is one less service to run. As I said, experimentation is what is required with it.

So, make a folder where you'll place this (for me, it is `/home/pi`). To run it automatically, use this *systemd* unit file:

```
[Unit]
Description=Apple Backup Service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=pi
Group=pi
ExecStart=/home/pi/backup/ibackup.sh
ExecStop=usbmuxd -X
WorkingDirectory=/home/pi/backup

[Install]
WantedBy=multi-user.target
```

The script, I called it `/home/pi/backup/ibackup.sh` contains this:

{% highlight bash%}
#!/bin/bash

while true; do



CURDATE=$(date +"%Y%m%d")
if [[ -f "$CURDATE" ]]; then
        echo "[ibackup] Backup for today exists."
        current_epoch=$(date +%s)
        target_epoch=$(date -d "tomorrow 00:00:01" +%s)
        to_sleep=$(( $target_epoch - $current_epoch ))
        echo "[ibackup] Sleeping for $to_sleep seconds (current epoch: $current_epoch)..."
        sleep $to_sleep
        rm $CURDATE
fi

# backup
echo "[ibackup] Killing network daemon..."
usbmuxd -X
echo "[ibackup] Starting network daemon..."
usbmuxd -d -U pi -v --nousb
echo "[ibackup] Waiting for network daemon..."
sleep 3
echo "[ibackup] Starting backup..."
try=0
while : ; do
        ((try=try+1))
        if [ $try -eq 1080 ]; then
                CURDATE=$(date +"%Y%m%d")
                echo failed > $CURDATE
                break
        fi
        ideviceinfo -n 2>&1 > /dev/null
        dv=$?
        if [ $dv -eq 0 ]; then
                echo "[ibackup] Device is online."
                break
        fi
        echo "[ibackup] Device is offline, sleeping a bit until retrying [$try]..."
        sleep 10
done
if [[ -f "$CURDATE" ]]; then
        echo "[ibackup] Timed out waiting for device, maybe we'll backup tomorrow."
else
        echo "[ibackup] Backing up..."

        idevicebackup2 -n -i backup /share/iPhone/Backup
        dv=$?
        if [ $dv -eq 0 ]; then
                # save backup status
                echo "[ibackup] Backup completed."
                CURDATE=$(date +"%Y%m%d")
                echo success > $CURDATE
                echo "[ibackup] Saving backup status for today."
        else
                echo "[ibackup] Backup failed, sleeping a bit until retrying..."
                sleep 10
        fi
fi
echo "[ibackup] Killing network daemon..."
usbmuxd -X


done
{% endhighlight %}

Do not forget to make the script executable (`chmod +x ibackup.sh`). Also, if you decide to run my script as is, you have to make *usbmuxd* execute as root when launched directly, as the script runs as your standard `pi` user. To do so, you can set the setuid bit on it:

```
sudo chmod u+s $(which usbmuxd)
```

## Conclusion

I think this is a very powerful way to keep you device's data safe, using open software on an open platform. The implementation is pretty flexible in my opinion, it only requires more testing and experience over what scenarios is the best. I will certainly update this post once I get some time to test different configurations for this idea.

In the end, I would like to profoundly thank and express my appreciation for the authors of these projects. I am sure improvements will come, as the network support, for example, was only recently implemented, so work is actively done in this area.

Please share your thoughts and suggestions using the comments area below.