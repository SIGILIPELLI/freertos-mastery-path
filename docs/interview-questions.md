---
title: "FreeRTOS Interview Questions and Answers (179 Questions)"
description: "179 FreeRTOS interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the FreeRTOS Mastery Path."
---

# FreeRTOS Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free FreeRTOS Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · Why an RTOS?](level-1/01-why-an-rtos.md)

**Q1.** What limits a superloop as a design?

**Answer:** One slow or blocking step delays everything else; timing and responsiveness become entangled

An RTOS gives each activity its own task.

**Q2.** What does an RTOS scheduler provide?

**Answer:** Priority-based preemptive scheduling with bounded, predictable response

Determinism.

**Q3.** What's a real-time system defined by?

**Answer:** Meeting deadlines predictably, not raw speed

Hard vs soft.

**Q4.** When is an RTOS overkill?

**Answer:** Very simple devices with one or two simple activities

Weigh overhead.

### [02 · Your First Tasks](level-1/02-first-tasks.md)

**Q5.** What is a FreeRTOS task?

**Answer:** A C function with its own stack and priority that runs an infinite loop

xTaskCreate.

**Q6.** How must a task function end?

**Answer:** It must not return; loop forever or delete itself with vTaskDelete(NULL)

Else undefined behavior.

**Q7.** What does xTaskCreate's stack size parameter mean in FreeRTOS?

**Answer:** The stack depth in words (not bytes)

Port-specific word size.

**Q8.** What does vTaskDelay do?

**Answer:** Blocks the calling task for a number of ticks, letting others run

Use vTaskDelayUntil for periodic work.

### [03 · The Scheduler](level-1/03-the-scheduler.md)

**Q9.** What's the scheduler's core rule?

**Answer:** Run the highest-priority task that is Ready

Equal priorities time-slice.

**Q10.** What are task states?

**Answer:** Running, Ready, Blocked and Suspended

State diagram.

**Q11.** What runs when no task is ready?

**Answer:** The idle task

Runs at priority 0.

**Q12.** What happens to equal-priority ready tasks?

**Answer:** They share the CPU by time slicing each tick

configUSE_TIME_SLICING.

### [04 · Queues](level-1/04-queues.md)

**Q13.** Why use queues instead of globals?

**Answer:** Safe copy-based data transfer between tasks with blocking and timeouts

No races.

**Q14.** What does xQueueReceive with portMAX_DELAY do?

**Answer:** Blocks until an item arrives

Indefinite wait.

**Q15.** What does a queue item copy mean?

**Answer:** Data is copied in; senders can reuse their buffers

Copy by value.

**Q16.** What's the downside of sending large structs by value?

**Answer:** Copy cost; send pointers to buffers with clear ownership

Ownership rules.

### [05 · Semaphores &amp; Mutexes](level-1/05-semaphores-mutexes.md)

**Q17.** What's the difference between a binary semaphore and a mutex?

**Answer:** A mutex has ownership and priority inheritance; a semaphore is for signaling

Use mutexes to protect resources.

**Q18.** What problem does priority inheritance solve?

**Answer:** Priority inversion: a low-priority holder temporarily inherits the waiter's priority

Mutex feature.

**Q19.** Why can't a mutex be given from an ISR?

**Answer:** Mutexes have task ownership; use semaphores for ISR signaling

FromISR APIs.

**Q20.** What does a counting semaphore model?

**Answer:** A number of available resources or pending events

Counts up and down.

### [06 · Software Timers](level-1/06-software-timers.md)

**Q21.** When use a software timer?

**Answer:** For lightweight periodic/one-shot callbacks without a dedicated task

Timer service task runs callbacks.

**Q22.** Where do timer callbacks run?

**Answer:** In the timer service (daemon) task, so they must not block

Keep short.

**Q23.** What controls the timer task's priority?

**Answer:** configTIMER_TASK_PRIORITY

Set high enough.

**Q24.** What can't a timer callback do?

**Answer:** Block (e.g. wait on a queue forever)

Would stall all timers.

### [07 · Task Notifications &amp; Event Groups](level-1/07-notifications-event-groups.md)

**Q25.** What are task notifications?

**Answer:** A lightweight per-task signal/value mechanism, faster and smaller than queues or semaphores

xTaskNotify.

**Q26.** What are event groups for?

