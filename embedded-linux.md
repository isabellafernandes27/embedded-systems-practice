# Embedded Linux

- [Embedded Linux](#embedded-linux)
  - [User Space vs Kernel Space](#user-space-vs-kernel-space)
    - [User space](#user-space)
    - [Kernel space](#kernel-space)
    - [Why can't my application just access hardware?](#why-cant-my-application-just-access-hardware)
  - [System Calls](#system-calls)
  - [Device Drivers](#device-drivers)
    - [Example:](#example)
  - [/dev](#dev)
  - [ioctl()](#ioctl)
  - [Memory-Mapped I/O](#memory-mapped-io)
  - [Virtal Memory](#virtal-memory)
  - [Processes vs Threads](#processes-vs-threads)
    - [Process](#process)
    - [Threads](#threads)
  - [Embedded Linux Debugging](#embedded-linux-debugging)
    - [Useful Linux tools:](#useful-linux-tools)
  - [Cross-Compilation](#cross-compilation)
  - [Example 1](#example-1)
  - [Example 2](#example-2)
  - [Interrupts in Linux](#interrupts-in-linux)
    - [Top half](#top-half)
    - [Bottom half](#bottom-half)
  - [Example 3](#example-3)


When we're talking about embedded Linux, we're putting Linux onto a specialized piece of hardware:

```text
┌──────────────────────────────┐
│      Your application        │
│      C/C++ / Python          │
├──────────────────────────────┤
│       Linux user space       │
├──────────────────────────────┤
│          Linux kernel        │
├──────────────────────────────┤
│       Device drivers         │
├──────────────────────────────┤
│       Hardware / FPGA        │
├──────────────────────────────┤
│       CPU / SoC              │
└──────────────────────────────┘
```

The embedded software engineer's job is often sitting right at that boundary between software and hardware.

## User Space vs Kernel Space

Linux separates applications from the kernel.

### User space

- Python
- C++
- Java
- Your application

They don't have unrestricted access to the hardware.

### Kernel space

The Linux kernel runs here.

It has privileged access to things like:

- hardware
- memory management
- CPUs
- device drivers
- networking
- scheduling

Conceptually:

```text
             USER SPACE
┌───────────────────────────────┐
│ Application                   │
│                               │
│ "I want to read sensor data"  │
└───────────────┬───────────────┘
                │
                │ system call
                ↓
             KERNEL
┌───────────────────────────────┐
│ Device driver                 │
│ Memory management             │
│ Scheduler                     │
│ Networking                    │
└───────────────┬───────────────┘
                │
                ↓
            HARDWARE
```

### Why can't my application just access hardware?

If a normal program has:
```c
uint32_t *reg = (uint32_t *)0x40000000;
```

You could:

- corrupt hardware state
- overwrite kernel memory
- crash the system
- interfere with other processes

So Linux says:

> "Nope. If you want hardware, you go through the kernel."

## System Calls

This is how the application uses the kernel to talk to hardware, through system calls.

Some types of system calls are:

```c
read()
write()
open()
close()
mmap()
```

For example, if you want the kernel to interact with a device for the application:

```c
int fd = open("/dev/sensor", O_RDONLY);

read(fd, buffer, sizeof(buffer));

close(fd);
``` 

So, effectively, what is happening here is:

```text
Application
     │
     │ read()
     ↓
Linux kernel
     │
     ↓
sensor driver
     │
     ↓
hardware
```

## Device Drivers

A device driver is software that allows the operating system to communicate with a particular piece of hardware.
For example:

```text
Linux
  │
  ├── Ethernet driver
  ├── UART driver
  ├── SPI driver
  ├── I2C driver
  ├── GPIO driver
  └── custom spacecraft hardware driver
```
The application doesn't need to understand every hardware register. Instead, they do:

```text
Application
      ↓
generic interface
      ↓
Driver
      ↓
hardware registers
      ↓
Hardware
```

### Example:

Imagine a custom radio chip:

- The hardware has:
    
    ```c
    CONTROL register
    STATUS register
    DATA register
    ```
- The driver knows:
    ```c
    CONTROL = 0x40000000
    STATUS  = 0x40000004
    DATA    = 0x40000008
    ```
    It knows: which bits mean what, how to initialize the hardware, how to handle interrupts, how to read/write data, how to deal with errors, etc.
- The application simply says:
    > "Send this packet."
    
    And the driver handles all the ugly stuff.

## /dev

Linux represents many devices as files.

You might see:
```bash
ls /dev
```
and get things like:
```bash
/dev/ttyUSB0
/dev/i2c-1
/dev/spidev0.0
/dev/null
```

The application can interact with these through familiar file operations:
```c
open()
read()
write()
close()
```

This is part of Linux's famous:

> "Everything is a file"

philosophy.

Not literally everything, but many devices expose file-like interfaces. These are device nodes.

## ioctl()

Now imagine a device needs some operation that doesn't fit neatly into:

```c
read()
write()
```

For example:
>"Set the radio frequency to 2.4 GHz."

You don't necessarily want to encode that as writing arbitrary bytes.

Linux provides ioctl() for device-specific control operations.

Conceptually:
```c
ioctl(fd, SET_FREQUENCY, &frequency);
```

So, while read/write are for normal data transfer, ioctl is for device-specific commands/configuration.

## Memory-Mapped I/O

Hardware registers can be mapped into an address space.

For example:
```text
0x40000000 → CONTROL
0x40000004 → STATUS
0x40000008 → DATA
```
The CPU accesses them like memory:
```c
volatile uint32_t *status =
    (volatile uint32_t *)0x40000004;
```
But in Linux, the physical hardware address generally needs to be mapped appropriately into the kernel's virtual address space.

Conceptually:
```text
Physical hardware address
          ↓
     kernel mapping
          ↓
kernel virtual address
          ↓
       driver
```
**A user-space application normally doesn't just grab that physical address directly.**

## Virtal Memory

Linux applications generally see virtual addresses.

For example:
```text
Process A:

0x1000 → virtual address
          ↓
       physical RAM
          0x8F000

Process B might also have:

0x1000 → virtual address
          ↓
       completely different physical RAM
```

So two processes can both believe they have memory at **0x1000** without actually sharing the same physical memory.

This provides:

- isolation
- protection
- virtual memory
- easier process management

**The kernel manages the translation.**

## Processes vs Threads

### Process

A process is an independent running program with its own virtual address space.
```text
Process A
┌─────────────────┐
│ Code            │
│ Heap            │
│ Stack           │
│ Resources       │
└─────────────────┘

Process B:

Process B
┌─────────────────┐
│ Code            │
│ Heap            │
│ Stack           │
│ Resources       │
└─────────────────┘
```

They're isolated from each other.

### Threads

Threads exist within a process and share much of the process's resources.
```text
Process
│
├── Thread 1
├── Thread 2
└── Thread 3
```
**They share:**

- **code**
- **heap**
- **global variables**

**but each thread has its own:**

- **stack**
- **register state**
- **program counter**

So:

```text
Threads → easier communication
         BUT
         → shared memory
         → synchronization problems
```

## Embedded Linux Debugging

> "The device boots, but your application can't communicate with a sensor. How would you debug it?"

A good answer should be systematic:

```text
1. Is the hardware powered?
        ↓
2. Does Linux detect the device?
        ↓
3. Did the driver load?
        ↓
4. Are there kernel errors?
        ↓
5. Can I communicate with the device manually?
        ↓
6. Is the driver receiving interrupts?
        ↓
7. Is the application using the correct interface?
        ↓
8. Is the application interpreting the data correctly?
```

### Useful Linux tools:

1. dmesg - Kernel messages:
    ```bash
    dmesg | grep sensor

    driver failed to probe
    I2C error
    device not found
    interrupt failure
    ```
2. lsmod - See loaded kernel modules:
    ```bash
    lsmod
    ```
3. ps - See running processes:
    ```bash
    ps
    ```
4. top - See CPU/memory usage:
    ```bash
    top
    ```
5. gdb - Debug C/C++ programs.
6. strace - See system calls made by a user-space process:
    ```bash
    strace ./my_program

    For example, you might discover:

    open("/dev/sensor", ...)
    → ENOENT
    ```

## Cross-Compilation

You might develop on: Mac/PC, x86-64

but the target is: Xilinx SoC, ARM

You can't necessarily compile your program normally on your Mac and expect it to run on the target.

Instead:
```text
Development machine
        │
        │ cross-compiler
        ↓
ARM executable
        │
        ↓
Embedded target
```
For example:
```text
x86_64 machine
     ↓
aarch64-linux-gnu-gcc
     ↓
ARM64 binary
```

## Example 1

You have a Linux-based spacecraft computer.

There is a custom temperature sensor connected over SPI.

The hardware team says:

>"The sensor is powered and responding."

But your C application does:

```c
int fd = open("/dev/temp_sensor", O_RDONLY);

if (fd < 0) {
    perror("open");
    return 1;
}
```
and gets:
```c
open: No such file or directory
```
Walk through what you'd investigate.

Specifically:

1. What does /dev/temp_sensor represent?
   
    It is a device node, which provides a file-like interface to the underlying device/driver. It isn't literally the physical sensor itself. The flow of this is:
    ```text
    C application
        ↓
    open("/dev/temp_sensor")
        ↓
    Linux VFS (virtual file system)
        ↓
    temperature sensor driver
        ↓
    SPI subsystem (serial peripheral interface aka driver)
        ↓
    SPI controller
        ↓
    physical sensor
    ```

2. Is this primarily an application problem or could it be a driver/kernel problem?
   
   Since the hardware team has confirmed that the sensor is powered and responding, I'd work my way up the stack from the hardware interface through the kernel driver to the device node and finally the application. open() itself is a generic Linux system call. The device driver's implementation of the open operation is what ultimately handles the request, so I'd potentially investigate that.

   ```text
   /dev/temp_sensor
        ↓
    device node exists?
        ↓
    driver registered?
        ↓
    driver probed successfully?
        ↓
    SPI device configured?
    ```

3. What Linux commands/tools would you use?

    I'd check whether the device node exists with:
    ```c
    ls -l /dev/temp_sensor
    ```
    and then I'd check the kernel messages for anything to do with temperature or spi:
    ```c
    dmesg | grep -i temp
    dmesg | grep -i spi
    ```
    and also strace to see where things are breaking.

4. What would you want to know about the SPI driver?

5. Where does the device tree potentially enter the picture?

    The device tree is a description of the hardware that exists on the board, which Linux can use to determine how hardware should be configured and which driver should handle it. For example:

    ```text
    SPI controller
    │
    └── temperature sensor
          ├── SPI bus
          ├── chip select
          ├── interrupt
          └── compatible = "mycompany,temp-sensor"
    ```

Overall I'd debug this from the bottom of the software stack upward. First I'd verify that /dev/temp_sensor actually exists, because that's the device node exposed to user space. I'd check **dmesg** for driver probe or SPI errors and inspect whether the appropriate driver is loaded and registered. I'd use **strace** on the application to determine exactly where the system call is failing. If the driver isn't probing correctly, I'd investigate the device tree configuration, including the SPI controller, any required interrupts, or GPIOs (General purpose input/output). If the device node exists and opens successfully but reads fail, I'd continue debugging the driver and SPI communication rather than focusing on the application.

## Probe

When Linux discovers a device that matches a driver, the driver's probe function is called.

Conceptually:

```text
Device discovered
       ↓
Does "compatible" match a driver?
       ↓
      YES
       ↓
   driver probe()
       ↓
Initialize hardware
       ↓
Map registers
Configure interrupts
Initialize device
       ↓
Register device with Linux
       ↓
/dev/temp_sensor
```

So if someone tells you: "The driver's probe is failing."

**That means the driver has been found, but it couldn't successfully initialize/bind to the hardware.**

## Example 2

You have:
```text
Application
    ↓
/dev/temp_sensor
    ↓
Temperature driver
    ↓
SPI
    ↓
Sensor
```
The application successfully does:

```c
fd = open("/dev/temp_sensor", O_RDONLY);
````
So /dev/temp_sensor exists and open() succeeds.

But:
```c
read(fd, buffer, sizeof(buffer));
```
returns -1.

Walk through how you'd debug this.

Since open() succeeds, we've already established:

- the device node exists
- Linux can find it
- the driver is registered enough for open() to reach it
- the application's path is correct

Because open() succeeds, the driver has already successfully registered/bound enough for the device node to work. So I don't need to spend much time investigating /dev/temp_sensor itself anymore (no probing). Now read(fd, buffer, sizeof(buffer)) fails. That means I need to figure out: Does the failure happen inside the driver, during the SPI transaction, or because the sensor isn't returning valid data?

I would run 
```c
perror("read");
```
or inspect errno to know a little more about the error.

Then, I would run strace() to see how far the read command made it. Also, checking the kernel logs with dmesg() might provide some further information here.

I would suspect the driver here. I would look at the implementation of the driver's read() function and see where that might fail.

If a log message mentions anything to do with hardware, I would also check the hardware configuration and device tree:

```text
Device Tree

SPI controller
    │
    └── temp_sensor
          ├── chip-select = ?
          ├── max-frequency = ?
          ├── mode = ?
          └── compatible = ?
```

The debugging framework would be to find the lowest level I know is working and then work down from there:

```text
Application
    │
    │ read()
    ↓
Device node
    │
    ↓
Driver
    │
    ↓
Kernel subsystem
    │
    ↓
Hardware interface
    │
    ↓
Physical hardware
```

## Interrupts in Linux

```text
Sensor
   │
   │ "DATA READY!"
   ↓
Hardware interrupt
   ↓
Linux interrupt handling
   ↓
Driver
   ↓
Process data
```

### Top half

**The immediate interrupt handler does the minimum necessary work.**

Things that need to happen immediately:

- acknowledge interrupt
- capture essential state
- schedule further processing

### Bottom half

More expensive work happens later outside the immediate interrupt context.

Depending on the Linux mechanism, this might involve:

- threaded interrupts
- workqueues
- tasklets (historically/common concept, though modern kernel code has moved away from them)

## Example 3

You're writing a Linux device driver.

Your hardware generates an interrupt whenever a new sensor sample is available.

The interrupt handler needs to:

- Acknowledge the hardware interrupt.
- Read a small status register.
- Perform a complicated calculation on the sensor data.
- Write a large amount of data to disk.
- Sleep while waiting for another resource.

Which of these belong in the immediate interrupt handler, and which should be deferred?

Immediately, at the hardware interrupt level, you should only need to acknowledge the interrupt and read a small status register for that interrupt status. Any complicated calcs or anything that would take a large amount of disk space and CPU, should be deferred to the linux interrupt handler.