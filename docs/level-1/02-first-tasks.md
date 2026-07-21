# 02 · Your First Tasks

A **task** is FreeRTOS's unit of concurrency: an ordinary C function with its
own stack and priority, running an infinite loop, scheduled independently of
every other task. In this module you create your first tasks with
`xTaskCreate`, blink two LEDs at unrelated rates with none of the superloop's
timing entanglement, learn the difference between `vTaskDelay` and
`vTaskDelayUntil`, and meet the four states every task lives in.

## Anatomy of a task function

A task function has a fixed shape:

```cpp
void myTask(void *pvParameters) {   // must take void*, return void
  // one-time setup for this task can go here
  for (;;) {                        // tasks run forever...
    // do work
    vTaskDelay(pdMS_TO_TICKS(100)); // ...but must block, not busy-wait
  }
  // if a task ever must exit, it deletes itself:
  // vTaskDelete(NULL);  — a task function must NEVER simply return
}
```

Two hard rules:

1. **Never return** from a task function. If a task is finished, call
   `vTaskDelete(NULL)`. Returning corrupts the scheduler (the ESP32 port
   aborts with an error).
2. **Every loop iteration must block** somewhere (`vTaskDelay`, queue
   receive, semaphore take...). A task that never blocks starves everything
   at lower priority — more on that in the next module.

## `xTaskCreate`, parameter by parameter

```cpp
BaseType_t xTaskCreate(
    TaskFunction_t  pvTaskCode,     // the task function
    const char     *pcName,         // debug name, e.g. "blink1"
    uint32_t        usStackDepth,   // stack size (BYTES on ESP32 — see note)
    void           *pvParameters,   // void* handed to the function
    UBaseType_t     uxPriority,     // 0 = lowest; higher number = higher priority
    TaskHandle_t   *pxCreatedTask   // out: handle, or NULL if not needed
);
```

- **`pcName`** — purely for humans: shows up in debuggers, crash dumps, and
  `vTaskList()` output. Keep it short (default limit 16 chars).
- **`usStackDepth`** — how much stack this task gets, *forever*. Too small →
  stack overflow and a crash; too large → wasted RAM. Module 9 shows how to
  measure the right size. Starting points on ESP32: 1024-2048 for a simple
  loop, 4096 if the task uses `Serial`/`String`/`printf`.
- **`pvParameters`** — an arbitrary pointer passed to the task. Lets one
  function serve many tasks (see below). The pointed-to data must outlive
  the task — pass a pointer to a `static`/global, never to a local that dies.
- **`uxPriority`** — 0 (lowest, shared with the idle task) up to
  `configMAX_PRIORITIES - 1` (25 on ESP32). The Arduino `loop()` task runs at
  priority 1.
- **`pxCreatedTask`** — receives a `TaskHandle_t` you can use later to
  suspend, resume, notify, or delete the task. Pass `NULL` if you'll never
  need it.
- **Return value** — `pdPASS` on success, or `errCOULD_NOT_ALLOCATE_REQUIRED_MEMORY`
  if there wasn't enough heap for the stack + task control block. Check it.

!!! warning "ESP32 quirk: stack depth is in **bytes**"
    In vanilla FreeRTOS `usStackDepth` is in *words* (×4 bytes on 32-bit
    MCUs). The ESP32 port changed it to **bytes**. `xTaskCreate(..., 2048,
    ...)` means 2048 bytes on ESP32 but 8192 bytes on, say, an STM32. Keep
    this in mind when reading generic FreeRTOS examples.

## Two blinking LEDs, zero timing entanglement

Wire two LEDs (with resistors) to GPIO 25 and GPIO 26 in Wokwi, or just
watch the serial log.

```cpp
// Two independent blinkers — ESP32 Arduino (FreeRTOS built in, no includes needed)

const uint8_t LED1 = 25;
const uint8_t LED2 = 26;

void blink1(void *pvParameters) {
  pinMode(LED1, OUTPUT);
  for (;;) {
    digitalWrite(LED1, !digitalRead(LED1));
    vTaskDelay(pdMS_TO_TICKS(250));          // 2 Hz
  }
}

void blink2(void *pvParameters) {
  pinMode(LED2, OUTPUT);
  for (;;) {
    digitalWrite(LED2, !digitalRead(LED2));
    vTaskDelay(pdMS_TO_TICKS(700));          // ~0.7 Hz — totally unrelated rate
  }
}

void setup() {
  Serial.begin(115200);

  xTaskCreate(blink1, "blink1", 1024, NULL, 1, NULL);
  xTaskCreate(blink2, "blink2", 1024, NULL, 1, NULL);

  Serial.println("Tasks created — scheduler was already running.");
}

void loop() {
  // The loop() task keeps running too — it's just another task (priority 1).
  vTaskDelay(pdMS_TO_TICKS(2000));
  Serial.printf("uptime: %lu s\n", millis() / 1000);
}
```

