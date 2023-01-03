---
layout: post
title:  "Install Intel Unison on Windows 11 22000-based builds"
date:   2023-01-03 00:00:00 +0000
categories: 
excerpt: "From the ashes of Dell Mobile Connect, which was based on Screenovate technologies which were purchased by Intel last year, a similar app with a different name has arrived - this is a quick guide showing you how to install it on 21H2 (22000-based) Windows 11 builds, despite it only officially working on 22H2 (22621-based) builds and up."
---

From the ashes of Dell Mobile Connect, which was based on Screenovate technologies which were purchased by Intel last year, a similar app with a different name has arrived - this is a quick guide showing you how to install it on 21H2 (22000-based) Windows 11 builds, despite it only officially working on 22H2 (22621-based) builds and up.

The problem: if you attempt to install the app via the Microsoft Store on 22000-based builds, it will tell you that the app is not supported, suggesting you upgrade to 22H2. If you cannot or do not have time to do that, as a workaround, I suggest this:

1. Source the application. For this, I recommend using the [adguard store](https://store.rg-adguard.net/): from the drop down, choose "ProductId" and as product ID, use "9PP9GZM2GN26". Choose the retail channel, click the checkbox, wait a bit, and then download the `AppUp.IntelTechnologyMDE_2.2.2315.0_neutral_~_8j3eq9eme6ctt.msixbundle` file.

2. Using 7zip or something similar, extract the file as if it were a regular archive file. In the resulting folder, you'll get a `WebPhonePackaging_2.2.2315.0_x64.msix` file. Repeat the process, unzipping that as well.

3. In the resulting folder, identify and delete the `AppxMetadata` folder, and the `[Content_Types].xml`, `AppxBlockMap.xml`, and `AppxSignature.p7x` files.

4. Edit the `AppxManifest.xml` file. Identify the line which mentions OS build 22621 as minimum, and replace it to specify build 22000 instead as a minimum. So, turn line 11 from:

```
    <TargetDeviceFamily Name="Windows.Desktop" MinVersion="10.0.22621.0" MaxVersionTested="10.0.22621.0" />
```

To:

```
    <TargetDeviceFamily Name="Windows.Desktop" MinVersion="10.0.22000.0" MaxVersionTested="10.0.22000.0" />
```

5. Enable developer mode on your PC. For this, open Windows Settings -> Privacy & Security -> For developers -> Developer Mode -> ON. This will enable sideloading unsigned apps on your PC. The Intel Unison app's signature is broken since we edited the `AppxManifest.xml` file, and we removed it altogether by deleting the folder and mentioned files in step 3.

6. Open an administrative PowerShell window and browse to the folder containing the `AppxManifest.xml` file we have just edited in step 4. When in that directory, issue the following command in order to install the application on your system:

```
Add-AppxPackage -Register .\AppxManifest.xml
```

That's it! You should now be able to find the app in your Start menu, and after launhing it, it is business as usual - I was able to use it to sync my iPhone to my PC, make calls and so on, almost as if I were doing the same thing with a Mac. Certainly better than nothing, that's for sure - it makes owning a PC, yet Apple devices otherwise a less clunky experience.

The way it works is it sets up your PC as if it were a Bluetooth handsfree device - your phone 'sees' the PC as if it were your tpical car stereo. It's a fine way to do it, and I am amazed that it takes a multti-million dollar company to attempt this, instead of a hobby, open source project hosted on GitHub. The things is doable at that scale in my opinion; it could open the door to neat implementations, like having a Windows or Android based tablet as a car stereo, whihc of course comes with plenty advantages, like having a software package which you are more in control of, compared to what car manufacturers offer. But yeah, maybe another day... I am sure there are clunky Chinese implementations on Aliexpress already, but those aren't as polished as what a DIY solution could achieve.

If you get an error on step 6, you might need to install the other prerequiste packages for the app to work. These are offered on the adguard store as well: just manually download the relevant package (like `Microsoft.VCLibs.140.00.UWPDesktop_14.0.30704.0_x64__8wekyb3d8bbwe.appx` for example) and install it by double clicking it in File Explorer. If you do not have the App Installer app installed on your Windows install, you can alternatively open an administrative PowerShell window where you have downloaded the file and issue a command like this in order to install the package:

```
Add-AppxPackage .\Microsoft.VCLibs.140.00.UWPDesktop_14.0.30704.0_x64__8wekyb3d8bbwe.appx
```

Happy New Year, 2023!