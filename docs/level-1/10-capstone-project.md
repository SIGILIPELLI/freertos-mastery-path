# 10 · Capstone — Smart Sensor Device

Time to assemble everything. This capstone builds a complete multi-task
"smart sensor device" on the ESP32 in Wokwi: a sensor task samples at a fixed
rate and streams readings through a **queue** to a processing task that
checks them against a threshold; a **button ISR** arms/disarms the alarm via
a **task notification**; a **software timer** drives the status LED; and a
serial command task edits a shared configuration protected by a **mutex**.
Every module of this level appears here doing the exact job it was designed
for — plus a wiring list, a test plan, and extensions to make it your own.

## Architecture

```text
                     ┌───────────────┐   Reading    ┌────────────────┐
  potentiometer ───► │  sensorTask   │──── queue ──►│  processTask   │──► alarm LED
  (GPIO 34)          │ 5 Hz, DelayUntil│  (8 deep)  │ threshold check│
                     └───────────────┘              └───────┬────────┘
                                                            │ reads config
  button (GPIO 27) ──ISR──notification──► armTask           │ (mutex)
                          toggles cfg.armed (mutex)         │
                                                            │
  serial "set/get/stats" ──► commandTask ──── writes config (mutex)
                                                            
  statusTimer (auto-reload) ──► heartbeat LED: slow=disarmed, fast=armed
```

Design decisions worth noticing before the code:

- **Queue for readings** — a stream of data items: exactly what queues are
  for (Module 4). The struct is small, so we queue it by value.
- **Notification for the button** — one known consumer, no payload: the
  cheapest correct tool (Modules 7-8).
- **Mutex for `config`** — multi-field shared state read and written by
  three tasks (Module 5). Readers snapshot it and get out.
- **Timer for the LED** — a small timed side-effect; no task or precision
  needed (Module 6).
- **Priorities** — sensor 3 (tightest deadline), processing 2, command and
  arm 1 (human-speed). All well below the button *handling* path's
  requirements? No — the arm task is human-speed too; the *ISR* is what
  makes the press never get lost.

## Wiring (Wokwi parts)

| Part | Connection |
|---|---|
| Potentiometer (our "sensor") | wiper → GPIO 34, ends → 3V3 and GND |
| Pushbutton | one leg → GPIO 27, other leg → GND (internal pull-up) |
| Red LED + 220 Ω | anode → GPIO 25 (alarm) |
| Green LED + 220 Ω | anode → GPIO 26 (status heartbeat) |

In Wokwi: create an ESP32 project, click "+" to add `wokwi-potentiometer`,
`wokwi-pushbutton`, and two `wokwi-led`s, and wire as above.

## Full sketch

