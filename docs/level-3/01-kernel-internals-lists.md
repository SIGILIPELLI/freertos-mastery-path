# Kernel Internals — Lists & Ready Lists

Every module so far has used the kernel through its public API — `xTaskCreate`,
`xQueueSend`, `vTaskDelay` — treating the scheduler as a black box that "just
works." This module opens the box. Underneath every task, queue, and timer,
FreeRTOS uses exactly one data structure: the doubly linked list defined in
`list.c`/`list.h`. Understanding it is the key that unlocks everything else in
this level — context switching, SMP scheduling, and tracing all reduce to
"which list is this TCB on, and why."

## `List_t` and `ListItem_t`

```c
struct xLIST_ITEM {
    TickType_t xItemValue;             // sort key (usually a tick count or priority)
    struct xLIST_ITEM *pxNext;
    struct xLIST_ITEM *pxPrevious;
    void *pvOwner;                     // back-pointer to the owning TCB
    void *pvContainer;                 // back-pointer to the List_t it's on
};

typedef struct xLIST {
    UBaseType_t uxNumberOfItems;
    ListItem_t *pxIndex;                // roving marker, used for round-robin
    MiniListItem_t xListEnd;            // sentinel node, always present
} List_t;
```

Every list is circular and always contains at least the sentinel
`xListEnd`, which has the maximum possible `xItemValue`
(`portMAX_DELAY`) so it always sorts last. Insertion
(`vListInsert`) walks the list from the front and inserts in ascending
`xItemValue` order — this is how the delayed-task list stays sorted by wake
time without any separate sort step. `vListInsertEnd` skips the search and
inserts immediately before the sentinel, which is what the *ready* lists use,
since ready-to-run tasks don't need ordering by value, only round-robin
rotation via `pxIndex`.

Each `TCB_t` embeds two `ListItem_t` members directly (not pointers to
separately allocated nodes): `xStateListItem` (which list — ready, blocked,
suspended — the task is currently on) and `xEventListItem` (used when the
task is blocked on a queue/semaphore/event group's wait list, sorted by
priority so the *highest-priority* waiter is woken first on a give).

## The ready lists: `pxReadyTasksLists[]`

```c
PRIVILEGED_DATA static List_t pxReadyTasksLists[configMAX_PRIORITIES];
```

One `List_t` per priority level, index 0 = lowest priority (idle).
`prvAddTaskToReadyList` inserts a task's `xStateListItem` at the end of
`pxReadyTasksLists[uxPriority]`. Finding the next task to run is then not
a search over all tasks — it's:

```c
uxTopReadyPriority = <highest index with a non-empty ready list>;
pxCurrentTCB = listGET_OWNER_OF_NEXT_ENTRY(&pxReadyTasksLists[uxTopReadyPriority]);
```

`listGET_OWNER_OF_NEXT_ENTRY` advances `pxIndex` to the next item in that
list and returns its owner — this is exactly the round-robin rotation between
equal-priority tasks. On architectures with `configUSE_PORT_OPTIMISED_TASK_SELECTION`
(most Cortex-M ports), finding the top non-empty priority isn't a linear scan
either — the kernel maintains `uxTopReadyPriority` as a 32-bit bitmap (one bit
per priority level) and uses a hardware count-leading-zeros instruction
(`__CLZ` / `configISR_STACK_SIZE`-adjacent portable macro
`portGET_HIGHEST_PRIORITY`) to find the highest set bit in O(1), not O(n).
This is why FreeRTOS scales cleanly to `configMAX_PRIORITIES` in the dozens:
the scheduling decision is a handful of instructions regardless of how many
priority levels or tasks exist.

## The delayed-task lists

```c
static List_t xDelayedTaskList1;
static List_t xDelayedTaskList2;
static List_t * volatile pxDelayedTaskList;
static List_t * volatile pxOverflowDelayedTaskList;
```

`vTaskDelay`/`vTaskDelayUntil` compute a wake tick and insert the task's
`xStateListItem` into `pxDelayedTaskList`, sorted by wake tick (ascending,
via `vListInsert`). The list's *head* is therefore always the next task due
to wake — `xTaskIncrementTick` only has to look at the front of the list each
tick, not scan every delayed task. Two lists exist, not one, to handle tick
counter overflow: when the current tick count wraps past `0xFFFFFFFF`, the
two lists swap roles (`pxDelayedTaskList` and `pxOverflowDelayedTaskList`
trade places) so that a task whose wake time was computed pre-overflow
doesn't get compared against a post-overflow tick count using ordinary
unsigned arithmetic. This "list swap on overflow" is a specific, deliberate
kernel design decision worth remembering — it's the answer to "what happens
to `vTaskDelay` after 49.7 days at a 1kHz tick" (`0xFFFFFFFF` ticks) on a
32-bit tick counter.

