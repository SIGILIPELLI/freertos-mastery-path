# ESP-IDF FreeRTOS Specifics

Everything in Level 1 targets vanilla, single-core FreeRTOS. ESP-IDF ships
a fork of FreeRTOS (based on the mainline kernel, with Espressif's
extensions merged in) that adds dual-core (or single-core, on some newer
chips) SMP scheduling, an extra API surface for pinning work to a specific
core, and its own idle/power-management layer. If you've been developing
against an ESP32 target all along, several Level 1 assumptions need
revisiting explicitly.

## Dual-core: two schedulers sharing one ready list

The original ESP32 has two cores, conventionally called PRO_CPU (0) and
APP_CPU (1). ESP-IDF's FreeRTOS runs one scheduler *instance* that spans
both cores — a single ready list, but two cores can each be truly running
a task **at the same time**. That is a fundamentally different concurrency
model from single-core preemptive multitasking:

```cpp
void taskA(void *pv) { for (;;) { sharedCounter++; } }   // core 0
void taskB(void *pv) { for (;;) { sharedCounter++; } }   // core 1

xTaskCreatePinnedToCore(taskA, "A", 2048, NULL, 2, NULL, 0);
xTaskCreatePinnedToCore(taskB, "B", 2048, NULL, 2, NULL, 1);
```

On single-core FreeRTOS, `sharedCounter++` (read-modify-write) is only at
risk from *preemption* — a critical section (`taskENTER_CRITICAL`, which
just disables interrupts) is enough to protect it. On dual-core, taskA and
taskB can execute that instruction **simultaneously on separate cores** —
disabling interrupts on core 0 does nothing to stop core 1 from writing at
the same instant. This is genuine parallelism, not just concurrency, and it
needs genuine mutual exclusion:

```cpp
static portMUX_TYPE counterMux = portMUX_INITIALIZER_UNLOCKED;

void taskA(void *pv) {
  for (;;) {
    portENTER_CRITICAL(&counterMux);   // spinlock — blocks the OTHER core too
    sharedCounter++;
    portEXIT_CRITICAL(&counterMux);
  }
}
```

`portMUX_TYPE` is a spinlock: it disables interrupts on the calling core
*and* spins until it wins the lock against the other core. Keep the
protected section extremely short — every cycle spent inside it is a cycle
the other core may be spinning, wasting power and adding latency.

## Task affinity: pinned vs. unpinned

```cpp
xTaskCreatePinnedToCore(fn, "name", stackWords, arg, prio, &handle, 0);   // PRO_CPU only
xTaskCreatePinnedToCore(fn, "name", stackWords, arg, prio, &handle, 1);   // APP_CPU only
xTaskCreatePinnedToCore(fn, "name", stackWords, arg, prio, &handle,
                        tskNO_AFFINITY);                                  // either core
xTaskCreate(fn, "name", stackWords, arg, prio, &handle);                 // = tskNO_AFFINITY
```

Espressif's own guidance: Wi-Fi/Bluetooth stack tasks are typically pinned
by the driver itself, and application tasks that must never contend with
radio-stack timing are commonly pinned to the other core. Unpinned tasks
can migrate between cores between runs (never mid-run), which maximizes
scheduler flexibility but makes cache-affinity and worst-case-latency
reasoning harder — pin explicitly whenever a task has a real timing
relationship to another pinned task or ISR.

## `IRAM_ATTR` and ISR placement

Flash on ESP32 is accessed through a cache; certain operations (writing to
flash, some `spi_flash_*` calls) **disable that cache**, during which any
code or ISR not already resident in IRAM cannot execute. An ISR that reads
a GPIO and must never miss an edge needs:

