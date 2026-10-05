# Interrupt Handlers

- [Interrupt Handlers](#interrupt-handlers)
  - [What an ISR does](#what-an-isr-does)
  - [Example](#example)
  - [Avoid in an ISR](#avoid-in-an-isr)
  - [Summary](#summary)

Hardware says that something needs attention, and the CPU temporarily stops normal execution to run a special function called an interrupt handler.

```text
Sensor
   │
   │ interrupt
   ↓
CPU
   │
   ├── pauses normal code
   │
   ↓
interrupt handler
   │
   └── handles event
   │
   ↓
resume normal code
```

## What an ISR does

An interrupt service routine (ISR) is meant to respond quickly to a hardware event.

- Keep it short and simple.
- Clear the interrupt flag.
- Set a flag or store a small amount of state.
- Do not perform long or blocking work inside the ISR.

```c
void sensor_isr(void)
{
    // handle sensor interrupt
    // keep it short and simple

    // NEVER malloc in an ISR
    // do not wait for a mutex or do complex operations here
}
```

## Example

```c
volatile bool data_ready = false;

void sensor_isr(void)
{
    clear_sensor_interrupt();
    data_ready = true;
}

int main(void)
{
    while (1) {
        if (data_ready) {
            data_ready = false;
            process_sensor_data();
        }
    }
}
```

This pattern is common in embedded systems: the ISR acknowledges the event and sets a flag, while the main loop handles the heavier processing.

## Avoid in an ISR

Avoid doing things like this inside an interrupt handler:

```c
void sensor_isr(void)
{
    uint8_t *data = malloc(128);
}
```

Instead, preallocate memory when possible:

```c
uint8_t sensor_data[128];
static uint8_t buffer[128];
```

The general rule is:

- ISR = fast acknowledgment and state update
- main loop = detailed processing and decision-making

## Summary

Interrupt handlers are designed for immediate response to hardware events, not for complex logic. Keeping them brief and predictable helps avoid timing issues, race conditions, and system instability.
