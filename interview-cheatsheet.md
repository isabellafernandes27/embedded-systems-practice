# Interview Cheatsheet

## 1. My 30-Second Story

- Computer Engineering → aerospace software → embedded systems.
- Mines senior design: Raspberry Pi + Arduino + sensors + OpenCV for simulated space-debris detection/detumbling.
- Professional SWE: backend Java/C++ + AWS/infrastructure.
- Now pursuing Space Systems Engineering at JHU.
- Why embedded: I realized I had used the Pi more like a tiny desktop; I wanted to understand the layers underneath—bootloader, kernel, device tree, drivers, Linux image/build system, and hardware interfaces.
- Why CesiumAstro: software directly interacts with real spacecraft/RF hardware; **intersection between space, hardware, and software**; phased-array communications make the embedded layer especially interesting.

## 2. C / Memory — My Biggest Review Area

### Pointer / memory basics

- `int *p` = pointer storing an address.
- `*p` = dereference → access the value at that address.
- `&x` = address of `x`.
- `malloc()` → heap memory; must be paired with `free()`.
- `realloc()` → use a temporary pointer so failure does not lose the original allocation.
- Stack = automatic/local memory, typically limited.
- Heap = dynamic lifetime memory.
- `sizeof(array)` = size of the whole array.
- `sizeof(pointer)` = pointer size.

### `volatile`

Use `volatile` for values that can change outside normal program flow, such as:

- memory-mapped hardware registers
- variables modified by an ISR
- some shared hardware state

Important: `volatile` does not make code thread-safe or atomic.

### Memory-mapped register

```c
volatile uint32_t *status = (volatile uint32_t *)0x40000000;
uint32_t v = *status;
```

- The address refers to hardware, not normal RAM.

### Bit operations

```c
x |=  (1u << n);   // set
x &= ~(1u << n);   // clear
x ^=  (1u << n);   // toggle
if (x & (1u << n)) // check
```

- W1C interrupt flag: write 1 to clear the corresponding bit. Do not blindly do read-modify-write.

### Alignment / padding

- Struct fields may have padding so members are naturally aligned.
- `sizeof(struct)` can therefore exceed the sum of member sizes.

### Endianness

- Little endian: least-significant byte at lowest address.
- Big endian: most-significant byte at lowest address.

## 3. Interrupts / Concurrency

### ISR rules

Do:

- acknowledge/clear the interrupt
- capture minimal data
- signal/defer work

Don't:

- block
- sleep
- do heavy computation
- hold a mutex unnecessarily

### Mutex vs semaphore vs spinlock

- Mutex: ownership; one thread at a time; can sleep/block.
- Semaphore: counting/signaling; useful for producer-consumer/resource availability.
- Spinlock: busy-waits; useful for very short critical sections or contexts where sleeping is not allowed.

### Race condition

- Two execution contexts access shared state and the result depends on timing.

### Deadlock

Classic conditions:

- mutual exclusion
- hold-and-wait
- no preemption
- circular wait

Prevention:

- consistent lock ordering
- minimize lock scope
- avoid nested locks where possible

### Producer-consumer

- Producer puts work/data into a shared buffer; consumer removes it.
- Typical tools: a mutex protects the buffer, and semaphores/condition variables signal available items/space.

### Priority inversion

- High-priority task waits on a lock held by low-priority task while a medium-priority task keeps running.
- Solution: priority inheritance/ceiling.
- Mars Pathfinder: priority inversion caused system resets; priority inheritance was used to fix it.

### Ring buffer

- Fixed-size circular queue using head/tail indices.
- Great for UART/RF/ISR → main-thread data flow because it avoids repeated allocation.

## 4. Embedded Linux

### User vs kernel space

- User space: applications, limited hardware access.
- Kernel space: drivers, scheduling, memory management, direct hardware control.
- System calls are the controlled boundary.

### Device driver

- Software layer translating generic OS requests into hardware-specific operations.

Typical flow:

- Device Tree → driver matching → `probe()` → map/configure hardware → register interfaces → use device

### Device Tree

- Describes hardware to Linux: addresses, interrupts, buses, GPIOs, compatible strings, etc.
- Lets the same kernel support different boards/configurations without hard-coding everything.

### Debugging commands

```bash
dmesg
lsmod
modprobe <module>
ps
top
strace <program>
gdb <program>
```

## 5. Boot Flow — Say This Cleanly

