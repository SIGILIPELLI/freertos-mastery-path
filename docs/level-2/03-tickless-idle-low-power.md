# Tickless Idle & Low Power

By default FreeRTOS wakes the CPU on every tick interrupt — typically every
1 ms — whether or not any task needs to run. On a battery-powered device
that "wake up, check, go back to sleep" cycle can dominate power draw even
when the system is 99% idle. **Tickless idle** mode lets the idle task tell
the hardware "nothing is scheduled for the next N ticks — stop the tick
interrupt and let the CPU sleep for real," waking exactly when the next
timer or event actually needs attention.

## How it works: suppressing the tick

Normally the idle task just spins (`portSUPPRESS_TICKS_AND_SLEEP` never
called). With tickless idle enabled, when the scheduler determines no task
is ready and finds the next unblock time is more than one tick away, it
calls into `portSUPPRESS_TICKS_AND_SLEEP()`:

```c
// FreeRTOSConfig.h
#define configUSE_TICKLESS_IDLE  1   // 1 = whenever possible, 2 = ESP-IDF's "always" style variant
```

The sequence, per the portable layer's default implementation:

1. Idle task runs, scheduler computes `xExpectedIdleTime` — how many ticks
   until the next task is due to wake (from a delay, timer, or blocked
   queue timeout).
2. If that's below `configEXPECTED_IDLE_TIME_BEFORE_SLEEP` (default varies
   by port), it's not worth the sleep/wake overhead — busy-idle as normal.
3. Otherwise, the port stops the periodic tick source, reconfigures a
   single-shot low-power timer for the expected idle duration, and puts the
   core into a sleep mode (`WFI`, or a deeper low-power state on ports that
   support it).
4. On wake (either the low-power timer firing, or any other interrupt), the
   port corrects `xTickCount` for the number of ticks that were "missed"
   while asleep, based on how long the CPU actually slept, then resumes
   normal ticking.

That correction step is exactly why this needs port-specific support: the
kernel must know how to read elapsed time from a hardware timer that kept
running (or can report elapsed time) through the sleep, and it must
guarantee no interrupt is lost in the process.

## Application hooks around sleep

Two hooks let application code participate in the sleep decision and in
power-domain management:

```c
// Optional: veto or shorten a sleep, e.g. because a peripheral transaction
// is mid-flight and must not be interrupted by a clock change.
void vApplicationDeepSleepHook(void) {
  if (uartTransactionInFlight()) {
    // some ports check a global/flag here instead of a dedicated hook —
    // consult your port's documentation for the exact veto mechanism
  }
}

// Called by the port immediately before/after actually sleeping —
// typical use: switch peripheral clocks, disable unneeded domains.
void vApplicationSleepPriorToSleep(void)  { disable_unneeded_peripherals(); }
void vApplicationSleepAfterSleep(void)    { restore_peripheral_clocks(); }
```

Exact hook names and availability are port-specific (ESP-IDF, for example,
layers its own power-management API — `esp_pm_configure()` — on top of
this rather than exposing the raw hooks above). Always check your port's
`port.c` for the precise integration points before relying on a hook name.

## The interrupt-latency tradeoff

Waking from a deep sleep state is not free — it costs cycles to restore
clocks, PLLs, and peripheral state before the CPU can service an interrupt.
That directly increases worst-case interrupt latency, which matters if the
system has a hard real-time deadline on an external event:

```c
// Trade: allow tickless idle, but never let it choose a sleep depth
// deeper than what your worst-case interrupt-latency budget allows.
#define configEXPECTED_IDLE_TIME_BEFORE_SLEEP  2   // ticks — tune, don't guess
```

A system with a 1 ms hard deadline on a GPIO interrupt cannot afford a
sleep mode with a 5 ms wake latency, even if it saves substantial power —
measure your specific low-power state's wake latency against your tightest
ISR deadline before enabling anything deeper than the shallowest sleep
mode your platform offers.

## Traps

- **Assuming tick-based delays stay tick-accurate across a sleep**: the
  kernel corrects `xTickCount` for elapsed sleep time, but any code reading
  a *hardware* timer/counter directly during that window (not going
  through the RTOS tick) may see a discontinuity if that timer itself was
  gated during sleep. Route all timing through `xTaskGetTickCount()` or an
  always-running hardware timer, never a clock that pauses.
- **`configUSE_TICKLESS_IDLE` alone does nothing without port support**:
  not every FreeRTOS port implements `portSUPPRESS_TICKS_AND_SLEEP` — on
  ports without it, the macro compiles but the idle task never actually
  sleeps deeper than a `WFI`/spin. Check the port layer, not just the
  config header.
- **Priority inversion via missed ticks**: a task blocked with a short
  relative timeout (`pdMS_TO_TICKS(2)`) can wake later than expected if the
  system just entered a deep sleep and the wake-up correction has coarser
  granularity than the timeout itself — don't rely on sub-tick timing
  precision immediately after a tickless sleep window.
- **Peripheral clocks disabled mid-transaction**: entering a deep sleep
  mode that gates a peripheral clock while a DMA transfer or UART byte is
  still in flight corrupts that transfer. Use the deep-sleep veto hook (or
  your port's equivalent) to hold off sleep during active I/O.
- **Debugging is harder**: many debuggers/JTAG probes lose the target (or
  show it as unresponsive) once the core clock stops in a deep sleep state.
  Keep a build variant with tickless idle disabled for active debugging
  sessions.

## Cheat sheet

| Config / API | Purpose |
|---|---|
| `configUSE_TICKLESS_IDLE` | 0 = classic ticking idle; 1 = sleep when idle time allows; 2 = ESP-IDF variant, more aggressive |
| `configEXPECTED_IDLE_TIME_BEFORE_SLEEP` | Minimum idle ticks before it's worth sleeping at all |
| `portSUPPRESS_TICKS_AND_SLEEP(ticks)` | Port-layer function that actually performs the sleep + tick correction |
| `xTaskGetTickCount()` | Always correct across sleeps — use for all app-level timing |
| `vApplicationSleepPriorToSleep/AfterSleep` | Hooks to gate/restore peripherals around a sleep |
| `esp_pm_configure()` (ESP-IDF) | Higher-level power-management config layered over tickless idle |
| Interrupt latency | Deeper sleep = higher wake latency — bound by your tightest ISR deadline |

## Exercise

1. Enable `configUSE_TICKLESS_IDLE` on a port that supports it (ESP-IDF or
   a Cortex-M port with tickless support) on an otherwise-idle system with
   one task blocked on `vTaskDelay(pdMS_TO_TICKS(2000))`. Measure current
   draw (or, without hardware, log actual wake timestamps) with tickless
   idle on vs. off.
2. Add a hard real-time task that must respond to a GPIO interrupt within
   1 ms. Measure observed interrupt latency with tickless idle enabled at
   your platform's deepest sleep mode, and again at its shallowest. State
   which sleep depth you'd ship for this deadline.
3. Simulate an in-flight UART transaction that must not be interrupted by
   a clock-domain change: add the veto/hook logic and confirm the system
   holds off sleeping until the transaction completes, then sleeps
   immediately after.
