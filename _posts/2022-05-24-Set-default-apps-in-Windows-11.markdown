---
layout: post
title:  "Set default apps in Windows 11 (restore the Windows 10 Settings app in Windows 11)"
date:   2022-05-24 00:00:00 +0000
categories: 
excerpt: "Today I spent time fixing yet another questionable decision of Microsoft, restoring the legacy Windows 10 Settings app in Windows 11 so that I could finally properly adjust app defaults (and change File History settings)."
---

I just got fed up with the new idiotic way in which Microsoft wants to force us to set defaults in Windows 11. Instead of the good old Settings - Apps - Default apps page from Windows 10 that easily let you assign an app for an entire category (which is pretty logical, for example, you could set the default music player in a pinch, which immediatly associated the app with mp3, wav etc files), you now have to go through each extension and assign it the app that you want, or go to the app and manually assign each extension to it. I mean, why?

Other than that, the Windows 11 Settings app is fine, it looks nice, but this omission is a huge downgrade. People have resorted to all sorts of hacks, and even big players like Firefox [have hacked their way into bypassing Microsoft's protections](https://www.reddit.com/r/firefox/comments/pnemdm/mozilla_has_defeated_microsofts_default_browser/) because the current way for setting defaults is just awful.

Well, here I am again, fixing Microsoft's OS. This time, I decided enough is enough and looked into a way to bring back the Settings app from Windows 10 which thankfully included the beloved "Default apps" page.

![image](https://user-images.githubusercontent.com/6503598/170130743-33352050-25fb-43f4-b0ab-4253a5f36d16.png)

### How to?

This time, you'll have to do things manually; I have yet to decide on a way to automate this, if any - suggestions are welcome in the comments section; should this be integrated in ExplorerPatcher, and if so, how to go around the technical details? Anyway, for the tutorial, read on.

You'll need the following things from build 22000.1, the single public build of Windows 11 that ever shipped with the legacy Settings app:

* the `C:\Windows\ImmersiveControlPanel` folder; this is the main folder where the UWP Settings app lives
* the `C:\Windows\SystemResources\Windows.UI.SettingsAppThreshold` folder; this is the folder that contains the resources used by the Settings app
* the `C:\Windows\System32\SettingsEnvironment.Desktop.dll` file

If you stop at only the 2 things above, the legacy Settings app will work, but none of the built-in pages will actually work. If you connect with a debugger to the running `SystemSettings.exe` process, some first-time exceptions will be thrown, and a message from the Windows Runtime along the lines of `WinRT information: Cannot find a resource with the given key: DefaultEntityItemSettingPageTemplate` will be mentioned. Needless to say, the debug info is totally worthless; only by looking on the modules list and testing file by file afterwards was I able to successfully determine what was still required to be brought over, i.e. this file.

So, on a live system:

* Take ownership of `C:\Windows\ImmersiveControlPanel` and rename it to `ImmersiveControlPanelO`, and paste the `C:\Windows\ImmersiveControlPanel` folder from 22000.1 in there.
* Again, Windows is bugged up and won't let you replace the `C:\Windows\SystemResources\Windows.UI.SettingsAppThreshold` folder, even after taking ownership of it. I don't really understand what is wrong with this OS, but the solution is to manually rename the 3 items inside (`pris` to `prisO`, `SystemSettings` to `SystemSettingsO` and `Windows.UI.SettingsAppThreshold.pri` to `Windows.UI.SettingsAppThreshold.priO`) and copy in there the corresponding 3 items from 22000.1.
* Instead of replacing the `SettingsEnvironment.Desktop.dll` in `C:\Windows\System32`, I recommend copying it from 22000.1 in `C:\Windows\ImmersiveControlPanel`. This DLL is loaded in a "classic" manner so to speak, thus the loader first looking in the applicaiton folder, and then in the system directories; thus, the DLL placed in `C:\Windows\ImmersiveControlPanel` can override the one in `C:\Windows\System32`.

That's it. If you've done everything correctly, open the Settings app using Start, for example - you should be greeted by the legacy Settings app from Windows 10:

![image](https://user-images.githubusercontent.com/6503598/170138015-0b01ec18-4661-42a5-99b0-fc24821b0d50.png)

To spare you having to spin up a VM and gather the required files, you can [download the following archive](https://github.com/valinet/ExplorerPatcher/files/8766287/LegacySettingsAppForWindows10Files.zip) which contains the 2 folders from 22000.1: `C:\Windows\ImmersiveControlPanel` (already contains the proper `SettingsEnvironment.Desktop.dll` file in it as well), and `C:\Windows\System32\SettingsEnvironment.Desktop.dll`. All you have to do is to place these instead of the built-in folders that you have, of course, taking backups of the originals.

Needless to say, this will probably be reverted by Windows Update, but nevertheless, it's a solution for at least doing some maintenance tasks that you can't really do with the new Settings app. As I said, we can discuss in the comments section about a proper way to automate this, if interested.

If you stumble upon any problem and would like to restore a clean slate, you can run the following command and restart the computer when it finishes:

```
sfc /scannow
```

### Bonus

Yeah, since we restored the full Windows 10 Settings app, you can now use the other UIs that were also forgotten by Microsoft and never replicated in the new Settings app. For example, I could finally properly adjust File History settings (Settings - Update & Security - Backup) which is another omission from the new app.

### Conclusion

What can I say, another stupid move by Microsoft. I am VERY glad that this hack works after all, but the fact that we needed it is just mind boggling. Hopefully, things will improve with the upcoming updates, but I honestly doubt it. Anyway, in the mean time, we have this.

Looking forward to hearing your thoughts in the comments below.