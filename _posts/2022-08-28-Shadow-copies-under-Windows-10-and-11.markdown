---
layout: post
title:  "Shadow copies under Windows 10 and Windows 11"
date:   2022-08-28 00:00:00 +0000
categories: 
excerpt: "Ever since I got into the world of managing servers, working with VMs etc, I have kind of grown acustomed to being able to quickly restore the systems I work with to a previous point in time. But having this confidence with most of the tools I manage (the company servers, the hypervisors I use), the OS has always been a sore spot for me - I was always scared of some program installing and then messing up my Windows install in some irreversible way. Today, it's time to address this."
---

Ever since I got into the world of managing servers, working with VMs etc, I have kind of grown acustomed to being able to quickly restore the systems I work with to a previous point in time. But having this confidence with most of the tools I manage (the company servers, the hypervisors I use), the OS has always been a sore spot for me - I was always scared of some program installing and then messing up my Windows install in some irreversible way.

It was only until relatively recently that I have discovered that Windows does indeed supports snapshots on its default file system (NTFS). And not only on recent versions, but for almost 20 years now, ever since the days of Windows XP. It's called shadow copies and saw its prime time during the Windows Vista/7 era, only to see it kind of dismissed with the launch of Windows 8 for reasons I do not really understand. This feature is EXTREMLY useful to have - not only can you restore the entire C: drive in case some software messes it up, but you can also restore previous versions of the files you have been working on. This is immensily useful and time saving, so since I have discovered this, I really cannot live without it.

The journey begun when I wondered how to have the "Previous versions" tab (hosted by `twext.dll`; `tw` stands for "timewarp") in the Properties sheet of an item from File Explorer actually populated with previous versions. Items displayed in there can come from one of these providers:

* Volume Shadow Copy ("snapshot") - Windows' disk snapshots functionality
* Windows 7 Backup / Restore ("safedocs") - the legacy backup solution in Windows
* File History ("fhs") - Windows 8-era replacement for safedocs
* System Restore points

Despite that, the instructions in the window say that files displayed there only come from File History or restore points; File History creates backups of your files on an external drive on specified intervals. This works, but it's been always extremly buggy, with Windows sometimes failing to recognize the backups once you reinstalled the OS, plus some random slowdowns when copying files, or even the feature not being able to make an initial snapshot. Plus this worked for specific folders, not entire drives. I think even Microsoft knew how poor and laughable their implementation was, plus they surely do anything to push you over to storing backups in OneDrive, so, with Windows 11, they completely removed the UI for configuring this via the Settings app (as with other things), putting a final nail in the coffin for this feature that never worked right (there is remaining UI in the legacy Control Panel, but that alone can't be used to configure all aspects of this feature).

But as I have discovered, snapshots taken using the volume shadow copy service also generate entries in the "Previous versions" tab. Cool! So it means that ll we have to do is enable snapshots for our volumes and we'll start seeing entries there, right?

