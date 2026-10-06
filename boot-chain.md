# Boot Chain

- [Boot Chain](#boot-chain)
  - [Summary](#summary)
  - [What happens when an embedded Linux board boots?](#what-happens-when-an-embedded-linux-board-boots)
  - [Bootloader (U-Boot)](#bootloader-u-boot)
  - [Linux Kernel](#linux-kernel)
  - [Device Tree](#device-tree)

## Summary

This is a simple flow based on everything else I have learned about embedded systems:

```text
                    HARDWARE
                        ↓
                    Power On
                        ↓
                    Boot ROM
                        ↓
                U-Boot (universal bootloader)
              ↙         ↓          ↘
          Kernel   Device Tree    bootargs
              \         |           /
                        ↓
                Linux Kernel
                        ↓
                Device Drivers
                        ↓
                Root Filesystem
                        ↓
                    User Space
                        ↓
                Application
```

Similar to U-boot, Yocto sits somewhere above/besides this image with:

```text
                Yocto / BitBake
                      │
       ┌──────────────┼──────────────┐
       ↓              ↓              ↓
   Kernel         Device Tree    Root FS
       │              │              │
       └──────────────┼──────────────┘
                      ↓
                 SD Card/Image
                      ↓
                 Raspberry Pi
```

## What happens when an embedded Linux board boots?

Imagine you plug your Raspberry Pi into power. What happens?

1. Power-on

    The CPU starts executing code from a location defined by the hardware. **But the Linux kernel isn't immediately running.**
    
    There's a boot chain.

2. Boot ROM

    The processor has some code built into the chip itself.

    This is often called: Boot ROM. Its job is basically:
    > "Where should I find the next piece of software?"

    Depending on the platform, it may look at flash/eMMC/SD/etc. and **load the next-stage bootloader.**


## Bootloader (U-Boot)

This is where the bootloader (U-Boot) comes in. Think of it as a bridge between the hardware and Linux.

It performs tasks such as:

- hardware initialization
- loading the Linux kernel
- loading the device tree
- setting kernel boot arguments
- selecting which kernel/configuration to boot
- sometimes networking/TFTP
- sometimes firmware updates/recovery

Eventually U-Boot says:

> "Here's the kernel, here's the hardware description, here's the boot configuration — go."

And transfers execution to Linux.

## Linux Kernel

Now Linux starts.

The kernel:

- initializes CPU/memory
- initializes subsystems
- initializes drivers
- discovers/configures hardware
- mounts the root filesystem
- starts the first userspace process

```text
                       kernel
                          ↓
                    init / systemd
                          ↓
                    user space
```

## Device Tree

The Linux kernel needs to know: What hardware actually exists on this particular board?

For example:

```text
This board has:
    SPI controller
    GPIO controller
    temperature sensor
    interrupt on GPIO 17
    sensor connected to SPI chip select 0
```
Rather than hard-coding all of that into the kernel, many ARM embedded systems use a Device Tree.

Conceptually:

```text
Device Tree
     ↓
"Here is the hardware configuration"
     ↓
Linux kernel
     ↓
drivers know what hardware to initialize
```

> [!NOTE]
> We've already learned compatible (what drivers can load this), interrupts, GPIOs (general purpose I/O), SPI (serial peripheral interface), chip select (master controller selects chip to talk to slave device), etc.


