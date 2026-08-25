# Project — Multi-Sensor Gateway

This project assembles Level 2's toolkit into one device: a **gateway**
that ingests several sensor streams, filters and frames them, ships them
out over a single-writer transport, and stays alive under fault
conditions — using stream/message buffers, static allocation, queue sets,
a gatekeeper task, and watchdog supervision together, the way a real
field-deployed data logger would.

## Architecture

```text
  sensorA (10 Hz) ──┐                                  ┌─► radioGatekeeperTask
  sensorB (20 Hz) ──┼─► queue set ──► fusionTask ──────┤   (owns the single-writer
  sensorC (5 Hz)  ──┘   (3 queues)   (filters, frames)  │    message buffer)
                                                          └─► logs via Module 8
  watchdogTask ◄── liveness tokens from every task above     gatekeeper pattern

  All tasks + queues + the message buffer: created with ...CreateStatic()
```

Design decisions worth noticing before the code:

- **Queue set for fan-in** (Module 6) — three independent sensor tasks at
  different rates, one `fusionTask` blocking on all of them at once instead
  of polling three queues or running three consumer tasks.
- **Message buffer for the radio link** (Module 1) — each fused reading is
  one discrete, variable-length frame; the radio link's driver is
  single-writer, so only `radioGatekeeperTask` ever calls
  `xMessageBufferSend` on it.
- **Gatekeeper for both the radio and the log** (Module 8) — `fusionTask`
  never touches the radio driver or the log directly; it hands off frames
  and log lines and moves on, so a slow/blocked radio never stalls sensor
  fusion.
- **Everything statically allocated** (Module 2) — this device runs
  unattended for weeks; nothing about its task/queue population changes at
  runtime, so there is no reason to risk heap fragmentation after day 40.
- **Watchdog supervision** (Module 9) — every task updates a liveness
  token each iteration; the watchdog task only pets hardware (or, in the
  POSIX build, only refrains from calling `abort()`) when all tokens are
  fresh.
- **Priorities**: sensor tasks 3 (tightest, short, periodic), `fusionTask`
  2, `radioGatekeeperTask` 1 (I/O-bound, can be slow), `watchdogTask` 1
  but scheduled to run often enough to catch a hang within its timeout
  window.

## Config and static storage

```c
// FreeRTOSConfig.h additions on top of Level 1's baseline
#define configSUPPORT_STATIC_ALLOCATION   1
#define configSUPPORT_DYNAMIC_ALLOCATION  0   // prove nothing allocates at runtime
#define configUSE_QUEUE_SETS               1
#define configCHECK_FOR_STACK_OVERFLOW     2
#define configUSE_MALLOC_FAILED_HOOK       0  // no heap linked in at all
```

```cpp
#define STACK_WORDS 512

// One macro-generated static task+stack pair per task, to avoid repeating
// this boilerplate five times.
#define DECLARE_STATIC_TASK(name) \
  static StackType_t  name##Stack[STACK_WORDS]; \
  static StaticTask_t name##TCB;

DECLARE_STATIC_TASK(sensorA)
DECLARE_STATIC_TASK(sensorB)
DECLARE_STATIC_TASK(sensorC)
DECLARE_STATIC_TASK(fusion)
DECLARE_STATIC_TASK(radioGatekeeper)
DECLARE_STATIC_TASK(watchdog)

typedef struct { uint8_t source; uint32_t ms; int32_t value; } Reading;

static uint8_t       qAStorage[4 * sizeof(Reading)];
static StaticQueue_t qABuf;
static uint8_t       qBStorage[4 * sizeof(Reading)];
static StaticQueue_t qBBuf;
static uint8_t       qCStorage[4 * sizeof(Reading)];
static StaticQueue_t qCBuf;

static uint8_t             frameBufStorage[512];
static StaticStreamBuffer_t frameBufCtrl;
static MessageBufferHandle_t radioFrames;   // built with xMessageBufferCreateStatic

#define NUM_SUPERVISED_TASKS 5
static volatile TickType_t lastAlive[NUM_SUPERVISED_TASKS];
enum { TASK_A, TASK_B, TASK_C, TASK_FUSION, TASK_RADIO };
```

