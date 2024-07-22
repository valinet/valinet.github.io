---
layout: post
title:  "Reserve RAM properly (and easily) on embedded devices running Linux"
date:   2024-07-23 00:00:00 +0000
categories: 
excerpt: "Nowadays, we are surrounded by a popular class of embedded devices: specialized hardware powered by an FPGA which is tightly coupled with a CPU on the same die which most of the times only runs mostly software that configures, facilitates and monitors the design. This article showcases a (rather old) technique which enables easy data sharing between hardware and software using the RAM."
---

Nowadays, we are surrounded by a popular class of embedded devices: specialized hardware powered by an FPGA which is tightly coupled with a CPU on the same die which most of the times only runs mostly software that configures, facilitates and monitors the design. Such examples are the ZYNQ devices from AMD (formely Xilinx). 

Sometimes, the application in question has to share some large quantity of data from the "hardware side" to the "software side". For example, suppose you have a video pipeline configured in hardware that acquires images from a camera sensor and displays them out over HDMI. The system also has some input device attached, so the user can press some key there (just an example) and have a screenshot saved to the storage medium (an SD card, let's say). A whole frame is quite big (1920 columns * 1080 lines * 2 color channels bytes, for a 4:2:0 1080p image let's say means approx. 4.2MB), so wasting hardware resources (like LUT/BRAM) is not really possible. 

Most of the times, designs of such systems have shared RAM modules that both sides can access at the same time: the FPGA can write to "some" memory area, and applications running on the CPU can read from that "some" memory area. Thus, the video pipeline could write the frames to RAM as well, and the software is free to grab frames from there whenever the user presses a button. Well, technically a bit after the user requests so, as frame grabbing would have to take place at specific moments (probably when the video pipeline just finished writing a new frame) in order to avoid having the image teared (saving the data when the video pipeline is just writing a new frame), but for the sake of this exercise let's not bother with that and the fact that the thing would be merely an illusion and not immediate (interrupt style, let's say).

So the hardware part is actually easy: we'll have some design that we can configure from software at runtime and tell what physical address in RAM to write to. Done.

