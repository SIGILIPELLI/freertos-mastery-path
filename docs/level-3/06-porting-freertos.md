# Porting FreeRTOS to a New MCU

Every vendor SDK you've used through Level 2 (ESP-IDF, an S32K BSP) already
ships a working FreeRTOS port. This module covers what's actually inside
that port layer, using the real POSIX simulator port examined throughout
this course as the running example of a *complete, working* port whose full
source is small enough to read end to end — and generalizing from it to what
changes for a bare-metal Cortex-M target.

## The three files every port provides

Every FreeRTOS port, regardless of architecture, implements the same
three-piece contract:

1. **`portmacro.h`** — type definitions and macros: `portBASE_TYPE`,
   `portSTACK_TYPE`, `portMAX_DELAY`, `portYIELD()`,
   `portDISABLE_INTERRUPTS()`/`portENABLE_INTERRUPTS()`,
   `portENTER_CRITICAL()`/`portEXIT_CRITICAL()`.
2. **`port.c`** — `pxPortInitialiseStack` (lays out a brand-new task's
   initial stack frame so the first "resume" looks like a normal context
   restore), `xPortStartScheduler` (starts the tick source and performs the
   very first context switch into the first task), and the tick/switch
   interrupt handlers.
3. A **linker/startup interaction** — the initial stack pointer, vector
   table entries for the tick timer and the switch-request mechanism
   (PendSV on Cortex-M; a POSIX signal in the simulator port), and any
   architecture-specific interrupt priority setup.

## `pxPortInitialiseStack`: making a new task look like a saved one

This is the single most important function to understand when porting,
because it's the only place a *brand-new* task (one that has never actually
run and been context-switched-out) has to produce a stack frame
indistinguishable from one Module 2's context-switch-in code expects to
restore:

```c
// Simplified Cortex-M shape — the real one in port.c is denser
StackType_t *pxPortInitialiseStack(StackType_t *pxTopOfStack,
                                    TaskFunction_t pxCode, void *pvParameters) {
    pxTopOfStack--;
    *pxTopOfStack = portINITIAL_XPSR;                 // xPSR — Thumb bit set
    pxTopOfStack--;
    *pxTopOfStack = ((StackType_t) pxCode) & 0xfffffffeUL;  // PC — task entry point
    pxTopOfStack--;
    *pxTopOfStack = (StackType_t) prvTaskExitError;   // LR — if task ever returns
    pxTopOfStack -= 5;                                 // r12, r3, r2, r1 (skip)
    *pxTopOfStack = (StackType_t) pvParameters;        // r0 — task's argument
    pxTopOfStack -= 8;                                 // r11..r4 (manually saved regs)
    return pxTopOfStack;
}
```

This must exactly mirror what `xPortPendSVHandler` (Module 2) expects to pop
on that task's *first* switch-in — get one register's position wrong and the
task either starts with garbage in a register, jumps to the wrong address on
"return," or corrupts the very next task's context when the mismatched
push/pop count shifts the stack pointer by the wrong amount. This is the
number one source of "new port boots, first task runs, then everything
crashes on the second switch" bugs.

## The POSIX port as a complete worked reference

The `portable/ThirdParty/GCC/Posix` port used throughout this course
implements the same three-piece contract with wildly different mechanism,
which is exactly why comparing them clarifies what's essential versus
incidental:

| Contract piece | Cortex-M port | POSIX simulator port |
|---|---|---|
| "Task" backing | Real CPU register context on a stack | `pthread_t`, native OS thread |
| Tick source | Hardware timer (SysTick), fires a real interrupt | `setitimer`/`SIGALRM`-family signal |
| Switch trigger | PendSV exception | A signal handler calling into the port's own scheduler logic, using condition variables (`wait_for_event.c`) to suspend/resume threads |
| Critical section | Disable interrupts (`cpsid i` / `basepri`) | Block signals on the current thread |
| Stack | Real MCU RAM, sized by `configMINIMAL_STACK_SIZE` | A `pthread` stack, OS-managed |

Every port, no matter how different the mechanism, has to answer the same
five questions this table poses. This is why reading a second, very
different port is one of the fastest ways to internalize which parts of
`tasks.c`'s expectations are architectural necessities and which are
Cortex-M-specific incidental detail.

## The minimal porting checklist

1. Define `portSTACK_TYPE`, `portBASE_TYPE`, byte order
   (`portBYTE_ALIGNMENT`) for the target.
