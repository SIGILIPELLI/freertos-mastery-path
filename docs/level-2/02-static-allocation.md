# Static Allocation

Every `xTaskCreate`, `xQueueCreate`, and `xSemaphoreCreateMutex` call in
Level 1 pulled memory from the FreeRTOS heap (`pvPortMalloc` under the
hood). That's convenient, but on a long-running embedded device it raises
a hard question: what happens when the heap fragments or runs out, hours
or days into operation, inside a `Create()` call you assumed could never
fail? Static allocation answers by moving every byte of RTOS object memory
into `.bss`, decided at compile time, with zero runtime allocation and zero
possibility of an out-of-memory failure at task-creation time.

## The `...CreateStatic()` family

Every dynamic create function in FreeRTOS has a static twin that takes the
same configuration plus caller-supplied storage:

```cpp
#define STACK_WORDS 256   // stack size in words, not bytes

static StackType_t  sensorStack[STACK_WORDS];
static StaticTask_t sensorTaskBuffer;

static QueueHandle_t sensorQueue;
static uint8_t       sensorQueueStorage[8 * sizeof(int32_t)];
static StaticQueue_t sensorQueueBuffer;

void app_init(void) {
  sensorQueue = xQueueCreateStatic(
      8, sizeof(int32_t),
      sensorQueueStorage,      // raw byte storage for the items
      &sensorQueueBuffer);     // control structure — replaces heap metadata

  TaskHandle_t h = xTaskCreateStatic(
      sensorTask, "sensor", STACK_WORDS, NULL, tskIDLE_PRIORITY + 2,
      sensorStack,             // caller-owned stack memory
      &sensorTaskBuffer);      // caller-owned TCB memory
  // h is never NULL — there is nothing left to allocate that can fail
}
```

Notice what disappears: no `NULL` check after `xTaskCreateStatic`. The
function returns a valid handle unconditionally once the arguments are
well-formed, because every byte it needs was handed to it by the caller.
Contrast with `xTaskCreate`, which returns `pdFAIL`/`NULL` on heap
exhaustion — a failure mode that is easy to forget to check and, worse,
tends to appear only under memory pressure you didn't test.

## Enabling it: the config knobs

```c
// FreeRTOSConfig.h
#define configSUPPORT_STATIC_ALLOCATION   1   // enables ...CreateStatic()
#define configSUPPORT_DYNAMIC_ALLOCATION  1   // set to 0 to remove the heap entirely
```

With only static allocation enabled (`DYNAMIC_ALLOCATION` at 0), the heap
(`heap_1.c`..`heap_5.c`) is never linked in at all — useful on the tightest
targets where you want the linker to prove, by omission, that nothing can
allocate at runtime. If you disable dynamic allocation you must also
supply two callback functions the kernel uses for the idle and timer
tasks, which it creates internally:

```cpp
static StaticTask_t idleTaskTCB;
static StackType_t  idleStack[configMINIMAL_STACK_SIZE];

void vApplicationGetIdleTaskMemory(StaticTask_t **ppxIdleTaskTCBBuffer,
                                    StackType_t **ppxIdleTaskStackBuffer,
                                    uint32_t *pulIdleTaskStackSize) {
  *ppxIdleTaskTCBBuffer   = &idleTaskTCB;
  *ppxIdleTaskStackBuffer = idleStack;
  *pulIdleTaskStackSize   = configMINIMAL_STACK_SIZE;
}
// vApplicationGetTimerTaskMemory() is the equivalent for the timer service task
```

Forgetting these two callbacks when `configSUPPORT_DYNAMIC_ALLOCATION` is 0
is a link-time error, not a subtle runtime bug — the linker will tell you
immediately, which is exactly the kind of failure you want at compile time
rather than in the field.

## Static vs dynamic: the real tradeoff

Static allocation isn't strictly "better" — it trades flexibility for
determinism:

- **Dynamic** lets you create and delete tasks/queues at runtime in
  response to conditions (a plug-in sensor detected, a Bluetooth peer
  connects). Total RAM use adapts to what's actually active.
- **Static** commits worst-case RAM for every object up front, at compile
  time, whether or not it's ever needed simultaneously — but that
  worst-case is knowable at link time via the `.map` file, and it can
  *never* fail at 3 AM because of fragmentation from six hours of
  create/delete churn.

The common real-world pattern is "static for everything that lives for the
life of the program" (which is most embedded firmware — tasks, queues, and
semaphores set up once in `app_init()` and never destroyed) combined with
dynamic allocation left enabled only for the rare genuinely-dynamic object
class (e.g., per-connection buffers on a device that accepts a variable
number of BLE peers).

