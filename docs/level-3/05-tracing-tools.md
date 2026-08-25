# Tracing — Tracealyzer & SystemView

**Verification note for this module:** both Percepio Tracealyzer and Segger
SystemView require their proprietary host-side viewer applications and,
for full-fidelity traces, a debug probe or vendor SDK streaming channel —
neither runs in this environment. What's verified here is the underlying
kernel-side mechanism (`configUSE_TRACE_FACILITY`, the `traceTASK_SWITCHED_IN`
family of hook macros) by reading `tasks.c`/`queue.c` directly, plus the
event-count-based technique from Module 1/2 which *does* run on the POSIX
port as a lightweight, tool-free substitute. Actual timeline visualization
in Tracealyzer/SystemView was not exercised — treat the tool-specific
screenshots/workflow described here as accurate to their documented
behavior, not as something reproduced on this machine.

## Why "add a print statement" stops working here

Every module so far could be debugged by reasoning about a handful of tasks.
Real systems have dozens of tasks, ISRs, queues, and timers interacting at
microsecond granularity — a `printf` in a task changes its timing enough to
mask the very race or latency spike you're chasing (Level 1's ISR-safety
rules already forbid `printf`-from-ISR outright), and even from task context,
UART output at typical baud rates is far slower than the events being
observed. Tracing tools solve this by recording lightweight *events*
(task switch, queue send/receive, ISR entry/exit) into a RAM ring buffer via
the kernel's trace hook macros — cheap enough to leave running continuously
— and reconstructing a full timeline offline on a host machine.

## The kernel-side hook: `configUSE_TRACE_FACILITY`

FreeRTOS doesn't ship a tracer itself — it ships empty hook macros that a
tracing tool's recorder library redefines:

```c
// tasks.c, called on every context switch — no-op unless a recorder redefines it
traceTASK_SWITCHED_IN();
traceTASK_SWITCHED_OUT();

// queue.c, called on every send/receive
traceQUEUE_SEND(pxQueue);
traceQUEUE_RECEIVE(pxQueue);

// Percepio's or Segger's recorder library provides, e.g.:
#define traceTASK_SWITCHED_IN() \
    vTraceStoreEvent1(EVENT_TASK_SWITCHED_IN, (uint32_t)pxCurrentTCB)
```

`configUSE_TRACE_FACILITY` in `FreeRTOSConfig.h` must be enabled for these
macros to compile with any effect (`uxTaskGetSystemState`, used by both
tools to get an initial task-name/priority snapshot, also depends on it).
The macros themselves cost essentially nothing when undefined — this is why
the instrumentation can ship in production images without a real recorder
attached and cost nothing until one is.

## Percepio Tracealyzer

Tracealyzer's recorder (`TraceRecorder` library, linked into the firmware)
captures every scheduling event, queue/semaphore/mutex operation, and
(with additional instrumentation macros the application adds explicitly)
user-defined events, into a RAM buffer, then streams or dumps it for the
desktop Tracealyzer application to render as:

- A **vertical trace view**: one row per task, showing running/ready/blocked
  state over time — this is where priority inversion (Level 1) and
  unexpected preemption patterns become visually obvious that would be
  invisible in log output.
- **CPU load per task**, computed automatically from the switch-in/switch-out
  event pairs.
- **Response time / latency histograms** for a chosen event pair (e.g. ISR
  fires → handler task runs), directly answering the WCET-adjacent questions
  Module 2 and Module 7 raise, from real captured data rather than a
  hand-rolled instrumented build.

## Segger SystemView

SystemView takes a similar approach with its own lightweight recorder,
commonly paired with a J-Link debug probe streaming events in real time
over SWO or RTT (Real-Time Transfer) rather than a full JTAG halt — critical,
because halting the CPU to read a trace buffer would itself perturb the very
timing being measured. SystemView's view is more focused on ISR/task
interleaving and per-API-call timing than Tracealyzer's broader workflow
analysis, and its RTT-based streaming is why it pairs so naturally with
Segger's own J-Link hardware ecosystem specifically.

## The tool-free substitute this course *can* verify

