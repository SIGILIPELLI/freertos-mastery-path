# Long-Term Maintenance & LTS Strategy

This module covers the engineering-process side of shipping FreeRTOS in a
product with a multi-year field lifetime — a genuinely different discipline
from the design and implementation work of every prior module, and one most
engineers only learn by living through a product's full lifecycle once.
Nothing here requires code execution; it's reviewed against
FreeRTOS-Kernel's actual published LTS policy and general embedded
long-term-support practice.

## FreeRTOS-Kernel's own LTS branches

The FreeRTOS project itself maintains **Long Term Support (LTS)** releases
specifically for this reason: mainline FreeRTOS-Kernel development
continues adding features and occasionally changing APIs, but a shipping
product that has already gone through Level 4 Module 4's certification
workflow, or simply wants stability, cannot afford to rebase onto a moving
target every time a new feature ships upstream. An LTS branch instead
receives **backported bug fixes and security patches only**, with a
committed support window, letting a product pin to a specific LTS line and
receive only the fixes it actually needs without absorbing unrelated new
features or API churn. Always check the current FreeRTOS-Kernel LTS
schedule directly (the project's own release page) before committing a
product roadmap to a specific line, since support windows and current LTS
version numbers are maintained externally and do change over time.

## Why "just update to the latest version" is the wrong reflex for a shipped product

For an actively-developed product still in initial bring-up, chasing the
latest FreeRTOS-Kernel release is often reasonable. For a product that has
already shipped — especially one with a certification safety case (Level 4
Module 1/4) built against a specific frozen kernel version and
configuration — an update is not routine maintenance, it's a change to a
certified artifact that may require re-verification, re-testing, and in
some cases re-certification work. The LTS model exists precisely to give
shipped products a path to receive *security and correctness* fixes without
forcing that full re-certification cost for every kernel release.

## Vulnerability management: knowing when a fix is actually needed

A shipped fleet needs a defined process for tracking security advisories
against every dependency it carries — the FreeRTOS-Kernel itself,
FreeRTOS-Plus-TCP (Level 3 Module 9, a network-facing attack surface that
warrants particular vigilance), any TLS/mbedTLS library, and vendor SDK
components. This means:

- **Subscribing to and monitoring the FreeRTOS project's own security
  advisories** (and CVE databases more broadly) for every FreeRTOS-derived
  component actually in the shipped bill of materials, not just the kernel.
- **Maintaining an accurate software bill of materials (SBOM)** per shipped
  product version — you cannot assess whether a new advisory affects a
  fielded device if you don't have an accurate, versioned record of
  exactly what that device is running.
- **A defined severity-triage and patch-deployment process**, connecting
  directly to Level 4 Module 6's OTA infrastructure: a vulnerability with no
  deployable fix path (no OTA capability, or an OTA mechanism the fleet
  can't reliably reach) is a materially worse risk than the same
  vulnerability in a fleet with working, tested OTA.

## Field data and the "the field is different from the lab" reality

No amount of pre-ship testing (Level 4 Module 4's coverage-criteria
discipline included) discovers every fault a large, long-lived fleet will
eventually hit — rare timing windows, unusual environmental conditions, and
long-duration effects (flash wear, subtle memory leaks that only matter
after months of continuous uptime) are disproportionately a
field-observation problem, not a pre-ship-testing one. A mature long-term
maintenance program has:

- **Field telemetry/crash reporting** feeding back into engineering,
  ideally including enough context (a crash dump, a fault-cause code from
  Level 4 Module 7's fault-detection mechanisms) to actually diagnose a
  field-only failure without needing to reproduce it locally.
- **A defined triage process connecting field data to the LTS patch
  cadence** — not every field report warrants an emergency OTA, but a
  process for deciding which do is a real requirement, not optional
  overhead.

## End-of-life and long-lifetime hardware realities

Embedded products frequently outlive the specific silicon they were built
on — a microcontroller part can go end-of-life while a product built on it
is still being manufactured or supported in the field, forcing either a
last-time-buy inventory strategy or a hardware revision requiring its own
re-verification (and, if certified, re-certification) work. This is a
supply-chain and product-management concern as much as an engineering one,
but it directly constrains engineering choices: a hardware abstraction
layer discipline (Level 4 Module 3) that cleanly separates application code
from a specific silicon's peripherals materially reduces the cost of a
forced hardware revision partway through a product's field life.

## Traps

- **Treating LTS patch adoption as optional "if it ain't broke."**
  Security-relevant backports specifically exist because something *is*
  broken from a security standpoint, even if it hasn't yet been exploited
  against your fleet — deferring adoption indefinitely accumulates real,
  quantifiable risk.
- **No accurate SBOM per shipped version.** Without this, assessing a new
  CVE's relevance to already-fielded devices becomes guesswork rather than a
  quick, confident lookup — this is foundational infrastructure that pays
  for itself the first time a serious advisory needs a fast answer.
- **Assuming a certified product can absorb kernel updates the same way an
  uncertified one can.** As Level 4 Module 1 and Module 4 established, a
  kernel version is part of a frozen certified configuration — LTS patches
  still need to be evaluated against the specific safety case's
  re-verification requirements, not applied as routine maintenance.
- **No field telemetry or crash-reporting path**, leaving the engineering
  team blind to exactly the class of fault (rare, timing-dependent,
  long-duration) that pre-ship testing structurally under-detects.
- **Ignoring silicon end-of-life planning until forced by a supplier
  notice.** A HAL-disciplined codebase (Level 4 Module 3) makes a forced
  hardware revision materially cheaper — but only if that discipline was
  followed from early in the product's life, not retrofitted under
  last-time-buy time pressure.

## Cheat sheet

| Concern | Mechanism | Connects to |
|---|---|---|
| Stability without feature churn | FreeRTOS-Kernel LTS branches | — |
| Security patch tracking | Advisory monitoring + accurate SBOM | Level 3 Module 9 (network attack surface) |
| Deploying a fix to the field | OTA infrastructure | Level 4 Module 6 |
| Certified-product update discipline | Re-verification against the frozen safety case | Level 4 Module 1, Module 4 |
| Field-only fault discovery | Telemetry/crash reporting | Level 4 Module 7's fault-detection mechanisms |
| Silicon end-of-life resilience | HAL/layering discipline | Level 4 Module 3 |

## Exercise

1. Look up FreeRTOS-Kernel's current published LTS release(s) and support
   window(s) directly from the project's own release documentation, and
   write a short summary of what backporting policy currently applies (bug
   fixes only vs. security fixes vs. both) for the current LTS line.
2. Draft a minimal SBOM template for a hypothetical shipped product using
   FreeRTOS-Kernel, FreeRTOS-Plus-TCP, and one vendor SDK — specify exactly
   what fields (component name, version, source/commit, license) you'd need
   to answer "does CVE-XXXX-YYYY affect any device we've shipped" quickly.
3. Write a short field-fault-to-patch triage policy: given a crash report
   from the field with a fault code from a Level 4 Module 7-style
   fault-detection mechanism, outline the decision process for whether it
   warrants an emergency OTA (Level 4 Module 6), a scheduled patch in the
   next release, or further investigation before any action.