**Answer:** Waiting on combinations of bits (events) from several sources

xEventGroupWaitBits.

**Q27.** When prefer notifications?

**Answer:** One-to-one signaling to a known task

Lower RAM/CPU.

**Q28.** What can an event group do that a semaphore can't?

**Answer:** Wait for ALL or ANY of several event bits

Synchronization.

### [08 · Interrupts &amp; the RTOS](level-1/08-interrupts.md)

**Q29.** Which API do you call from an ISR?

**Answer:** The FromISR variants (e.g. xQueueSendFromISR)

Never block in ISR.

**Q30.** What does the higher-priority-task-woken flag do?

**Answer:** Lets you request a context switch on ISR exit (portYIELD_FROM_ISR)

Minimize latency.

**Q31.** What should an ISR do?

**Answer:** Minimal work: capture data and defer processing to a task

Deferred interrupt handling.

**Q32.** What interrupt priorities may call FreeRTOS APIs?

**Answer:** Those at or below configMAX_SYSCALL_INTERRUPT_PRIORITY

Cortex-M rule.

### [09 · Memory &amp; Debugging](level-1/09-memory-debugging.md)

**Q33.** How do you size a task stack?

**Answer:** Measure high-water marks (uxTaskGetStackHighWaterMark) with margin

Evidence.

**Q34.** What does configCHECK_FOR_STACK_OVERFLOW provide?

**Answer:** A hook when a stack overflow is detected

Debug aid.

**Q35.** What does a hard fault often indicate in FreeRTOS apps?

**Answer:** Stack overflow, bad pointer or wrong ISR API use

Inspect registers.

**Q36.** What tool helps see task runtime?

**Answer:** Run-time stats (vTaskGetRunTimeStats) or trace tools

Profile.

### [10 · Capstone — Smart Sensor Device](level-1/10-capstone-project.md)

**Q37.** What does the capstone combine?

**Answer:** Sensor, processing and output tasks with queues, timers and a watchdog

Smart sensor device.

**Q38.** Why decouple tasks with queues?

**Answer:** Loose coupling and bounded buffering between rates

Pipeline.

**Q39.** How do you verify timing?

**Answer:** Measure latency/jitter and stack usage under load

Evidence.

**Q40.** What should you do on queue full?

**Answer:** Define a policy: drop oldest, block with timeout or count overruns

Overload handling.

## Level 2 · Intermediate

### [Stream &amp; Message Buffers](level-2/01-stream-message-buffers.md)

**Q41.** When use stream/message buffers?

**Answer:** Byte streams or variable-length messages between one writer and one reader

UART/DMA handoff.

**Q42.** What's the restriction on stream buffers?

**Answer:** Single writer and single reader (unless protected)

Lock-free design.

**Q43.** What's a trigger level?

**Answer:** How many bytes must be available to unblock the reader

Reduce wakeups.

**Q44.** Why use them over queues for byte streams?

**Answer:** Less overhead than one queue item per byte

Efficient.

### [Static Allocation](level-2/02-static-allocation.md)

**Q45.** What's static allocation?

**Answer:** Providing RAM for kernel objects at compile time instead of using the heap

xTaskCreateStatic.

**Q46.** Why use it?

**Answer:** Deterministic memory, no heap fragmentation or runtime failures

Safety-critical.

**Q47.** What must you provide for a static task?

**Answer:** Stack buffer and a StaticTask_t TCB

Application-owned.

**Q48.** What extra callback is needed?

**Answer:** vApplicationGetIdleTaskMemory (and timer task memory)

When static only.

### [Tickless Idle &amp; Low Power](level-2/03-tickless-idle-low-power.md)

**Q49.** What does tickless idle do?

**Answer:** Stops the periodic tick while idle so the MCU can sleep longer

Saves power.

**Q50.** What's configUSE_TICKLESS_IDLE?

**Answer:** Enables suppression of ticks in the idle task

Port support needed.

**Q51.** What does the expected idle time determine?

**Answer:** Whether it's worth entering a low-power sleep

Minimum threshold.

**Q52.** What must you restore after sleep?

**Answer:** The tick count to account for elapsed time

Kernel compensates.

### [ESP-IDF FreeRTOS Specifics](level-2/04-esp-idf-freertos.md)

**Q53.** How does ESP-IDF FreeRTOS differ?

