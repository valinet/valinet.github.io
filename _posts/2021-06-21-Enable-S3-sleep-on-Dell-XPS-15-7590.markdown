---
layout: post
title:  "Enable S3 sleep on Dell XPS 15 7590"
date:   2021-06-21 00:00:00 +0000
categories: 
excerpt: "My fight against S3 sleep continues, I guess. You can read [here](https://valinet.ro/2020/12/08/Enable-S3-sleep-on-Lenovo-Yoga-Slim-7-14are05.html) about my previous partially successful attempt for disabling this on the Lenovo Yoga Slim 7 14are05, an AMD Ryzen 4000-based laptop. Today, I attempt to do the same on the Dell XPS 15 7590 - an older machine from 2019, powered by the 9th generation Intel Core platform."
---

My fight against S3 sleep continues, I guess. You can read [here](https://valinet.ro/2020/12/08/Enable-S3-sleep-on-Lenovo-Yoga-Slim-7-14are05.html) about my previous partially successful attempt for disabling this on the Lenovo Yoga Slim 7 14are05, an AMD Ryzen 4000-based laptop. Today, I attempt to do the same on the Dell XPS 15 7590 - an older machine from 2019, powered by the 9th generation Intel Core platform. I have recently got this laptop because I felt the need for a bigger screen (which is also extraordinary, very bright and "only" FullHD so very power efficient) when on the go. I chose this older model instead of the latest 2021 model because it still has useful ports (the reality is, after a few years since the introduction of USB-C, the single USB-C thing I use often is the iPhone charger); the latest Dell XPS 15 only comes with USB-C, so you need adapters for everything including plugging a damn thumb drive into the laptop. Also, not to mention the price, as I was able to get a barely used unit brought from the United States at the beginning of 2021 for half the price of a new unit on the local market. It included a base config, but with the important bits coming from factory, like the beefy 97Whr battery (which was also a factor for choosing this, as it offers me insane runtimes compared to my previous laptop), but otherwise barebones config (8GB RAM, 256GB storage); that hasn't been a problem, as it allowed me to save some cash and I have reused the components from my previous notebook (a ThinkPad 25 which I surely miss the extraordinary keyboard), and also still keep that functional.

Anyway, enough about the laptop, let's talk about the topic of this article: forcing S3 sleep instead of connected/modern standby. The reasons are plenty known, including less power consumption during sleep, and the lack of necessity for being "connected" during sleep - the laptop does not notify you when new mail arrives anyway, so why would that be needed?...

Compared to the Lenovo Yoga Slim 7 I attempted this fix previously on, the situation here is a bit better, in that you do not need to patch ACPI: the BIOS still advertises S3 support, but it gets disabled in Windows, as the BIOS also advertises S0 low power idle sleep support. Thus, to have S3, one needs to disable S0 low power idle sleep. There are 2 ways to do this:

* The generic method is to use `PlatformAoAcOverride` available since Windows 10 20h2. I described this method in the past (basically `reg add HKLM\System\CurrentControlSet\Control\Power /v PlatformAoAcOverride /t REG_DWORD /d 0`).
* Particularly for this machine, one can set a UEFI variable corresponding to whether the BIOS should advertise S0 low power idle or not. The Dell BIOS used to have an option that allowed you to change this, but since a few BIOS releases ago, the toggle was removed from the GUI. The reason? Probably the fact that they do not really want to support S3 sleep, as there are some quirks we will address bellow. The procedure is easy: [get a patched grub](https://github.com/datasone/grub-mod-setup_var) that supports the `setup_var` command and toggle the option. It will come in the form of an EFI application that you have to "boot" from. There are multiple ways to do this: add a boot entry using the BIOS, [using efibootmgr from Linux](https://wiki.archlinux.org/title/EFISTUB), [boot it from a USB stick](https://www.reddit.com/r/Dell/comments/iftkdw/enable_s3_sleepdisable_modern_standby_on_xps_7590/) or just  format a thumb drive using [Ventoy](https://www.ventoy.net), an open-source tool that let's you boot from a USB stick that will allow you to subsequently boot any ISO/EFI stored on it. It is very much recommended, I came to depend on it heavily. After booting in the patched grub, check the current value at offset `0x14`, which should be `0x01`, and then disable S0 low power idle support by issuing the command `setup_var 0x14 0x00`.

That's it; reboot and check that S3 is available using `powercfg -a` in an administrative command prompt.

There is, unfortunately, a side effect of doing this: Bluetooth does not work after sleep. That's one of the things Dell choose not to bother with and thus the decision to hide S3 support was chosen. Fortunately, a fairly robust solution exists for this: you have to reset the USB root hub after sleep, and then Bluetooth will be redetected and available again.

To do this, I recommend the following procedure, as described by [this](https://www.reddit.com/r/Dell/comments/ge7f7v/solution_xps_15_7590_disappearing_bluetooth_after/) Reddit thread, and updated by [this](https://www.reddit.com/r/Dell/comments/mhtqzh/solution_updated_2021_xps_15_7590_disappearing/) post as well:

1. Identify "Intel(R) Wireless Bluetooth(R)" in Device Manager under the Bluetooth category. Right click, choose "Properties", go to "Details", browse for "Last known parent" in the drop down list, right click on the value and choose "Copy". In my case, the value is `USB\ROOT_HUB30\4&1f0d482e&0&0`.
2. We'll create a scheduled task that will run when waking from sleep and will reset the USB root hub. Open Task Scheduler, click on "Task Scheduler Library", right click an empty area in the list and choose "Create Task".
3. Switch to "General" and type in a name that you want.
4. Tick "Run with highest privileges". Click "Change user or group..." and type in "SYSTEM" in the text box that appears and then OK.
5. Switch to "Triggers". Click "New...".
6. Change "Begin the task:" to "On an event".
7. Change to "Custom" and click "New event filter...".
8. Click "By source" and tick "Power-Troubleshooter" in the drop-down menu (**make sure you only see this one in the "Event sources:" box**).
9. Change "All Event IDs" to "**1**" (1 is an event that happens after waking up).
10. Click "OK", then click "OK" again.
11. Switch to "Conditions".
12. Make sure **NOTHING IS TICKED** (if you cannot untick something, first tick it's parent and then untick the child and then the parent).
13. Switch to "Actions". Click "New...".
14. "Action:" > "Start a program".
15. Paste `C:\Windows\System32\pnputil.exe` in the "Program/script" box.
16. Paste `/restart-device "USB\ROOT_HUB30\4&1f0d482e&0&0"` in the "Add arguments (optional)" box. Make sure to replace `USB\ROOT_HUB30\4&1f0d482e&0&0` with the path of your USB hub you obtained in step 1.
17. That's it, okay everything and it should be good to go. Test by putting the laptop to sleep and then waking up. You may see the Device Manager refresh and the Bluetooth will appear there after sleep as well, like it should do under normal operation.

The updated post mentioned using `pnputil` instead of `devcon` or `devmanview`. This has the advantage that it works even if something like a USB thumb drive is plugged in the USB port, plus that it comes built-into Windows, so you do not need to install the Windows Driver Kit or download any third party tool.

So far, I haven't found any other problems regarding S3 on this laptop. It mostly behaves like any old laptop used to behave before this craze regarding connected standby came and plagued the industry. Indeed, resuming from sleep is a bit slower, but who cares? I'd rather take the more efficient sleep times and the piece of mind that it actually sleeps and does not start ramping the fans in the backpack because Windows Update decided to kick in or something like that, with the slight "annoyance" of waiting 1 second more on each resume. Whatever... plus, this is real ecology, as S3 is much more efficient than S0 low power idle, which is basically S0 normal operation but with most threads suspended.

Indeed, on this laptop, you can have more success with this technique, so much so that I actually recommend daily driving this (there are no weird problems like on the Lenovo laptop, like the keyboard backlight going crazy after sleep etc).

## References

[1] [[Solution - updated 2021] XPS 15 7590: Disappearing Bluetooth after S3 Sleep](https://www.reddit.com/r/Dell/comments/mhtqzh/solution_updated_2021_xps_15_7590_disappearing/)

[2] [[SOLUTION] XPS 15 7590 - Disappearing Bluetooth after SLEEP](https://www.reddit.com/r/Dell/comments/ge7f7v/solution_xps_15_7590_disappearing_bluetooth_after/)

[3] [Disabling modern standby on the XPS 15 9570?](https://www.reddit.com/r/Dell/comments/myeizg/disabling_modern_standby_on_the_xps_15_9570/)

[4] [Enable S3 sleep/disable modern standby on XPS 7590 (August 2020)](https://www.reddit.com/r/Dell/comments/iftkdw/enable_s3_sleepdisable_modern_standby_on_xps_7590/)