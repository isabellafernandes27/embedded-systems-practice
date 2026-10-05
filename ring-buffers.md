# Ring Buffers

- [Ring Buffers](#ring-buffers)
  - [Basic idea](#basic-idea)
  - [Write to the ring buffer](#write-to-the-ring-buffer)
  - [Read from the ring buffer](#read-from-the-ring-buffer)
  - [ISR and ring buffers](#isr-and-ring-buffers)
  - [Important concurrency notes](#important-concurrency-notes)
  - [Example with UART or sensor data](#example-with-uart-or-sensor-data)
  - [What happens when the buffer is full?](#what-happens-when-the-buffer-is-full)
  - [How to detect full vs empty](#how-to-detect-full-vs-empty)
  - [Summary](#summary)


A ring buffer, also called a circular queue, is a fixed-size buffer that wraps around when it reaches the end.

This is useful when a sensor or peripheral generates data faster than the application can process it.

```text
Sensor → → → → → → → → → Application
```

## Basic idea

The buffer keeps a fixed number of slots and reuses them by wrapping around.

```text
┌────┬────┬────┬────┬────┬────┐
│  A │  B │  C │    │    │    │
└────┴────┴────┴────┴────┴────┘
   ↑              ↑
  head           tail
```

When the index reaches the end, it wraps back to the beginning using modulo arithmetic.

```c
#define BUFFER_SIZE 8

typedef struct {
    uint8_t buffer[BUFFER_SIZE];
    int head;  // where we write
    int tail;  // where we read
} RingBuffer;
```

## Write to the ring buffer

```c
RBuffer->buffer[RBuffer->head] = data;
RBuffer->head = (RBuffer->head + 1) % BUFFER_SIZE;
```

## Read from the ring buffer

```c
data = RBuffer->buffer[RBuffer->tail];
RBuffer->tail = (RBuffer->tail + 1) % BUFFER_SIZE;
```

The modulo operation makes the buffer wrap around cleanly as it fills and drains.

## ISR and ring buffers

A common pattern is to let an interrupt handler write into the ring buffer while the main program reads from it.

```text
ISR                Main program
writes head        reads tail
adds data          removes data
```

This is a good design because each side only touches one index: the producer updates the head and the consumer updates the tail.

## Important concurrency notes

Possible issues include:

- race conditions
- atomicity
- interrupt interaction
- volatile misuse

For a simple single-producer/single-consumer ring buffer, the producer and consumer usually update different indices, which reduces synchronization complexity.

> `volatile` does not make operations atomic.

That means it does not magically protect a multi-step operation from being interrupted or reordered.

## Example with UART or sensor data

```c
#define BUFFER_SIZE 4

typedef struct {
    uint8_t buffer[BUFFER_SIZE];
    int head;
    int tail;
} RingBuffer;

RingBuffer *RBuffer;
```

If we add values like 10, 20, 30:

```c
RBuffer->buffer[RBuffer->head] = 10;
RBuffer->head = (RBuffer->head + 1) % BUFFER_SIZE;

RBuffer->buffer[RBuffer->head] = 20;
RBuffer->head = (RBuffer->head + 1) % BUFFER_SIZE;

RBuffer->buffer[RBuffer->head] = 30;
RBuffer->head = (RBuffer->head + 1) % BUFFER_SIZE;
```

## What happens when the buffer is full?

There are a few common strategies:

1. Drop the new data and notify the application.
2. Overwrite the oldest data and notify the application.
3. Block the producer until space becomes available.
4. Signal an overflow condition.

## How to detect full vs empty

There are three common approaches:

1. Maintain a counter in the struct.
2. Reserve one slot so the buffer is considered full when head and tail are equal after a different condition check.
3. Use a separate boolean or flag to indicate full status.

## Summary

Ring buffers are a practical way to decouple data production from data consumption in embedded systems. They are especially useful in interrupt-driven designs where the producer and consumer run at different rates and with different timing constraints.

