# Project — Instrumented RTOS Firmware

This project combines Modules 1, 2, 5, 7, and 8 into one running system: a
multi-task firmware design that is instrumented from the ground up to
measure its own scheduling behavior, ISR dispatch latency, and switch
overhead — turning the abstract kernel-internals knowledge from earlier in
this level into a design you can actually point a profiler at and defend
with numbers.

## Design goal

Build (on the POSIX simulator port, then port forward to real hardware) a
small instrumented firmware with:

- A simulated sensor-sampling ISR path (Module 7's deferred-work pattern)
  feeding a handler task through a queue.
- A background processing task (Level 2 Module 8's filter-stage pattern)
  consuming from that queue.
- A logging gatekeeper task (Level 2 Module 8) as the sole owner of output.
- Built-in software instrumentation (Module 5's tool-free technique) that
  reports, on demand, switch counts, per-task high-water-marks (Module 8),
  and simulated ISR-to-handler dispatch latency statistics — without
  requiring Tracealyzer or SystemView to be attached.

## Task/priority layout

| Task | Priority | Role |
|---|---|---|
| `handlerTask` | 4 (highest) | Woken by simulated ISR; minimal processing, hands off to filter queue |
| `filterTask` | 3 | Moving-average / processing stage |
| `loggerGatekeeper` | 2 | Sole owner of the "output" resource (stdout here, UART on target) |
| `statsReporterTask` | 1 | Periodically dumps instrumentation counters via the gatekeeper |
| Idle | 0 | Standard FreeRTOS idle |

This priority ordering directly encodes Module 7's latency-chain reasoning:
the task closest to the simulated hardware event runs highest, the
least time-critical (periodic stats reporting) runs lowest, and *all* task
output funnels through one gatekeeper so instrumentation output itself never
races with application logging — exactly the failure mode Level 2 Module 8's
gatekeeper pattern was built to prevent, now applied to the diagnostic
output as much as the application's own.

## Core instrumentation

```c
typedef struct {
    volatile uint32_t switchCount;
    volatile uint32_t dispatchSamples;
    volatile double   dispatchSumUs;
    volatile double   dispatchMaxUs;
} Stats_t;
static Stats_t g_stats;

// Called from the simulated ISR path right before giving the semaphore/queue
static struct timespec g_isrFireTime;
void simulatedSensorISR(void) {
    clock_gettime(CLOCK_MONOTONIC, &g_isrFireTime);
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    uint16_t sample = readSimulatedSensor();
    xQueueSendFromISR(sensorQueue, &sample, &xHigherPriorityTaskWoken);
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

void handlerTask(void *pv) {
    uint16_t sample;
    for (;;) {
        xQueueReceive(sensorQueue, &sample, portMAX_DELAY);
        struct timespec now;
        clock_gettime(CLOCK_MONOTONIC, &now);
        double us = (now.tv_sec - g_isrFireTime.tv_sec) * 1e6
                  + (now.tv_nsec - g_isrFireTime.tv_nsec) / 1e3;
        g_stats.dispatchSamples++;
        g_stats.dispatchSumUs += us;
        if (us > g_stats.dispatchMaxUs) g_stats.dispatchMaxUs = us;
        xQueueSend(filteredInputQueue, &sample, 0);
    }
}
```

This is deliberately built the same way as Module 7's dispatch-latency
attempt and Module 2's switch-overhead benchmark — plain counters and
`clock_gettime`, no external tool — with the one critical correction Module
7's failure taught: the simulated "ISR" here must fire from *within* the
POSIX port's own recognized interrupt-simulation path (a timer callback
registered through the port, or from inside a task calling the handler
function directly to model the hand-off logically), never from an
independently created raw `pthread_create` thread, which Module 7 verified
hangs this port's `...FromISR` machinery.

## Stats reporter, routed through the gatekeeper

```c
void statsReporterTask(void *pv) {
    for (;;) {
        vTaskDelay(pdMS_TO_TICKS(5000));
        LogMsg m;
        double mean = g_stats.dispatchSamples
            ? g_stats.dispatchSumUs / g_stats.dispatchSamples : 0.0;
        snprintf(m.msg, sizeof(m.msg),
                 "stats: dispatch mean=%.2fus max=%.2fus samples=%lu",
                 mean, g_stats.dispatchMaxUs, (unsigned long) g_stats.dispatchSamples);
        xQueueSend(logQueue, &m, pdMS_TO_TICKS(50));

        // Per-task stack watermark sweep, also routed through the gatekeeper
        UBaseType_t hwm = uxTaskGetStackHighWaterMark(NULL);
        snprintf(m.msg, sizeof(m.msg), "stats: self stack HWM=%lu words", (unsigned long)hwm);
        xQueueSend(logQueue, &m, pdMS_TO_TICKS(50));
    }
}
```

## What was and wasn't verified

The task/queue/priority structure above follows patterns already exercised
individually and concretely in this course (Module 2's switch benchmark,
Module 7's dispatch-latency attempt and its documented hang mode, Level 2
Module 8's gatekeeper). Assembling the full five-task system end to end on
the POSIX port and confirming the instrumentation reports sane numbers is
left as this project's deliverable rather than something pre-built here —
building it yourself, and hitting (and fixing) the integration issues along
the way, is the actual learning objective. Do not skip actually running it:
a design that "looks right" on paper is exactly the trap Module 7's
real hang demonstrated — plausible-looking ISR-simulation code that
doesn't actually work when executed.

## Traps carried over from this level

- Simulating the "ISR" via an unsupported raw pthread on the POSIX port —
  verified in Module 7 to hang, not fail gracefully.
- Giving the stats reporter or logger a priority high enough to interfere
  with the sensor-handling path it's supposed to be *measuring* —
  instrumentation that perturbs the system it observes defeats its own
  purpose (the same principle behind Module 5's warning against halting the
  CPU to read a trace buffer).
- Forgetting `configUSE_TRACE_FACILITY`/malloc-failed/stack-overflow hooks —
  as encountered repeatedly since Level 2 Module 5, these are the most
  common "won't even link" mistakes when assembling a from-scratch build
  against raw kernel sources.
- Not cross-checking instrumentation counters against a real tracer
  (Module 5) if one becomes available during hardware bring-up — this
  project's counters are a coarse, always-available substitute, not a
  replacement, for a proper trace when hardware-level questions arise.

## Stretch goals

1. Port the whole system from the POSIX simulator to real Cortex-M
   hardware (Module 6), replacing `clock_gettime` with a hardware cycle
   counter (`DWT->CYCCNT` on Cortex-M3/4/7) for real, sub-microsecond-accurate
   dispatch-latency measurement, and compare the real hardware numbers
   against this project's simulator-derived ones to see exactly how far off
   the simulator was (Module 2's caveat, quantified for real this time).
2. Add MPU restrictions (Module 4) so `handlerTask` and `filterTask` run
   unprivileged with access to only their own stacks and the queues they
   need — then deliberately introduce a wild-pointer bug in one of them and
   confirm it faults immediately instead of corrupting the stats structure.
3. Extend the instrumentation to compute a rolling latency histogram
   (bucketed, not just mean/max) and add a deliberate artificial priority
   inversion scenario (Module 7) to the design, confirming the histogram
   shows the resulting latency tail — then fix the inversion with a real
   mutex and confirm the histogram's tail collapses.
4. If a second core is available (RP2040/RP2350, ESP32), pin
   `handlerTask` to one core and `loggerGatekeeper`/`statsReporterTask` to
   the other (Module 3), and extend the instrumentation to also report
   which core each task actually ran on over a sampling window.