**Answer:** It's SMP-capable with core affinity and extra APIs (xTaskCreatePinnedToCore)

Dual-core.

**Q54.** What's task pinning?

**Answer:** Binding a task to a specific core

Affects performance.

**Q55.** Why use critical sections carefully on SMP?

**Answer:** They need spinlocks across cores (portENTER_CRITICAL with a mux)

Multi-core hazards.

**Q56.** What does the IDF add?

**Answer:** Tick hooks, dual idle tasks and task watchdog integration

Platform.

### [POSIX/Linux Simulation Port](level-2/05-posix-port.md)

**Q57.** What does the POSIX port allow?

**Answer:** Running FreeRTOS on a Linux/macOS host for development and testing

Simulation.

**Q58.** What's a limitation of the host port?

**Answer:** Timing isn't real-time and hardware isn't present

Logic only.

**Q59.** Why test on host?

**Answer:** Fast iteration and CI without hardware

Automation.

**Q60.** What do you stub on host?

**Answer:** Hardware access via an abstraction layer

HAL.

### [Queue Sets &amp; Advanced Queue Patterns](level-2/06-queue-sets-patterns.md)

**Q61.** What do queue sets do?

**Answer:** Let a task block on multiple queues/semaphores at once

xQueueSelectFromSet.

**Q62.** What's often a simpler alternative?

**Answer:** Task notifications or event groups, or a single queue of tagged messages

Design.

**Q63.** What's a producer-consumer pattern?

**Answer:** Tasks produce items into a queue and others consume them

Decoupling.

**Q64.** What problem does a pipeline pattern solve?

**Answer:** Splitting processing into stages communicating via queues

Parallelism.

### [FreeRTOSConfig.h Deep Dive](level-2/07-freertosconfig-deep-dive.md)

**Q65.** Which config enables preemption?

**Answer:** configUSE_PREEMPTION

1 = preemptive.

**Q66.** What does configTICK_RATE_HZ set?

**Answer:** The tick frequency (scheduler resolution)

Typically 100-1000.

**Q67.** What does configASSERT do?

**Answer:** Traps invalid API use and internal errors during development

Define it.

**Q68.** What does configMAX_PRIORITIES control?

**Answer:** Number of priority levels (RAM use)

Choose small.

### [Gatekeeper Tasks &amp; Design Patterns](level-2/08-gatekeeper-patterns.md)

**Q69.** What's the gatekeeper task pattern?

**Answer:** One task owns a resource (e.g. UART/logging) and others send it requests via a queue

Avoids contention.

**Q70.** Why prefer gatekeeper over a mutex?

**Answer:** Avoids priority inversion and deadlock risks and serializes access cleanly

Single owner.

**Q71.** How do clients send log messages?

**Answer:** Post message structs to the gatekeeper's queue

Thread-safe logging.

**Q72.** What's a trade-off?

**Answer:** Latency of queueing and the gatekeeper's stack/queue usage

Design.

### [Watchdogs &amp; Task Supervision](level-2/09-watchdogs-supervision.md)

**Q73.** What does a supervisor/task watchdog do?

**Answer:** Ensures every critical task checks in, resetting or recovering if one hangs

Heartbeats.

**Q74.** How avoid blind feeding?

**Answer:** Feed the hardware watchdog only after all tasks reported healthy

Aggregate.

**Q75.** What should happen after a watchdog reset?

**Answer:** Record the cause and enter a safe state or recover

Post-mortem.

**Q76.** What do you track per task?

**Answer:** Last check-in time against a deadline

Supervision.

### [Project — Multi-Sensor Gateway](level-2/10-project-multi-sensor-gateway.md)

**Q77.** What does the gateway project combine?

**Answer:** Multiple sensor streams, filtering/framing, buffering and uplink with supervision

Level 2 project.

**Q78.** How do you handle slow uplinks?

**Answer:** Buffer with queues/stream buffers and apply backpressure/drop policy

Overload.

**Q79.** What verifies robustness?

**Answer:** Fault injection: sensor loss, queue full, link down

Test failures.

**Q80.** What proves timing?

**Answer:** Measuring latencies and checking worst-case response

Evidence.

## Level 3 · Advanced

### [Kernel Internals — Lists &amp; Ready Lists](level-3/01-kernel-internals-lists.md)