## Following one task's journey through the lists

1. `xTaskCreate` → task's `xStateListItem` inserted into
   `pxReadyTasksLists[priority]` via `vListInsertEnd`.
2. Task calls `xQueueReceive(q, &v, pdMS_TO_TICKS(100))` and the queue is
   empty → removed from the ready list, `xStateListItem` inserted into
   `pxDelayedTaskList` (sorted by timeout), and `xEventListItem` inserted
   into the queue's own `xTasksWaitingToReceive` list (sorted by priority).
3. Another task calls `xQueueSend` → the queue's wait list is checked;
   the waiting task is removed from *both* the delayed list and the queue's
   event list, and reinserted into its ready list. If it's now the highest
   priority ready task, a context switch is requested.
4. If nothing sends within 100ms → `xTaskIncrementTick` finds the task at
   the head of `pxDelayedTaskList` due, removes it from both the delayed
   list and the queue's event list, and moves it back to ready with a
   timeout return value.

Every one of those transitions is a `uxListRemove` plus a
`vListInsert`/`vListInsertEnd` — no allocation, no free, just pointer
rewiring on list items that were embedded in the TCB from the start. This is
why FreeRTOS task/queue operations have bounded, predictable execution
time: no dynamic list-node allocation ever happens on the hot path.

## Traps

- **Assuming ready lists are scanned linearly for the highest priority.**
  On ports with `configUSE_PORT_OPTIMISED_TASK_SELECTION` (the default on
  ARM Cortex-M), it's a bitmap + CLZ, O(1) in the number of priority levels.
  Without it (some smaller/older ports), it *is* a linear scan from
  `configMAX_PRIORITIES - 1` downward, which is one reason raising
  `configMAX_PRIORITIES` unnecessarily has a real (if usually small) cost.
- **Forgetting `xEventListItem` is sorted by priority, not FIFO.** When
  multiple tasks block on the same queue/semaphore, the *highest-priority*
  waiter is unblocked first on a give — not whichever blocked first. Two
  equal-priority waiters do resolve FIFO relative to each other, but priority
  always wins first.
- **Confusing the two delayed lists for "short" and "long" delays.** They are
  not partitioned by delay duration — they swap wholesale on tick-counter
  overflow. A common misreading of `tasks.c` is to assume list 1 is always
  "current" and list 2 is a spare; in reality either can be `pxDelayedTaskList`
  at any given moment depending on how many overflows have occurred.
- **Manipulating `TCB_t` or list fields directly.** They're `PRIVILEGED_DATA`
  and considered kernel-internal even though the struct layout is readable in
  `tasks.c` — always go through the public API (`vTaskPrioritySet`, etc.),
  since direct manipulation bypasses the critical sections that keep list
  operations atomic with respect to the tick interrupt and other tasks.
- **Assuming list operations are free of critical sections.** `vListInsert`/
  `uxListRemove` themselves are simple, but every *caller* in `tasks.c` wraps
  them in `taskENTER_CRITICAL`/`taskEXIT_CRITICAL` (or is already called from
  one) — the list itself has no internal locking, by design, to keep it fast.

## Cheat sheet

| Structure | Purpose | Sort order |
|---|---|---|
| `pxReadyTasksLists[N]` | One per priority; tasks ready to run | Insertion order (round-robin via `pxIndex`) |
| `pxDelayedTaskList` / overflow list | Tasks blocked with a timeout | Ascending wake tick |
| Queue/semaphore `xTasksWaitingToReceive`/`Send` | Tasks blocked on that object | Descending priority (then FIFO) |
| `xSuspendedTaskList` | Tasks suspended indefinitely (`vTaskSuspend`) | Insertion order |
| `xStateListItem` | Embedded in TCB; tracks *which* state list a task is on | — |
| `xEventListItem` | Embedded in TCB; tracks position on an object's wait list | Priority |
| `uxTopReadyPriority` bitmap | O(1) "highest non-empty ready priority" lookup | Bit position |

## Exercise

1. Read `list.c` in a real FreeRTOS-Kernel checkout end to end (it's under
   250 lines) and identify every place `configASSERT` would fire on a
   corrupted list — this is the fastest way to internalize the invariants
   the kernel depends on.
2. Trace through `tasks.c`'s `prvAddCurrentTaskToDelayedList` and
   `xTaskIncrementTick` and write out, in your own words, exactly what
   happens to a task's two list items (`xStateListItem`, `xEventListItem`)
   when it times out waiting on a queue versus when it's woken by a send —
   the removal bookkeeping differs subtly between the two paths.
3. On a Cortex-M target (or by reading the port source), find where
   `configUSE_PORT_OPTIMISED_TASK_SELECTION` is used and confirm which CLZ
   instruction your port relies on — then explain why this optimization is
   unavailable on a port without a count-leading-zeros instruction.
