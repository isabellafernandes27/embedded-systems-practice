# Concurrency in Embedded Systems

- [Concurrency in Embedded Systems](#concurrency-in-embedded-systems)
  - [Race conditions](#race-conditions)
  - [Mutexes](#mutexes)
  - [Semaphores](#semaphores)
    - [Binary semaphore vs. mutex](#binary-semaphore-vs-mutex)
  - [Spinlocks](#spinlocks)
  - [Deadlocks](#deadlocks)
  - [Producer-consumer pattern](#producer-consumer-pattern)
  - [Priority Inversion](#priority-inversion)
  - [Mars Pathfinder Example](#mars-pathfinder-example)
  - [Volatile vs Synchronization](#volatile-vs-synchronization)

A mutex provides exclusive ownership of a shared resource. A semaphore is typically used either to count available resources or signal between threads. A spinlock repeatedly checks for availability instead of blocking, so it is useful when the wait is expected to be extremely short, but it wastes CPU while spinning.

## Race conditions

A race condition happens when the result depends on the timing or interleaving of multiple threads accessing shared data.

Example:

```c
int counter = 0;
counter = 1;
```

If two threads do this at the same time:

```text
Thread A              Thread B
read counter (0)      read counter (0)
add 1                 add 1
write 1               write 1
```

This pattern can lead to lost updates or inconsistent results if the operations are not synchronized.

## Mutexes

Only one thread can hold the mutex at a time.

```text
Thread A
  ↓
lock mutex
  ↓
critical section
  ↓
unlock mutex
```

When Thread B tries to access the same resource, it will be blocked until Thread A releases the mutex.

```c
lock(mutex);

counter += 1;

unlock(mutex);
```

Mutexes are good for protecting shared data structures.

## Semaphores

A semaphore represents available resources or signals.

- `count = 3` means 3 resources are available.
- `count = 2` means a thread has acquired one resource.

When the count reaches zero, another thread must wait.

### Binary semaphore vs. mutex

- A mutex has ownership. Generally, the thread that locks it should unlock it.
- A semaphore is often used for signaling between threads.

```text
Producer
  │
  │ produces data
  ↓
semaphore_post()
  │
  ↓
Consumer wakes up
```

## Spinlocks

A spinlock is a lock that causes a thread to wait in a loop while it checks whether the lock is available. Spinlocks are commonly used in embedded systems where the overhead of context switching is too high.

> "Don't sleep; keep checking until the lock becomes available."

```c
bool lock_is_taken = true;

int main(void) {
    while (lock_is_taken) {
        // spin
    }
    return 0;
}
```

Useful when:

1. The critical section is extremely short.
2. You cannot afford to sleep or block.
3. You are in kernel or low-level code.
4. Waiting is expected to be very brief.

The downside is that the CPU is busy doing work while waiting.

## Deadlocks

A deadlock occurs when two or more threads are stuck waiting on each other.

Example:

- Thread A holds mutex 1 and waits for mutex 2.
- Thread B holds mutex 2 and waits for mutex 1.

Both threads are now stuck, creating a deadlock.

Deadlock requires all four conditions:

1. Mutual exclusion: a resource can only be held by one thread at a time.
2. Hold and wait: a thread holds one resource while waiting for another.
3. No preemption: you cannot forcibly take the resource away.
4. Circular wait: Thread A waits for B while B waits for A.

If you break any one of these conditions, you can prevent deadlock.

A simple prevention strategy is to lock resources in a consistent order:

```text
Thread A: lock(A) -> lock(B)
Thread B: lock(A) -> lock(B)
```

## Producer-consumer pattern

The producer-consumer pattern coordinates work between threads using shared buffers and synchronization primitives such as mutexes and semaphores. The producer adds items to a queue while the consumer removes them, ensuring the buffer is used safely and without data races.

```text
        PRODUCER
           │
           │ produces data
           ↓
      ┌───────────┐
      │   Queue   │
      └───────────┘
           │
           ↓
        CONSUMER
```

The producer puts data into the buffer. The consumer removes data.

Synchronization needs to ensure:

- producer doesn't corrupt the queue
- consumer doesn't read invalid data
- producer doesn't overwrite unread data
- consumer doesn't read empty slots

**A semaphore can be useful to signal: "Hey, there's data available."**

And a mutex could protect a queue if multiple threads can access it.

## Priority Inversion

Imagine three threads:

```text
HIGH priority     H
MEDIUM priority   M
LOW priority      L
```

The low priotiry task acquires a mutex:
```text
L → locks resource
```
Then, the high priority task needs that resource:
```text
H → tries to lock → BLOCKED
```
So H is waiting for L. But now M starts running.

L can't get enough CPU time to finish its critical section and release the mutex.

So the high-priority task is effectively being delayed by the medium-priority task.

That's priority inversion.

```text
H
│
│ waiting for L
↓
L ← can't run because M is running
↑
M
```

## Mars Pathfinder Example

The Mars Pathfinder system experienced resets associated with a priority inversion problem involving tasks sharing a resource.

In a real-time embedded system, synchronization can interact with scheduling and priorities in ways that cause high-priority work to miss its timing requirements.

A common mitigation is priority inheritance.

The low-priority task holding the mutex temporarily inherits the higher priority of the task waiting for it.

```text 
Before:

H waits → L
M runs → L can't finish

After ()

H waits → L

L temporarily becomes H priority
        ↓
L finishes quickly
        ↓
L releases mutex
        ↓
H runs
        ↓
L goes back to its normal priority
```

## Volatile vs Synchronization

Given:

```c
volatile int ready = 0;

threadA->ready = 1;

while(threadB->ready == 0){
    // wait
}
```
**Some might say: "It's volatile, so this is thread-safe." But that is not correct!**

Volatile means: 
> "Don't optimize away accesses to this object; its value may change outside the normal flow of this code."

That's very useful for:

- hardware registers
- memory-mapped I/O
- values changed by an ISR in some embedded contexts

But volatile does not inherently provide:

- mutual exclusion
- atomicity (single, indivisible step)
- thread synchronization
- memory ordering