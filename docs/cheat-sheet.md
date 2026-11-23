---
title: "FreeRTOS Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free FreeRTOS cheat sheet: 36 quick-reference tables from every level of the FreeRTOS Mastery Path, each linked to its full lesson."
---

# FreeRTOS Cheat Sheet

Quick-reference tables from **36 lessons** of the free FreeRTOS Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [01 · Why an RTOS?](level-1/01-why-an-rtos.md)

| Concept | Meaning |
|---|---|
| Superloop | One big `loop()` doing everything; response time = sum of all jobs |
| Real-time | *Deterministic* response within a deadline — not "fast" |
| Hard / soft real-time | Missed deadline = failure / = degraded quality |
| Cooperative scheduling | Tasks yield voluntarily; one bad task stalls all |
| Preemptive scheduling | Scheduler interrupts lower-priority work the instant higher-priority work is ready |
| FreeRTOS | MIT-licensed, ~10 kB kernel; the world's most deployed RTOS; built into the ESP32 Arduino core |
| When to skip an RTOS | Simple timing, tiny RAM, pure ISR designs, or no team experience + no time |

### [02 · Your First Tasks](level-1/02-first-tasks.md)

| API | Purpose |
|---|---|
| `xTaskCreate(fn, name, stack, param, prio, &handle)` | Create a task (stack in **bytes** on ESP32); returns `pdPASS` on success |
| `vTaskDelay(pdMS_TO_TICKS(ms))` | Block for a relative time (task uses no CPU while blocked) |
| `vTaskDelayUntil(&lastWake, period)` | Block until an absolute time — drift-free fixed periods |
| `pdMS_TO_TICKS(ms)` | Convert milliseconds → ticks portably |
| `vTaskDelete(NULL)` | Delete the calling task (never just `return`) |
| `vTaskSuspend(h)` / `vTaskResume(h)` | Park / unpark a task by handle |
| `xTaskGetTickCount()` | Current tick count (like `millis()` in ticks) |
| Task states | Running · Ready · Blocked (waiting, zero CPU) · Suspended |

### [03 · The Scheduler](level-1/03-the-scheduler.md)

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

### [04 · Queues](level-1/04-queues.md)

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

### [05 · Semaphores &amp; Mutexes](level-1/05-semaphores-mutexes.md)

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

### [06 · Software Timers](level-1/06-software-timers.md)

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

### [07 · Task Notifications &amp; Event Groups](level-1/07-notifications-event-groups.md)

| API | Purpose |
|---|---|
| `xTaskNotifyGive(h)` / `ulTaskNotifyTake(clear, timeout)` | Fast binary/counting semaphore aimed at one task |
| `xTaskNotify(h, val, action)` | Send value/bits: `eSetValueWithOverwrite`, `eSetBits`, `eIncrement`... |
| `xTaskNotifyWait(clrIn, clrOut, &val, timeout)` | Wait and receive the 32-bit value |
| `xEventGroupCreate()` | New event group (≤24 usable bits) |
| `xEventGroupSetBits(eg, bits)` / `xEventGroupClearBits(eg, bits)` | Set / clear flags |
| `xEventGroupWaitBits(eg, bits, clearOnExit, waitAll, timeout)` | Block on ANY/ALL combinations; returns bits at wake |
| `xEventGroupGetBits(eg)` | Peek current bits without blocking |

### [08 · Interrupts &amp; the RTOS](level-1/08-interrupts.md)

| Rule / API | Detail |
|---|---|
| Never call blocking / normal API in ISR | Use the `...FromISR` variant, always |
| `FromISR` calls never block | No timeout param; check return for `errQUEUE_FULL` |
| `woken` pattern | init `pdFALSE` → pass to every FromISR call → `portYIELD_FROM_ISR(woken)` last |
| `portYIELD_FROM_ISR(w)` | Switch straight to the newly-woken task as the ISR exits |
| Defer work | ISR: record + notify. Task: everything else |
| `IRAM_ATTR` | Put ESP32 ISRs in IRAM; no Serial, no heap, no mutexes inside |
| Best ISR→task signal | Task notification (fastest); queue when the event carries data |
| Attach order | Create the handler task *before* `attachInterrupt` |

