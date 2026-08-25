# Capstone — Production RTOS Product

This capstone pulls together the entire course — Level 1's primitives,
Level 2's patterns, Level 3's kernel internals and measurement discipline,
and every Level 4 module — into one coherent product design: a
network-connected, field-updatable industrial sensor gateway with a
credible (if not fully executed) path to a safety/security posture
appropriate for real deployment. Like Module 10 in Level 3, this is a
design exercise to build and defend, not something pre-built here — the
value is in doing the integration work yourself and hitting the real
trade-offs each module only discussed individually.

## Product scope

An industrial sensor gateway: reads several sensors, runs local control/
alarm logic, buffers and forwards data over a network link, supports secure
OTA updates, and is expected to run unattended in the field for years.

## Architecture layer-by-layer, module by module

**Kernel and hardware foundation** (Level 3): a real target's port
selected and verified per Module 6's checklist; kernel internals knowledge
(Module 1) informing priority and ready-list-scale decisions; context-switch
overhead (Module 2) and ISR-latency-adjacent measurement (Module 7) used to
validate the design's timing budget has real margin, not just hoped-for
margin; SMP (Module 3) evaluated and consciously accepted or rejected based
on whether the target hardware and workload actually benefit from it, not
adopted by default; MPU (Module 4) enabled for the sensor-reading and
network-facing tasks specifically, since those are the tasks most exposed
to untrusted or noisy input.

**Task/priority design** (Levels 1-3 patterns):

| Task | Priority | Role | Isolation |
|---|---|---|---|
| Sensor ISR handler | Highest | Deferred-work handoff (L3M7) | MPU-restricted |
| Control/alarm logic | High | Time-critical decisions | MPU-restricted |
| Network/IP task (L3M9) | Medium-high | FreeRTOS-Plus-TCP stack | Standard (stack-owned) |
| OTA task (L4M6) | Low, deliberately | Background download/flash-write | MPU-restricted, sole flash-write owner |
| Logging gatekeeper (L2M8) | Low | Sole owner of persistent/remote logging | — |
| Supervisor/watchdog task (L4M7) | Low, but watchdog-fed only on full-system health | Fault detection | — |

**Timing analysis** (Level 4 Module 2): the task set above is run through
RMA — periods and WCET budgets for the sensor and control tasks assigned
first, utilization checked against the Liu & Layland bound, and the exact
response-time test applied to any task that fails the sufficient bound,
with kernel/context-switch overhead explicitly included in each task's
effective `Cᵢ` rather than assumed away.

**Certification posture** (Level 4 Modules 1 and 4): a deliberate, explicit
decision recorded — either "this product does not require certification to
IEC 61508/ISO 26262 and standard FreeRTOS plus this course's own testing
discipline is the accepted posture," or "this product's control/alarm path
specifically requires certification, and a certified kernel (SAFERTOS) or
an independently-built safety case for that specific subsystem is required"
— made consciously per Module 4's workflow, not defaulted into.

**Security** (Level 4 Module 5): if the target hardware supports
TrustZone-M, firmware-signature verification and any device credentials/
keys live in the Secure world behind a minimal, carefully validated NSC
interface; the bulk of the sensor/network/OTA application logic runs
Non-secure, isolated from each other additionally by MPU.

**OTA** (Level 4 Module 6): dual-bank partitioning, cryptographic signature
verification anchored in the same trust root as secure boot, a self-test/
rollback window before committing to a new image, and the OTA task itself
built as a gatekeeper-pattern sole owner of flash writes.

