# SMP Scheduling

**Verification note for this module:** FreeRTOS-Kernel's SMP support requires
a multi-core target port (dual-core RP2040/RP2350, ESP32's dual-core Xtensa
port, or the SMP-capable Cortex-M/A ports) — there is no SMP-capable build of
the POSIX simulator port used elsewhere in this course, so nothing here was
built or run on this machine. What follows is a careful technical review
against the FreeRTOS-Kernel SMP source (`tasks.c` under
`configNUMBER_OF_CORES > 1`) and the official SMP documentation, not an
executed test.

## Single scheduler, multiple cores — not multiple schedulers

FreeRTOS SMP (introduced as an official kernel feature in the v10.4+ SMP
branches, later merged mainline) is **not** "one FreeRTOS instance per core."
It's a single scheduler, a single set of ready lists (Module 1's
`pxReadyTasksLists`), and a single set of kernel data structures — but now up
to `configNUMBER_OF_CORES` tasks can be *simultaneously* in the running state,
one per core:

```c
// FreeRTOSConfig.h
#define configNUMBER_OF_CORES 2
#define configUSE_CORE_AFFINITY 1
#define configRUN_MULTIPLE_PRIORITIES 1   // allow different priorities to run concurrently
```

This single-scheduler design is deliberate: it means the mental model from
every prior module (one ready list per priority, highest priority runs) still
applies — the difference is that "runs" now means up to N tasks pulled off
the ready lists concurrently, each core running the independent
`vTaskSwitchContext`-equivalent path guarded by the *same* kernel data
structures, protected now by real spinlocks instead of single-core
interrupt-disable.

## Core affinity

```c
TaskHandle_t sensorTask;
xTaskCreate(vSensorTask, "Sensor", stackSize, NULL, 3, &sensorTask);
vTaskCoreAffinitySet(sensorTask, (1 << 0));          // pin to core 0 only
vTaskCoreAffinitySet(otherTask, (1 << 0) | (1 << 1)); // either core (default)
```

Affinity is a bitmask, not a single core index — a task can be pinned to one
core, allowed on any core, or restricted to a subset on systems with more
than two cores. Pinning matters for two distinct reasons: **hardware
locality** (a task that talks to a peripheral only reachable from core 0's
bus) and **cache behavior** (a task that migrates between cores on every
switch cold-misses its working set in that core's L1 each time — pinning a
cache-sensitive hot loop avoids this).

## `configRUN_MULTIPLE_PRIORITIES`: the setting that changes everything

With this off (the default, and the safer starting point), FreeRTOS SMP
enforces a stronger invariant closer to single-core intuition: a lower
priority task is *not* allowed to run on any core while a higher priority
task is ready anywhere in the system, even if that would mean leaving a core
idle. With it on, the scheduler will run **any two ready tasks concurrently**
regardless of priority difference, to keep both cores busy — a priority-5
task and a priority-1 task really can execute at the literal same instant on
two different cores. This directly changes what "priority" *means* under
SMP: it no longer guarantees a lower-priority task never executes while a
higher one is ready — only that on a *given* core, the ready task with the
highest priority *for that core's affinity set* gets picked. Any code that
was written assuming single-core priority semantics (a common gatekeeper or
mutual-exclusion assumption from Level 2 Module 8) needs re-auditing under
this flag.

## Spinlocks: the real critical section on SMP

Single-core FreeRTOS gets mutual exclusion from `taskENTER_CRITICAL()`
disabling interrupts — trivially sufficient when there is exactly one
execution context. SMP breaks this assumption completely: disabling
interrupts on core 0 does nothing to stop core 1 from concurrently mutating
the same ready list. The SMP kernel replaces this with real spinlocks
(`portGET_TASK_LOCK`/`portGET_ISR_LOCK` on top of a port-supplied atomic
compare-and-swap) guarding kernel data structures, on top of (not instead of)
each core's own local interrupt-disable:

```c
// conceptual shape inside SMP tasks.c — not exact source
taskENTER_CRITICAL();          // disables interrupts on THIS core
                                // AND takes a cross-core spinlock
    /* touch pxReadyTasksLists, pxCurrentTCBs[coreID], etc. */
taskEXIT_CRITICAL();
```

The practical consequence: **critical sections on SMP cost more** than the
equivalent single-core interrupt-disable, because a core can now spin waiting
for another core to release the kernel lock, not just briefly disable its
own interrupts. Keeping critical sections short is no longer just a latency
concern for other tasks on the same core — it's contention that stalls a
*different physical core* entirely.

## Cache coherency: the trap that has no single-core equivalent

