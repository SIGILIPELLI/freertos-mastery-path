# 08 · Interrupts & the RTOS

Interrupts are how hardware talks to your firmware: a pin changes, a timer
fires, a byte arrives, and the CPU drops everything to run your ISR. But an
ISR is *not a task* — it runs outside the scheduler's world, and calling
normal FreeRTOS functions from it will corrupt the kernel. This module
covers the ISR-safe `...FromISR` API variants, the golden pattern of
**deferring work from ISRs to tasks**, and `portYIELD_FROM_ISR` — the one
line that makes deferred handling feel instantaneous.

## Why ISRs are special

An ISR can fire *between any two instructions* — including halfway through a
kernel operation. Regular API calls like `xQueueSend` may block, and may
trigger an immediate context switch; neither is legal mid-interrupt. So
FreeRTOS provides parallel **interrupt-safe versions** of the communication
APIs:

| Task context | ISR context |
|---|---|
| `xQueueSend(q, &v, timeout)` | `xQueueSendFromISR(q, &v, &woken)` |
| `xSemaphoreGive(s)` | `xSemaphoreGiveFromISR(s, &woken)` |
| `xTaskNotifyGive(h)` | `vTaskNotifyGiveFromISR(h, &woken)` |
| `xEventGroupSetBits(eg, b)` | `xEventGroupSetBitsFromISR(eg, b, &woken)` |
| `xTimerStart(t, wait)` | `xTimerStartFromISR(t, &woken)` |

Two systematic differences: the `FromISR` versions **never block** (no
timeout parameter — an ISR cannot sleep), and they take a
`BaseType_t *pxHigherPriorityTaskWoken` out-parameter explained below.
Calling the non-ISR version from an ISR is a real crash on ESP32
(`assert failed` in the queue code) — not a style nit.

## The pattern: defer work to a task

The entire discipline of RTOS interrupt handling is one sentence: **the ISR
records that the event happened and wakes a task; the task does the work.**

Why keep ISRs minimal?

- While your ISR runs, same-or-lower-priority interrupts wait — long ISRs
  create system-wide jitter (serial bytes dropped, tick delayed).
- ISR context is fragile: tiny stack, no blocking, on ESP32 code should be
  in IRAM (`IRAM_ATTR`) and avoid `Serial`/`printf`/heap entirely.
- Work in a task is schedulable, measurable, and debuggable; work in an ISR
  is invisible to every tool you'll meet in Module 9.

Button → notification → handler task, the canonical form:

```cpp
const uint8_t BUTTON_PIN = 27;
const uint8_t LED_PIN    = 25;

TaskHandle_t buttonTaskHandle;

void IRAM_ATTR buttonISR() {                       // keep it tiny
  BaseType_t higherPrioWoken = pdFALSE;
  vTaskNotifyGiveFromISR(buttonTaskHandle, &higherPrioWoken);
  portYIELD_FROM_ISR(higherPrioWoken);             // context-switch NOW if needed
}

void buttonTask(void *pv) {
  uint32_t presses = 0;
  for (;;) {
    ulTaskNotifyTake(pdTRUE, portMAX_DELAY);       // sleep until the ISR fires
    presses++;
    digitalWrite(LED_PIN, !digitalRead(LED_PIN));  // the "work"
    Serial.printf("[%lu ms] press #%lu\n", millis(), presses);
    // crude debounce: ignore bounces for 200 ms, then drain stale notifications
    vTaskDelay(pdMS_TO_TICKS(200));
    ulTaskNotifyTake(pdTRUE, 0);                   // clear any bounce notifications
  }
}

void setup() {
  Serial.begin(115200);
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  xTaskCreate(buttonTask, "button", 2048, NULL, 4, &buttonTaskHandle);  // high prio
  attachInterrupt(digitalPinToInterrupt(BUTTON_PIN), buttonISR, FALLING);
}

void loop() { vTaskDelay(portMAX_DELAY); }
```

In Wokwi, add a pushbutton between GPIO 27 and GND. Every press wakes
`buttonTask` — which, at priority 4, typically starts running within
microseconds of the ISR returning. The ISR itself is three lines and
touches nothing dangerous. (Attach the interrupt *after* creating the task —
otherwise a press could notify a NULL handle.)

