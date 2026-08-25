# OTA Updates for RTOS Firmware

**Verification note for this module:** OTA update mechanisms depend on flash
memory layout, a bootloader, and typically a real or emulated network
connection to a real update server — none of which exist in this
environment. This module reviews the standard architecture (dual-bank/A-B
partitioning, the AWS IoT OTA library's documented design as a concrete
widely-used reference, and the security requirements around it) based on
published documentation, not a working build.

## Why OTA on an RTOS device is harder than "download and reboot"

A field device performing a software update has failure modes a desktop
update simply doesn't: it can lose power mid-write, lose network mid-
download, or receive a corrupted or malicious image — and unlike a desktop
or phone, there is often no user physically present to intervene, and no
easy way to re-flash a device that's bricked itself in the field. Every
credible OTA design for embedded/RTOS firmware is built around one
non-negotiable requirement: **a failed or interrupted update must never
prevent the device from booting into *some* working firmware.**

## Dual-bank (A/B) partitioning: the standard answer

```
┌──────────────┐   ┌──────────────┐
│  Bank A       │   │  Bank B       │
│  (currently   │   │  (new image   │
│   running)    │   │   downloads   │
│               │   │   here)       │
└──────────────┘   └──────────────┘
        ▲                   │
        │  bootloader picks │
        └── which bank to ──┘
            boot, verifies
            signature first
```

The device downloads and writes the new image entirely into the *inactive*
bank while the *active* bank keeps running the current, known-good
firmware — the running system is never itself being overwritten. Only after
the new image is fully written and its integrity (checksum) and
authenticity (cryptographic signature) are verified does the bootloader
mark it as the boot candidate and reset. If power fails at any point during
the download or write, the active bank (untouched) simply keeps booting on
the next power-up — the update attempt is lost, not the device.

## The self-test / rollback window

A more sophisticated and now near-universal refinement: after switching to
the new bank, the bootloader boots it in a **provisional / self-test**
state rather than immediately committing to it permanently. The new
firmware must explicitly confirm its own health (successfully initializing
its subsystems, successfully reaching a known-good running state, often
confirmed by successfully checking in with a server) within a bounded
window — if it crashes, hangs, or fails to confirm health within that
window, the bootloader automatically rolls back to the previous
known-good bank on the next reset (often triggered by an independent
watchdog timer that resets the device if the new firmware never explicitly
confirms health, exactly the kind of watchdog supervision pattern this
course's Level 2 covered, now operating at the update-safety level rather
than the single-task-hang level). This converts "the new firmware boots but
is subtly broken" from a bricked device into an automatic, unattended
recovery.

## Signature verification: non-negotiable, not optional

Every stage of an OTA chain that accepts a new image from the network must
cryptographically verify it before trusting it — this connects directly to
Level 4 Module 5's TrustZone/secure-boot discussion: ideally, signature
verification of an OTA image happens in (or is anchored by) the same
Secure-world / immutable-bootloader trust chain that establishes secure boot
in the first place, so that OTA cannot become a path to bypass the very
security boundary TrustZone was set up to enforce. A design that verifies
a checksum (integrity against corruption) but not a cryptographic signature
(authenticity against a malicious image) has solved the "network glitch"
problem but not the "attacker pushes a malicious update" problem — these are
distinct requirements and both are needed.

## OTA as an RTOS task design problem

The download/write/verify sequence itself is an ordinary FreeRTOS task
design exercise using patterns from throughout this course: a dedicated OTA
task (often deliberately low priority, so it doesn't starve time-critical
application tasks — Level 3 Module 7's priority-starvation reasoning applies
directly) receiving image chunks from the network stack (Level 3 Module 9)
through a queue or stream buffer, writing them to the inactive flash bank
through a flash-driver interface owned by exactly one task (Level 2 Module
8's gatekeeper pattern — concurrent, uncoordinated flash writes from
multiple tasks during an update is a serious corruption risk), with
progress and errors reported through the logging gatekeeper.

## Traps

- **Overwriting the currently-running firmware image in place**, rather
  than writing to an inactive bank. This is the single most catastrophic
  OTA design mistake — any interruption during the write leaves the device
  with neither a complete new image nor an intact old one, and no way to
  recover without physical reflashing.
- **Committing to the new firmware immediately on boot, with no self-test/
  rollback window.** Without this, a new image that boots but is
  functionally broken (passes the signature check, still has a bug) is a
  permanent bricking risk with no automatic recovery path.
- **Checksum-only verification without a cryptographic signature.** As
  covered above, this protects against corruption but not against a
  malicious or unauthorized image — a genuine, distinct security gap, not a
  redundant extra step.
- **Running the OTA download/write task at a priority that starves
  time-critical application tasks**, or conversely at a priority so low that
  a busy application starves the OTA process from ever completing a
  download within a reasonable window — this needs deliberate priority
  design, not a default.
- **Multiple tasks or code paths writing to flash during an update.**
  Flash writes are commonly slow, block-erase-then-write operations with
  their own hardware constraints — concurrent, uncoordinated access from
  more than one task is a real corruption risk that the gatekeeper pattern
  specifically exists to prevent.
- **Not testing power-loss-mid-update as an explicit test case.** This is
  the actual failure mode the entire dual-bank design exists to survive —
  a design that has never been tested with power cut at multiple points
  during an update has not actually verified its core safety property.

## Cheat sheet

| Concept | Purpose | Failure mode it prevents |
|---|---|---|
| Dual-bank (A/B) partitioning | New image never overwrites the running one | Bricking on power loss mid-update |
| Signature verification | Confirms authenticity, not just integrity | Malicious/unauthorized image acceptance |
| Self-test / rollback window | New image must prove health before permanent commit | Bricking on a functionally broken (but well-formed, signed) image |
| Flash-write gatekeeper task | Single owner of flash writes | Corruption from concurrent writers |
| Watchdog-backed health confirmation | Forces rollback if new firmware hangs | Silent permanent failure with no recovery |

## Exercise

1. Design (on paper) a full OTA state machine for a dual-bank device:
   states from "idle" through "downloading," "verifying," "provisional
   boot," "confirmed," and "rolled back" — and specify, for each state
   transition, what triggers it and what happens on a power loss during
   that specific state.
2. Read the AWS IoT OTA library's published documentation (or another
   widely-used embedded OTA framework's docs) and compare its actual state
   machine and self-test mechanism against your own design from question 1
   — note any safety consideration it handles that yours missed.
3. Write out the specific failure sequence that would occur if an OTA
   design verified an image's checksum but not its cryptographic signature,
   and an attacker with network access pushed a modified image with a
   correct checksum for the modified content — trace exactly where in the
   pipeline this should have been caught and wasn't.
