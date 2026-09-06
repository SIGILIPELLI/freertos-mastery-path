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

## How It Actually Works

Under the hood, every task is a **Task Control Block (TCB)** — a C struct
holding the task's name, priority, current state, stack pointer, and a
`pxTopOfStack` value that points into that task's own private stack region.
When the scheduler decides task A must stop and task B must run, it does not
"transfer control" in any abstract sense — it performs a **context switch**:
the current CPU registers (program counter, general-purpose registers, stack
pointer) are pushed onto A's stack, A's TCB is updated to record where its
stack pointer now sits, then B's TCB is read back to restore B's saved
registers from B's stack, and execution resumes at exactly the instruction B
was at when it was last preempted. This is why each task needs its own
stack — the switch is really just "save state to this stack, load state from
that stack."

The tick interrupt is what makes this happen without cooperation. On every
tick, the interrupt handler decrements the delay counters of any Blocked
tasks; when a counter reaches zero, that task's TCB moves from a **delayed
list** to the **Ready list** for its priority. The handler then asks the
scheduler to re-evaluate: is the highest-priority Ready task the one
currently running? If not, it sets a flag requesting a context switch, which
happens either immediately (on return from the ISR) or, on the ESP32's dual
FreeRTOS ports, via a similar mechanism per core. FreeRTOS keeps a small
array of Ready lists indexed by priority (`pxReadyTasksLists[priority]`), so
finding "the highest-priority Ready task" is just scanning from the top of
that array down to the first non-empty list — O(1) in practice via a
bitmap of occupied priorities on most ports.

Queues and semaphores extend this same TCB machinery with a **blocked-on
this object** list. When a task calls `xQueueReceive()` on an empty queue,
its TCB is removed from the Ready list and appended to that queue's own
list of waiting tasks (ordered by priority), and the scheduler immediately
runs something else — the blocked task consumes zero CPU while it waits.
When another task or ISR calls `xQueueSend()`, the kernel checks that
waiting list: if a task is there, its TCB is moved straight back to Ready
(and, if it outranks the currently running task, a context switch is
requested on the spot) — the data reaches it without ever passing through
the general Ready-list scan.

This blocked-list design is also the root of **priority inversion** and its
fix. If a low-priority task holds a mutex a high-priority task needs, the
high-priority task's TCB sits on that mutex's blocked list — invisible to
the scheduler as "urgent" until the mutex is released. FreeRTOS's mutex
implementation (unlike a plain binary semaphore) checks this case: when a
higher-priority task blocks on a mutex, the kernel temporarily raises the
*holder's* priority to match, via **priority inheritance**, so the holder
gets scheduled, finishes, and releases the mutex sooner — after which its
priority drops back to normal. This is purely bookkeeping in the TCB's
priority field plus a re-sort of Ready-list membership; no separate
"inheritance task" exists.

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