**Q81.** How does FreeRTOS keep ready tasks?

**Answer:** Doubly linked lists per priority (pxReadyTasksLists)

O(1) operations.

**Q82.** What's a delayed list?

**Answer:** A list ordered by wake time for blocked-with-timeout tasks

Tick processing.

**Q83.** What are event lists?

**Answer:** Lists of tasks waiting on a queue or semaphore, ordered by priority

Wake highest priority first.

**Q84.** Why intrusive lists?

**Answer:** List nodes live inside the TCB, avoiding extra allocation

Efficient.

### [Context Switching Deep Dive](level-3/02-context-switching.md)

**Q85.** What does a context switch save and restore?

**Answer:** The task's CPU registers and stack pointer

PendSV on Cortex-M.

**Q86.** Why use PendSV for switching on Cortex-M?

**Answer:** It runs at the lowest priority after other ISRs complete

Deferred switch.

**Q87.** What triggers a switch?

**Answer:** The tick interrupt, yielding or a higher-priority task becoming ready

Scheduler decision.

**Q88.** What's the cost driver?

**Answer:** Register count and memory access for saving state

Optimize ISR latency.

### [SMP Scheduling](level-3/03-smp-scheduling.md)

**Q89.** What is SMP in FreeRTOS?

**Answer:** One kernel scheduling tasks across multiple cores

Symmetric multiprocessing.

**Q90.** What's a new hazard in SMP?

**Answer:** True parallelism: data races between tasks on different cores

Locks needed.

**Q91.** What does core affinity do?

**Answer:** Restricts a task to specific cores

Control placement.

**Q92.** Which primitive protects data across cores?

**Answer:** Spinlocks/critical sections designed for SMP

portENTER_CRITICAL_SMP.

### [MPU Support &amp; Memory Protection](level-3/04-mpu-memory-protection.md)

**Q93.** What does the MPU provide?

**Answer:** Hardware memory protection regions restricting task access

Fault isolation.

**Q94.** Why use MPU-enabled FreeRTOS?

**Answer:** A faulty or malicious task can't corrupt kernel or other tasks' memory

Safety/security.

**Q95.** What are restricted tasks?

**Answer:** Tasks with defined memory regions and unprivileged mode

xTaskCreateRestricted.

**Q96.** What happens on violation?

**Answer:** A memory-management fault you can handle

Detect.

### [Tracing — Tracealyzer &amp; SystemView](level-3/05-tracing-tools.md)

**Q97.** What do trace tools (Tracealyzer/SystemView) show?

**Answer:** Task switching, ISR timing and kernel events over time

Timeline.

**Q98.** Why trace rather than print?

**Answer:** Printing perturbs timing; tracing is low overhead

Observe real behavior.

**Q99.** What can you find with traces?

**Answer:** Priority inversion, CPU load and missed deadlines

Insight.

**Q100.** What does the trace buffer need?

**Answer:** RAM and a way to stream or snapshot

Design.

### [Porting FreeRTOS to a New MCU](level-3/06-porting-freertos.md)

**Q101.** What does porting FreeRTOS need?

**Answer:** Architecture-specific context switch, tick and critical section code (port layer)

portmacro.h/port.c.

**Q102.** What must you define for a new MCU?

**Answer:** Tick timer, PendSV/SVC handlers and stack frame layout

Port files.

**Q103.** How verify a port?

**Answer:** Run the kernel test suite and demo apps on hardware

Validation.

**Q104.** What licenses/config matter?

**Answer:** FreeRTOSConfig.h and heap implementation selection

Config.

### [Advanced ISR &amp; Latency Design](level-3/07-advanced-isr-latency.md)

**Q105.** What increases interrupt latency?

**Answer:** Long critical sections and disabled interrupts in kernel/app code

Keep short.

**Q106.** How reduce ISR latency?

**Answer:** Keep ISRs short, use appropriate priorities and avoid long critical sections

Deferred handling.

**Q107.** What's zero-latency interrupts?

**Answer:** Interrupts above the syscall priority that never call kernel APIs

Cortex-M design.

**Q108.** How measure latency?

**Answer:** With GPIO toggles and an oscilloscope or logic analyzer

Real data.

### [Performance Tuning &amp; Measurement](level-3/08-performance-tuning.md)

**Q109.** What's the first step in tuning?