### [09 · Memory &amp; Debugging](level-1/09-memory-debugging.md)

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

## Level 2 · Intermediate

### [Stream &amp; Message Buffers](level-2/01-stream-message-buffers.md)

| API / concept | Stream buffer | Message buffer |
|---|---|---|
| Create | `xStreamBufferCreate(size, triggerLevel)` | `xMessageBufferCreate(size)` |
| Static create | `xStreamBufferCreateStatic(...)` | `xMessageBufferCreateStatic(...)` |
| Send | `xStreamBufferSend(h, data, len, timeout)` | `xMessageBufferSend(h, data, len, timeout)` |
| Receive | `xStreamBufferReceive(h, buf, bufLen, timeout)` | `xMessageBufferReceive(h, buf, bufLen, timeout)` |
| ISR variants | `...FromISR(..., &xHigherPriorityTaskWoken)` | same pattern |
| Boundary semantics | None — arbitrary byte chunks | Preserved — one send = one receive |
| Writers/readers | Exactly one of each | Exactly one of each |
| Best for | Continuous byte streams (UART, DMA) | Discrete variable-length messages (packets, logs) |
| Wake tuning | `xStreamBufferSetTriggerLevel()` | N/A — wakes per message |

### [Static Allocation](level-2/02-static-allocation.md)

| Dynamic | Static | Notes |
|---|---|---|
| `xTaskCreate(...)` | `xTaskCreateStatic(..., stackBuf, &tcbBuf)` | static never returns NULL |
| `xQueueCreate(len, itemSize)` | `xQueueCreateStatic(len, itemSize, storage, &queueBuf)` | storage = `len * itemSize` bytes |
| `xSemaphoreCreateBinary()` | `xSemaphoreCreateBinaryStatic(&semBuf)` | |
| `xSemaphoreCreateMutex()` | `xSemaphoreCreateMutexStatic(&semBuf)` | |
| `xTimerCreate(...)` | `xTimerCreateStatic(..., &timerBuf)` | |
| `xStreamBufferCreate(size, trig)` | `xStreamBufferCreateStatic(size, trig, storage, &sbBuf)` | |
| `xEventGroupCreate()` | `xEventGroupCreateStatic(&egBuf)` | |
| Failure mode | Never fails at create-time | RAM committed at compile time either way |
| Config | `configSUPPORT_DYNAMIC_ALLOCATION` | `configSUPPORT_STATIC_ALLOCATION` |

### [Tickless Idle &amp; Low Power](level-2/03-tickless-idle-low-power.md)

| Config / API | Purpose |
|---|---|
| `configUSE_TICKLESS_IDLE` | 0 = classic ticking idle; 1 = sleep when idle time allows; 2 = ESP-IDF variant, more aggressive |
| `configEXPECTED_IDLE_TIME_BEFORE_SLEEP` | Minimum idle ticks before it's worth sleeping at all |
| `portSUPPRESS_TICKS_AND_SLEEP(ticks)` | Port-layer function that actually performs the sleep + tick correction |
| `xTaskGetTickCount()` | Always correct across sleeps — use for all app-level timing |
| `vApplicationSleepPriorToSleep/AfterSleep` | Hooks to gate/restore peripherals around a sleep |
| `esp_pm_configure()` (ESP-IDF) | Higher-level power-management config layered over tickless idle |
| Interrupt latency | Deeper sleep = higher wake latency — bound by your tightest ISR deadline |

### [ESP-IDF FreeRTOS Specifics](level-2/04-esp-idf-freertos.md)

