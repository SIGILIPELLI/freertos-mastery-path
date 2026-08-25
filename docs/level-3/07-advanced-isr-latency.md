# Advanced ISR & Latency Design

Level 1 covered the basic rule: never call a blocking API from an ISR, use
the `...FromISR` variants. This module goes deeper into *why* those rules
exist mechanically, how interrupt latency actually decomposes into
measurable pieces, and a real, reproducible failure this course hit while
trying to build an ISR-latency benchmark on the POSIX simulator — which
turned out to be more instructive than a clean successful measurement would
have been.

## Where interrupt latency actually comes from

"Interrupt latency" is not one number — it's a chain, and each link has a
different owner and a different fix:

1. **Hardware response time**: cycles from the physical trigger to the CPU
   vectoring into the ISR — fixed by silicon, not FreeRTOS's concern.
2. **Interrupt masking**: `taskENTER_CRITICAL()`/`portDISABLE_INTERRUPTS()`
   anywhere in the system — including inside the kernel's own queue/list
   operations — delays *any* interrupt of equal or lower priority than the
   mask level from being taken at all until the critical section ends. This
   is why keeping critical sections short (Module 1's list operations are
   deliberately O(1) partly for this reason) is a latency concern, not just
   a throughput one.
3. **ISR execution time itself**: time spent actually running the handler
   before it hands off — this is exactly why the rule is "do the minimum in
   the ISR, defer the rest."
4. **Dispatch latency**: time from the ISR calling a `...FromISR` give
   (semaphore, queue, task notification) to the woken handler task actually
   running — this includes the PendSV-mediated context switch from Module 2,
   plus however long a higher-or-equal priority task already running has to
   finish or yield.

`configMAX_SYSCALL_INTERRUPT_PRIORITY` (Cortex-M) controls how much of link
2 is unavoidable: interrupts at or below this priority are masked during
kernel critical sections and can call FreeRTOS APIs; interrupts configured
*above* it run with kernel-comparable latency to real hardware response time
but must **never** call any FreeRTOS API at all, since the kernel isn't
masking them and can't protect its own data structures from them.

## Deferred-work: the standard fix for a slow ISR

```c
void ADC_IRQHandler(void) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    uint16_t sample = ADC->DR;                      // fast register read only
    xQueueSendFromISR(adcQueue, &sample, &xHigherPriorityTaskWoken);
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);   // switch immediately if needed
}
```

The pattern is always the same shape: read/acknowledge hardware state fast,
hand the data off through a `...FromISR` primitive, and use the returned
`xHigherPriorityTaskWokenByPost`/`pxHigherPriorityTaskWoken` output parameter
to request an immediate context switch (via `portYIELD_FROM_ISR`) if the
newly-woken task outranks whatever was interrupted — otherwise the switch
would be deferred until the next tick, adding up to a full tick period of
avoidable dispatch latency.

## A real failure encountered building this module's benchmark

This course's earlier modules verified real timing on the POSIX port
(Module 2's 2.014 us/switch measurement). The natural next experiment was an
ISR-dispatch-latency benchmark: spawn a raw POSIX thread to act as a "fake
ISR" firing `xSemaphoreGiveFromISR` on a timer, and measure how long a
FreeRTOS handler task took to wake and process it. **This was attempted on
this machine and it hung indefinitely** — the process never completed and
had to be killed.

The root cause, on inspection, is itself the module's most important trap:
the POSIX port's `...FromISR` implementation and its underlying
`wait_for_event.c` signaling mechanism are built around the port's own
notion of which pthreads are "FreeRTOS tasks" versus "the simulated
interrupt path" (driven by the port's internal timer thread and signal
handling) — calling `xSemaphoreGiveFromISR` from an arbitrary, separately
created `pthread_create` thread that the port doesn't recognize as part of
its own interrupt-simulation machinery is **outside the port's supported
usage** and produced a hang rather than a clean result or a documented
error. This is not a FreeRTOS logic bug — it's a real, reproducible
demonstration that **the POSIX simulator's `...FromISR` path only faithfully
represents real hardware ISR behavior when the "ISR" is simulated through
the port's own documented mechanism** (typically a signal handler installed
via the port's own APIs, or by calling the ISR-simulation function from
within the port's timer-tick signal context), not from an arbitrary
independent OS thread. Anyone reproducing ISR-latency experiments on this
port should route the simulated interrupt through the port's supported path
and expect this exact class of hang if they don't.

