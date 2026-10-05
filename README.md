# Overview of Embedded Systems

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