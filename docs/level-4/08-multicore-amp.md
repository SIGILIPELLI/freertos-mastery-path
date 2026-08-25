# Multi-Core AMP Architectures

**Verification note for this module:** Asymmetric Multi-Processing across
heterogeneous cores (e.g. a Cortex-M4 + Cortex-M0 pairing, or an
application-processor-plus-microcontroller SoC) requires specific
multi-core silicon and vendor inter-processor communication tooling not
available in this environment. Content reflects the documented architecture
of AMP FreeRTOS deployments (as distinct from the SMP architecture already
covered and verified conceptually in Level 3 Module 3) based on vendor and
FreeRTOS-project documentation, not a hands-on build.

## AMP vs. SMP: a fundamentally different relationship between cores

Level 3 Module 3 covered SMP: **one** FreeRTOS scheduler instance, one set
of ready lists, spread across symmetric, identical cores, all running the
same kernel image. AMP is a different architecture entirely: **each core
runs its own, entirely independent FreeRTOS instance** (or, in a
heterogeneous case, one core might run FreeRTOS while another runs Linux, a
different RTOS, or bare-metal code) — there is no shared scheduler, no
shared ready list, and no automatic load balancing between cores. The two
(or more) instances are, from the kernel's point of view, completely
separate systems that happen to share a die and some memory, communicating
only through an explicitly designed inter-processor communication (IPC)
mechanism.

## Why AMP exists: heterogeneous cores, heterogeneous jobs

AMP is the natural fit when the cores themselves aren't interchangeable —
a common real pattern is a beefier core (Cortex-A, or a higher-clocked
Cortex-M) running a general-purpose OS or a feature-rich FreeRTOS
application (networking, UI, application logic) alongside a smaller,
lower-power core (Cortex-M0/M0+) dedicated to a specific always-on,
latency-critical, or power-sensitive job (sensor polling that must continue
even while the main core sleeps, safety-critical motor control that must
not be delayed by anything happening on the "big" core's general-purpose
workload). This is architecturally distinct from SMP's "same job, spread
across identical cores for more throughput" motivation.

## Inter-processor communication: the whole design problem

Since there is no shared kernel state, every piece of coordination between
the two independent FreeRTOS instances must go through an explicit,
designed IPC mechanism — typically one or more of:

- **Shared memory regions** with a defined, agreed-upon protocol (a ring
  buffer, a mailbox structure) — this reintroduces, in a sharper form,
  exactly the cache-coherency hazard Level 3 Module 3 flagged for SMP's
  shared application data: on an AMP system, the two cores may not even be
  running the same kernel or have any automatic memory-model agreement at
  all, so cache maintenance and memory-barrier discipline around shared
  regions is entirely the application's responsibility, with no kernel
  primitive (queue, semaphore) automatically providing it the way it does
  within a single FreeRTOS instance.
- **Hardware mailbox/doorbell peripherals**, where the SoC provides a
  dedicated interrupt-generating register specifically for one core to
  signal the other — this is often the cleanest mechanism because it maps
  naturally onto FreeRTOS's own ISR-to-task deferred-work pattern (Level 3
  Module 7) on the *receiving* core: the mailbox interrupt fires an ISR,
  which does a `...FromISR` give to wake a handling task, exactly the same
  shape as any other ISR-driven design in this course, just with the
  "hardware event" being another core's message instead of a peripheral.
- **The OpenAMP framework** (an open-source, industry-collaborative
  standard specifically for this: it defines a virtio-based transport and a
  "remoteproc" life-cycle model for one core loading, starting, and stopping
  firmware on another) — increasingly the standard answer on AMP-capable
  SoCs (widely used, for example, on NXP i.MX and ST STM32MP1-class parts
  pairing a Cortex-A application processor with a Cortex-M real-time core)
  rather than a fully bespoke shared-memory protocol per project.

## Boot and lifecycle coordination

