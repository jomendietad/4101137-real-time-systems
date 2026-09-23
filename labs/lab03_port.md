# Week 3 — The S3 port and the first thread
> - **Reading:** [READINGS.md](../READINGS.md), week 3
> - **Module:** 2
> - **Firmware:** your week-2 `firmware/superloop/` (TASKs 1–3 filled in). Commit it before you start: today's first piece of evidence is a diff.

**From:** Eng. Samuel Cifuentes — *"Two pieces of news. One: production picked the
ESP32-S3 for the node — more memory, a radio, and two cores we'll use later. Two: I
saw the baseline with the blocking command; I'm authorizing a kernel evaluation.
First, move the superloop to the S3 **without rewriting it** — if we chose the
software platform well, that costs an overlay, not a porting effort. Then, the
first thread."*

| Stakeholder | Their question | How this session answers it |
|---|---|---|
| **Samuel** | What did changing silicon cost? | Changed lines: count them in the diff |
| **Gustavo** | Is the S3 "worse" for real time than the L476? | Same-code jitter comparison, two chips |

## What you'll measure

| Measurement | L476RG (wk 2) | S3 (today) |
|---|---|---|
| Max sampling jitter (superloop) | (copy) | ____ µs |
| ISR → service latency | (copy) | ____ µs |
| Max sampling jitter **with the sampling thread** (kernel) | — | ____ µs |

## Tasks

### Task A — The port (devicetree in action)
- `west build -p -b esp32s3_devkitc/esp32s3/procpu firmware/superloop` with the S3
  pin overlay (the instrumentation GPIOs move to a different port).
- Count the real cost of the port: `git diff --stat` — how many lines, and of what
  kind (C code, or hardware description)?
- **Evidence:** the diff-stat + the superloop running on the S3.

### Task B — Two silicons, same code
- Repeat the week-2 baseline measurement on the S3; fill in the S3 column.
- Week 2 already showed you what a flash cache does to jitter on one board. The
  S3 adds external flash behind its cache, plus a radio core on the same bus:
  which of the two effects dominates? Two sentences in the RET.
- **Evidence:** comparison table + captures.

### Task C — The first thread
- Migrate **only the sampling task** to a kernel thread (`K_THREAD_DEFINE`,
  preemptive priority, `k_msgq` toward the main loop) following the talk's mapping.
  Everything else stays in the superloop.
- Measure: does the blocking command still ruin the sampling?
- **Evidence:** capture with the blocking command active + the thread's jitter.

## What about FreeRTOS?

Task C would be `xTaskCreate(sample_task, "sample", stack, NULL, prio, NULL)` +
`xQueueSend`/`xQueueReceive` — different API, same concept. What FreeRTOS does
**not** have is Task A: without devicetree, moving from STM32 to ESP32 means
changing SDKs, not overlays.

## Deliverables (RET)

- **§3 Week-3 evidence:** two-silicon table + explanation of the new jitter.
- **§1:** `C_i` re-measured on the S3 (the task set's final platform).

## Rubric (100 pts)

| | pts |
|---|---|
| **Execution** — port via overlay, no C touched (15) · sampling thread working (25) | 40 |
| **Evidence** — complete comparison table (20) · port diff-stat (10) | 30 |
| **Analysis** — correct reading of L476 vs. S3 jitter (15) · why the thread survives the blocking command (15) | 30 |
