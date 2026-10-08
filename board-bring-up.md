# Board Bring Up

- [Board Bring Up](#board-bring-up)
  - [Bring-up progression](#bring-up-progression)
  - [Practice Questions:](#practice-questions)

## Bring-up progression

```text
             NEW BOARD
                 │
                 ↓
          Power / clocks / reset
                 │
                 ↓
            CPU executes?
                 │
                 ↓
             Bootloader
                 │
                 ↓
              Linux
                 │
                 ↓
           Device Tree
                 │
                 ↓
             Drivers
                 │
                 ↓
        Peripherals work?
                 │
                 ↓
           Userspace
                 │
                 ↓
          Applications
```

The key principle: Get the lowest layer working before debugging the layer above it.

> [!IMPORTANT]
> If the CPU isn't even executing code, don't debug your application.
> If Linux isn't booting, don't debug your SPI sensor application.
> If the driver isn't probing, don't debug the application calling /dev/my_sensor.


## Practice Questions:

1. You've been given a brand-new custom board. You power it on, but nothing appears to be working. How would you approach bringing the board up?
    Find the lowest layer that is known to be working, then investigate the next layer.
    
    Powering on doesn't necessarily mean the CPU is actually executing correctly. Absolutely nothing appears, so I'd investigate:

    - Is the board actually powered?
    - Are voltage rails correct?
    - Are the clocks running?
    - Is the CPU coming out of reset?
    - Is the boot source correct?
    - Is the UART connected/configured correctly?
    - Can you get any evidence that Boot ROM/U-Boot is executing?
   
    I don't start debugging Linux because I don't know that Linux is even being reached. Summary:

    I'd debug from the lowest layer upward rather than immediately assuming it's a software problem. I'd first verify power, clocks, reset, and that the CPU is actually executing. Then I'd verify the bootloader and use the serial console to determine how far through the boot process we're getting. If U-Boot works but Linux doesn't, I'd investigate the kernel image, boot arguments, Device Tree, and kernel logs. Once Linux is running, I'd verify that the relevant drivers are enabled and probing correctly, then investigate Device Tree configuration and hardware communication. Finally, I'd move into userspace and debug the application-to-driver interface. The goal is to identify the first layer where behavior diverges from what we expect.
