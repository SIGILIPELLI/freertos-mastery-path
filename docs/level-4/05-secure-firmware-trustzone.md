# Secure Firmware & TrustZone

**Verification note for this module:** ARM TrustZone-M requires an
ARMv8-M target (Cortex-M23/M33/M55 and similar) with real secure/non-secure
memory partitioning configured through vendor-specific tooling — there is no
TrustZone-capable simulation available in this environment and nothing here
was built or run. Content is a technically accurate review of the TrustZone-M
architecture and its FreeRTOS integration model (FreeRTOS-Kernel's secure-side
port plus the secure/non-secure interface libraries used with it), based on
ARM's own architecture documentation.

## TrustZone-M vs. MPU: a different, stronger boundary

Level 3 Module 4 was explicit that FreeRTOS-MPU is bug containment between
*cooperating* tasks, not a security boundary against a hostile actor.
TrustZone-M is architecturally different and stronger: it partitions the
**entire processor and memory map** into a Secure world and a Non-secure
world, enforced by hardware at every bus transaction — code running in the
Non-secure world, even with full "privileged" rights *within* that world,
cannot read or execute Secure-world memory at all; the hardware simply
doesn't route the access. This is a fundamentally different guarantee than
MPU regions, which all live within one privilege/security context.

## The two-FreeRTOS-instance (or one-secure-service) architecture

There are two common architectural shapes for FreeRTOS on TrustZone-M
hardware:

1. **A full Non-secure FreeRTOS application** (the common case — most
   application logic, most tasks, the full scheduler, runs Non-secure) that
   calls into a **minimal Secure-side firmware** for specific
   security-critical services only (key storage, cryptographic operations,
   secure boot verification) through a narrow, hardware-enforced call
   gateway (the **Non-Secure Callable**, NSC, region — the *only* addresses
   the Non-secure world is permitted to call into the Secure world through).
2. **FreeRTOS running on both sides** — a Secure-side FreeRTOS instance
   managing security-critical tasks and a separate Non-secure FreeRTOS
   instance for the application, communicating only through the same
   NSC-gated interface. FreeRTOS-Kernel's official TrustZone support
   (`secure_context.c`/`secure_heap.c`/`secure_init.c` on ARMv8-M ports)
   specifically supports Non-secure tasks calling into Secure-side
   functions safely, managing a per-task Secure-side context/stack so that
   Secure-side state isn't corrupted by concurrent Non-secure task calls.

## What the kernel-level TrustZone integration actually manages

The Non-secure kernel's context-switch mechanism (Level 3 Module 2's
PendSV-based switch, on a TrustZone-aware ARMv8-M port) has an added
responsibility: if a Non-secure task is preempted *while it has an
outstanding call into Secure-side code*, the Secure-side call context for
that specific task must be preserved and correctly restored on that task's
next switch-in — otherwise one task's Secure-side call state could leak
into or corrupt another's. FreeRTOS's `secure_context.c` manages exactly
this: an allocated Secure-side stack and context slot per Non-secure task
that ever calls into Secure code, switched by the kernel's own
context-switch hook in lockstep with the Non-secure switch.

```c
// Conceptual application-level call shape — actual API is vendor/SDK-specific
// This runs in the Non-secure world:
uint32_t signature_valid = Secure_VerifyFirmwareSignature(imageAddr, imageLen);
// The NSC veneer at a fixed address is the ONLY entry point the hardware
// permits the Non-secure world to call into the Secure world through —
// any other Secure-world address is simply inaccessible from here.
```

## Secure boot: the root the rest depends on

TrustZone-based security is only as strong as the boot chain that
establishes it. A typical secure boot chain: an immutable (mask-ROM or
write-protected) first-stage bootloader verifies the signature of the
Secure-world firmware image before executing it; the Secure-world firmware
then verifies and launches the Non-secure application image, establishing
the security world partition (via SAU/IDAU — Security Attribution Unit /
Implementation Defined Attribution Unit register configuration, done once at
boot and typically locked from further modification) before handing control
to it. If any stage in this chain is skippable or its verification is
bypassable, the entire TrustZone partition built on top of it provides no
real security guarantee regardless of how correctly the application-level
code uses it.

## Traps

- **Treating TrustZone as a drop-in replacement for MPU-based task
  isolation (Level 3 Module 4).** They solve different problems at
  different granularity — TrustZone partitions the whole system into two
  worlds; MPU partitions tasks *within* one world from each other. A
  production design commonly uses both: TrustZone for the
  security-critical/non-critical split, MPU within the Non-secure world for
  task-level bug containment.
- **Putting too much in the Secure world "to be safe."** Every line of code
  in the Secure world expands the attack surface that must be reviewed with
  the highest scrutiny, and the NSC interface must correctly validate every
  parameter passed from the (untrusted, from the Secure world's point of
  view) Non-secure caller — a large, loosely-audited Secure-world codebase
  undermines the entire point of having a minimal, more easily verified
  trusted computing base.
- **Not validating parameters passed across the NSC boundary.** The NSC
  gateway enforces *where* a call can enter, not *what* data it's allowed to
  contain — Secure-side code must independently validate every pointer and
  length received from the Non-secure world as if it were attacker-supplied,
  because from a security standpoint, it is.
- **Assuming a secure boot chain is "on" just because TrustZone is
  configured.** SAU/IDAU partitioning and a verified boot chain are separate
  pieces — a system can have a correctly partitioned Secure/Non-secure
  memory map and still boot an unverified, potentially malicious
  Non-secure image if the boot chain itself doesn't check signatures.
- **Forgetting the context-preservation requirement for Non-secure tasks
  calling into Secure code.** A custom or incomplete TrustZone integration
  that doesn't correctly save/restore per-task Secure-side context on
  preemption can produce rare, hard-to-reproduce corruption exactly when
  two Non-secure tasks both call into Secure code and get preempted
  mid-call — using FreeRTOS-Kernel's own maintained TrustZone support rather
  than a partial custom implementation avoids this class of bug.

## Cheat sheet

| Concept | Purpose | Enforced by |
|---|---|---|
| Secure / Non-secure worlds | Whole-system security partition | Hardware (SAU/IDAU), every bus transaction |
| MPU (Level 3 Module 4) | Task-level bug containment within one world | MPU peripheral, within one security state |
| NSC (Non-Secure Callable) region | Only permitted entry point into Secure world | Hardware, fixed veneer addresses |
| `secure_context.c` (FreeRTOS-Kernel) | Preserves per-task Secure-side call state across preemption | Kernel context-switch hook |
| Secure boot chain | Establishes trust before the partition matters at all | Immutable first-stage verification |

## Exercise

1. Read ARM's TrustZone-M architecture documentation (the Cortex-M33
   Technical Reference Manual's security sections are a solid primary
   source) and diagram, for a specific real SoC, which memory regions
   (flash, SRAM, peripherals) are Secure vs. Non-secure vs. NSC in its
   default/typical partition.
2. Read FreeRTOS-Kernel's `secure_context.c` for an ARMv8-M port and explain,
   in your own words, exactly what state it saves/restores and why this
   must happen in lockstep with the Non-secure task's own context switch
   (Level 3 Module 2) rather than independently.
3. Design (on paper) a minimal Secure-world API surface for a device that
   needs firmware-signature verification and a hardware key store, listing
   every NSC entry point and, for each, what parameter validation the
   Secure-side implementation must perform on inputs coming from the
   (untrusted) Non-secure caller.
