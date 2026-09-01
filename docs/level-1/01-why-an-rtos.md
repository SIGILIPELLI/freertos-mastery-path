# 01 · Why an RTOS?

Almost every microcontroller program starts life as a **superloop**: one
`loop()` that reads inputs, does work, and writes outputs, forever. It's a
fine architecture — until the firmware has to do several timing-sensitive
things at once. This module explains where the superloop breaks down, what
"real-time" actually means (hint: it's not "fast"), how an RTOS scheduler
solves the problem, and — just as important — when you *shouldn't* reach for
one. It ends with how to set up a free, no-hardware environment for the whole
level.

## The bare-metal superloop and where it cracks

Here is a perfectly reasonable bare-metal sketch: blink an LED at 2 Hz and
print a sensor reading every second.

```cpp
void setup() {
  Serial.begin(115200);
  pinMode(2, OUTPUT);
}

void loop() {
  digitalWrite(2, HIGH);
  delay(250);
  digitalWrite(2, LOW);
  delay(250);

  Serial.println(analogRead(34));   // runs every ~500 ms, not every 1000 ms
  delay(500);                       // ...so patch it with another delay
}
```

The timings are already entangled: change the blink rate and the print rate
changes too. Experienced Arduino programmers fix this with `millis()`-based
cooperative scheduling (covered in depth in the
[Embedded Systems Mastery Path](https://sigilipelli.github.io/embedded-mastery-path/)),
which works well — but it has structural limits:

- **One slow job stalls everything.** If parsing a command or writing to
  flash takes 80 ms, every other "task" in the loop is late by 80 ms. The
  loop is only as responsive as its slowest step.
- **No priorities.** The loop gives every job equal standing. You can't say
  "the motor-control update at 1 kHz *always* wins over the logging."
- **State machines everywhere.** Each job that must not block gets rewritten
  as a state machine with its own timestamps and flags. At five jobs it's
  manageable; at fifteen it's a maintenance nightmare.
- **Worst-case timing is a guess.** The time between two runs of any job is
  the sum of *everything else* in the loop — which changes with every code
  edit.

## What "real-time" actually means

A real-time system is **not** a fast system. Real-time means
**deterministic**: the system *guarantees* it responds within a known
deadline, every time — and you can reason about that bound.

- A 16 MHz Arduino toggling a pin within 10 µs of an interrupt, *always*, is
  real-time.
- A 3 GHz server that usually answers in 1 ms but occasionally takes 2 s
  (garbage collection, page fault...) is *fast but not real-time*.

Two flavors you'll hear about:

- **Hard real-time** — missing a deadline is a system failure (airbag
  trigger, motor commutation, pacemaker pulse).
- **Soft real-time** — missing a deadline degrades quality but isn't fatal
  (dropped audio frame, delayed screen refresh).

An RTOS is an operating system whose scheduler is built for determinism: the
highest-priority ready task runs, immediately, with a small and bounded
switching cost — rather than "whenever the loop gets around to it."

## Preemptive vs. cooperative scheduling

- **Cooperative**: a task runs until it *voluntarily* gives up the CPU. The
  `millis()` superloop is cooperative scheduling by hand. Cheap and
  predictable — but one misbehaving task freezes the system, and response
  time depends on every task's good manners.
- **Preemptive**: the scheduler can **interrupt (preempt) a running task the
  moment a higher-priority task becomes ready**. The slow flash-write no
  longer delays the motor update — the motor task simply preempts it, runs,
  and the flash write resumes where it left off.

FreeRTOS is preemptive by default (cooperative mode exists as a config
option). Preemption is what buys you the core RTOS promise: **the latency of
your critical task no longer depends on the rest of your code.**

```text
Superloop (cooperative):          RTOS (preemptive):

|--blink--|--LOG 80ms--|--motor--   |--motor--| (motor READY → runs NOW)
                 ^                       preempts
   motor is 80 ms late             |--log......log--|  log resumes after
```

## What FreeRTOS gives you

FreeRTOS is the most widely deployed RTOS kernel in the world — billions of
devices, from sensors to spacecraft. The relevant facts:

- **Tiny**: the kernel is roughly 6-12 kB of flash and a few kB of RAM in a
  typical build — three C files plus a port layer.
- **Free and permissive**: MIT-licensed, no royalties, commercial use
  allowed. (A safety-certified derivative, SAFERTOS, exists for regulated
  industries — Level 4 territory.)
- **Ubiquitous**: it's the kernel inside the ESP32 Arduino core, ESP-IDF,
  AWS IoT device SDKs, and countless vendor SDKs. Learn it once, use it
  everywhere.
- **A small, learnable API**: tasks, queues, semaphores, mutexes, timers,
  notifications, event groups — that's the entire Level 1 syllabus, and it's
  90 % of what production firmware uses.

## When NOT to use an RTOS

An RTOS is a tool, not a badge of professionalism. Skip it when:

- **The superloop is genuinely enough.** One or two activities, simple
  timing — `millis()` scheduling is simpler to write, test, and debug.
- **RAM is critically scarce.** Every task needs its own stack (hundreds of
  bytes to kilobytes each). On a 2 kB-RAM part, tasks are a luxury.
- **Your timing is purely interrupt-driven.** If ISRs plus a trivial
  background loop already meet the deadlines, a scheduler adds nothing.
- **The team doesn't know it yet and the deadline is tomorrow.** Concurrency
  bugs (races, deadlocks, starvation) are a real tax; this course exists to
  make you pay it once, cheaply, in a simulator.

Rule of thumb: when you find yourself building your *third* hand-rolled state
machine with its own timing bookkeeping, or you need priorities, it's RTOS
time.

## Your lab for this level: ESP32 + Wokwi (no hardware)

The ESP32 Arduino core **runs on FreeRTOS out of the box** — `setup()` and
`loop()` themselves execute inside a FreeRTOS task, and the entire FreeRTOS
API is available with no extra includes or installs. That makes it the
easiest way in the world to *run* real RTOS code:

1. Open [wokwi.com](https://wokwi.com/) in a browser.
2. Create a new **ESP32** project.
3. Paste any sketch from this course into `sketch.ino` and press the play
   button. Serial output appears in the built-in monitor.

Everything works identically on a real ESP32 board via the Arduino IDE.

!!! note "Pure-C alternatives (no Arduino)"
    Prefer studying FreeRTOS as plain C on your PC? Two good options, both
    covered further in Level 2:

    - The official **[POSIX/Linux simulation port](https://www.freertos.org/FreeRTOS-simulator-for-Linux.html)**
      compiles the real kernel into a native Linux/macOS program — full API,
      gdb debugging, no hardware.
    - **QEMU** emulates real boards (e.g. `qemu-system-arm -machine
      mps2-an385` runs official FreeRTOS Cortex-M demos).

    The API you learn here is identical in all three environments.

## Cheat sheet

| Concept | Meaning |
|---|---|
| Superloop | One big `loop()` doing everything; response time = sum of all jobs |
| Real-time | *Deterministic* response within a deadline — not "fast" |
| Hard / soft real-time | Missed deadline = failure / = degraded quality |
| Cooperative scheduling | Tasks yield voluntarily; one bad task stalls all |
| Preemptive scheduling | Scheduler interrupts lower-priority work the instant higher-priority work is ready |
| FreeRTOS | MIT-licensed, ~10 kB kernel; the world's most deployed RTOS; built into the ESP32 Arduino core |
| When to skip an RTOS | Simple timing, tiny RAM, pure ISR designs, or no team experience + no time |

## Exercise

1. Take the entangled blink-plus-print sketch from the top of this page and
   list every timing bug it has (there are at least three: blink duty cycle,
   print period, and serial-command latency if you added one).
2. Sketch (on paper) how you'd fix it with `millis()` — count how many state
   variables you need.
3. Write down two activities from a device you own (e.g. a smart kettle:
   heater control + display update + button handling) and classify each as
   hard or soft real-time, with the deadline you'd assign.
4. Set up the Wokwi ESP32 project described above and run the default blink
   sketch — you'll build on this project in every following module.