| Concept | Vanilla FreeRTOS | ESP-IDF FreeRTOS |
|---|---|---|
| Task creation | `xTaskCreate(...)` | `xTaskCreate(...)` (= `tskNO_AFFINITY`) or `xTaskCreatePinnedToCore(..., core)` |
| Cross-core protection | N/A (single core) | `portMUX_TYPE` spinlock + `portENTER/EXIT_CRITICAL(&mux)` |
| ISR flash safety | N/A | `IRAM_ATTR` required for ISRs that may fire during flash-cache-disabled windows |
| Power management | Raw tickless-idle hooks (Module 3) | `esp_pm_configure()` (DFS + light sleep), Kconfig-driven |
| Watchdog | Opt-in, manual wiring (Module 9) | Task Watchdog Timer (TWDT) active by default |
| Stack units | Words (`StackType_t`) on most ports | Bytes, on ESP-IDF's `xTaskCreate` |
| Affinity constant | N/A | `tskNO_AFFINITY`, or explicit core `0`/`1` |

### [POSIX/Linux Simulation Port](level-2/05-posix-port.md)

| Aspect | Real hardware port | POSIX simulator port |
|---|---|---|
| Task backing | Real stack + register context switch | `pthread_t` per task |
| Tick source | Hardware timer peripheral | POSIX interval timer / signal |
| Priority effect | Real preemptive scheduling by the kernel | FreeRTOS's own logic, layered over signal-based thread suspension |
| ISR | Real interrupt controller, real ISR-safe API rules enforced by hardware | Ordinary function call or signal handler — rules are *documented*, not *enforced* by hardware |
| Timing fidelity | Ground truth | Host-OS-dependent, not deterministic |
| Best use | Final validation, power/timing measurement | Early design validation, CI, fast iteration |
| Build here | N/A (toolchain per target) | Verified: `gcc` + `-lpthread`, needs `heap_3.c` (or another heap) + both application hooks |

### [Queue Sets &amp; Advanced Queue Patterns](level-2/06-queue-sets-patterns.md)

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

### [FreeRTOSConfig.h Deep Dive](level-2/07-freertosconfig-deep-dive.md)

| Category | Key macros |
|---|---|
| Scheduling | `configUSE_PREEMPTION`, `configUSE_TIME_SLICING`, `configMAX_PRIORITIES`, `configTICK_RATE_HZ` |
| Memory | `configTOTAL_HEAP_SIZE`, `configSUPPORT_STATIC/DYNAMIC_ALLOCATION`, `configMINIMAL_STACK_SIZE` |
| Feature inclusion | `INCLUDE_vTaskDelete`, `INCLUDE_vTaskSuspend`, `INCLUDE_eTaskGetState`, etc. — compiled OUT at 0 |
| Debug/robustness | `configCHECK_FOR_STACK_OVERFLOW`, `configASSERT`, `configUSE_TRACE_FACILITY` |
| Subsystems | `configUSE_TIMERS`, `configUSE_QUEUE_SETS`, `configUSE_TASK_NOTIFICATIONS`, `configUSE_MUTEXES` |
| Required hooks | See hooks table — tied to specific `configUSE_*`/`configCHECK_*` flags |

### [Gatekeeper Tasks &amp; Design Patterns](level-2/08-gatekeeper-patterns.md)

| Pattern | Use when | Cost |
|---|---|---|
| Mutex | Short, fixed-latency critical section on shared data | Priority inheritance handles inversion; no extra task |
| Gatekeeper task | Slow/variable-latency resource; want reordering/filtering/rate-limiting | Extra task + queue; callers never block on the I/O itself |
| Producer–filter–consumer chain | Multi-stage processing pipeline | Extra latency per stage; independent reprioritization/testing |
| Relay / fan-in task | Multiple producers, single-writer-only API (stream buffer, some drivers) | Extra task; moves multi-writer problem to an ordinary (safe) queue |

### [Watchdogs &amp; Task Supervision](level-2/09-watchdogs-supervision.md)

