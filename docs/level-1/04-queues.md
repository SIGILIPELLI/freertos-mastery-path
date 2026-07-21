# 04 · Queues

Tasks are only half the story — the other half is getting data between them
without corruption. Sharing globals between preempting (or truly parallel,
on ESP32) tasks is a race condition waiting to happen. The **queue** is
FreeRTOS's primary answer: a thread-safe FIFO that *copies* items in and
out, blocks consumers until data arrives, and blocks producers when full.
This module builds the classic producer/consumer pipeline and covers the
struct-vs-pointer decision that trips up every new RTOS programmer.

## Why not just use a global?

```cpp
volatile int latestReading;    // written by task A, read by task B — BAD
```

A preemption (or the other core) can strike *between* the read and write of
a multi-word value, or between "check flag" and "clear flag," yielding torn
values and lost updates. `volatile` prevents compiler caching — it does
**nothing** about atomicity or synchronization. Queues solve both problems
at once: the copy happens inside a critical section, and the blocking
behavior replaces flag-polling entirely.

## Creating, sending, receiving

```cpp
QueueHandle_t q = xQueueCreate(8, sizeof(int32_t));
//                             │        └ size of ONE item (bytes)
//                             └ capacity: 8 items

int32_t v = 42;
xQueueSend(q, &v, pdMS_TO_TICKS(100));      // copy v IN  (waits ≤100 ms if full)
int32_t out;
xQueueReceive(q, &out, portMAX_DELAY);      // copy OUT   (waits forever if empty)
```

Key semantics:

- **Items are copied**, both on send and receive. After `xQueueSend`
  returns, the queue has its own copy — you can reuse or destroy `v`.
- **Receive blocks** until an item arrives or the timeout expires — the
  consumer task uses zero CPU while waiting. This is the RTOS-idiomatic
  replacement for "poll a flag in a loop."
- **Send blocks** (up to its timeout) if the queue is full — automatic
  back-pressure on a producer that outruns its consumer.
- Both return `pdPASS` on success or `errQUEUE_FULL` / `pdFALSE` on timeout
  — **check the result** and decide: drop the sample? log it? increase the
  queue length?
- Timeouts: `0` = try without blocking; `portMAX_DELAY` = wait forever;
  anything else via `pdMS_TO_TICKS`.

## Producer/consumer: the fundamental RTOS pattern

A fast, simple producer hands work to a slower, smarter consumer — decoupled
by the queue's buffering:

```cpp
struct Reading {
  uint32_t ms;        // when it was taken
  int      raw;       // ADC value
};

QueueHandle_t readingQueue;

void producerTask(void *pv) {                    // fast, dumb, regular
  TickType_t lastWake = xTaskGetTickCount();
  for (;;) {
    vTaskDelayUntil(&lastWake, pdMS_TO_TICKS(200));   // exactly 5 Hz
    Reading r = { millis(), analogRead(34) };
    if (xQueueSend(readingQueue, &r, 0) != pdPASS) {
      Serial.println("queue full — dropped a reading");
    }
  }
}

void consumerTask(void *pv) {                    // slow, smart, event-driven
  Reading r;
  for (;;) {
    xQueueReceive(readingQueue, &r, portMAX_DELAY);   // sleeps until data
    float volts = r.raw * 3.3f / 4095.0f;
    Serial.printf("[%lu ms] %.2f V\n", r.ms, volts);
    vTaskDelay(pdMS_TO_TICKS(350));         // simulate slow processing
  }
}

void setup() {
  Serial.begin(115200);
  readingQueue = xQueueCreate(8, sizeof(Reading));    // create BEFORE tasks
  if (readingQueue == NULL) { Serial.println("queue alloc failed"); for(;;); }
  xTaskCreate(producerTask, "prod", 2048, NULL, 2, NULL);
  xTaskCreate(consumerTask, "cons", 4096, NULL, 1, NULL);
}

void loop() { vTaskDelay(portMAX_DELAY); }
```

Run it in Wokwi: the producer stays perfectly periodic (its deadline is
protected by its higher priority), the consumer chews through the backlog at
its own pace, and when the 8-slot buffer overflows, you *see* the drops
instead of silently corrupting data. Multiple producers can share one queue
safely — that's the standard many-to-one logging pattern.

