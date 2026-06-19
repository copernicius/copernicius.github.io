---
title: The Silent Performance Killer - False Sharing in C++
date: 2026-06-19 16:39:11
tags: [performance, C++, under-the-hood]
---

> 🏎️ I've encounter some weird performance issues when I was tunning my C++ for a high performance RPC framework in TikTok long time ago. 
👬 I cooperated with another colleague, aimming to write a fast ring buffer, somehow he updated the key `struct` definition, we weren't on the same page though.
In hindsight, the degradation in performance was not `directly` a bug in the code, but a feature of the hardware. In our benchmark, the performance under 2 threads was better than that of 8 threads, which was really interesting~

This is `false sharing`, one of the most counterintuitive performance bugs in concurrent C++. Now, let's see what's happening!

## *TL;DR*
- `False sharing` occurs when multiple threads access and modify variables that reside on the same cache line, leading to a degradation in performance.
- `False sharing` can be avoided by using padding or alignment to separate variables on different cache lines.

## What's False Sharing?
When multiple threads access and modify variables that reside on the same cache line, the cache line is constantly invalidated between cores, leading to a degradation in performance.

## Feynman Method
  ![](/images/false_sharing_7.png)
Imagine you and a colleague are sharing a single whiteboard, but you each only need one small corner of it.
You're writing your notes in the top-left corner. Your colleague is writing theirs in the top-right corner. You never touch each other's work. Completely independent.
**But here's the catch: the whiteboard can only be erased and rewritten one person at a time**. Every time you want to write something, you have to claim the whole whiteboard, which means your colleague has to stop, wait for you to finish, then reclaim it before they can write again. And vice versa.
Neither of you is doing anything wrong. You're not interfering with each other's content at all. But you're constantly blocking each other from using the board, because the rule operates on the whole whiteboard, not on individual corners.
That whiteboard is a cache line. Your corner is variable x. Your colleague's corner is variable y. The "claim the whole board" rule is the `MESI` coherence protocol.
**The fix is simple: get a second whiteboard. Put your notes on one, your colleague's notes on the other. Now you never have to wait for each other**. That's what `alignas does`, it forces each variable onto its own cache line, its own whiteboard, so the two cores never have to fight over the same one.

## Start with an Example
Consider the following code snippet:

```cpp
#include <chrono>
#include <cstdint>
#include <iostream>
#include <new>
#include <thread>
#include <vector>

constexpr uint64_t kMaxTimes = 1000000000UL;

#if 0 
// X : bad example
struct FalseSharing {
  volatile uint64_t x{0};
  volatile uint64_t y{0};
};

#else

// √ : good example, fixed the false-sharing issue
struct FalseSharing {
  alignas(64) volatile uint64_t x{0};
  alignas(64) volatile uint64_t y{0};
};
#endif

FalseSharing data;

void WorkerA() {
  for (uint64_t i = 0; i < kMaxTimes; ++i) {
    ++data.x;
  }
}

void WorkerB() {
  for (uint64_t i = 0; i < kMaxTimes; ++i) {
    ++data.y;
  }
}

int main() {
  auto start = std::chrono::steady_clock::now();

  std::thread t1(WorkerA);
  std::thread t2(WorkerB);
  t1.join();
  t2.join();

  auto end = std::chrono::steady_clock::now();
  auto cost =
      std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
  std::cout << "Time cost totally: " << cost.count() << " ms \n";
}
```

Let's run the code snippet!
![Build and Run](/images/false_sharing_2.png)

* When false-sharing:
  ![Bad result](/images/false_sharing_3.png)
* When fixed:
  ![Good result](/images/false_sharing_4.png)

The performance result gap is significant, it costs `1776 ms` to run the code snippet with false-sharing, while `409 ms` to run it with fixed false-sharing issue.
And **it will be even worse if we run it on more cores**.

When reproducing the case by yourself, it should be noted that:
1. We should compile with `-O3`, otherwise the compiler will optimize away the false-sharing issue. Sepecifically, it will simply remove the loop and add the final result. It was like "You nerd, why not add the variable all at once??".

2. We should use `volatile` to ensure that the compiler doesn't optimize away the access to the variables **in the example**. Otherwise, the variable is cached in the CPU register, making it hard to observe the performance degradation. Since it optimizes away a ton of `DRAM memory access`.


## Under the Hood

CPUs don't read individual bytes from RAM. They read in fixed-size chunks called cache lines — 64 bytes on most modern x86 and ARM processors. When a core accesses any byte in memory, the entire 64-byte line containing it is pulled into L1 cache.

This is usually a win. But in multithreaded code, it creates a trap.

![](/images/false_sharing_1.png)

Suppose Thread A writes to x and Thread B writes to y. Different variables, different threads, no logical sharing. But if x and y sit within the same 64-byte cache line, the CPU doesn't know that. Every time Thread A writes, it acquires exclusive ownership of the line — invalidating Thread B's copy. Thread B then reloads the line before it can write. Then A's copy is invalidated. Back and forth, millions of times per second.

This is false sharing: threads that don't share data logically, but are forced by hardware to behave as if they do.

The mechanism is the MESI cache coherence protocol. A write transitions a cache line to Modified on the writing core and broadcasts Invalid to all other cores holding a copy. Each invalidation costs dozens to hundreds of cycles and they never stop.

## Solution
### Method1: Padding
By adding padding to the struct, we can separate the variables on different cache lines, thus avoiding false sharing. In C++, we can use keyword `alignas` to align the variables.
```cpp
// √ : good example, fixed the false-sharing issue
// Usually 64 byte cache line, in C++17 `alignas(std::hardware_destructive_interference_size)` is the right answer.
struct FixFalseSharing {
  alignas(64) volatile uint64_t x{0};
  alignas(64) volatile uint64_t y{0};
};
```
The padding happens during compilation phase by adding padding bit between the variables.
![](/images/false_sharing_5.png)

### Method2: Thread local
Another way to avoid false sharing is to use thread-local storage by rethinking the design if possible.
By declaring the variables as thread-local, each thread will have its own copy of the variable, thus avoiding false sharing.
It's like:
```cpp
thread_local uint64_t x; // each process has its own version
```

## Caveats
* **Padding wastes memory:** A padded `char` occupies 64 bytes instead of 1. For a large array of padded structs, that 8× memory bloat can hurt cache efficiency at a higher level.

* **Cache line size varies:** 64 bytes is standard on x86/ARM today, but not universal. Always prefer `std::hardware_destructive_interference_size` over a hardcoded 64.

* **True sharing is different:** If two threads genuinely need to access the same variable, padding does nothing — that's a synchronisation problem, not a layout problem.

* **To confirm false sharing is the root cause:** To confirm false sharing is the root cause, use `perf stat -e cache-misses ./xxxx` on Linux. Typically, a suspiciously high cache miss rate under parallel workload is the tell.

* **False sharing happens across all cache levels, but hurts most at L1:** Each core's L1 and L2 are private — so when two cores write to variables on the same cache line, the coherence protocol repeatedly invalidates those private copies. **The reloading core typically finds a valid copy in the shared L3 (~40 cycles) rather than going all the way to RAM, because the contested line stays hot in the hierarchy from the other core's writes**. The pain comes not from fetch distance but from the frequency of invalidations — what should be a 4-cycle L1 hit becomes a 40-cycle L3 reload on every single write. Across NUMA sockets where there's no shared L3, costs climb further still.

## Key takeaway
![](/images/false_sharing_6.png)