| Concept | Purpose |
|---|---|
| Hardware WDT | Independent peripheral; resets chip if not petted in time |
| Fixed-schedule pet | Proves the watchdog task is alive — proves nothing else |
| Per-task liveness token | `lastAlive[task] = xTaskGetTickCount()` each iteration |
| `allTasksHealthy()` gate | Only pet hardware if every supervised task proved recent progress |
| ESP-IDF Task Watchdog Timer | Built-in equivalent, subscribes idle tasks by default (Module 4) |
| Persist-before-reset | Log the stale task/reason to non-volatile/battery-backed memory pre-reset |
| Fault vs. hang | Update liveness token even on a recoverable domain fault; report the fault separately |

## Level 3 · Advanced

### [Kernel Internals — Lists &amp; Ready Lists](level-3/01-kernel-internals-lists.md)

| Structure | Purpose | Sort order |
|---|---|---|
| `pxReadyTasksLists[N]` | One per priority; tasks ready to run | Insertion order (round-robin via `pxIndex`) |
| `pxDelayedTaskList` / overflow list | Tasks blocked with a timeout | Ascending wake tick |
| Queue/semaphore `xTasksWaitingToReceive`/`Send` | Tasks blocked on that object | Descending priority (then FIFO) |
| `xSuspendedTaskList` | Tasks suspended indefinitely (`vTaskSuspend`) | Insertion order |
| `xStateListItem` | Embedded in TCB; tracks *which* state list a task is on | — |
| `xEventListItem` | Embedded in TCB; tracks position on an object's wait list | Priority |
| `uxTopReadyPriority` bitmap | O(1) "highest non-empty ready priority" lookup | Bit position |

### [Context Switching Deep Dive](level-3/02-context-switching.md)

| Concept | Where it lives | Key fact |
|---|---|---|
| Task context | Pushed onto the task's own stack | Not a separate global save area |
| `pxTopOfStack` | First member of TCB | Assembly depends on zero offset |
| PendSV | Lowest interrupt priority | Deferred switch mechanism, not immediate |
| `vTaskSwitchContext` | Portable C | Only updates `pxCurrentTCB`; no register access |
| Hardware-saved regs (Cortex-M) | r0-r3, r12, LR, PC, xPSR | Auto stacked/unstacked by exception hardware |
| Manually-saved regs | r4-r11 (+ s16-s31 if FPU used) | Explicit `stmdb`/`ldmia` in the port asm |
| Measured here (POSIX sim) | 2.014 us/switch | Host-OS artifact — shape, not magnitude, transfers |

### [SMP Scheduling](level-3/03-smp-scheduling.md)

| Concept | Single-core FreeRTOS | SMP FreeRTOS |
|---|---|---|
| Ready lists | One set, one scheduler | Same one set, shared across cores |
| Critical section | Disable interrupts (this core only) | Disable interrupts + spinlock (cross-core) |
| "Running" | Exactly one task | Up to `configNUMBER_OF_CORES` tasks |
| Priority guarantee | Higher priority always preempts lower | Only guaranteed per-core unless `configRUN_MULTIPLE_PRIORITIES=0` |
| Task placement | N/A | `vTaskCoreAffinitySet` bitmask |
| Shared app data hazard | Race conditions (mutex/critical section fixes it) | Race conditions **and** cache incoherency — mutex alone may not fix it |
| Verified on this machine | N/A (source review only) | N/A — no SMP-capable POSIX simulator exists |

### [MPU Support &amp; Memory Protection](level-3/04-mpu-memory-protection.md)

| Concept | Standard FreeRTOS | FreeRTOS-MPU |
|---|---|---|
| Address space | Flat, every task sees everything | Per-task restricted regions |
| Privilege | All tasks run privileged | Privileged / unprivileged distinction |
| Bad memory access | Silent corruption | Immediate `MemManage` fault |
| Kernel API from tasks | Direct calls | `MPU_`-wrapped syscalls for unprivileged tasks |
| Stack overflow detection | Pattern/watermark (after the fact) | Optional guard region (at the instant of overflow) |
| Region limits | N/A | Hardware-fixed count + alignment rules |
| Security boundary strength | None | Bug containment — not attacker-resistant, not TrustZone-equivalent |
| Verified on this machine | N/A | N/A — no MPU-capable simulation exists; source/spec review only |

