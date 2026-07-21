# 07 · Task Notifications & Event Groups

Queues and semaphores are general-purpose — and pay for it in RAM and
cycles. When the communication pattern is simple and the target is a *known,
single task*, **task notifications** do the same job dramatically cheaper:
every task has a built-in notification slot that can act as a faster binary
semaphore, counting semaphore, or tiny mailbox. At the other end of the
spectrum, **event groups** solve the many-flags problem: one task waiting on
*combinations* of events ("WiFi up AND time synced", "any error bit"). This
module covers both, and when each earns its place.

## Task notifications: the built-in fast path

Every task carries a 32-bit notification value plus a pending flag — no
object to create. The simplest use is a direct replacement for a binary
semaphore, signaling a specific task:

```cpp
TaskHandle_t workerHandle;

void workerTask(void *pv) {
  for (;;) {
    // Like xSemaphoreTake — sleeps until notified:
    ulTaskNotifyTake(pdTRUE,           // clear count on exit (binary style)
                     portMAX_DELAY);   // wait forever
    Serial.printf("[%lu ms] worker: notified!\n", millis());
  }
}

void triggerTask(void *pv) {
  for (;;) {
    vTaskDelay(pdMS_TO_TICKS(1200));
    xTaskNotifyGive(workerHandle);     // like xSemaphoreGive, but targeted
  }
}

void setup() {
  Serial.begin(115200);
  xTaskCreate(workerTask,  "worker",  2048, NULL, 2, &workerHandle);
  xTaskCreate(triggerTask, "trigger", 2048, NULL, 1, NULL);
}

void loop() { vTaskDelay(portMAX_DELAY); }
```

Why bother, when a semaphore worked fine?

- **~45 % faster** and no RAM beyond the TCB the task already has —
  the FreeRTOS docs' own numbers; no separate object to create or leak.
- The giver targets *one task by handle* — no risk of the wrong waiter
  winning.

Limitations (the price of the fast path): only **one task** can wait on a
notification (it's *that task's* slot), a notification can't be "given" to
multiple waiters, and there's no buffering beyond the 32-bit value.

### Counting and mailbox flavors

`ulTaskNotifyTake(pdFALSE, ...)` **decrements** instead of clearing —
counting-semaphore behavior (three gives = three wakes). And the full API
sends actual values:

```cpp
// Sender: overwrite the 32-bit value (latest-wins mailbox)
xTaskNotify(workerHandle, analogRead(34), eSetValueWithOverwrite);

// Receiver:
uint32_t value;
xTaskNotifyWait(0,               // bits to clear on entry
                0xFFFFFFFF,      // bits to clear on exit
                &value,          // receives the notification value
                portMAX_DELAY);
```

Other actions: `eSetBits` (OR bits in — a mini event group),
`eIncrement` (counting), `eSetValueWithoutOverwrite` (fail if one is already
pending — a depth-1 queue). Rule of thumb: **one known consumer + a signal
or a single latest-value word → notification. Multiple consumers, multiple
producers needing buffering, or payloads bigger than 32 bits → queue.**

## Event groups: many flags, flexible waits

An **event group** is a set of up to 24 named bits. Tasks set bits as things
happen; other tasks block until a *combination* of bits is set — the
many-to-one synchronization tool.

