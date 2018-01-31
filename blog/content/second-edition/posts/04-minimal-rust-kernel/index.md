+++
title = "A Minimal Rust Kernel"
order = 4
path = "minimal-rust-kernel"
date  = 0000-01-01
template = "second-edition/page.html"
+++

In this post we create a minimal 64-bit Rust kernel. We built upon the [freestanding Rust binary] from the previous post to create a bootable disk image, that prints something to the screen.

[freestanding Rust binary]: ./second-edition/posts/03-freestanding-rust-binary/index.md

<!-- more -->

TODO github, issues, comments, etc

## The Boot Process
When you turn on a computer, it begins executing firmware code that is stored in motherboard [ROM]. This code performs a [power-on self-test], detects available RAM, and pre-initializes the CPU and hardware. Afterwards it looks for a bootable disk and starts booting the operating system kernel.

[ROM]: https://en.wikipedia.org/wiki/Read-only_memory
[power-on self-test]: https://en.wikipedia.org/wiki/Power-on_self-test

On x86, there are two firmware standards: the “Basic Input/Output System“ (**[BIOS]**) and the newer “Unified Extensible Firmware Interface” (**[UEFI]**). The BIOS standard is old and outdated, but simple and well-supported on any x86 machine since the 1980s. UEFI, in contrast, is more modern and has much more features, but is more complex to set up (at least in my opinion).

[BIOS]: https://en.wikipedia.org/wiki/BIOS
[UEFI]: https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface

Currently, we only provide BIOS support, but support for UEFI is planned, too. If you'd like to help us with this, check out the [Github issue](https://github.com/phil-opp/blog_os/issues/349).

### BIOS Boot
Almost all x86 systems have support for BIOS booting, even newer UEFI-based machines (they include an emulated BIOS). This is great, because you can use the same boot logic across all machines from the last centuries. But this wide compatibility is at the same time the biggest disadvantage of BIOS booting, because it means that the CPU is put into a 16-bit compability mode called [real mode] before booting so that that arcane bootloaders from the 1980s would still work.

But let's start from the beginning:

When you turn on a computer, it loads the BIOS from some special flash memory located on the motherboard. The BIOS runs self test and initialization routines of the hardware, then it looks for bootable disks. If it finds one, the control is transferred to its _bootloader_, which is a 512-byte portion of executable code stored at the disk's beginning. Most bootloaders are larger than 512 bytes, so bootloaders are commonly split into a small first stage, which fits into 512 bytes, and a second stage, which is subsequently loaded by the first stage.

The bootloader has to determine the location of the kernel image on the disk and load it into memory. It also needs to switch the CPU from the 16-bit [real mode] first to the 32-bit [protected mode], and then to the 64-bit [long mode], where 64-bit registers and the complete main memory are available. Its third job is to query certain information (such as a memory map) from the BIOS and pass it to the OS kernel.

[real mode]: https://en.wikipedia.org/wiki/Real_mode
[protected mode]: https://en.wikipedia.org/wiki/Protected_mode
[long mode]: https://en.wikipedia.org/wiki/Long_mode
[memory segmentation]: https://en.wikipedia.org/wiki/X86_memory_segmentation

Writing a bootloader is a bit cumbersome as it requires assembly language and a lot of non insightful steps like “write this magic value to this processor register”. Therefore we don't cover bootloader creation in this post and instead provide a tool named [bootimage] that automatically appends a bootloader to your kernel.

[bootimage]: https://github.com/phil-opp/bootimage

If you are interested in building your own bootloader, check out our “[Booting]” posts, where we explain in detail how a bootloader is built.

[Booting]: TODO

### The Multiboot Standard

### UEFI

## A Minimal Kernel
Now that we know how a computer boots, it's time to create our own minimal kernel. Our goal is to create a bootable disk image that prints a “Hello World!” to the screen when booted. For that we build upon the [freestanding Rust binary] we created in the previous post.

We already have our `_start` entry point, which will be called by the boot loader. So let's output something to screen from it.

### Printing to Screen

- VGA buffer
    - format
    - at 0xb8000
- Rust code
    - Hello World or just OK?

### Creating a Bootimage

- bootimage tool

## Booting it!

- qemu
- bochs? virtualbox?
- makefile? cargo-make?

## Summary & What's next?
