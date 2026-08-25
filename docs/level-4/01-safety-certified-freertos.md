# Safety-Certified FreeRTOS (SAFERTOS)

**Verification note for this module:** SAFERTOS is a separate, commercially
licensed codebase from WITTENSTEIN high integrity systems (HIS) — it is not
open source, not part of the FreeRTOS-Kernel repository used throughout this
course, and there is no way to build or run it in this environment. This
module is a careful, technically accurate description of SAFERTOS's design
and certification posture based on its public documentation and safety-case
literature, not hands-on verification. Treat every certification-scheme
detail here as a starting point for reading the actual current standard and
your certification body's guidance, not a substitute for it.

## SAFERTOS is not "FreeRTOS with a flag turned on"

This is the single most important framing correction for this module.
SAFERTOS is a **separate codebase**, independently developed by HIS
specifically to be certifiable, that shares FreeRTOS's API and scheduling
concepts (tasks, queues, semaphores, mutexes with priority inheritance) so
that application code and design knowledge transfer, but the kernel
implementation itself is different, smaller in scope, and developed under a
formal safety-certified software development lifecycle from the start —
not standard FreeRTOS retroactively certified. `configENABLE_MPU`
(Level 3 Module 4) and standard FreeRTOS-MPU builds are a *different*,
separate mechanism for bug containment; they do not make standard FreeRTOS
itself a certified kernel.

## What "certified" actually means here

A certified RTOS ships with a **safety case** — a structured argument,
backed by evidence, that the software meets a specific standard's
requirements for a specific safety integrity level. For SAFERTOS this
typically includes:

- **Requirements traceability**: every kernel requirement traced forward to
  design, to code, to test — and every test traced backward to the
  requirement it verifies. This is fundamentally different from typical
  open-source kernel development, where tests exist but a full bidirectional
  traceability matrix generally does not.
- **Independent verification**: SAFERTOS's development process includes
  independent (organizationally separate from the developers) verification
  activity, a specific requirement of standards like IEC 61508 at higher
  integrity levels.
- **Certification to specific standards**, historically including IEC 61508
  (SIL 3, industrial), ISO 26262 (ASIL D, automotive — the highest
  automotive integrity level), and applicable pieces of avionics-adjacent
  guidance depending on the specific product variant and target — the exact
  current certifications and target-processor variants should always be
  confirmed directly against WITTENSTEIN's current published certificates,
  since certification scope is processor/compiler/toolchain-specific and
  does change over product versions.
- **A frozen, versioned, documented configuration** — unlike standard
  FreeRTOS's actively evolving `FreeRTOSConfig.h`-driven flexibility, a
  certified deployment locks a specific kernel version, specific
  configuration, specific compiler and compiler flags, and specific target
  processor as *part of the certified artifact* — changing any of them
  without re-certification work invalidates the safety case.

## API-level compatibility, implementation-level difference

```c
// This looks identical to every prior module's code in this course:
xTaskHandle sensorHandle;
xTaskCreate(vSensorTask, "Sensor", STACK_SIZE, NULL, 2, &sensorHandle);
xQueueHandle dataQueue = xQueueCreate(10, sizeof(SensorData_t));
xSemaphoreHandle configMutex = xSemaphoreCreateMutex();
```

The point of this similarity is deliberate and valuable — an engineering
team that has designed a system against standard FreeRTOS's task/queue/mutex
model can move that *design* to SAFERTOS with much less rework than
switching to an unrelated safety RTOS entirely. But it is a mistake to
assume this extends to binary or even source-level interchangeability:
SAFERTOS has its own restricted feature set (deliberately smaller and more
constrained than full FreeRTOS — some dynamic-allocation patterns and
less-deterministic features are intentionally excluded specifically because
unbounded behavior is hard to certify), its own configuration mechanism, and
its own qualified toolchain requirements, and application code must be
re-verified against SAFERTOS specifically, not merely recompiled.

## Why some FreeRTOS features are deliberately absent

