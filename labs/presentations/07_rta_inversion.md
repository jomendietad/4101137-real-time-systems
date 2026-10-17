---
marp: true
---

<!--
theme: course
paginate: true
title: RTA and priority inversion
math: mathjax
style: |
  section { --course: #0d9488; }
-->

<!-- _class: lead -->
<!-- _paginate: false -->

# Response time and priority inversion

**Week 7** · each task's worst case, and the bug that rebooted a lander on Mars

The e-stop has a 5 ms deadline, not a percentage.

<!--
0:00–2:00. Utilization says whether the set fits; it doesn't say how long each task
takes at worst. Today: that number, and the way a mutex can break it.
-->

---

<!-- _class: agenda -->

## Today

1. **RTA**: one task's worst case, by iteration
2. **Inversion**: low holds, high waits, medium runs
3. **Inheritance**: the wait becomes bounded
4. **FreeRTOS**: same trap, same fix
5. **Lab**: reproduce it, fix it, predict it

Goal: run the RTA recurrence by hand, and bound the blocking a mutex adds.

<!--
2:00–3:00. Reading: Buttazzo §4.5.1–4.5.2, §7.1–7.3, PIP (§7.6.1), PCP (§7.7.1),
Theorem 7.2 (statement).
-->

---

<!-- _class: s1 -->
<!-- _header: 1 · RTA -->

## A task's worst response: itself, plus everyone above it

$$R_i = C_i + \sum_{h < i} \left\lceil \frac{R_i}{T_h} \right\rceil C_h$$

- $R_i$ is on both sides: start at $C_i$, iterate until it stops moving
- Stop early if it passes $D_i$: the task isn't schedulable
- Fifteen lines of Python, fed with **measured** $C_i$: a tool, not a theorem

<!--
3:00–6:00. Buttazzo Fig. 4.17 is the algorithm. The ceiling counts how many times
each higher-priority task can release inside the window R_i.
-->

---

<!-- _class: s1 figure -->
<!-- _header: 1 · RTA -->

## The book's example, step by step

![h:430](img/rta_iteration.svg)

<!--
6:00–10:00. Buttazzo Table 4.3, computed by img/rta_figs.py. Work steps 1 and 2 on
the board: R = 1 + ⌈1/4⌉·1 + ⌈1/5⌉·1 + ⌈1/6⌉·2 = 5; then with R = 5 the window
holds two releases of τ1. It settles at 10, exactly D4. Then run the same
iteration on the node's measured numbers from RET §1.
-->

---

<!-- _class: s1 question -->
<!-- _header: 1 · RTA -->

## Control's RTA says R = 2.1 ms. A 30 s capture shows 1.8 ms at most. Which is wrong?

1. The RTA: it's pessimistic, ignore it
2. The capture: it missed events
3. Neither: the critical instant is rare, the capture didn't hit it
4. Both: average them

<!--
10:00–12:00. Answer: 3, and they write it down in Task A. The RTA assumes every
higher-priority task releases at the same instant; 30 s rarely contains that.
If the capture ever exceeds the RTA, the C_i are wrong. Numbers are illustrative.
-->

---

<!-- _class: s2 -->
<!-- _header: 2 · Inversion -->

## Mars, July 1997: Pathfinder keeps rebooting

- A high-priority bus-management task and a low-priority weather task shared a mutex
- A medium-priority communications task preempted the low one while it held the lock
- The high task waited on the **medium** one; the watchdog saw it late and reset the lander
- JPL replayed the trace on Earth; the fix was one mutex attribute: priority inheritance

<!--
12:00–15:00. Pathfinder ran VxWorks; inheritance was available and off. The story
is the course's house rule in action: the trace found it.
-->

---

<!-- _class: s2 figure -->
<!-- _header: 2 · Inversion -->

## Three threads, one lock, two policies

![h:460](img/rta_inversion.svg)

<!--
15:00–20:00. Simulated by img/rta_figs.py. Top: L takes the lock, H asks for it and
blocks, M (no lock at all) preempts L: H now waits for M, 9 units. Bottom: while L
holds the lock it runs at H's priority, so M can't intervene: H waits only for L's
critical section, 3 units. Ask which of our threads plays each role: control (H),
telemetry (M), console (L). That is this afternoon's Task B.
-->

---

<!-- _class: s2 question -->
<!-- _header: 2 · Inversion -->

## Control and the console share a `k_sem` of 1 as a lock. What protects control?

1. Its higher priority
2. Nothing: a semaphore has no owner to inherit through
3. The semaphore's FIFO wait queue
4. The short critical section

<!--
20:00–22:00. Answer: 2. Misconception to anticipate: "a semaphore is a mutex". A
binary semaphore has no owner, so the kernel can't raise anyone's priority: it
allows exactly the top timeline. 4 bounds the critical section, not the time M
runs. For mutual exclusion across priorities: a PI mutex, always.
-->

---

<!-- _class: s3 -->
<!-- _header: 3 · Inheritance -->

## The wait doesn't vanish: it becomes bounded

$$R_i = C_i + B_i + \sum_{h < i} \left\lceil \frac{R_i}{T_h} \right\rceil C_h$$

- $B_i$: the longest critical sections that can block task $i$ (Theorem 7.2)
- Zephyr's `k_mutex` inherits priority; `k_sem` doesn't
- PCP (ceilings): no deadlock, at most one blocking; worth knowing it exists

<!--
22:00–27:00. Under PIP a task can be blocked at most once per lower-priority task
that shares a resource with it, by at most one critical section each: that is the
min(l_i, s_i) in Theorem 7.2. In the lab the extended script must predict the
PI-mutex regime's max within measurement noise.
-->

---

<!-- _class: s4 -->
<!-- _header: What about FreeRTOS? -->

## What about FreeRTOS?

- `xSemaphoreCreateMutex` inherits priority, like `k_mutex`
- FreeRTOS binary semaphores allow Task B's mistake, like `k_sem`
- Pathfinder ran VxWorks: the lesson doesn't depend on the kernel

<!--
27:00–28:00.
-->

---

<!-- _class: takeaway -->

## The one thing to remember

> Across priorities, mutual exclusion needs a mutex that inherits. Then add $B_i$ to the RTA, and the worst case is a number again.

<!--
28:00–29:00.
-->

---

<!-- _class: s5 -->
<!-- _header: 5 · Lab -->

## Today in the lab

| Predict now | Then measure |
|---|---|
| Control's R from your RTA script | the max over a 30 s capture |
| Control's R during the inversion | the bare-semaphore regime |
| Control's R with `k_mutex` | ≤ C + B + interference |
| B: the console's longest critical section | its GPIO pulse |

<!--
29:00–34:00. Three regimes, three captures: no contention, inversion, PI mutex.
Prediction and trace agreeing is the point of the module. Problem Set 2 goes out
today.
-->

---

## References

- Buttazzo, §4.5.1–4.5.2 (RTA, Fig. 4.17); §7.1–7.3, §7.6.1 (PIP), §7.7.1 (PCP), Theorem 7.2
- G. Reeves, "What really happened on Mars?", JPL, 1997 (the Pathfinder account)
- Zephyr docs: *Mutexes* (priority inheritance)
- FreeRTOS docs: *Mutexes*

<!--
34:00–40:00. Questions, then the lab.
-->