Note the order in `setup()`: **create the queue before the tasks that use
it** — a task might run the moment it's created and would dereference a
NULL handle.

## Structs vs. pointers — and the ownership pitfall

Queues copy items, so item size drives a design decision:

- **Small items (a few dozen bytes): queue the struct itself.** Copying is
  cheap and there is *no ownership question* — each side has its own copy.
  Default choice; used above.
- **Large items (a camera frame, a log line buffer): queue a pointer.**
  `xQueueCreate(n, sizeof(uint8_t*))` copies only the 4-byte pointer. Fast —
  but now two tasks can see the same memory, and you've signed an ownership
  contract:

!!! warning "The pointer-queue ownership rule"
    Once a pointer is sent, the **receiver owns the memory**. The sender
    must not write to it, reuse it, or free it. The receiver must free it
    (or return it to a pool) when done. Break this rule and you get the
    classic heisenbug: the sender reuses the buffer while the consumer is
    still reading it, corrupting data only under load.

```cpp
// Pointer-queue sketch (ownership transfers with the pointer)
QueueHandle_t lineQueue;                     // holds char*

void senderTask(void *pv) {
  for (;;) {
    char *line = (char *)pvPortMalloc(64);           // allocate
    snprintf(line, 64, "reading=%d at %lu", analogRead(34), millis());
    if (xQueueSend(lineQueue, &line, 0) != pdPASS) {
      vPortFree(line);                               // NOT sent → still ours
    }                                                // sent → never touch again
    vTaskDelay(pdMS_TO_TICKS(500));
  }
}

void printerTask(void *pv) {
  char *line;
  for (;;) {
    xQueueReceive(lineQueue, &line, portMAX_DELAY);  // we own it now
    Serial.println(line);
    vPortFree(line);                                 // receiver frees
  }
}
```

## Useful variations

- `xQueuePeek(q, &item, timeout)` — read the front item *without* removing
  it.
- `xQueueOverwrite(q, &item)` — for **length-1 queues** only: always
  succeeds, replacing the old value. Perfect "mailbox" for
  *latest-value-wins* data (current temperature, latest setpoint) where
  history doesn't matter.
- `uxQueueMessagesWaiting(q)` — how many items are queued (good for health
  monitoring: a steadily growing queue means your consumer can't keep up).
- `xQueueReset(q)` — empty it.
- **Queue sets** (a glimpse): `xQueueCreateSet` lets one task block on
  *several* queues/semaphores at once and learn which one fired — like
  `select()` for RTOS objects. When you find yourself polling two queues
  with timeout 0 in a loop, a queue set (Level 2) is the clean answer.

## Cheat sheet

| API | Purpose |
|---|---|
| `xQueueCreate(len, itemSize)` | Create a queue (returns NULL if out of memory) |
| `xQueueSend(q, &item, timeout)` | Copy item in; blocks while full; `pdPASS`/`errQUEUE_FULL` |
| `xQueueReceive(q, &item, timeout)` | Copy item out; blocks while empty |
| `xQueuePeek(q, &item, timeout)` | Read front item without removing |
| `xQueueOverwrite(q, &item)` | Length-1 "mailbox": latest value always wins |
| `uxQueueMessagesWaiting(q)` | Items currently queued |
| Timeout values | `0` = no wait · `pdMS_TO_TICKS(ms)` · `portMAX_DELAY` = forever |
| Struct vs pointer | Small data: queue the struct (no ownership issues). Big data: queue a pointer + strict ownership transfer |

## Exercise

Build a two-producer, one-consumer pipeline:

1. `tempTask` sends a `Reading {source=1, value}` every 400 ms; `lightTask`
   sends `{source=2, value}` every 250 ms — both into the *same* 5-slot
   queue.
2. `displayTask` (lower priority) receives with `portMAX_DELAY` and prints
   which sensor each reading came from.
3. Make `displayTask` artificially slow (`vTaskDelay(300)` per item) and
   watch send failures appear; fix the drops *without* enlarging the queue
   (hint: which producer data is latest-value-wins? Use a second, length-1
   queue with `xQueueOverwrite` for it).
4. Explain in a comment why `volatile int latestTemp` would not have been a
   safe design even on a single core.
