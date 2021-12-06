---
layout: post
title:  "Functional Windows 10 flyouts in Windows 11"
date:   2021-11-18 00:00:00 +0000
categories: 
excerpt: "One problem that was still unsolved was how to restore the functionality of the Windows 10 network, battery and language switcher flyouts in the taskbar. While volume and clock work well, and the language switcher is the one from Windows 11 at least, the network and battery ones just do not launch properly. So, what gives?"
---

Quite some time has passed since my last post here. And that's for no ordinary reason: the past months, I have been pretty busy with work, both at my workplace and while working on [ExplorerPatcher](https://github.com/valinet/ExplorerPatcher). So many have happened regarding that since I last talked about it here, that it's just simpler to check it out yourself, and look through the [CHANGELOG](https://github.com/valinet/ExplorerPatcher/blob/master/CHANGELOG.md) if you want, then for me to reiterate all the added funcionality. Basically, it has become a full fledged program in its own.

But today's topic is not really about ExplorerPatcher. Or rather it is, but not directly about it, but about how I enabled a certain functionality for it and how it can be achieved even without it and what are the limitations.

### Background

As you are probably aware, Windows 11 has recently launched, and with it, a new radical design change. To keep it short, there are a couple of utilities that let you restore the old Windows 10 UI, one of which being ExplorerPatcher (there is also [StartAllBack](https://www.startallback.com/), for example).

One problem that was still unsolved was how to restore the functionality of the Windows 10 network, battery and language switcher flyouts in the taskbar. While volume and clock work well, and the language switcher is the one from Windows 11 at least, the network and battery ones just do not launch properly. So, what gives?

### Investigation

I started my investigation... well, it took me the better part of a whole day to figure my way around how Windows does those things, but to spare you the time, let me tell you directly how it works.

First, most flyouts seem to be invoked by using some "experience manager" interface. Let's generically call that `IExperienceManager`. It derives from `IInspectable` and the next 2 methods are the aptly named `ShowFlyout` and `HideFlyout` methods.

These usually live in `twinui.dll`. At least the ones common to all types of Windows devices (for example, the battery, clock, sound one). By the way, sound is called "MtcUvc", whatever that means. The more specific ones live some places else, but are instanced by `twinui.dll`, so you usually can find the required interface IDs there. Also in there are the required names for these interfaces (dump all string containing `Windows.Internal.ShellExperience` from `twinui.dll` using `strings64.exe`  from Sysinternals, for example).

So, by looking on some disassembly in `twinui.dll`, I figured out that the way to invoke these flyouts is something like this:

```cpp
void InvokeFlyout(BOOL bAction, DWORD dwWhich)
{
    HRESULT hr = S_OK;
    IUnknown* pImmersiveShell = NULL;
    hr = CoCreateInstance(
        &CLSID_ImmersiveShell,
        NULL,
        CLSCTX_NO_CODE_DOWNLOAD | CLSCTX_LOCAL_SERVER,
        &IID_IServiceProvider,
        &pImmersiveShell
    );
    if (SUCCEEDED(hr))
    {
        IShellExperienceManagerFactory* pShellExperienceManagerFactory = NULL;
        IUnknown_QueryService(
            pImmersiveShell,
            &CLSID_ShellExperienceManagerFactory,
            &CLSID_ShellExperienceManagerFactory,
            &pShellExperienceManagerFactory
        );
        if (pShellExperienceManagerFactory)
        {
            HSTRING_HEADER hstringHeader;
            HSTRING hstring = NULL;
            WCHAR* pwszStr = NULL;
            switch (dwWhich)
            {
            case INVOKE_FLYOUT_NETWORK:
                pwszStr = L"Windows.Internal.ShellExperience.NetworkFlyout";
                break;
            case INVOKE_FLYOUT_CLOCK:
                pwszStr = L"Windows.Internal.ShellExperience.TrayClockFlyout";
                break;
            case INVOKE_FLYOUT_BATTERY:
                pwszStr = L"Windows.Internal.ShellExperience.TrayBatteryFlyout";
                break;
            case INVOKE_FLYOUT_SOUND:
                pwszStr = L"Windows.Internal.ShellExperience.MtcUvc";
                break;
            }
            hr = WindowsCreateStringReference(
                pwszStr,
                pwszStr ? wcslen(pwszStr) : 0,
                &hstringHeader,
                &hstring
            );
            if (hstring)
            {
                IUnknown* pIntf = NULL;
                pShellExperienceManagerFactory->lpVtbl->GetExperienceManager(
                    pShellExperienceManagerFactory,
                    hstring,
                    &pIntf
                );
                if (pIntf)
                {
                    IExperienceManager* pExperienceManager = NULL;
                    pIntf->lpVtbl->QueryInterface(
                        pIntf,
                        dwWhich == INVOKE_FLYOUT_NETWORK ? &IID_NetworkFlyoutExperienceManager :
                        (dwWhich == INVOKE_FLYOUT_CLOCK ? &IID_TrayClockFlyoutExperienceManager :
                            (dwWhich == INVOKE_FLYOUT_BATTERY ? &IID_TrayBatteryFlyoutExperienceManager :
                                (dwWhich == INVOKE_FLYOUT_SOUND ? &IID_TrayMtcUvcFlyoutExperienceManager : &IID_IUnknown))),
                        &pExperienceManager
                    );
                    if (pExperienceManager)
                    {
                        RECT rc;
                        SetRect(&rc, 0, 0, 0, 0);
                        if (bAction == INVOKE_FLYOUT_SHOW)
                        {
                            pExperienceManager->lpVtbl->ShowFlyout(pExperienceManager, &rc, NULL);
                        }
                        else if (bAction == INVOKE_FLYOUT_HIDE)
                        {
                            pExperienceManager->lpVtbl->HideFlyout(pExperienceManager);
                        }
                        pExperienceManager->lpVtbl->Release(pExperienceManager);
                    }

                }
                WindowsDeleteString(hstring);
            }
            pShellExperienceManagerFactory->lpVtbl->Release(pShellExperienceManagerFactory);
        }
        pImmersiveShell->lpVtbl->Release(pImmersiveShell);
    }
}
```

Full source code: [ImmersiveFlyouts.c](https://github.com/valinet/ExplorerPatcher/blob/master/ExplorerPatcher/ImmersiveFlyouts.c), [ImmersiveFlyouts.h](https://github.com/valinet/ExplorerPatcher/blob/master/ExplorerPatcher/ImmersiveFlyouts.h)

Okay, so with the above one can pretty much manually invoke the flyouts for any of the system icons shown in the notification area. Yet still, network and battery crash.

So, I attached with WinDbg to `explorer.exe` and clicked the network icon. Then, i saw it spits some error message:

```
(a0a0.6880): Windows Runtime Originate Error - code 40080201 (first chance)
(a0a0.6880): C++ EH exception - code e06d7363 (first chance)
(a0a0.6880): C++ EH exception - code e06d7363 (first chance)
(a0a0.6880): C++ EH exception - code e06d7363 (first chance)
(a0a0.6880): C++ EH exception - code e06d7363 (first chance)
pcshell\twinui\viewmanagerinterop\lib\windowmanagerbridge.cpp(216)\twinui.pcshell.dll!00007FFE213CA625: (caller: 00007FFE212B91D2) ReturnHr(11) tid(6880) 80070490 Element not found.
    Msg:[Platform::Exception^: Element not found.

pcshell\twinui\viewmanagerinterop\lib\windowmanagerbridge.cpp(174)\twinui.pcshell.dll!00007FFE212B94ED: (caller: 00007FFE212B91D2) Exception(7) tid(6880) 80070490 Element not found.
] 
shell\twinui\experiencemanagers\lib\networkexperiencemanager.cpp(107)\twinui.dll!00007FFE16C266DD: (caller: 00007FFE16C45B05) LogHr(2) tid(30d8) 80070578 Invalid window handle.
pcshell\twinui\viewmanagerinterop\lib\windowmanagerbridge.cpp(148)\twinui.pcshell.dll!00007FFE212B966C: (caller: 00007FFE212B91D2) LogHr(11) tid(84b0) 80070490 Element not found.
pcshell\twinui\viewmanagerinterop\lib\windowmanagerbridge.cpp(211)\twinui.pcshell.dll!00007FFE212B96A6: (caller: 00007FFE212B91D2) Exception(8) tid(84b0) 80070490 Element not found.
(a0a0.84b0): Windows Runtime Originate Error - code 40080201 (first chance)
(a0a0.84b0): C++ EH exception - code e06d7363 (first chance)
(a0a0.84b0): C++ EH exception - code e06d7363 (first chance)
(a0a0.84b0): C++ EH exception - code e06d7363 (first chance)
(a0a0.84b0): C++ EH exception - code e06d7363 (first chance)
pcshell\twinui\viewmanagerinterop\lib\windowmanagerbridge.cpp(216)\twinui.pcshell.dll!00007FFE213CA625: (caller: 00007FFE212B91D2) ReturnHr(12) tid(84b0) 80070490 Element not found.
    Msg:[Platform::Exception^: Element not found.

pcshell\twinui\viewmanagerinterop\lib\windowmanagerbridge.cpp(211)\twinui.pcshell.dll!00007FFE212B96A6: (caller: 00007FFE212B91D2) Exception(8) tid(84b0) 80070490 Element not found.
]
```

Okay... also, not much else can be done by attaching to `explorer.exe`, as the flyouts run out of process. After some trial and error, I figured out that the process they run into is `ShellExperienceHost.exe`. This just crashes when the 2 flyouts are invoked. So, let's attach to it. To have it spawn and be able to attach to it, besides `.child dbg 1` in the parent process, in this instance we can see that once a flyout is launched, the process remains resident in memory, and gets suspended. So, I opened the sound flyout, and then attached with WinDbg. Clicked network and then the error was trapped. The call stack looked like this (up to the first frame that did not look like some error handler):

```
[0x0]   KERNELBASE!RaiseFailFastException + 0x152   
[0x1]   Windows_UI_QuickActions!wil::details::WilDynamicLoadRaiseFailFastException + 0x53   
[0x2]   Windows_UI_QuickActions!wil::details::WilRaiseFailFastException + 0x22   
[0x3]   Windows_UI_QuickActions!wil::details::WilFailFast + 0xbc   
[0x4]   Windows_UI_QuickActions!wil::details::ReportFailure_NoReturn<3> + 0x29f   
[0x5]   Windows_UI_QuickActions!wil::details::ReportFailure_Base<3,0> + 0x30   
[0x6]   Windows_UI_QuickActions!wil::details::ReportFailure_Msg<3> + 0x67   
[0x7]   Windows_UI_QuickActions!wil::details::ReportFailure_HrMsg<3> + 0x6e   
[0x8]   Windows_UI_QuickActions!wil::details::in1diag3::_FailFast_HrMsg + 0x33   
[0x9]   Windows_UI_QuickActions!wil::details::in1diag3::FailFast_HrIfNullMsg<Windows::UI::Xaml::ResourceDictionary ^ __ptr64,0> + 0x5c   
[0xa]   Windows_UI_QuickActions!QuickActions::QuickActionTemplateSelector::[Windows::UI::Xaml::Controls::IDataTemplateSelectorOverrides]::SelectTemplateCore + 0x99a   
```

So, it looks like the error is in that `SelectTemplateCore`. Also, the Windows Runtime again spat out something in the debugger:

```
minkernel\mrt\mrm\mrmex\priautomerger.cpp(65)\MrmCoreR.dll!00007FFE3F2561BD: (caller: 00007FFE3F2676AD) LogHr(4) tid(3394) 80070002 The system cannot find the file specified.
ModLoad: 00007ffe`3cba0000 00007ffe`3cbd5000   C:\Windows\System32\Windows.Energy.dll
shellcommon\shell\windows.ui.shell\quickactions\quickactions\QuickActionTemplateSelector.cpp(84)\Windows.UI.QuickActions.dll!00007FFD8848A8DE: (caller: 00007FFD884954B7) FailFast(1) tid(3394) 80070490 Element not found.
    Msg:[QuickActionTemplateSelector missing a valid TemplateDictionary.] 
(8c30.3394): Security check failure or stack buffer overrun - code c0000409 (!!! second chance !!!)
Subcode: 0x7 FAST_FAIL_FATAL_APP_EXIT 
KERNELBASE!RaiseFailFastException+0x152:
00007ffe`57e01de2 0f1f440000      nop     dword ptr [rax+rax]
```

What's interesting in that is it coroborates with what we have got from `explorer.exe`: that `80070490` HRESULT is the same we got in Explorer. Okay, so it seems some "dictionary" is broken for some quick action toggle, apparently... I don't know exactly their internal structure, I am just guessing, of course.

Let's load `Windows.UI.QuickActions.dll` in IDA and take a look at it. You can find this in the same folder as `ShellExperienceHost.exe`: `C:\Windows\SystemApps\ShellExperienceHost_cw5n1h2txyewy`.

We find that function quickly, among the sea of symbols, especially if we search for that `QuickActionTemplateSelector missing a valid TemplateDictionary` error string.

That code area gets called if either branch of some `if` check eventually fails. So we're presumably on one of the branches. Without looking too much beside that, I thought: what if we would be on the other branch? A small note on that:

> This thought didn't really came that out of the blue. For the better part of a whole day, I started trying to find a way to do the same thing the lock screen (`LockApp.exe`) does: that can show the Windows 10 network flyout just fine. So I knew that has to be in some other condition, and I intially thought that maybe this `if` check here is the determinant for the lock screen's working condition versus it failing on the desktop. Again, just a wild guess.
>
> Also, since we are talking about that, as I learned when I studied the language switcher, there is some global flag in the network flyout implementation, let's called it generically, that determines what mode to show. The implementation mostly lives in `C:\Windows\ShellExperiences\NetworkUX.dll`. In there, look for a method called `NetworkUX::ViewContext::SetNetworkUXMode`. That's the single thing that sets a global variable that is used all around the place to determine the type of UX to show, called `s_networkUXMode`. 
>
> The desktop seems to set `s_networkUXMode` to 0. The lock screen sets that to 7 (also, it cannot be launched in desktop mode, it crashes for some other reason which needs to be investigated as well). There are also other interesting modes: the Windows 10 OOBE screen is 4, which looks quite funny when enabled instead of the regular one:
>
> ![image](https://user-images.githubusercontent.com/6503598/142466279-066793bf-4d81-486b-9115-ed4313b82bf1.png)
>
> And no, clicking Next does not advance you to the next in the "desktop OOBE" =)))
>
> The Windows 11 one is 5 if I remember correctly. Find out for yourself. The assembly instructions where that is set look like:
>
> ```asm
> .text:000000018006BC0C                 mov     Ns_networkUXMode, edi ; 
> .text:000000018006BC12                 mov     rcx, [rbp+var_8]
> .text:000000018006BC16                 xor     rcx, rsp        ; StackCookie
> .text:000000018006BC19                 call    __security_check_cookie
> ```
>
> So, there's plenty of space to write something like:
>
> ```asm
> mov     edi, 7
> mov     Ns_networkUXMode, edi ; 
> ```
>
> Just `nop` the stack protector check to gain the necessary space and make sure to adjust that relative `mov     Ns_networkUXMode, edi ;` if you shift it a few bytes down due to the `mov edi, 7`.
>

Back to the main story, so what if we would be on the other branch? What controls that `if` check? Scrolling a few lines above, we see this pseudocode:

```cpp
    {
      Init_thread_header(&dword_18057130C);
      if ( dword_18057130C == -1 )
      {
        byte_180571308 = FlightHelper::CalculateRemodelEnabled();
        Init_thread_footer(&dword_18057130C);
      }
    }

    if ( byte_180571308 )
    {
```

That `byte_180571308` is the `if` check we are talking about. So it seems that the branch we take ultimately gets determined by that `FlightHelper::CalculateRemodelEnabled();` call. Let's hop into that: it's a mostly useless call, as it does not seem to have any other return path other than a plain `return 1` at the end (maybe the stuff in there can hard fail, but it doesn't seem to be the case here, we seem to reach that `return 1` at the end). Okay, so according to this, `byte_180571308` is going to be a 1. So let's try setting it to 0.

Looking on the disassembly, the ending is something like:

```asm
.text:00000001800407C9                 mov     r8b, 3
.text:00000001800407CC                 mov     dl, 1
.text:00000001800407CE                 lea     rcx, ?impl@?1??GetImpl@?$Feature@U__WilFeatureTraits_Feature_TestNM@@@wil@@CAAEAV?$FeatureImpl@U__WilFeatureTraits_Feature_TestNM@@@details@3@XZ@4V453@A ; wil::details::FeatureImpl<__WilFeatureTraits_Feature_TestNM> `wil::Feature<__WilFeatureTraits_Feature_TestNM>::GetImpl(void)'::`2'::impl
.text:00000001800407D5                 call    ?ReportUsage@?$FeatureImpl@U__WilFeatureTraits_Feature_TestNM@@@details@wil@@QEAAX_NW4ReportingKind@3@_K@Z ; wil::details::FeatureImpl<__WilFeatureTraits_Feature_TestNM>::ReportUsage(bool,wil::ReportingKind,unsigned __int64)
.text:00000001800407DA                 mov     al, 1
.text:00000001800407DC                 jmp     short loc_1800407E0
.text:00000001800407DE ; ---------------------------------------------------------------------------
.text:00000001800407DE                 xor     al, al
.text:00000001800407E0
.text:00000001800407E0 loc_1800407E0:                          ; CODE XREF: FlightHelper__CalculateRemodelEnabled+F0↑j
.text:00000001800407E0                 add     rsp, 20h
.text:00000001800407E4                 pop     rdi
.text:00000001800407E5                 pop     rsi
.text:00000001800407E6                 pop     rbx
.text:00000001800407E7                 retn
```

Coresponding to pseudocode looking like this:

```asm
  LOBYTE(v4) = 3;
  LOBYTE(v3) = 1;
  wil::details::FeatureImpl<__WilFeatureTraits_Feature_TestNM>::ReportUsage(
    &`wil::Feature<__WilFeatureTraits_Feature_TestNM>::GetImpl'::`2'::impl,
    v3,
    v4);
  return 1;
```

But the disassembly is more interesting. Specifically, if you look at address `00000001800407DE`, you see that `xor al, al` that is bypassed altogether by the preceding unconditional jump to the instruction below it. That's great, we do not even have to insert too much stuff ourselves. That jump is 2 bytes: `EB 02`. Let's nop those, so replace with `90 90`.

Drop the file instead of the original `Windows.UI.QuickActions.dll` near `ShellExperienceHost.exe`, make sure to reload it if it's loaded in memory, click the network icon and...:

![image](https://user-images.githubusercontent.com/6503598/142405580-5db9e79a-deb4-4314-bd82-603090847e3c.png)

YEEEEEY!!! Call it sheer luck or whatever, but it works. The battery fix came for free as well, after this, as it seemed to suffer from the same thing:

![image-20211118174828628](https://user-images.githubusercontent.com/6503598/142454619-7954e217-3965-4796-9540-00ad58a9f149.png)

Okay, so it's only 2 bytes Microsoft had to fix to make this happen, but yet again I have to come and clean up the mess...? Well, almost. See, there's a caveat with this: it enables this behavior generally in `ShellExperienceHost.exe`. Try launching `ms-availablenetworks:`, which in Windows 11 should open a list like this:

![image](https://user-images.githubusercontent.com/6503598/142406030-e83231f9-6139-4da2-a188-38d9fe5037ce.png)

Instead, it will now open the Windows 11 notification center combined with calendar thing. So, the way it seems to me is that setting `byte_180571308` to 0 globally enables some legacy behavior in `ShellExperienceHost.exe`, let's call it. That's great for network and battery, as it fixes those, but it disables newer stuff, like the Windows 11 WiFi list or the Windows 11 action center. It is like an `UndockingDisabled` but for this case. Ideally, we would want to keep the best of the both worlds. This is discussed in the next part, where I develop a shippable solution as functionality for ExplorerPatcher.

### Implementation

Okay, so how do we wrap this knwoledge to deliver something that's actually shippable?

As you saw, the static patch is kind of limited in the sense that it enables only either of the 2 worlds. Of course, we could patch in our own assembly, but that's too much of a hassle, if what we did until now wasn't.

So let's consider dynamic patching. How to approach that?

First of all, let's consider what the common methods of executing our code there would be:

* Exploiting the library search order by placing our custom library in the same folder as the target executable, with the name and exports of a well known library. ExplorerPatcher already does that by masqueraiding as `dxgi.dll` (DirectX Graphics Infrastructure) for `explorer.exe` and `StartMenuExperienceHost.exe`.

  Looking on the list of imports from `dxgi.dll` for `ShellExperienceHost.exe`, similar to `explorer.exe`, it only calls `DXGIDeclareAdapterRemovalSupport` very early in the execution stage, so that is a viable option and candidate for our entry point.

* Hooking and patching the executable at run time. For this, there are a plethora of methods: the basic idea is to `CreateRemoteThread` in the target process which executes shellcode we wrote there using `WriteProcessMemory` that loads our library in the target process and executes its entry point. I used this in the past in plenty of places, including as recent as hooking `StartMenuExperienceHost.exe` with that. Unfortunately, due to some reason that's still unknown to me at the moment, calling `LoadLibraryW` when using this method in the remote process to load our library *sometimes* fails with `ERROR_ACCESS_DENIED`. Same config, it only happens on some systems. It's very weird, seems like a half baked "security" feature oby Microsoft, since I can execute the shell code just fine, it is only `LoadLibraryW` that fails. If this is indeed considered "security", it's just plain stupid because the solution is obviously for one to write their own library loader, which I will certainly do at some point for future projects. Prefferably in amd64 assembly, so that it's just one `char*` array in C into which I write some offsets at runtime, write it to the remote process, execute it and off to the races.

So, considering the above, I will go with the first option. For ExplorerPatcher, this is advantageous as this is basically the same infrastructure as for the 2 other executables that are hooked (`explorer.exe` and `StartMenuExperienceHost.exe`), so it keeps the implementation rather tidy.

Now, what do we do once we get to execute code in the target? We need to patch that `jmp` from before basically, or patch out the entire function maybe? It depends on the strategy that we choose. Let's recall our options:

* Hook/inject functions exported by system libraries used by the target application. This is used extensively though ExplorerPatcher and it is the prefferable method, when possible. It involves hooking some known function exported by libraries such as `user32.dll`, `shell32.dll` etc. This works when the thing we want to modify is calling some function like this, or depends on it etc. The patch is done by altering the delay import table or the import table for a library from the target. The limitations are that you need to have the target actually call some known function from some library (which is less likely in these new "Windows Runtime" executables Microsoft is flooding more and more of the OS with) and sometimes it's hard to filter calls from other actors from calls from actors you are interested in.

* Hook/inject any function. When I say "any", I mean the rest of the functions, those actually contained in the target we want to exploit. Conviniently, Microsoft makes symbols available for its system files. These files tell you human-friendly names for various things in the executable code, including function sites. Hooking is usually done by installing a trampoline, basically patching the first few bytes of the function to jump to our custom crafted function where we do what we want and then we call the original we saved a pointer to. There are plenty of libraries to aid with this, like [funchook](https://github.com/kubo/funchook) or [Microsoft Detours](https://github.com/microsoft/Detours). I have done it manually as well in the past, the big problem is that in order to automate this, you need a disassembler at runtime: recall that x86 is CISC (instructions length varies). Say your trampoline takes 7 bytes, you need to patch the first 7 bytes + whatever it takes to complete the last instruction. To determine what that additional amount is, obviously the program needs to be able to understand x86 instructions at runtime.

  The disadvantage is that this is global, affects the entire executable address space, plus the symbol addresses change with each version compiled, so those have to be updated for every new Windows build somehow (ExplorerPatcher uses a combination of downloading the symbols for your current build from Microsoft and extracting the info from there, and hardcoding the addresses for some known file releases). This is used in ExplorerPatcher for the `Win`+`X` menu build function from `twinui.pcshell.dll`, the context menu owner draw functions still from there and for hooking some function calls in `StartMenuExperienceHost.exe`; this is usually a last resort method.

* Pattern matching. This is the classic of the classics. Basically, you try to determine as generic of a strategy as possible to identify some bytes in your target function that is likely to work on future versions and that is unique to your function. If you are smart, you can pull it off, if you are lucky, it also resists more than one build.

I feel lucky today, or rather, I feel that some things in that function are pretty unique and likely not to change based on my previous experiences diassembling Microsoft's stuff, so I will go with this. The reason is also that I want to try to minimize the symbols fiasco (Microsoft recently delays publishing the symbols for some unknown reason for some builds, and then people running beta builds start flooding the forums with requests for me to "fix" it); it's just one thing to patch, I don't want to introduce the need for other symbols, and if we anyway have 2 methods of hooking stuff already present in ExplorerPatcher, what can a third one do...?

```cpp
void InjectShellExperienceHost()
{
#ifdef _WIN64
    HMODULE hQA = LoadLibraryW(L"Windows.UI.QuickActions.dll");
    if (hQA)
    {
        PIMAGE_DOS_HEADER dosHeader = hQA;
        if (dosHeader->e_magic == IMAGE_DOS_SIGNATURE)
        {
            PIMAGE_NT_HEADERS64 ntHeader = (PIMAGE_NT_HEADERS64)((u_char*)dosHeader + dosHeader->e_lfanew);
            if (ntHeader->Signature == IMAGE_NT_SIGNATURE)
            {
                char* pSEHPatchArea = NULL;
                char seh_pattern1[14] =
                {
                    // mov al, 1
                    0xB0, 0x01,
                    // jmp + 2
                    0xEB, 0x02,
                    // xor al, al
                    0x32, 0xC0,
                    // add rsp, 0x20
                    0x48, 0x83, 0xC4, 0x20,
                    // pop rdi
                    0x5F,
                    // pop rsi
                    0x5E,
                    // pop rbx
                    0x5B,
                    // ret
                    0xC3
                };
                char seh_off = 12;
                char seh_pattern2[5] =
                {
                    // mov r8b, 3
                    0x41, 0xB0, 0x03,
                    // mov dl, 1
                    0xB2, 0x01
                };
                BOOL bTwice = FALSE;
                PIMAGE_SECTION_HEADER section = IMAGE_FIRST_SECTION(ntHeader);
                for (unsigned int i = 0; i < ntHeader->FileHeader.NumberOfSections; ++i)
                {
                    if (section->Characteristics & IMAGE_SCN_CNT_CODE)
                    {
                        if (section->SizeOfRawData && !bTwice)
                        {
                            DWORD dwOldProtect;
                            VirtualProtect(hQA + section->VirtualAddress, section->SizeOfRawData, PAGE_EXECUTE_READWRITE, &dwOldProtect);
                            char* pCandidate = NULL;
                            while (TRUE)
                            {
                                pCandidate = memmem(
                                    !pCandidate ? hQA + section->VirtualAddress : pCandidate,
                                    !pCandidate ? section->SizeOfRawData : (uintptr_t)section->SizeOfRawData - (uintptr_t)(pCandidate - (hQA + section->VirtualAddress)),
                                    seh_pattern1,
                                    sizeof(seh_pattern1)
                                );
                                if (!pCandidate)
                                {
                                    break;
                                }
                                char* pCandidate2 = pCandidate - seh_off - sizeof(seh_pattern2);
                                if (pCandidate2 > section->VirtualAddress)
                                {
                                    if (memmem(pCandidate2, sizeof(seh_pattern2), seh_pattern2, sizeof(seh_pattern2)))
                                    {
                                        if (!pSEHPatchArea)
                                        {
                                            pSEHPatchArea = pCandidate;
                                        }
                                        else
                                        {
                                            bTwice = TRUE;
                                        }
                                    }
                                }
                                pCandidate += sizeof(seh_pattern1);
                            }
                            VirtualProtect(hQA + section->VirtualAddress, section->SizeOfRawData, dwOldProtect, &dwOldProtect);
                        }
                    }
                    section++;
                }
                if (pSEHPatchArea && !bTwice)
                {
                    DWORD dwOldProtect;
                    VirtualProtect(pSEHPatchArea, sizeof(seh_pattern1), PAGE_EXECUTE_READWRITE, &dwOldProtect);
                    pSEHPatchArea[2] = 0x90;
                    pSEHPatchArea[3] = 0x90;
                    VirtualProtect(pSEHPatchArea, sizeof(seh_pattern1), dwOldProtect, &dwOldProtect);
                }
            }
        }
    }
#endif
}
```

Firstly, I match by this pattern:

```asm
.text:00000001800407DA                 mov     al, 1
.text:00000001800407DC                 jmp     short loc_1800407E0
.text:00000001800407DE ; ---------------------------------------------------------------------------
.text:00000001800407DE                 xor     al, al
.text:00000001800407E0
.text:00000001800407E0 loc_1800407E0:                          ; CODE XREF: FlightHelper__CalculateRemodelEnabled+F0↑j
.text:00000001800407E0                 add     rsp, 20h
.text:00000001800407E4                 pop     rdi
.text:00000001800407E5                 pop     rsi
.text:00000001800407E6                 pop     rbx
.text:00000001800407E7                 retn
```

Then skip some bytes (the load address of and function call to that address, some telemetry call) and try to match this which is also a pattern I have seen in many of their libraries, I haven't bothered to understand but it seems to stay there:

```asm
.text:00000001800407C9                 mov     r8b, 3
.text:00000001800407CC                 mov     dl, 1
```

On builds 22000.318 and 22000.346, first pattern yields 2 results, while adding the last one yields only the result we are interested in. Additionally, I patch only if I find a single match, as otherwise it is likely the file actually changed drastically.

Okay, so we do this patch at runtime, now what? It's still going to behave like the static patch, ain't it?

If we leave it like this, indeed it is. So, we have to enhance it a bit. At first I tried signaling `ShellExperienceHost.exe` to switch between the 2 modes, by patching and reverting those 2 bytes. Unfortunately, this doesn't really work: once a mode is set, it stays like that: things already load in that mode and calling stuff only available in the other mode crashes `ShellExperienceHost.exe` apparently.

So what next? From the way I toggle the Windows 11 WiFi list, I already have code that opens something that is a `Windows.UI.Core.CoreWindow` and waits for it to close. The principle would be simple here: when the network or battery flyout is invoked, kill `ShellExperienceHost.exe` if running, somehow signal the instance that will open that we want it to run in legacy mode, wait for the flyout to be dismissed by the user (using the infrastructure from above), and then kill `ShellExperienceHost.exe` again so that Windows 11 things still open normally.

Last quest: how do we signal `ShellExperienceHost.exe` that we want it to run in legacy mode. Better said, how do we signal out code running in there this?

Well, UWP apps are kind of jailed. They have limited access to the file system and registry. I experienced this as well when working in `StartMenuExperienceHost.exe`. I don't know exactly how to determine what they have/do not have access to, as that also seems to vary quite a lor based on the actual executable we are talking about, plus I do not really care. I care about making this working. So, after failing getting it to work with named events, I looked into what keys in the registry this program acesses. Indeed, I found this path in `HKCU`:

```
Control Panel\Quick Actions\Control Center\QuickActionsStateCapture
```

So, to finish this off, to signal the legacy mode, when it's time to invoke the network/battery flyout, I create an `ExplorerPatcher` key in there. When our library gets injected in `ShellExperienceHost.exe`, we check whether that key exists and only then patch the executable; add this to the function above, right at the beginning:

```cpp
    HKEY hKey;
    if (RegOpenKeyW(HKEY_CURRENT_USER, _T(SEH_REGPATH), &hKey))
    {
        return;
    }
```

After the flyout is dismissed, we make sure to delete the key, and then terminate `ShellExperienceHost.exe`, so that the new instance that will be launched eventually loads normally, in Windows 11 mode.

That would be it here.

### Bonus: the language switcher

I mentioned this in the beginning, so let's talk about this a bit as well: I recently needed a way to detect when the input language changes for sws (my custom, written from scratch implementation of a Windows 10-like `Alt`-`Tab` switcher). The reason I needed to detect this is so that I can change the key mapping for `Alt`+`~`, which shows the window switcher but only for windows of the current application. I need to change it because `~` has a different virtual key code depending on the layout that it is loaded: it is either `VK_OEM_3`, either `VK_OEM_5`, alrgely. Or rather, not `~` specifically, but the key above `Tab`. I always wanted to map that key in combination with `Alt`. I know it by scan code (29 if I remember correctly), but `RegisterHotKey` only accepts virtual key codes, to the hot key has to be unregistered and registered every time the user changes the layout.

As it is pretty standard with Windows these days, there are a couple of old APIs that largely do not work anymore or are weird:

* `WM_INPUTLANGCHANGE` is received only by the active window, which in 99.99% of the cases is not the window switcher
* this thing ([ITfLanguageProfileNotifySink](https://docs.microsoft.com/en-us/windows/win32/api/msctf/nn-msctf-itflanguageprofilenotifysink?redirectedfrom=MSDN)) does not really work, or I could not get it ot work, or rather, when I was seeing how much time I was wasting trying ot make it work and with Google failing to provide me with input from users facing the same hurdles I was facing, I decided to give up on it

So, what to do? Naturally, disassemble their executables and see how they do it. Isn't this normal? Isn't a closed system you are discouraged from tempering and playing with the norm these days?

I looked again though `explorer.exe`: in one of its `CoCreateInstance` calls, it requests some `IID_InputSwitchControl` interface from `CLSID_InputSwitchControl` (actual GUIDs are in ExplorerPatcher's source code):

```cpp
__int64 __fastcall CTrayInputIndicator::_RegisterInputSwitch(CTrayInputIndicator *this)
{
  LPVOID *ppv; // rbx
  HRESULT Instance; // edi

  ppv = (LPVOID *)((char *)this + 328);
  Microsoft::WRL::ComPtr<IVirtualDesktopNotificationService>::InternalRelease((char *)this + 328);
  Instance = CoCreateInstance(&CLSID_InputSwitchControl, 0i64, 1u, &IID_InputSwitchControl, ppv);
  if ( Instance >= 0 )
  {
    Instance = (*(__int64 (__fastcall **)(LPVOID, _QWORD))(*(_QWORD *)*ppv + 24i64))(*ppv, 0i64);
    if ( Instance < 0 )
    {
      Microsoft::WRL::ComPtr<IVirtualDesktopNotificationService>::InternalRelease(ppv);
    }
    else
    {
      (*(void (__fastcall **)(LPVOID, char *))(*(_QWORD *)*ppv + 32i64))(*ppv, (char *)this + 16);
      (*(void (__fastcall **)(LPVOID))(*(_QWORD *)*ppv + 64i64))(*ppv);
    }
  }

  return (unsigned int)Instance;
}
```

Okay, next step for determining the virtual table of this interface is to consult the registry and see which library actually implements it:

```
Computer\HKEY_LOCAL_MACHINE\SOFTWARE\Classes\CLSID\{B9BC2A50-43C3-41AA-A086-5DB14E184BAE}\InProcServer32
```

It points us to:

```
C:\Windows\System32\InputSwitch.dll
```

How do you know how the interface is named? Well, you don't, but look around in the DLL. There should be some methods called `CreateInstance` that corespond to the factories for a COM object. If something has a `CreateInstance` method, it's probably an interface. In `InputSwitchControl.dll`, there aren't many, and there's one called `CInputSwitchControl`. They usually name the interfaces after the implementations, prefixed with "C". 

Let's take that as a candidate. The strategy next is to go though all the functions of that "class" and see if they are mentioned in any virtual table, and then try to see if the positions in that virtual table match with whatever code in `explorer.exe` is calling it. We eventually get to a virtual table that looks like this one:

```
dq offset ?QueryInterface@?$RuntimeClassImpl@U?$RuntimeClassFlags@$01@WRL@Microsoft@@$00$0A@$0A@UIInputSwitchControl@@@Details@WRL@Microsoft@@UEAAJAEBU_GUID@@PEAPEAX@Z
dq offset ?AddRef@?$RuntimeClassImpl@U?$RuntimeClassFlags@$01@WRL@Microsoft@@$00$0A@$0A@UIInputSwitchControl@@@Details@WRL@Microsoft@@UEAAKXZ ; Microsoft::WRL::Details::RuntimeClassImpl<Microsoft::WRL::RuntimeClassFlags<2>,1,0,0,IInputSwitchControl>::AddRef(void)
dq offset ?Release@?$RuntimeClassImpl@U?$RuntimeClassFlags@$01@WRL@Microsoft@@$00$0A@$0A@UIInputSwitchControl@@@Details@WRL@Microsoft@@UEAAKXZ ; Microsoft::WRL::Details::RuntimeClassImpl<Microsoft::WRL::RuntimeClassFlags<2>,1,0,0,IInputSwitchControl>::Release(void)
dq offset ?Init@CInputSwitchControl@@UEAAJW4__MIDL___MIDL_itf_inputswitchserver_0000_0000_0001@@@Z ; CInputSwitchControl::Init(__MIDL___MIDL_itf_inputswitchserver_0000_0000_0001)
dq offset ?SetCallback@CInputSwitchControl@@UEAAJPEAUIInputSwitchCallback@@@Z ; CInputSwitchControl::SetCallback(IInputSwitchCallback *)
...
```

Is this the virtual table we're after?

If we look back on the disassembly from `explorer.exe`, we see 2 calls are made for functions within this interface:

```cpp
Instance = (*(__int64 (__fastcall **)(LPVOID, _QWORD))(*(_QWORD *)*ppv + 24i64))(*ppv, 0i64);
```

And

```cpp
(*(void (__fastcall **)(LPVOID, char *))(*(_QWORD *)*ppv + 32i64))(*ppv, (char *)this + 16);
```

That means the first 2 functions after `QueryInterface`, `AddRef` and `Release`, which would mean `Init` and `SetCallback` from above. Besides obviously checking at runtime (it's easy since the called library is an in-process server and handler), we can also be more sure by looking at the second call, specifically how it sends a pointer, `(char *)this + 16` to the library. That's equivalent to `(_QWORD*)this + 2` (an INT64 or QWORD is made up of 8 bytes or chars). If we look in the constructor of `CTrayInputIndicator` from `explorer.exe` (`CTrayInputIndicator::CTrayInputIndicator`), we see this on the first lines:

```cpp
  *(_QWORD *)this = &CTrayInputIndicator::`vftable'{for `CImpWndProc'};
  *((_QWORD *)this + 2) = &CTrayInputIndicator::`vftable'{for `IInputSwitchCallback'};
  *((_QWORD *)this + 1) = 0i64;
```

So, `(char *)this + 16` is the virtual table for an `IInputSwitchCallback`. That's what the `SetCallback` method from above also states it takes and what you would expect from a `SetCallback` method. So yeah, that pretty much seems to be it.

Now, how do you use this?

As shown above. First of all you initialize the input switch control by calling `Init` with a paramater 0. What's that 0? Let's see what `InputSwitchControl.dll` does with it; it's a long function (`CInputSwitchControl::_Init`), but at some point it does this:

```cpp
v5 = IsUtil::MapClientTypeToString(a2);
```

Sounds very interesting. And it looks even better:

```cpp
const wchar_t *__fastcall IsUtil::MapClientTypeToString(int a1)
{
  int v1; // ecx
  int v2; // ecx
  int v4; // ecx
  int v5; // ecx

  if ( !a1 )
    return L"DESKTOP";

  v1 = a1 - 1;
  if ( !v1 )
    return L"TOUCHKEYBOARD";

  v2 = v1 - 1;
  if ( !v2 )
    return L"LOGONUI";

  v4 = v2 - 1;
  if ( !v4 )
    return L"UAC";

  v5 = v4 - 1;
  if ( !v5 )
    return L"SETTINGSPANE";

  if ( v5 == 1 )
    return L"OOBE";

  return L"OTHER";
}
```

Now I understand. So `explorer.exe` seems to want the "desktop" user interface (kind of like the `SetNetworkUXMode` from above). There are other interfaces too. One that looks like the Windows 10 one I determined, by testing, to be `1`, aka `TOUCHKEYBOARD` (I exposed the rest via ExplorerPatcher's Properties GUI as well, each work to a certain degree).

Also, in my window switcher, I don't want any UI, I just want to benefit from using the callback, as you'll see in a moment. Again, by experimentation, it seems that providing any value not in that list (like 100) makes it not draw a UI, but the callback still works, the thing still works fine I mean. That's great!

That's cool. What about the callback? Well, I think the vtable explians it pretty well:

```
CTrayInputIndicator::QueryInterface(_GUID const &,void * *)
[thunk]:CTrayInputIndicator::AddRef`adjustor{16}' (void)
[thunk]:CTrayInputIndicator::Release`adjustor{16}' (void)
CTrayInputIndicator::OnUpdateProfile(__MIDL___MIDL_itf_inputswitchserver_0000_0000_0002 const *)
CTrayInputIndicator::OnUpdateTsfFloatingFlags(ulong)
CTrayInputIndicator::OnProfileCountChange(uint,int)
CTrayInputIndicator::OnShowHide(int,int,int)
CTrayInputIndicator::OnImeModeItemUpdate(__MIDL___MIDL_itf_inputswitchserver_0000_0000_0003 const *)
CTrayInputIndicator::OnModalitySelected(__MIDL___MIDL_itf_inputswitchserver_0000_0000_0005)
CTrayInputIndicator::OnContextFlagsChange(ulong)
CTrayInputIndicator::OnTouchKeyboardManualInvoke(void)
```

The method of interest for me in my window switcher was `OnUpdateProfile`, so I looked a bit on that. By trial and error and looking around, I determined its signature to actually be something like:

```cpp
static HRESULT STDMETHODCALLTYPE _IInputSwitchCallback_OnUpdateProfile(IInputSwitchCallback* _this, IInputSwitchCallbackUpdateData *ud)
```

Where the `IInputSwitchCallbackUpdateData` looks like this:

```cpp
typedef struct IInputSwitchCallbackUpdateData
{
    DWORD dwID; // OK
    DWORD dw0; // always 0
    LPCWSTR pwszLangShort; // OK ("ENG")
    LPCWSTR pwszLang; // OK ("English (United States)")
    LPCWSTR pwszKbShort; // OK ("US")
    LPCWSTR pwszKb; // OK ("US keyboard")
    LPCWSTR pwszUnknown5;
    LPCWSTR pwszUnknown6;
    LPCWSTR pwszLocale; // OK ("en-US")
    LPCWSTR pwszUnknown8;
    LPCWSTR pwszUnknown9;
    LPCWSTR pwszUnknown10;
    LPCWSTR pwszUnknown11;
    LPCWSTR pwszUnknown12;
    LPCWSTR pwszUnknown13;
    LPCWSTR pwszUnknown14;
    LPCWSTR pwszUnknown15;
    LPCWSTR pwszUnknown16;
    LPCWSTR pwszUnknown17;
    DWORD dwUnknown18;
    DWORD dwUnknown19;
    DWORD dwNumber; // ???
} IInputSwitchCallbackUpdateData;
```

The `dwID` is what contains a HKL combined with a language ID. Its an entire theory that is kind of well explained here that you understand in one evening, you make use of it and then you quickly forget as its too damaging for the human brain to remember such over engineered complications:

https://referencesource.microsoft.com/#system.windows.forms/winforms/Managed/System/WinForms/InputLanguage.cs

Also, take a look on my window switcher's implementation, to see how I extract the HKL from that:

https://github.com/valinet/sws/blob/74a906c158a91100377a6e8220b0a3c5a8e98657/SimpleWindowSwitcher/sws_WindowSwitcher.c#L3

So, back to `explorer.exe`, how do we make it load the Windows 10 switcher instead? That 0 seems to be hardcoded in the code there.

Well, it is, but if we look at the disassembly:

```asm
.text:000000014013B21E                 call    cs:__imp_CoCreateInstance
.text:000000014013B225                 nop     dword ptr [rax+rax+00h]
.text:000000014013B22A                 mov     edi, eax
.text:000000014013B22C                 test    eax, eax
.text:000000014013B22E                 js      short loc_14013B294
.text:000000014013B230                 mov     rcx, [rbx]
.text:000000014013B233                 mov     rax, [rcx]
.text:000000014013B236                 mov     r10, 0ABF9B6BC16DC1070h
.text:000000014013B240                 mov     rax, [rax+18h]
.text:000000014013B244                 xor     edx, edx
.text:000000014013B246                 call    cs:__guard_xfg_dispatch_icall_fptr
```

Maybe patching the virtual table of the COM interface could be a solution, but it is a bit difficult because the executable is protected by control flow guard. How do you tell? Besides confirming with `dumpbin`, it's right there in front of you. That `mov     r10, 0ABF9B6BC16DC1070h` is the canary that is written before the target call site. So you have to write that + 1 (`0ABF9B6BC16DC1071h`, you can confirm by looking above `?Init@CInputSwitchControl` in `InputSwitchControl.dll`) above your target function. Doable, I think. Haven't tried, but will probably will when the hack that I did breaks, or with some other oppotunity (I have to figure out how to have the compiler place that number there automatically, without me patching the binary afterwards manually). Also, I don't know if those numbers change between versions of the libraries, I presume not, as that would break compatibility between versions, but I do really have to learn more about CFG.

For this, I instead obted to hack it away, by observing that `edx` is never changed beside that `xor edx, edx`. So, I just have to neuter that, and then set it myself and immediatly return from my `CoCreateInstance` hook. Like this:

```cpp
//globals:
char mov_edx_val[6] = { 0xBA, 0x00, 0x00, 0x00, 0x00, 0xC3 };
char* ep_pf = NULL;

//...
// in CoCreateInstance hook:
char pattern[2] = { 0x33, 0xD2 };
DWORD dwOldProtect;
char* p_mov_edx_val = mov_edx_val;
if (!ep_pf)
{
	ep_pf = memmem(_ReturnAddress(), 200, pattern, 2);
	if (ep_pf)
	{
		// Cancel out `xor edx, edx`
		VirtualProtect(ep_pf, 2, PAGE_EXECUTE_READWRITE, &dwOldProtect);
		memset(ep_pf, 0x90, 2);
		VirtualProtect(ep_pf, 2, dwOldProtect, &dwOldProtect);
	}
	VirtualProtect(p_mov_edx_val, 6, PAGE_EXECUTE_READWRITE, &dwOldProtect);
}
if (ep_pf)
{
	// Craft a "function" which does `mov edx, whatever; ret` and call it
	DWORD* pVal = mov_edx_val + 1;
	*pVal = dwIMEStyle;
	void(*pf_mov_edx_val)() = p_mov_edx_val;
	pf_mov_edx_val();
}
```

https://github.com/valinet/ExplorerPatcher/blob/ff26abe9a39fb90510450356ba2a807fb97cfa69/ExplorerPatcher/dllmain.c#L4247

The result?

![image](https://user-images.githubusercontent.com/6503598/142456688-87640438-7812-467f-ad1b-00e1db8a594d.png)

### Conclusion

Not the most beautiful patches in the world, but they work out rather nicely. Quite some stuff has been achieved with the limited resources available at our disposal. Of course, Microsoft fixing the interface would still be the preferable option, like, in some cases, it only takes 2 bytes give or take, but they chose to deliver a label-less taskbar instead, that's simply a productivity nightmare. Oh, well... at least [ExplorerPatcher](https://github.com/valinet/ExplorerPatcher) exists.

Yeah, a long post, but hopefully it gave you some ideas. Let's hack away!

P.S. Before asking, no, the 2 battery icons are not some "side effect" or a bug or anything like that: one is Windows' icon, and the other is the icon of the excellent and versatile [Battery Mode](https://github.com/tarcode-apps/BatteryMode) app one can use as a better replacement to what Windows offers in any of its modes.