2. Implement `pxPortInitialiseStack` matching your switch-in code exactly.
3. Implement the tick interrupt: increment the tick, call
   `xTaskIncrementTick()`, request a switch if it returns `pdTRUE`.
4. Implement the switch mechanism itself (PendSV-equivalent, or whatever the
   architecture provides for a software-triggerable, priority-controllable
   exception/trap).
5. Implement `portDISABLE_INTERRUPTS`/`portENABLE_INTERRUPTS` and the nested
   critical-section counter (`uxCriticalNesting`) correctly — critical
   sections must nest safely, since kernel code calls them from within other
   critical sections routinely.
6. Supply `vApplicationMallocFailedHook`/`vApplicationStackOverflowHook` if
   the corresponding config options are enabled (the same requirement that
   surfaced immediately when building the POSIX port in Level 2 Module 5 —
   this bites on every port, not just the simulator).
7. Get one task running and printing/toggling something observable *before*
   adding a second task — the single-task case exercises
   `pxPortInitialiseStack` and `xPortStartScheduler` without yet exercising
   the switch-in-from-a-real-context path, isolating the two most common
   classes of bugs from each other.

## Traps

- **Getting register order right for the first task but wrong for the
  switch-back path.** The stack layout `pxPortInitialiseStack` builds must
  match what the switch-in code pops *exactly* — a bug here is often
  invisible on the very first task start (which has its own separate
  "initial start" code path in `xPortStartScheduler`) and only manifests on
  the first real switch *back* to a previously-run task.
- **Forgetting nested critical sections.** `portENTER_CRITICAL`/
  `portEXIT_CRITICAL` must track nesting depth (`uxCriticalNesting`) and only
  actually re-enable interrupts when the count returns to zero — kernel code
  frequently nests critical sections, and a port that naively enables
  interrupts on the first `EXIT` call while still logically inside an outer
  critical section reintroduces exactly the races the section was meant to
  prevent.
- **Choosing a tick/switch interrupt priority incorrectly.** As Module 2
  covered for PendSV specifically, the switch-trigger mechanism generally
  needs to run at the *lowest* priority in the system so it never delays
  higher-priority interrupt work — a new port that doesn't replicate this on
  its own architecture's equivalent mechanism can silently corrupt the
  latency guarantees the rest of this course assumes.
- **Skipping the "one task only" bring-up step.** Jumping straight to a
  multi-task, multi-priority test on a new port conflates initial-start bugs
  with switch-path bugs and makes root-causing dramatically harder than
  isolating them in sequence.
- **Assuming a working single-core port trivially extends to SMP.** As
  Module 3 covered, SMP needs real cross-core spinlocks and per-core state
  (`pxCurrentTCBs[coreID]` instead of a single `pxCurrentTCB`) — porting to a
  new *single-core* architecture and porting an existing single-core design
  to *multiple* cores are different, only partially overlapping exercises.

## Cheat sheet

| Porting artifact | Purpose | Where the bug usually hides |
|---|---|---|
| `portmacro.h` types/macros | Basic ABI contract | Wrong stack growth direction or alignment |
| `pxPortInitialiseStack` | New task's fake "saved" context | Register order mismatch with switch-in code |
| Tick handler | Drives `xTaskIncrementTick` | Wrong interrupt priority, missed switch request |
| Switch mechanism (PendSV-equivalent) | Actual register save/restore | Priority too high, incomplete register set |
| `portENTER/EXIT_CRITICAL` | Mutual exclusion primitive | Missing nesting counter |
| Application hooks | Required by enabled config options | Missing hook → link error, easy but easy to miss |

## Exercise

1. Read `portable/ThirdParty/GCC/Posix/port.c` end to end (it's the
   reference used throughout this course) and map every function in it to
   the six-item checklist above — note explicitly which checklist items the
   POSIX port satisfies via signals/pthreads instead of hardware.
2. Find a real Cortex-M port (`GCC/ARM_CM3` is a good compact reference) and
   do the same mapping — then write, in your own words, the single biggest
   structural difference between how the two ports solve "request a
   context switch."
3. Without copying an existing port, write out (pseudocode is fine) what
   `pxPortInitialiseStack` would need to look like for a hypothetical
   architecture with 16 general-purpose registers and no hardware exception
   stacking (i.e., a port that must manually save *every* register, not just
   the subset Cortex-M's hardware doesn't already handle) — and explain how
   this changes the PendSV-equivalent handler's instruction count versus
   Module 2's Cortex-M example.
