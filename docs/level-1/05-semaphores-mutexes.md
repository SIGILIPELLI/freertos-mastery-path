# 05 · Semaphores & Mutexes

Queues move *data*; semaphores move *signals* — "something happened," "a
resource is free." Their cousin the mutex protects *shared things* — a
serial port, an I2C bus, a config struct — from being used by two tasks at
once. The two look similar (both are take/give objects) but solve opposite
problems, and mixing them up causes real bugs: this module draws the lines
sharply, demonstrates priority inheritance, and gives you your first
deadlock — on purpose.

## Binary semaphores: signaling that something happened

A **binary semaphore** holds either 0 or 1. One task (or ISR — next module)
**gives** it; another task **takes** it, blocking until the give happens.
It's the RTOS way to say *event* — with no data attached, just "go."

```cpp
SemaphoreHandle_t startSignal;

void workerTask(void *pv) {
  for (;;) {
    // Sleeps (0% CPU) until someone gives the semaphore:
    if (xSemaphoreTake(startSignal, portMAX_DELAY) == pdTRUE) {
      Serial.printf("[%lu ms] worker: triggered!\n", millis());
    }
  }
}

void triggerTask(void *pv) {
  for (;;) {
    vTaskDelay(pdMS_TO_TICKS(1500));
    Serial.printf("[%lu ms] trigger: firing worker\n", millis());
    xSemaphoreGive(startSignal);
  }
}

void setup() {
  Serial.begin(115200);
  startSignal = xSemaphoreCreateBinary();     // starts EMPTY (must be given first)
  xTaskCreate(workerTask,  "worker",  2048, NULL, 2, NULL);
  xTaskCreate(triggerTask, "trigger", 2048, NULL, 1, NULL);
}

void loop() { vTaskDelay(portMAX_DELAY); }
```

Because a binary semaphore can only hold one "give," **events don't
accumulate**: two gives before a take still wake the taker once. If every
event must be counted, use a counting semaphore or a queue.

## Counting semaphores: tracking a countable resource

A **counting semaphore** holds 0..N. Classic uses: a pool of N identical
resources (take = claim one, give = return one), or counting events that
mustn't be lost:

```cpp
// 3 identical "slots" (e.g. buffers); 4 tasks compete for them
SemaphoreHandle_t slots = xSemaphoreCreateCounting(3, 3);  // max 3, start 3

void userTask(void *pv) {
  int id = (int)(intptr_t)pv;
  for (;;) {
    xSemaphoreTake(slots, portMAX_DELAY);          // claim a slot (or wait)
    Serial.printf("task %d: got a slot (%u left)\n",
                  id, uxSemaphoreGetCount(slots));
    vTaskDelay(pdMS_TO_TICKS(500 + 300 * id));     // "use" it
    xSemaphoreGive(slots);                         // return it
    vTaskDelay(pdMS_TO_TICKS(200));
  }
}
```

At any moment at most three tasks are inside the take/give window — the
fourth waits, automatically, with no polling.

## Mutexes: protecting shared state

A **mutex** (mutual exclusion) guards a resource so only one task uses it at
a time. The take/give calls are the same, but the *meaning* is different —
and so is the machinery:

```cpp
struct Config { float threshold; uint32_t periodMs; };
Config config = {2.5f, 1000};
SemaphoreHandle_t configMutex;

void writerTask(void *pv) {
  for (;;) {
    vTaskDelay(pdMS_TO_TICKS(3000));
    xSemaphoreTake(configMutex, portMAX_DELAY);   // ---- critical section ----
    config.threshold += 0.1f;                     // both fields updated
    config.periodMs  += 50;                       // together, atomically
    xSemaphoreGive(configMutex);                  // --------------------------
  }
}

void readerTask(void *pv) {
  for (;;) {
    vTaskDelay(pdMS_TO_TICKS(700));
    xSemaphoreTake(configMutex, portMAX_DELAY);
    Config snapshot = config;                     // consistent copy
    xSemaphoreGive(configMutex);
    Serial.printf("threshold=%.1f period=%lu\n",
                  snapshot.threshold, snapshot.periodMs);
  }
}

void setup() {
  Serial.begin(115200);
  configMutex = xSemaphoreCreateMutex();          // starts AVAILABLE
  xTaskCreate(writerTask, "writer", 2048, NULL, 1, NULL);
  xTaskCreate(readerTask, "reader", 2048, NULL, 2, NULL);
}

void loop() { vTaskDelay(portMAX_DELAY); }
```

