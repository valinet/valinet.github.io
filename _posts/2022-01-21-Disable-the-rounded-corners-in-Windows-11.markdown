---
layout: post
title:  "Disable the rounded corners in Windows 11"
date:   2022-01-21 00:00:00 +0000
categories: 
excerpt: "This is another article related to ExplorerPatcher and Windows 11. This time, I wanted to take a more in-depth look at the changes in the compositor shipped with Windows 11 (the Desktop Window Manager). In this article, we learn how to disable the rounded corners in dwm.exe."
---

This is another article related to [ExplorerPatcher](https://github.com/valinet/ExplorerPatcher) and Windows 11. This time, I wanted to take a more in-depth look at the changes in the compositor shipped with Windows 11 (the "Desktop Window Manager").

Quite a few things have changed regarding it compared to Windows 10, like new window animations when maximizing and restoring down a window, while some things have regressed: Aero Peek is more broken than ever, basically a barely working artifact that is due for removal, apparently. Some things are the same, like the bug where clicks (on the very top of the title bar, or on the corner of the close button) on a maximized System-enhanced scaled foreground window displayed on a monitor with 150% and 3840x2160 resolution go to the window behind it, potentially accidentally closing it.

There is a rather visible new addition though: the window corners are now rounded. The whole design aesthetic for Windows 11 proposes rounded things everywhere. Personally I like them, but a lot of people do not, and for good reason: it would have been logical for Microsoft to offer at least a hidden option to enable the legacy behavior from Windows 10, where the corners are sharp, 90-degree angle. Unfortunately, they did not, so we are once again left to scramble through their executables and see what we can find.

The Desktop Window Manager is powered by many libraries; the one that apparently is tasked with doing the actual drawing on the screen surface is called `uDWM.dll`. I am a bit familiar with it from my work on [WinCenterTitle](https://github.com/valinet/WinCenterTitle), a program which let you center the text displayed in the non-client title bar of windows, as in Windows 8 and 8.1. Take notice, I said non-client: the compositor is only responsible for drawing the title bar of windows that do not elect to do it themselves (and inform it so). More and more applications are moving to custom-drawn title bars (or how the GNU/Linux/GNOME user land calls them, "client-side decorations"); whether that's a good or a bad thing, that's a subject of much debate. Thus, the effect of the patch becomes less and less impressive and consistent, as some applications will have their text centered while a whole host of others won't. The operating system does not provide a mechanism to inform applications how the text should be drawn: it is generally implied that the text is left-aligned, while Windows 8 and 8.1 should be treated as the exception and the title be custom-drawn centered there. Again, very poor design from Microsoft, if you ask me.

Okay, so let's start with `uDWM.dll`. If you look through the method list for it, one can quickly find a very interesting function: `CTopLevelWindow::GetEffectiveCornerStyle`. Here's its pseudocode:

```cpp
__int64 __fastcall CTopLevelWindow::GetEffectiveCornerStyle(__int64 a1)
{
  unsigned int v2; // ecx
  int v3; // ebx
  char Variant; // al

  if ( *((_BYTE *)CDesktopManager::s_pDesktopManagerInstance + 27)
    && !*((_BYTE *)CDesktopManager::s_pDesktopManagerInstance + 28)
    || *((int *)CDesktopManager::s_pDesktopManagerInstance + 8) >= 2 )
  {
    return 1;
  }
  else
  {
    v2 = *(_DWORD *)(*(_QWORD *)(a1 + 752) + 184i64);
    if ( !v2 )
    {
      v3 = *(_DWORD *)(a1 + 608);
      if ( (v3 & 2) != 0 )
      {
        return 3;
      }
      else
      {
        if ( (unsigned __int8)IsOpenThemeDataPresent() && (v3 & 6) != 0 )
          return 2;

        Variant = wil::details::FeatureImpl<__WilFeatureTraits_Feature_VTFrame>::__private_GetVariant(&`wil::Feature<__WilFeatureTraits_Feature_VTFrame>::GetImpl'::`2'::impl);
        v2 = 1;
        if ( Variant == 1 )
          return 2;
      }
    }
  }

  return v2;
}
```

Let's analyze it a bit. For once, we can determine which branch in that main `if` is taken at run time on a system where rounded corners work in Windows 11 (more on that a bit later), and also determine the actual return value.

Because we need to attach a debugger to `dwm.exe`, things are a bit more complicated: you have to debug it on a virtual machine (or on a remote system in general). You can't debug it on your development box as breaking into `dwm.exe` will prevent it from drawing updates to the screen, and thus you won't be able to operate the computer.

As usual, I use WinDbg. Start it on the remote computer. Now, there are 2 methods to go on:

* Attach to `dwm.exe`. When it breaks, the output will freeze, but the command text box will be focused. You can type in there `.server tcp:port=5005` followed by `g`. That will have the debugger listen on port 5005 and then continue executing `dwm.exe`. The output window will display a connection string that you can use to connect to this instance from the development box.
* Attach to `winlogon.exe` - this is the process that spawns new `dwm.exe` instances in case it crashes. After attaching, use `.childdbg 1` to have it break and also attach to child processes spawned by it. Press `g` to continue, and then kill `dwm.exe`. WinDbg will break and from there you can interact with the newly spawned `dwm.exe`.

Okay, so inspecting this function at runtime, we can see that it returns a `2`. If we statically patch it to return a `0` or a `1`, window corners will draw not rounded. So, that would be it, right? Actually, this patch was already implemented in my previous [Win11DisableOrRestoreRoundedCorners](https://github.com/valinet/Win11DisableRoundedCorners) utility, so why all the fuss with this article (I mean, it was quite popular, to the point that it got featured in a LinusTechTips video: https://www.youtube.com/watch?v=GZPRrYLGrhI&t=461s. Well, I have 2 more goals for this:

* Get rid of the dependency on symbol data
* Fix the context menus not having a proper shadows applied when using this mode

For the first bullet point, let's look at this part of the `if` statement: `*((_BYTE*)CDesktopManager::s_pDesktopManagerInstance + 27) && !*((_BYTE *)CDesktopManager::s_pDesktopManagerInstance + 28)`. At runtime, we see that the first expression is a `false`, while the `true`. So, we would need to make `*((_BYTE*)CDesktopManager::s_pDesktopManagerInstance + 27)` behave as if it were `true`, for example. Looking on the opcodes, we can see that `80 78 1B 00` (which coresponds to `cmp  byte ptr [rax+1Bh], 0`) is unique through the entire program. So we have an easy pattern match, right, just modify the comparison so that it behaves like a `jz` on the next instruction instead of `jnz loc_18007774E`, or modify the jump. Easy, but there's still bullet point 2. Also, what does this `if` statement really check? Maybe it hides some registry setting which we could use to enable this easily without patching the executable...? let's break down the `if` statement in parts:

* `!*((_BYTE *)CDesktopManager::s_pDesktopManagerInstance + 28)`

  Naturally, I have started looking in the constructor for `CDesktopManager` (which is an object but it is used as if it is a singleton throughout the entire program, it's only instantiated once, and its reference kept in the `s_pDesktopManagerInstance` global variable).

  `CDesktopManager::CDesktopManager` just zeroizes fields and fills in the virtual function tables properly, but let's see where it is called from: `CDesktopManager::Create` (which in turn is called by `DwmClientStartup`, which is exported by ordinal 101 and looks like the entry point of this library). After the constructor, a call to `CDesktopManager::Initialize` is made.

  Right in the beginning of that, some registry calls are performed. What's interesting is that one of them writes to `*((_BYTE *)this + 28) = 1;`. Bingo, so DWORD `ForceEffectMode` in `HKLM\Software\Microsoft\Windows\Dwm` sets `*((_BYTE *)CDesktopManager::s_pDesktopManagerInstance + 28)`. So the condition is `true` when the registry value is NOT set to `2`.

* `*((_BYTE *)CDesktopManager::s_pDesktopManagerInstance + 27)`

  This one was a bit more complicated. I now know it's written in `CDesktopManager::CreateMonitorRenderTargetsLegacy` by `*((_BYTE *)this + 27) = IsWarpAdapterLuid;`. `IsWarpAdapterLuid` is the value returned by `CDWMDXGIEnumeration::IsWarpAdapterLuid`.

  But first, how do I know this was written here? Well, as Cheat Engine, WinDbg also has a very nice feature where it can break when a memory location is accessed: `ba w 1 address` (w is for write-access, 1 is for 1-byte length). So, just break at some early point where you have access to `CDesktopManager::s_pDesktopManagerInstance` and then set a breakpoint on access for writes to that `+ 27` and that's it.

  Okay, so what is `CDWMDXGIEnumeration::IsWarpAdapterLuid`. From its body (and name), we see that it tries to determine whether the graphics adapter (used) is the WARP (software rendered) adapter. This is plenty obvious as well once we take a look at this `if` statement specifically: `if ( a2 == *(_QWORD *)(v5 + 336) && *(_DWORD *)(v5 + 296) == 5140 && *(_DWORD *)(v5 + 300) == 140 )` - `5140` is `0x1414` in hex, and `140` is `0x8c`. According to [this](https://docs.microsoft.com/en-us/windows/win32/direct3ddxgi/d3d10-graphics-programming-guide-dxgi#new-info-about-enumerating-adapters-for-windows-8), those IDs corespond to the "Microsoft Basic Render Driver", which is basically the software-based graphics adapter that is used as a fallback when graphics drivers for the real adapter are not installed etc. Why is this important? Well, we can observe that on such a setup, the rounded corners are disabled. So, it has to be that in such scenarios,`*((_BYTE *)CDesktopManager::s_pDesktopManagerInstance + 27)` is set to `true`, as the adapter used was the software one, and then ``!*((_BYTE *)CDesktopManager::s_pDesktopManagerInstance + 28)` is still 1 so the main branch is taken, the function returns `1` and rounded corners are disabled. Well, that means we are onto something: if we somehow make `!*((_BYTE *)CDesktopManager::s_pDesktopManagerInstance + 28)` return `false`, we can enable rounded corners when the software display adapter is used. But that's pretty easy: all we have to do is to set `ForceEffectMode` in the registry to `2` and test it out:

  ![image](https://user-images.githubusercontent.com/6503598/150562231-d0b71810-e7f8-461c-b8e6-c2710153475d.png)

  So, the opposite of what we want works (including context menus). Great start =))