`Boot ROM → bootloader (U-Boot) → Linux kernel + Device Tree + bootargs → drivers initialize → root filesystem → userspace/application`

- Bootloader: initializes enough hardware to load/start the kernel and passes DT/boot arguments.
- Kernel: manages CPU, memory, processes, drivers, and hardware.
- Rootfs: userspace filesystem containing libraries, binaries, configs, etc.

### Board bring-up

Start low-level and work upward:

- power/clock → bootloader → UART → DDR → kernel → DT → drivers → peripherals → application

When debugging: first determine which layer is actually failing.

## 6. Yocto — Know This Story

- Yocto is a build system/ecosystem for creating customized Linux distributions for embedded devices.
- BitBake: executes recipes/tasks and resolves dependencies.
- Recipe: instructions for fetching/configuring/compiling/installing a component. -> like Dockerfiles
- Layer: organized collection of recipes/configuration; enables customization without modifying core layers. -> kinda like Terraform Modules
- BSP: board-specific support/configuration.
- Image: assembled Linux system (kernel + rootfs + required components/artifacts).

### My project

- Raspberry Pi 4 (64-bit)
- Yocto Scarthgap
- `meta-raspberrypi`
- custom layer
- custom C application recipe
- kernel module/character driver
- device-tree or kernel-config modification

Strong talking point:

> I hit a real source-fetch issue, traced it through BitBake’s fetch/mirror configuration, identified a malformed fallback mirror rule, removed it, cleaned the stale Git cache, and successfully fetched the kernel.

### Mirrors

- `PREMIRRORS`: alternate sources tried before the original.
- `MIRRORS`: fallback sources tried after the original fails.
- Useful for internal caching, reliability, reproducibility, and controlled builds.

## 7. Hardware / Debugging Mindset

If hardware isn't working:

- Is it powered correctly?
- Correct voltage levels?
- Wiring/pinout correct?
- Is the peripheral visible to Linux?
- Is the driver loaded?
- Does `dmesg` show errors?
- Can I isolate/test one layer at a time?
- Is the problem hardware, driver, kernel/DT, or application?

Never jump straight to rewriting application code.

## 8. Talking Points

- Phased array: many antenna elements whose relative phase is controlled so signals reinforce in desired directions and cancel elsewhere → electronically steerable beams without mechanical movement.

### Why I want this role

> “What really interests me is the boundary between software and physical spacecraft hardware. I like that the software here isn't just moving data through a backend—it is configuring, controlling, and monitoring systems that have to work reliably in the real world.”

### If asked about gaps

> “My recent professional experience has been more backend and infrastructure focused, so I don't want to overstate my embedded experience. But my Computer Engineering background gave me the fundamentals, my senior design gave me hands-on hardware experience, and I've been deliberately closing the Linux/embedded gap. I learn quickly and I really enjoy understanding what's happening underneath the abstraction.”

### x.509 auth in EKS

> "The data delivery service needed to make sure only authorized internal clients could call it, so I designed certificate-based authentication at the load balancer level in our EKS cluster: each client presented an x509 certificate, and the load balancer validated it against a security policy before traffic reached the service. The hardest part was that this was my first real exposure to EKS and load balancers. I understood x509 certificates conceptually, but I’d never had to wire certificate validation into a load balancer before, so I had to learn both pieces and how they fit together. I did a lot of self-study, and leaned on a Kubernetes expert on my team when I got stuck. That experience is actually what pushed me to go get my AWS Solutions Architect certification afterward, I wanted to solidify what I’d learned the hard way."

### Disagreement with Teammate

> "At ForeFlight, I owned the data egress layer, which handled database connectivity, separate from the delivery layer, which handled order configuration logic. A teammate wanted order parameter validation to live in my egress layer, and I disagreed, I felt the delivery side, which actually understood what valid parameters looked like, should own that logic. This came up during a period when we didn’t have a project manager, so there were no written requirements, which was really the root problem: we each had assumptions about what the other’s application should handle. So I used a standup to propose we actually write requirements together instead of debating it ad hoc. Once we did, it became clear: some validation did make sense at the egress layer, but most belonged in delivery. Everyone agreed once we had it written down, and it fixed not just that one disagreement but gave us a shared reference going forward."

### Embedde 1 Note

> “Just to be transparent, I really like the company and I am very excited about this opportunity, so I also applied for the Embedded Software Engineer I position here that was posted recently.”