# Memory and Registers in Embedded C

- [Memory and Registers in Embedded C](#memory-and-registers-in-embedded-c)
  - [Registers and memory management in C](#registers-and-memory-management-in-c)
  - [Pointers and storage duration](#pointers-and-storage-duration)
  - [Registers vs. RAM](#registers-vs-ram)
    - [CPU registers](#cpu-registers)
    - [Peripheral registers](#peripheral-registers)
  - [Memory-mapped I/O](#memory-mapped-io)
    - [Example: control register](#example-control-register)
  - [Program memory layout](#program-memory-layout)
  - [const and volatile qualifiers](#const-and-volatile-qualifiers)
  - [Example: creating and using pointers](#example-creating-and-using-pointers)
  - [Pointer arithmetic and alignment](#pointer-arithmetic-and-alignment)
  - [Struct sizing and padding](#struct-sizing-and-padding)
  - [Summary](#summary)
  - [Endianness](#endianness)


This page summarizes the key ideas around memory management, pointers, registers, and memory-mapped I/O in embedded systems.

## Registers and memory management in C

In C, memory management is usually handled manually with pointers and functions such as `malloc` and `free`.

Registers are special hardware storage locations used by the CPU or peripherals for fast access and control.

```c
volatile uint32_t *status = (volatile uint32_t *)0x40000000;
```

Here:

- `status` is the address of the register.
- `*status` is the value stored at that address.
- `volatile` tells the compiler that the value may change at any time outside the program's normal flow.

This is especially important for hardware registers because they can change without the C code writing to them directly.

## Pointers and storage duration

Local variables created inside a function have automatic storage duration and are typically stored on the stack.

```c
void foo(void) {
    int x = 42;
}
```

When `foo()` returns, `x` no longer exists.

Returning a pointer to a local variable is a bug:

```c
int *foo(void) {
    int x = 42;
    return &x;   // BAD: pointer dangles after the function returns
}
```

A valid pattern is to allocate memory dynamically:

```c
int *foo(void) {
    int *x = malloc(sizeof(int));
    *x = 42;
    return x; // GOOD: memory remains allocated until free(x)
}
```

When you are done with it, free the memory and set the pointer to `NULL`.

```c
free(x);
x = NULL;
```

## Registers vs. RAM

A register is a hardware-visible storage location used to control or observe the CPU or a peripheral.

```text
CPU
  │
  ├── General-purpose registers
  │   ├── R0
  │   ├── R1
  │   └── ...
  │
  └── Memory bus
      │
      ├── RAM
      │   └── variables, stack, heap, buffers
      │
      └── Peripheral registers
          ├── STATUS
          ├── CONTROL
          └── DATA
```

### CPU registers

These are used for:

- arithmetic
- temporary values
- stack pointer
- program counter
- function arguments

### Peripheral registers

These are used to interact with hardware such as:

- GPIO
- UART
- SPI
- timers
- ADCs
- radios
- sensors
- FPGA interfaces

## Memory-mapped I/O

A memory-mapped I/O register is a hardware register accessed through a specific address.

```c
#define STATUS_REG (*(volatile uint32_t *)0x40000000)
```

This means:

- interpret `0x40000000` as a pointer
- treat the memory at that address as a `uint32_t`
- read it as volatile so the compiler does not optimize away reads

This is how embedded code often interacts with hardware registers.

### Example: control register

```c
#define CONTROL_REG (*(volatile uint32_t *)0x40001000)

CONTROL_REG |= (1 << 0);   // enable
CONTROL_REG |= (1 << 2);   // transmit
CONTROL_REG &= ~(1 << 1);  // clear reset bit
```

These are read-modify-write operations.

Some registers have special behavior. For example, a datasheet might say that writing a `1` to a bit clears the corresponding interrupt flag. This is often called a W1C (write-one-to-clear) register.

```c
INT_STATUS = (1 << 1); // clear DATA_READY interrupt without touching other flags
```

## Program memory layout

A program is typically laid out like this:

```text
┌──────────────────────────┐
│ Code / text              │
│ foo(), main(), etc.      │
├──────────────────────────┤
│ Global / static data     │
│ global = 10              │
├──────────────────────────┤
│ Heap                     │
│ malloc'd memory          │
│         ↑ grows          │
├──────────────────────────┤
│ Stack                    │
│ local variables          │
│ function call frames     │
│         ↓ grows          │
└──────────────────────────┘
```

The exact layout and direction of heap/stack growth depends on the system and compiler.

## const and volatile qualifiers

Understanding `const` and `volatile` is important when working with registers.

```c
const int *const p = &x;  // both pointer and value are constant
int *const p = &x;        // pointer is constant, value can change
const int *p = &x;        // value is constant, pointer can change
```

Rule of thumb:

- `const` nearest to `*` means the pointer itself is constant.
- `const` on the other side means the pointed-to value is constant.

```c
const volatile uint32_t *status;
```

This means:

- the value may change unexpectedly
- the compiler must read it each time
- the code is not allowed to modify the value through that pointer

This pattern is common for read-only hardware registers.

## Example: creating and using pointers

```c
#include <stdio.h>
#include <stdlib.h>

int *create_value(void)
{
    int x = 42;
    return &x; // dangling pointer
}

int *create_value_heap(void)
{
    int *x = malloc(sizeof(int));
    *x = 42;
    return x;
}

int main(void)
{
    int *a = create_value();
    int *b = create_value_heap();

    printf("%d\n", *a); // undefined behavior
    printf("%d\n", *b); // 42

    free(b);
    return 0;
}
```

The first pointer is invalid because it points to a local variable that no longer exists after the function returns.

## Pointer arithmetic and alignment

Pointer arithmetic moves based on the pointer type.

```c
int arr[4] = {10, 20, 30, 40};
int *p = arr;
int *q = p + 1;
int x = *(p + 2); // x = 30
```

For hardware addresses:

```c
uint8_t *p = (uint8_t *)0x40000000;
uint8_t *q = p + 1;  // moves by 1 byte

uint32_t *r = (uint32_t *)0x40000000;
uint32_t *s = r + 1;  // moves by 4 bytes
```

Alignment matters. A 32-bit value should generally be aligned to a 4-byte boundary.

```c
uint32_t *p = (uint32_t *)0x40000001; // potentially misaligned access
```

If the hardware requires N-byte alignment, the address should be a multiple of `N`.

## Struct sizing and padding

Struct layout can include padding between fields to satisfy alignment requirements.

```c
struct Sensor {
    uint8_t status;  // 1 byte
    uint32_t value;  // 4 bytes
    uint16_t count;  // 2 bytes
};
```

`sizeof(struct Sensor)` may be larger than the sum of the fields because of padding.

```text
0 ─────── status
1 ─────── padding
2 ─────── padding
3 ─────── padding
4 ─────── value
```

This is important in embedded systems where memory layout and register mappings directly affect correctness.

## Summary

The key ideas are:

- local variables die when the function returns
- heap memory must be managed explicitly
- registers are hardware-owned and often accessed via pointers
- `volatile` prevents unsafe optimization around hardware state
- alignment and padding matter for memory correctness
- dangling pointers and misaligned accesses can cause undefined behavior

These concepts are fundamental to embedded programming and safe low-level C design.

```c
// 5
// 6
// 7
// 8 ─────── count
// 9
// 10 ────── padding
// 11 ────── padding

int arr[5]; // int = 4 bytes
int *p = arr;  // pointer = 8 bytes
int x = sizeof(arr); //= 20
int y = sizeof(p);   //= 8
```


## Endianness

Endianness is how bytes are stored in memory:
```c
uint32_t x = 0x12345678;
// a uint32_t occupies four separate bytes in memory: 12 24 56 78
// The question is: Which byte goes at the lowest memory address?
```

In memory, this could be stored as:
Little-endian: 78 56 34 12 --> The least significant byte goes at the lowest address.
Big-endian:    12 34 56 78 --> The most significant byte goes at the lowest address.

Why do we care? Because your system may communicate with another system using a different byte ordering.