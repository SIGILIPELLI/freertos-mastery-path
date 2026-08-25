# Queue Sets & Advanced Queue Patterns

Level 1 Module 4 covered a task blocking on a single queue. Real systems
often need a task to wait on **several** input sources at once — a command
queue *and* a sensor queue *and* a semaphore — without spinning across them
with short timeouts (which burns CPU and adds latency). FreeRTOS's answer
is the **queue set**: a container that lets one task block on "any of
these queues/semaphores has something" and then tells you which one fired.

## Creating and using a queue set

```cpp
QueueHandle_t cmdQueue, sensorQueue;
SemaphoreHandle_t buttonSem;
QueueSetHandle_t qset;

void setup(void) {
  cmdQueue    = xQueueCreate(4, sizeof(Command));
  sensorQueue = xQueueCreate(8, sizeof(Reading));
  buttonSem   = xSemaphoreCreateBinary();

  // capacity = sum of the lengths of everything you plan to add
  qset = xQueueCreateSet(4 + 8 + 1);

  // every member must be added to the set BEFORE anything is sent to it
  xQueueAddToSet(cmdQueue, qset);
  xQueueAddToSet(sensorQueue, qset);
  xQueueAddToSet(buttonSem, qset);
}

void dispatcherTask(void *pv) {
  for (;;) {
    // blocks until ANY member has data, returns which one
    QueueSetMemberHandle_t activated =
        xQueueSelectFromSet(qset, portMAX_DELAY);

    if (activated == cmdQueue) {
      Command c;
      xQueueReceive(cmdQueue, &c, 0);      // guaranteed non-blocking now
      handleCommand(c);
    } else if (activated == sensorQueue) {
      Reading r;
      xQueueReceive(sensorQueue, &r, 0);
      handleReading(r);
    } else if (activated == buttonSem) {
      xSemaphoreTake(buttonSem, 0);
      handleButtonPress();
    }
  }
}
```

The pattern that trips people up: `xQueueSelectFromSet` only tells you
*which* handle became ready — you still must call the ordinary
`xQueueReceive`/`xSemaphoreTake` on that specific handle to actually
consume the item. Always use a `0` timeout there since the set already told
you data is present.

## The rule that makes this work: single consumer per member

A queue (or semaphore) added to a set can still be written to normally by
any number of producers — that part is unchanged. But **only the queue set
itself should be read from** once a queue is a set member; a second task
directly calling `xQueueReceive` on a queue that's also in a set creates a
race where the set's internal notification and the direct read can
disagree about whether data is still present. Pick one consumer path per
queue: either "read it directly" or "read it via the set that contains it,"
never both.

## Comparing alternatives: why not just poll everything?

```cpp
// BAD: burns CPU, adds latency up to the poll period, doesn't scale
for (;;) {
  if (xQueueReceive(cmdQueue, &c, 0) == pdPASS) { handleCommand(c); }
  if (xQueueReceive(sensorQueue, &r, 0) == pdPASS) { handleReading(r); }
  vTaskDelay(pdMS_TO_TICKS(10));
}
```

Polling with short timeouts trades CPU/power for latency, scales badly
past two or three sources, and gets worse the tighter you make the poll
period. A queue set gives you genuine blocking (zero CPU while idle) across
an arbitrary number of sources with response latency bounded only by
scheduling, not by a poll interval.

The other common alternative — one dedicated task per queue, each doing its
own `xQueueReceive`, sharing state via a mutex — works, but multiplies task
(and stack) count and reintroduces the shared-state synchronization problem
Module 4 (Level 1) queues exist to avoid. Queue sets let a single task
own the dispatch logic with no shared mutable state at all.

## Traps

- **Adding a queue to a set that already has items in it**: `xQueueAddToSet`
  fails (returns `pdFAIL`) if the queue is non-empty at the time you add
  it, because the set's internal accounting can't retroactively account for
  items already there. Always create → add to set → *then* start sending.
- **Removing from a set while items are pending**: `xQueueRemoveFromSet`
  also requires the queue be empty first, for the same reason. Drain (or
  guarantee no further sends, then drain) before removing.
- **Reading a set member directly, bypassing the set**: as above — this
  desyncs the set's notification state from the queue's actual contents,
  and can cause `xQueueSelectFromSet` to return a handle with nothing left
  to receive, or to miss signaling a handle that has data.
- **Sizing the set too small**: the set's capacity argument to
  `xQueueCreateSet` must be at least the sum of the queue lengths (and 1
  per binary/counting semaphore) of everything you intend to add — too
  small and later `xQueueAddToSet` calls fail silently unless checked.
- **Using a queue set as a substitute for priority between sources**: a
  queue set gives you "something is ready," not "the highest-priority
  thing is ready first" — if `cmdQueue` must always be checked before
  `sensorQueue` when both are ready simultaneously, you still need that
  ordering logic in the `if`/`else` chain after `xQueueSelectFromSet`
  returns (it returns exactly one ready member per call, in an
  unspecified but consistent underlying order — don't assume priority
  ordering from it).
- **ISR-safe use**: `xQueueSelectFromSetFromISR` exists but is unusual —
  sets are almost always consumed from task context; sending to a set
  member from an ISR uses the ordinary `...FromISR` send/give calls on the
  member itself.

## Cheat sheet

| API | Purpose |
|---|---|
| `xQueueCreateSet(sumOfLengths)` | Create a set sized for everything you'll add |
| `xQueueAddToSet(queueOrSem, set)` | Register a member — queue/semaphore must be empty |
| `xQueueRemoveFromSet(queueOrSem, set)` | Deregister — member must be empty |
| `xQueueSelectFromSet(set, timeout)` | Block until any member is ready; returns which handle |
| `xQueueSelectFromSetFromISR(set)` | Non-blocking ISR variant |
| Then: `xQueueReceive(handle, &item, 0)` | Actually consume — always `0` timeout after a select |
| Alternative: polling loop | Simpler, wastes CPU, latency bounded by poll period |
| Alternative: one task per source + mutex | No CPU waste, but multiplies tasks and reintroduces shared-state sync |

## Exercise

1. Build the `dispatcherTask` above with all three source types (two
   queues, one semaphore). Confirm sending to any one of them
   independently wakes the dispatcher and routes to the correct handler.
2. Deliberately violate the single-consumer rule: spawn a second task that
   calls `xQueueReceive(sensorQueue, ...)` directly while the dispatcher
   also has `sensorQueue` in a set. Send several items and see the set's
   `xQueueSelectFromSet` miscount or hang — explain why in terms of the
   set's internal notification bookkeeping.
3. Replace the dispatcher with a naive polling loop (as shown in the "bad"
   example) and measure both CPU usage (or, in the POSIX port, wall time
   spent not blocked) and worst-case response latency to a button press at
   three different poll periods. Compare against the queue-set version's
   latency.