**Answer:** Measure CPU load, latency and stack/heap usage

Profile.

**Q110.** What reduces context-switch overhead?

**Answer:** Fewer tasks, coarser priorities and event-driven design

Design.

**Q111.** What's wasteful?

**Answer:** Polling loops with short delays instead of blocking on events

Block.

**Q112.** What do task notifications improve?

**Answer:** Signaling speed and RAM use versus semaphores

Lightweight.

### [Networking — FreeRTOS-Plus-TCP](level-3/09-freertos-plus-tcp.md)

**Q113.** What is FreeRTOS+TCP?

**Answer:** A TCP/IP stack integrated with FreeRTOS tasks and buffers

Networking.

**Q114.** What's a network buffer's ownership model?

**Answer:** Buffers move between driver, stack and application with defined ownership

Zero copy.

**Q115.** What task does the stack use?

**Answer:** An IP task processing events and packets

Priority matters.

**Q116.** What must you size carefully?

**Answer:** Network buffers, IP task stack and socket buffers

RAM budget.

### [Project — Instrumented RTOS Firmware](level-3/10-project-instrumented-firmware.md)

**Q117.** What does the instrumented firmware project add?

**Answer:** Tracing, stats, assertions and fault handlers to observe a running system

Observability.

**Q118.** What do you capture on a crash?

**Answer:** Fault registers, stack and task info stored for post-mortem

Diagnostics.

**Q119.** What proves instrumentation is cheap?

**Answer:** Measuring overhead with and without it

Quantify.

**Q120.** What should be disabled in production?

**Answer:** Verbose tracing and debug-only checks (or made low overhead)

Balance.

## Level 4 · Master

### [Safety-Certified FreeRTOS (SAFERTOS)](level-4/01-safety-certified-freertos.md)

**Q121.** What is SAFERTOS/FreeRTOS safety variant for?

**Answer:** Certified kernel for safety-critical systems with supporting evidence

IEC 61508.

**Q122.** What do certifications require?

**Answer:** Documented process, traceability, testing and independent assessment

Evidence.

**Q123.** Why use a certified kernel?

**Answer:** Reduces certification effort by providing pre-approved evidence

Pre-certified.

**Q124.** Does certification of the kernel certify your product?

**Answer:** No: the integration and application still need assessment

System-level.

### [Formal Timing Analysis (RMA &amp; WCET)](level-4/02-formal-timing-analysis.md)

**Q125.** What is response time analysis?

**Answer:** Computing a task's worst-case response time from higher-priority interference

Formal timing.

**Q126.** Utilization of tasks (C,T)=(1,4),(2,8),(1,10). What's total CPU utilization?

```python
u=1/4+2/8+1/10
print(round(u,3))
```

**Answer:** 0.6

Sum of C/T. Verified by running it.

**Q127.** The Liu-Layland bound for n=3 rate-monotonic tasks is?

```python
n=3
print(round(n*(2**(1/n)-1),3))
```

**Answer:** 0.78

If utilization is below this bound, RMS guarantees schedulability. Verified by running it.

**Q128.** Why measure worst-case execution time (WCET)?

**Answer:** Deadline guarantees need upper bounds, not averages

Static analysis/measurement.

### [Production Firmware Architecture](level-4/03-production-architecture.md)

**Q129.** What does a production architecture include?

**Answer:** Layered modules, HAL, task design, error handling, logging and update support

Maintainable.

**Q130.** What's a good task decomposition rule?

**Answer:** Tasks by timing/ownership boundaries, not by function count

Fewer, well-defined tasks.

**Q131.** How document timing?

**Answer:** Task table with period, priority, deadline and WCET

Analysis.

**Q132.** What reduces coupling?

**Answer:** Message passing instead of shared state

Queues.

### [Certification Workflows (IEC 61508 / ISO 26262)](level-4/04-certification-workflows.md)

**Q133.** What does certification workflow include?

**Answer:** Requirements, design, implementation, verification and traceability artifacts

Process.

**Q134.** Why traceability?

**Answer:** Show each requirement is implemented and tested

Audit.

**Q135.** What coding standard is common?

**Answer:** MISRA C

Safety-critical C.

**Q136.** What's independent verification?

**Answer:** Review/testing by someone other than the author

Reduce bias.