```cpp
// Smart Sensor Device — Level 1 capstone (ESP32 Arduino / FreeRTOS)

const uint8_t SENSOR_PIN = 34;
const uint8_t BUTTON_PIN = 27;
const uint8_t ALARM_LED  = 25;
const uint8_t STATUS_LED = 26;

// ---------- shared configuration (mutex-guarded) ----------
struct Config {
  bool     armed;        // alarm enabled?
  int      threshold;    // raw ADC alarm level (0..4095)
  uint32_t periodMs;     // sensor sampling period
};
Config config = { true, 3000, 200 };
SemaphoreHandle_t configMutex;

Config getConfig() {                       // snapshot helper: lock briefly
  xSemaphoreTake(configMutex, portMAX_DELAY);
  Config c = config;
  xSemaphoreGive(configMutex);
  return c;
}

// ---------- reading pipeline ----------
struct Reading { uint32_t ms; int raw; };
QueueHandle_t readingQueue;
volatile uint32_t droppedReadings = 0;     // stats only

// ---------- button → arm/disarm ----------
TaskHandle_t armTaskHandle;

void IRAM_ATTR buttonISR() {
  BaseType_t woken = pdFALSE;
  vTaskNotifyGiveFromISR(armTaskHandle, &woken);
  portYIELD_FROM_ISR(woken);
}

void armTask(void *pv) {
  for (;;) {
    ulTaskNotifyTake(pdTRUE, portMAX_DELAY);         // wait for a press
    xSemaphoreTake(configMutex, portMAX_DELAY);
    config.armed = !config.armed;
    bool nowArmed = config.armed;
    xSemaphoreGive(configMutex);
    Serial.printf("[%lu] button: system %s\n", millis(),
                  nowArmed ? "ARMED" : "DISARMED");
    vTaskDelay(pdMS_TO_TICKS(250));                  // debounce window
    ulTaskNotifyTake(pdTRUE, 0);                     // drain bounces
  }
}

// ---------- status LED timer ----------
TimerHandle_t statusTimer;

void statusCallback(TimerHandle_t t) {
  digitalWrite(STATUS_LED, !digitalRead(STATUS_LED));
}

// ---------- tasks ----------
void sensorTask(void *pv) {                          // priority 3
  TickType_t lastWake = xTaskGetTickCount();
  for (;;) {
    uint32_t period = getConfig().periodMs;
    vTaskDelayUntil(&lastWake, pdMS_TO_TICKS(period));
    Reading r = { millis(), analogRead(SENSOR_PIN) };
    if (xQueueSend(readingQueue, &r, 0) != pdPASS) {
      droppedReadings++;                             // never block the sampler
    }
  }
}

void processTask(void *pv) {                         // priority 2
  Reading r;
  bool alarmLatched = false;
  for (;;) {
    xQueueReceive(readingQueue, &r, portMAX_DELAY);
    Config c = getConfig();
    bool over = (r.raw >= c.threshold);
    bool alarm = c.armed && over;
    if (alarm != alarmLatched) {                     // log transitions only
      alarmLatched = alarm;
      digitalWrite(ALARM_LED, alarm ? HIGH : LOW);
      Serial.printf("[%lu] ALARM %s (raw=%d thr=%d)\n",
                    r.ms, alarm ? "ON" : "off", r.raw, c.threshold);
      // armed + alarming → fast heartbeat, else slow
      xTimerChangePeriod(statusTimer,
                         pdMS_TO_TICKS(alarm ? 100 : 700), 0);
    }
  }
}

void commandTask(void *pv) {                         // priority 1
  String line;
  for (;;) {
    while (Serial.available()) {
      char ch = (char)Serial.read();
      if (ch == '\n') {
        line.trim();
        if (line == "get") {
          Config c = getConfig();
          Serial.printf("armed=%d threshold=%d period=%lu\n",
                        c.armed, c.threshold, (unsigned long)c.periodMs);
        } else if (line.startsWith("set thr ")) {
          int v = line.substring(8).toInt();
          xSemaphoreTake(configMutex, portMAX_DELAY);
          config.threshold = constrain(v, 0, 4095);
          xSemaphoreGive(configMutex);
          Serial.printf("ok threshold=%d\n", v);
        } else if (line.startsWith("set period ")) {
          int v = line.substring(11).toInt();
          xSemaphoreTake(configMutex, portMAX_DELAY);
          config.periodMs = max(50, v);
          xSemaphoreGive(configMutex);
          Serial.printf("ok period=%d\n", v);
        } else if (line == "stats") {
          Serial.printf("queued=%u dropped=%lu heap=%u minheap=%u stack=%u\n",
                        uxQueueMessagesWaiting(readingQueue),
                        (unsigned long)droppedReadings,
                        ESP.getFreeHeap(), ESP.getMinFreeHeap(),
                        uxTaskGetStackHighWaterMark(NULL));
        } else if (line.length()) {
          Serial.println("cmds: get | set thr N | set period N | stats");
        }
        line = "";
      } else {
        line += ch;
      }
    }
    vTaskDelay(pdMS_TO_TICKS(20));                   // human-speed polling
  }
}

void setup() {
  Serial.begin(115200);
  pinMode(ALARM_LED, OUTPUT);
  pinMode(STATUS_LED, OUTPUT);
  pinMode(BUTTON_PIN, INPUT_PULLUP);

  // kernel objects FIRST (Module 4's lesson)
  configMutex  = xSemaphoreCreateMutex();
  readingQueue = xQueueCreate(8, sizeof(Reading));
  statusTimer  = xTimerCreate("status", pdMS_TO_TICKS(700), pdTRUE,
                              NULL, statusCallback);
  if (!configMutex || !readingQueue || !statusTimer) {
    Serial.println("FATAL: object creation failed"); for (;;);
  }

  xTaskCreate(armTask,     "arm",     2048, NULL, 1, &armTaskHandle);
  xTaskCreate(sensorTask,  "sensor",  2048, NULL, 3, NULL);
  xTaskCreate(processTask, "process", 3072, NULL, 2, NULL);
  xTaskCreate(commandTask, "command", 4096, NULL, 1, NULL);

  attachInterrupt(digitalPinToInterrupt(BUTTON_PIN), buttonISR, FALLING);
  xTimerStart(statusTimer, 0);
  Serial.println("smart sensor up — try: get | set thr N | set period N | stats");
}

void loop() { vTaskDelay(portMAX_DELAY); }
```