* `*((int *)CDesktopManager::s_pDesktopManagerInstance + 8) >= 2`

  Using a similar break-on-access trick as described above, we can see that this is set in `CDesktopManager::UpdateRemotingMode` in possibly a couple of places. But that happens only when `GetSystemMetrics(4096)` is `true`, otherwise it is set in the end of the function to `0`: `*((_DWORD *)this + 8) = 0;`. So what does `GetSystemMetrics(4096)` do? Well, that's sparsely documented on various forums on the Internet: it check whether the current session is a remote desktop session. From my impression, what this does is disable rounded corners under certain remote desktop scenarios. So yeah, we won't bother anymore with that, it's pretty useless for this experiment. I mean, it would actually be if we decided to inject the remote process, as we could IAT patch the call to `GetSystemMetrics` and return `true` when we get a `4096` so that we enter the `if` and from there IAT patch away the rest until we make sure we leave some number at that memory address.

Yeah, so the conclusion is: rounded corners are disabled when the software display adapter is used, or when connected via certain remote sessions. Also, there are a couple of avenues. At this point, my idea was simple: have the `CTopLevelWindow::GetEffectiveCornerStyle` function somehow return `1` via the methods described above.

Well, I did that, but it has at least one nasty effect: context menus are drawn without a shadow. It kind of makes sense, if we look at where the function is called from at runtime: `CTopLevelWindow::UpdateWindowVisuals`. In there, of interest is a `float` that is initialized to `0.0` and then assigned *some* value when the corners are determined to be rounded - probably the radius of the curve. When corners are not touched, the number stays at `0.0`. If we look further down, there is an `if` check: `if ( (v10 & 0x20) == 0 && (IsOpenThemeDataPresent() && (v10 & 6) != 0 || v4 > 0.0) )`. I tired messing with it in all the ways, in combination with patching `CTopLevelWindow::GetEffectiveCornerStyle` as well - I always broke some scenario: I had windows without shadows, windows that are drawn square cornered even when rounded corners are enabled (tooltips, for example) display visual glitches, the mouse display visual glitches (a transparent box behind it when clicked). I was stuck...