This is left in the module deliberately, instead of a clean result, because
it's a genuine, verified example of exactly the kind of simulator-vs-hardware
mismatch Level 2 Module 5 warned about in the abstract — "real ISR behavior
does not transfer to the POSIX simulator" — encountered concretely rather
than asserted.

## Priority inversion is a latency bug, not just a correctness one

Level 1 covered priority inheritance for mutexes as a *correctness*
mechanism (unbounded priority inversion can deadlock a system). Under this
module's lens, it's equally a *latency* mechanism: without inheritance, a
high-priority task's worst-case dispatch latency after an ISR wakes it is
unbounded (it depends on how long an unrelated medium-priority task happens
to run), which makes any WCET analysis (Level 4 Module 2) built on top of it
meaningless. `configUSE_MUTEXES` with real mutexes (not binary semaphores
used *as* a mutex, which do not inherit priority) is a hard prerequisite for
any latency-bounded design, not an optional correctness nicety.

## Traps

- **Calling any FreeRTOS API from an ISR configured above
  `configMAX_SYSCALL_INTERRUPT_PRIORITY`.** This is not a "the kernel might
  be slow" problem — it's undefined behavior, since the kernel's critical
  sections don't mask that interrupt priority at all, so it can preempt the
  kernel mid-list-manipulation (Module 1).
- **Forgetting `portYIELD_FROM_ISR`.** Sending to a queue/semaphore from an
  ISR without checking and acting on the higher-priority-task-woken flag
  means the switch waits for the next tick — silently adding up to one full
  tick period of latency that a correct call would have avoided entirely.
- **Simulating "ISR" behavior from an arbitrary thread on the POSIX port**,
  exactly as this module's own benchmark attempt demonstrated — expect a
  hang or undefined behavior, not a graceful degradation, and never trust a
  POSIX-port ISR-latency number without confirming the simulated interrupt
  went through the port's own supported mechanism.
- **Using a binary semaphore where a real mutex is needed for priority
  inheritance.** They look interchangeable in code (`xSemaphoreTake`/`Give`
  on both) but only a real mutex (`xSemaphoreCreateMutex`) participates in
  priority inheritance — this is an easy, latency-invisible-until-it-isn't
  mistake.
- **Measuring "typical" ISR latency and calling it done.** Latency analysis
  needs a worst-case bound (Level 4 Module 2's formal treatment), not an
  average — a system that's fast 999 times out of 1000 and blows its budget
  once is still a real-time failure.

## Cheat sheet

| Latency component | Owner | Typical fix |
|---|---|---|
| Hardware response | Silicon | Fixed; not FreeRTOS's concern |
| Interrupt masking | Kernel critical sections | Keep critical sections short (Module 1) |
| ISR execution time | Your ISR code | Defer work via `...FromISR` |
| Dispatch latency | Scheduler + context switch | `portYIELD_FROM_ISR`, correct priorities |
| Unbounded inversion | Missing priority inheritance | Real mutexes, not binary semaphores |
| Simulated ISR on POSIX port | Port's own signal/thread machinery | Only simulate via the port's documented path — verified here that an arbitrary pthread hangs |

## Exercise

1. Reproduce this module's failure: write the "fake ISR as an independent
   pthread calling `xSemaphoreGiveFromISR`" benchmark exactly as described,
   confirm it hangs on your machine too, then fix it by routing the
   simulated interrupt through the POSIX port's own documented ISR
   simulation mechanism, and get a real (if still host-timing-only) latency
   number out of it.
2. On real Cortex-M hardware, configure one interrupt above
   `configMAX_SYSCALL_INTERRUPT_PRIORITY` and deliberately call a FreeRTOS
   API from it — observe what actually happens (it will likely not be a
   clean, obvious failure) and explain why this class of bug is especially
   dangerous in the field.
3. Build a small system with a low-priority task holding a plain binary
   semaphore, a high-priority task blocked waiting for that semaphore, and a
   medium-priority task that keeps the CPU busy — confirm unbounded
   inversion occurs, then swap the binary semaphore for a real mutex and
   confirm priority inheritance bounds the high-priority task's wait.
