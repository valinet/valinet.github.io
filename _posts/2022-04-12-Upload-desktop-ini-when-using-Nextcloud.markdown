---
layout: post
title:  "Upload desktop.ini when using Nextcloud"
date:   2022-04-12 00:00:00 +0000
categories: 
excerpt: "Is it worth binary patching open source programs, when you need (or fee like needing) that teeny-tiny change? And then, how do you approach it? Learn my take on this matter, with a practical example on the Nextcloud desktop client for Windows that refused to syncronize some files on my end."
---

Again, much time has passed since the last post here. Again, [ExplorerPatcher](https://github.com/valinet/ExplorerPatcher) is the "culprit". Anyway, I have decided to take a small break off it, for now.

Without further do, let's begin.

### Background

Recently, I have decided to find a solution to keep files syncronized between my home and work PC. I am using my own server for this, as I kind of hate storing personal files in the public clouds offered by various vendors, for obvious reasons. After going through the pitfalls of offline files (XP-era tech) and Work Folders (8.1-era tech), I decided those simply won't cut it. It didn't help that these features are largely forgotten and left by Microsoft in a zombie state in the latest iterations of the OS.

Anyway, having picked up Nextcloud next, I turned to their rather good (and open source) desktop client. Things went relatively smoothly, except I had this persisting problem: there is this `desktop.ini` file that Windows creates in various folders whenever you customize it in one way or another, and for some reason I have later learnt about, Nextcloud refuses to sync that, so it always remains in a pending state when looked at using File Explorer.

If you just see the pending icon on the parent folder, yet all files inside seemed to have synced, than you might simply not be able to see the `desktop.ini` files. To enable these, go to "Folder Options - View tab - Hide protected operating system files (Recommended)".

### Investigation

So, what about this? First, I used Google to search for stuff realted to this. I came up with [this link](https://git.furworks.de/opensourcemirror/nextcloud-desktop/commit/a480a318fd4bd2c6bbfa980ccb901e976a83e4aa) which in turn was a mirror of [the official repo](https://github.com/nextcloud/desktop/commit/a480a318fd4bd2c6bbfa980ccb901e976a83e4aa) which contains information about why they have chosen not to sync `desktop.ini` files: see, they create such a file to specify information for File Explorer that the Nextcloud folder should use a custom icon, yet they can't sync that because the path where that folder is located may differ from one PC to another, depending on where you have Nextcloud installed. I mean, sure, they could just patch the file for each computer instead, or just enforce this restriction in the root folder, yet it seems they exclude files named like this in all the subdirectories and this is what I don't really get why.

Anyway, next step was to look on the current source code and see how it looks currently. So:

```
git clone https://github.com/nextcloud/desktop nextcloud-desktop
```

Then, use `grep` to find where `desktop.ini` is mentioned:

```
grep -rn "desktop.ini"
```

It mentions these places:

```
ChangeLog:307:* Excludes: Hardcode desktop.ini
src/csync/csync_exclude.cpp:201:    /* Do not sync desktop.ini files anywhere in the tree. */
src/csync/csync_exclude.cpp:202:    const auto desktopIniFile = QStringLiteral("desktop.ini");
```

Of interest is `csync_exclude.cpp`. The mentioned lines look like this:

```cpp
    /* Do not sync desktop.ini files anywhere in the tree. */
    const auto desktopIniFile = QStringLiteral("desktop.ini");
    if (blen == static_cast<qsizetype>(desktopIniFile.length()) && bname.compare(desktopIniFile, Qt::CaseInsensitive) == 0) {
        return CSYNC_FILE_SILENTLY_EXCLUDED;
    }
```

Of course, I could recompile the whole application in order to take that out, but for taking an `if` out, all the dance (setting up a build environment etc) is not really necessary. We could binary patch the existing files, a strategy I really love when it comes to these types of modifications. It's much easier and takes way less time and effort, and it also teaches you things along the way.

So it's time to identify where that piece of code ends in the binaries that are actually shipped with the Windows client. Go to `C:\Program Files\Nextcloud` and look there. Of course, `nextcloud_csync.dll` seems like a good candidate, but suppose it wouldn't have been so obvious, you can use the [Strings](https://docs.microsoft.com/en-us/sysinternals/downloads/strings) tool from Sysinternals to look for the string in all the files in the folder, like so:

```
strings64.exe -o "C:\Program Files\Nextcloud\*" | findstr /i "desktop.ini"
```

The `-o` flag will have it print the offsets to the string in the file, like so (for `nextcloud_csync.dll`):

```
979304:/Desktop.ini
979712:Desktop.ini
979792:Remove Desktop.ini from
1184248:desktop.ini
```

Eventually, we conclude the "culprit" is indeed `nextcloud_csync.dll`. Of course, the next step is to load this in IDA and look on it.

After loading, search for `desktop.ini` (`Alt`+`T`) and we get these results:

![image](https://user-images.githubusercontent.com/6503598/162841414-205162f4-4171-461c-9cfc-bc15588064c4.png)

By exploring those, we can see that the disassembly does not match anything that resembles the source code. So, what's up, does IDA miss our string, does it not live in this file after all? To clear the mystery, I learnt about the "String" window in IDA (View - Open subviews - Strings). There, right click and choose "Setup...", and in there you can choose to display "Unicode C-style (16 bits)" strings as well. With `Ctrl`+`F` afterwards, we can search for it in the String window and identify it.

How do I know it's Unicode 16-bits? Well, you can tell by limiting `strings64.exe` to searching only for Unicode 16-bits strings (with the `-u` flag), in which case it will identify that. But IDA knows those, how did it not pick it up? Well, it doesn'tr ecognize it as a string, before it's nowhere referenced in the disassembly. But how could that be, the source code clearly accesses that string. Well, not really...

First, if we go to the string location we identified using the "Strings" window in IDA, it looks like this:

```
.data:00000001801237F8                 db  64h ; d
.data:00000001801237F9                 db    0
.data:00000001801237FA                 db  65h ; e
.data:00000001801237FB                 db    0
.data:00000001801237FC                 db  73h ; s
.data:00000001801237FD                 db    0
.data:00000001801237FE                 db  6Bh ; k
.data:00000001801237FF                 db    0
.data:0000000180123800                 db  74h ; t
.data:0000000180123801                 db    0
.data:0000000180123802                 db  6Fh ; o
.data:0000000180123803                 db    0
.data:0000000180123804                 db  70h ; p
.data:0000000180123805                 db    0
.data:0000000180123806                 db  2Eh ; .
.data:0000000180123807                 db    0
.data:0000000180123808                 db  69h ; i
.data:0000000180123809                 db    0
.data:000000018012380A                 db  6Eh ; n
.data:000000018012380B                 db    0
.data:000000018012380C                 db  69h ; i
.data:000000018012380D                 db    0
.data:000000018012380E                 db    0
.data:000000018012380F                 db    0
```

Why is that? The clue is right there in the source code: the string is defined as `QStringLiteral("desktop.ini")` actually, which is actually a specialized container for string stuff. The disassembly never accesses the string data directly, rather, as we can clearly see from the source code, it's passed to helper implementations that do stuff for us, like `.compare(` aka `QStringRef::compare`. What these helpers receieve is a pointer to this `QStringLiteral`, so the address of where its data is in memory. And how does it look in memory? Well, it indeed contains the actual string, but it actually starts with the length of the entire string. Thus, things like `.length` can be easily performed without iterating over the characters until reaching a `\0`, also enabling the possibility of strings that are not null terminated. The disassembly clearly tells us this story, actually:

```
.data:00000001801237E0 unk_1801237E0   db 0FFh                 ; DATA XREF: sub_18001F690:loc_18001F943↑o
.data:00000001801237E0                                         ; csync_is_windows_reserved_word(QStringRef const &)+276↑o
.data:00000001801237E1                 db 0FFh
.data:00000001801237E2                 db 0FFh
.data:00000001801237E3                 db 0FFh
.data:00000001801237E4 dword_1801237E4 dd 0Bh                  ; DATA XREF: sub_18001F690+2BE↑r
.data:00000001801237E8                 align 10h
.data:00000001801237F0                 db  18h
.data:00000001801237F1                 db    0
.data:00000001801237F2                 db    0
.data:00000001801237F3                 db    0
.data:00000001801237F4                 db    0
.data:00000001801237F5                 db    0
.data:00000001801237F6                 db    0
.data:00000001801237F7                 db 0
.data:00000001801237F8                 db  64h ; d
.data:00000001801237F9                 db    0
.data:00000001801237FA                 db  65h ; e
.data:00000001801237FB                 db    0
.data:00000001801237FC                 db  73h ; s
.data:00000001801237FD                 db    0
.data:00000001801237FE                 db  6Bh ; k
.data:00000001801237FF                 db    0
.data:0000000180123800                 db  74h ; t
.data:0000000180123801                 db    0
.data:0000000180123802                 db  6Fh ; o
.data:0000000180123803                 db    0
.data:0000000180123804                 db  70h ; p
.data:0000000180123805                 db    0
.data:0000000180123806                 db  2Eh ; .
.data:0000000180123807                 db    0
.data:0000000180123808                 db  69h ; i
.data:0000000180123809                 db    0
.data:000000018012380A                 db  6Eh ; n
.data:000000018012380B                 db    0
.data:000000018012380C                 db  69h ; i
.data:000000018012380D                 db    0
.data:000000018012380E                 db    0
.data:000000018012380F                 db    0
```

So, actually, IDA identified the beginning of the `QStringLiteral` instance, and called it `unk_1801237E0`. Where is that referenced in the disassembly? Well, exactly where the `QStringLiteral` is used, to the portion of the disassembly that corresponds to the source code I presented above. In pseudocode form, it looks like this:

```cpp
      v25 = &unk_1801237E0;
	  if ( v9 != dword_1801237E4 || (unsigned int)QStringRef::compare(&v19, &v25, 0i64) )
      {
        if ( v2 && OCC::Utility::isConflictFile(v3, v15) )
          v5 = 9;
        else
          v5 = 0;
      }
```

So, `v9 != dword_1801237E4` basically says "if the length of the current string (v9) is different from the length of the `desktop.ini` string (which is just the memory location `dword_1801237E4` from the above, as, you can see, right there at the beginning, after some zeros, it contains the length of the string at `dword_1801237E4` (`0xB` which is 11, as `desktop.ini` has 11 characters)". Current string in this conetxt means a string containing the name of the current file that the program is working with.

Then, `(unsigned int)QStringRef::compare(&v19, &v25, 0i64)` says "if the current string is not equal to the `desktop.ini` string, represented by that `QStringLiteral` instance". If we want IDA to pick that string when searching text using `Alt`+`T`, we can define the portion where the string is located as a string by clicking on the beginning of it, then `Alt`+`A` and then choosing the "Unicode C-style (16 bits)" option. 

The entire condition from the `if` check in the source code is negated, so to say. The statements are mathematically proven to be equivalent, you can look into [De Morgan's laws](https://en.wikipedia.org/wiki/De_Morgan%27s_laws) for that. Basically:

```
if (blen == static_cast<qsizetype>(desktopIniFile.length()) && bname.compare(desktopIniFile, Qt::CaseInsensitive) == 0)

is equivalent to NOT the following:

if ( v9 != dword_1801237E4 || (unsigned int)QStringRef::compare(&v19, &v25, 0i64) )
```

So, in the pseudocode generated from the disassembly, what's inside the if branches is what followed if we wouldn't have entered the if branch in the original source code.

To patch this, we basically want the if check to disappear and always continue with that it has inside, equivalent to failing the if on the original source code and continuing with what was next there. To devise a patch, a look on the disassembly is required:

```
.text:000000018001F943 loc_18001F943:                          ; CODE XREF: sub_18001F690+268↑j
.text:000000018001F943                 lea     rax, unk_1801237E0
.text:000000018001F94A                 mov     [rbp+arg_18], rax
.text:000000018001F94E                 movsxd  rax, cs:dword_1801237E4
.text:000000018001F955                 cmp     r15, rax
.text:000000018001F958                 jnz     short loc_18001F96F
.text:000000018001F95A                 xor     r8d, r8d
.text:000000018001F95D                 lea     rdx, [rbp+arg_18]
.text:000000018001F961                 lea     rcx, [rbp+var_20]
.text:000000018001F965                 call    cs:?compare@QStringRef@@QEBAHAEBVQString@@W4CaseSensitivity@Qt@@@Z ; QStringRef::compare(QString const &,Qt::CaseSensitivity)
.text:000000018001F96B                 test    eax, eax
.text:000000018001F96D                 jz      short loc_18001F990
.text:000000018001F96F
.text:000000018001F96F loc_18001F96F:                          ; CODE XREF: sub_18001F690+2C8↑j
.text:000000018001F96F                 test    r12b, r12b
.text:000000018001F972                 jz      short loc_18001F98E
.text:000000018001F974                 mov     rcx, rsi        ; this
.text:000000018001F977                 call    ?isConflictFile@Utility@OCC@@YA_NAEBVQString@@@Z ; OCC::Utility::isConflictFile(QString const &)
.text:000000018001F97C                 test    al, al
.text:000000018001F97E                 jz      short loc_18001F98E
.text:000000018001F980                 mov     edi, 9
.text:000000018001F985                 jmp     short loc_18001F990
```

So, the first part of the `if` check is this:

```
.text:000000018001F958                 jnz     short loc_18001F96F
```

Only if the comparison is not zero, we jump to the area inside the brackets (`loc_18001F96F`), but instead, we always want to jump there.

How do the opcodes for that look? Of course, since the thing is so close in the disassembly, a short conditional jump instruction is used: `0x75 0x15`: `0x75` means `jnz` and `0x15` is the offset relative to the current instruction. To jump there all the times, the patch is easy: turn `0x75` to `0xEB` which is an unconditional jump. It's very handy that only a single byte has to be patched after all.

Of course, I replace that byte in a hex editor, close Nextcloud, replace the original file with the modified one, reopen Nextcloud, wait a bit, and yeah, indeed, the client now simply uploaded the `desktop.ini` files as well. Nice.

### Future

How to find this easily in the future? By examining the source code, we can try to identify based on the call to `OCC::Utility::isConflictFile` from below. That function has 2 overloads; in the case of our source code, the one taking in a `QString` appears to be used, since `path` is of type `QString`, as defined in the function prototype. Of course, overloads only exist in the C++ world - when everything gets compiled, 2 actual methods are generated, with symbol names that reflect the arguments for each of the methods, since the name is not enough anymore to distinguish the two, but we know that each overload has different arguments (in that, `void foo(char a)` and `void foo(char b)` is illegal in C++).

So, open the DLL in IDA, search for `OCC::Utility::isConflictFile` in the Functions window, pick the method that takes in a `QString`. `Ctrl`+`X` will find its references, and at the moment there are only 3 of them:

![image](https://user-images.githubusercontent.com/6503598/162845134-c94267e0-431e-435a-8872-57e09b54e396.png)

Out of those, as you can see, only one of them is from an actual method, so probably what we are interested in. If we go there, we can see that was indeed the case, it gets us right to where that check for whether the file is called `desktop.ini` is performed in the code. From there, identifying the byte to patch shouldn't be too difficult.

### Conclusion

Is it worth binary patching open source programs? Well, it's always worth binary patching programs in general, if you can. With open source programs, the BIG advantage is that you always have the source code at hand, which can help you much easily and better understand what the heck is happening in the final executable. Sometimes, the compiler takes decisions that definitely help in some way, like reducing space, increasing execution speed, but which make human reading the program very tought to actually understand, and of course, having the source code is of big help.

And the truth is, upstreaming this is pretty tough. It's a very niche use case that I have, maybe it just glitches on my end or maybe the thing really is troublesome on the long run for some other reason. Idk, offering this behavior as an option may be too much work indeed, as well.

The trouble with binary patching are updates to the patched software. Anyway, here, since it's just a single byte, the program is open source, maintaining it wouldn't be that big of a pain. But still, it would be nice if computers were smart as to perform the steps described above (identifying `isConflictFile` etc) themselves on new versions and patch it automatically using the knowledge above. A framework where that would help in doing things like this with less struggle than the tradiitional methods. But yeah, unless I am missing something, that's still a dream, and a big burden, as I have learnt many times, with having such a patch apply at runtime, would be the difficulty it would take to have our piece of code loaded in the Nextcloud process, and then determining a reliable strategy of identifying that byte and patching it - most of the times without symbols/help generated by a tool like IDA. Idk, maybe one day though... Although, yeah, I am curious whether anyone knows a tool available today that would facilitate these kinds of binary patches - I personally don't know of any, at the moment.
