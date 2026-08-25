# Context Switching Deep Dive

Module 1 explained *which* task the scheduler picks next; this module covers
what actually happens in the microseconds around swapping one task's
execution context for another's — the mechanism, not the policy. On ARM
Cortex-M (the most common FreeRTOS target family), this is the PendSV
exception handler, `pxCurrentTCB`, and a carefully choreographed sequence of
register saves that has to be correct at the assembly level, because a single
misordered instruction here corrupts every task in the system.

## What a context switch actually saves

A task's "context" is everything needed to resume it exactly where it left
off: the CPU's general-purpose registers, the stack pointer, and (on
Cortex-M) the floating-point registers if the task used any. FreeRTOS does
**not** keep this in a separate save area — it pushes it onto the *task's own
stack*, and the TCB's `pxTopOfStack` field is simply updated to point at
whatever the current stack pointer is at the moment of the switch:

```c
typedef struct tskTaskControlBlock {
    volatile StackType_t *pxTopOfStack;   // must be the first member — asm depends on this
    ListItem_t xStateListItem;
    ListItem_t xEventListItem;
    UBaseType_t uxPriority;
    StackType_t *pxStack;
    char pcTaskName[configMAX_TASK_NAME_LEN];
    // ...
} tskTCB;
```

`pxTopOfStack` being the *first* struct member isn't incidental — the
portable assembly context-switch code accesses it via `pxCurrentTCB` with
zero offset, so it can be reached without knowing the rest of the struct
layout. This is one of the few places in the kernel where C struct layout
and hand-written assembly are load-bearing together.

## PendSV: why not do it in the tick ISR directly

Cortex-M's PendSV (Pendable Service Call) exception exists specifically for
this. The tick interrupt (SysTick) runs at a fixed, often high priority; the
actual context switch is deferred to PendSV, configured at the **lowest**
interrupt priority in the system (`configKERNEL_INTERRUPT_PRIORITY` /
`configMIN_INTERRUPT_PRIORITY` in FreeRTOSConfig.h — the port sets
`NVIC_SetPriority(PendSV_IRQn, 0xFF)`-equivalent). The reasons this matters:

- **PendSV can be "pended" (requested) from any context** — the tick ISR,
  another (higher-priority) ISR calling an API's `...FromISR` variant, or a
  task calling `taskYIELD()` — and it only actually *fires* once the CPU
  returns to a priority level where PendSV is allowed to run. This means a
  higher-priority ISR that arrives mid-switch-request simply runs to
  completion first, and PendSV (and the actual switch) happens exactly once,
  after all pending higher-priority interrupt work is done — not once per
  ISR that requested it.
- **Running the switch at the lowest priority guarantees it never itself
  delays a higher-priority ISR.** If context switching ran directly inside
  SysTick or another interrupt at elevated priority, a slow switch would add
  directly to that interrupt's latency and, transitively, to every lower
  one waiting behind it.

```c
void xPortPendSVHandler(void) {
    __asm volatile (
    "   mrs r0, psp                      \n"  // get task's stack pointer
    "   isb                              \n"
    "   ldr r3, =pxCurrentTCB            \n"
    "   ldr r2, [r3]                     \n"
    "   stmdb r0!, {r4-r11}              \n"  // push r4-r11 onto task's stack
    "   str r0, [r2]                     \n"  // save new SP into pxCurrentTCB->pxTopOfStack
    "   stmdb sp!, {r3, r14}             \n"
    "   mov r0, %0                       \n"
    "   msr basepri, r0                  \n"
    "   bl vTaskSwitchContext            \n"  // pick the new pxCurrentTCB
    "   mov r0, #0                       \n"
    "   msr basepri, r0                  \n"
    "   ldmia sp!, {r3, r14}             \n"
    "   ldr r1, [r3]                     \n"  // pxCurrentTCB (now the NEW task)
    "   ldr r0, [r1]                     \n"  // its saved pxTopOfStack
    "   ldmia r0!, {r4-r11}              \n"  // pop r4-r11 from new task's stack
    "   msr psp, r0                      \n"
    "   bx r14                           \n"  // return — hardware pops r0-r3,r12,LR,PC,xPSR
    ::"i"(configMAX_SYSCALL_INTERRUPT_PRIORITY):"memory");
}
```

Note what's *not* explicitly saved: `r0-r3`, `r12`, `LR`, `PC`, and `xPSR`.
Cortex-M's exception entry/exit hardware automatically stacks and unstacks
those on every exception, including PendSV — the handler only needs to
manually save the registers hardware *doesn't* already handle (`r4-r11`,
and `s16-s31` if `configUSE_FPU_SUPPORT` and the task actually used the
FPU, tracked via the lazy-stacking `FPCA` bit). This split is exactly why
Cortex-M's exception model was attractive for RTOS use in the first place —
it halves the manual save/restore work compared to architectures with no
hardware exception-stacking convention.

## `vTaskSwitchContext`: policy lives in C, mechanism lives in asm

The line `bl vTaskSwitchContext` is the entire connection point between
Module 1's list-based scheduling policy and this module's register-save
mechanism. It is ordinary, portable C:

```c
void vTaskSwitchContext(void) {
    if (uxSchedulerSuspended != (UBaseType_t) 0U) {
        xYieldPending = pdTRUE;   // scheduler locked — defer, don't switch now
    } else {
        xYieldPending = pdFALSE;
        taskSELECT_HIGHEST_PRIORITY_TASK();   // finds new pxCurrentTCB from ready lists
    }
}
```

