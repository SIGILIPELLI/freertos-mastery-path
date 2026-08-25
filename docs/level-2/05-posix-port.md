# POSIX/Linux Simulation Port

Every prior module assumed real hardware — flash it, watch an LED, measure
current draw. But most FreeRTOS *design* work (task decomposition, queue
sizing, priority assignment) doesn't need a chip at all. The **POSIX
simulator port** (`portable/ThirdParty/GCC/Posix` in FreeRTOS-Kernel) runs
the real scheduler, the real task/queue/timer code, on your development
machine, mapping each FreeRTOS task onto a POSIX thread and using signals
to simulate preemption. This module was built and run on this machine
during writing — the build commands and output below are real, not
illustrative.

## What was actually verified here

A minimal build was assembled directly against the kernel sources (no demo
project scaffolding) on macOS with Apple clang (invoked as `gcc`), linking
against `pthread`:

```bash
gcc -o test main.c \
  tasks.c queue.c list.c timers.c event_groups.c stream_buffer.c \
  portable/MemMang/heap_3.c \
  portable/ThirdParty/GCC/Posix/port.c \
  portable/ThirdParty/GCC/Posix/utils/wait_for_event.c \
  -I config -I include \
  -I portable/ThirdParty/GCC/Posix -I portable/ThirdParty/GCC/Posix/utils \
  -lpthread
```

with a `main.c` creating one task that prints and delays three times, then
calls `vTaskEndScheduler()`:

```
hello from task1 0
hello from task1 1
hello from task1 2
scheduler ended
```

Two things had to be added before it linked, and both are worth calling
out because they're easy to forget on *any* port, not just this one:
`vApplicationMallocFailedHook()` and `vApplicationStackOverflowHook()` are
referenced unconditionally by `heap_3.c` and `tasks.c` once
`configUSE_MALLOC_FAILED_HOOK`/`configCHECK_FOR_STACK_OVERFLOW` are enabled
in `FreeRTOSConfig.h` — the linker error ("symbol not found") names them
precisely, so this is a quick fix once seen, not a mystery.

## How the simulation maps concepts to your OS

| FreeRTOS concept | POSIX-port reality |
|---|---|
| Task | A `pthread`, started suspended, released by the scheduler |
| Preemption / tick interrupt | A POSIX interval timer (`SIGALRM`-family signal) drives the tick |
| Critical section | Signal masking on the current thread, not real interrupt disable |
| Task priority | Advisory to the port's own scheduler logic — **not** OS thread priority/`nice` |
| "ISR" (`...FromISR` calls) | Ordinary function calls from a signal handler or another thread in the demo harness — there is no real interrupt controller |
| Stack overflow | Detected the same way as real hardware (pattern/watermark checks) — but a genuine POSIX thread stack overrun can also just segfault outside FreeRTOS's own checks |

The critical honesty point: this is a **simulation for logic and design
validation**, not a timing model. Tick "interrupts" are delivered by the
host OS's scheduler on a general-purpose kernel with its own preemption,
so a task in this simulator can be involuntarily delayed by unrelated host
load in ways a real MCU never would be. It's excellent for validating
*that a design is race-free and logically correct*; it is not a substitute
for measuring real timing on target hardware.

## What's genuinely useful here

- **Design and code review before hardware exists** — write and exercise
  task/queue/priority logic while board bring-up is still in progress.
- **Fast iteration** — no flashing, no JTAG, a native binary that runs and
  exits in milliseconds; good for exercising `vTaskDelay`/queue timeout
  logic and printing diagnostics with a normal debugger (`lldb`/`gdb`)
  attached to a normal process.
- **CI**: a POSIX-port build can run correctness tests for pure scheduling
  logic (task ordering, queue behavior, priority-based preemption
  sequencing) on every commit, with no hardware in the loop.
- **Teaching/learning** — every code sample so far in this course could, in
  principle, be adapted to build and run this way for experimentation.

## What does not transfer to real hardware

- Any timing number (jitter, worst-case latency, actual tick-to-tick
  interval) — the host OS, not a deterministic timer peripheral, drives
  the simulated tick.
- Real ISR behavior — `...FromISR` calls in the simulator are just regular
  calls with no real interrupt-context restrictions being enforced by the
  hardware; a Level 1 Module 8 rule like "never call a blocking API from
  ISR context" is not something this simulator can catch for you.
- Power/sleep behavior (Module 3) — there is no low-power hardware to
  model.
- Anything MPU/MMU or vendor-specific (Module 4's ESP-IDF dual-core
  behavior has no equivalent here — the simulator is single-scheduler).

## Traps

- **Treating simulated priorities as OS thread priorities**: the port maps
  FreeRTOS priority to its own internal scheduling decisions using signals
  to suspend/resume threads — it does **not** call `pthread_setschedparam`
  to elevate real OS scheduling priority, so don't reason about starvation
  or priority inversion using host-OS scheduler intuition.
  the outputs above only prove logical ordering, not real-time behavior.
- **Missing hook functions**: as shown above, enabling
  `configUSE_MALLOC_FAILED_HOOK` / `configCHECK_FOR_STACK_OVERFLOW`
  requires supplying the corresponding hook — this bites on real ports too,
  but the simulator's fast build/link cycle is exactly where it's cheapest
  to discover and fix these config/implementation mismatches early.
- **Forgetting `-lpthread`**: the port is built entirely on POSIX threads
  and condition variables; omitting the link flag produces undefined
  reference errors for `pthread_*` symbols that look unrelated to
  FreeRTOS at first glance.
- **Running signal-unsafe code from the "ISR" simulation path**: because
  there's no real interrupt hardware, some demo harnesses simulate ISRs via
  real POSIX signal handlers — and real signal handlers have their own
  strict async-signal-safety rules (no `malloc`, no `printf`) independent
  of and stricter than FreeRTOS's own ISR-safety rules.
- **Believing "it worked in simulation" ends the verification job**: the
  simulator is a design/logic gate, not a hardware sign-off — treat a
  passing simulator run as "ready to try on real hardware," not "done."

## Cheat sheet

| Aspect | Real hardware port | POSIX simulator port |
|---|---|---|
| Task backing | Real stack + register context switch | `pthread_t` per task |
| Tick source | Hardware timer peripheral | POSIX interval timer / signal |
| Priority effect | Real preemptive scheduling by the kernel | FreeRTOS's own logic, layered over signal-based thread suspension |
| ISR | Real interrupt controller, real ISR-safe API rules enforced by hardware | Ordinary function call or signal handler — rules are *documented*, not *enforced* by hardware |
| Timing fidelity | Ground truth | Host-OS-dependent, not deterministic |
| Best use | Final validation, power/timing measurement | Early design validation, CI, fast iteration |
| Build here | N/A (toolchain per target) | Verified: `gcc` + `-lpthread`, needs `heap_3.c` (or another heap) + both application hooks |

## Exercise

1. Reproduce the build above: clone `FreeRTOS/FreeRTOS-Kernel`, write a
   minimal `FreeRTOSConfig.h` and `main.c` with two tasks at different
   priorities printing timestamps, and confirm the higher-priority task's
   output interleaves as expected.
2. Add a queue between two simulated tasks (Level 1 Module 4's
   producer/consumer) and confirm blocking/timeout behavior works
   identically to the description in that module — but time one full
   producer/consumer cycle and compare it to the nominal delay you
   requested, to see host-OS jitter first-hand.
3. Deliberately starve a lower-priority task with a higher-priority
   task that never blocks, and observe what actually happens in this
   simulator versus what you'd predict on real hardware — then explain
   why the simulator's behavior here should not be trusted as a timing
   guarantee.