## Sensor tasks and the queue set

```cpp
QueueSetHandle_t sensorSet;
QueueHandle_t qA, qB, qC;

void sensorTaskGeneric(uint8_t source, QueueHandle_t q,
                        TickType_t period, int aliveIdx) {
  TickType_t lastWake = xTaskGetTickCount();
  for (;;) {
    vTaskDelayUntil(&lastWake, period);
    Reading r = { source, (uint32_t)xTaskGetTickCount(), readAdc(source) };
    xQueueSend(q, &r, 0);   // never block a periodic sampler on a full queue
    lastAlive[aliveIdx] = xTaskGetTickCount();
  }
}

void fusionTask(void *pv) {
  for (;;) {
    QueueSetMemberHandle_t ready = xQueueSelectFromSet(sensorSet, pdMS_TO_TICKS(200));
    if (ready != NULL) {
      Reading r;
      xQueueReceive(ready, &r, 0);   // non-blocking — the set already proved data is there
      char frame[48];
      int len = snprintf(frame, sizeof(frame), "{\"src\":%u,\"t\":%lu,\"v\":%ld}",
                          r.source, (unsigned long)r.ms, (long)r.value);
      xMessageBufferSend(radioFrames, frame, len, pdMS_TO_TICKS(20));
    }
    // ready == NULL just means the 200 ms timeout elapsed with nothing —
    // still a chance to update the liveness token below
    lastAlive[TASK_FUSION] = xTaskGetTickCount();
  }
}
```

Note the queue-set timeout: `fusionTask` uses a bounded wait, not
`portMAX_DELAY`, specifically so it can keep proving liveness to the
watchdog even during a real lull in sensor traffic — a `portMAX_DELAY`
wait with nothing arriving would look identical to a genuine hang.

## The gatekeeper: single writer to the radio

```cpp
void radioGatekeeperTask(void *pv) {
  // radioFrames is a message buffer — single-writer by construction,
  // and fusionTask is the only other task that ever touches it, so this
  // task's real job is draining it onto the actual (slow) radio driver.
  uint8_t buf[64];
  for (;;) {
    size_t len = xMessageBufferReceive(radioFrames, buf, sizeof(buf),
                                        pdMS_TO_TICKS(500));
    if (len > 0) {
      radio_transmit(buf, len);          // can be slow — nobody else waits on this
    }
    lastAlive[TASK_RADIO] = xTaskGetTickCount();
  }
}
```

## Watchdog supervision tying it together

```cpp
bool allTasksHealthy(void) {
  TickType_t now = xTaskGetTickCount();
  for (int i = 0; i < NUM_SUPERVISED_TASKS; i++) {
    if (now - lastAlive[i] > pdMS_TO_TICKS(2000)) return false;
  }
  return true;
}

void watchdogTask(void *pv) {
  hw_watchdog_init(pdMS_TO_TICKS(3000));
  for (;;) {
    if (allTasksHealthy()) {
      hw_watchdog_pet();
    } else {
      logFaultBeforeReset();   // persist which task went stale, then let the WDT fire
    }
    vTaskDelay(pdMS_TO_TICKS(500));
  }
}
```

## Wiring it up