## Test plan

Run each check in Wokwi and tick it off:

1. **Boot** — banner prints; green status LED blinks slowly (~0.7 Hz);
   alarm LED off.
2. **Alarm path** — turn the potentiometer up past the threshold: alarm LED
   on, `ALARM ON` logged once (not spammed), status blink goes fast. Turn it
   down: `ALARM off`, slow blink. *(Modules 4, 6: queue flow + timer
   period change.)*
3. **Arm toggle** — press the button: `DISARMED` logs; with the pot still
   high, alarm LED turns off on the next reading. Press again: re-arms.
   Hammer the button — exactly one toggle per press. *(Modules 7-8:
   ISR → notification + debounce drain.)*
4. **Commands** — `get` echoes config; `set thr 1000` makes the alarm trip
   earlier (verify); `set period 1000` visibly slows sampling (watch alarm
   response lag grow); bogus input prints the help line. *(Module 5: mutex'd
   writes while readers keep running.)*
5. **Health** — `stats` shows `dropped=0` in normal running, stable heap,
   and a sane stack mark. Now `set period 50` and add `vTaskDelay(300)`
   inside `processTask`'s loop (simulated slow processing): `stats` shows
   the queue filling and drops counting up — back-pressure made visible.
   Remove the sabotage. *(Modules 4, 9.)*
6. **Timing integrity** — the status LED never stutters during command
   parsing or alarm printing, because it lives on the timer task and nothing
   in the system blocks for long. *(Module 3's promise, delivered.)*

## Exercise — extensions

Pick at least two:

1. **Latest-value display** — add a length-1 `xQueueOverwrite` "mailbox"
   holding the newest reading, and a `disp` command that prints it — without
   disturbing the alarm pipeline.
2. **Event-group startup** — gate all tasks on a `BIT_READY` event group bit
   set at the end of `setup()`, removing any chance of racing object
   creation (then explain why the current create-order already protects
   you).
3. **Sensor fusion** — add a second analog input as "sensor 2" with its own
   task sending into the *same* queue (add a `source` field). The processor
   alarms only if *both* exceed threshold within 1 s of each other.
4. **Persist config** — save `config` to `Preferences` (ESP32 NVS) on every
   change and restore at boot. Which task should own the (slow!) flash
   write, and why not `commandTask` while holding the mutex?
5. **Health task** — port your Module 9 monitor: every 10 s log all stack
   high-water marks and heap stats, and alarm on any downward trend.
6. **Port it** — rebuild the pipeline (minus pins) on the FreeRTOS
   POSIX/Linux simulator with `getchar()` as the "button": what changes,
   and what (satisfyingly) doesn't?

Finish those and you're genuinely operating an RTOS: you've scheduled,
communicated, synchronized, deferred interrupts, and measured memory — the
foundation Levels 2-4 build on. Onward → [Level 2](../level-2/index.md).