Certification bodies generally penalize (or outright disallow, depending on
integrity level) unbounded or non-deterministic behavior, because a safety
case needs to bound worst-case behavior with evidence, not "usually
fine." This is why safety-certified kernels commonly restrict or exclude:

- **Fully dynamic memory allocation** at runtime after startup — favoring
  static or pool-based allocation with bounded worst-case behavior
  (Level 2's `configSUPPORT_STATIC_ALLOCATION` static-allocation APIs are the
  standard-FreeRTOS feature this maps most directly onto, though SAFERTOS's
  own allocation model is its own separate design, not simply "FreeRTOS with
  static allocation forced on").
- **Certain flexible/late-bound configuration paths** that would otherwise
  require re-verifying an unbounded space of possible configurations, in
  favor of a small number of qualified, pre-verified configurations.
- **Recursive or otherwise less-analyzable control flow** in kernel-internal
  code, in favor of code shaped for easier static analysis and worst-case
  execution time bounding (Level 4 Module 2).

## Traps

- **Assuming "runs the same code as FreeRTOS" means "inherits FreeRTOS's
  certification."** Standard FreeRTOS, including FreeRTOS-MPU, is not
  independently safety-certified to IEC 61508/ISO 26262 — SAFERTOS is a
  separate product specifically built and maintained for that purpose. A
  design that needs a certified kernel needs SAFERTOS (or another
  purpose-certified RTOS), not "FreeRTOS plus enough of our own testing."
- **Modifying a certified configuration or toolchain and assuming the
  safety case still holds.** The certificate covers a specific frozen
  combination of kernel version, configuration, compiler, compiler flags,
  and target processor — changing any of these is, from a certification
  standpoint, a different, uncertified system until re-verified.
- **Treating this module (or any secondary source) as authoritative for
  current certification scope, standard versions, or covered processor
  variants.** These details are commercially maintained and do change —
  always confirm directly against WITTENSTEIN's current published
  certificates and your own certification body's requirements before
  making a real product decision.
- **Assuming certification alone proves your *application* is safe.** A
  certified kernel provides evidence about the kernel's own behavior — your
  application code, its interaction with the kernel, and the overall system
  safety case (Level 4 Module 4) are separate, additional certification
  work the kernel certificate does not cover by itself.
- **Underestimating the toolchain qualification requirement.** Higher
  integrity levels often require a *qualified* compiler/toolchain
  (evidence the compiler itself doesn't introduce miscompilation risk) —
  swapping compiler versions on a certified project is not a routine
  update, it's a certification-relevant change.

## Cheat sheet

| Aspect | Standard FreeRTOS | SAFERTOS |
|---|---|---|
| Codebase | Open source, actively evolving | Separate, commercially licensed, independently developed |
| API shape | Familiar to this whole course | Deliberately similar, for design/skill transfer |
| Feature set | Full — dynamic allocation, broad config space | Deliberately restricted for certifiability |
| Certification | None (kernel itself is not certified) | IEC 61508 / ISO 26262 and others, depending on product/target — confirm current scope directly |
| Configuration | Flexible, frequently changed | Frozen, versioned, part of the certified artifact |
| Toolchain | Any conforming compiler | Qualified toolchain often required at higher integrity levels |
| Verified in this module | N/A | N/A — documentation/safety-case review only, no hands-on build |

## Exercise

1. Locate WITTENSTEIN's current published SAFERTOS certification
   documentation and identify exactly which standards, integrity levels, and
   target processor variants are currently certified — note how specific
   the certified scope is (processor family, sometimes even specific
   silicon revisions) compared to how broadly standard FreeRTOS runs.
2. Take a design you built earlier in this course (Level 2 Module 8's
   gatekeeper pattern is a good candidate) and write out, feature by
   feature, which parts would need to change or be re-verified to move it
   onto a certified kernel with restricted dynamic allocation.
3. Research and summarize, in your own words, what "toolchain
   qualification" concretely requires for a compiler used in an IEC
   61508 SIL 3 or ISO 26262 ASIL D project, and why swapping a compiler
   version on such a project is a certification event, not a routine
   upgrade.
