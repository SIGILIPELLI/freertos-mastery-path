# FreeRTOSConfig.h Deep Dive

Every module so far has referenced individual `configXXX` macros in
passing. This module treats `FreeRTOSConfig.h` as a single artifact worth
understanding end-to-end — because it's the one file that silently
determines which APIs exist, how much RAM the kernel itself consumes, and
whether certain bugs are even detectable. Two projects with identical
application code but different `FreeRTOSConfig.h` files can behave, and
fail, completely differently.

## Scheduling knobs

```c
#define configUSE_PREEMPTION            1   // 0 = cooperative only (rare, legacy)
#define configUSE_TIME_SLICING          1   // equal-priority tasks round-robin per tick
#define configMAX_PRIORITIES            8   // priorities 0..(N-1); higher = more RAM per ready list
#define configTICK_RATE_HZ              1000
#define configCPU_CLOCK_HZ              ( configured per target )
#define configUSE_PORT_OPTIMISED_TASK_SELECTION  1   // CLZ-based O(1) ready-task lookup vs O(n) scan
```

`configUSE_PREEMPTION=0` turns FreeRTOS into a cooperative scheduler — a
running task keeps the CPU until it explicitly yields or blocks, even if a
higher-priority task becomes ready. This removes an entire class of race
condition (nothing can preempt you mid-statement) at the cost of every
task needing to be disciplined about yielding — almost no modern embedded
project ships this way, but it's useful to recognize if you inherit one
that does. `configMAX_PRIORITIES` sets an upper bound on priority levels
and directly costs RAM (one `List_t` ready-list per level) — don't set it
to 32 "to be safe" if you use four priorities.

## Memory & allocation knobs

```c
#define configTOTAL_HEAP_SIZE           ( ( size_t ) ( 64 * 1024 ) )   // heap_1..4 only; irrelevant to heap_5 or static-only
#define configSUPPORT_STATIC_ALLOCATION 1
#define configSUPPORT_DYNAMIC_ALLOCATION 1
#define configMINIMAL_STACK_SIZE        ( configSTACK_DEPTH_TYPE ) 128   // idle task's own stack, in words
#define configSTACK_DEPTH_TYPE          uint16_t   // widen to uint32_t if any stack exceeds 65535 words
```

`configTOTAL_HEAP_SIZE` only matters if you're linking one of the
`heap_1.c`–`heap_4.c` allocators (Module 2's static-allocation discussion
covers when you can skip it entirely). Undersizing it produces the most
common first-week FreeRTOS bug: `xTaskCreate` or `xQueueCreate` returning
`NULL`/`pdFAIL` on a board that "should have plenty of RAM," because the
*kernel's* heap is a separate, fixed-size carve-out from your linker
script's remaining free RAM, not a magic pool that grows.

## Feature-inclusion knobs: `INCLUDE_*`

```c
#define INCLUDE_vTaskDelete            1
#define INCLUDE_vTaskSuspend           1
#define INCLUDE_vTaskDelayUntil        1
#define INCLUDE_uxTaskPriorityGet      1
#define INCLUDE_eTaskGetState          1
#define INCLUDE_xTimerPendFunctionCall 1
```

Unlike `configXXX` macros (which tune existing behavior), `INCLUDE_XXX`
macros are **compiled out entirely** when set to 0 — calling
`vTaskDelete()` in application code when `INCLUDE_vTaskDelete` is 0
produces a compile error, not a runtime failure, because the function
doesn't exist in the built image at all. This is deliberate: every
`INCLUDE_` flag you leave off saves flash. Leaving them all at 1 "to be
safe" is a reasonable default during development; trimming unused ones is
a legitimate flash-budget optimization for a near-final build once you
know exactly which APIs you call.

## Debug & robustness knobs

```c
#define configCHECK_FOR_STACK_OVERFLOW  2   // 0 off, 1 lightweight SP check, 2 pattern-fill check (Module 2)
#define configUSE_MALLOC_FAILED_HOOK    1   // requires vApplicationMallocFailedHook()
#define configASSERT( x )  if( ( x ) == 0 ) { taskDISABLE_INTERRUPTS(); for( ;; ); }
#define configUSE_TRACE_FACILITY        1   // enables uxTaskGetSystemState, trace hooks
#define configGENERATE_RUN_TIME_STATS   0   // needs a high-res timer wired up; CPU-time-per-task stats
#define configRECORD_STACK_HIGH_ADDRESS 1   // improves stack overflow diagnostics
```

