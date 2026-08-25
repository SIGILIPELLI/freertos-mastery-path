# Gatekeeper Tasks & Design Patterns

Level 1 built individual primitives — queues, semaphores, mutexes. This
module is about composing them into recurring, named *design patterns*
that experienced FreeRTOS engineers reach for automatically, starting with
the single most useful one: the **gatekeeper task**, which owns a shared
resource outright so nothing else needs to lock it.

## The gatekeeper pattern

Instead of letting every task that wants to print/log/write-to-flash take
a mutex around the resource, give the resource to exactly one task, and
have everyone else send it *requests* through a queue:

```cpp
QueueHandle_t logQueue;

typedef struct { char msg[64]; } LogMsg;

void loggerGatekeeperTask(void *pv) {
  LogMsg m;
  for (;;) {
    xQueueReceive(logQueue, &m, portMAX_DELAY);
    // this task is the ONLY thing that ever touches the UART/flash/etc.
    uart_write(m.msg, strlen(m.msg));
  }
}

void anyOtherTask(void *pv) {
  LogMsg m;
  snprintf(m.msg, sizeof(m.msg), "event at %lu", (unsigned long)xTaskGetTickCount());
  xQueueSend(logQueue, &m, pdMS_TO_TICKS(50));
}
```

Compare this to a mutex-guarded resource: with a mutex, every caller must
remember to take/give correctly, holds the lock for the duration of the
(possibly slow) I/O operation, and a bug in any one caller can corrupt
shared state or deadlock everyone else. With a gatekeeper, **the mutex
disappears entirely** — mutual exclusion is structural, not a discipline
callers must uphold, because only one task ever touches the resource.
Callers pay a queue-send instead of a lock/unlock, and are never blocked
waiting for the (possibly slow) I/O itself — they hand off and move on.

## When a gatekeeper beats a mutex

Use a gatekeeper when: the operation is slow or variable-latency (UART,
flash, network) and callers shouldn't block on it; the resource needs
serialized *and* possibly reordered/prioritized access (a gatekeeper can
inspect the queue and reorder, rate-limit, or coalesce requests — a mutex
can't); or you want a single choke point to add cross-cutting behavior
(timestamps, filtering by log level, rate-limiting) without touching every
caller.

Keep a plain mutex when the critical section is short, fixed-latency, and
purely about correctness of a shared data structure (Level 1 Module 5's
struct-guarded-by-mutex case) — spinning up a whole task and queue for that
is unnecessary overhead.

## Producer–filter–consumer chains

A closely related pattern: instead of one gatekeeper, chain several tasks,
each doing one transformation stage, connected by queues:

```cpp
QueueHandle_t rawQueue, filteredQueue;

void acquireTask(void *pv) {           // stage 1: fast, dumb sampling
  for (;;) {
    int raw = adc_read();
    xQueueSend(rawQueue, &raw, 0);
    vTaskDelay(pdMS_TO_TICKS(10));
  }
}

void filterTask(void *pv) {            // stage 2: moving-average filter
  static int history[8]; static int idx = 0;
  int raw;
  for (;;) {
    xQueueReceive(rawQueue, &raw, portMAX_DELAY);
    history[idx++ % 8] = raw;
    int sum = 0; for (int i = 0; i < 8; i++) sum += history[i];
    int avg = sum / 8;
    xQueueSend(filteredQueue, &avg, 0);
  }
}

void reportTask(void *pv) {            // stage 3: gatekeeper for the UART
  int avg;
  for (;;) {
    xQueueReceive(filteredQueue, &avg, portMAX_DELAY);
    printf("filtered: %d\n", avg);
  }
}
```

Each stage has one job, one input, one output — easy to test in isolation
(feed `filterTask` a known sequence via `rawQueue` and check `filteredQueue`'s
output) and easy to reprioritize independently (give `acquireTask` a higher
priority than `filterTask` if sampling jitter matters more than filtering
latency).

## The relay/fan-in pattern

Module 1's stream-buffer single-writer constraint comes up constantly:
when multiple producers need to feed one single-consumer resource (a
stream buffer, or any resource that isn't internally thread-safe for
multiple writers), insert a small relay task that owns the resource and
fans in via an ordinary (multi-writer-safe) queue:

```cpp
QueueHandle_t fanInQueue;          // safe for many producers
StreamBufferHandle_t sharedStream; // single-writer only

void relayTask(void *pv) {
  uint8_t chunk[32]; size_t len;
  for (;;) {
    // receiving struct carries {chunk, len} — details omitted for brevity
    // ... xQueueReceive(fanInQueue, &item, portMAX_DELAY);
    xStreamBufferSend(sharedStream, chunk, len, portMAX_DELAY);
  }
}
```

This is the same shape as the gatekeeper — one task, exclusive ownership —
applied specifically to satisfy an API's single-writer contract rather
than to serialize a slow I/O operation.

## Traps

- **Gatekeeper becomes the bottleneck**: if the gatekeeper's own processing
  (not just the final I/O) is slow, its input queue backs up and every
  caller's `xQueueSend` starts blocking or failing. Keep the gatekeeper's
  per-message work minimal, or give it enough queue depth to absorb
  bursts, and always check `xQueueSend`'s return value.
- **Priority mismatch**: a gatekeeper running at a lower priority than its
  callers can be preempted indefinitely, letting its queue fill even
  though "in principle" it's ready to drain it — set the gatekeeper's
  priority based on how quickly its queue must drain, not on some default.
- **Losing backpressure information**: routing everything through a
  gatekeeper queue can hide *which* caller is overwhelming the system,
  since the queue mixes all producers — tag messages with a source ID if
  you need to diagnose an overload later.
- **Chains that grow unbounded latency**: each stage in a producer-filter-consumer
  chain adds at minimum one context switch and one queue traversal of
  latency; a chain five stages deep processing at 1 kHz can add
  meaningfully to end-to-end latency even though each stage individually
  looks cheap. Measure end-to-end, not stage-by-stage.
- **Forgetting the relay task still needs its own queue sized for burst
  traffic**: the relay/fan-in pattern moves the multi-writer problem to an
  ordinary queue, but that queue can still overflow under a burst from
  several producers at once — size it for the real worst case, not the
  steady-state average.

## Cheat sheet

| Pattern | Use when | Cost |
|---|---|---|
| Mutex | Short, fixed-latency critical section on shared data | Priority inheritance handles inversion; no extra task |
| Gatekeeper task | Slow/variable-latency resource; want reordering/filtering/rate-limiting | Extra task + queue; callers never block on the I/O itself |
| Producer–filter–consumer chain | Multi-stage processing pipeline | Extra latency per stage; independent reprioritization/testing |
| Relay / fan-in task | Multiple producers, single-writer-only API (stream buffer, some drivers) | Extra task; moves multi-writer problem to an ordinary (safe) queue |

## Exercise

1. Build the `loggerGatekeeperTask` pattern with three producer tasks at
   different priorities all logging concurrently. Confirm output is never
   interleaved/corrupted, unlike a naive shared `printf` without any
   serialization.
2. Build the three-stage acquire/filter/report pipeline. Feed `acquireTask`
   a synthetic ramp signal and verify `filterTask`'s moving average tracks
   it with the expected lag.
3. Take Module 1's stream-buffer example (single-producer only) and add a
   second producer task. Confirm corruption occurs, then fix it with a
   relay task fed by a queue, and confirm the corruption disappears under
   the same concurrent load.