```cpp
#include "freertos/event_groups.h"

EventGroupHandle_t sysEvents;
#define BIT_WIFI_UP     (1 << 0)
#define BIT_TIME_SYNCED (1 << 1)
#define BIT_SENSOR_OK   (1 << 2)

void wifiSimTask(void *pv) {
  vTaskDelay(pdMS_TO_TICKS(1000));
  Serial.println("wifi: connected");
  xEventGroupSetBits(sysEvents, BIT_WIFI_UP);
  vTaskDelay(pdMS_TO_TICKS(800));
  Serial.println("time: synced");
  xEventGroupSetBits(sysEvents, BIT_TIME_SYNCED);
  vTaskDelete(NULL);                       // done — this task exits
}

void sensorSimTask(void *pv) {
  vTaskDelay(pdMS_TO_TICKS(400));
  Serial.println("sensor: self-test passed");
  xEventGroupSetBits(sysEvents, BIT_SENSOR_OK);
  vTaskDelete(NULL);
}

void appTask(void *pv) {
  Serial.println("app: waiting for ALL subsystems...");
  EventBits_t bits = xEventGroupWaitBits(
      sysEvents,
      BIT_WIFI_UP | BIT_TIME_SYNCED | BIT_SENSOR_OK,  // bits to wait for
      pdFALSE,          // don't clear them on exit (they're system state)
      pdTRUE,           // wait for ALL bits (pdFALSE = ANY bit)
      pdMS_TO_TICKS(5000));                            // timeout

  if ((bits & (BIT_WIFI_UP | BIT_TIME_SYNCED | BIT_SENSOR_OK)) ==
      (BIT_WIFI_UP | BIT_TIME_SYNCED | BIT_SENSOR_OK)) {
    Serial.printf("[%lu ms] app: all systems go!\n", millis());
  } else {
    Serial.printf("app: startup timeout, got bits 0x%02x\n", (int)bits);
  }
  for (;;) vTaskDelay(portMAX_DELAY);
}

void setup() {
  Serial.begin(115200);
  sysEvents = xEventGroupCreate();
  xTaskCreate(appTask,       "app",    2048, NULL, 2, NULL);
  xTaskCreate(wifiSimTask,   "wifi",   2048, NULL, 1, NULL);
  xTaskCreate(sensorSimTask, "sensor", 2048, NULL, 1, NULL);
}

void loop() { vTaskDelay(portMAX_DELAY); }
```

`xEventGroupWaitBits` parameters worth internalizing:

- **`xClearOnExit`** — `pdTRUE` for *event*-style bits (consume them),
  `pdFALSE` for *state*-style bits (WiFi is still up after you notice).
- **`xWaitForAllBits`** — `pdTRUE` = AND ("everything ready"), `pdFALSE` =
  OR ("anything happened — inspect the return value to see what").
- **Return value** is the bit state *at the moment the wait ended* — on a
  timeout, check it to see which bits you did get (as above; note the wait
  can also return early with a partial set if it timed out).

Unlike notifications, **many tasks can wait on the same event group**, and
one `xEventGroupSetBits` wakes every waiter whose condition is now true —
that's the broadcast/barrier superpower (there's even
`xEventGroupSync` for rendezvous points). The trade-off: every set
operation has to re-evaluate waiters, so it's the heavyweight of this
module.

## Choosing between the signaling tools

| Pattern | Best tool |
|---|---|
| ISR/task wakes ONE known task, no data | Task notification (`xTaskNotifyGive`) |
| Same, with a count that must not be lost | Notification, `ulTaskNotifyTake(pdFALSE, ...)` |
| One latest-value word to one task | Notification, `eSetValueWithOverwrite` |
| Stream of data items, buffering, any # of producers | Queue |
| N-resource pool, or unknown/multiple waiters | Semaphore |
| Wait for combinations of conditions; broadcast | Event group |

## Cheat sheet

| API | Purpose |
|---|---|
| `xTaskNotifyGive(h)` / `ulTaskNotifyTake(clear, timeout)` | Fast binary/counting semaphore aimed at one task |
| `xTaskNotify(h, val, action)` | Send value/bits: `eSetValueWithOverwrite`, `eSetBits`, `eIncrement`... |
| `xTaskNotifyWait(clrIn, clrOut, &val, timeout)` | Wait and receive the 32-bit value |
| `xEventGroupCreate()` | New event group (≤24 usable bits) |
| `xEventGroupSetBits(eg, bits)` / `xEventGroupClearBits(eg, bits)` | Set / clear flags |
| `xEventGroupWaitBits(eg, bits, clearOnExit, waitAll, timeout)` | Block on ANY/ALL combinations; returns bits at wake |
| `xEventGroupGetBits(eg)` | Peek current bits without blocking |

## Exercise

1. Take your Module 5 exercise (trigger + worker via binary semaphore) and
   port it to task notifications — count the lines and objects you deleted.
2. Extend the event-group demo with a `BIT_ERROR` flag set by a fourth task
   at a random moment: `appTask`, after startup, loops on
   `xEventGroupWaitBits(..., BIT_ERROR, pdTRUE /* clear */, pdFALSE, portMAX_DELAY)`
   and prints an alarm each time. Why is `xClearOnExit = pdTRUE` correct
   here but wrong for `BIT_WIFI_UP`?
3. Design question (answer in comments): a button ISR must wake a handler
   task; a logger task must receive readings from three producer tasks; a
   startup sequence must wait for four subsystems. Which tool for each, and
   why?