**Fault tolerance** (Level 4 Module 7): per-task supervision with a
watchdog fed only on whole-system health evidence; a documented per-subsystem
recovery policy (contained restart for the OTA or network subsystems,
full reset for anything touching the control/alarm path's own corruption);
graceful degradation defined explicitly for "network down" (keep local
control/alarm running, buffer data locally) versus "sensor feed silent"
(enter a defined safe state, per Module 7's control-loop example).

**Long-term maintenance** (Level 4 Module 9): the shipped kernel version
pinned to a specific FreeRTOS-Kernel LTS line, an SBOM maintained per
shipped firmware version, and a documented triage process connecting field
crash/fault reports (from the fault-detection mechanisms above) to the OTA
patch pipeline.

## What to actually produce for this capstone

1. A written architecture document following the shape above, filled in
   with real, specific decisions (not placeholders) for a chosen target
   platform.
2. A task/priority table with real period and WCET estimates, and the RMA
   utilization/response-time calculation (Level 4 Module 2) worked by hand
   or in a spreadsheet, showing the design is schedulable as specified.
3. A working implementation of at least the sensor-handler → control-logic
   → logging-gatekeeper chain on real or simulated (POSIX-port, with Module
   7's caveats in mind) hardware, instrumented in the style of Level 3
   Module 10's project.
4. A one-page fault-domain and recovery-policy table (Level 4 Module 3 and
   7's format) covering every major subsystem.
5. An explicit, justified certification-posture decision (Level 4 Module 4),
   even if the decision is "not required for this product," with the
   reasoning documented, not just the conclusion.

## Traps carried through the whole course

- **Skipping the measurement step and trusting the design "looks right."**
  Every level of this course demonstrated real gaps between an
  intuitively-reasonable design and its measured or formally-analyzed
  behavior (Level 3 Module 7's real hang, Level 3 Module 8's O(1)
  switch-cost confirmation, Level 4 Module 2's WCET-vs-measured-maximum
  distinction) — a capstone that skips verification repeats this course's
  own most consistently repeated lesson.
- **Treating any one Level 4 module's coverage as sufficient depth for a
  real regulated or high-consequence product.** Every Level 4 module was
  explicit that it's orientation and technically accurate review, not a
  substitute for a qualified specialist (a functional-safety assessor, a
  security auditor, a WCET tool vendor) on a real product with real
  consequences.
- **Designing the "interesting" subsystems (control logic, networking) in
  detail and treating fault tolerance, OTA, and maintenance as an
  afterthought.** This course's Level 4 sequence exists specifically
  because these are exactly the areas that separate a working demo from a
  product that survives years in the field.
- **Choosing SMP, TrustZone, or AMP because they're available on the
  target, rather than because the workload and risk profile actually call
  for them.** Every one of these adds real design and verification
  complexity (Level 3 Module 3, Level 4 Module 5, Level 4 Module 8 all
  covered specific, non-obvious hazards each introduces) — use them because
  the requirements demand it, not by default.

## Stretch goals

1. Build and measure the sensor-handler → control-logic → gatekeeper chain
   on real target hardware (not just the POSIX simulator), and compare the
   real measured context-switch and dispatch-latency numbers against
   Level 3 Module 2 and Module 7's simulator-derived figures — quantify
   exactly how far off the simulator was, closing the loop this course
   opened in Level 2 Module 5.
2. Implement the full dual-bank OTA state machine from Level 4 Module 6 for
   your chosen target, including the self-test/rollback window, and
   deliberately test power-loss at three different points in the update
   sequence to confirm the device never bricks.
3. If your target supports TrustZone-M, implement a minimal Secure-world
   service (firmware signature verification is the natural choice) behind
   an NSC interface, and write the parameter-validation code the Secure
   side needs to treat the Non-secure caller as untrusted.
4. Take your task set's RMA analysis and deliberately violate it (add a
   task that pushes utilization over the schedulability bound), observe
   what actually happens under the resulting overload on real or simulated
   hardware, and write up how the observed failure mode compares to what
   the formal analysis predicted.
5. Write a one-page certification gap analysis for your capstone design
   against IEC 61508 SIL 2 (a moderate, realistic target for an industrial
   product) — identify, honestly, which of Level 4 Module 4's workflow
   steps your capstone actually satisfies and which remain open work for a
   real certification effort.
