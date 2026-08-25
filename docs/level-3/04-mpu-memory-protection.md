# MPU Support & Memory Protection

**Verification note for this module:** the Memory Protection Unit is real
silicon (ARMv7-M/ARMv8-M MPU peripheral) with no meaningful host-side
simulation — the POSIX port used elsewhere in this course has no MPU
equivalent (there is no memory protection concept in a POSIX process the way
FreeRTOS-MPU means it). Nothing in this module was built or run; it is a
careful technical review against the FreeRTOS-Kernel MPU ports
(`GCC/ARM_CM4_MPU`, `GCC/ARM_CM33/secure` etc.) and ARM's own MPU
architecture reference.

## What FreeRTOS-MPU actually protects against

Standard FreeRTOS has no memory isolation between tasks at all — any task
can read or write any other task's stack, the kernel's own data structures,
or any peripheral register, because everything runs in a single flat address
space at the CPU's most privileged level. FreeRTOS-MPU (`configENABLE_MPU`)
changes this by giving each task an **MPU region configuration**: a task can
be restricted to only its own stack plus explicitly-granted regions, and
demoted to "unprivileged" execution so it cannot itself reconfigure the MPU
or execute privileged-only instructions.

```c
// FreeRTOSConfig.h
#define configENABLE_MPU 1
#define configTOTAL_MPU_REGIONS 8
#define configTEX_S_C_B_FLASH 0x07UL
#define configTEX_S_C_B_SRAM  0x07UL
#define configENFORCE_SYSTEM_CALLS_FROM_KERNEL_ONLY 1

// Region access rights, packed into an enum per FreeRTOS-MPU's API:
typedef struct MPU_REGION_INFO {
    void *pvBaseAddress;
    uint32_t ulLengthInBytes;
    uint32_t ulRegionPermissions;   // e.g. tskMPU_READ_PERMISSION | tskMPU_WRITE_PERMISSION
} MemoryRegion_t;
```

## Privileged vs. unprivileged tasks

```c
TaskParameters_t xSensorTaskParams = {
    .pvTaskCode     = vSensorTask,
    .pcName         = "Sensor",
    .usStackDepth   = 256,
    .pvParameters   = NULL,
    .uxPriority     = 2 | portPRIVILEGE_BIT,   // OR this bit in to run privileged
    .puxStackBuffer = sensorStack,
    .xRegions = {
        { (void *)0x20001000, 256, tskMPU_READ_PERMISSION | tskMPU_WRITE_PERMISSION },
        { (void *)ADC_PERIPH_BASE, 0x1000, tskMPU_READ_PERMISSION | tskMPU_WRITE_PERMISSION },
        { 0, 0, 0 },  // unused regions must be zeroed
    },
};
xTaskCreateRestricted(&xSensorTaskParams, &sensorHandle);
```

A task created via `xTaskCreateRestricted` *without* `portPRIVILEGE_BIT` runs
in unprivileged mode: it can only access its own stack (region 0, always
auto-configured by the kernel), plus whatever's listed in `xRegions` — one
peripheral's registers, one shared buffer, nothing else. Trying to touch
memory outside its configured regions faults immediately (a Cortex-M
`MemManage` fault) rather than silently corrupting whatever it hit. This
converts an entire class of bug — a stray pointer write clobbering another
task's stack or the kernel's TCB list from Module 1 — from silent corruption
discovered hours later into an immediate, precisely-located hardware fault
at the moment of the bad access.

## Why this is not "security" in the general sense

FreeRTOS-MPU is a **fault-isolation and bug-containment** mechanism, not a
security boundary against a determined attacker with code-execution
capability inside the "protected" region. A few concrete limits:

- The kernel itself, and any privileged task, can reconfigure the MPU or
  touch anything — MPU regions constrain *unprivileged* code only.
- System calls (`MPU_xQueueSend` etc. — the MPU-wrapped API variants
  unprivileged tasks must use instead of the raw kernel calls) briefly raise
  privilege to execute the real kernel function, then drop back down; a bug
  in kernel code reachable through a syscall is not contained by the task's
  own restricted regions.
