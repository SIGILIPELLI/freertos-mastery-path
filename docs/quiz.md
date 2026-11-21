---
title: "Final Quiz"
description: "Test what you have learned across all four levels of FreeRTOS Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "Why does an RTOS use priorities instead of a superloop?",
    "options": [
      "Less code",
      "More features",
      "Less RAM",
      "So urgent activities preempt less urgent ones, giving bounded response times"
    ],
    "correctIndex": 3,
    "explanation": "Determinism."
  },
  {
    "question": "Which API do you call inside an ISR to send to a queue?",
    "options": [
      "xQueueSend",
      "xQueueSendFromISR (and yield if a higher-priority task was woken)",
      "vTaskDelay",
      "xSemaphoreTake"
    ],
    "correctIndex": 1,
    "explanation": "Never block in ISRs."
  },
  {
    "question": "A task's xTaskCreate stack depth of 128 means what on a 32-bit port?",
    "options": [
      "128 words (512 bytes), not bytes",
      "128 KB",
      "128 bytes",
      "128 bits"
    ],
    "correctIndex": 0,
    "explanation": "Check your port."
  },
  {
    "question": "Two tasks share a UART. What's the cleanest design?",
    "options": [
      "Two writers",
      "Disable interrupts",
      "A global flag",
      "A gatekeeper task owning the UART with a queue of requests"
    ],
    "correctIndex": 3,
    "explanation": "Avoids contention and inversion."
  },
  {
    "question": "A low-priority task holds a mutex needed by a high-priority task while a medium task runs. What is this?",
    "options": [
      "Race",
      "Starvation",
      "Priority inversion; mutexes with priority inheritance mitigate it",
      "Deadlock"
    ],
    "correctIndex": 2,
    "explanation": "Mars Pathfinder."
  },
  {
    "question": "You need one task to signal another with minimum RAM and overhead. What do you use?",
    "options": [
      "Queue",
      "Event group",
      "Mutex",
      "A direct-to-task notification"
    ],
    "correctIndex": 3,
    "explanation": "Lightweight."
  },
  {
    "question": "A timer callback calls a blocking queue receive. What goes wrong?",
    "options": [
      "Faster",
      "Nothing",
      "It blocks the timer service task and stalls every software timer",
      "Crash"
    ],
    "correctIndex": 2,
    "explanation": "Callbacks must not block."
  },
  {
    "question": "The ISR handles a UART byte stream at high rate. Which primitive fits?",
    "options": [
      "A stream buffer with a trigger level (single writer, single reader)",
      "Event group",
      "Mutex",
      "Queue of bytes"
    ],
    "correctIndex": 0,
    "explanation": "Low overhead."
  },
  {
    "question": "How do you size a task stack?",
    "options": [
      "Max",
      "Guess",
      "Default",
      "Measure the high-water mark under worst-case load and add margin"
    ],
    "correctIndex": 3,
    "explanation": "uxTaskGetStackHighWaterMark."
  },
  {
    "question": "A hard fault happens only under load. What's a likely cause?",
    "options": [
      "Bad clock",
      "Network",
      "Bad flash",
      "Stack overflow in a task (enable stack overflow checking)"
    ],
    "correctIndex": 3,
    "explanation": "configCHECK_FOR_STACK_OVERFLOW."
  },
  {
    "question": "Why use static allocation in a safety product?",
    "options": [
      "Deterministic memory with no runtime heap failures or fragmentation",
      "Smaller code",
      "Faster",
      "Law"
    ],
    "correctIndex": 0,
    "explanation": "xTaskCreateStatic."
  },
  {
    "question": "How does tickless idle save power?",
    "options": [
      "Disables tasks",
      "Slows clock",
      "It suppresses the periodic tick so the MCU sleeps until the next event or timeout",
      "Powers off RAM"
    ],
    "correctIndex": 2,
    "explanation": "configUSE_TICKLESS_IDLE."
  },
  {
    "question": "A task never feeds the watchdog because it's blocked forever. What's the right supervision design?",
    "options": [
      "Feed in a timer",
      "Feed in ISR",
      "Never feed",
      "Each critical task checks in with a supervisor that feeds the hardware watchdog only when all are healthy"
    ],
    "correctIndex": 3,
    "explanation": "Avoid blind feeding."
  },
  {
    "question": "On a dual-core ESP32, two tasks race on shared data even with taskENTER_CRITICAL on one core. Why?",
    "options": [
      "Timer",
      "Bad compiler",
      "Critical sections need an SMP spinlock (mux) to exclude the other core",
      "Cache"
    ],
    "correctIndex": 2,
    "explanation": "Use portMUX."
  },
  {
    "question": "Tasks (C,T): (1,4), (2,8), (1,10). What's the CPU utilization?\n```python\nprint(round(1/4+2/8+1/10,3))\n```\n",
    "options": [
      "1.0",
      "0.6",
      "0.75",
      "0.5"
    ],
    "correctIndex": 1,
    "explanation": "Sum of C/T. Verified by running it."
  },
  {
    "question": "Utilization is below the Liu-Layland bound for the task set. What does that imply under rate-monotonic scheduling?",
    "options": [
      "Needs more",
      "It fails",
      "Schedulability is guaranteed (the bound is sufficient, not necessary)",
      "Unknown"
    ],
    "correctIndex": 2,
    "explanation": "Otherwise use response-time analysis."
  },
  {
    "question": "What does PendSV do on Cortex-M FreeRTOS?",
    "options": [
      "Handles faults",
      "Performs the context switch at the lowest interrupt priority",
      "Runs timers",
      "Starts tasks"
    ],
    "correctIndex": 1,
    "explanation": "Deferred switch."
  },
  {
    "question": "How do you prevent one faulty task from corrupting another's memory?",
    "options": [
      "MPU-enabled FreeRTOS with restricted tasks",
      "Bigger heap",
      "Stack canary only",
      "Mutex"
    ],
    "correctIndex": 0,
    "explanation": "Hardware isolation."
  },
  {
    "question": "Trace shows a high-priority task misses deadlines when a low-priority task runs. Likely cause?",
    "options": [
      "Tick",
      "Idle",
      "Bad queue",
      "Priority inversion or long critical sections holding interrupts/locks"
    ],
    "correctIndex": 3,
    "explanation": "Use inheritance and shorten critical sections."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### Rate-monotonic schedulability

Implement utilization, the Liu-Layland bound and fixed-priority response-time analysis (R = C + sum of ceil(R/Tj)*Cj over higher-priority tasks, iterated to a fixed point; deadline = period) and apply them to two task sets.

**Starter code**

```c
import math

def utilization(tasks):
    # sum of C/T
    return 0.0

def ll_bound(n):
    # Liu & Layland rate-monotonic bound n*(2^(1/n)-1)
    return 0.0

def response_times(tasks):
    # tasks = [(name, C, T)], deadline = period, priority by shorter period (rate monotonic).
    # worst-case response time: R = C + sum(ceil(R/Tj)*Cj over higher-priority tasks), iterate to a fixed point;
    # stop if R exceeds the period. Return {name: (R, meets_deadline)}.
    return {}

SET_A = [("sensor", 1, 4), ("control", 2, 8), ("logger", 1, 10)]
SET_B = [("fast", 2, 5), ("mid", 3, 7), ("slow", 4, 20)]
for label, ts in [("A", SET_A), ("B", SET_B)]:
    u = utilization(ts)
    print(f"set {label}: U={u:.3f} bound={ll_bound(len(ts)):.3f} -> {'guaranteed' if u <= ll_bound(len(ts)) else 'test needed'}")
    for name, (r, ok) in response_times(ts).items():
        print(f"  {name}: R={r} {'meets' if ok else 'MISSES'} deadline")
```

**Expected output**

```text
set A: U=0.600 bound=0.780 -> guaranteed
  sensor: R=1 meets deadline
  control: R=3 meets deadline
  logger: R=4 meets deadline
set B: U=1.029 bound=0.780 -> test needed
  fast: R=2 meets deadline
  mid: R=5 meets deadline
  slow: R=21 MISSES deadline
```

<details><summary>Show solution and explanation</summary>

<pre><code>import math

def utilization(tasks):
    return sum(c / t for _, c, t in tasks)

def ll_bound(n):
    return n * (2 ** (1 / n) - 1)

def response_times(tasks):
    &quot;&quot;&quot;Fixed-priority response-time analysis. tasks = [(name, C, T)], deadline = period.
    Higher priority = shorter period (rate-monotonic).&quot;&quot;&quot;
    tasks = sorted(tasks, key=lambda x: x[2])
    out = {}
    for i, (name, c, t) in enumerate(tasks):
        r = c
        while True:
            interference = sum(math.ceil(r / tj) * cj for _, cj, tj in tasks[:i])
            nxt = c + interference
            if nxt == r:
                break
            r = nxt
            if r &gt; t:
                break
        out[name] = (r, r &lt;= t)
    return out

SET_A = [(&quot;sensor&quot;, 1, 4), (&quot;control&quot;, 2, 8), (&quot;logger&quot;, 1, 10)]
SET_B = [(&quot;fast&quot;, 2, 5), (&quot;mid&quot;, 3, 7), (&quot;slow&quot;, 4, 20)]
for label, ts in [(&quot;A&quot;, SET_A), (&quot;B&quot;, SET_B)]:
    u = utilization(ts)
    print(f&quot;set {label}: U={u:.3f} bound={ll_bound(len(ts)):.3f} -&gt; {&#x27;guaranteed&#x27; if u &lt;= ll_bound(len(ts)) else &#x27;test needed&#x27;}&quot;)
    for name, (r, ok) in response_times(ts).items():
        print(f&quot;  {name}: R={r} {&#x27;meets&#x27; if ok else &#x27;MISSES&#x27;} deadline&quot;)
</code></pre>

<p>Set A passes the sufficient utilization bound, so it&#x27;s guaranteed. Set B exceeds the bound, so the exact response-time test is needed, and it shows which task misses its deadline. Verified by running the solution.</p>
</details>

### A fixed-priority preemptive scheduler

Simulate FreeRTOS-style scheduling one tick at a time: each tick run the highest-priority released task with remaining work. Show the timeline for a low-priority task preempted by medium and high-priority tasks.

**Starter code**

```c
def simulate(tasks, horizon):
    # tasks = [(name, priority, release_tick, work_ticks)], higher number = higher priority.
    # each tick run the highest-priority released task that still has work (preemptive);
    # return (timeline string with one letter per tick and '.' when idle, {name: finish_tick})
    return "", {}

# Three tasks: L (low) starts first, M (medium) and H (high) preempt it.
tasks = [("L", 1, 0, 6), ("M", 2, 2, 3), ("H", 3, 4, 2)]
line, finish = simulate(tasks, 14)
print(line)
print(finish)
# A priority-inversion-free run versus the same set where H must wait for L's critical section:
tasks2 = [("L", 1, 0, 3), ("H", 3, 1, 2)]
print(simulate(tasks2, 8))
```

**Expected output**

```text
LLMMHHMLLLL...
{'H': 6, 'M': 7, 'L': 11}
('LHHLL...', {'H': 3, 'L': 5})
```

<details><summary>Show solution and explanation</summary>

<pre><code>def simulate(tasks, horizon):
    &quot;&quot;&quot;Fixed-priority preemptive scheduler, 1 tick per step.
    tasks = [(name, priority, release_tick, work_ticks)], higher number = higher priority.&quot;&quot;&quot;
    remaining = {n: w for n, _, _, w in tasks}
    finish = {}
    timeline = []
    for t in range(horizon):
        ready = [(p, n) for n, p, r, _ in tasks if r &lt;= t and remaining[n] &gt; 0]
        if not ready:
            timeline.append(&quot;.&quot;)
            continue
        _, running = max(ready)
        timeline.append(running)
        remaining[running] -= 1
        if remaining[running] == 0:
            finish[running] = t + 1
    return &quot;&quot;.join(timeline), finish

# Three tasks: L (low) starts first, M (medium) and H (high) preempt it.
tasks = [(&quot;L&quot;, 1, 0, 6), (&quot;M&quot;, 2, 2, 3), (&quot;H&quot;, 3, 4, 2)]
line, finish = simulate(tasks, 14)
print(line)
print(finish)
# A priority-inversion-free run versus the same set where H must wait for L&#x27;s critical section:
tasks2 = [(&quot;L&quot;, 1, 0, 3), (&quot;H&quot;, 3, 1, 2)]
print(simulate(tasks2, 8))
</code></pre>

<p>A newly ready higher-priority task preempts immediately, which is why L finishes last and H runs as soon as it&#x27;s released. This models the rule &#x27;run the highest-priority Ready task&#x27;, not FreeRTOS internals. Verified by running the solution.</p>
</details>