![Previous versions tab in the Properties sheet](https://user-images.githubusercontent.com/6503598/187086678-e8decbec-33d0-4058-b0a8-259dd29636b3.png)

Well, yeah. Unfortunately, due to another idiotic Microsoft move, they have completely removed the UI for configuring shadow copies from client Windows (it's still available in server SKUs). But the functionality is still there, they just don't deem it important, for some stupid reason, for regular users.

So, becuase we have no UI for configuring a schedule for snapshots, I have resorted to setting up a scheduled task that runs the following command every 15 minutes:

```
wmic shadowcopy call create Volume=C:\
```

What this does is create a snapshot of drive C: every 15 minutes. By policy, snapshots cannot take more than 10 seconds, and I experience no slowdowns anyway, so I am fine with executing them this often. This means I can lose, at worst, a maximum of 15 minutes of saved work, which sounds really good to me.

This is enough to get you the basics. Even if you stop reading here to return at a later time, you'll be covered against the basics. Follow on to learn more.

### Increase the number of snapshots

By default, Windows is pretty conservative in the number of snapshots you can take (64) before older snapshots are deleted. The system allows for a more reasonable maximum of 512, which you can configure by setting `MaxShadowCopies` (DWORD) under `HKEY_LOCAL_MACHINE\System\CurrentControlSet\Services\VSS\Settings` as per [these instructions](https://docs.microsoft.com/en-us/windows/win32/backup/registry-keys-for-backup-and-restore). Reboot after making the changes.

### Increase the disk quota snapshots can take

By default, snapshot contents are stored on the same drive you are snapshotting. You can increase the maximum space snapshots can take on your hard driver by issuing such a command in an administrative prompt:

```
vssadmin Resize ShadowStorage /For=C: /On=C: /MaxSize=UNBOUNDED
```

The command above removed any quota on the space snapshots can take - be careful, this can quickly fill your disk; you can also specify something like `100GB` instead of `UNBOUNDED` there and so on. Find out more about the command by running: `vssadmin Resize ShadowStorage`.

### View all snapshots

You may notice, depending on your usage, that some snapshots are not displayed in the "Previous versions" tab (only 64 of them or so may be displayed). It's indeed a display bug that sometimes happens; to consult the entire list, use:

```
vssadmin List Shadows
```

Alternatively, you can use a full featured third party GUI like [ShadowExplorer](https://www.shadowexplorer.com/downloads.html) or [ShadowCopyView](https://www.nirsoft.net/utils/shadow_copy_view.html).

Do note that under Windows 11 22H2-based builds (22621+), this functionality seems to be broken altogether; read more about my investigation regarding this [here].

### Restore an entire drive

This is where thing get a bit more complicated. But, first of all, do not right click a drive and go to Properties - Previous versions and attempt to restore from there. It's inneficcient and takes a ton of time and will likely mess your system. That feature basically copies over files from the snapshot over your current files, which is very unlikely to work properly, so do not goi that route. You have been warned.

Instead, remember how I told you that Microsoft 'inteligentlly' removed the GUI portion of shadow copies from client SKUs? Well, not only that; you see, `vssadmin` under client SKUs offers fewer option than it does under server SKUs.

For example, here's `vssadmin /?` under Windows 11:

```
vssadmin 1.1 - Volume Shadow Copy Service administrative command-line tool
(C) Copyright 2001-2013 Microsoft Corp.

---- Commands Supported ----

Delete Shadows        - Delete volume shadow copies
List Providers        - List registered volume shadow copy providers
List Shadows          - List existing volume shadow copies
List ShadowStorage    - List volume shadow copy storage associations
List Volumes          - List volumes eligible for shadow copies
List Writers          - List subscribed volume shadow copy writers
Resize ShadowStorage  - Resize a volume shadow copy storage association
```

And here it is under Windows Server 2022:

```
vssadmin 1.1 - Volume Shadow Copy Service administrative command-line tool
(C) Copyright 2001-2013 Microsoft Corp.

---- Commands Supported ----

Add ShadowStorage     - Add a new volume shadow copy storage association
Create Shadow         - Create a new volume shadow copy
Delete Shadows        - Delete volume shadow copies
Delete ShadowStorage  - Delete volume shadow copy storage associations
List Providers        - List registered volume shadow copy providers
List Shadows          - List existing volume shadow copies
List ShadowStorage    - List volume shadow copy storage associations
List Volumes          - List volumes eligible for shadow copies
List Writers          - List subscribed volume shadow copy writers
Resize ShadowStorage  - Resize a volume shadow copy storage association
Revert Shadow         - Revert a volume to a shadow copy
Query Reverts         - Query the progress of in-progress revert operations.
```

Again, why they decided to do this is beyond my understanding, probably some stupid business decision. Fortunately, as described [in this Reddit post](https://www.reddit.com/r/sysadmin/comments/w7fwgp/restore_shadow_copies_from_cli/), you can trick `vssadmin` into thinking it's running under a server SKU by using `WinPrivCmd`, a redirector for some API calls programs use to find various information from Windows, including whether the program runs under a client or server SKU. To get the server `vssadmin`, run this:

```
winprivcmd /ServerEdition vssadmin /?
```

If the drive you're trying to restore is not the system drive, than you can use a command like this one to revert to a snapshot of it:

```
vssadmin Revert Shadow /Shadow={c5946237-af12-3f23-af80-51aadb3b20d5} /ForceDismount
```

Again, you can identify the exact snapshot GUID using `vssadmin List Shadows`. Actually, I don't really know if this works from client SKUs anyway, even if you trick it into recognizing it like this - I don't know if the backend supports it. Anyway, read on for a workaround that works for all scenarios, including your system drive.


First of all, you can't restore a snapshot on the system drive from an online system. To workaround this, I recommend booting into a Windows Server-based live image. The easiest way to do this is to grab a pen drive and use [Rufus](https://rufus.ie/en/) to create a Windows To Go installation on it using a Windows Server ISO. Then, boot that. When making the bootable drive, make sure to uncheck the "Prevent Windows To Go from accessing internal disks" option when clicking "Start" in Rufus.

Then, there are a couple of pre-requisties for the revert operation to work. I skipped the documentation and looked directly on the disassembly of the `VSSVC.exe` executale (the Volume Shadow Copy service):

```
bool __fastcall CVssCoordinator::IsRevertableVolume(CVssCoordinator *this, const unsigned __int16 *a2)
{
  CVssCoordinator *v3; // rcx
  char v4; // bl
  CVssCoordinator *v5; // rcx
  CVssCoordinator *v6; // rcx

  v4 = 0;
  if ( !CVssCoordinator::IsSystemVolume(this, a2)
    && !CVssCoordinator::IsBootVolume(v3, a2)
    && !CVssCoordinator::IsPagefileVolume(v5, a2) 
	&& !CVssCoordinator::IsSharedClusterVolume(v6, a2) )
  {
    return 1;
  }
  return v4;
}
```

So, basically, you cannot restore a system or booted volume, so that's why you can't restore the `C:` drive while the OS s running. Kind of makes sense. You also cannot restore it if it's a shared cluster volume; I don't know exactly what that is, but on regular configurations, it's not. All that remains is not to have a page file on the volume. That's easy to get around: in Start, search for "advanced system settings", under "Performance", click "Settings", "Advanced", "Virtual memory" section, "Change", untick "Automatically manage paging file size for all drives", then select your "C:" drive, check "No paging file", click "Set", then "OK", "OK", "OK" and reboot the system.

Most of these checks are also performed in `VSSUI.dll`, which is the DLL that hosts the UI for the shadow copies tab in Windows Server, in the `IsValidRevertVolume` function. Now, with the page file disabled, you'r ready to boot into the Windows Server environment.

A quick note before that: you can also edit the `VSSVC.exe` and `VSSUI.dll` binaries and nullify those checks. Does it work then? Can it work from a live booted OS restoring the system drive itself? I don't know, I haven't tried it, since I was doing all of this on my main workstation which I was trying to revert, but hopefully one day I play more with this under a VM and see what happens when doing this.

Now that you have the Windows Server environemnt, boot into it. When you get to the desktop, open File Explorer and identify your main OS system drive. Right click on it and choose "Configure Shadow Copies".

In there, select your drive in the list and identify the snapshot you want to revert to, then click the "Revert" button. A pop-up window will ask you to confirm the operation. Check the box, then click to proceed. After a brief freeze of the interface, the restore operation will commence. It shouldn't take long - it took around 20 minutes when I restored a snapshot on my NVMe drive filled with around 400GB, although it of course varies based on the hardware that you have.

![Restoring a shadow copy](https://user-images.githubusercontent.com/6503598/187089970-1520f5d5-448d-472a-b2f6-5ee4fc0a3053.png)

You can also monitor the progress in the command line using this:

```
vssadmin Query Reverts
```

### Conclusion

Shadow Copies are a VERY useful feature. I am actually impressed they aren't much more popularized. They indeed may require quite a bit of space, depending on your configured options, but overall, the piece of mind and convenience they bring make them worth using despite the aritifical roadblocks imposed by Microsoft. I hope the functionality will continue to live on in future Windows versions, considering how it lived and did its job just fine for the past 20 years, despite alarming signs from Microsoft in their latest OS versions. 

Also, keep in mind that these are not a replacement for a backup system! They are though a complement to your computing habits, and came in handy to me: I researched this article while trying to revert back to a config of my system before I messed up with it trying various programs and hacks to get my WhatsApp messages transfered from my iPhone to Android/Windows Subsystem for Android, in order to archive them and free up the storage on my phone (they take quite a bit of space). Maybe I will tell you about this in a future story. For now, this will do, peace!