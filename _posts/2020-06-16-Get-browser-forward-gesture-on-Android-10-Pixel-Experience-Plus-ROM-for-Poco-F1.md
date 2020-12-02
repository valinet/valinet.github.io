---
layout: post
title:  "Get browser forward gesture on Android 10 Pixel Experience Plus ROM for Poco F1"
date:   2020-06-16 00:00:00 +0000
categories: 
excerpt: Android 10 includes a new system of gesture navigation that mimicks the iPhone. But contrary to the iPhone, swyping from the right edge of the screen always goes back one screen, whereas in the browser it would be more useful to navigate forward. Read on to find out how to tweak this behavior, by binary patching Android system files.
---

Works in Chrome.

(This technique possibly works for other models as well, but you need to understand it and adapt it for your use case; it is copy-paste for Poco F1 only)

Disclaimer: my code makes the "long swipe" gesture unavailable, as I was not using it, but with patience you can come up with a better patch that may retain that functionality as well. This is more for me to remember how I did it and for educational purposes (finding pertinent and updated Android tutorials on stuff like this is pretty hard).

I have done the process on Windows with the help of WSL2, on Linux it should work 100%, maybe with other utilities.

1. Root phone using Magisk (<https://github.com/topjohnwu/Magisk/releases>) - excellent tool btw.
2. Extract SystemUI.apk from phone (mine was located at /system/product/priv-app/SystemUI/SystemUI.apk).
3. Open SystemUI.apk with 7-zip and extract "classes.dex".
4. Open file with hex editor (I used HxD) and search for pattern:
   "6E10D1950300".
5. Replace from pattern start with:
   "6E10D19503000A0355E580517C55950703050000124638070B0013077D007030CB921E077030CB922E072809380308007030CB921E067030CB922E063903040038010300"
6. Save file and fix checksum using dexRepair (<https://github.com/anestisb/dexRepair>):
   ./dexRepair -I /mnt/c/Users/Valentin/Downloads/classes.dex
7. Rename generated file "classes.dex_repaired.dex" to "classes.dex" and add back to SystemUI.apk.
8. Easiest way is to take an existing light Magisk module and put the apk in its system folder in a path like: "system/product/priv-app/SystemUI". I personally chose to modify the "Oxygen OS gesture bar for AOSP" as I use that module as well on my phone: <https://github.com/DanGLES3/Oxygen-OS-gesture-bar-for-AOSP>
9. Install module with Magisk and you are done.

### How it works

I learnt all of this from scratch, so maybe I am not 100% accurate in all places, but I get the big picture.

DEX files represent Android Runtime/Dalvik Virtual Machine "object" files, in that they contain an OS specific byte code that will be translated just in time to Java bytecode and then to architecture dependent assembly. Basically, dex files are the binaries of the Android world while APKs are just containers for them (an APK is basically a fancy zip file).

DEX byte code is different from Java byte codes, but fortunately I found a great resource listing all the opcodes I encountered when I took a look on this: <http://pallergabor.uw.hu/androidblog/dalvik_opcodes.html> - thank you very much.

With the code eventually being Java, we can fortunately take a look at the source code of the APK as well. For this, I used the online APK decompiler so I did not bother too much: <http://www.javadecompilers.com/apk>. The file we are interested in is located in: "sources/com/android/systemui/statusbar/phone/EdgeBackGestureHandler.java". I found this out by looking on the Gerrit of the Pixel Experience project and found the commit (<https://gerrit.pixelexperience.org/c/frameworks_base/+/3024/2/packages/SystemUI/src/com/android/systemui/statusbar/phone/EdgeBackGestureHandler.java#180>) where they added the settings to choose the height of the left swipe gesture zone (I initially wanted to disable Android's built-in gestures and use Fluid Navigation Gestures to replicate them, but I preferred this solution because it means less overhead and one less daemon running in the background).

From the source, the method we are interested in is: "onMotionEvent". I decided to turn the relevant part from:

```java
//...
boolean shouldTriggerBack = this.mEdgePanel.shouldTriggerBack();
boolean shouldTriggerLongSwipe = this.mEdgePanel.shouldTriggerLongSwipe();
int i = 4;
if (shouldTriggerLongSwipe) {
    sendEvent(0, 4, 128);
    sendEvent(1, 4, 128);
} else if (shouldTriggerBack) {
    sendEvent(0, 4);
    sendEvent(1, 4);
}
if (shouldTriggerBack || shouldTriggerLongSwipe) {
    z = true;
}
OverviewProxyService overviewProxyService = this.mOverviewProxyService;
//...
```

Into this (I show what I ended up by decompiling my manually patched APK, which is the idea I was heading towards):

```java
// ...
boolean shouldTriggerBack = this.mEdgePanel.shouldTriggerBack();
int i = 4;
if (shouldTriggerBack && (~(this.mIsOnLeftEdge ? 1 : 0)) != false) {
    sendEvent(0, 125);
    sendEvent(1, 125);
} else if (shouldTriggerBack) {
    sendEvent(0, 4);
    sendEvent(1, 4);
}
if (shouldTriggerBack || 0 != 0) {
    z = true;
}
OverviewProxyService overviewProxyService = this.mOverviewProxyService;
// ...
```

It looks bad because the code is hacked, not compiler generated, but mathematically it makes sense and achieves the flow that I want so it is good enough. I tried to modify as few things as possible, because I did not want to mess with mistaking recalculating jump offsets etc. (I was in a bit of a rush, did not have much time), and also, while doing these kinds of mods, resort to chainging as few as possible because it is very easy to introduce bugs which can have great consequences. For references, here is the high level code I intended to get to:

```java
// ...
boolean shouldTriggerBack = this.mEdgePanel.shouldTriggerBack();
int i = 4;
if (shouldTriggerBack) {
    if (this.mIsOnLeftEdge) {
    	sendEvent(0, 4);
	    sendEvent(1, 4);
    } else {
	    sendEvent(0, 125);
    	sendEvent(1, 125);
    }
}
if (shouldTriggerBack) {
    z = true;
}
OverviewProxyService overviewProxyService = this.mOverviewProxyService;
// ...
```

"sendEvent" sends the specified key as if it were pressed from the keyboard. The values are taken from: <https://developer.android.com/reference/android/view/KeyEvent>. 125 is for *KEYCODE_FORWARD*, which is the same as *VK_BROWSER_FORWARD* in Win32 for example, the "web browser forward" button some mice and keyboard have.

Now, you have to find where the first portion of code is located in the dex file. The way I did it was to dump the opcodes to a file and search through it. For dumping Android opcodes, there is a pretty good utility called dexdump in the Android SDK (yes, you need to download that as well, you can get it by downloading Android Studio from Google: <https://developer.android.com/studio#downloads>).

On my install, it was located at: "Sdk\build-tools\30.0.0". To dump the opcodes to a file, run something like:

```
dexdump -o SystemUIDump.txt -d classes.dex
```

Beware that the text file is huge (aprox. 100MB). Now, the best strategy is to search for what you believe is the thing that is less likely to be repeated for a lot of times and get from there, by observing how opcode patterns relate to the source code you got above. Fortunately, the dump is annotated so it is not hard to find what we are looking for. The method we are looking for, "onMotionEvent", is located around line 1038684, and the code portion we are modifying is at line 1038829. The dump looks like this (up to "z = true" above):

```
...
3292de: 54e3 7051                              |00ed: iget-object v3, v14, Lcom/android/systemui/statusbar/phone/EdgeBackGestureHandler;.mEdgePanel:Lcom/android/systemui/statusbar/phone/NavigationBarEdgePanel; // field@5170
3292e2: 6e10 d195 0300                         |00ef: invoke-virtual {v3}, Lcom/android/systemui/statusbar/phone/NavigationBarEdgePanel;.shouldTriggerBack:()Z // method@95d1
3292e8: 0a03                                   |00f2: move-result v3
3292ea: 54e5 7051                              |00f3: iget-object v5, v14, Lcom/android/systemui/statusbar/phone/EdgeBackGestureHandler;.mEdgePanel:Lcom/android/systemui/statusbar/phone/NavigationBarEdgePanel; // field@5170
3292ee: 6e10 d295 0500                         |00f5: invoke-virtual {v5}, Lcom/android/systemui/statusbar/phone/NavigationBarEdgePanel;.shouldTriggerLongSwipe:()Z // method@95d2
3292f4: 0a05                                   |00f8: move-result v5
3292f6: 1246                                   |00f9: const/4 v6, #int 4 // #4
3292f8: 3805 0b00                              |00fa: if-eqz v5, 0105 // +000b
3292fc: 1307 8000                              |00fc: const/16 v7, #int 128 // #80
329300: 7040 cc92 1e76                         |00fe: invoke-direct {v14, v1, v6, v7}, Lcom/android/systemui/statusbar/phone/EdgeBackGestureHandler;.sendEvent:(III)V // method@92cc
329306: 7040 cc92 2e76                         |0101: invoke-direct {v14, v2, v6, v7}, Lcom/android/systemui/statusbar/phone/EdgeBackGestureHandler;.sendEvent:(III)V // method@92cc
32930c: 2809                                   |0104: goto 010d // +0009
32930e: 3803 0800                              |0105: if-eqz v3, 010d // +0008
329312: 7030 cb92 1e06                         |0107: invoke-direct {v14, v1, v6}, Lcom/android/systemui/statusbar/phone/EdgeBackGestureHandler;.sendEvent:(II)V // method@92cb
329318: 7030 cb92 2e06                         |010a: invoke-direct {v14, v2, v6}, Lcom/android/systemui/statusbar/phone/EdgeBackGestureHandler;.sendEvent:(II)V // method@92cb
32931e: 3903 0400                              |010d: if-nez v3, 0111 // +0004
329322: 3805 0300                              |010f: if-eqz v5, 0112 // +0003
...
```

I leave as an exercise to figure out how the opcodes relate to the Java code by researching using the resources above. The opcodes I have obtained follow this logic (I hope I haven't forgot to replicate back to the scratchpad the modifications I was doing in HxD, any discrepancies between this and the pattern at the beginning, follow that):

```
...
6e10d1950300	invoke-virtual {v3}, Lcom/android/systemui/statusbar/phone/NavigationBarEdgePanel;.shouldTriggerBack:()Z // method@95d1
0a03	move-result v3
55e58051	iget-boolean v5, v14, Lcom/android/systemui/statusbar/phone/EdgeBackGestureHandler;.mIsOnLeftEdge:Z // field@5180
7C5595070305	not-int v5, v5; and-int v7, v3, v5 ( v7 = v3 & v5)
0000	
1246	const/4 v6, #int 4 // #4
38070b00	if-eqz v7, 0105 // +000b
13077d00	const/16 v7, #int 125 // #7D
7030cb921e07	invoke-direct {v14, v1, v7}, Lcom/android/systemui/statusbar/phone/EdgeBackGestureHandler;.sendEvent:(II)V // method@92cb
7030cb922e07	invoke-direct {v14, v2, v7}, Lcom/android/systemui/statusbar/phone/EdgeBackGestureHandler;.sendEvent:(II)V // method@92cb
2809	goto 010d // +0009
38030800 	if-eqz v3, 010d // +0008
7030cb921e06	invoke-direct {v14, v1, v6}, Lcom/android/systemui/statusbar/phone/EdgeBackGestureHandler;.sendEvent:(II)V // method@92cb
7030cb922e06	invoke-direct {v14, v2, v6}, Lcom/android/systemui/statusbar/phone/EdgeBackGestureHandler;.sendEvent:(II)V // method@92cb
39030400	if-nez v3, 0111 // +0004
38010300	if-eqz v1, 0112 // +0003 // v1 holds 0, easier to not modify this and write that 0 != 0 non sense in order for expression to still hold, than to modify the entire opcode
...
```

That's it, enjoy this free world we live in, things could have been an iPhone (rip unc0ver for 13.5+, unfortunately).