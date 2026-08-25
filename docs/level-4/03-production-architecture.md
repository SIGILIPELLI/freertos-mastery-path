# Production Firmware Architecture

This module is architectural review and engineering-practice guidance, not a
buildable code exercise — the patterns below are widely used, industry-
standard structuring approaches for shipping FreeRTOS firmware, presented
for careful design review rather than execution in this environment.

## Layered architecture: separating what changes from what doesn't

Production FreeRTOS firmware almost universally converges on a layered
structure, for the same reason this pattern recurs across the embedded
industry: it isolates the parts of the codebase that change per-product
(application logic) from the parts that change per-silicon-revision
(drivers) and the parts that rarely change at all (the RTOS itself).

```
┌─────────────────────────────────────────┐
│  Application layer                       │  ← business logic, task designs
│  (this course's Level 1/2 patterns)      │     from Levels 1-2
├─────────────────────────────────────────┤
│  Middleware / services layer             │  ← FreeRTOS-Plus-TCP (L3M9),
│  (comms, logging, OTA, storage)          │     logging gatekeeper, OTA (L4M6)
├─────────────────────────────────────────┤
│  Hardware Abstraction Layer (HAL)        │  ← vendor SDK / your own thin
│                                           │     wrapper over peripheral regs
├─────────────────────────────────────────┤
│  FreeRTOS kernel + port                  │  ← Level 3's internals, unmodified
└─────────────────────────────────────────┘
```

The critical discipline this diagram implies: application code should call
into the middleware/HAL layers through **stable interfaces it owns**, not
directly into vendor SDK functions scattered throughout task code. A team
that calls `HAL_UART_Transmit()` (a specific vendor's HAL call) directly from
a dozen application tasks has coupled its entire application layer to one
vendor's API; a team that instead defines its own
`platform_uart_write(buf, len)` and implements it once against the vendor
HAL can retarget hardware, mock the interface for host-side testing, or
swap vendors, by changing one file instead of a dozen call sites.

## Module boundaries and the "one gatekeeper per shared resource" discipline at scale

Level 2 Module 8's gatekeeper pattern and Level 2 Module 8's
producer-filter-consumer chains are not just individual-module techniques —
at production scale they become the primary architectural tool for defining
module boundaries. A well-architected production firmware typically has a
short, explicit list of "owned resources" (UART, flash, the network stack's
IP task from Level 3 Module 9, a shared configuration store) each with
exactly one task or subsystem responsible for it, and every other module
interacts with that resource exclusively through the owning module's queue-
based API — never by reaching into shared globals or a second mutex-guarded
path to the same hardware.

## Build system and configuration management for multiple product variants

Shipping firmware rarely means one binary — most products ship variant
builds (different hardware revisions, feature-gated SKUs, regional radio
configurations) from one codebase. Production architecture handles this with:

- **Compile-time feature flags** for genuinely different hardware
  (different sensor part numbers, different memory sizes) where runtime
  branching would waste flash/RAM on unused code paths.
- **A single `FreeRTOSConfig.h` per hardware variant**, not per feature —
  since Level 4 Module 1 already established that kernel configuration is
  part of a certified artifact's frozen scope where certification applies;
  treating it as a place for arbitrary feature toggles undermines that
  discipline even in non-certified products.
- **Reproducible builds** — pinned toolchain version, pinned vendor SDK
  version, and a build system that produces bit-identical output from the
  same source tree, which matters enormously for both certification
  evidence (Level 4 Module 4) and for being able to reproduce a field bug
  against the exact binary that shipped.

## Fault domains and the blast radius of a single task crash

A core production-architecture question this course hasn't directly
addressed yet: what happens when one task genuinely misbehaves (a stack
overflow the MPU didn't catch, an infinite loop, a malformed input crashing a
parser)? Architecting for this means deciding, per subsystem, whether a
crash should:

- **Stay contained** — an MPU-restricted task (Level 3 Module 4) faulting
  cleanly, caught by the fault handler, with that one subsystem restarted
  (`vTaskDelete` + recreate) while the rest of the system continues.
- **Escalate to a full system reset** — appropriate when the failure mode
  indicates the fault could be systemic (kernel data structure corruption,
  a stack overflow that already wrote past its guard region) rather than
  contained to one task's own state.

This decision needs to be made deliberately per subsystem during
architecture, not discovered accidentally in the field — Level 4 Module 7's
fault-tolerant design patterns build directly on this foundation.

## Traps

- **Letting application code call vendor SDK functions directly, scattered
  throughout task bodies.** This is the single most common production
  architecture mistake — it couples the whole codebase to one vendor/silicon
  revision and makes host-side testing (mocking the HAL) impossible without
  a large refactor discovered under time pressure during a hardware
  transition.
- **Treating `FreeRTOSConfig.h` as a general-purpose feature-flag file.**
  As noted above, this conflicts with the certification-relevant discipline
  from Level 4 Module 1 even in non-certified products, and makes it harder
  to reason about which configuration a given build actually used.
- **No defined fault domain strategy.** A codebase where every task crash's
  blast radius is "whatever happens to happen" (sometimes contained,
  sometimes a full lockup, decided by accident rather than design) is a
  production reliability risk that's expensive to retrofit later.
- **Non-reproducible builds.** Not being able to reproduce the exact binary
  that's failing in the field (due to unpinned toolchain/SDK versions) turns
  every field bug investigation into extra, avoidable work.
- **Skipping module boundary discipline "because it's just one more task
  that needs the resource."** Every exception to the one-owner-per-resource
  rule reintroduces exactly the races and debugging difficulty the
  gatekeeper pattern (Level 2 Module 8) was adopted specifically to avoid.

## Cheat sheet

| Layer | Owns | Should never contain |
|---|---|---|
| Application | Task designs, business logic | Direct vendor SDK/register calls |
| Middleware/services | Comms, logging, OTA, storage | Application-specific business rules |
| HAL | Thin wrapper over vendor SDK/registers | Business logic of any kind |
| Kernel + port | Scheduling, IPC primitives (Level 3) | Application or vendor-specific code |

| Production concern | Tool/pattern |
|---|---|
| Multi-variant builds | Compile-time flags per hardware variant, one config per variant |
| Reproducibility | Pinned toolchain + SDK versions, deterministic build system |
| Fault containment | MPU (L3M4) + deliberate per-subsystem restart-vs-reset decision |
| Resource ownership | One gatekeeper task per shared resource (L2M8), no exceptions |

## Exercise

1. Take a codebase you've written earlier in this course (or a real project)
   and audit it for direct vendor-SDK calls scattered through application
   task code — refactor at least one shared resource behind a HAL interface
   your own code owns, and note how much of the application layer had to
   change (ideally: none, beyond the call sites).
2. Design a fault-domain policy document (even a one-page table is enough)
   for a hypothetical product with a sensor task, a network task, and a
   display task — for each, decide and justify: contained restart, or full
   system reset, on an unrecoverable fault.
3. Write out what "reproducible build" would concretely require for a real
   project you have access to (toolchain version pinning mechanism, vendor
   SDK version pinning, build system determinism) and identify any gaps
   against that standard in the current setup.