Then, after playing with it a bit more, I cam around with some other idea: how would we go about modifying the least amount of code and logic and still achieve what we want? Well, since the radius seems to be a `float`, what if instead of `0.0`, which would give us 90-degree angles, we'd make it `0.00001` let's say. That so small it would look like a square on the screen. Also, keep in mind that the thing is rasterized in the end and you have only a handful of pixels, so a very small float that's not zero is basically zero.

Okay, so let's inspect what happens with the value we get from `CTopLevelWindow::GetEffectiveCornerStyle`:

```cpp
 {  
    EffectiveCornerStyle = CTopLevelWindow::GetEffectiveCornerStyle((__int64)this);
    if ( EffectiveCornerStyle == 2 )
    {
LABEL_8:
      wil::details::FeatureImpl<__WilFeatureTraits_Feature_VTFrame>::GetCachedVariantState(
        (volatile signed __int64 *)&`wil::Feature<__WilFeatureTraits_Feature_VTFrame>::GetImpl'::`2'::impl,
        (__int64)&v118);
      v4 = (float)v119;
      goto LABEL_9;
    }

    if ( EffectiveCornerStyle != 3 )
    {
      if ( EffectiveCornerStyle != 4 )
        goto LABEL_9;

      goto LABEL_8;
    }

    wil::details::FeatureImpl<__WilFeatureTraits_Feature_VTFrame>::GetCachedVariantState(
      (volatile signed __int64 *)&`wil::Feature<__WilFeatureTraits_Feature_VTFrame>::GetImpl'::`2'::impl,
      (__int64)&v118);
    v4 = (float)v119 * 0.5;
  }