## `portYIELD_FROM_ISR`: why deferred still feels instant

Here's the subtlety that separates working code from *correct* code. The
scheduler normally re-evaluates "who should run" at the tick or when a task
makes an API call. Your ISR just made a high-priority task Ready — but the
interrupted code might be the idle task, and without help, the handler
wouldn't run until the interrupted task hits the next scheduling point —
adding up to one full tick (1 ms) of latency, sometimes more.

That's what the `woken` flag is for:

1. Initialize `BaseType_t higherPrioWoken = pdFALSE;`
2. Pass `&higherPrioWoken` to every `FromISR` call — the kernel sets it to
   `pdTRUE` if the call unblocked a task with higher priority than the one
   the interrupt preempted.
3. End the ISR with `portYIELD_FROM_ISR(higherPrioWoken);` — if the flag is
   set, the context switch happens *as the ISR returns*, so the CPU goes
   ISR → handler task directly, never resuming the interrupted task first.

Skip step 3 and everything still "works" — just with sporadic extra latency,
the kind of bug that only shows on a scope. Make the three-step pattern
muscle memory.

## Queues from ISRs — when events carry data

When the interrupt produces data (a byte, a timestamp, an encoder delta),
use `xQueueSendFromISR`. Periodic hardware-timer example — the ESP32's
`esp_timer` calls its callback from ISR-like context, so the FromISR rules
apply:

```cpp
QueueHandle_t sampleQueue;

void IRAM_ATTR onSampleTimer(void *arg) {          // esp_timer callback
  BaseType_t woken = pdFALSE;
  uint32_t stamp = (uint32_t)esp_timer_get_time(); // µs since boot — ISR-safe
  xQueueSendFromISR(sampleQueue, &stamp, &woken);  // never blocks; drops if full
  portYIELD_FROM_ISR(woken);
}

void processTask(void *pv) {
  uint32_t stamp;
  for (;;) {
    xQueueReceive(sampleQueue, &stamp, portMAX_DELAY);
    Serial.printf("sample at %lu us\n", stamp);
  }
}

void setup() {
  Serial.begin(115200);
  sampleQueue = xQueueCreate(16, sizeof(uint32_t));
  xTaskCreate(processTask, "proc", 2048, NULL, 3, NULL);

  const esp_timer_create_args_t args = {
    .callback = &onSampleTimer, .arg = NULL,
    .dispatch_method = ESP_TIMER_TASK,  // use ESP_TIMER_ISR only if required
    .name = "sampler"
  };
  esp_timer_handle_t t;
  esp_timer_create(&args, &t);
  esp_timer_start_periodic(t, 250000);             // every 250 ms (µs units)
}

void loop() { vTaskDelay(portMAX_DELAY); }
```

If the queue is full, `xQueueSendFromISR` returns `errQUEUE_FULL`
immediately — decide in the ISR whether dropping is acceptable (usually:
count the drops in a variable a task reports later; never print from the
ISR).

Final rules of thumb: **no mutexes in ISRs** (priority inheritance is
meaningless there — use a binary semaphore or notification), keep ISR
handlers in `IRAM_ATTR` on ESP32, no `Serial`/`malloc`/blocking, and
measure ISR length in microseconds, not milliseconds.

## Cheat sheet

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

## Exercise

1. Build the button demo, then press the button rapidly: confirm the
   debounce drain works by printing the press count. Remove
   `portYIELD_FROM_ISR` and re-test — can you observe any difference in
   this small sketch? Explain why the bug it would cause is invisible here
   but real (what else would the CPU have to be busy doing?).
2. Change the button ISR to `xQueueSendFromISR` a struct
   `{millis(), pressCount}` into a 4-deep queue instead of a notification.
   Hold the button down with Wokwi's autorepeat or click furiously — watch
   `errQUEUE_FULL` drops happen by counting them in a `volatile uint32_t
   dropCount` that `loop()` prints every 2 s.
3. In comments: list the three things the ISR in exercise 2 is still allowed
   to do, and three things it must never do — with the failure mode each
   violation causes.
