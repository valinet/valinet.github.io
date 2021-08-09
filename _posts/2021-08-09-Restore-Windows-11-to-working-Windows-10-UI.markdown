---
layout: post
title:  "Restore Windows 11 to working Windows 10 UI"
date:   2021-08-09 00:00:00 +0000
categories: 
excerpt: "As you probably know already, Windows 11, among other changes, comes with a new and revamped taskbar. While it looks nice, it removes a ton of functionality users have come to depend on since it was introduced 20 or so years ago in Windows NT 4. Let's fix that!"
---

As you probably know already, Windows 11, among other changes, comes with a new and revamped taskbar. While it looks nice, it removes a ton of functionality users have come to depend on since it was introduced 20 or so years ago in Windows NT 4, like:

* the ability to ungroup taskbar buttons and show a button for each window
* ability to launch Task Manager from the right click menu
* ability to open files in respective applications by dragging the file onto the taskbar icon of an app
* ability to show labels for running tasks

I tried living with the new taskbar, but the productivity losses were much greater than what I gained. Also, I could go back to Windows 10, but I actually really enjoy the fact that Windows 11 is based on a much newer Windows build (22000 vs 19043) and that means it comes with a couple of improvements I really really like:

* improvements in how applications on multiple monitors are handled (i.e. when undocking a laptop, apps minimize, and then when connecting back, they are restored to where hey originally were)
* snap groups (i.e. hover over the maximize button of any window and a suggestion pops up with different layouts in which you can arrange the windows on your screen)
* support for hard disk mounting in WSL2
* support for GUI apps in WSL2 using WSLg
* support for GPU compute applications in WSL2
* Timeline is removed and in its place is the classical GNOME Overview-like interface which makes my WinOverview2 application unnecessary

I have been waiting for those improvements for ages in Windows 10 2004, 20H2, 21H1 and they never came. Thus, I have to stick to Windows 11 and try to make it feel right like at home. In this guide, I will show you how to get a fully working Windows 10 GUI in Windows 11, and also explain what I did to acomplish this so hopefully you are able to adapt this to your preference/setup.

### Uninstall CUs

I decided to uninstall all the cumulative updates and go back to build 22000.1, which is the RTM release. This also gave me back the Windows 10 Start menu which I much prefer over the lackluster menu Windows 11 proposes.

To uninstall all CUs, go to Control Panel - Programs and Features - View installed updates and look for each of the KBs under "Microsoft Windows" until you identify  the one corresponding to a particular CU. Uninstall that, reboot and  come back there and keep doing this until you get back to 22000.1 (check the build number in "winver"). Here is a list of KB numbers for each  build to date:

* KB5005188 - 22000.120
* KB5004300 - 22000.100
* KB5004252 - 22000.71
* KB5004745 - 22000.65
* KB5004564 - 22000.51

There is no need to uninstall other updates, for example, you can keep the servicing stack to the most recent version and only uninstall CUs this way.

### Enable the Windows 10 Start menu

Merge the following registry file to get back the Windows 10 Start menu:

```
Windows Registry Editor Version 5.00

[HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced]
"Start_ShowClassicMode"=dword:00000001
```

Do note that this works only up to and including build 22000.51.

### Enable the Windows 10 taskbar

Now, here comes a topic of lengthy discussions. Basically, there are a couple of methods to achieve this, each with a degree of missing functionality. In the end, I will describe what I went with instead.

##### 1. UndockingDisabled

Merge the following in the registry:

```
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Shell\Update\Packages]
"UndockingDisabled"=dword:00000001
```

Sign out, and when you sign back in, the old taskbar will be on screen. Unfortunately, this method has a couple of drawbacks:

* Search does not work
* Screen snip (Win + Shift + S) is broken
* Task view may crash when opened from the taskbar
* A delay may be added at log on before the desktop is shown
* If using a newer build and want the new Explorer features, this also disables them
* Win+X is not working

Considering the above, I don't recommend this method, but I use it as a base for my own method.

##### 2. File Explorer from Windows Server 2022