### [Secure Firmware &amp; TrustZone](level-4/05-secure-firmware-trustzone.md)

**Q137.** What does TrustZone-M provide?

**Answer:** Hardware-isolated secure and non-secure worlds on Cortex-M

Secure firmware.

**Q138.** What belongs in the secure world?

**Answer:** Keys, crypto and secure boot logic

Minimize.

**Q139.** How does non-secure code call secure services?

**Answer:** Through veneers/non-secure-callable entry points

SG instruction.

**Q140.** What's a caution with RTOS and TrustZone?

**Answer:** Secure context management for tasks calling secure code

Port support (secure context).

### [OTA Updates for RTOS Firmware](level-4/06-ota-rtos-firmware.md)

**Q141.** What makes RTOS OTA safe?

**Answer:** Dual banks, signature verification and rollback

Fail-safe.

**Q142.** How is update integrity ensured?

**Answer:** Signed images verified by the bootloader before activation

Authenticity.

**Q143.** What happens if a new image fails to confirm?

**Answer:** The bootloader reverts to the previous one

Health check.

**Q144.** Why stage rollouts?

**Answer:** Limit damage of a bad firmware

Canary.

### [Fault-Tolerant &amp; High-Availability Design](level-4/07-fault-tolerant-design.md)

**Q145.** What is fault-tolerant design?

**Answer:** Continuing safely or recovering when components fail

Redundancy and supervision.

**Q146.** What are common techniques?

**Answer:** Watchdogs, redundancy, plausibility checks and safe states

Layered.

**Q147.** What's a safe state?

**Answer:** A defined condition that avoids harm on faults

Design for it.

**Q148.** How test fault tolerance?

**Answer:** Fault injection and chaos-style tests

Prove it.

### [Multi-Core AMP Architectures](level-4/08-multicore-amp.md)

**Q149.** What is AMP?

**Answer:** Asymmetric multiprocessing: cores run different OS/firmware (e.g. Linux + FreeRTOS)

Heterogeneous.

**Q150.** How do cores communicate in AMP?

**Answer:** Shared memory with mailboxes/RPMsg

IPC.

**Q151.** Why use AMP?

**Answer:** Isolate hard real-time work from a rich OS

Determinism.

**Q152.** What's the hazard?

**Answer:** Cache coherence and synchronization across cores

Careful design.

### [Long-Term Maintenance &amp; LTS Strategy](level-4/09-lts-maintenance.md)

**Q153.** What does LTS maintenance involve?

**Answer:** Using long-term support releases and applying security patches

Stability.

**Q154.** Why pin kernel versions?

**Answer:** Reproducibility and controlled upgrades

Test before upgrade.

**Q155.** How track vulnerabilities?

**Answer:** Subscribe to advisories and maintain an SBOM

Security.

**Q156.** What protects upgrades?

**Answer:** Regression tests and hardware-in-the-loop CI

Automation.

### [Capstone — Production RTOS Product](level-4/10-capstone-production-product.md)

**Q157.** What does the capstone integrate?

**Answer:** Architecture, timing analysis, security, OTA and maintenance for a product

Production product.

**Q158.** How do you prove readiness?

**Answer:** Timing analysis, fault injection, HIL tests and review

Evidence.

**Q159.** What documentation matters?

**Answer:** Architecture, timing budget, safety/security case and runbooks

Lifecycle.

**Q160.** What sustains a product?

**Answer:** Maintenance process and monitoring in the field

Years.

## Mixed questions across all levels

**Q161.** Why does an RTOS use priorities instead of a superloop?

**Answer:** So urgent activities preempt less urgent ones, giving bounded response times

Determinism.

**Q162.** Which API do you call inside an ISR to send to a queue?

**Answer:** xQueueSendFromISR (and yield if a higher-priority task was woken)

Never block in ISRs.

**Q163.** A task's xTaskCreate stack depth of 128 means what on a 32-bit port?

**Answer:** 128 words (512 bytes), not bytes

Check your port.

**Q164.** Two tasks share a UART. What's the cleanest design?

**Answer:** A gatekeeper task owning the UART with a queue of requests

Avoids contention and inversion.

**Q165.** A low-priority task holds a mutex needed by a high-priority task while a medium task runs. What is this?

**Answer:** Priority inversion; mutexes with priority inheritance mitigate it

Mars Pathfinder.