LABEL_9:
```

Actually, the pseudocode this time is pretty bad. The raw diassembly is much better:

```
.text:0000000180029B3D                 call    ?GetEffectiveCornerStyle@CTopLevelWindow@@AEAA?AW4CORNER_STYLE@@XZ ; CTopLevelWindow::GetEffectiveCornerStyle(void)
.text:0000000180029B42                 mov     ecx, eax
.text:0000000180029B44                 sub     ecx, 2
.text:0000000180029B47                 jz      short loc_180029B57
.text:0000000180029B49                 sub     ecx, r15d
.text:0000000180029B4C                 jz      loc_180029C6A
.text:0000000180029B52                 cmp     ecx, r15d
.text:0000000180029B55                 jnz     short loc_180029B74
.text:0000000180029B57
.text:0000000180029B57 loc_180029B57:                          ; CODE XREF: CTopLevelWindow::UpdateWindowVisuals(void)+97↑j
.text:0000000180029B57                 lea     rdx, [rsp+1F0h+var_190]
.text:0000000180029B5C                 lea     rcx, ?impl@?1??GetImpl@?$Feature@U__WilFeatureTraits_Feature_VTFrame@@@wil@@CAAEAV?$FeatureImpl@U__WilFeatureTraits_Feature_VTFrame@@@details@3@XZ@4V453@A ; wil::details::FeatureImpl<__WilFeatureTraits_Feature_VTFrame> `wil::Feature<__WilFeatureTraits_Feature_VTFrame>::GetImpl(void)'::`2'::impl
.text:0000000180029B63                 call    ?GetCachedVariantState@?$FeatureImpl@U__WilFeatureTraits_Feature_VTFrame@@@details@wil@@AEAA?ATwil_details_FeatureStateCache@@XZ ; wil::details::FeatureImpl<__WilFeatureTraits_Feature_VTFrame>::GetCachedVariantState(void)
.text:0000000180029B68                 mov     eax, [rsp+1F0h+var_18C]
.text:0000000180029B6C                 xorps   xmm6, xmm6
.text:0000000180029B6F                 cvtsi2ss xmm6, rax
```

So, if the corner style is `2`, we jump to that place where we make the weird `GetCachedVariantState` call. Then, we move some value from the stack in `rax` and from there convert it to a single-precision floating point number.

At runtime, the value we obtain is `0x8`. It's pretty weird... I investigated with different values, it doesn't seem to actually hold a meaningful IEE754 single-precision floating point value, or maybe it just didn't tick for me how to work with it. Anyway, by experimenting, I saw that a `0x0` there is essentially `CTopLevelWindow::GetEffectiveCornerStyle` returning `1`, while if we set to `0x1`... boom, we get the nice 90-degree corners from Windows 10, complete with the context menu working.

Okay, so it seems the way to go is to make the radius as small as to be basically square nut not mathematically square, so `dwm.exe` would still think it works with rounded corners.

How do we patch? Well, if we look at the disassembly, we see that `xorps  xmm6, xmm6; cvtsi2ss  xmm6, rax` only appears in constructs specific to a check similar to the one showed here after getting the corner style. So it's actually safe to patch based on this pattern. But how? Well, again, if we look at all the `4` matches, we see they are all preceded by a `mov` instruction that fetches the value that will be ultimately converted to a `float`. So we can overwrite that safely, and in all places is even better, it is as if it would have read that value. So, how long is the `mov`? Well, 4 bytes. We need to write a 1 there, so `mov eax, 1` which is `b8 01 00 00 00` which is... 5 bytes long :(... CISC baby, what do we do now? Well, we take advantage of the fact that `x86` has a billion instructions and opcodes, so we can trick it in 4 bytes like so: `xor eax, eax; inc eax` - `31 c0 ff c0`. Oh yeah, good old `inc`.

So the patch is simple. Find this pattern `0x0F, 0x57, 0xF6, 0xF3, 0x48, 0x0F` and replace the preceding 4 bytes with `0x31, 0xC0, 0xFF, 0xC0`.

Lastly, how do we patch this time? The problem with `dwm.exe` is that it runs either as SYSTEM account, either under some obscure service accounts (`DWM-1`, `DWM-2` etc). We cannot, obviously, inject it from a process running with standard rights. Not even from an administrator one. We have to inject it from a process running as SYSTEM, only though there does it work.

Naturally, the way to go for this is to create a service, as that always runs as SYSTEM. So, we create a service that enumerates the Desktop Window Manager processes and patches 4 bytes at various locations. Since we only do that, no need to run remote code, we can forego injecting in `dwm.exe` and instead use `ReadProcessMemory` and `WriteProcessMemory` to alter its memory.

An example implementation is [here (ep_dwm)](https://github.com/valinet/ep_dwm). it can be compiled as a library and then called from your own application, for example.

![image](https://user-images.githubusercontent.com/6503598/150561707-3c6eeae2-298a-4512-8a35-9de33d49b888.png)

That's it! The functionality has been incorporated in the latest ExplorerPatcher, version 22000.434.41.10. Hopefully it will serve you well.