[People online suggest](https://forums.mydigitallife.net/threads/solved-build-22000-xx-old-taskbar-and-keep-explorer-new-features.83668/) replacing the built-in explorer.exe from C:\Windows with the File Explorer from Windows Server 2022. While this works, and indeed, it does not require `UndockingDisabled` to be set, it also has the following drawbacks:

* Modern applications (think Settings, Weather) do not show on taskbar, when opened
* Win+X is not working

Again, I do not recommend this either, especially since it can get pretty confusing when modern applications do not show in the taskbar.

##### 3. Using a different Taskbar.dll

As [many](https://twitter.com/thebookisclosed/status/1375006215189753860?ref_src=twsrc%5Etfw%7Ctwcamp%5Etweetembed%7Ctwterm%5E1375006215189753860%7Ctwgr%5E%7Ctwcon%5Es1_&ref_url=https%3A%2F%2Fmspoweruser.com%2Fmicrosoft-windows-10-taskbar-dll%2F) have noted online, Microsoft has been working on separating the taskbar code from the rest of the code in explorer.exe. Thus, they are migrating everything taskbar related to a new DLL called Taskbar.dll. In Windows 11, this DLL contains the code for the updated taskbar, while in some previous Windows 10 Insider builds, it contained the Windows 10 taskbar (sometimes partially working). In the mean time, they have also left the code for the Windows 10 taskbar in Explorer. When `UndockingDisabled` is not set, the code from the DLL is used and thus the new taskbar is drawn. However, it is possible to replace the DLL with one from previous Insider builds and have the Windows 10 taskbar instead of the new one. Specifically:

* the DLL from 21359 has the same bug as the Server 2022 explorer in that it does not show buttons for modern applications
* the DLL from 21364 shows modern apps, but does not display a Start button
* the DLL from 21390 is same as 21364, but it has a bug in which closing any window crashes Explorer

So, this solution means using a third party start menu like OpenShell which draws its own Start button. As I wanted to use the original Start menu and button from Windows 10, I felt the need to install some extra program is unnecessary, and moved on towards finding a better solution.

##### 4. Patching the current File Explorer

Using [Process Monitor](https://www.winhelponline.com/blog/process-monitor-track-events-generate-log-file/), I was able to determine a call stack for when Explorer.exe checks the `UndockingDisabled` registry entry when it starts up. It is pretty convoluted, but with this, I was able to identify the relevant section in the disassembly of explorer.exe where this check "begins". The pseudocode reads something along these lines:

```
 v32 = 0i64;
 // ...
 if ( winrt::impl::factory_cache_entry_v<winrt::WindowsUdk::Management::Deployment::PackageManager,winrt::WindowsUdk::Management::Deployment::IPackageManagerStatics> )
  {
    // ... not going here
  }
  else
  {
    // ...
    winrt::impl::factory_cache_entry<winrt::WindowsUdk::Management::Deployment::PackageManager,winrt::WindowsUdk::Management::Deployment::IPackageManagerStatics>::call<_lambda_720477a61d445079ad03f3ff7eed504d_ &>(
      v5,
      &v31,
      &v33);
    v17 = 9;
  }
  v6 = v17 | 6;
  v8 = (unsigned __int8)winrt::operator==(&v31, &v32) == 0;
  // ... some stuff here that does not alter "v8"
  if ( !v8 )
  {
    // ... some telemtry call
    v10 = TrayUI_CreateInstance((struct ITrayUIHost *)&unk_1403705B8, v9, &qword_1403708F0);
    // ... some other stuff
    goto end;
  }
  v10 = CTray::InitializeTrayUIComponent(v7);
  // ... some other stuff
  end:
```

It is not very friendly, but better than the raw assembly. This call appears on the call stack as accessing `UndockingDisabled`: `winrt::impl::factory_cache_entry<winrt::WindowsUdk::Management::Deployment::PackageManager,winrt::WindowsUdk::Management::Deployment::IPackageManagerStatics>::call<_lambda_720477a61d445079ad03f3ff7eed504d_ &>` (we will call this "the undocking lambda call").

The idea is what matters. in my opinion, the way it works is the following: 

* the undocking lambda call is made - when `UndockingDisabled` is set, the call does not create whatever thing it tries to create and return 0 in v31; this is subsequently compared to v32 (which is 0) in this line `v8 = (unsigned __int8)winrt::operator==(&v31, &v32) == 0;` and assigned to v8, which becomes `false` aka 0 (as v31 == v32 is true and true == 0 is false); then, when doing `if ( !v8 )`,  as v8 is false, we enter that branch and `TrayUI_CreateInstance` is called and if we further inspect that we will see that that is responsible for creating all the taskbar UI, including the *pearl* (which is the internal name for the Start button)
* when `UndockingDisabled`, the call succeeds and v8 ends up being 1 or `true` so that only the call to `CTray::InitializeTrayUIComponent(v7)` is made and whatever comes after it; this makes sense, as in the new taskbar, as described [here](https://rammichael.com/7-taskbar-tweaker-and-a-first-look-at-windows-11), the only "legacy" part that is not a XAML island is the tray UI, so probably that gets created here - the rest of the taskbar is somehow loaded from Taskbar.dll and created from whatever is found in there

So, my idea is to patch out that "if" so that the code flows as if `UndockingDisabled` is set, creating the legacy taskbar at all times. For this, the relevant section in assembly is:

```
test    sil, sil
jnz     loc_1401324F5
```

Or, in hex: `40 84 F6 0F 85 BC 59 0A 00`. The `test` is the first 3 bytes, the jump is all the rest.

The easy way is to patch out the jump and replace it with `nop`. I initially tested this by tweaking a bit my previous WinOverview2 project which injected explorer but for other means and it worked.

Now, the easy way would be to patch the 6 bytes in the executable, replace the original one in C:\Windows and call it a day. Unfortunately, this breaks its digital signature and Windows will refuse loading it. To get around this, I decided to binary patch it when Explorer starts up, as I found out about the [Taskman](https://www.geoffchappell.com/notes/windows/shell/explorer/taskman.htm) registry key when looking on Explorer's disassembly (specifically `wWinMain`). basically, it is a registry key that specifies an executable that Explorer should launch early in its startup process. We can specify a small executable we create that will spawn, patch Explorer's memory and then just close itself, and Explorer will then go on and do its duties and always spawn the old taskbar, as the if check will never happen. To minimize the racing condition, as soon as we are launched by explorer, we attach ourself as a debugger to it. This will have the effect of stopping all its threads, thus waiting for our small app to finish. After we patch the memory, we detach from Explorer as a debugger, and this will resume Explorer and we can safely exit. From my tests, the likelihood of Explorer already reaching the if check before our executable gets a chance to do anything is non-existent, but of course, your mileage may vary.

Thus, I created this example project which allows you to do just that. First, place it in a safe folder where you won't delete it from. Then, run it as admin once so it can set the `Taskman` registry entry. Then, just restart Explorer, sign out and back in or reboot the PC and you are done, the old taskbar will open instead of the new one.

[ExplorerPatcher](https://github.com/valinet/ExplorerPatcher) is available on my GitHub with precompiled binaries for build 22000.1. Compared to all the other methods, the only draw back of this is that Win+X still does not work (apparently, this has been removed altogether, but I will investigate this issue more, maybe we find some way to revive it). Other than that, search works, Modern apps show on the taskbar, everything else just works.

##### Enabling the system icons

When using all of these methods, the system icons (clock, network, battery, sound) will probably be hidden from the system tray. To enable them, run the following (Win + R):

```
shell:::{05d7b0f4-2121-4eff-bf6b-ed3f69b894d9}\SystemIcons
```

In the window that appears, enable anything to your liking. If using builds newer than 22000.1, the network and battery flyouts will probably be broken, so instead you can use the gear icon (control center) to connect to networks, and for battery info, you can use the [Battery Mode](https://en.bmode.tarcode.ru/) application.

If running build 22000.1, this can also be achieved via the Settings app, which is still the classic one from Windows 10 in this build. For this, just temporarily set `UndockingDisabled` and open the taskbar settings in the app, and instead of the page with settings for the new taskbar, the old page containing all the usual settings will be shown. Here you can also do the following:

* enable system icons
* configure the multi monitor behavior
* choose which folders should appear on Start above the power button
* hide/show the search and task view icons

### Conclusion

For the moment, that's all I have to say on this topic. I will update it subsequently if I find a way to make the old Win+X menu work in this setup, which would pretty much represent feature parity with Windows 10. Stay tuned!



