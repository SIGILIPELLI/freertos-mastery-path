# 06 · Software Timers

Not everything deserves a task. "Blink the status LED," "time out this
pairing window," "debounce that button" — giving each of these a dedicated
task (with its own kilobyte of stack) is wasteful. FreeRTOS **software
timers** run little callback functions at a scheduled time — one-shot or
repeating — all sharing a single hidden service task and one stack. This
module covers creating and controlling timers, the rules callbacks must obey,
and how timers relate to the `millis()` patterns you may know from Arduino.

## One-shot vs. auto-reload

Every timer has a period and one of two personalities:

- **Auto-reload** (`pdTRUE`): fires every period — heartbeats, periodic
  polls, blink patterns.
- **One-shot** (`pdFALSE`): fires once, `period` after being started —
  timeouts, "turn the backlight off in 10 s," debounce windows. It can be
  re-armed later with `xTimerStart` or `xTimerReset`.

```cpp
TimerHandle_t heartbeatTimer;   // auto-reload: LED heartbeat
TimerHandle_t backlightTimer;   // one-shot: "backlight off after 5 s"

const uint8_t LED_PIN = 25;
const uint8_t BACKLIGHT_PIN = 26;

void heartbeatCallback(TimerHandle_t t) {
  digitalWrite(LED_PIN, !digitalRead(LED_PIN));       // short & sweet
}

void backlightCallback(TimerHandle_t t) {
  digitalWrite(BACKLIGHT_PIN, LOW);
  Serial.printf("[%lu ms] backlight off\n", millis());
}

void setup() {
  Serial.begin(115200);
  pinMode(LED_PIN, OUTPUT);
  pinMode(BACKLIGHT_PIN, OUTPUT);
  digitalWrite(BACKLIGHT_PIN, HIGH);                  // starts on

  heartbeatTimer = xTimerCreate(
      "heartbeat",              // debug name
      pdMS_TO_TICKS(500),       // period
      pdTRUE,                   // auto-reload
      NULL,                     // timer ID (a void* you can attach)
      heartbeatCallback);       // called each time it fires

  backlightTimer = xTimerCreate(
      "backlight", pdMS_TO_TICKS(5000), pdFALSE, NULL, backlightCallback);

  xTimerStart(heartbeatTimer, 0);   // 2nd arg: how long to wait if the
  xTimerStart(backlightTimer, 0);   // timer command queue is full (ticks)
}

void loop() { vTaskDelay(portMAX_DELAY); }
```

Run it: the LED toggles every 500 ms forever; the backlight goes off exactly
once at t≈5 s. Two timed behaviors, zero extra tasks, zero extra stacks.

### Controlling timers

```cpp
xTimerStart(t, wait);          // start / re-arm (one-shot restarts its period)
xTimerStop(t, wait);           // stop
xTimerReset(t, wait);          // restart the period from NOW ← debounce gold
xTimerChangePeriod(t, pdMS_TO_TICKS(250), wait);   // also starts it
xTimerIsTimerActive(t);        // pdTRUE if running
pvTimerGetTimerID(t);          // read the void* ID (per-timer context)
```

`xTimerReset` is the key to *inactivity timeouts*: every time the user
touches something, reset the one-shot; the callback fires only after a full
quiet period. That's a screensaver in three lines.

## How it works: the timer service task

Software timers are not hardware timers. The kernel runs a hidden **timer
service task** ("Tmr Svc") that sleeps until the soonest timer expires, then
calls your callback *from its own context*. The API functions don't touch
the timer directly — they post commands to the service task's **command
queue** (that's what the second argument to `xTimerStart` is for: how long
to wait if that queue is momentarily full).

Consequences worth knowing:

- **Callbacks run at the service task's priority** (1 on ESP32 Arduino).
  Higher-priority tasks can delay a callback — timers are for human-scale
  timing (LEDs, timeouts), not microsecond-precision control loops. Hardware
  timers or `vTaskDelayUntil` tasks own the precision end.
- **All callbacks share one stack** (the service task's) and run one at a
  time — see the rules below.
- Precision is one tick (1 ms on ESP32 Arduino) at best, plus any
  service-task scheduling delay.

## Callback rules: never block

Your callback borrows the service task. While it runs, **every other timer
in the system waits.** Therefore:

1. **Never call anything that blocks** — no `vTaskDelay`, no
   `xSemaphoreTake`/`xQueueReceive` with a nonzero timeout, no `delay()`.
   A callback that sleeps 100 ms delays *all* timers by 100 ms.
2. **Keep it short.** Toggle a pin, set a flag, stamp a value.
3. **Punt real work to a task.** The idiom: the callback sends to a queue or
   gives a semaphore *with timeout 0*, and a worker task does the heavy
   lifting:

```cpp
void sampleTimerCallback(TimerHandle_t t) {
  uint32_t now = millis();
  xQueueSend(workQueue, &now, 0);       // timeout 0 — never block here
}
// workerTask receives from workQueue and does the slow part
```

## Timers vs. the `millis()` pattern vs. a delay task

Coming from Arduino, you've written this a hundred times:

```cpp
if (millis() - last >= 500) { last = millis(); doThing(); }
```

| Approach | Costs | Best for |
|---|---|---|
| `millis()` checks in a loop | You re-implement scheduling by hand; only runs when that loop runs | Superloop code without an RTOS |
| Dedicated task + `vTaskDelayUntil` | One stack per timed activity | Precise periodic work that *does real processing* |
| Software timer | Shared stack, callback restrictions | Many small timed actions: blinks, timeouts, debounce |

Rule of thumb: **periodic heavy work → task; small timed side-effects (and
especially one-shots) → software timer.** One-shots are where timers shine
brightest — a hand-rolled `millis()` one-shot needs a flag *and* a
timestamp and a reset dance; `xTimerReset` replaces all of it.

## Cheat sheet

| API | Purpose |
|---|---|
| `xTimerCreate(name, period, autoReload, id, cb)` | Create (`pdTRUE`=repeating, `pdFALSE`=one-shot) |
| `xTimerStart(t, wait)` / `xTimerStop(t, wait)` | Arm / disarm |
| `xTimerReset(t, wait)` | Restart period from now — inactivity timeouts |
| `xTimerChangePeriod(t, newPeriod, wait)` | Change period (and start) |
| `pvTimerGetTimerID(t)` / `vTimerSetTimerID(t, p)` | Per-timer context pointer |
| Service task | Priority 1 on ESP32 Arduino; runs ALL callbacks, one at a time |
| Callback rules | Never block; keep short; timeout-0 queue/semaphore to hand off work |
| Precision | ~1 tick + scheduling delays — not for control loops |

## Exercise

Build a "smart night-light" in Wokwi (two LEDs + one button):

1. Auto-reload timer (700 ms): heartbeat LED toggle.
2. Button press (poll it in a low-priority task for now — the *right* way,
   an ISR, is next module) turns the main LED on and calls `xTimerReset` on
   a 4-second one-shot; the callback turns the LED off. Verify that repeated
   presses keep the light alive and it dies exactly 4 s after the *last*
   press.
3. Add a third auto-reload timer at 100 ms whose callback only does
   `xQueueSend(logQueue, &msNow, 0)`; a logger task prints each timestamp.
   Then sabotage it: put `delay(300)` inside the heartbeat callback and
   watch *all three* timers stutter in the log — write one sentence
   explaining why, then remove the sabotage.