Not every project has budget for Tracealyzer/SystemView licenses, and not
every question needs a full timeline — Module 2's event-counting technique
generalizes into a lightweight, always-available substitute for coarse
questions ("roughly how much time is this task spending blocked vs.
running", "how many times did this ISR fire during this window"):

```c
// A minimal software event counter — costs one increment, no I/O, no ring buffer
static volatile uint32_t switchCount = 0;
static volatile uint32_t queueSendCount = 0;

void vApplicationTickHook(void) {   // or inline in a hot path being audited
    // sample-based approximation of task state, cheap enough for continuous use
}
```

This was the exact technique used in Module 2's 20,000-switch benchmark: no
tracing tool, no ring buffer, no host viewer — just counters and
`clock_gettime`/a hardware cycle counter, read out at the end of a run. It
answers "is this in the right ballpark" and "did this regress after a
change" reliably; it does not give you a visual timeline, per-task CPU-load
breakdown, or a captured worst-case latency distribution the way a real
tracer does — for those, Tracealyzer or SystemView (or reading a logic
analyzer trace directly) is the right tool, and no amount of counter-based
cleverness substitutes for one when you actually need it.

## Traps

- **Enabling `configUSE_TRACE_FACILITY` and never actually attaching a
  recorder library "just in case."** The macros are cheap when undefined,
  but `uxTaskGetSystemState` and related trace-adjacent bookkeeping do add a
  small, permanent cost even without a tool attached — decide deliberately,
  don't leave it on by default in a shipped image without a reason.
  `configGENERATE_RUN_TIME_STATS` is a related, separately-gated feature with
  its own always-on runtime-counter cost — don't conflate the two.
- **Reading a trace buffer by halting the CPU under a debugger.** This is
  the single most common way to get a trace that "proves" a latency problem
  that doesn't exist in real operation — the halt itself perturbs every
  in-flight timer and interrupt; streamed (RTT/SWO) or post-mortem RAM-buffer
  approaches avoid this.
- **Trusting a lightly-instrumented counter-based measurement (like Module
  2's) as a substitute for a real trace when chasing an intermittent,
  rare-event latency spike.** Counters answer aggregate/average questions
  well; they cannot show you the *one* worst-case event and what was
  happening around it the way a timeline view can.
- **Forgetting that both tools' recorders consume RAM and a small amount of
  CPU continuously.** On a tightly RAM-constrained target, budget for the
  recorder's ring buffer explicitly rather than discovering it during
  integration.
- **Confusing "no visible problem in the trace" with "no problem exists."**
  A trace only shows what happened during the captured window — a
  once-a-day race condition may simply not have occurred during a 30-second
  capture; combine tracing with long-duration soak testing, not as a
  replacement for it.

## Cheat sheet

| Tool/technique | Mechanism | Best for | Cost |
|---|---|---|---|
| Tracealyzer | RAM ring buffer + desktop viewer | Full timeline, priority inversion, CPU load, latency histograms | License; recorder RAM/CPU |
| SystemView | RAM buffer + RTT/SWO streaming, J-Link-centric | Real-time ISR/task interleaving without halting the CPU | License-free (Segger); needs J-Link-class probe for streaming |
| Software event counters (verified here) | Plain volatile counters + `clock_gettime`/cycle counter | Coarse "is this in the right ballpark", regression checks | Free; no tool; no timeline |
| Halting the CPU under a debugger | JTAG/SWD halt + memory read | Last resort, one-shot inspection | Perturbs timing — avoid for latency-sensitive questions |

## Exercise

1. Enable `configUSE_TRACE_FACILITY` in a real project, integrate either
   Tracealyzer's or SystemView's recorder library, and capture a trace
   covering Level 1's producer/consumer queue example — confirm you can
   visually identify the send/receive pairing and measure the actual latency
   between them from the tool.
2. Deliberately introduce Level 1's priority inversion scenario (a
   high-priority task blocked behind a low-priority task holding a
   mutex, with a medium-priority task preempting the low-priority holder)
   and capture it in a trace — confirm the tool's timeline makes the
   inversion visually obvious in a way a log file would not.
3. Extend Module 2's software-counter technique to record a histogram (not
   just mean) of your ISR-latency-adjacent measurement from Module 7, using
   only counters and no tracing tool, and explain concretely what
   information you still cannot get from this approach that a real tracer
   would give you.
