# Certification Workflows (IEC 61508 / ISO 26262)

**Verification note for this module:** certification workflows are
process and documentation disciplines defined by published international
standards — there is no code to build or run for this topic. Content here
reflects the actual structure of IEC 61508 and ISO 26262 as publicly
documented, presented for accurate understanding of the workflow shape, not
as a substitute for the standards themselves or for guidance from a
qualified functional-safety assessor, which any real certification effort
requires.

## Two related but distinct standards

**IEC 61508** ("Functional Safety of Electrical/Electronic/Programmable
Electronic Safety-related Systems") is the generic, industry-agnostic
parent standard, using **Safety Integrity Levels (SIL 1-4)**, SIL 4 being
the most stringent. **ISO 26262** ("Road Vehicles — Functional Safety") is
the automotive-specific derivative, using **Automotive Safety Integrity
Levels (ASIL A-D)**, ASIL D being the most stringent, plus "QM" (quality
managed — no specific safety integrity requirement). Other sectors have
their own IEC-61508-derived sector standards (e.g. IEC 62304 for medical
device software, IEC 61511 for process industry) with the same underlying
risk-based structure adapted to sector-specific hazards.

## The core workflow shape, common across these standards

1. **Hazard and risk analysis** — before any design work, identify what can
   go wrong at the system level (not the software level yet) and how
   severe/likely/controllable each hazard is. ISO 26262 formalizes this as
   HARA (Hazard Analysis and Risk Assessment), producing the ASIL rating
   itself — the integrity level is a *risk assessment output*, not a
   starting assumption.
2. **Safety requirements derivation** — the hazard analysis output is
   translated into concrete, testable safety requirements at the system,
   then software, level. A software safety requirement traces back to a
   specific hazard it mitigates — an untraceable requirement is a
   certification gap.
3. **Design and implementation under a qualified process** — coding
   standards (MISRA C is the dominant one for automotive/embedded C,
   restricting or banning undefined-behavior-prone or hard-to-analyze
   language constructs), static analysis, and a development process with
   defined, auditable gates between phases.
4. **Verification and validation** — this is where the RTOS-specific work
   from this course concentrates: unit/integration testing with defined
   coverage criteria (often including MC/DC — Modified Condition/Decision
   Coverage — at higher integrity levels, a materially stronger criterion
   than simple statement or branch coverage), and, per Level 4 Module 2,
   formal timing analysis demonstrating schedulability and bounded WCET
   where the standard requires deterministic timing evidence.
5. **The safety case** — the full traceability chain (hazard → requirement
   → design → code → test → result) assembled into a documented argument,
   reviewed by an independent assessor, that the system meets its target
   integrity level. This is the deliverable a certification body actually
   evaluates — not the code in isolation.

## Where FreeRTOS-specific decisions plug into this workflow

- **Kernel choice** (Level 4 Module 1): using SAFERTOS or another
  independently certified kernel provides pre-built evidence for the
  kernel-level portion of the safety case; using standard FreeRTOS means the
  project's own safety case must independently justify the kernel's
  behavior — a substantially larger and more expensive undertaking, and one
  reason certified kernels exist commercially at all.
- **Scheduling analysis** (Level 4 Module 2): RMA/exact schedulability
  results and static WCET bounds are exactly the kind of deterministic
  timing evidence higher integrity levels require — "it worked in testing"
  is not acceptable evidence at ASIL C/D or SIL 3/4.
- **MPU-based fault containment** (Level 3 Module 4): can serve as evidence
  toward freedom-from-interference requirements — ISO 26262 in particular
  has explicit requirements around demonstrating that software elements of
  different ASILs coexisting on one processor don't interfere with each
  other, and hardware memory protection is a standard technique cited for
  this.
- **Coding standard compliance (MISRA C)**: static analysis tooling
  (commercial MISRA checkers) is run as part of the verification workflow,
  and deviations from the coding standard must be individually justified and
  documented, not silently ignored — a MISRA violation isn't automatically
  disqualifying, but an *undocumented* one is a process gap.

## ASIL decomposition: a specific ISO 26262 technique worth knowing

ISO 26262 permits **ASIL decomposition** — splitting a high-ASIL requirement
across multiple, sufficiently independent architectural elements such that
each element individually needs a lower ASIL, provided the independence
between them is itself argued and evidenced (e.g., an ASIL D requirement met
by two independent ASIL B elements with a demonstrated absence of common-
cause failure between them). This is directly relevant to RTOS-based
architecture decisions: whether a mixed-criticality design (some tasks ASIL
D, others QM, on one processor) is viable depends heavily on whether MPU-
based or other technical isolation can support the required independence
argument — this is a real, non-trivial technical judgment call requiring a
qualified safety assessor's involvement, not something a generic course
module can settle in the abstract for a specific product.

## Traps

- **Treating certification as a testing phase bolted on at the end.**
  Every credible certification workflow assumes safety requirements exist
  *before* implementation and drive design decisions — retrofitting a safety
  case onto already-complete code is dramatically more expensive and often
  reveals architectural gaps (Level 4 Module 3's fault-domain and isolation
  decisions) that require real rework, not just paperwork.
- **Assuming a SIL/ASIL rating is chosen rather than derived.** The
  integrity level comes out of the hazard/risk analysis; picking a target
  level first and working backward inverts the actual required process and
  is not how a credible safety case is built.
- **Confusing coverage criteria.** MC/DC (required at higher integrity
  levels for many standards) is a substantially stronger requirement than
  branch coverage — a test suite reporting "100% branch coverage" has not
  automatically satisfied an MC/DC requirement, and treating them as
  interchangeable is a common, serious gap.
- **Assuming ASIL decomposition is a free way to avoid a high-ASIL
  component.** The independence argument between decomposed elements is
  itself substantial engineering and assessment work — decomposition is a
  legitimate technique, not a shortcut, and an unconvincing independence
  argument invalidates the whole decomposition.
- **Treating this module, or any single course, as sufficient preparation
  to lead a real certification effort.** Certification workflows involve
  qualified functional-safety assessors, sector-specific expertise, and
  current standard text — this module is orientation, not qualification.

## Cheat sheet

| Concept | IEC 61508 | ISO 26262 |
|---|---|---|
| Integrity levels | SIL 1-4 | ASIL A-D (+QM) |
| Level determination | Risk-based hazard analysis | HARA (Hazard Analysis and Risk Assessment) |
| Coding standard (common practice) | Sector-dependent | MISRA C (dominant in automotive) |
| Coverage at higher levels | MC/DC often required | MC/DC often required |
| Splitting a high-integrity requirement | Not standard's own mechanism | ASIL decomposition (with independence argument) |
| RTOS timing evidence | RMA/schedulability, WCET (L4M2) | Same |
| Kernel-level evidence shortcut | Certified kernel (e.g. SAFERTOS, L4M1) | Same |

## Exercise

1. Find a publicly available ISO 26262 HARA example (several are published
   in academic/industry papers) and trace, step by step, how a specific
   hazard leads to a specific ASIL rating — identify the severity,
   exposure, and controllability factors that combine into the rating.
2. Take one of this course's own examples (Level 2's gatekeeper logging
   task) and write a plausible software safety requirement it might need
   to satisfy in a hypothetical ASIL B context, then identify what test
   evidence (including coverage criterion) would be needed to verify it.
3. Research MISRA C's actual rule categories (mandatory, required,
   advisory) and find three specific rules that would affect FreeRTOS
   application code patterns used earlier in this course (e.g. dynamic
   memory allocation patterns, recursive functions) — explain the safety
   rationale behind each rule you pick.