**Q166.** You need one task to signal another with minimum RAM and overhead. What do you use?

**Answer:** A direct-to-task notification

Lightweight.

**Q167.** A timer callback calls a blocking queue receive. What goes wrong?

**Answer:** It blocks the timer service task and stalls every software timer

Callbacks must not block.

**Q168.** The ISR handles a UART byte stream at high rate. Which primitive fits?

**Answer:** A stream buffer with a trigger level (single writer, single reader)

Low overhead.

**Q169.** How do you size a task stack?

**Answer:** Measure the high-water mark under worst-case load and add margin

uxTaskGetStackHighWaterMark.

**Q170.** A hard fault happens only under load. What's a likely cause?

**Answer:** Stack overflow in a task (enable stack overflow checking)

configCHECK_FOR_STACK_OVERFLOW.

**Q171.** Why use static allocation in a safety product?

**Answer:** Deterministic memory with no runtime heap failures or fragmentation

xTaskCreateStatic.

**Q172.** How does tickless idle save power?

**Answer:** It suppresses the periodic tick so the MCU sleeps until the next event or timeout

configUSE_TICKLESS_IDLE.

**Q173.** A task never feeds the watchdog because it's blocked forever. What's the right supervision design?

**Answer:** Each critical task checks in with a supervisor that feeds the hardware watchdog only when all are healthy

Avoid blind feeding.

**Q174.** On a dual-core ESP32, two tasks race on shared data even with taskENTER_CRITICAL on one core. Why?

**Answer:** Critical sections need an SMP spinlock (mux) to exclude the other core

Use portMUX.

**Q175.** Tasks (C,T): (1,4), (2,8), (1,10). What's the CPU utilization?

```python
print(round(1/4+2/8+1/10,3))
```

**Answer:** 0.6

Sum of C/T. Verified by running it.

**Q176.** Utilization is below the Liu-Layland bound for the task set. What does that imply under rate-monotonic scheduling?

**Answer:** Schedulability is guaranteed (the bound is sufficient, not necessary)

Otherwise use response-time analysis.

**Q177.** What does PendSV do on Cortex-M FreeRTOS?

**Answer:** Performs the context switch at the lowest interrupt priority

Deferred switch.

**Q178.** How do you prevent one faulty task from corrupting another's memory?

**Answer:** MPU-enabled FreeRTOS with restricted tasks

Hardware isolation.

**Q179.** Trace shows a high-priority task misses deadlines when a low-priority task runs. Likely cause?

**Answer:** Priority inversion or long critical sections holding interrupts/locks

Use inheritance and shorten critical sections.

## Practical coding questions

### Rate-monotonic schedulability

Implement utilization, the Liu-Layland bound and fixed-priority response-time analysis (R = C + sum of ceil(R/Tj)*Cj over higher-priority tasks, iterated to a fixed point; deadline = period) and apply them to two task sets.

Starter code:

```
import math

def utilization(tasks):
    # sum of C/T
    return 0.0

def ll_bound(n):
    # Liu & Layland rate-monotonic bound n*(2^(1/n)-1)
    return 0.0

def response_times(tasks):
    # tasks = [(name, C, T)], deadline = period, priority by shorter period (rate monotonic).
    # worst-case response time: R = C + sum(ceil(R/Tj)*Cj over higher-priority tasks), iterate to a fixed point;
    # stop if R exceeds the period. Return {name: (R, meets_deadline)}.
    return {}

SET_A = [("sensor", 1, 4), ("control", 2, 8), ("logger", 1, 10)]
SET_B = [("fast", 2, 5), ("mid", 3, 7), ("slow", 4, 20)]
for label, ts in [("A", SET_A), ("B", SET_B)]:
    u = utilization(ts)
    print(f"set {label}: U={u:.3f} bound={ll_bound(len(ts)):.3f} -> {'guaranteed' if u <= ll_bound(len(ts)) else 'test needed'}")
    for name, (r, ok) in response_times(ts).items():
        print(f"  {name}: R={r} {'meets' if ok else 'MISSES'} deadline")
```

Expected output:

```text
set A: U=0.600 bound=0.780 -> guaranteed
  sensor: R=1 meets deadline
  control: R=3 meets deadline
  logger: R=4 meets deadline
set B: U=1.029 bound=0.780 -> test needed
  fast: R=2 meets deadline
  mid: R=5 meets deadline
  slow: R=21 MISSES deadline
```

