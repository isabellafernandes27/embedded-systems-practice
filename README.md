# Overview of Embedded Systems

- [Overview of Embedded Systems](#overview-of-embedded-systems)
  - [TODOs](#todos)
  - [Hands-on Practice](#hands-on-practice)
    - [Plan](#plan)
    - [Setup for this project](#setup-for-this-project)
      - [Docker](#docker)
      - [Poky](#poky)
      - [Meta RaspberryPi](#meta-raspberrypi)
      - [Meta OpenEmbedded](#meta-openembedded)
      - [Ubuntu in Docker](#ubuntu-in-docker)
    - [Initializing Yocto](#initializing-yocto)
    - [Configuring the RaspberryPi](#configuring-the-raspberrypi)
    - [Configuring the Layers](#configuring-the-layers)
    - [Build a ```core-image-minimal```](#build-a-core-image-minimal)
  - [Troubleshooting](#troubleshooting)
    - [BitBake RasPi BSP was using 6.6 which no longer exists](#bitbake-raspi-bsp-was-using-66-which-no-longer-exists)


These are some of my notes about embedded systems to prepare for interviews. In here I review many topics including:

```text
                    Embedded Software
                           │
        ┌──────────────────┼──────────────────┐
        ↓                  ↓                  ↓
     Hardware           Memory            Concurrency
        │                  │                  │
   Registers            Pointers          Mutexes
   Bit masks            malloc            Semaphores
   volatile             alignment         Spinlocks
   Interrupts           padding           Race conditions
        │               endianness         Deadlocks
        │                  │               Priority inversion
        └──────────────┬───┴──────────────────┘
                       ↓
                  Ring buffers
                       │
                       ↓
                Producer/Consumer
```

This includes a hands-on project with a raspberryPi to put all these notes into practice. I had already used a Raspberry Pi in my senior design project, but I realized afterward that I was using it more like a desktop than an embedded platform. I wanted to go back and understand the layers underneath it — bootloader, kernel, device tree, root filesystem, drivers, and the build system — so I built a small Yocto image from scratch.

## TODOs

>[!IMPORTANT]
> maybe clean up and  MAKE THE DIRECTORIES LOOK LIKE THIS:
>
> ```text
>embedded-practice/
>├── day1-c/
>├── day2-linux/
>├── day3-boot-yocto/
>├── notes/
>└── yocto-pi/
>```

## Hands-on Practice

I will build a simple Yocto image for my Raspberry Pi 4 Model B.

```text
Raspberry Pi 4
      │
      ├── Yocto Linux
      │     ├── custom image
      │     ├── custom C application
      │     └── kernel module
      │
      └── GPIO
             │
        433 MHz RF
             │
       transmitter/receiver
```

### Plan 

Docker working → Yocto environment → build image → identify output → THEN carefully identify SD card → flash it.

This is the file structure:

```text
Mac:
~/Projects/embedded-practice/yocto-pi/
├── poky/                 ← cloned source
├── meta-raspberrypi/     ← cloned source
└── meta-openembedded/    ← cloned source

Docker Linux filesystem:
~/yocto-build/            ← actual Yocto build output
├── conf/
├── tmp/
├── downloads/
└── sstate-cache/
```
**Source layers are the ingredients; build/ is where we configure what we're building.**

The current LTS is Yocto 6.0 “Wrynose”, released May 2026, but for our hands-on exercise I want to use a stable, well-documented release/compatible Raspberry Pi layer rather than blindly using master. I think Scarthgap (Yocto 5.0 LTS) is the best choice for this practice project: it's still supported through April 2028, and the Raspberry Pi ecosystem has a well-established scarthgap branch.

### Setup for this project

#### Docker

Because I'm on an Apple Silicon Mac, we're going to run the Yocto build inside an ARM64 Linux container. However, Yocto builds are resource-hungry, so we should make sure Docker has enough resources.

In Docker Desktop, go to: Settings → Resources

I'll aim for approximately:
- CPU: 6–8 cores if available
- Memory: 8–12 GB
- Disk: at least 50 GB free

A full Yocto build can take a while and consume a lot of disk space, especially the first time.

#### Poky 

Poky is the Yocto Project's reference distribution and build environment. It includes BitBake and the core metadata needed to build an image. 

Installed it using: 

```bash
git clone -b scarthgap https://git.yoctoproject.org/poky.git
```
#### Meta RaspberryPi

We need to actually tell Yocto that we are building for a raspberryPi 4.

Installed this using:

```bash
git clone -b scarthgap https://github.com/agherzan/meta-raspberrypi.git
```

meta-raspberrypi is a BSP and contains Raspberry Pi-specific Yocto metadata such as:

- machine configurations
- Raspberry Pi kernel configuration
- Device Tree-related files/configuration
- bootloader configuration
- hardware-specific recipes
- Raspberry Pi-specific build settings

So we're basically telling Yocto:

> "Here's the generic embedded Linux build system. Here's everything you need to know about Raspberry Pi hardware."

meta-raspberrypi has dependencies on other OpenEmbedded layers. In particular, we'll need meta-openembedded.

#### Meta OpenEmbedded

Installed it with:

```bash
git clone -b scarthgap https://github.com/openembedded/meta-openembedded.git
```

#### Ubuntu in Docker

The Yocto documentation explicitly says that macOS should use an OCI container such as Docker as the build host. My Mac is the development machine, but I used an Ubuntu Docker container as the Yocto build host and targeted a Raspberry Pi 4.

```text
                    MY MAC
                       │
                 Docker Desktop
                       │
             ┌─────────▼─────────┐
             │   Ubuntu Linux    │
             │                   │
             │  Yocto / BitBake  │
             │        │          │
             └────────┼──────────┘
                      │
                ~/yocto-pi
                 ↙    ↓    ↘
              poky   meta-   meta-
                    open... raspberrypi
                      │
                      ▼
                Raspberry Pi 4
```

From ~/Projects/embedded-practice/yocto-pi run:
```bash
docker run --rm -it \
  -v "$(pwd):/yocto-pi" \
  ubuntu:24.04 \
  bash
```
A few things are happening here:

1. ubuntu:24.04 → gives us an actual Linux environment
2. -v "$(pwd):/yocto-pi" → shares your project directory with the container
3. --rm → the container itself disappears when we're done
4. -it → gives us an interactive terminal

The files stay on the Mac. The container is just our temporary Linux build machine. 

This will install a bare Ubuntu image, so we will still need to get some Linux packages installed so we can easily do this:

```bash
apt-get update
apt-get install -y \
  build-essential \
  chrpath \
  cpio \
  debianutils \
  diffstat \
  file \
  gawk \
  gcc \
  git \
  iputils-ping \
  libacl1 \
  liblz4-tool \
  locales \
  python3 \
  python3-pip \
  python3-pexpect \
  python3-subunit \
  socat \
  texinfo \
  unzip \
  wget \
  xz-utils \
  zstd \
  bzip2 \
  lz4 \
  rsync \
  sudo \
  vim \
  nano

# and then check a few things before moving on 
git --version
which chrpath gawk wget flock objcopy readelf
```

And then we also need to make a user since Yocto deliberately refuses to run BitBake as root.

```bash

useradd -m -s /bin/bash isabella

# Then give that user ownership of the project directory:
chown -R isabella:isabella /yocto-pi

# Then switch into the new user:\
su - isabella
```

macOS's default filesystem is case-insensitive, and Yocto refuses to put its tmp/ build directory there because that can cause subtle build corruption. So we need to move the build directory only into the Linux container's own filesystem. Your source repos can stay mounted from your Mac.

```bash
mkdir -p ~/yocto-build # This will be the new build directory
```
### Initializing Yocto

In the yocto-pi/poky directory run:

```bash 
source oe-init-build-env ~/yocto-build
```

This script is one of those things that looks like magic if you've never used Yocto. It actually does two things:

1. Sets up the shell environment so commands like bitbake are available.
2. Populates a build/ directory containing the configuration for this particular build.


### Configuring the RaspberryPi

Run ```grep -n "MACHINE" conf/local.conf``` and check which machine configuration is uncommented based on the initial Yocto setup. It should be **MACHINE = "raspberrypi4-64".** If it's not, run:

```bash 
sed -i '' 's/MACHINE ??= "qemux86-64"/MACHINE = "raspberrypi4-64"/' conf/local.conf
```

to change the machine.

### Configuring the Layers

First, check how the layers are already configured with: ```cat conf/bblayers.conf```. It should already know about the layers that came from Poky but not the metadata layers we brought in.

To add those layers, run:

```bash
bitbake-layers add-layer ../meta-openembedded/meta-oe
bitbake-layers add-layer ../meta-raspberrypi

# Then verify the layers with:
bitbake-layers show-layers

# You should see something like
layer                 path                                      priority
===========================================================================
core                  .../poky/meta                              5
yocto                  .../poky/meta-poky                        5
yoctobsp               .../poky/meta-yocto-bsp                   5
openembedded-layer    .../meta-openembedded/meta-oe             6
raspberrypi           .../meta-raspberrypi                      9
```

### Build a ```core-image-minimal```

To start the minimal image run:

```bash
# Check the machine target again
bitbake -e core-image-minimal | grep '^MACHINE='
# Start the build
bitbake core-image-minimal
```

> Think of it as: give me the **smallest practical Linux operating system image** that Yocto can build for my target hardware.


It is not the Linux kernel by itself. It's an entire bootable Linux system consisting roughly of:

```text
┌─────────────────────────────────────┐
│       Raspberry Pi 4                │
│                                     │
│  Bootloader                         │
│       ↓                             │
│  Linux Kernel                       │
│       ↓                             │
│  Device Tree                        │
│       ↓                             │
│  Root Filesystem                    │
│    ├── /bin                         │
│    ├── /sbin                        │
│    ├── /etc                         │
│    ├── /dev                         │
│    ├── /proc                        │
│    ├── /sys                         │
│    └── basic utilities              │
│                                     │
│  → boots into a minimal Linux       │
└─────────────────────────────────────┘
```
So when we eventually put the generated image on your SD card, the Pi should be able to actually boot Linux.

**Why "minimal"?**

Yocto gives you a bunch of predefined image recipes.

For example:
```text
Image	General idea
core-image-minimal	        Tiny basic Linux
core-image-full-cmdline	    More command-line tools
core-image-sato	            GUI + graphical desktop
core-image-weston	        Wayland graphical environment
```
We don't need a desktop environment for this project. So a minimal image is actually perfect.

>[!NOTE]
> I'm not building generic Linux. I'm telling Yocto "build Linux specifically for a 64-bit Raspberry Pi 4. BitBake, figure out everything necessary to construct core-image-minimal for my Raspberry Pi."


## Troubleshooting


### BitBake RasPi BSP was using 6.6 which no longer exists

The Yocto build initially failed during the kernel fetch. I traced the failure through the BitBake environment, identified that the Raspberry Pi BSP was defaulting to a 6.6 kernel revision that no longer existed upstream, and overrode the preferred kernel version to 6.12 in my build configuration.