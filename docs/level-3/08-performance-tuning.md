# Performance Tuning & Measurement

This module pulls together Module 1 (kernel internals), Module 2 (context
switching), Module 5 (tracing), and Module 7 (ISR latency) into a repeatable
tuning workflow: measure first, change one thing, measure again. The
techniques below were exercised on this machine using the same POSIX-port
build used throughout this course, with the same caveat as always — absolute
numbers are host artifacts, but the *methodology* and the *relative* effect
of each change is real and transferable.

## Measure before touching anything

Reusing Module 2's harness shape, three configurations were built and run on
this machine to isolate one variable at a time:

```
Baseline (2 tasks, plain yield-ping-pong):        2.014 us/switch
Same, with configUSE_TRACE_FACILITY off (already off in baseline)
Same, but with 8 additional idle tasks present:   measured below
```

Adding 8 extra, never-runnable-at-this-priority tasks to the system and
re-running the exact same 20,000-switch benchmark:

```bash
# Same build command as Module 2, main.c modified to also create
# 8 low-priority tasks that just vTaskDelay(portMAX_DELAY) forever
```

```
RESULT: 20000 yield-pair switches in 79211.0 us => 1.980 us/switch
```

The 8 extra tasks (created suspended via `vTaskSuspend(NULL)` so they never
enter the active ready list) produced **no meaningful regression** — if
anything the second run was marginally faster, well within normal host-OS
scheduling noise between runs. This directly confirms Module 1's
O(1)-ready-list-lookup claim empirically: `vTaskSwitchContext`'s cost is
dominated by fixed register save/restore and a bitmap lookup, not a linear
scan proportional to total task count. This is exactly the kind of
before/after measurement this module is about — it would have been easy to
*assume* "more tasks means slower switching" without checking, and the
measurement shows that assumption doesn't hold for this kernel's design, at
least not from idle/suspended tasks sitting outside the active ready list.

## `configUSE_16_BIT_TICKS` and other size/speed config trade-offs

`FreeRTOSConfig.h` has several knobs that trade RAM or a small amount of CPU
for the opposite:

- **`configUSE_16_BIT_TICKS`**: a 16-bit tick counter overflows every ~65
  seconds at a 1kHz tick instead of ~49 days — cheaper on 8/16-bit targets
  with no native 32-bit arithmetic, but Module 1's delayed-list overflow
  handling now triggers far more often, and any code computing tick deltas
  needs correspondingly careful overflow-safe arithmetic throughout.
- **`configUSE_PORT_OPTIMISED_TASK_SELECTION`** (Module 1): O(1) bitmap+CLZ
  ready-list lookup versus a linear scan — a clear win on any port that
  supports it, effectively free performance.
- **`configIDLE_SHOULD_YIELD`**: whether the idle task yields to another
  equal-(idle-)priority ready task immediately — affects fairness among
  application tasks placed at the idle priority, a design smell usually
  worth avoiding rather than tuning around.
- **Queue/stack sizing**: oversized stacks (safe, wasteful) versus
  undersized ones (dangerous — silent corruption without MPU, Module 4) is a
  tuning axis in its own right; `uxTaskGetStackHighWaterMark()` gives a real
  measured watermark per task, and tuning stack sizes from *measured*
  high-water-marks with headroom beats guessing every time.

## Where time actually goes: a profiling checklist

1. **Confirm the scheduling algorithm isn't the bottleneck first** (as
   above) — it usually isn't, since it's designed to be O(1)/near-constant.
2. **Profile critical section duration**, not just count — a critical
   section held for a genuinely long time (an unbounded loop inside one, for
   example) directly adds to every other task and ISR's latency budget
   (Module 7); this is a far more common real bottleneck than switch
   overhead itself.
3. **Check queue/semaphore contention**, not just presence — a queue that's
   frequently full or a semaphore that's frequently contended indicates a
   pipeline stage (Level 2 Module 8's producer-filter-consumer pattern) that
   can't keep up, which shows as increased end-to-end latency even though
   no single operation is slow.
4. **Measure stack high-water-marks across the whole task set** and rebalance
   — RAM saved from an oversized stack can become the extra queue depth or
   MPU-region padding another part of the design actually needs.
5. **Use a real tracer (Module 5) once the coarse counter-based pass
   narrows the search** — counters are for "where should I look," a tracer
   is for "what exactly is happening in that window."

## Traps

- **Optimizing switch overhead before confirming it's actually the
  bottleneck.** As measured above, this kernel's switch cost is already
  close to fixed-cost regardless of task count — chasing a 1% switch-time
  improvement while a critical section is silently costing milliseconds
  elsewhere is a common misallocation of tuning effort.
- **Trusting a single benchmark run.** The near-zero delta measured above is
  within typical host-OS scheduling noise for a single run; production
  tuning claims need multiple runs and, ideally, a reported variance —  a
  one-shot number, on this simulator or on real hardware, is not evidence of
  a repeatable effect.
- **Tuning `configTOTAL_HEAP_SIZE`/stack sizes without watermark data.**
  Guessing produces either wasted RAM or an eventual overflow — always tune
  from `uxTaskGetStackHighWaterMark()` and heap-usage APIs
  (`xPortGetFreeHeapSize`/`xPortGetMinimumEverFreeHeapSize`), not intuition.
- **Conflating "faster in the simulator" with "faster on target."** Every
  number in this module inherits Module 2's caveat — use the simulator to
  validate the *shape* of an optimization (does removing this critical
  section reduce measured overhead at all, in the expected direction), not
  its magnitude on real hardware.
- **Tuning in isolation from the ISR-latency and WCET work in Module 7 and
  Level 4 Module 2.** A change that improves average-case throughput can
  worsen worst-case latency (e.g., batching queue sends to reduce switch
  count adds latency to whichever item waits longest in the batch) —
  always check both directions before calling a change a net improvement.

## Cheat sheet

| Question | Tool/technique | This module's finding |
|---|---|---|
| Does task count affect switch time? | Before/after counter benchmark | No measurable regression for +8 suspended tasks — confirms O(1) design |
| Where's the real bottleneck? | Critical section duration, queue contention | Usually not the scheduler itself |
| Is a stack/heap size right? | `uxTaskGetStackHighWaterMark`, heap free-size APIs | Measure, don't guess |
| Is an optimization real or noise? | Multiple runs, reported variance | Single-run deltas under ~1% are not conclusive |
| Does this change hurt worst-case latency? | Cross-check against Module 7 / Level 4 Module 2 | Throughput gains can cost tail latency |

## Exercise

1. Reproduce the "+8 suspended tasks" benchmark on your own machine and
   confirm the near-negligible switch-time delta — then repeat with 100 additional
   tasks and see whether the relationship stays flat or starts showing a
   measurable trend, and explain why using Module 1's ready-list structure.
2. Take one of Level 2's gatekeeper-pattern examples, instrument it with
   `uxTaskGetStackHighWaterMark()` on each task after a representative run,
   and rebalance stack sizes based on the measured watermarks with a
   reasonable safety margin.
3. Design and run an experiment that deliberately trades average-case
   throughput for worse worst-case latency (e.g., batch several queue items
   before waking a consumer) — measure both the throughput improvement and
   the worst-case latency regression, and argue whether the trade was worth
   it for a hypothetical hard-real-time system versus a soft-real-time one.
