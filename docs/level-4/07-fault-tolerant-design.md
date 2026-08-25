# Fault-Tolerant & High-Availability Design

This module is architectural/design-pattern content covering how production
RTOS systems detect and recover from faults — building directly on Level 4
Module 3's fault-domain decisions and Level 2's watchdog/supervision
patterns. No new hardware-specific claims are made here beyond what earlier
modules already covered, so no separate verification caveat is needed beyond
what those modules already stated.

## Fault detection: you can't recover from what you don't detect

Every fault-tolerance mechanism below assumes a detection layer exists
first. The standard toolkit, most of it already introduced earlier in this
course and now assembled into a deliberate strategy:

- **Hardware watchdog timer**, fed only by evidence that the *whole system*
  is healthy, not just that one task happens to be running — a common
  mistake is feeding the watchdog from a low-priority housekeeping task that
  keeps running even while a higher-priority task is deadlocked; the
  watchdog then fails to catch exactly the fault it exists to catch.
- **Per-task supervision** (Level 2's watchdog/supervision module): a
  supervisor task tracking a "last seen alive" timestamp or counter per
  monitored task, distinct from the hardware watchdog, able to detect a
  *specific* task's hang without necessarily resetting the whole device.
- **MPU-based fault isolation** (Level 3 Module 4): converts memory-access
  bugs into immediate, precisely-located faults rather than silent
  corruption discovered much later, which is a detection mechanism as much
  as a containment one.
- **Stack overflow detection** (`configCHECK_FOR_STACK_OVERFLOW`, Level 1)
  and heap-exhaustion detection (`vApplicationMallocFailedHook`) — resource
  exhaustion is one of the most common real-world embedded fault classes and
  needs its own explicit detection, not just crash-after-the-fact discovery.

## Recovery strategies, from lightest to heaviest

| Strategy | Recovers from | Cost |
|---|---|---|
| Task restart (`vTaskDelete` + recreate) | A single task's hang/crash, if state can be safely reset | Cheapest; requires the task's state to be re-initializable |
| Subsystem restart | A group of related tasks/resources in a bad state together | Moderate; needs a defined "subsystem" boundary (Level 4 Module 3) |
| Watchdog-triggered full reset | Systemic corruption, deadlock, anything not cleanly containable | Most disruptive; loses all in-progress work, but reliably recovers |
| Graceful degradation | Non-critical subsystem failure (e.g. network down) | Application continues in a reduced-functionality mode rather than failing entirely |

The right strategy for a given fault is a **design decision made per
subsystem in advance** (exactly Level 4 Module 3's fault-domain question),
not something to improvise when a specific bug is discovered in the field.
A system with no defined recovery tier below "full reset" pays the cost of
losing all in-progress work for every fault, even ones that could have been
contained cheaply.

## Redundancy patterns for high availability

- **N-modular redundancy / voting**: multiple independent computations of
  the same result (potentially on separate hardware, or at minimum separate
  tasks with independently-verified logic), compared before acting — used
  where the cost of acting on a wrong result is severe enough to justify
  the redundant computation cost. This connects to Level 4 Module 4's ASIL
  decomposition: redundant, independent elements are exactly what makes an
  ASIL-decomposition independence argument credible.
- **Heartbeat/liveness protocols between subsystems**, not just from
  application to a single central supervisor — in a system with genuinely
  independent, possibly separately-processored subsystems (Level 4 Module
  8's AMP architectures), each side needs to detect the other going silent,
  not just be detected by one master supervisor.
- **State checkpointing to non-volatile storage**, so a full reset can
  resume from a recent known-good state rather than a cold start — valuable
  when cold-start time itself is a real-time-relevant cost (a device that
  must resume control within a bounded time after any reset, tying directly
  back to Level 4 Module 2's schedulability guarantees, which typically
  assume the system is already running, not still booting).

## Defensive design at the task level

Fault tolerance isn't only a system-level architecture question — it shows
up in ordinary task code decisions covered throughout this course:

```c
void controlLoopTask(void *pv) {
    for (;;) {
        SensorData_t data;
        BaseType_t ok = xQueueReceive(sensorQueue, &data, pdMS_TO_TICKS(50));
        if (ok != pdTRUE) {
            // Sensor feed has gone silent — this is a fault, not a normal
            // "nothing to do" case. Enter a defined safe state, don't just
            // reuse stale data or skip the iteration silently.
            enterSafeState();
            continue;
        }
        if (!isPlausible(&data)) {
            // Range/plausibility check — garbage input is a fault class
            // distinct from "no input," and needs its own handling.
            reportFault(FAULT_IMPLAUSIBLE_SENSOR_DATA);
            continue;
        }
        runControlAlgorithm(&data);
    }
}
```

The discipline here: every blocking call with a timeout (Level 1's queue
APIs, used throughout this course) has an implicit fault-detection
opportunity at its timeout branch — treating a timeout as "just retry
silently" throws away a detection signal that a supervisor or safe-state
mechanism could otherwise use.

## Traps

- **Feeding a hardware watchdog from a task that doesn't actually confirm
  system health** — as covered above, this is the single most common way a
  watchdog fails to do its job, converting a safety mechanism into a false
  sense of security.
- **No defined recovery tier below full reset.** Every fault escalating to
  the most disruptive recovery wastes the investment already made in
  MPU-based containment (Level 3 Module 4) and Level 4 Module 3's
  fault-domain design — if containment exists, use it.
- **Silent retry on timeout without fault accounting.** A control loop that
  quietly reuses stale sensor data on every timeout, with no counter or
  escalation logic, can mask a real, ongoing sensor failure indefinitely.
- **Redundancy without independence.** N-modular redundancy only helps
  against faults the redundant elements don't share — redundant software
  running identical, buggy logic on identical hardware doesn't protect
  against a systematic bug in that logic; true independence (different
  implementations, or at minimum different hardware/timing) is needed for
  the redundancy to add real fault coverage.
- **Not testing the failure paths themselves.** As with Level 4 Module 6's
  OTA power-loss testing, a fault-tolerance design that's never had its
  recovery paths actually exercised (deliberately induced task crashes,
  deliberately corrupted sensor data, deliberately silenced a subsystem) has
  not actually verified the property it claims to provide.

## Cheat sheet

| Fault class | Detection mechanism | Typical recovery |
|---|---|---|
| Single task hang | Per-task supervisor heartbeat (Level 2) | Task restart |
| Memory corruption / wild pointer | MPU fault (Level 3 Module 4) | Contained restart or reset, per Level 4 Module 3's policy |
| Stack/heap exhaustion | `configCHECK_FOR_STACK_OVERFLOW`, malloc-failed hook | Usually reset — state is unreliable at this point |
| Systemic deadlock/corruption | Hardware watchdog (correctly fed) | Full reset |
| Sensor/input feed silence | Queue receive timeout | Safe-state entry, fault counter, escalate if persistent |
| Non-critical subsystem failure | Subsystem-specific health check | Graceful degradation, not full failure |

## Exercise

1. Take a task design from earlier in this course (Level 2 Module 8's
   three-stage acquire/filter/report pipeline is a good candidate) and add
   an explicit fault-detection and recovery tier to each stage: what happens
   if `acquireTask` stops producing, what happens if `filterTask` hangs, and
   what the appropriate recovery action is for each — contained restart or
   full reset, and justify the choice.
2. Design a heartbeat protocol between two independent subsystems (not a
   single central supervisor watching both) and specify exactly what each
   side does when it stops hearing from the other, including how to avoid
   both sides resetting simultaneously in a way that loses the ability to
   diagnose which one actually failed first.
3. Write a short design note arguing for or against N-modular redundancy for
   a specific hypothetical safety-critical control loop, addressing
   explicitly whether your proposed redundant elements would actually be
   independent enough to catch a systematic software bug, not just a
   random hardware fault.
