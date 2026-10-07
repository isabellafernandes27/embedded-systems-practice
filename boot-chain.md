# Boot Chain

- [Boot Chain](#boot-chain)
  - [Summary](#summary)
  - [What happens when an embedded Linux board boots?](#what-happens-when-an-embedded-linux-board-boots)
  - [Boot ROM](#boot-rom)
  - [Bootloader (U-Boot)](#bootloader-u-boot)
  - [Linux Kernel](#linux-kernel)
  - [Device Tree](#device-tree)
  - [Root File System](#root-file-system)
  - [Yocto](#yocto)
    - [BitBake](#bitbake)
    - [Recipe](#recipe)
    - [Layer](#layer)
    - [BSP](#bsp)
    - [Image](#image)
    - [Yocto Overall](#yocto-overall)
  - [Sample Questions](#sample-questions)

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

## Boot ROM

When the processor first powers on, Linux isn't running.

The CPU needs some initial code to execute, and processors typically have a small piece of code permanently stored in ROM (read-only memory).

That's the Boot ROM.

Its job is platform-specific, but conceptually:

> Find/load the next stage of the boot process and start executing it.

For example, it might determine whether to boot from:

- eMMC
- SD card
- SPI flash
- USB
- network
- another boot source

> [!NOTE]
> Boot ROM is hardware/vendor-provided and isn't something you're normally rebuilding as part of your Linux image.

## Bootloader (U-Boot)

This is where the bootloader (U-Boot) comes in. Think of it as a bridge between the hardware and Linux. It launches Linux.

It performs tasks such as:

- hardware initialization
- loading the Linux kernel 
- loading the device tree 
- setting kernel boot arguments
- selecting which kernel/configuration to boot
- sometimes networking/TFTP
- sometimes firmware updates/recovery

For example, conceptually:
```text
SD card
 ├── bootloader
 ├── kernel
 └── device tree
```
U-Boot might load:
```text
kernel      → RAM
device tree → RAM
```
U-Boot can also pass boot arguments to the kernel. SO basically:

```text
                 U-Boot
                   │
       ┌───────────┼───────────┐
       ↓           ↓           ↓
    Kernel     Device Tree   bootargs
       │           │           │
       └───────────┼───────────┘
                   ↓
                Linux
```

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

And importantly, the kernel needs to know: **What hardware is actually on this board?**

## Device Tree

The Linux kernel needs to know: What hardware actually exists on this particular board?

For example:

```text
SPI controller
    |
    └── temperature sensor
          ├── chip select = 0
          ├── SPI frequency = 10 MHz
          ├── interrupt = GPIO 17
          └── compatible = "vendor,temp123"
```
Rather than hard-coding all of that into the kernel, many ARM embedded systems use a Device Tree to know how the hardware is configured and which driver to bind to it.

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


## Root File System

Linux also needs somewhere to get:

- /bin
- /etc
- /lib
- /usr
- applications
- configuration
- shared libraries
- device nodes (/dev)

Basically that's the root filesystem.

For example:

```text
/
├── bin/
├── dev/
├── etc/
├── home/
├── lib/
├── proc/
├── sys/
└── usr/
```

And this is one of Yocto's major jobs: it builds the complete embedded Linux system/root filesystem for the target.

Eventually the kernel starts the first userspace process, often something like:
```bash
systemd
```
And now you would be in the user space.

## Yocto

**Yocto is a build framework/project for creating customized Linux distributions/images for embedded systems.** Kinda like a dockerfile from what I understand. Yocto gives you a reproducible way to build a customized embedded Linux image.

Think:
```text
Ubuntu
    ↓
a Linux distribution (image)

Yocto
    ↓
a system for BUILDING your own embedded Linux image
```
You tell Yocto:

> "I have this hardware. I want these packages, this kernel, these drivers, these applications, these configurations..."

And Yocto builds an image.

```text
            Your configuration
                    │
                    ↓
              ┌───────────┐
              │   Yocto   │
              └─────┬─────┘
                    │
        ┌───────────┼───────────┐
        ↓           ↓           ↓
      Kernel    Applications   Libraries
        │           │           │
        └───────────┼───────────┘
                    ↓
              Root Filesystem
                    │
                    ↓
              Bootable Image
```

### BitBake

The Yocto build engine. Kinda like running docker build, npm install, or make.

It reads recipes and determines:
```text
        what needs to be built
                ↓
        dependencies
                ↓
        how to compile it
                ↓
        how to package it
                ↓
        how to assemble the final image
```

> "Yocto is the overall project/framework; BitBake is the engine doing the work. It compile, installs, and packages your project into an image."

### Recipe

A ```.bb``` file describing how to build/install something. This is like the actual Dockerfile.

For example, myapp.bb might say: here's my source code, compile it with gcc, and install the resulting executable to /usr/bin.

A recipe is like:

```text
Where is the source?
        ↓
How do I compile it?
        ↓
Where should the resulting files go?
        ↓
What does it depend on?
```

### Layer

A collection of related metadata/recipes/configuration. This is still like the Dockerfile kinda but also the device tree.

Think:
```text
meta-raspberrypi/
    ↓
Raspberry Pi-specific knowledge

meta-myproject/
    ↓
YOUR project's customizations
```

Each layer can contain:

- recipes
- configuration
- patches
- machine definitions
- classes
- other metadata

### BSP

Board Support Package. This is the collection of software/configuration necessary to support a particular hardware platform.

For example, meta-raspberrypi gives Yocto knowledge/configuration for Raspberry Pi hardware. Like a device tree.

In Yocto, this can be provided through BSP layers containing things like machine configuration, kernel configuration and patches, device-tree files, bootloader configuration, and hardware-specific recipes.

> Think: "What does Linux/Yocto need to know to work on this particular board?"

```text
 ┌─────────────────────────────────────────────────────────┐
 │ 4. Application Layer (meta-my-custom-app)               │ <-- Your high-level software
 ├─────────────────────────────────────────────────────────┤
 │ 3. UI/Middleware Layer (meta-qt5 / meta-python)         │ <-- Higher-level libraries & frameworks
 ├─────────────────────────────────────────────────────────┤
 │ 2. BSP Hardware Layer (meta-raspberrypi / meta-intel)   │ <-- The BSP (Drivers, Kernel, Bootloader)
 ├─────────────────────────────────────────────────────────┤
 │ 1. Core Yocto Layer (meta)                              │ <-- The absolute baseline Linux skeleton
 └─────────────────────────────────────────────────────────┘
```

>![CAUTION]
> This is NOT a collection of layers!! It is the board-specific software and configuration needed to support a specific hardware platform.

### Image

The final thing Yocto builds.

You can customize it with your own:

- applications
- kernel
- drivers
- configuration
- packages

### Yocto Overall

```text
                    YOCTO
                      │
                  BitBake
                      │
          ┌───────────┴───────────┐
          ↓                       ↓
       Layers                  Config
          │
     ┌────┴─────┐
     ↓          ↓
 Recipes      BSPs
     │
     ↓
 Applications / Drivers / Config
     │
     └──────────┬──────────┘
                ↓
              IMAGE
                ↓
        Bootable Linux system
```

And the hierarchy is: 

```text
                    Yocto
                      │
               ┌──────┴──────┐
               ↓             ↓
            Layers        Configuration
               │
       ┌───────┴────────┐
       ↓                ↓
    Recipes          Hardware/BSP
       │
       ↓
   Individual
   components
       │
       ↓
     BitBake
       │
       ↓
     Build
       │
       ↓
     Image
```


## Sample Questions
1. Walk me through what happens when an embedded Linux system boots, from power-on until a user application can run. Answer:

    When the system powers on, the CPU begins executing code from a vendor-provided Boot ROM (read-only memory). The Boot ROM performs the initial bootstrapping and determines where to load the next stage of the boot process. Typically, that's a bootloader such as U-Boot (universal bootloader).

    U-Boot prepares the system to launch Linux (like a bridge between hardware and linux). It can initialize hardware needed for boot and loads the Linux kernel, Device Tree, and boot arguments into memory. Once everything is ready, U-Boot transfers control to the Linux kernel.

    The kernel then initializes the CPU, memory management, interrupts, and drivers, uses the Device Tree to understand the hardware configuration, mounts the root filesystem, and starts the first userspace process. From there, userspace services and applications can run.
2. How is Yocto different from Ubuntu?
    Ubuntu is a pre-built Linux distribution that you install and then customize. Yocto is a build framework for creating a customized embedded Linux image. With Yocto, you can define the exact packages, kernel configuration, drivers, applications, and other components you want, and generate a reproducible image for your target hardware.