It never touches a register or a stack pointer directly — it only updates
which TCB `pxCurrentTCB` points at. All the actual register shuffling
happens in the assembly wrapper around it. This separation is why porting
FreeRTOS to a new architecture (Module 6) means rewriting the small assembly
layer, not the scheduling algorithm.

## Measuring switch overhead — verified on this machine

The POSIX simulator port (Level 2 Module 5) doesn't do real register-context
switching — it uses `pthread` + signals — so it cannot measure real Cortex-M
cycle counts. But it *can* measure wall-clock overhead per
scheduler-invoked switch on this host, which is a legitimate way to sanity-check
switch-count-driven overhead in a design before hardware exists. This was
built and run on this machine:

```c
#define N_SWITCHES 20000
static volatile int toggle = 0;
static int count = 0;

void taskA(void *pv) {
    for (;;) {
        while (toggle != 0) taskYIELD();
        count++;
        if (count >= N_SWITCHES) { /* report elapsed time, vTaskEndScheduler() */ }
        toggle = 1;
        taskYIELD();
    }
}
void taskB(void *pv) {
    for (;;) { while (toggle != 1) taskYIELD(); toggle = 0; taskYIELD(); }
}
```

Built against `tasks.c`, `queue.c`, `list.c`, `timers.c`, `event_groups.c`,
`stream_buffer.c`, `portable/MemMang/heap_3.c`, and
`portable/ThirdParty/GCC/Posix/{port.c,utils/wait_for_event.c}` with
`-lpthread`, exactly as in Level 2 Module 5. Output from the actual run:

```
RESULT: 20000 yield-pair switches in 80553.0 us => 2.014 us/switch
scheduler ended
```

**Read this number correctly.** ~2 microseconds per switch here reflects
*this host's* pthread-signal-based simulated switch — not a Cortex-M
register-save cost, and it includes general-purpose OS scheduling noise. On
real Cortex-M hardware, a PendSV-based switch is typically single-digit
microseconds or less at moderate clock speeds (dominated by ~16-20 pushed/
popped registers plus fixed hardware exception entry/exit overhead, roughly
constant regardless of task count) — the *shape* of the result (switching
has a small, roughly fixed per-switch cost you can budget for) transfers;
the absolute number does not.

## Traps

- **Assuming task switch time scales with the number of tasks.** It doesn't
  — `vTaskSwitchContext` cost is dominated by the O(1) ready-list lookup
  (Module 1) plus a fixed register save/restore; it is not a function of how
  many total tasks exist in the system.
- **Believing PendSV "runs immediately" when pended.** It runs at the lowest
  priority by design — if higher-priority interrupts are active or pending,
  PendSV (and the switch) is deferred until the CPU returns to a priority it's
  allowed to run at. This is correct and intentional, not a bug, but it means
  a system saturated with high-priority interrupt work can delay task
  switching arbitrarily long.
- **Forgetting `pxTopOfStack` must remain the first TCB member** if you ever
  touch a fork/vendor variant of `tasks.h` — the hand-written assembly
  context-switch routines on every port assume this offset is zero.
- **Not accounting for FPU lazy-stacking cost** on `configUSE_FPU_SUPPORT`
  ports — a task that touches floating point adds `s16-s31` save/restore to
  every switch it's involved in; a system with one float-heavy task among
  many integer-only tasks pays this cost only when that task participates in
  a switch, which can produce inconsistent, hard-to-predict switch timing
  if not accounted for in WCET budgets (Level 4 Module 2).
- **Trusting POSIX-port switch timing as a stand-in for real hardware
  numbers**, as called out above — use it only to validate switch-*count*
  driven design decisions, never absolute latency budgets.

## Cheat sheet

| Concept | Where it lives | Key fact |
|---|---|---|
| Task context | Pushed onto the task's own stack | Not a separate global save area |
| `pxTopOfStack` | First member of TCB | Assembly depends on zero offset |
| PendSV | Lowest interrupt priority | Deferred switch mechanism, not immediate |
| `vTaskSwitchContext` | Portable C | Only updates `pxCurrentTCB`; no register access |
| Hardware-saved regs (Cortex-M) | r0-r3, r12, LR, PC, xPSR | Auto stacked/unstacked by exception hardware |
| Manually-saved regs | r4-r11 (+ s16-s31 if FPU used) | Explicit `stmdb`/`ldmia` in the port asm |
| Measured here (POSIX sim) | 2.014 us/switch | Host-OS artifact — shape, not magnitude, transfers |

## Exercise

1. Reproduce the measurement above: clone `FreeRTOS/FreeRTOS-Kernel`,
   build the two-task yield-ping-pong program against the POSIX port, and
   confirm you get a similar per-switch overhead order of magnitude on your
   own machine — then explain in your own words why this number cannot be
   quoted as a Cortex-M switch time.
2. Find a real Cortex-M port's `port.c` (e.g. `GCC/ARM_CM4F`) and count the
   exact register pushes/pops in `xPortPendSVHandler` for a task that does
   *not* use the FPU versus one that does — compute the instruction-count
   delta directly from the assembly.
3. Explain, from first principles (not by looking it up), why PendSV is
   configured at the *lowest* priority rather than, say, the same priority
   as the tick interrupt — what specifically would break if it ran at a
   higher priority than an application ISR that calls a `...FromISR` API?