### [Tracing — Tracealyzer &amp; SystemView](level-3/05-tracing-tools.md)

| Tool/technique | Mechanism | Best for | Cost |
|---|---|---|---|
| Tracealyzer | RAM ring buffer + desktop viewer | Full timeline, priority inversion, CPU load, latency histograms | License; recorder RAM/CPU |
| SystemView | RAM buffer + RTT/SWO streaming, J-Link-centric | Real-time ISR/task interleaving without halting the CPU | License-free (Segger); needs J-Link-class probe for streaming |
| Software event counters (verified here) | Plain volatile counters + `clock_gettime`/cycle counter | Coarse "is this in the right ballpark", regression checks | Free; no tool; no timeline |
| Halting the CPU under a debugger | JTAG/SWD halt + memory read | Last resort, one-shot inspection | Perturbs timing — avoid for latency-sensitive questions |

### [Porting FreeRTOS to a New MCU](level-3/06-porting-freertos.md)

| Porting artifact | Purpose | Where the bug usually hides |
|---|---|---|
| `portmacro.h` types/macros | Basic ABI contract | Wrong stack growth direction or alignment |
| `pxPortInitialiseStack` | New task's fake "saved" context | Register order mismatch with switch-in code |
| Tick handler | Drives `xTaskIncrementTick` | Wrong interrupt priority, missed switch request |
| Switch mechanism (PendSV-equivalent) | Actual register save/restore | Priority too high, incomplete register set |
| `portENTER/EXIT_CRITICAL` | Mutual exclusion primitive | Missing nesting counter |
| Application hooks | Required by enabled config options | Missing hook → link error, easy but easy to miss |

### [Advanced ISR &amp; Latency Design](level-3/07-advanced-isr-latency.md)

| Latency component | Owner | Typical fix |
|---|---|---|
| Hardware response | Silicon | Fixed; not FreeRTOS's concern |
| Interrupt masking | Kernel critical sections | Keep critical sections short (Module 1) |
| ISR execution time | Your ISR code | Defer work via `...FromISR` |
| Dispatch latency | Scheduler + context switch | `portYIELD_FROM_ISR`, correct priorities |
| Unbounded inversion | Missing priority inheritance | Real mutexes, not binary semaphores |
| Simulated ISR on POSIX port | Port's own signal/thread machinery | Only simulate via the port's documented path — verified here that an arbitrary pthread hangs |

### [Performance Tuning &amp; Measurement](level-3/08-performance-tuning.md)

| Question | Tool/technique | This module's finding |
|---|---|---|
| Does task count affect switch time? | Before/after counter benchmark | No measurable regression for +8 suspended tasks — confirms O(1) design |
| Where's the real bottleneck? | Critical section duration, queue contention | Usually not the scheduler itself |
| Is a stack/heap size right? | `uxTaskGetStackHighWaterMark`, heap free-size APIs | Measure, don't guess |
| Is an optimization real or noise? | Multiple runs, reported variance | Single-run deltas under ~1% are not conclusive |
| Does this change hurt worst-case latency? | Cross-check against Module 7 / Level 4 Module 2 | Throughput gains can cost tail latency |

### [Networking — FreeRTOS-Plus-TCP](level-3/09-freertos-plus-tcp.md)

