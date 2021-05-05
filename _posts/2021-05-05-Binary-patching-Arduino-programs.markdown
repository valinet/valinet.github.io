---
layout: post
title:  "Binary patching Arduino programs"
date:   2021-05-05 00:00:00 +0000
categories: 
excerpt: "I have recently changed the subnet of my home network because reasons. The old subnet was 192.168.105.0/24, while the new subnet is 10.1.1.0/24. This poses a challenge because, in order to save up on precious program memory, my homepi+ Arduino has a static IP set (this way, I do not have to compile in the DHCP code which is pretty large). So, I have to change the IP in the Arduino. Sure, I can recompile the program and flash it on the Arduino, but, as I said, I have hit a golden master, I want the same program output and just to change 3 bytes in the IP address. So, binary patching it is then."
---

As you may already know, I am quite a fan of Arduinos. Not that I think C++ is really necessary on a microcontroller of its size, but rather I appreciate the fairly good documentation and awesome community and libraries that's built around it. All in all, the ecosystem is pretty great.

I have a few Arduino projects around my house, with the most important being [homepi+](https://github.com/valinet/homepi-plus), which is a home control system. Basically, the Arduino presents me with a page on the local network (and even on the Internet, password protected) from which I can control various stuff in my work room, like lights, monitor brightness, monitor input etc.

I have this running 24/7 for a few months now, unattended and it's been rock solid after some hardening (implemented a watch dog, just in case etc). So I am reluctant to changing the code on it, it seems that I have hit a golden master. Though, I have recently changed the subnet of my home network "because reasons". The old subnet was 192.168.105.0/24, while the new subnet is 10.1.1.0/24. This poses a challenge because, in order to save up on precious program memory, my homepi+ Arduino has a static IP set (this way, I do not have to compile in the DHCP code which is pretty large). So, I have to change the IP in the Arduino. Sure, I can recompile the program and flash it on the Arduino, but, as I said, I have hit a golden master, I want the same program output and just to change 3 bytes in the IP address. So, binary patching it is then.

### Dump the program

The first step is to connect the Arduino to the PC and dump the program that exists on the golden master. For this, open a command prompt, browse to the binaries folder of your Arduino IDE installation and run something like this:

```
cd "C:\Program Files (x86)\Arduino\hardware\tools\avr\bin"
avrdude.exe -CC:\Users\<user name>\AppData\Local\Arduino15\packages\MiniCore\hardware\avr\2.1.2/avrdude.conf -v -patmega328p -carduino -PCOM10 -b115200 -n -U flash:r:backup.bin
```

This will dump the file as "backup.hex" in the same folder. I would have expected this to fail, as that folder is in Program Files and I run the command unelevated, but something even stranger happened. It actually worked, but the file was created in a "virtualized" copy of the folder, at:

```
C:\Users\<user name>\AppData\Local\VirtualStore\Program Files (x86)\Arduino\hardware\tools\avr\bin
```

Apparently, this is a compatibility feature introduced since Windows Vista, when User Account Control was introduced, in order to make the transition easier. Older programs that are not UAC-aware (not Vista aware, more precisely) and that are 32-bit are not denied a write to a sensible location, but rather the location to which they write is virtualized. Neat, read more about this feature [here](https://answers.microsoft.com/en-us/windows/forum/windows_7-windows_programs/please-explain-virtualstore-for-non-experts/d8912f80-b275-48d7-9ff3-9e9878954227).

Anyway, how do you figure out that "avrdude.exe ..." command? How do you guess those parameters? Well, my approach is to open the Arduino IDE, go to File - Preferences and check "compilation" and "upload" near "Show verbose output during". Then, just upload a test sketch and copy the command Arduino IDE uses to upload to my board and go from there.

Now, once you have the dump, let's move on and patch it.

### Determine what to patch

Let's look on the source code of my homepi+ project to figure out how I store the IP. In a real scenario, you probably wouldn't have access to the source code, so you'd have to figure this out yourself, but of course, to get good at this, you need some experience which you can definitely get through exercises like this one.

There is this relevant section:

{% highlight c %}
const static uint8_t ip[] = {192,168,105,24}; //24
const static uint8_t gateway[] = {192,168,105,1};
{% endhighlight %}

So, conveniently enough, the IP is stored as a byte array in memory. Let's try to find that in the dumped file.

I recommend opening the file in a hex editor like [HxD](https://mh-nexus.de/en/hxd/). Then, go to Search - Find, Hex values and look for "c0a869". That's the network of those addresses (i.e. "192.168.105"). Click Search all. HxD finds 2 matches. The one followed by "01" is probably the gateway address, as it translates to "192.168.105.1", while the other is exactly our current IP, "192.168.105.24".

![HxD matches](/res/20210505/hxd-matches.png)

That's great, it seems mission accomplished. All I have to do is edit these 6 bytes, substituting "c0a869" with "0a0101" (which is 10 1 1). Save the file and we are ready to flash it back.

### Flashing the modified program

In order to flash this new file, issue a command similar to the one below:

```
avrdude.exe -CC:\Users\<user name>\AppData\Local\Arduino15\packages\MiniCore\hardware\avr\2.1.2/avrdude.conf -v -patmega328p -carduino -PCOM10 -b115200 -D -U flash:w:edited.bin
```

With that done, I connected the Arduino back to the breadboard, powered everything on and navigated my browser to "10.1.1.24". To my surprise, it actually worked, although it asks me for the password. This happens when it detects the request as coming from the Internet, not the local network. To keep it short, the check responsible for this is the following in the source code:

{% highlight c %}
uint8_t* ip_src = ether.buffer + 0x1A;
if (!(ip_src[0] == ip[0] && ip_src[1] == ip[1] && ip_src[2] == ip[2])) { ... }
{% endhighlight %}

I know, I know, this is not really correct. To determine if the source IP is from the local network, I have to match it against the local network address (compare source IP AND mask with my IP AND mask). Because I was lazy, since I use the rather common /24 mask, and since it worked for my use case. I left this trivial check in the code for the time being; whether it is correct or not is not the scope of this tutorial. So, apparently, this check fails even for addresses in the local network. But why, all I did was change the device's IP and suddenly it does not recognize that as its network for that if statement. Why?

### Source code is not machine code

When doing this kind of stuff, remember that the source code is not a direct representation of the final machine code that makes up the program. The compiler is free and will do all kinds of shenanigans on your source code.

I was left puzzled as to why this would not work. I even got tired and went and flashed the microcontroller with a program I modified the IP directly in the source code and it worked just fine. And then it hit me: in the actual program, probably the comparisons have been optimized - since I compare to values known at compile time, it does not have to read elements from the `ip` array as I say in the source code. The comparisons can actually be done against immediate values. So, let's pursue this avenue.

### Identifying actual code within the dump file

While text buffers, or buffers of any kind usually pretty much go unchanged in the actual program file that is generated at compile time to be flashed on the MCU, immediate values do not necessarily have to appear verbose in the generated file. Why? Think about the familiar x86 architecture. A high level C instruction like:

{% highlight c %}
a = a + 1;
{% endhighlight %}

Might get into assembly as:

{% highlight asm %}
add rax, 1
{% endhighlight %}

But it might rather get as:

{% highlight asm %}
inc rax
{% endhighlight %}

Which is actually `48 ff c0`, while the `add rax, 1` would be `48 83 c0 01`. So, quite a difference. The machine code produces the same result, but x86 has a dedicated instruction for incrementing a register by 1. It is useless to search for occurrences of `1` in the dumped file if the compiler would have chosen to use `inc rax`, for example. So, for this step, besides basic knowledge of the architecture you are working with, you also actually need to disassemble the dump, i.e. to dump the opcodes from the file - have a program interpret the bytes in the dump as if they were instructions on that architecture.

As the Arduino is based on AVR MCUs, I found an AVR disassembler that gets the job done: [AVRDisassembler](https://github.com/twinearthsoftware/AVRDisassembler). To have it dump the opcodes from a program file into a file, issue a command like the following:

```
AVRDisassembler.exe -i backup.hex > backup.txt
```

As you see, you need the dump to be in "Intel Hex" format. There are 2 ways to get this:

1. Dump directly in Intel Hex with a command like this (notice the `:i` at the end which tells avrdude to dump in Intel Hex format as opposed to the raw default format):

   ```
   avrdude.exe -CC:\Users\<user name>\AppData\Local\Arduino15\packages\MiniCore\hardware\avr\2.1.2/avrdude.conf -v -patmega328p -carduino -PCOM10 -b115200 -n -U flash:r:backup.bin:i
   ```

2. [Convert a raw dump to an Intel Hex dump using `objcopy`](https://stackoverflow.com/questions/26961795/converting-from-hex-to-bin-for-arm-on-linux) (this runs on Linux):

   ```
   objcopy --input-target=binary --output-target=ihex backup.bin backup.hex
   ```

After running the disassembler, you get a huge text file with every group of bytes translated into op codes and a human readable description. Of course, not every data in the file is actual op codes when i actually runs on the MCU - data in PROGMEM, like global variables is also wrongly translated into op codes by this tool. But don't worry, since the architecture is RISC and so op codes are fixed length, misinterpretation of regions cannot produce different opcodes interpretations, as opposed to a CISC architecture like x86.

Now, how do we go from here? Well, knowing how the C code looked, I decided that a good way to look around the code would be to search for either of the 3 values: `c0`, `a8`, `69` and see whether the remaining 2 values are located nearby. Since `0xa8` produces the fewest matches, I began looking around it. Sure enough, the last match looked very promising:

```
69A8:	80-3C       	cpi r24, 0xc0         ; Compare with Immediate
69AA:	49-F4       	brbc 1, .+18          ; Branch if Bit in SREG is Cleared
69AC:	80-91-E0-03 	lds r24, 0x03e0       ; Load Direct from Data Space (32-bit)
69B0:	88-3A       	cpi r24, 0xa8         ; Compare with Immediate
69B2:	29-F4       	brbc 1, .+10          ; Branch if Bit in SREG is Cleared
69B4:	80-91-E1-03 	lds r24, 0x03e1       ; Load Direct from Data Space (32-bit)
69B8:	89-36       	cpi r24, 0x69         ; Compare with Immediate
69BA:	09-F4       	brbc 1, .+2           ; Branch if Bit in SREG is Cleared
69BC:	BE-CA       	rjmp .-2692           ; Relative Jump
```

Bingo! As I thought, that `if` statement seemed to have been indeed transformed into comparisons against immediate values. If you look at the code, it actually does 3 successive comparisons against those 3 values, similar to my C code. And, as I said, if you look at the op codes, they contain nothing like `0xc0`, `0xa8`, or `0x69`. The opcodes we have to edit are: `80 3c`, `88 3a` and `89 36`. So, we go to those offsets in HxD and let's replace. But with that?

As you can see, apparently, AVR has dedicated opcodes for every comparison with an immediate against any or most of its registers. So, a table of opcodes is more then necessary; use the one [here](http://lyons42.com/AVR/Opcodes/AVRAllOpcodes.html), for example.

If you search in there for `cpi r24, 0xc0` for example, you will find a match in a table where the row will read `3c8x` and the column `0`. Combine those and you get `3c80`, which is exactly `80 3c` from above. Nice. We need to replace this with `cpi r24, 0x0a` (192 is now 10), which is `308x` and `a`, so `308a`, so `8a 30`. Repeat this procedure for the other 2 instructions and replace those bytes in HxD (168 becomes 1, 105 becomes 1 as well).

Lastly, save the file in HxD and flash the MCU again. Checking the result by visiting the web site yields a big success: I am not prompted for the password anymore, which means that my judging was correct. Yay!

### Conclusion

In the end, things turned out rather nice - I was able to patch my own program without touching the source code. I always prefer to do this if I can rather then recompiling because this does not, usually, necessitate the whole toolchain, has a limited scope and rather limited lateral effects, with the patch usually being confined to a few instructions that are to be changed.

What you need to pay attention to, and the morale of this write up: never forget that source code is not assembly or actual instructions or "machine code". Compilers do architecture specific and algorithmic optimizations, and those may turn even machine code friendly code like C into pretty different stuff. And always take a systemic approach to binary patching, changing a few stuff as possible, in order to avoid creating unforeseen problems.

Happy hacking!