- Region granularity and alignment are hardware-constrained (ARMv7-M MPU
  regions must be power-of-two sized and aligned to their own size;
  ARMv8-M's MPU is more flexible but still has real minimum granularity) —
  you cannot always give a task *exactly* the memory it needs and nothing
  more; padding to the next valid region size/alignment is normal and can
  waste RAM on small targets.
- This is a materially different, much lighter-weight mechanism than
  TrustZone (Level 4 Module 5), which provides a hardware-enforced secure/
  non-secure world split even privileged code in the non-secure world cannot
  cross — MPU protects tasks from *each other*; TrustZone protects a secure
  world from an entire non-secure OS including its kernel.

## The stack-overflow-detection connection

MPU-backed builds get a stronger version of Level 1's stack overflow
detection: instead of `configCHECK_FOR_STACK_OVERFLOW`'s pattern/watermark
checks (which only detect overflow *after* it already wrote past the stack,
if it happens to hit the watermark region), an MPU region placed immediately
past a task's stack with no access rights turns any stack overflow into an
instant `MemManage` fault at the exact instruction that overran — a strictly
stronger and earlier guarantee than the pattern-based check alone, at the
cost of consuming one of the limited MPU regions per task.

## Traps

- **Assuming `configENABLE_MPU` provides isolation without also auditing
  every unprivileged task's use of raw (non-`MPU_`-prefixed) kernel API
  calls.** Unprivileged tasks must go through the `MPU_`-wrapped syscall
  layer; calling raw kernel functions directly from unprivileged code is
  either blocked by the port or undefined depending on configuration —
  check your specific port's enforcement.
- **Running out of MPU regions.** ARMv7-M typically provides 8 regions
  total; the kernel reserves some (flash, kernel RAM, the task's own stack)
  before the application gets any — `configTOTAL_MPU_REGIONS` budgets this
  explicitly, and a design with many distinct shared buffers per task can
  exhaust it fast. Consolidate shared regions rather than one region per
  buffer where possible.
- **Forgetting alignment/size constraints.** An `xRegions` entry with a base
  address or length that doesn't satisfy the underlying MPU hardware's
  alignment rules either fails silently to the nearest valid boundary or
  is rejected outright depending on the port — always round explicitly
  rather than relying on the port to do the "obviously correct" thing.
- **Treating MPU protection as equivalent to TrustZone-grade security.** As
  above: MPU faults contain *bugs* (a wild pointer, a stack overflow) between
  cooperating-but-imperfect application tasks. It does not stop a task that
  has arbitrary code execution rights within its own region from calling
  legitimate syscalls to manipulate shared kernel state it's allowed to
  reach, and it provides no protection at all against a hostile actor with
  physical or debug-port access.
- **Not testing the *fault path*.** It's easy to test that a well-behaved
  MPU-restricted task runs correctly and skip deliberately triggering an
  out-of-region access to confirm the fault handler actually catches it
  (rather than, say, silently succeeding due to a misconfigured region) —
  the whole value of this feature is in the fault path, so it needs its own
  explicit test, not just happy-path testing.

## Cheat sheet

| Concept | Standard FreeRTOS | FreeRTOS-MPU |
|---|---|---|
| Address space | Flat, every task sees everything | Per-task restricted regions |
| Privilege | All tasks run privileged | Privileged / unprivileged distinction |
| Bad memory access | Silent corruption | Immediate `MemManage` fault |
| Kernel API from tasks | Direct calls | `MPU_`-wrapped syscalls for unprivileged tasks |
| Stack overflow detection | Pattern/watermark (after the fact) | Optional guard region (at the instant of overflow) |
| Region limits | N/A | Hardware-fixed count + alignment rules |
| Security boundary strength | None | Bug containment — not attacker-resistant, not TrustZone-equivalent |
| Verified on this machine | N/A | N/A — no MPU-capable simulation exists; source/spec review only |

## Exercise

1. Read ARM's Cortex-M4 (ARMv7-M) MPU architecture reference (region
   alignment/size rules specifically) and design an `xRegions` layout for a
   task needing access to a 300-byte buffer and a peripheral with a
   0x400-byte register block — work out what actual region sizes and
   alignments the hardware requires, since neither input size is already a
   valid MPU region size.
2. On an MPU-capable Cortex-M board, deliberately write a task that
   overflows its stack while running unprivileged with a guard region
   configured, and confirm you get an immediate fault rather than silent
   corruption — capture and interpret the fault status register contents.
3. Write down, in your own words, three specific attacks or bugs FreeRTOS-MPU
   *does* contain and three it explicitly does *not* — and explain why
   TrustZone (Level 4 Module 5) is needed for the ones MPU alone can't cover.
