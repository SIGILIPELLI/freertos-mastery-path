# 09 · Memory & Debugging

The two questions every new RTOS programmer eventually asks at 2 a.m.: *"how
big should this task's stack be?"* and *"why did it crash?"* — and they're
usually the same question. This module gives you the tools: measuring real
stack usage with high-water marks, understanding where queues and tasks
actually get their RAM (the five heap schemes), reading task lists and
runtime stats, and a field guide to the crashes you'll actually meet.

## Task stacks: sizing without guessing

Every task's stack must hold its deepest call chain — locals, function
frames, `printf` buffers — *at the worst moment*. Overflow doesn't politely
error; it silently tramples whatever memory sits next door. Symptoms:
corrupted variables, garbled task names, hard crashes minutes after the real
cause. (ESP32 checks stack canaries on context switches and usually aborts
with `Guru Meditation` / a stack-overflow message naming the task — vanilla
FreeRTOS offers the same via `configCHECK_FOR_STACK_OVERFLOW` and the
`vApplicationStackOverflowHook`.)

The honest approach: **start generous, measure, then trim.** The measuring
tool is the **stack high-water mark** — the minimum free stack the task has
*ever* had:

```cpp
void monitoredTask(void *pv) {
  for (;;) {
    doRealWork();
    // Bytes of stack that have NEVER been used (ESP32: bytes; vanilla: words!)
    UBaseType_t freeBytes = uxTaskGetStackHighWaterMark(NULL);  // NULL = me
    Serial.printf("worker: min free stack ever = %u bytes\n", freeBytes);
    vTaskDelay(pdMS_TO_TICKS(2000));
  }
}
```

Workflow: give the task 4096, exercise **every code path** (especially error
paths and the biggest `printf`), read the high-water mark, then set the
stack to *peak usage plus ~25-50 % margin*. A mark near zero means you're
one function call from disaster; a mark of 3000 on a 4096 stack means you
can hand ~2.5 kB back. You can also query other tasks by handle — a
monitor task that walks your handles and reports all marks every 10 s is
cheap insurance in development builds.

## Where the RAM comes from: heap_1 … heap_5

`xTaskCreate`, `xQueueCreate`, and friends allocate from **FreeRTOS's own
heap**, not the C library's `malloc` pool (via `pvPortMalloc`). Vanilla
FreeRTOS ships five interchangeable implementations — you pick one at build
time:

| Scheme | Behavior | Use when |
|---|---|---|
| `heap_1` | Allocate only — **no free** | Everything created once at boot; simplest, zero fragmentation |
| `heap_2` | Free, best-fit, no coalescing | Legacy — superseded by heap_4 |
| `heap_3` | Thin wrapper over C `malloc`/`free` | You must share one heap with library code |
| `heap_4` | Free + **coalescing** of adjacent blocks | The sensible general-purpose default |
| `heap_5` | heap_4 across **multiple memory regions** | RAM split across non-contiguous banks |

The ESP32 supplies its own thread-safe allocator (heap_4-like, multi-region
— closest in spirit to heap_5) so on this course's platform you don't
choose; but the *discipline* transfers everywhere: **create tasks, queues,
and semaphores once at startup and never delete them.** Create/delete churn
at runtime risks fragmentation — the heap has bytes free, but no block big
enough for a new task stack. Check for it:

```cpp
Serial.printf("heap free: %u, min ever: %u, largest block: %u\n",
              ESP.getFreeHeap(),        // xPortGetFreeHeapSize() in vanilla
              ESP.getMinFreeHeap(),     // xPortGetMinimumEverFreeHeapSize()
              ESP.getMaxAllocHeap());   // fragmentation tell-tale
```

`getMinFreeHeap` is the heap's high-water mark — if it trends toward zero
across hours, you have a leak; if `getMaxAllocHeap` shrinks while
`getFreeHeap` stays flat, you have fragmentation. (Level 2's static
allocation module shows how to opt out of the heap entirely with
`xTaskCreateStatic`.)

## Seeing the system: task lists & runtime stats

`vTaskList` prints a snapshot of every task — state, priority, stack
high-water mark, task number:

```cpp
void statsTask(void *pv) {
  char buf[768];
  for (;;) {
    vTaskDelay(pdMS_TO_TICKS(5000));
    vTaskList(buf);                     // requires configUSE_TRACE_FACILITY
    Serial.println("Name          State  Prio   FreeStk  Num");
    Serial.println(buf);
  }
}
```