`configASSERT` is the single highest-leverage debug macro in the file: it's
called from dozens of internal sanity checks throughout the kernel (bad
parameters, corrupted list pointers, calling a blocking API from ISR
context) and is a no-op by default in many starter configs. Defining it to
actually halt (or log + reset) turns silent corruption into an immediate,
attributable failure — always define it for anything past the earliest
bring-up stage. Leaving `configGENERATE_RUN_TIME_STATS` off is fine until
you need `vTaskGetRunTimeStats()`'s per-task CPU-time breakdown, at which
point it requires a free-running timer with resolution higher than the
tick — wiring that up is port- and target-specific.

## Hook functions the config enables

Several `config` flags require you to *supply* a corresponding function or
the link fails — this is the same pattern you already hit in Modules 2 and
5:

| Flag | Required hook |
|---|---|
| `configUSE_IDLE_HOOK` | `vApplicationIdleHook(void)` |
| `configUSE_TICK_HOOK` | `vApplicationTickHook(void)` |
| `configCHECK_FOR_STACK_OVERFLOW > 0` | `vApplicationStackOverflowHook(TaskHandle_t, char*)` |
| `configUSE_MALLOC_FAILED_HOOK` | `vApplicationMallocFailedHook(void)` |
| `configSUPPORT_STATIC_ALLOCATION` + no dynamic | `vApplicationGetIdleTaskMemory`, `vApplicationGetTimerTaskMemory` |

## Reading it as a whole system, not a checklist

The productive way to review an unfamiliar `FreeRTOSConfig.h` is to ask,
in order: (1) preemptive or cooperative, and how many priority levels; (2)
where does memory come from — heap variant, static, or both, and is the
heap size actually justified by what's created; (3) what's on for
debugging — assert, stack overflow detection, trace; (4) which optional
subsystems are compiled in at all (timers, event groups, stream buffers,
queue sets) via their respective `configUSE_*` flags, since each is extra
flash whether or not application code exercises it.

## Traps

- **Copy-pasting a demo's `FreeRTOSConfig.h` wholesale**: demo configs are
  tuned for the demo, not your app — `configTOTAL_HEAP_SIZE`, `configMAX_PRIORITIES`,
  and the `INCLUDE_*` set are the three most commonly wrong-for-your-project
  values inherited this way.
- **`configASSERT` left as a no-op past bring-up**: this converts every
  internal kernel sanity check into silent nothing — corruption that
  would have halted immediately instead surfaces as an unrelated crash
  minutes later, far from the actual cause.
- **Changing `configTICK_RATE_HZ` without re-checking every `pdMS_TO_TICKS`
  call site**: `pdMS_TO_TICKS` is defined in terms of the current tick
  rate, so raw tick-count literals (not run through the macro) silently
  change their real-world meaning if the tick rate changes.
- **Assuming `configMAX_PRIORITIES` is free**: each additional priority
  level costs one `List_t` (kernel ready-list) permanently, regardless of
  whether any task ever uses it.
- **Forgetting a required hook after flipping a flag on**: every row in the
  hooks table above is a linker error waiting to happen the first time
  someone enables the flag without reading what it requires.

## Cheat sheet

| Category | Key macros |
|---|---|
| Scheduling | `configUSE_PREEMPTION`, `configUSE_TIME_SLICING`, `configMAX_PRIORITIES`, `configTICK_RATE_HZ` |
| Memory | `configTOTAL_HEAP_SIZE`, `configSUPPORT_STATIC/DYNAMIC_ALLOCATION`, `configMINIMAL_STACK_SIZE` |
| Feature inclusion | `INCLUDE_vTaskDelete`, `INCLUDE_vTaskSuspend`, `INCLUDE_eTaskGetState`, etc. — compiled OUT at 0 |
| Debug/robustness | `configCHECK_FOR_STACK_OVERFLOW`, `configASSERT`, `configUSE_TRACE_FACILITY` |
| Subsystems | `configUSE_TIMERS`, `configUSE_QUEUE_SETS`, `configUSE_TASK_NOTIFICATIONS`, `configUSE_MUTEXES` |
| Required hooks | See hooks table — tied to specific `configUSE_*`/`configCHECK_*` flags |

## Exercise

1. Take a working project's `FreeRTOSConfig.h` and answer the four
   "reading it as a whole system" questions above in writing, without
   running anything — then verify each answer against actual behavior
   (e.g., print `configMAX_PRIORITIES` and the actual number of distinct
   priorities your tasks use).
2. Set `configASSERT` to a real halting implementation (if it isn't
   already) and deliberately trigger one internal assert — e.g., call a
   queue API with a `NULL` handle — and confirm it halts immediately
   rather than corrupting state silently.
3. Reduce `configTOTAL_HEAP_SIZE` in small steps until `xTaskCreate` starts
   failing on your actual application's task/queue set, and record the
   minimum that still boots cleanly — then add 25% headroom and justify
   that number in a comment.