??? success "Show a solution"

    ```
    import math

    def utilization(tasks):
        return sum(c / t for _, c, t in tasks)

    def ll_bound(n):
        return n * (2 ** (1 / n) - 1)

    def response_times(tasks):
        """Fixed-priority response-time analysis. tasks = [(name, C, T)], deadline = period.
        Higher priority = shorter period (rate-monotonic)."""
        tasks = sorted(tasks, key=lambda x: x[2])
        out = {}
        for i, (name, c, t) in enumerate(tasks):
            r = c
            while True:
                interference = sum(math.ceil(r / tj) * cj for _, cj, tj in tasks[:i])
                nxt = c + interference
                if nxt == r:
                    break
                r = nxt
                if r > t:
                    break
            out[name] = (r, r <= t)
        return out

    SET_A = [("sensor", 1, 4), ("control", 2, 8), ("logger", 1, 10)]
    SET_B = [("fast", 2, 5), ("mid", 3, 7), ("slow", 4, 20)]
    for label, ts in [("A", SET_A), ("B", SET_B)]:
        u = utilization(ts)
        print(f"set {label}: U={u:.3f} bound={ll_bound(len(ts)):.3f} -> {'guaranteed' if u <= ll_bound(len(ts)) else 'test needed'}")
        for name, (r, ok) in response_times(ts).items():
            print(f"  {name}: R={r} {'meets' if ok else 'MISSES'} deadline")
    ```

    Set A passes the sufficient utilization bound, so it's guaranteed. Set B exceeds the bound, so the exact response-time test is needed, and it shows which task misses its deadline. Verified by running the solution.

### A fixed-priority preemptive scheduler

Simulate FreeRTOS-style scheduling one tick at a time: each tick run the highest-priority released task with remaining work. Show the timeline for a low-priority task preempted by medium and high-priority tasks.

Starter code:

```
def simulate(tasks, horizon):
    # tasks = [(name, priority, release_tick, work_ticks)], higher number = higher priority.
    # each tick run the highest-priority released task that still has work (preemptive);
    # return (timeline string with one letter per tick and '.' when idle, {name: finish_tick})
    return "", {}

# Three tasks: L (low) starts first, M (medium) and H (high) preempt it.
tasks = [("L", 1, 0, 6), ("M", 2, 2, 3), ("H", 3, 4, 2)]
line, finish = simulate(tasks, 14)
print(line)
print(finish)
# A priority-inversion-free run versus the same set where H must wait for L's critical section:
tasks2 = [("L", 1, 0, 3), ("H", 3, 1, 2)]
print(simulate(tasks2, 8))
```

Expected output:

```text
LLMMHHMLLLL...
{'H': 6, 'M': 7, 'L': 11}
('LHHLL...', {'H': 3, 'L': 5})
```

??? success "Show a solution"

    ```
    def simulate(tasks, horizon):
        """Fixed-priority preemptive scheduler, 1 tick per step.
        tasks = [(name, priority, release_tick, work_ticks)], higher number = higher priority."""
        remaining = {n: w for n, _, _, w in tasks}
        finish = {}
        timeline = []
        for t in range(horizon):
            ready = [(p, n) for n, p, r, _ in tasks if r <= t and remaining[n] > 0]
            if not ready:
                timeline.append(".")
                continue
            _, running = max(ready)
            timeline.append(running)
            remaining[running] -= 1
            if remaining[running] == 0:
                finish[running] = t + 1
        return "".join(timeline), finish

    # Three tasks: L (low) starts first, M (medium) and H (high) preempt it.
    tasks = [("L", 1, 0, 6), ("M", 2, 2, 3), ("H", 3, 4, 2)]
    line, finish = simulate(tasks, 14)
    print(line)
    print(finish)
    # A priority-inversion-free run versus the same set where H must wait for L's critical section:
    tasks2 = [("L", 1, 0, 3), ("H", 3, 1, 2)]
    print(simulate(tasks2, 8))
    ```

    A newly ready higher-priority task preempts immediately, which is why L finishes last and H runs as soon as it's released. This models the rule 'run the highest-priority Ready task', not FreeRTOS internals. Verified by running the solution.

---

More free practice: the [FreeRTOS Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