```text
Name          State  Prio   FreeStk  Num
button          B      4      1420    6      ← B = Blocked (good: waiting)
prod            B      2       804    5
statsTask       R      1       520    7      ← R = Running/Ready
IDLE0           R      0       628    3
Tmr Svc         B      1      1608    4
```

Read it like a doctor: tasks should spend their lives **B**locked; a task
that's always **R**eady is a starvation suspect; a `FreeStk` under a couple
hundred bytes is a stack overflow waiting to happen. The companion
`vTaskGetRunTimeStats(buf)` (needs a runtime counter configured; available
in ESP-IDF builds) shows *percent CPU per task* — the ground truth for "what
is eating my processor."

!!! note "Arduino-ESP32 config"
    `vTaskList`/runtime stats need `configUSE_TRACE_FACILITY` (and friends)
    enabled. Recent arduino-esp32 cores ship with the trace facility on; if
    your build errors on `vTaskList`, fall back to the always-available
    singles: `uxTaskGetNumberOfTasks()`, `uxTaskGetStackHighWaterMark(h)`,
    and `eTaskGetState(h)` — or use ESP-IDF where it's a menuconfig switch.

## Field guide to common crashes

| Symptom | Likely cause | First move |
|---|---|---|
| `Debug exception reason: Stack canary watchpoint triggered (taskname)` | **Stack overflow** in that task | Raise its stack ×2, then right-size via high-water mark |
| Corrupted variables / garbled strings near a task's globals | Stack overflow trampling neighbors | Same as above — check *all* marks |
| `assert failed: xQueueGenericSend queue.c` (IN_ISR variants mentioned) | Called a **non-FromISR API in an ISR** | Audit every callback that runs in ISR context |
| `Guru Meditation Error: ... (LoadProhibited)` at address `0x0` | NULL handle — used a queue/semaphore **before creating it** | Create kernel objects before the tasks that use them; check create results |
| `Task watchdog got triggered (IDLE0)` | A task **never blocks** (starvation) | Find the busy loop; make it block (Module 3) |
| Works for hours, then `pvPortMalloc` fails / create returns NULL | **Heap exhaustion** — leak or fragmentation | Watch `getMinFreeHeap` trend; stop runtime create/delete churn |
| Random corruption only under load | Torn shared data (no mutex) or pointer-queue **ownership violation** | Re-read Modules 4-5; audit every shared pointer |
| Everything freezes but ISRs still fire | **Deadlock** between mutexes | Take with timeouts in debug; log each take/give |

General debugging posture for concurrent firmware: add timestamps
(`millis()`) to every log line, log *transitions* not states, keep a serial
"health" task at low priority printing heap + stack marks — and when
something is weird, suspect sharing first.

## Cheat sheet

| Tool | What it tells you |
|---|---|
| `uxTaskGetStackHighWaterMark(h)` | Minimum ever free stack (ESP32: bytes) — size stacks with it |
| Stack sizing workflow | Start big → exercise worst paths → trim to peak + 25-50 % |
| `heap_1`…`heap_5` | Vanilla FreeRTOS heap options; heap_4 = default choice; ESP32 has its own |
| `ESP.getFreeHeap()` / `getMinFreeHeap()` / `getMaxAllocHeap()` | Current / worst-ever / largest-block heap health |
| `vTaskList(buf)` | Snapshot: every task's state, priority, free stack |
| `vTaskGetRunTimeStats(buf)` | CPU % per task (needs runtime counter) |
| Golden rule | Create kernel objects at startup; don't create/delete at runtime |
| Crash suspects, in order | Stack overflow → ISR API misuse → NULL handles → starvation → heap → sharing bugs |

## Exercise

1. Write a task that calls a recursive function (e.g. naive `fib(n)`), and
   a monitor that prints its high-water mark every second. Increase `n` run
   by run until the ESP32 reports the canary watchpoint — note the
   `fib` depth and the mark you saw just before death. Then size the stack
   "correctly" for `fib(15)` with 30 % margin and prove it survives.
2. Deliberately cause, observe, and then fix: (a) a NULL-handle crash
   (use a queue before creating it), (b) an idle-watchdog trigger (a
   priority-2 task with no blocking call). Record the exact error text each
   produced — you're building your personal crash-signature dictionary.
3. Add a `health` task (priority 1) to your Module 4 pipeline printing
   free heap, min heap, and every task's stack mark every 5 s. Run it for
   several minutes: is anything trending? How much stack could you reclaim
   in total?