The software part is tricky though. Most of the times, ideally, one wants to dedicate all "unused" RAM to the CPU, so there is flexibility for the applications running there. In practice, mostly just an application, since these are not general purpose computing platforms usually, but run custom made "distributions" which only include the (Linux) kernel, the minimum strictly necessary user land and the custom apps that help the system do whatever it has to do. For this goal, [Buildroot](https://buildroot.org/) is a popular open way to achieve highly customized, very specialized and lightweight system images. Or such designs forgo Linux entirely and run a single binary bare metal, especially when running real time is a goal. Anyway, the point is, the "coordinating" app is mostly the single process running for the entire lifetime of the system.

Despite that, Linux is not really thought out with that expectation in mind - it reasonably expects a couple of processes to run at any given time. We cannot just pick out some random address in RAM and write to it - there could be processes that were allocated memory there. So what's to do?

At the moment, I know three solutions, and I will go over each of them and tell you why I picked the last one.

# 1. Limit the RAM the OS 'sees'

This is done using the [devicetree](https://en.wikipedia.org/wiki/Devicetree). The hardware is then free to write data there without fearing it is going to hurt something in the software/CPU side. The problem here is that the OS cannot ever access that memory, but this technique works when the hardware needs to buffer data for itself and the CPU doesn't really need access to it.

# 2. Reserve some CMA

[Contiguous Memory Allocation](https://stackoverflow.com/questions/56415606/why-is-contiguous-memory-allocation-is-required-in-linux) is a feature where the kernel reserves some memory area for hardware devices that need support for physical contigous memory. It can be enabled using the device tree or the `cma=128MB` boot flag (that one, for example, instructs the kernel to reserve 128MB at the end of the physical memory space for CMA).

The workflow then would be to write a kernel module that requests the amount of CMA memory you want from the kernel, then configure the hardware with the physical address of that. And this approach works just fine. The problem is, you have to write a kernel module, implement communication from the user land with it - probably some `ioctl` where you tell the driver how much memory to request from the kernel and via which it tells you back the physical address it got. Certainly doable on a system we fully control, but do we really have to bother with a writing a driver and risk introducing instability in the one area where it hurts the most?

Actually, we can hack away a solution to this: simply pick some address in the CMA region without telling the kernel about it and configure the hardware to use it. From the software side, to access pages in that physical area, you `mmap` it using the `/dev/mem` device (an in-tree driver which allows full access to the entire visible RAM from the user space), which gives you a virtual address you can work with that points for x number of pages in the physical area you requested. Something like this:

```
uintptr_t phys_addr = 0x17000000;
int size = 1024;
int fd = open("/dev/mem", O_RDWR | O_SYNC);
char* virt_addr = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, phys_addr);
strcpy(virt_addr, "Hello, world");
// ...
```

Pretty simple, so what's wrong with it? 2 main things I'd say. Firstly, maybe you do not want access to the entire memory. What if you screw something up and write out of bounds? Yeah, there's that `size` parameter that protects you but still... what if some other process misbehaves, or some unwanted program is loaded...? Access like this to the entire memory is not really desired and not really needed, as you'll see in a bit. Secondly, and this is the big problem, the kernel knows nothing about what we are doing. We did not tell it we are using memory from there. The CMA is not for us to do what we want with it. It's a feature the kernel expects to use to service drivers that request memory from it. If you happen to have some driver that requests memory there and you use in your code the starting address of the CMA to do your own stuff as I showed you above, you're in for some funny times let's say - things will be screwed up in weird ways.

You can still hack around this: you could query from the user land and find out how much CMA the kernel handed out to requesting parties. It is allocated in number of pages from the beginning. There is no direct syscall to work with the CMA from the user space, as far as I know, since one's not really supposed to, but there is actually [a `debugfs` interface](https://stackoverflow.com/questions/65202674/finding-out-whats-taking-up-the-cma-contiguous-memory-allocation-in-linux) which can tell you how much is used, but really... there is no guarantee some device won't request more. Sure, if you control the hardware and know about it as well, just work with some memory a bit further away from where you expect/see the requests finish at and 99.99% of the time you can get away with this. But it's... hacky. Ugly. We need something better.

# 3. Properly reserve memory

Specifically for this requirement outlines above, the kernel actually has a dedicated mechanism: reserved memory. It's configured using the device tree, where you introduce an entry that looks like this:

```
	reserved-memory {
        #address-cells = <1>;
        #size-cells = <1>;
        ranges ;

        reserved: buffer@0x2000000 {
            no-map;
            reg = <0x2000000 0x4000000>;
        };
    };
```

What matters in the syntax above is the `reg` line, which reads like this: reserve `0x4000000` number of bytes starting from address `0x2000000`. This is on 32-bit. On 64-bit, there are 4 numbers, the first and third one the high part of the 64-bit number (so it looks something like this: `reg = <0 0x2000000 0 0x4000000>`). Correct values for `#address-cells` and `#size-cells` can be determined from the documentation of your SoC (or examples you find online/the provided examples of your platform).

At runtime, that simply divides the system RAM in 2 regions, and the OS can use either part but not the reservation that you made in the middle. You are guaranteed that, that's great. Even better, it's also excluded from tools like `htop`, as opposed to the CMA approach. If you check `/proc/iomem`, it looks something like this:

```
00000000-01ffffff : System RAM
  00008000-007fffff : Kernel code
  00900000-00951eef : Kernel data
06000000-1fffffff : System RAM
44a01000-44a0ffff : serial
...
```

And, how do you access that from the user space? Well, at this point, still with `/dev/mem`, so we only solved half of the problem. For the other half, we can employ a safe hack this time: overlay a `generic-uio` device over it (again, using the device tree), which is a simple driver which just maps memory to user space of a certain size. That size and physical address is, you guessed it, specified in the device tree. In the user land, it is accessible at `/sys/class/uio/...`.

Finally, the device tree additions are these:

```
	reserved-memory {
        #address-cells = <1>;
        #size-cells = <1>;
        ranges ;

        reserved: buffer@0x2000000 {
            no-map;
            reg = <0x2000000 0x4000000>;
        };
    };
	reserved_memory1@0x2000000 {
		compatible = "generic-uio";
		reg = <0x2000000 0x4000000>;
	};
```

From the user land, you access that memory like this:

```
int size;
int fd = open("/dev/uio0", O_RDWR);
char* virt_addr = mmap(NULL, size, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
strcpy(virt_addr, "Hello, world");
// ...
```

The code does not really need to have knowledge in advance about the physical address of the memory area or its size. These can be queried directly from properties exposed by the UIO device:

```
# cat /sys/class/uio/uio0/maps/map0/addr
# cat /sys/class/uio/uio0/maps/map0/size
```

And each uio has a `name` file associated which contains the name you specified in the device tree. You can read those for each folder in `/sys/class/uio` to learn which entry in there coresponds to which device you have set up in your device tree:

```
# cat /sys/class/uio/uio0/name
reserved_memory1
```

# Useful links

[Linux Reserved Memory](https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18841683/Linux+Reserved+Memory)

[Device tree Reserve Memory "Invalid reg property"](https://support.xilinx.com/s/question/0D52E00006hpK3vSAE/device-tree-reserve-memory-invalid-reg-property)