This is the sharpest new hazard SMP introduces and the reason this module
exists at Level 3 rather than being folded into Level 1's queue coverage.
FreeRTOS's own kernel data structures are correctly protected by the
spinlocks above — but **application data shared between tasks pinned to
different cores is the application's responsibility**, and most
microcontroller SMP targets have per-core data caches that are not
automatically kept coherent by hardware the way desktop-class multicore CPUs
are.

```c
// DANGEROUS on a non-cache-coherent dual-core MCU:
volatile int sharedFlag = 0;    // written by core 0, polled by core 1

void core0Task(void *pv) {
    sharedFlag = 1;              // write may sit in core 0's cache line only
}
void core1Task(void *pv) {
    while (sharedFlag == 0) {}   // may spin forever reading a stale cached value
}
```

`volatile` prevents the *compiler* from caching the value in a register — it
says nothing about the *hardware* data cache. On a target without hardware
cache coherency between cores, this pattern can hang indefinitely, or
intermittently "work" depending on cache eviction timing, which makes it
brutal to debug. The correct fix is architecture-specific: explicit cache
maintenance operations (clean/invalidate a cache line after writing/before
reading shared data), placing genuinely shared data in an uncached memory
region if the target provides one, or — the FreeRTOS-idiomatic answer —
routing the communication through a FreeRTOS primitive (queue, semaphore,
stream buffer) whose SMP-aware kernel code already performs the necessary
cross-core synchronization and memory barriers for you, and treating raw
shared globals between core-pinned tasks as something to avoid rather than
something to patch with `volatile`.

## Traps

- **Assuming `configRUN_MULTIPLE_PRIORITIES=0` restores full single-core
  priority guarantees.** It's closer, but not identical — with two cores
  idle-eligible, the exact task selected can still differ in edge cases from
  single-core round-robin among equal priorities; always test priority
  assumptions on the actual SMP configuration you ship, not by analogy.
- **Using `volatile` as if it were a substitute for cache maintenance or a
  real synchronization primitive** between core-pinned tasks — as shown
  above, this is the single most dangerous SMP-specific bug class and has no
  single-core equivalent to build intuition from.
- **Pinning everything to one core "to be safe."** This defeats the purpose
  of SMP (you paid for two cores and use one) and can silently reintroduce
  every single-core assumption while still carrying SMP's spinlock overhead
  on kernel calls — audit affinity assignments deliberately, not defensively.
- **Ignoring `configUSE_CORE_AFFINITY` when a peripheral is only wired to one
  core's bus matrix** — a task that isn't pinned can migrate mid-run and lose
  access to memory-mapped peripheral registers that are only visible from
  the core it started on, depending on the SoC's bus topology.
- **Treating this module's coverage as a substitute for your specific
  vendor's SMP porting notes.** ESP32 (Xtensa, asymmetric core capabilities),
  RP2040/RP2350 (symmetric Cortex-M0+/M33), and other SMP targets each have
  vendor-specific caveats (e.g., which core services which interrupt by
  default) not covered by the generic FreeRTOS-Kernel SMP documentation
  alone — always cross-check the vendor's own SMP port notes.

## Cheat sheet

| Concept | Single-core FreeRTOS | SMP FreeRTOS |
|---|---|---|
| Ready lists | One set, one scheduler | Same one set, shared across cores |
| Critical section | Disable interrupts (this core only) | Disable interrupts + spinlock (cross-core) |
| "Running" | Exactly one task | Up to `configNUMBER_OF_CORES` tasks |
| Priority guarantee | Higher priority always preempts lower | Only guaranteed per-core unless `configRUN_MULTIPLE_PRIORITIES=0` |
| Task placement | N/A | `vTaskCoreAffinitySet` bitmask |
| Shared app data hazard | Race conditions (mutex/critical section fixes it) | Race conditions **and** cache incoherency — mutex alone may not fix it |
| Verified on this machine | N/A (source review only) | N/A — no SMP-capable POSIX simulator exists |

## Exercise

1. Read the SMP-specific sections of `tasks.c` in a recent FreeRTOS-Kernel
   checkout (search for `configNUMBER_OF_CORES` and `portGET_TASK_LOCK`) and
   identify every place a single-core-only assumption from Module 1
   (this course, kernel internals) had to be replaced with cross-core-safe
   logic.
2. On a real dual-core target (ESP32 or RP2040/RP2350), reproduce the
   shared-flag cache-coherency hazard above deliberately: pin one task per
   core, use a raw shared `volatile int`, and observe whether/when it fails,
   then fix it with a proper FreeRTOS primitive and confirm the failure
   disappears.
3. Design a task/priority/affinity layout for a two-core sensor+network
   system (one core dedicated to time-critical sampling, one to
   network/logging) and justify each `vTaskCoreAffinitySet` call and your
   `configRUN_MULTIPLE_PRIORITIES` choice in writing.