```cpp
void IRAM_ATTR gpio_isr_handler(void *arg) {
  BaseType_t xHigherPriorityTaskWoken = pdFALSE;
  xTaskNotifyFromISR((TaskHandle_t)arg, 0, eNoAction, &xHigherPriorityTaskWoken);
  portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

Without `IRAM_ATTR`, this handler lives in flash and — if it happens to
fire during a flash-cache-disabled window (e.g., a concurrent NVS write) —
the core will hard-fault trying to fetch instructions from inaccessible
flash. This has no equivalent in Level 1's vanilla-port coverage; it's an
ESP32-specific hazard tied to the shared instruction/flash cache.

## `esp_pm_configure`: ESP-IDF's power-management layer

Rather than exposing the raw tickless-idle hooks from Module 3 directly,
ESP-IDF layers dynamic frequency scaling on top:

```c
esp_pm_config_t cfg = {
  .max_freq_mhz = 240,
  .min_freq_mhz = 80,
  .light_sleep_enable = true,
};
esp_pm_configure(&cfg);
```

With `light_sleep_enable` set, ESP-IDF automatically drops into light sleep
during otherwise-idle periods, using the same tickless-idle mechanism from
Module 3 under the hood but managed through this higher-level API instead
of raw port hooks. `CONFIG_FREERTOS_USE_TICKLESS_IDLE` in `sdkconfig` is
the underlying Kconfig knob.

## Watchdogs are on by default

ESP-IDF enables the **Task Watchdog Timer** (TWDT) and typically subscribes
the idle task(s) of both cores to it out of the box. A task that hogs a
core without yielding starves that core's idle task, which then fails to
"pet" the watchdog, producing a reset with a backtrace — this is the
mechanism behind the common ESP-IDF crash `"Task watchdog got triggered"`.
Module 9 covers watchdog design in depth; the ESP-IDF-specific point here
is that this exists and is active from boot, unlike vanilla FreeRTOS where
you must wire up watchdog integration yourself.

## Traps

- **Porting a Level 1 single-core critical section verbatim**: `taskENTER_CRITICAL()`
  with no argument, or with the pre-SMP single-argument form, does not
  protect against the *other* core on ESP-IDF's dual-core SMP port — use
  the `portMUX_TYPE`-based spinlock APIs (`portENTER_CRITICAL(&mux)`) for
  anything shared across cores.
- **Assuming `configNUMBER_OF_CORES` behaves like an unrelated single-core
  build**: code that assumes "only one task runs at any instant" (a common
  unstated assumption when reasoning about atomicity) is simply false on
  this port when two pinned tasks target different cores.
- **ISRs in flash during a flash write**: missing `IRAM_ATTR` on a
  high-priority GPIO/timer ISR causes intermittent, hard-to-reproduce
  crashes that only occur near NVS/OTA/flash-write activity.
- **Fighting the watchdog by disabling it instead of fixing the task**:
  `esp_task_wdt_delete()` on a task that's genuinely starving a core hides
  a real scheduling bug rather than solving it — prefer shortening the
  task's busy-loop or raising its yield frequency.
- **Stack sizing surprises**: ESP-IDF's default stack sizes and units
  (bytes, via `xTaskCreate`'s `usStackDepth` parameter on this port —
  double-check against vanilla FreeRTOS's word-based convention) differ
  from mainline conventions; always check `CONFIG_FREERTOS_*` defaults in
  `sdkconfig` rather than assuming Level 1's numbers port over unchanged.

## Cheat sheet

| Concept | Vanilla FreeRTOS | ESP-IDF FreeRTOS |
|---|---|---|
| Task creation | `xTaskCreate(...)` | `xTaskCreate(...)` (= `tskNO_AFFINITY`) or `xTaskCreatePinnedToCore(..., core)` |
| Cross-core protection | N/A (single core) | `portMUX_TYPE` spinlock + `portENTER/EXIT_CRITICAL(&mux)` |
| ISR flash safety | N/A | `IRAM_ATTR` required for ISRs that may fire during flash-cache-disabled windows |
| Power management | Raw tickless-idle hooks (Module 3) | `esp_pm_configure()` (DFS + light sleep), Kconfig-driven |
| Watchdog | Opt-in, manual wiring (Module 9) | Task Watchdog Timer (TWDT) active by default |
| Stack units | Words (`StackType_t`) on most ports | Bytes, on ESP-IDF's `xTaskCreate` |
| Affinity constant | N/A | `tskNO_AFFINITY`, or explicit core `0`/`1` |

## Exercise

1. Create two tasks pinned to different cores that both increment a shared
   `volatile uint32_t` a million times each without any locking. Compare
   the final count to the expected 2,000,000 and explain the discrepancy
   in terms of genuine cross-core parallelism, not just preemption.
2. Fix it with a `portMUX_TYPE` spinlock and confirm the count is exact.
   Measure the wall-clock time cost of the fix versus the unprotected
   version.
3. Write a GPIO ISR without `IRAM_ATTR`, trigger a concurrent flash
   operation (e.g., `nvs_set_*` in a loop from another task) while pulsing
   the GPIO, and observe the crash. Add `IRAM_ATTR` and confirm stability.
