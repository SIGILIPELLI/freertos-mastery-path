# Watchdogs & Task Supervision

Every pattern so far assumes tasks behave: they yield, they don't deadlock,
they don't spin forever inside a bad driver loop. Production firmware
can't assume that — a stuck I2C bus, a corrupted pointer, or a firmware bug
under an edge case can hang a task or the whole system. A **watchdog**
is the last line of defense: independent hardware (or a supervising task)
that resets the system if it stops seeing expected liveness signals in
time.

## Hardware watchdog basics

A hardware watchdog timer (WDT) is a peripheral, usually clocked
independently of the main CPU clock, that resets the chip unless
periodically "petted" (refreshed) by software:

```cpp
// Generic shape — exact API is vendor-specific (IWDG on STM32, WDT on AVR, etc.)
void watchdogTask(void *pv) {
  hw_watchdog_init(pdMS_TO_TICKS(2000));  // must be petted within 2s or reset
  for (;;) {
    // only pet if EVERY supervised task has proven liveness since last pet —
    // see the software supervision layer below
    if (allTasksHealthy()) {
      hw_watchdog_pet();
    }
    vTaskDelay(pdMS_TO_TICKS(500));
  }
}
```

The critical design decision is *what* triggers the pet. Petting
unconditionally on a fixed schedule (as long as `watchdogTask` itself
runs) only proves the watchdog task is alive — it says nothing about
whether the tasks that actually matter are still making progress. A stuck
sensor task with the watchdog task happily petting on schedule is exactly
the failure mode a watchdog exists to catch, silently defeated.

## Software supervision: liveness tokens per task

The fix is to have each supervised task report its own liveness, and have
the watchdog task require *all* of them to have reported recently before
it pets the hardware:

```cpp
#define NUM_SUPERVISED_TASKS 3
static volatile TickType_t lastAlive[NUM_SUPERVISED_TASKS];

void sensorTask(void *pv) {
  for (;;) {
    doWork();
    lastAlive[TASK_SENSOR] = xTaskGetTickCount();   // prove liveness EVERY iteration
    vTaskDelay(pdMS_TO_TICKS(100));
  }
}

bool allTasksHealthy(void) {
  TickType_t now = xTaskGetTickCount();
  for (int i = 0; i < NUM_SUPERVISED_TASKS; i++) {
    if (now - lastAlive[i] > pdMS_TO_TICKS(1500)) {
      return false;   // this task hasn't proven progress recently — don't pet
    }
  }
  return true;
}
```

Now a genuinely stuck `sensorTask` stops updating `lastAlive[TASK_SENSOR]`,
`allTasksHealthy()` returns false, the hardware watchdog stops being
petted, and the whole system resets — recovering from a hang that no
amount of internal error handling caught. This is the same idea behind
ESP-IDF's Task Watchdog Timer (Module 4), generalized to any port: task
liveness tokens, checked before petting real hardware.

## What a watchdog should and shouldn't reset

A watchdog reset is a blunt instrument — the whole system reboots,
in-flight state is lost, and whatever caused the hang recurs unless it was
transient. Before shipping one, decide:

- **Log before reset**: write the reason (which task's token was stale) to
  a persistent location (a small battery-backed RAM region, or flash) that
  survives the reset, so the *next* boot can report what happened.
- **Don't paper over a fixable bug**: a watchdog that silently recovers
  from a recurring hang every few minutes, in the field, unnoticed, is
  worse than a watchdog that resets rarely because the underlying bug got
  fixed. Treat every real-world watchdog reset as an incident to
  investigate, not a feature working as intended.
- **Reset scope**: some designs distinguish a full chip reset (window
  watchdog trips) from a narrower recovery (kill and restart just the
  hung task, via `vTaskDelete` + recreate) for hangs known to be isolated
  to one subsystem — narrower recovery avoids dropping unrelated in-flight
  work, but only apply it when you're confident the hang can't have
  corrupted shared state used elsewhere.

## Task health beyond "did it call a function recently"

A task can call its liveness-token update on schedule while still being
logically wrong — stuck in a bad state, retrying a failed operation
forever, but still executing the loop body. Layer in domain checks where
they're cheap:

```cpp
void sensorTask(void *pv) {
  int consecutiveFailures = 0;
  for (;;) {
    if (readSensor(&value) != OK) {
      consecutiveFailures++;
      if (consecutiveFailures > 10) {
        logFault(FAULT_SENSOR_UNRESPONSIVE);
        // still update the liveness token — the TASK isn't hung, the SENSOR is —
        // but surface the fault through a separate channel (Module 8's gatekeeper)
      }
    } else {
      consecutiveFailures = 0;
    }
    lastAlive[TASK_SENSOR] = xTaskGetTickCount();
    vTaskDelay(pdMS_TO_TICKS(100));
  }
}
```

This distinguishes "the task is hung" (watchdog's job) from "the task is
alive but the thing it depends on is broken" (application-level fault
reporting, potentially through the gatekeeper logging pattern from
Module 8) — conflating the two either resets the system for a recoverable
sensor glitch, or hides a genuinely stuck task behind superficially normal
liveness updates.

## Traps

- **Petting on a fixed schedule regardless of task health**: the single
  most common watchdog anti-pattern — it makes the watchdog decorative.
- **Watchdog timeout shorter than legitimate worst-case task latency**: a
  task that's merely blocked briefly on a legitimately slow but bounded
  operation (a flash erase, a slow I2C transaction) can trip a
  too-aggressive timeout — size the timeout against measured worst-case,
  not typical-case, latency.
- **Petting from an ISR on a fixed timer, decoupled from any task
  liveness check**: this reintroduces the "always alive from the
  watchdog's perspective" problem even more directly than a naive task.
- **No persisted crash reason**: without logging *why* before reset, every
  watchdog-triggered reboot looks identical in the field, making the
  underlying bug far harder to diagnose from returned/support units.
- **Recovering too eagerly and hiding a real defect**: automatic recovery
  without any escalation path (e.g., counting resets-per-hour and
  eventually surfacing a persistent fault state instead of silently
  rebooting forever) can mask a hardware or firmware defect for a long
  time in the field.

## Cheat sheet

| Concept | Purpose |
|---|---|
| Hardware WDT | Independent peripheral; resets chip if not petted in time |
| Fixed-schedule pet | Proves the watchdog task is alive — proves nothing else |
| Per-task liveness token | `lastAlive[task] = xTaskGetTickCount()` each iteration |
| `allTasksHealthy()` gate | Only pet hardware if every supervised task proved recent progress |
| ESP-IDF Task Watchdog Timer | Built-in equivalent, subscribes idle tasks by default (Module 4) |
| Persist-before-reset | Log the stale task/reason to non-volatile/battery-backed memory pre-reset |
| Fault vs. hang | Update liveness token even on a recoverable domain fault; report the fault separately |

## Exercise

1. Implement the liveness-token pattern for three tasks running at
   different periods. Deliberately hang one (an infinite loop with no
   yield) and confirm the watchdog task correctly stops petting and the
   system resets — measure the actual time from hang to reset against
   your configured timeout.
2. Add persisted-reason logging (a static/global struct is enough for the
   POSIX-port simulator; real hardware would use a no-init RAM section)
   and confirm the next boot can report which task caused the reset.
3. Extend `sensorTask` with the consecutive-failure counter shown above.
   Simulate a sensor that always fails (but the task keeps running) and
   confirm the system does NOT reset (task is alive) while a separate
   fault log correctly reports the sensor as unresponsive.
