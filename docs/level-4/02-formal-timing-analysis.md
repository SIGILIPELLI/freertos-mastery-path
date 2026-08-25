# Formal Timing Analysis (RMA & WCET)

**Verification note for this module:** Rate Monotonic Analysis is
mathematics, not code — the schedulability formulas below were checked by
hand and against the standard references (Liu & Layland's original 1973
bound, and the more precise Lehoczky/Sha/Ding exact test); no simulation or
build was needed or performed. Worst-case execution time (WCET) analysis, by
contrast, genuinely requires either a certified static-analysis tool (e.g.
AbsInt aiT) or measurement on real target hardware with cache/pipeline
behavior matching production silicon — neither is available in this
environment, so the WCET discussion below is a careful conceptual and
methodological treatment, explicitly not a computed bound for any real
program.

## Rate Monotonic Analysis: the theory behind priority assignment

Every module in this course has assigned task priorities by intuition or
by domain reasoning ("the sensor ISR handler should be higher priority than
logging"). RMA formalizes this: for a set of independent, periodic,
preemptible tasks where each task's priority is assigned inversely to its
period (shortest period = highest priority — this is the "rate monotonic"
rule itself), the system is schedulable — every task always meets its
deadline — if total CPU utilization stays under a specific bound.

**Liu & Layland's sufficient (but not necessary) bound**, for n tasks:

```
U = Σ (Cᵢ / Tᵢ)  ≤  n(2^(1/n) − 1)
```

where `Cᵢ` is task i's worst-case execution time and `Tᵢ` is its period. As
n → ∞, the bound converges to `ln(2) ≈ 0.693` — meaning a large rate
monotonic task set is *guaranteed* schedulable if total utilization stays
under roughly 69%, regardless of the specific period values. This is a
**sufficient** condition, not a necessary one: a task set exceeding this
bound might still be perfectly schedulable — the bound is deliberately
conservative so that satisfying it is a fast, simple, always-safe check.

**Worked example**, three tasks:

| Task | Period (Tᵢ, ms) | WCET (Cᵢ, ms) | Utilization (Cᵢ/Tᵢ) |
|---|---|---|---|
| Sensor sampling | 10 | 1 | 0.100 |
| Control loop | 20 | 4 | 0.200 |
| Logging | 100 | 8 | 0.080 |

`U = 0.100 + 0.200 + 0.080 = 0.380`. For n=3, the Liu & Layland bound is
`3(2^(1/3) − 1) ≈ 3 × 0.2599 ≈ 0.7798`. Since `0.380 ≤ 0.7798`, this task
set is guaranteed schedulable under rate monotonic priority assignment
(sensor sampling highest, logging lowest — exactly the priority ordering
intuition from earlier modules would also suggest, now with a formal
guarantee behind it rather than just intuition).

## The exact test, when the sufficient bound fails

If a task set fails the Liu & Layland bound, it isn't necessarily
unschedulable — the **exact characterization** (Lehoczky, Sha, and Ding,
1989) checks, for each task i, whether its own worst-case response time
(accounting for preemption by all higher-priority tasks) stays within its
period, using the well-known response-time recurrence:

```
Rᵢ = Cᵢ + Σ_{j ∈ hp(i)} ⌈ Rᵢ / Tⱼ ⌉ · Cⱼ
```

solved iteratively (start with `Rᵢ = Cᵢ`, substitute, repeat until it
converges or exceeds `Tᵢ`, in which case the task set is unschedulable as
assigned). This test is exact — a task set that passes it is schedulable,
full stop, no false negatives — but requires per-task computation rather
than one aggregate utilization check.

## Worst-case execution time: the number the whole analysis depends on

Every formula above depends entirely on having a trustworthy `Cᵢ` for each
task — and this is where RMA's clean mathematics runs into genuinely hard
engineering reality. WCET is **not** "the slowest time I measured in
testing." A measured maximum only tells you the worst case *among the
inputs and conditions you happened to exercise* — it systematically misses
rare paths (an error-handling branch, a cache-cold worst case, a specific
data pattern that causes maximum loop iterations) that a certification-grade
safety case cannot accept as "probably fine."

Two genuinely different approaches exist, with different guarantees:

- **Static WCET analysis tools** (e.g. AbsInt aiT, and similar
  academically- and commercially-developed tools) analyze the compiled
  binary's control-flow graph directly, modeling the target processor's
  pipeline, cache, and branch prediction behavior to compute a
  provable upper bound without executing the code at all. This is the
  approach certification-grade safety cases generally require, because it
  produces a bound with a mathematical guarantee rather than a statistical
  one — but it requires a processor-specific timing model the tool vendor
  has built and validated for your exact target, and can be pessimistic
  (a safe but not tight bound) for code with highly data-dependent timing.
- **Measurement-based approaches** (running the code many times under
  varied, deliberately adversarial inputs and cache states, then taking the
  observed maximum plus a safety margin) are cheaper and more widely
  applicable, but fundamentally cannot *prove* an upper bound — only
  provide increasing confidence that the true worst case has been observed
  or approximated closely, which is why they are typically insufficient
  alone for the highest integrity levels.

## Why context switching (Level 3 Module 2) and ISR latency (Module 7) feed directly into this

RMA's `Cᵢ` for a task is not just "the task's own code path time" — it must
include the kernel overhead the task actually experiences: context switch
cost (Level 3 Module 2), any time spent in a critical section it's blocked
behind (Level 3 Module 7's masking-latency discussion), and, if using real
mutexes, its priority-inheritance-bounded blocking time from lower-priority
tasks holding a shared resource — this last term is itself a separate,
well-known extension to the basic RMA model (blocking-time terms, associated
with the priority ceiling / priority inheritance protocols Level 1 and
Level 3 Module 7 already covered as correctness mechanisms). A WCET number
computed in isolation from the RTOS's own scheduling overhead
systematically understates real worst-case response time.

## Traps

- **Using a measured maximum as WCET without justification.** As covered
  above, this is the single most common and most dangerous shortcut —
  acceptable for informal engineering estimates, generally not acceptable
  as certification evidence at meaningful integrity levels.
- **Applying the Liu & Layland *sufficient* bound and concluding a task
  set that fails it is definitely unschedulable.** It's a conservative
  sufficient condition — a failing task set requires the exact
  response-time test before concluding anything.
- **Computing utilization or response time without including
  RTOS scheduling overhead** (context switch cost, critical-section
  masking, priority-inheritance blocking terms) as part of each task's
  effective `Cᵢ` — this systematically produces an optimistic, unsafe bound.
- **Ignoring task interdependency.** Basic RMA assumes independent,
  periodic tasks with no blocking on shared resources beyond the standard
  blocking-time extension — a design with complex inter-task dependencies
  (Level 2 Module 8's multi-stage pipelines) needs the corresponding
  extended analysis, not the bare formulas above.
- **Trusting a static WCET tool's number blindly without confirming its
  processor timing model actually matches your silicon revision, cache
  configuration, and compiler/optimization settings.** A mismatched model
  can produce a bound that looks rigorous but doesn't actually apply to the
  system being shipped.

## Cheat sheet

| Concept | Formula/method | Guarantee |
|---|---|---|
| Rate monotonic priority assignment | Shortest period = highest priority | Optimal fixed-priority assignment for this task model |
| Liu & Layland sufficient bound | `U ≤ n(2^(1/n)−1)`, → ln(2)≈0.693 as n→∞ | Sufficient, not necessary — conservative |
| Exact schedulability test | Iterative response-time recurrence | Exact — no false negatives |
| Measured "WCET" | Max observed execution time + margin | Not a proof; can miss rare worst-case paths |
| Static WCET analysis | Tool models CFG + processor pipeline/cache | Provable upper bound, given an accurate processor model |
| Blocking-time extension | Adds priority-inheritance-bounded blocking to Cᵢ | Required for tasks that share mutexes |

## Exercise

1. Take the three-task example above, add a fourth task (period 50ms, WCET
   6ms) and recompute total utilization against the Liu & Layland bound for
   n=4 — if it fails, run the exact response-time recurrence by hand for
   each task and determine whether the four-task set is actually
   schedulable.
2. Research one real static WCET analysis tool (AbsInt aiT is the most
   widely cited in aerospace/automotive contexts) and summarize, in your own
   words, how it builds its processor timing model and why that model is
   specific to a processor/cache/compiler combination rather than portable
   across targets.
3. Take one of Level 3's real measured numbers (Module 2's 2.014 us/switch
   POSIX-simulator context-switch overhead) and explain precisely why it
   cannot be plugged into a real RMA/WCET calculation for a Cortex-M target
   — connect this explicitly back to Level 3 Module 2's own caveat about
   simulator timing not transferring to hardware.