```cpp
void app_init(void) {
  qA = xQueueCreateStatic(4, sizeof(Reading), qAStorage, &qABuf);
  qB = xQueueCreateStatic(4, sizeof(Reading), qBStorage, &qBBuf);
  qC = xQueueCreateStatic(4, sizeof(Reading), qCStorage, &qCBuf);

  sensorSet = xQueueCreateSet(4 + 4 + 4);
  xQueueAddToSet(qA, sensorSet);
  xQueueAddToSet(qB, sensorSet);
  xQueueAddToSet(qC, sensorSet);

  radioFrames = xMessageBufferCreateStatic(sizeof(frameBufStorage),
                                            frameBufStorage, &frameBufCtrl);

  xTaskCreateStatic(fusionTask, "fusion", STACK_WORDS, NULL, 2,
                     fusionStack, &fusionTCB);
  xTaskCreateStatic(radioGatekeeperTask, "radio", STACK_WORDS, NULL, 1,
                     radioGatekeeperStack, &radioGatekeeperTCB);
  xTaskCreateStatic(watchdogTask, "wdt", STACK_WORDS, NULL, 1,
                     watchdogStack, &watchdogTCB);
  // sensorA/B/C creation omitted — each wraps sensorTaskGeneric with its
  // own source id, queue handle, period, and lastAlive index
}
```

## Test plan

1. **Steady state** — all three sensors running at their nominal rate;
   confirm `radio_transmit` receives one well-formed JSON frame per
   sensor reading, in roughly the order readings occurred (queue-set
   ordering is not strictly FIFO across members — verify your app can
   tolerate that, or add per-frame sequence numbers if not).
2. **Back-pressure** — stall `radio_transmit` (sleep inside it) and confirm
   `radioFrames` fills, `xMessageBufferSend` in `fusionTask` starts timing
   out, and the *sensors* keep sampling on schedule regardless (Module 8's
   whole point: the gatekeeper absorbs the stall, not the producers).
3. **Static allocation proof** — build with `configSUPPORT_DYNAMIC_ALLOCATION`
   at 0 and confirm it still links and boots, proving no code path
   allocates at runtime.
4. **Watchdog trip** — hang one sensor task (infinite loop, no yield) and
   confirm the watchdog stops petting and the system resets within its
   configured timeout; confirm the persisted fault log correctly names
   the stalled task on the next boot.
5. **POSIX-port smoke test** — build the whole pipeline against the POSIX
   simulator (Module 5) with `radio_transmit` replaced by a `printf`, and
   confirm the same steady-state and back-pressure behavior is observable
   without any real hardware attached.

## Stretch goals

Pick at least two:

1. **Sequence numbers and loss detection** — add a monotonic counter to
   each frame and have a companion "ground station" script (or a second
   POSIX-port task) detect gaps, distinguishing genuine radio loss from
   gatekeeper back-pressure drops.
2. **Priority-aware queue set draining** — modify `fusionTask` so that if
   `qA` (highest-priority sensor) has data, it's always drained before `qB`
   or `qC` even when `xQueueSelectFromSet` returns a lower-priority member
   first; measure the added worst-case latency for the lower-priority
   sensors under sustained high-priority traffic.
3. **Config over the same link** — add a second message buffer for
   inbound commands from the radio, and a small command parser task; work
   out why this needs its own gatekeeper-owned single-writer buffer rather
   than sharing `radioFrames`.
4. **Tickless idle** — enable Module 3's tickless idle and measure how
   much idle time the gateway actually gets between sensor periods at its
   real sampling rates; tune `configEXPECTED_IDLE_TIME_BEFORE_SLEEP` to
   the result.
5. **ESP-IDF port** — rebuild on ESP-IDF with sensors pinned to one core
   and the radio gatekeeper pinned to the other (Module 4); confirm the
   liveness tokens still work correctly when written from one core and
   read from another (hint: consider whether `volatile TickType_t` alone
   is sufficient on a dual-core SMP target, or whether the read/compare in
   `allTasksHealthy()` needs a memory barrier or atomic access).
6. **Graduated fault response** — instead of an immediate hardware reset
   on the first stale token, add a warning state (blink an LED, log a
   fault) after one missed liveness window, and only let the hardware
   watchdog actually fire after a second consecutive miss; justify the
   two-strike threshold against the risk of resetting on a merely slow,
   not truly hung, task.

Finish those and you've built the Level 2 equivalent of the Level 1
capstone: real fan-in, real backpressure isolation, real static-RAM
guarantees, and real fault recovery — the operational concerns that
separate a demo from a device you'd actually deploy. Onward to Level 3.