## Stack sizing still matters — arguably more

Static allocation does not make stack overflow go away; it just changes
*how* it fails. A dynamically-sized stack that's too small corrupts heap
metadata near it; a statically-sized stack that's too small corrupts
whatever `.bss`/`.data` variable the linker happened to place next to it —
often another task's TCB. Either way, enable overflow detection while you
size things:

```c
#define configCHECK_FOR_STACK_OVERFLOW  2   // method 2: pattern-fill check, catches more cases

void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName) {
  // by the time this runs, damage may already be done — log and reset
  printf("STACK OVERFLOW: %s\n", pcTaskName);
  esp_restart();   // or NVIC_SystemReset(), or your platform's reset call
}
```

Measure real usage with `uxTaskGetStackHighWaterMark(handle)` (returns the
minimum free stack, in words, ever observed) under worst-case call paths —
deepest recursion, largest `printf` format string, any code path that
nests interrupts — then size static stacks with headroom, not to the
observed minimum.

## Traps

- **Reusing storage across two `...CreateStatic()` calls**: the
  `StaticTask_t`/`StaticQueue_t` buffer and the raw storage array must each
  back exactly one object for its entire lifetime. Passing the same buffer
  to two create calls (or reusing one after `vTaskDelete` without
  understanding TCB lifetime) corrupts kernel state silently.
- **`StackType_t` is words, not bytes**: `STACK_WORDS` in the example above
  is in units of `StackType_t` (4 bytes on most 32-bit ports) — a common
  off-by-4x sizing bug when porting a byte-count from another RTOS.
  `xTaskCreate`'s stack parameter has the exact same unit, so this trap
  isn't unique to static allocation, but static's fixed-size arrays make it
  more visible at compile time (a wrong `#define` is easy to typo).
- **Forgetting the idle/timer task memory callbacks**: only a problem when
  dynamic allocation is fully disabled, but produces a confusing
  undefined-reference linker error the first time, rather than a runtime
  symptom.
- **Global/static objects and construction order in C++**: a
  `StaticTask_t` at file scope is zero-initialized before `main()`, which
  is fine — but if you wrap creation in a C++ class constructor at global
  scope, static initialization order between translation units is
  unspecified. Do RTOS object creation explicitly in `app_init()`/`main()`,
  never as a side effect of a global constructor.
- **Assuming static means "no fragmentation risk anywhere"**: static
  allocation only protects the objects you created statically. A single
  remaining `pvPortMalloc()` call elsewhere in the app (a third-party
  library, `malloc()` inside `printf` on some libc configurations) can
  still fragment the heap.

## Cheat sheet

| Dynamic | Static | Notes |
|---|---|---|
| `xTaskCreate(...)` | `xTaskCreateStatic(..., stackBuf, &tcbBuf)` | static never returns NULL |
| `xQueueCreate(len, itemSize)` | `xQueueCreateStatic(len, itemSize, storage, &queueBuf)` | storage = `len * itemSize` bytes |
| `xSemaphoreCreateBinary()` | `xSemaphoreCreateBinaryStatic(&semBuf)` | |
| `xSemaphoreCreateMutex()` | `xSemaphoreCreateMutexStatic(&semBuf)` | |
| `xTimerCreate(...)` | `xTimerCreateStatic(..., &timerBuf)` | |
| `xStreamBufferCreate(size, trig)` | `xStreamBufferCreateStatic(size, trig, storage, &sbBuf)` | |
| `xEventGroupCreate()` | `xEventGroupCreateStatic(&egBuf)` | |
| Failure mode | Never fails at create-time | RAM committed at compile time either way |
| Config | `configSUPPORT_DYNAMIC_ALLOCATION` | `configSUPPORT_STATIC_ALLOCATION` |

## Exercise

1. Convert the producer/consumer pipeline from Level 1 Module 4 to use
   `xTaskCreateStatic` and `xQueueCreateStatic` exclusively. Check the
   linker `.map` file (or `arm-none-eabi-size`/equivalent) before and after
   to see the RAM committed to `.bss`.
2. Set `configSUPPORT_DYNAMIC_ALLOCATION` to 0 and try to build without
   supplying `vApplicationGetIdleTaskMemory`/`vApplicationGetTimerTaskMemory`.
   Read the linker error, then add both callbacks and confirm a clean
   build and successful boot.
3. Deliberately undersize a static task's stack by 75% and trigger the
   deepest call path you can construct for it. With
   `configCHECK_FOR_STACK_OVERFLOW` set to 2, confirm the hook fires;
   with it set to 0, observe (in a scratch/simulation environment only)
   what corrupts instead.
