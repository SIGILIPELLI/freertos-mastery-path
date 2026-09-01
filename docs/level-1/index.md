# Level 1 · Entry <span class="level-badge">Foundations</span>

Goal: understand what a real-time operating system is for, and get genuinely
comfortable with the FreeRTOS core — creating tasks, letting the scheduler
run them, and moving data and signals between them safely with queues,
semaphores, mutexes, timers, notifications, and event groups — finishing with
a complete multi-task smart sensor device.

**You do not need to own any hardware for this level.** Every example targets
the **ESP32 with the Arduino framework, which ships FreeRTOS built in**, and
runs in the free [Wokwi online simulator](https://wokwi.com/) — a
browser-based ESP32 simulator with virtual LEDs, buttons, and sensors.
Everything also works unchanged on a real ESP32 board, and the concepts (and
almost all the code) transfer directly to FreeRTOS on any other
microcontroller. If you'd rather study pure C on a PC, the
[FreeRTOS POSIX/Linux simulation port](https://www.freertos.org/FreeRTOS-simulator-for-Linux.html)
and QEMU are covered in Module 1 as alternatives.

New to the ESP32 or Arduino itself? Skim Level 1 of the sibling
[Embedded Systems Mastery Path](https://sigilipelli.github.io/embedded-mastery-path/)
first — it covers boards, GPIO, and serial from scratch, so this site can
stay focused on the RTOS layer.

## Modules

1. [Why an RTOS?](01-why-an-rtos.md)
2. [Your First Tasks](02-first-tasks.md)
3. [The Scheduler](03-the-scheduler.md)
4. [Queues](04-queues.md)
5. [Semaphores & Mutexes](05-semaphores-mutexes.md)
6. [Software Timers](06-software-timers.md)
7. [Task Notifications & Event Groups](07-notifications-event-groups.md)
8. [Interrupts & the RTOS](08-interrupts.md)
9. [Memory & Debugging](09-memory-debugging.md)
10. [Capstone — Smart Sensor Device](10-capstone-project.md)

By the end of this level you'll be able to split firmware into concurrent
tasks with sensible priorities, pass data between them with queues, protect
shared state with mutexes, signal from interrupts safely, size task stacks,
and combine all of it into a working multi-task device — entirely in the
browser if you want.