| Concept | Standard FreeRTOS pattern it mirrors | Purpose |
|---|---|---|
| IP task | Gatekeeper task (Level 2 Module 8) | Serializes all stack state access |
| Socket API calls | Queue send/receive to the gatekeeper | Decouples callers from stack internals |
| `NetworkBufferDescriptor_t` | Pass-by-reference buffer, like a stream buffer chunk | Avoids extra copies between driver and stack |
| IP task priority | Same priority-starvation rules as Module 7 | Network throughput is a scheduling problem too |
| `ipconfigTCP_WINDOW_SIZE` | Same RAM-budget trade-off as queue/stack sizing | Throughput vs. RAM |

## Level 4 · Master

### [Safety-Certified FreeRTOS (SAFERTOS)](level-4/01-safety-certified-freertos.md)

| Aspect | Standard FreeRTOS | SAFERTOS |
|---|---|---|
| Codebase | Open source, actively evolving | Separate, commercially licensed, independently developed |
| API shape | Familiar to this whole course | Deliberately similar, for design/skill transfer |
| Feature set | Full — dynamic allocation, broad config space | Deliberately restricted for certifiability |
| Certification | None (kernel itself is not certified) | IEC 61508 / ISO 26262 and others, depending on product/target — confirm current scope directly |
| Configuration | Flexible, frequently changed | Frozen, versioned, part of the certified artifact |
| Toolchain | Any conforming compiler | Qualified toolchain often required at higher integrity levels |
| Verified in this module | N/A | N/A — documentation/safety-case review only, no hands-on build |

### [Formal Timing Analysis (RMA &amp; WCET)](level-4/02-formal-timing-analysis.md)

| Concept | Formula/method | Guarantee |
|---|---|---|
| Rate monotonic priority assignment | Shortest period = highest priority | Optimal fixed-priority assignment for this task model |
| Liu & Layland sufficient bound | `U ≤ n(2^(1/n)−1)`, → ln(2)≈0.693 as n→∞ | Sufficient, not necessary — conservative |
| Exact schedulability test | Iterative response-time recurrence | Exact — no false negatives |
| Measured "WCET" | Max observed execution time + margin | Not a proof; can miss rare worst-case paths |
| Static WCET analysis | Tool models CFG + processor pipeline/cache | Provable upper bound, given an accurate processor model |
| Blocking-time extension | Adds priority-inheritance-bounded blocking to Cᵢ | Required for tasks that share mutexes |

### [Production Firmware Architecture](level-4/03-production-architecture.md)

| Layer | Owns | Should never contain |
|---|---|---|
| Application | Task designs, business logic | Direct vendor SDK/register calls |
| Middleware/services | Comms, logging, OTA, storage | Application-specific business rules |
| HAL | Thin wrapper over vendor SDK/registers | Business logic of any kind |
| Kernel + port | Scheduling, IPC primitives (Level 3) | Application or vendor-specific code |

| Production concern | Tool/pattern |
|---|---|
| Multi-variant builds | Compile-time flags per hardware variant, one config per variant |
| Reproducibility | Pinned toolchain + SDK versions, deterministic build system |
| Fault containment | MPU (L3M4) + deliberate per-subsystem restart-vs-reset decision |
| Resource ownership | One gatekeeper task per shared resource (L2M8), no exceptions |

### [Certification Workflows (IEC 61508 / ISO 26262)](level-4/04-certification-workflows.md)

| Concept | IEC 61508 | ISO 26262 |
|---|---|---|
| Integrity levels | SIL 1-4 | ASIL A-D (+QM) |
| Level determination | Risk-based hazard analysis | HARA (Hazard Analysis and Risk Assessment) |
| Coding standard (common practice) | Sector-dependent | MISRA C (dominant in automotive) |
| Coverage at higher levels | MC/DC often required | MC/DC often required |
| Splitting a high-integrity requirement | Not standard's own mechanism | ASIL decomposition (with independence argument) |
| RTOS timing evidence | RMA/schedulability, WCET (L4M2) | Same |
| Kernel-level evidence shortcut | Certified kernel (e.g. SAFERTOS, L4M1) | Same |

### [Secure Firmware &amp; TrustZone](level-4/05-secure-firmware-trustzone.md)

