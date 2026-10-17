---
marp: true
---

<!--
theme: course
paginate: true
title: Isolation
style: |
  section { --course: #0d9488; }
-->

<!-- _class: lead -->
<!-- _paginate: false -->

# Isolation and interference

**Week 12** · affinity, CPU isolation, IRQ steering: the Hub's dress rehearsal

This rig is, literally, the project's checkpoint 2.

<!--
0:00–2:00. The real Hub will render a GUI and take telemetry in bursts. The pump
loop has to be armored by design, and the design has to be reproducible.
-->

---

<!-- _class: agenda -->

## Today

1. **What the contract doesn't cover**: three more doors
2. **The layered recipe**: one layer per door
3. **Reproducibility**: the configuration as a deliverable
4. **Lab**: three configurations, one interference rig

Goal: name what each isolation layer removes, and write a recipe another team can replay.

<!--
2:00–3:00. Reading: all of Buttazzo ch. 12, §13.1.1.
-->

---

<!-- _class: s1 -->
<!-- _header: 1 · What the contract doesn't cover -->

## Week 11 secured CPU time. Interference has other doors

- **Device interrupts** land on your core and run before you
- **Kernel housekeeping**: the tick, RCU callbacks, run everywhere by default
- **The memory system**: a neighbor thrashing the shared cache slows your instructions without taking a single time slice

<!--
3:00–8:00. The last one is week 3's silicon lesson, now with four cores.
-->

---

<!-- _class: s2 figure -->
<!-- _header: 2 · The layered recipe -->

## One layer per door

![h:520](img/iso_layers.svg)

<!--
8:00–15:00. SCHED_DEADLINE: CPU time. Affinity (taskset, sched_setaffinity):
placement. isolcpus nohz_full rcu_nocbs on the kernel command line: evict the
scheduler's other clients and the housekeeping from core 3. /proc/irq/*/smp_affinity:
send device interrupts elsewhere. The lab's table adds one layer per row; each row
should eat tail. The shared cache has no Linux knob on the Pi: it is the residue
the measurement will show.
-->

---

<!-- _class: s2 question -->
<!-- _header: 2 · The layered recipe -->

## Which of these layers did week 9's dedicated core give you for free?

1. None: the S3 has no Linux
2. CPU time, placement, housekeeping and device interrupts
3. Only CPU time
4. All of them, the shared cache included

<!--
15:00–17:00. Answer: 2. A dedicated core is this recipe implemented in silicon:
nothing else runs there. 4 goes too far: the S3's two cores still share the cache.
Linux needs the layers spelled out because the box is shared.
-->

---

<!-- _class: s2 question -->
<!-- _header: 2 · The layered recipe -->

## `isolcpus=3` is set and the loop is pinned to core 3. Still jittery. Why?

1. isolcpus doesn't work on ARM
2. Device interrupts still land on core 3
3. The loop needs a higher FIFO priority
4. Core 3 is slower than the others

<!--
17:00–19:00. Answer: 2. Misconception to anticipate: "isolcpus alone is enough".
Unpinned IRQs still land there, and without the contract a rogue FIFO task pinned
to the same core still wins. No single layer survives alone: that's why the table
has three rows.
-->

---

<!-- _class: s3 -->
<!-- _header: 3 · Reproducibility -->

## A configuration in someone's shell history dies at deployment

- ADR-004 is the recipe as literal commands and command-line parameters
- Every line cites the table row that justifies it
- Edwin's bar: another group replays it in 10 minutes

<!--
19:00–23:00.
-->

---

<!-- _class: s3 -->
<!-- _header: What about FreeRTOS? -->

## What about FreeRTOS?

- FreeRTOS SMP has affinity (`vTaskCoreAffinitySet`); isolation is "put nothing else on that core"
- That works because every task is yours
- Linux needs four layers because the box runs software you don't own

<!--
23:00–25:00. Same principle as week 9 (partitioning cuts tails), a more hostile
ecosystem.
-->

---

<!-- _class: takeaway -->

## The one thing to remember

> Each layer closes one door. Stack them, measure after each, and keep the commands: the recipe is the deliverable.

<!--
25:00–26:00.
-->

---

<!-- _class: s4 -->
<!-- _header: 4 · Lab -->

## Today in the lab

| Predict now | Then measure |
|---|---|
| Max jitter, DEADLINE loop, shared CPUs | 5 min, full interference |
| + affinity to isolated core 3 | the same rig |
| + IRQs moved off core 3 | the same rig |

Full interference: a synthetic render load and `iperf3` bursts.

<!--
26:00–31:00. Expected shape: the max collapses as layers stack; the last row's
number is the one the project inherits. Content closes today: everything after
this is the Hub.
-->

---

## References

- Buttazzo, ch. 12; §13.1.1 (RT-POSIX)
- Linux kernel docs: *kernel-parameters* (`isolcpus`, `nohz_full`, `rcu_nocbs`); *IRQ affinity*
- SOP-10 §5: the commands

<!--
31:00–40:00. Questions, then the rig.
-->
