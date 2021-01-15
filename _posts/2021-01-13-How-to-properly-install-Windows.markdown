---
layout: post
title:  "How to properly install Windows"
date:   2021-01-13 00:00:00 +0000
categories: 
excerpt: "What do I mean by the title, you ask. Like, simple, Google 'windows 10', click the first result, change user agent of the browser to some other OS, Microsoft will finally let you download an ISO, download the ISO, use Rufus to burn it into some USB drive and install off that, right? Well, not exactly. I mean, that's fine, but what to do when you have multiple computers to install on? For example, I have a desktop at home, a workstation at work, and a laptop. I basically need the same setup on all of them, and that includes the same settings, plus quite a few programs."
---

What do I mean by the title, you ask. Like, simple, Google "windows 10", click the first result, change user agent of the browser to some other OS, Microsoft will finally let you download an ISO, download the ISO, use [Rufus](https://rufus.ie) to burn it into some USB drive and install off that, right?

Well, not exactly. I mean, that's fine, but what to do when you have multiple computers to install on? For example, I have a desktop at home, a workstation at work, and a laptop. I basically need the same setup on all of them, and that includes the same settings, plus quite a few programs: Visual Studio, Qt, Microsoft Office (for the occasional paper/presentation I have to work on), Android Studio etc.

That takes time. Also, what if you want to quickly refresh your OS? Reinstalling seems like a nightmare. Especially considering that it is not only the programs, you have to set up Windows again, including that horrible OOBE experience. And then disabling all the nagging and annoying stuff in Windows 10, telemetry etc. It takes lots of time and it is boring.

So, what I decided to do and, in the end, I highly recommend, is to take the time and do it once in a virtualized environment, and then use some Windows tools to capture the install, generalize it and generate an install medium and ISO that you can use to later deploy your image anywhere. Like, seriously, I now reinstall my OS monthly, and with this method, I am up and running in 15 minutes. Indeed, it took me a few days to figure it out, but it was worth the effort.

I recommend watching the following video which explains very well the general idea for the process. Afterwards, read on to find out how I adapted his idea for my use case.

{%- include youtube_embed.html id="PdKMiFKGQuc" -%}

Okay, so, I recommend doing everything in a virtual machine. I use Hyper-V, so I will refer to that, but you can use whatever virtualization software you want.

## Create an internal virtual switch

You'll need this because you need a way to communicate from the host to the VM, so you have a place to take all the scripts and installation files from. I communicate using a network share I set on the host.

## Create the VM

Make sure to add 2 NICs, one for NAT access to the Internet, one for the internal network which enables communication with the host. Leave Secure Boot enabled, assign a generous amount of RAM (I set 8192MB, I have 24GB physical), DISABLE dynamic memory (trust me, this saves you headaches further on with dism), "memory weight" to High, more than 1 virtual processors (I set it to 4, I have 8 physical), and attach a Windows 10 ISO to its virtual DVD drive.

Also, if you want to enable components like Hyper-V or WSL2 in your master image, you will need to enable nested virtualization for your VM. To do so, run this in  an administrative PowerShell, as explained [here](https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/user-guide/nested-virtualization):

```
Set-VMProcessor -VMName <VMName> -ExposeVirtualizationExtensions $true
```

## Install Windows

Boot from the DVD and install Windows. Do the standard stuff until you get to the "Select your region" screen (when the OOBE experience starts). Once on that screen, press [Ctrl] + [Shift] + [F3]. This will reboot you to the desktop in audit mode. That's one different thing I do as opposed to the video. This allows us to customize the default profile ([CopyProfile](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/customize-the-default-user-profile-by-using-copyprofile)). We will make the changes here, and when we are ready, we will generalize our image and prepare it as a source of files for a Windows installation.

## Install programs and customization

Now's the time to do the actual setup: install our custom software and customize the system with the settings we want. Between each major step, I recommend making a checkpoint for the VM, so that it is easier to recover to the last good state in case you mess something up. This proved very useful for me, and allowed me to have the cleanliest output in the end, but beware that checkpoints can become pretty large, so be prepared with hard drive space. In the end, you can consolidate checkpoints you do not need anymore and regain some of that hard drive space back.

These are the steps I do while in this mode (of course, feel free to take out or customize according to your needs):

* Install Microsoft Office - I do this first so that I can also get it updated later on along with Windows, and because some updates install some crap we have to remove later on as well (OneDrive and some PWAs). Personally, I install the C2R Office 2019 version, not 365.

* Run a preparation script - my example available [here](https://gist.github.com/valinet/54288631da9b153972bbf1fbb4fb0789). Basically, this does the following:

  * removes a bunch of unnecessary built-in apps (Spotify, Candy Crush, Xbox etc)
  * disables telemetry related settings, Cortana, SmartScreen filter, Wi-Fi Sense, Bing search in Start menu and some other crap
  * disables some unnecessary built-in components (Windows Media Player, Work Folders)
  * disable and purge OneDrive
  * disables some unnecessary built-in services
  * delete most of the default tiles in Start menu
  * enables some components I always use (.NET Frameork 3.5, Hyper-V, Internet Information Services, Telnet Client, Virtual Machine Platform, Windows Hypervisor Platform, Windows Subsystem for Linux)
  * applies my shell customizations (never combine taskbar buttons, hide search bar, task view button, hide desktop icons etc)
  * enables Windows Photo Viewer
  * add Take Ownership to Explorer context menu
  * set my custom wallpaper
  * patch conhost for black titlebars as explained [here](https://valinet.ro/2020/09/07/Case-study-Get-dark-command-windows-all-the-time-in-Windows-10.html)
  * patch shellstyle.dll to remove the command bar in Explorer as explained [here](https://www.askvg.com/how-to-make-folder-band-auto-hidden-in-windows-vista/)
  * patch Task Manager so it looks great on high DPI displays as explained [here](https://valinet.ro/2020/09/06/Fix-Task-Manager-bluriness-on-different-DPI-monitors-in-Windows-10.html)
  * pre-configure [OldNewExplorer](https://msfn.org/board/topic/170375-oldnewexplorer-119/)
  * remove the crappy Office PWAs that Microsoft Edge installs
  * register a task that runs at startup and centers the text in the title bars using my [WinCenterTitle](https://github.com/valinet/WinCenterTitle) app
  * register a service that runs at startup and that hides the search bar in Explorer ([here](https://github.com/valinet/HideExplorerSearchBar))
  * ~~register a service that runs at startup and allows my audio potentiometer to work~~ not needed anymore, I now made this device plug and play ([here](https://github.com/valinet/audiopot-plus))
  * register a task that runs at startup that updates (and initially installs) software that is available in the [Ninite](https://ninite.com/) catalogue; this way, most of my software is kept up to date, using a lightweight mechanism (not winget, not Chocolatey etc) and I avoid embedding this in the install image - an example XML is available [here](https://gist.github.com/valinet/53775f76784917e247c16d1232329342)
  * some other things; it is based on the script suggested in the video I recommended you, but it is not destructive, in that it does not uninstall stuff from the system, patch the base image etc.

* Run Windows Update - make sure to click on "Advanced options" and check "Receive updates for other Microsoft products when you update Windows" so that you get updates for WSL, maybe Office etc as well. Run, reboot as many times until no more updates are available.

* Update all apps from the Windows Store. You are not left with much, probably 10 updates or so will be available.

* Install all the other software. Here, I mean all the software that is pretty much the final version ever or has its own update mechanism, software that is not available from Ninite so can't be installed using the previous method. This is the order in which I do it:

  * 7-Zip and Everything - these first in order to configure them with my custom settings.
  * Firefox and Thunderbird - for these, I copy a saved profile that is void of any data except my customizations (custom CSS, add-ons etc). For Firefox, I recommend installing addons live when the user account gets set up on the target computer, as this seams more reliable of a method. In Thunderbird, it usually works fine preinstalling everything now.
  * Install and configure development tools:
    * Windows Software Development Kit
    * Visual Studio 2019
    * Windows Driver Kit
    * Windows Assessment and Deployment Kit
    * Qt
  * Install rest of the software:
    * Wireshark
    * Apache Directory Studio
    * Arduino IDE
    * Cheat Engine
    * Internet Download Manager
    * HxD
    * Typora
    * Android Studio
    * Resource Hacker
    * WinAeroTweaker
    * Windows 7 Games

* Prepare a script that runs on newly created user acounts and does all the initial preparation work - my example available [here](https://gist.github.com/valinet/4fcd335237a981250fe10ef03bde3e20). To have it run on new accounts, place it in "shell:startup", but make sure not to reboot the VM with it there as it will mess the VM. This is the easiest method and works really well. The script runs in the context of the new user, without elevation, so do all the admin related things in audit mode. It may not be the prettiest way, but it is versatile and quick, perfect for images you deploy for your own use, where you can refrain from messing with the command window while it does its job. For me, this script does the following:

  * Open Firefox, so it sets its profile
  * Uninstall Microsoft Photos, as it seemed to somehow get reinstalled.
  * Set keyboard layout, regional options etc
  * Install some appx packages:
    * Pride 2020 theme - a cool theme with variations of the default Windows 10 wallpapers
    * Ubuntu - the files for the distribution I use with WSL
    * HEVC Video Extension - enables support for viewing new image/video formats in some apps (this used to have a free OEM version which you can still find around, especially [here](https://store.rg-adguard.net/) - type "https://www.microsoft.com/en-us/p/hevc-video-extensions-from-device-manufacturer/9n4wgh0z6vhq" in the box and choose "Retail" instead of "RP" and download it from there)
  * Kill Firefox and copy over my addons
  * Autoconfigure my [Thunderbird Toasts](https://github.com/valinet/ThunderbirdToasts) add-on
  * Copy Thunderbird to startup
  * Open "Default Apps" in Settings so I can set my default apps (it is easier to do this manually then to embed it reliably in the install media - Microsoft became really paranoid about random agents changing your defaults)
  * Install the Qt VS extension. (it does not install properly if done in audit mode, for some reason)
  * Delete the script from startup, so that it does not run again on subsequent logons.
  * After this runs, all that is left for me to do manually is to set my cursors to large (haven't done a way to do so yet in audit mode), register my Internet Download Manager license and set the program up (differs for each computer), register Microsoft Office (again, the key differs for each computer), run Windows Update and install the missing drivers, if any, very important: change the computer name to whatever name I want, enable all my Firefox addons, unpin 3 remaining tiles in Start, unpin Edge and Store and pin Firefox and Thunderbird (again, easier to do like this than to embed in the image) and that's it. I can do this in 5 minutes maximum, which is a great improvement compared to how long it took me before with all that software install part especially.
  * The next time you log in, Ninite will run and will install all the other software; it will further on run at each startup, checking and updating your software when necessary.

* Prepare for deployment - I highly recommend checking your disk for errors and cleaning it off temporary files - this may save you a whole lot of headaches in the future, and is usually the cause later tools seem to work but the install fails.

  * To check the disk, right click on C: in This PC, choose Properties, Tools, Check

  * To clean the disk up, run this in an administrative command window:

    ```
    cmd.exe /c Cleanmgr /sageset:65535 & Cleanmgr /sagerun:65535
    ```

* Generalize and run Sysprep - once you have applied all the customizations, it is time to let Sysprep run. Have it shutdown, check "Generalize" and place your unattend.xml file in "C:\Windows\System32\Sysprep" beforehand. Once this is done, and the system shuts down, do not boot the VM again back up, as that will reinitialize the system and customize it for the hardware in the VM, and you will pretty much ruin all your work.

## Generating the unattend.xml file

The video explains this pretty well. You will need to use "Windows System Image Manager" that comes with the Windows ADK (Tools) to generate such a file. I have mine exactly the same as in the video, plus the "CopyProfile" functionality. For reference, check it out [here](https://gist.github.com/valinet/e39909c7dd414e73dcc79599befc22d0).

## Capturing the image file

The way I do this, is I set the VM to boot back again from the Windows ISO. Once in the Windows Setup screen again, press [Shift] + [F10] to bring up a command window again. Then, I execute these commands which will assign a proper IP to the NIC linked to the internal switch, mount my network share inside the Windows Setup environment, and use DISM to capture the hard drive and create an image:

```
wpeinit
netcfg -winpe
netsh interface ipv4 set address name="Ethernet 2" static 192.168.1.2 255.255.255.0 192.168.1.1
net use Z: \\192.168.1.1\Windows10Custom /user:Valentin

dism /Capture-Image /ImageFile:Z:\install.wim /CaptureDir:C:\ /Name:"Windows 10 Pro v20H2" /ScratchDir:Z:\temp
```

Of course, as usual, customize this to your own needs. If your account does not have a password, "net use" won't work, unless you enable password-less login from stuff other than the console in Group Policy on the host system at "Computer Configuration - Windows Settings - Security Settings - Local Policies - Security Options - Accounts: Limit local account use of blank password to console logon only". Set that to Disabled.

DISM will take some time. In the end, you will get an "install.wim" file. This replaces the "install.wim" file in a directory tree of files extracted from a Windows 10 ISO. "install.wim" is in the "sources folder".

## Install the OS

You may initially want to set up another test VM to test your settings into. To install the OS there or on some actual hardware, there are 2 ways, choose whatever you find more sensible for you. I recommend both of them:

* Generate a new ISO file with your modifications. To do that, open "Deployment and Imaging Tools Environment" from Start (as before, this comes with Windows ADK) and execute this:

  ```
  oscdimg.exe -m -o -u2 -udfver102 -bootdata:2#p0,e,bC:\Windows10Custom\iso\boot\etfsboot.com#pEF,e,bC:\Windows10Custom\iso\boot\efisys.bin C:\Windows10Custom\iso C:\Windows10Custom\Windows10v20h2.iso
  ```

  If you miss some files like "efisys.bin" from your ISO, get them from [here](https://github.com/joeldidier/Bootable-Windows-ISO-Creator/blob/master/releases/0.5.0-beta/bwic_release_0.5.0..zip), they were ripped from some Microsoft Windows ISO that actually contained them.

* Install using PXE; with PXE, you can install an OS over the network. A good, quick and free (for home) software that lets you do this is [Serva](https://www.vercot.com/~serva/). Setup is easy, follow something like [this](https://medium.com/mitchtalmadge/using-serva-to-install-windows-over-your-home-network-4cff21464b55).

## Conclusion

In the end, you end with a clean, customized install of Windows 10, the official Microsoft way. For me, this method is ideal, and not hard to follow, now that I know these stuff. That's why I summarized here these ideas, so that it is easier for you to follow along.

Again, as a reference, here is the list of custom files I use for this:

* [deploy.ps1](https://gist.github.com/valinet/54288631da9b153972bbf1fbb4fb0789) - script I use to prepare the install when in audit mode
* [StartupOnce.bat](https://gist.github.com/valinet/4fcd335237a981250fe10ef03bde3e20) - script that runs at startup, once, for each new user
* [unattend.xml](https://gist.github.com/valinet/e39909c7dd414e73dcc79599befc22d0) - my custom unattend.xml
* [ApplicationMaintenance.xml](https://gist.github.com/valinet/53775f76784917e247c16d1232329342) - template for scheduled task that runs Ninite at startup
* [WinCenterTitle.xml](https://gist.github.com/valinet/9b8ec0b8932557cecdbe516cda6ef8d2) - template for starting WinCenterTitle at startup (deploy.ps1 also contains code that automatically installs this - I should and probably will integrate this almost setup program in the WinCenterTitle upstream at some point)

Hopefully, this new style of article, when I keep it more condensed and just mention the important stuff is easier to follow and helps you do things quicker. Tell me your opinion down in the comments, if you'd like, that's be awesome.