| Concept | Purpose | Enforced by |
|---|---|---|
| Secure / Non-secure worlds | Whole-system security partition | Hardware (SAU/IDAU), every bus transaction |
| MPU (Level 3 Module 4) | Task-level bug containment within one world | MPU peripheral, within one security state |
| NSC (Non-Secure Callable) region | Only permitted entry point into Secure world | Hardware, fixed veneer addresses |
| `secure_context.c` (FreeRTOS-Kernel) | Preserves per-task Secure-side call state across preemption | Kernel context-switch hook |
| Secure boot chain | Establishes trust before the partition matters at all | Immutable first-stage verification |

### [OTA Updates for RTOS Firmware](level-4/06-ota-rtos-firmware.md)

| Concept | Purpose | Failure mode it prevents |
|---|---|---|
| Dual-bank (A/B) partitioning | New image never overwrites the running one | Bricking on power loss mid-update |
| Signature verification | Confirms authenticity, not just integrity | Malicious/unauthorized image acceptance |
| Self-test / rollback window | New image must prove health before permanent commit | Bricking on a functionally broken (but well-formed, signed) image |
| Flash-write gatekeeper task | Single owner of flash writes | Corruption from concurrent writers |
| Watchdog-backed health confirmation | Forces rollback if new firmware hangs | Silent permanent failure with no recovery |

### [Fault-Tolerant &amp; High-Availability Design](level-4/07-fault-tolerant-design.md)

| Fault class | Detection mechanism | Typical recovery |
|---|---|---|
| Single task hang | Per-task supervisor heartbeat (Level 2) | Task restart |
| Memory corruption / wild pointer | MPU fault (Level 3 Module 4) | Contained restart or reset, per Level 4 Module 3's policy |
| Stack/heap exhaustion | `configCHECK_FOR_STACK_OVERFLOW`, malloc-failed hook | Usually reset — state is unreliable at this point |
| Systemic deadlock/corruption | Hardware watchdog (correctly fed) | Full reset |
| Sensor/input feed silence | Queue receive timeout | Safe-state entry, fault counter, escalate if persistent |
| Non-critical subsystem failure | Subsystem-specific health check | Graceful degradation, not full failure |

### [Multi-Core AMP Architectures](level-4/08-multicore-amp.md)

| Aspect | SMP (Level 3 Module 3) | AMP (this module) |
|---|---|---|
| Kernel instances | One, shared across cores | One per core, independent |
| Core requirement | Symmetric (identical) cores | Can be heterogeneous |
| Work placement | Dynamic (affinity-constrained, but scheduler-chosen) | Static, design-time decision |
| Coordination | Shared kernel data structures + spinlocks | Explicit IPC (shared memory, mailbox, OpenAMP) |
| Cache coherency requirement | Application data only; kernel structures already handled | Everything shared is the application's responsibility |
| Firmware update | One image, one OTA process (Level 4 Module 6) | Potentially two+ images with a version-compatibility contract |
| Common framework | Built into FreeRTOS-Kernel's SMP support | OpenAMP/remoteproc (widely used, not universal) |

### [Long-Term Maintenance &amp; LTS Strategy](level-4/09-lts-maintenance.md)

| Concern | Mechanism | Connects to |
|---|---|---|
| Stability without feature churn | FreeRTOS-Kernel LTS branches | — |
| Security patch tracking | Advisory monitoring + accurate SBOM | Level 3 Module 9 (network attack surface) |
| Deploying a fix to the field | OTA infrastructure | Level 4 Module 6 |
| Certified-product update discipline | Re-verification against the frozen safety case | Level 4 Module 1, Module 4 |
| Field-only fault discovery | Telemetry/crash reporting | Level 4 Module 7's fault-detection mechanisms |
| Silicon end-of-life resilience | HAL/layering discipline | Level 4 Module 3 |

---

Go deeper with the full [FreeRTOS Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