Unlike SMP, where both cores boot into the same kernel image and start
scheduling together, AMP cores commonly have **asymmetric boot
responsibility**: one core (often the more capable one, or whichever is
wired to boot first by the SoC's boot ROM) is responsible for loading,
verifying, and releasing the other core from reset — remoteproc's life-cycle
model (load firmware image → start → stop → reload) formalizes exactly this
pattern. This has direct implications tying back to Level 4 Module 6's OTA
discussion: updating firmware on an AMP secondary core means the primary
core's OTA mechanism must also handle validating and loading a *second*
firmware image with its own versioning, and the two images' compatibility
with each other (a protocol version mismatch between the two independently
updatable firmwares) becomes a real, additional failure mode dual-bank OTA
alone doesn't address.

## Traps

- **Assuming AMP gets automatic load balancing or work migration the way
  SMP tasks can.** There is no mechanism for a task on one AMP core to
  "migrate" to the other — work placement is a static design decision made
  at build/deployment time, not a runtime scheduling behavior.
- **Reusing raw shared-memory patterns without hardware cache-coherency or
  explicit memory-barrier discipline**, exactly as Level 3 Module 3 warned
  for SMP, but with no shared kernel primitive available to fall back on —
  an AMP shared-memory protocol needs its own explicit cache maintenance
  and memory-ordering design, verified against the specific SoC's actual
  cache topology between the two cores.
- **Treating the two cores' firmware images as independently versioned with
  no compatibility contract.** An IPC protocol version mismatch between
  independently updated primary and secondary firmware is a real, distinct
  OTA failure mode that must be designed for explicitly (a documented,
  versioned IPC protocol, with version negotiation or a required 
  minimum-compatible-version check at startup).
- **Underestimating the boot sequencing dependency.** If the primary core's
  firmware fails before it releases the secondary core from reset, the
  secondary core never starts at all — the fault-tolerance and recovery
  strategies from Level 4 Module 7 need to explicitly account for
  this cross-core startup dependency, not just each core's own internal
  fault handling.
- **Choosing a fully bespoke shared-memory IPC protocol when the SoC vendor
  already supports OpenAMP/remoteproc.** A bespoke protocol means
  reinventing (and independently verifying) message framing, flow control,
  and lifecycle management that a maintained, widely-used framework has
  already solved and tested across many projects.

## Cheat sheet

| Aspect | SMP (Level 3 Module 3) | AMP (this module) |
|---|---|---|
| Kernel instances | One, shared across cores | One per core, independent |
| Core requirement | Symmetric (identical) cores | Can be heterogeneous |
| Work placement | Dynamic (affinity-constrained, but scheduler-chosen) | Static, design-time decision |
| Coordination | Shared kernel data structures + spinlocks | Explicit IPC (shared memory, mailbox, OpenAMP) |
| Cache coherency requirement | Application data only; kernel structures already handled | Everything shared is the application's responsibility |
| Firmware update | One image, one OTA process (Level 4 Module 6) | Potentially two+ images with a version-compatibility contract |
| Common framework | Built into FreeRTOS-Kernel's SMP support | OpenAMP/remoteproc (widely used, not universal) |

## Exercise

1. Read OpenAMP's remoteproc/rpmsg documentation and diagram the life-cycle
   states (offline → loading → running → stopping) for a secondary-core
   FreeRTOS image managed by a primary Linux or FreeRTOS core — identify
   exactly which side is responsible for each transition.
2. Design a mailbox-based IPC protocol for an AMP system where a
   Cortex-M0 core handles always-on sensor polling and a Cortex-M4 core
   handles application logic and networking — specify the message format,
   flow control (what happens if the M4 core is too busy to read a
   message), and a version field for future protocol changes.
3. Extend Level 4 Module 7's fault-tolerance thinking to this AMP design:
   what should the M4 core do if the M0 core's mailbox goes silent, and what
   should the M0 core do if it can no longer reach the M4 core — write out
   both sides' failure-handling behavior explicitly, including whether
   either side should be able to reset the other.