Without the mutex, the reader could be preempted between the writer's two
field updates and print a `threshold` from the new config with a `periodMs`
from the old — a *torn read* that no `volatile` can prevent. Mutex
discipline: **take before touching, give immediately after, keep the
critical section tiny** (copy out, then work on the copy — as `readerTask`
does). Never block (`vTaskDelay`, queue waits) while holding a mutex, and
only the task that took a mutex may give it.

### Priority inheritance — why a mutex isn't just a binary semaphore

The famous failure mode: **priority inversion**. Low-priority task L holds
the lock; high-priority task H blocks waiting for it; medium-priority task M
(unrelated) preempts L — so M, the *least* urgent runnable task, is
effectively blocking H, the *most* urgent, indefinitely. This bug famously
rebooted the Mars Pathfinder lander.

Mutexes fix it with **priority inheritance**: while H waits for the mutex, L
is temporarily boosted to H's priority, so M cannot preempt it; L finishes
the critical section quickly, releases, and drops back down. Binary
semaphores do **not** do this — which is exactly why the rule is:

> **Mutex for protecting a resource. Semaphore for signaling an event.
> Never swap them.**

(Mutexes also must not be used from ISRs, and shouldn't be used for
signaling — the inheritance bookkeeping assumes lock/unlock by the same
task.)

## Deadlock, in four lines

Two locks, taken in opposite orders by two tasks:

```cpp
// Task A:                         // Task B:
xSemaphoreTake(mutex1, ...);       xSemaphoreTake(mutex2, ...);
xSemaphoreTake(mutex2, ...);  ⛔   xSemaphoreTake(mutex1, ...);  ⛔
```

A holds 1 and wants 2; B holds 2 and wants 1. Both wait forever. Defenses,
in order of preference:

1. **Hold one lock at a time** (restructure so you never nest).
2. **Global lock ordering** — if you must nest, every task takes locks in
   the same documented order (always `mutex1` before `mutex2`).
3. **Timeouts** — take with `pdMS_TO_TICKS(100)` instead of forever, and
   back off (release everything, retry) on failure; at minimum, log it — a
   timeout that fires is a design bug telling you where.

## When a queue beats a semaphore

If the "event" carries *any* data — which reading, which button, how much —
a semaphore forces you to smuggle the data through a shared global next to
the signal, reinventing the race you were avoiding. **A queue is a semaphore
plus payload plus buffering**; use it whenever the consumer needs to know
more than "it happened." Likewise "N events pending" with data per event is
just a queue of depth N. Semaphores win only for pure, data-free signaling
(especially ISR→task wake-ups) and resource counting — and Module 7 shows an
even cheaper tool (task notifications) for the simplest of those cases.

## Cheat sheet

| API / concept | Purpose |
|---|---|
| `xSemaphoreCreateBinary()` | Event flag; starts empty; gives don't accumulate |
| `xSemaphoreCreateCounting(max, init)` | Resource pool / event counter 0..max |
| `xSemaphoreCreateMutex()` | Lock with **priority inheritance**; starts available |
| `xSemaphoreTake(s, timeout)` | Wait/claim — `pdTRUE` on success |
| `xSemaphoreGive(s)` | Signal/release (mutex: same task that took it) |
| `uxSemaphoreGetCount(s)` | Current count |
| Priority inversion | Low holds lock high needs; medium preempts low → high starves |
| Priority inheritance | Mutex boosts the holder to the waiter's priority — mutexes only |
| Deadlock defenses | Don't nest → fixed lock order → timeouts + backoff |
| Queue vs semaphore | Event carries data? Queue. Pure signal / resource count? Semaphore. Shared state? Mutex. |

## Exercise

1. Build the priority-inversion demo: `lowTask` (prio 1) takes a **binary
   semaphore** and busy-works 500 ms before giving it; `highTask` (prio 3)
   wakes every 200 ms and takes the same semaphore; `mediumTask` (prio 2)
   busy-works in 50 ms bursts. Log timestamps and measure how long `high`
   waits.
2. Replace the binary semaphore with a mutex and measure again — explain
   the improvement in terms of priority inheritance.
3. Construct the two-mutex deadlock on purpose (add logging before each
   take). Confirm both tasks freeze while the rest of the system keeps
   running. Fix it once with lock ordering, then again with 100 ms timeouts
   plus retry, and state which fix you'd ship and why.