Change one delay and the other blinker is completely unaffected — that's the
entire point. On a plain Arduino you'd have needed two `millis()` state
machines; here each task is three honest lines.

!!! note "No `vTaskStartScheduler()`?"
    Generic FreeRTOS programs create tasks in `main()` and then call
    `vTaskStartScheduler()`. On ESP32 Arduino the scheduler is **already
    running** before `setup()` is called (setup/loop live inside a task named
    `loopTask`), so you just create tasks and they start immediately.

### One function, many tasks — using `pvParameters`

```cpp
struct BlinkSpec {
  uint8_t pin;
  uint32_t periodMs;
};

// static: must outlive the tasks that receive pointers to them
static BlinkSpec spec1 = {25, 250};
static BlinkSpec spec2 = {26, 700};

void blinker(void *pvParameters) {
  BlinkSpec *spec = (BlinkSpec *)pvParameters;   // cast back from void*
  pinMode(spec->pin, OUTPUT);
  for (;;) {
    digitalWrite(spec->pin, !digitalRead(spec->pin));
    vTaskDelay(pdMS_TO_TICKS(spec->periodMs));
  }
}

void setup() {
  xTaskCreate(blinker, "blink25", 1024, &spec1, 1, NULL);
  xTaskCreate(blinker, "blink26", 1024, &spec2, 1, NULL);
}

void loop() { vTaskDelay(portMAX_DELAY); }       // nothing to do here
```

## `vTaskDelay` vs `vTaskDelayUntil`

`vTaskDelay(n)` blocks for *n ticks from now*. If the loop body itself takes
time, the period drifts:

```text
vTaskDelay(100 ms), body takes 7 ms:
run(7) + delay(100) + run(7) + delay(100)... → actual period 107 ms, drifting
```

`vTaskDelayUntil` blocks until an *absolute* tick count, giving a fixed
period with no drift — the right tool for sampling sensors, control loops,
or anything that must run at a precise rate:

```cpp
void sampler(void *pvParameters) {
  TickType_t lastWake = xTaskGetTickCount();     // initialize ONCE
  const TickType_t period = pdMS_TO_TICKS(100);  // exactly 10 Hz
  for (;;) {
    vTaskDelayUntil(&lastWake, period);          // wakes at lastWake + period
    int raw = analogRead(34);
    Serial.printf("[%lu ms] sample=%d\n", millis(), raw);
  }
}
```

`lastWake` is updated by the call itself, so the wake-up times form an exact
grid: t₀+100, t₀+200, t₀+300... regardless of how long the body takes (as
long as it takes less than one period).

Also useful: `vTaskDelay(pdMS_TO_TICKS(x))` converts milliseconds to ticks —
never hardcode tick counts, because the tick rate is configurable (1000 Hz on
ESP32 Arduino, often 100 Hz elsewhere).

## The four task states

Every task is always in exactly one state:

```text
                 ┌───────────┐
     scheduler   │  Running  │  the one task per core actually executing
    picks it ───►│           │───┐ preempted / time slice over
                 └───────────┘   │
                     ▲           ▼
   event occurs ┌───────────┐  ┌───────────┐
   or timeout   │  Blocked  │◄─┤   Ready   │  wants CPU, waiting its turn
       ┌───────►│           │  └───────────┘
       │        └───────────┘
  vTaskDelay,        │ vTaskSuspend()      ┌────────────┐
  queue wait,        └────────────────────►│ Suspended  │ invisible to the
  semaphore...                             │            │ scheduler until
                                           └────────────┘ vTaskResume()
```

- **Running** — executing right now (one task per core; the ESP32 has two
  cores — next module).
- **Ready** — able to run, waiting for the CPU (a higher-priority or
  equal-priority task is running).
- **Blocked** — waiting for time (`vTaskDelay`) or an event (queue,
  semaphore, notification) with a timeout. Blocked tasks consume **zero**
  CPU — this is why blocking beats busy-waiting.
- **Suspended** — explicitly parked with `vTaskSuspend(handle)`; won't run
  again until `vTaskResume(handle)`. No timeout involved.

`vTaskDelay` is therefore not "wasting time" like `delay()` busy-waiting —
it's telling the scheduler "wake me in 250 ms; give the CPU to someone else."

## Cheat sheet

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

## Exercise

Build a three-task sketch in Wokwi:

1. **`heartbeat`** — toggles GPIO 25 every 500 ms with `vTaskDelay`.
2. **`sampler`** — prints `analogRead(34)` at *exactly* 4 Hz using
   `vTaskDelayUntil` (verify the timestamps land on a 250 ms grid using
   `millis()` in the printout).
3. **`reporter`** — every 3 s prints how many samples have been taken so far
   (share the count through a global `volatile uint32_t` for now — modules 4
   and 5 will show why that's naive and what to use instead).

Then deliberately break it: change `sampler` to use `vTaskDelay(250)` and add
`delay(30)` inside its loop body to simulate slow work — watch the timestamps
drift, then restore `vTaskDelayUntil` and watch them snap back to the grid.
