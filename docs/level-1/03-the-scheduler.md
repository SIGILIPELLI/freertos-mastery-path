# 03 · The Scheduler

The scheduler is the heart of the kernel: the piece of code that decides, at
every moment, which task owns the CPU. Its rule is short enough to memorize —
**the highest-priority Ready task runs, always** — but the consequences
(preemption, time slicing, starvation, the idle task) are where real
firmware behavior is decided. This module makes the rule concrete, shows what
happens between equal-priority tasks, and covers the ESP32's dual-core twist.

## The one rule, and preemption

FreeRTOS priorities run from **0 (lowest)** to `configMAX_PRIORITIES - 1`
(24 on ESP32 Arduino). At every scheduling decision, the kernel runs the
highest-priority task that is **Ready**. If a higher-priority task *becomes*
Ready — its delay expires, its queue receives data, an ISR signals it — the
running task is **preempted immediately**, mid-statement, and the
higher-priority task runs. No politeness, no waiting for a yield.

Watch it happen:

```cpp
void lowTask(void *pv) {
  for (;;) {
    Serial.println("low: starting a long computation...");
    uint32_t t0 = millis();
    while (millis() - t0 < 300) { }        // 300 ms of BUSY work (no blocking)
    Serial.println("low: ...finished");
    vTaskDelay(pdMS_TO_TICKS(500));
  }
}

void highTask(void *pv) {
  for (;;) {
    vTaskDelay(pdMS_TO_TICKS(100));        // wakes every 100 ms
    Serial.printf("        HIGH ran at %lu ms\n", millis());
  }
}

void setup() {
  Serial.begin(115200);
  xTaskCreate(lowTask,  "low",  2048, NULL, 1, NULL);
  xTaskCreate(highTask, "high", 2048, NULL, 3, NULL);   // higher priority
}

void loop() { vTaskDelay(portMAX_DELAY); }
```

The output shows `HIGH ran at ...` lines appearing every 100 ms *right
through the middle* of low's 300 ms computation. The low task never
volunteered — the tick interrupt noticed high's delay had expired, and the
scheduler swapped tasks in a few microseconds. That context switch —
saving low's registers, restoring high's — is the mechanism behind
everything an RTOS promises.

### Assigning priorities sensibly

Priorities encode **urgency (deadline tightness), not importance**:

| Priority | Typical residents |
|---|---|
| High (5+) | Tight-deadline work triggered by ISRs; motor/control loops |
| Medium (2-4) | Sensor sampling, protocol handling, application logic |
| Low (1) | Logging, display refresh, `loop()` (Arduino puts it at 1) |
| 0 | Idle task lives here — avoid putting your tasks at 0 |

Rule of thumb (rate-monotonic): **the shorter a task's period/deadline, the
higher its priority.** And keep high-priority tasks *short* — they run at the
expense of everyone below.

## The tick, and time slicing between equals

A hardware timer fires the **tick interrupt** at `configTICK_RATE_HZ` —
**1000 Hz on ESP32 Arduino**, so one tick = 1 ms (this is why
`pdMS_TO_TICKS` exists: on a 100 Hz build, 1 tick = 10 ms and delays are
rounded accordingly). The tick wakes expired delays and drives scheduling
decisions.

When several tasks share the *same* priority and are all Ready, FreeRTOS
**time-slices** them: each tick, the scheduler rotates to the next
equal-priority Ready task (round-robin). They appear to run "in parallel,"
interleaved at 1 ms granularity. Tasks can also hand over early:
`taskYIELD()` says "I'm done for now — if another task of my priority is
Ready, switch to it" (it never gives the CPU to *lower* priority tasks).

## The idle task, and starvation

When *no* application task is Ready — everyone's Blocked — the scheduler
runs the **idle task**, a priority-0 task the kernel creates automatically.
It cleans up deleted tasks' memory and, when configured, puts the CPU into a
low-power state. The ESP32 also uses idle time to feed a watchdog.

**Starvation** is the classic scheduler bug: a task that never blocks starves
everything below it — forever, not just "slows it down."

```cpp
void greedy(void *pv) {
  for (;;) {
    // compute, compute, compute — never blocks
  }
}
```

At priority 2, `greedy` permanently freezes every priority-1 and 0 task: the
`loop()` task, your logger, and the idle task. On ESP32 the symptom is
dramatic — the **idle task watchdog** fires and prints
`Task watchdog got triggered` because idle never got to feed it. The fix is
never `taskYIELD()` (that only helps *equal* priorities) — the fix is making
the task **block**: on a delay, a queue, or a notification. In a preemptive
RTOS, blocking is not lost time; it is how you *donate* time downward.

## ESP32: two cores

The ESP32 has two cores (0 and 1), and its FreeRTOS schedules across both —
the rule becomes "on each core, run the highest-priority Ready task allowed
to run there." Arduino's `setup()`/`loop()` run on **core 1**; WiFi and
Bluetooth stacks run mostly on **core 0**. `xTaskCreate` lets the task run
on either core; to control placement use:

```cpp
xTaskCreatePinnedToCore(
    taskFn, "ctrl", 2048, NULL, 5, NULL,
    1                 // core id: 0, 1, or tskNO_AFFINITY (either core)
);
```

Pinning matters when you need cache locality, when a library demands a
specific core, or when you want your control loop unaffected by WiFi bursts
(pin it to core 1). It also means **two tasks really can run
simultaneously** — a mental-model upgrade that makes the data-sharing tools
of the next two modules non-optional, not just polite. Everything else in
this level works identically with or without pinning; Level 2's ESP-IDF
module digs into the dual-core details.

## Cheat sheet

| Concept / API | Meaning |
|---|---|
| Scheduling rule | Highest-priority **Ready** task runs — immediately, via preemption |
| Priorities | 0 = lowest (idle); ESP32 Arduino: up to 24; `loop()` runs at 1 |
| Priority heuristic | Shorter deadline → higher priority; keep high-prio tasks short |
| Tick | 1000 Hz on ESP32 Arduino (1 tick = 1 ms); drives delays & time slicing |
| Time slicing | Equal-priority Ready tasks round-robin each tick |
| `taskYIELD()` | Offer CPU to *equal*-priority tasks now (never to lower) |
| Idle task | Priority-0 kernel task; runs when nothing else is Ready; feeds ESP32 idle watchdog |
| Starvation | A never-blocking task freezes all lower priorities — fix by blocking, not yielding |
| `xTaskCreatePinnedToCore(..., core)` | ESP32: pin a task to core 0 or 1 (`tskNO_AFFINITY` = either) |

## Exercise

1. Run the preemption demo above and confirm `HIGH` lines appear during
   low's busy-wait. Then set both tasks to priority 1 — explain what you now
   see (hint: with a 1000 Hz tick, does time slicing let `high` still run
   during the busy loop? Why is it now late?).
2. Create the `greedy` task at priority 2 alongside a priority-1 blinker.
   Observe the blinker freeze and the watchdog complaint in the serial
   monitor. Fix it with a single `vTaskDelay(1)` and confirm recovery.
3. Pin two busy-ish tasks (each printing its core via `xPortGetCoreID()`)
   to core 0 and core 1 respectively, then to the *same* core, and compare
   the interleaving of their output.
