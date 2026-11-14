# RET — Timing Evidence Report

* **Team:** `Johan Sebastian Mendieta Dilbert, Jeronimo Novoa Giraldo`
* **Boards:** Nucleo-L476RG (STM32L476RG) / ESP32-S3 DevKitC (esp32s3_devkitc_esp32s3_procpu)
* **AI use:** Week 6 documentation integration used OpenAI GPT-5.6 to organize existing evidence, compare the report with authorized public sources, and review consistency. AI did not produce, execute, or validate any timing measurement.

**Living document** — updated every week; handed in at the workshop (week 7) and
at the close (week 14).

**House rule:** *"show me the trace"* — every timing claim cites a measurement.

## 1. The system and its task set

Requirements first — one sentence each, EARS style (*when/while `<condition>`, the
system shall `<response>` within `<deadline>*`) — then the task that implements them:

| ID | Requirement |
| --- | --- |
| REQ-CTRL-01 | While the system is irrigating, the control loop shall run every $10\text{ ms}$ with its deadline equal to its period. |
| REQ-CTRL-02 | When measured pressure exceeds the threshold, the system shall close the valve in less than $5\text{ ms}$. |
| REQ-CTRL-03 | While the system is powered on, `task_sampling()` shall run with a nominal period of $1000\ \mu\text{s}$ and maximum jitter of $\le 50\ \mu\text{s}$. |
| REQ-CONS-01 | When `calib` is requested, the firm console command shall complete within $1\text{ s}$. |
| REQ-PERF-06 | While the system operates under nominal workload, the loop tick backlog (`backlog_peak`) shall not exceed $2\text{ ticks}$ ($2000\ \mu\text{s}$). |

| Task | Req. | Type (H/F/S) | Period | Deadline | Measured $C_i$ | How it was measured |
| --- | --- | --- | --- | --- | --- | --- |
| `task_sampling` | REQ-CTRL-03 | Hard | $1\text{ ms}$ | $1\text{ ms}$ | $0.75\ \mu\text{s}$ `[GPIO]` / $33.84\ \mu\text{s}$ `[Trace]` | D3 / GPIO4 + logic analyzer / CTF trace |
| `task_control` | REQ-CTRL-01 | Hard | $10\text{ ms}$ | $10\text{ ms}$ | $1.35\ \mu\text{s}$ `[GPIO]` / $82.78\ \mu\text{s}$ `[Trace]` | D4 / GPIO5 + logic analyzer / CTF trace |
| `task_flow_batch` | N/A | Unclassified deferred work | Every 100 flow pulses | Not specified | $12.81\ \mu\text{s}$ `[GPIO]` | D7 / GPIO15 + logic analyzer |
| `task_console` | REQ-CONS-01 | Firm | Polled | $1\text{ s}$ for `calib` | $5794.55\ \mu\text{s}$ `[GPIO]` / $51.66\ \mu\text{s}$ `[Trace]` | D5 / GPIO6 + logic analyzer / CTF trace |
| `task_display` | N/A | Soft | $500\text{ ms}$ | N/A | $47.39\ \mu\text{s}$ `[Trace]` | D8 / GPIO17 + logic analyzer / CTF trace |
| `task_telemetry` | N/A | Soft | $1\text{ s}$ | N/A | $62.41\ \mu\text{s}$ `[Trace]` | D6 / GPIO7 + logic analyzer / CTF trace |

*Note: Instrument annotation bracket `[GPIO]` indicates physical pin toggling measurement; `[Trace]` indicates software kernel instrumentation hook measurement via CTF.*

The public [`PROJECT_SCENARIO.md`](https://github.com/jomendietad/4101137-real-time-systems/blob/main/PROJECT_SCENARIO.md), Week 2 guide, and firmware README support the listed classifications and deadlines for sampling, control, console, telemetry, display, and the e-stop path. The hard flow classification applies to the pulse-counting ISR; `task_flow_batch()` is deferred batch math and the public README leaves its type and deadline unspecified. The D3–D8 order and ESP32-S3 GPIO mapping come from the public instrumentation overlays. The e-stop is an end-to-end path detected in sampling and actuated at the valve; it is not represented as an independent periodic task.

## 2. ADRs

### ADR-001 — Node Architecture: Multithreaded Kernel

* **Context:** In Weeks 2 and 3, the cooperative superloop architecture suffered from severe temporal interference. Synchronous blocking operations such as `calib` (1000 iterations of 400 µs busy-waits in `task_console`) stalled execution in `main()`, causing the control period to stretch from 10 ms to 102 ms (and up to >1000 ms), while sampling jitter exceeded 14000 µs.
* **Decision:** Migrate all application tasks to a fully preemptive multi-threaded model on the Zephyr RTOS kernel, assigning Rate-Monotonic priorities (`sampling_thread`=2, `control_thread`=3, `flow_wq`=5, `display_thread`=7, `telemetry_thread`=8, `console_thread`=9) and deploying a dedicated workqueue for flow pulse processing.
* **Justification (with numbers):** The A/B comparison reports sampling jitter falling from $14200.0\ \mu\text{s}$ (superloop) to **$3.2\ \mu\text{s}$** (kernel RTOS), while the maximum control period returns from $102.0\text{ ms}$ to **$10.02\text{ ms}$**. A visible sampling-to-control GPIO gap of **$2.4\ \mu\text{s}$** was measured. The previously quoted **$0.312\%$** CPU penalty is only an unvalidated estimate because its assumed $\sim1300$ switches/s was not measured. Max reported user-thread stack utilization remained 50% (`console_tid` at 1024 / 2048 bytes).
* **Status:** Accepted.

### ADR-002 — Cross-priority mutual exclusion uses PI mutexes

* **Context:** A binary semaphore used as a lock allows a medium-priority thread to prolong the blocking of high-priority control while low-priority console owns the resource.
* **Decision:** Resources shared across priorities shall use a Zephyr `k_mutex` with priority inheritance, not a bare binary `k_sem` used as a lock.
* **Justification:** No-contention response-time analysis gives $R_{ctrl}=116.62\ \mu\text{s}$. With blocking, the full recurrence includes one sampling interference term per crossed millisecond; against the canonical $10\text{ ms}$ control deadline, the boundary calculation gives $B\le9578.82\ \mu\text{s}$ when ten sampling jobs interfere. The local captures are present but unannotated and do not measure $B$ or establish the complete no-contention/inversion/PI sequence.
* **Status:** Policy accepted; measured $B$ and empirical before/after validation remain pending.

### ADR-005 — Remove `calib` spinning and close the valve in the detecting job

* **Context:** The documented pre-fix `calib` loop demands approximately $400\text{ ms}$ of CPU, and the pre-fix e-stop path can wait one sampling period, one control period, and one PWM update (analytical bound: $12\text{ ms}$), which exceeds REQ-CTRL-02.
* **Decision:** Replace each `k_busy_wait(400)` settle with `k_usleep(400)`, and set `duty_pct = 0` in the overpressure branch of `task_sampling()` so the same sampling job can drive the valve low.
* **Justification:** The period-based post-fix estimate is approximately $1.034\text{ ms}$, below $5\text{ ms}$, and sleeping releases CPU capacity. The current public `firmware/kernel/src/main.c` still shows the pre-fix implementation, while the local Task C capture appears post-fix but lacks a before capture, cursors, and five repetitions. Therefore neither implementation completion nor physical maxima are claimed here.
* **Status:** Design accepted; source integration and measured before/after verification remain pending.

## 3. Evidence by week

Each entry cites the `REQ`(s) it verifies.

### Week 1 — Bring-up: the team, the board, the toolchain

#### Iteration Loop Measurement Matrix

| Quantity | Prediction | Measured value | Target |
| --- | ---: | ---: | ---: |
| Edit $\rightarrow$ build $\rightarrow$ flash $\rightarrow$ console message, one member's laptop | $35.0\text{ s}$ | **$27.4\text{ s}$** | $<60\text{ s}$ |
| Same, incremental build (one line changed, no `-p`) | $10.0\text{ s}$ | **$6.8\text{ s}$** | — |
| Members with `blinky` running from their own laptop | — | **$2/2$** | all |

#### Toolchain and board bring-up verification (Tasks A, B, and C)

1. **Cross-check on `nucleo_l476rg` and `native_sim`:** Both team members built and ran `zephyr/samples/basic/blinky` on the host simulator and then on the Nucleo-L476RG. The on-board LD2 (PA5) toggled at $1\text{ Hz}$ after flashing from each laptop.
2. **Serial console and build cycle:** The ST-Link virtual COM port was used at `115200 8N1`. The reported pristine cycle was $27.4\text{ s}$ ($23.1\text{ s}$ build plus $4.3\text{ s}$ flash/reset); the incremental cycle was $6.8\text{ s}$ ($2.5\text{ s}$ build plus $4.3\text{ s}$ flash/reset), a reported $4.03\times$ reduction.

   ```text
   *** Booting Zephyr OS build v4.1.0 ***
   Hello World! SoilSense Control Team (Johan & Jeronimo) - nucleo_l476rg
   ```

3. **Initial RET and task set:** The team initialized the report and the original task set before Week 2 instrumentation. These historical Week 1 measurements are imported from `ret (1).md`; no additional Week 1 evidence was supplied in this local unit.

### Week 2 — superloop baseline (Nucleo-L476RG)

#### Flow Diagram (10 Boxes)

```

┌─────────────────────────────────────────────────┐
│          1 kHz Timer ISR (Hardware)             │
│      atomic_inc(&ticks_pending)                 │
│      atomic_set(&backlog_peak, backlog)         │
└─────────────────────────────────────────────────┘
│
▼

┌───────────────────────────────────────────────────────────────┐
│ [1] START: main() -> enter while (1)                          │
└──────────────────────────────┬────────────────────────────────┘
│
▼
┌───────────────────────────────────────────────────────────────┐
│ [2] task_console()               (GPIO D5 HIGH -> LOW)        │
└──────────────────────────────┬────────────────────────────────┘
│
▼
┌───────────────────────────────────────────────────────────────┐
│ [3] task_display()               (GPIO D8 HIGH -> LOW)        │
└──────────────────────────────┬────────────────────────────────┘
│
▼
┌───────────────────────────────────────────────────────────────┐
│ [4] task_telemetry()             (GPIO D6 HIGH -> LOW)        │
└──────────────────────────────┬────────────────────────────────┘
│
▼
┌───────────────────────────────────────────────────────────────┐
│ [5] DECISION: atomic_get(&ticks_pending) > 0?                 │
└──────────────┬─────────────────────────────────┬──────────────┘
│ YES                                            │ NO
▼                                                │
┌──────────────────────────────┐                 │
│ atomic_dec(&ticks_pending)   │                 │
└──────────────┬───────────────┘                 │
│                                                │
▼                                                │
┌──────────────────────────────┐                 │
│ [6] task_sampling()          │                 │
│      (GPIO D3 HIGH -> LOW)   │                 │
└──────────────┬───────────────┘                 │
│                                                │
▼                                                │
┌──────────────────────────────┐                 │
│ [7] ++control_div >= 10?     │                 │
└──────┬───────────────┬───────┘                 │
│ YES              │ NO                          │
▼                  │                             │
┌────────────────────┐ │                         │
│ [8] task_control() │ │                         │
│   (GPIO D4 H -> L) │ │                         │
└──────┬─────────────┘ │                         │
│                │                               │
└───────┬────────┘                               │
│                                                │
└───────────┬─────────────────────┘
│
▼
┌───────────────────────────────────────────────────────────────┐
│ [9] task_flow_batch()            (GPIO D7 HIGH -> LOW)        │
└──────────────────────────┬────────────────────────────────────┘
│
▼
┌───────────────────────────────────────────────────────────────┐
│ [10] END OF ITERATION -> return to [1]                         │
└───────────────────────────────────────────────────────────────┘


```

#### Measurements Table

| Measurement | Your value | Reference / Verified REQ |
| --- | --- | --- |
| Actual sampling period (nominal 1 kHz): average | **997.89 µs** | $1000\ \mu\text{s}$ (verifies **REQ-CTRL-03**) |
| Sampling jitter: max over $\ge 30\text{ s}$ | **5581.46 µs** | Course baseline |
| ISR $\rightarrow$ loop-service latency (`task_flow_batch`) | **12.81 µs** (average) / **29.42 µs** (maximum) | Historical measured baseline; the public guide specifies no canonical deadline for this row |
| Sampling jitter, max, **with the flash cache off** | **6798.46 µs** | Gap vs row 2 is the ART accelerator |
| Sampling jitter **with the blocking command active** | **6663.79 µs** | Violates **REQ-CTRL-03**; `calib` completion against **REQ-CONS-01** was not timed here |
| `backlog_peak`, idle $\rightarrow$ during the blocking command | **1 $\rightarrow$ 6 ticks** | Verifies **REQ-PERF-06** |

#### Root-Cause Analysis & Observations

* **Impact of Disabling Flash Cache (`nocache.conf`):** Disabling the ART Accelerator and instruction prefetch buffer forces the STM32L476RG CPU to introduce wait states on every Flash fetch. This increases task execution times $C_i$ (e.g., `task_sampling` increases from $4.94\ \mu\text{s}$ to $7.26\ \mu\text{s}$), widening the loop pass duration and increasing worst-case sampling jitter.
* **Degradation under Blocking Command (`calib`):** In a non-preemptive superloop, tasks execute cooperatively to completion. The calibration routine inside `task_console()` (`firmware/superloop/src/main.c`) performs a synchronous blocking delay. While trapped in this function, the CPU cannot return to `main()` to evaluate `ticks_pending` or execute `task_sampling()`, causing $1\text{ kHz}$ timer ticks to backlog and severely degrading period determinism.

#### Local analyzer evidence

![Lab 2 Task A analyzer overview](<Labs RTS/Evidencias lab 2 RTS/Session A.png>)

<details>
<summary>Lab 2 Tasks B and C captures</summary>

![Lab 2 Task B capture 1 — condition not confirmed](<Labs RTS/Evidencias lab 2 RTS/Session B 1.png>)

![Lab 2 Task B capture 2 — condition not confirmed](<Labs RTS/Evidencias lab 2 RTS/Session B 2.png>)

![Lab 2 Task C blocking-command capture](<Labs RTS/Evidencias lab 2 RTS/Session C.png>)

</details>

The B1/B2 screenshots show analyzer activity, but their labels and visible channels do not establish which one is the default build and which one is the cache-off build. They are presence evidence only and are not used to assign either screenshot to a quantitative condition.

#### Baseline Reading

> The cooperative superloop can execute periodic sampling efficiently at idle, but it lacks temporal isolation: blocking I/O or a long command degrades determinism for the entire loop.

### Week 3 — S3 baseline and silicon comparison

#### Port Diff-Stat Evidence (Task A)

Porting the superloop application from the STM32L476RG to the ESP32-S3 was accomplished entirely through a Zephyr Devicetree overlay (`firmware/superloop/boards/esp32s3_devkitc_esp32s3_procpu.overlay`) without modifying any C source code in `firmware/superloop/src/main.c`.

```text
 firmware/superloop/boards/esp32s3_devkitc_esp32s3_procpu.overlay | 64 ++++++++++++++++++++++++++++++++++++++++++++
 1 file changed, 64 insertions(+)


```

* **Files changed:** 1 (`.overlay`)
* **Lines inserted:** 64
* **C source lines modified:** **0 lines of C**

#### Three-Column Measurement Comparison Matrix

| Measurement | L476RG (wk 2) | Prediction, S3 superloop | S3 superloop | S3 + sampling thread |
| --- | --- | --- | --- | --- |
| Max sampling jitter, $\ge 30\text{ s}$ | $5581.46\ \mu\text{s}$ | $5.0\ \mu\text{s}$ | $2.0\ \mu\text{s}$ | $1.2\ \mu\text{s}$ |
| Max sampling jitter, `calib` active | $6663.79\ \mu\text{s}$ | $15000\ \mu\text{s}$ | $14200\ \mu\text{s}$ | $3.5\ \mu\text{s}$ |
| `backlog_peak`, `calib` active | $6\text{ ticks}$ | $15\text{ ticks}$ | $14\text{ ticks}$ | $0\text{ ticks}$ |
| `lat_peak_us` (tick $\to$ thread start) | — | — | — | $6.0\ \mu\text{s}$ |
| Max control period (`instr_ctrl`), `calib` active | — | $105\text{ ms}$ | $102\text{ ms}$ | $1293\text{ ms}$ |

#### Silicon Comparison & Multi-threading Analysis

1. **Hardware UART Driver Architecture (`uart_stm32` vs `uart_esp32`):**
In `uart_stm32_poll_out`, character transmission performs a synchronous busy-wait spin loop checking the Transmit Data Register Empty (`TXE`) flag, holding CPU execution for the duration of every character transmission at $115200\text{ baud}$. In contrast, `uart_esp32_poll_out` writes bytes directly into the ESP32-S3's 128-byte hardware TX FIFO buffer. Telemetry transmission calls return almost instantaneously without CPU spin-locking as long as the hardware FIFO does not fill up, eliminating character-by-character UART latency from the superloop path.
2. **Clock Speed (240 MHz) vs. SPI Flash I-Cache Impact:**
Increasing clock frequency from 80 MHz to 240 MHz accelerates purely computation-bound task execution ($C_i$). However, because the ESP32-S3 executes code out of external SPI Flash through an instruction cache (I-Cache), cold-path execution or cache misses introduce non-deterministic tail latency spikes during instruction fetches, preventing a strictly linear scaling reduction in worst-case response times.
3. **Sampling Thread Isolation vs. Control Loop Degradation under `calib`:**

* **Sampling:** Moving `task_sampling()` into `sampling_thread` with priority `SAMPLING_PRIO = 2` (preemptive over `main` running at `CONFIG_MAIN_THREAD_PRIORITY = 10`) guarantees that when `tick_isr` posts to `tick_q`, the Zephyr RTOS scheduler immediately preempts `main`—even during a blocking `calib` loop—executes `task_sampling()`, and resumes `main`. Thus, sampling jitter drops to $3.5\ \mu\text{s}$ and `backlog_peak` remains at $0\text{ ticks}$.
* **Control:** `task_control()` remains inside the lower-priority `main` superloop. When `calib` performs 1000 rounds of `k_busy_wait(400)`, `main` is blocked from evaluating `control_pending`, causing the maximum control period to stretch to $1293\text{ ms}$.

4. **Shared State Access and Synchronization:**
Naturally aligned scalar reads and writes may avoid torn values on the target, but that property does not replace synchronization. It does not provide ordering, compound-operation atomicity, race freedom, or a coherent multi-field snapshot. Shared state such as `pressure_mv`, `estop`, and `setpoint_mv` still requires an explicit concurrency design (atomics with defined memory semantics, ownership, or a suitable lock/message mechanism).

### Week 4 — The full migration and the A/B

#### A/B Measurement Matrix (ESP32-S3)

| Measurement (S3) | S3 superloop (wk 3) | Prediction, kernel | Kernel (today) |
| --- | --- | --- | --- |
| Max sampling jitter | $2.0\ \mu\text{s}$ | $1.5\ \mu\text{s}$ | **$1.1\ \mu\text{s}$** |
| Max sampling jitter with `calib` | $14200.0\ \mu\text{s}$ | $5.0\ \mu\text{s}$ | **$3.2\ \mu\text{s}$** |
| Max control period with `calib` | $102.0\text{ ms}$ | $10.5\text{ ms}$ | **$10.02\text{ ms}$** |
| ISR → thread latency (`lat_peak_us`) | — | $5.0\ \mu\text{s}$ | **$6.0\ \mu\text{s}$** |
| Visible context-switch gap (Task C) | — | $3.0\ \mu\text{s}$ | **$2.4\ \mu\text{s}$** |
| Control thread stack high-water mark | — | $384\text{ bytes}$ | **$288\text{ bytes} / 1024\text{ allocated}\ (28\%)$** |

#### Context Switch Overhead Analysis (Task C)

* **Measured Gap (`instr_samp` falling → `instr_ctrl` rising):** $2.4\ \mu\text{s}$.
* **Exploratory overhead estimate:** If the workload were 1000 sampling activations/s + 100 control activations/s + an assumed ~200 background switches/s, the total would be approximately 1300 switches/s:

$$\text{CPU Overhead} = 1300\text{ switches/s} \times 2.4\ \mu\text{s/switch} = 3120\ \mu\text{s/s} \approx 0.312\%$$

This yields **0.312%** only as an unvalidated estimate. The $\sim200$ background switches/s and total switch rate were not measured, and the visible GPIO gap is not necessarily the complete per-switch scheduler cost, so this calculation is not accepted as a capacity measurement.

#### Stack High-Water Marks (Task D)

Reported via `thread_analyzer_print()` under maximum operational load (`calib` active + synthetic flow events):

```text
Thread analyze:
 flow_wq        : STACK: unused  720 usage  304 / 1024 ( 29 %)
 telemetry_tid  : STACK: unused 1328 usage  720 / 2048 ( 35 %)
 sampling_tid   : STACK: unused 1232 usage  304 / 1536 ( 19 %)
 display_tid    : STACK: unused 1776 usage  272 / 2048 ( 13 %)
 control_tid    : STACK: unused  736 usage  288 / 1024 ( 28 %)
 console_tid    : STACK: unused 1024 usage 1024 / 2048 ( 50 %)
 sysworkq       : STACK: unused  208 usage  304 /  512 ( 59 %)
 idle           : STACK: unused   48 usage  208 /  256 ( 81 %); CPU: 97-98%
 ISR0           : STACK: unused   64 usage  448 /  512 ( 87 %)


```

* **Closest User Thread to Limit:** `console_tid` at 50% stack usage (1024 / 2048 bytes) due to string formatting and CLI input parsing.

### Week 5 — Tracing: seeing what the analyzer can't

#### Verification & Setup (Task A)

CTF tracing was enabled on the ESP32-S3 streaming trace packets out of the UART port at 4 Mbaud. System threads (`sampling_tid`, `control_tid`, `console_tid`, `display_tid`, `telemetry_tid`) and hardware interrupts (`ISR`) were successfully captured and identified in Google Perfetto (`ui.perfetto.dev`).

<details>
<summary>Task A — named-thread trace overview (7 screenshots)</summary>

![Lab 5 Task A overview](<Labs RTS/Evidencias lab 5 RTS/Task A/Captura de pantalla_20261107_233006.png>)
![Lab 5 Task A view 2](<Labs RTS/Evidencias lab 5 RTS/Task A/Captura de pantalla_20261107_233024.png>)
![Lab 5 Task A view 3](<Labs RTS/Evidencias lab 5 RTS/Task A/Captura de pantalla_20261107_233049.png>)
![Lab 5 Task A view 4](<Labs RTS/Evidencias lab 5 RTS/Task A/Captura de pantalla_20261107_233500.png>)
![Lab 5 Task A view 5](<Labs RTS/Evidencias lab 5 RTS/Task A/Captura de pantalla_20261107_233519.png>)
![Lab 5 Task A view 6](<Labs RTS/Evidencias lab 5 RTS/Task A/Captura de pantalla_20261107_233541.png>)
![Lab 5 Task A view 7](<Labs RTS/Evidencias lab 5 RTS/Task A/Captura de pantalla_20261107_233559.png>)

</details>

#### Measurement Comparison Matrix (Trace vs. GPIO)

| Measurement | Prediction | Your value (Trace / Analyzer) | Contrast / Reference |
| --- | --- | --- | --- |
| Context switch per the trace | $15.0\ \mu\text{s}$ | **$30.69\ \mu\text{s}$–$32.52\ \mu\text{s}$** | vs. GPIO (Week 4): **$2.4\ \mu\text{s}$** |
| Tick ISR → sampling start per the trace | $10.0\ \mu\text{s}$ | **$31.50\ \mu\text{s}$–$32.52\ \mu\text{s}$** | vs. `lat_peak_us`: **$6.0\ \mu\text{s}$** |
| Per-thread CPU load (%) in normal operation | 100% total | `sampling_tid`: **0.81%**; `control_tid`: **0.20%**; `console_tid`: **0.30%**; `display_tid`: **<0.05%** | sums $\approx100\%$ with `idle` (**98.64%**) and tracer |
| Tracing overhead on max sampling jitter | $2.0\ \mu\text{s}$ | **$2.10\ \mu\text{s}$** ($\text{ON: } 3.20\ \mu\text{s} \text{ vs. OFF: } 1.10\ \mu\text{s}$) | analyzer, tracing ON vs. OFF |

#### The Story of One Preemption (Task B)

1. **Preemption Execution Sequence:**
* **Tick ISR:** Hardware clock tick triggers `ISR: Tick / Hardware` at $t = 8024.90\text{ ms}$.


* **Queue Posting (`k_msgq_put`):** The ISR executes `k_msgq_put`, placing a message into the queue and unblocking `sampling_tid`.


* **Console Preempted:** `console_tid` (running at priority 9) is immediately preempted by the RTOS scheduler and transitions from `RUNNING` to `READY` state.


* **Sampling Execution:** `sampling_tid` (priority 2) enters `RUNNING` state and executes for **$34.02\ \mu\text{s} - 34.17\ \mu\text{s}$**.


* **Queue Pending (`k_msgq_get`):** `sampling_tid` finishes processing and calls `k_msgq_get`, blocking itself on an empty queue (`PENDING` state).


* **CPU Return:** Control yields back to `console_tid`, resuming CLI parsing.

<details>
<summary>Task B — preemption trace selections (8 screenshots)</summary>

![Lab 5 Task B selection 1](<Labs RTS/Evidencias lab 5 RTS/Task B/Captura de pantalla_20261107_233844.png>)
![Lab 5 Task B selection 2](<Labs RTS/Evidencias lab 5 RTS/Task B/Captura de pantalla_20261107_233929.png>)
![Lab 5 Task B selection 3](<Labs RTS/Evidencias lab 5 RTS/Task B/Captura de pantalla_20261107_233952.png>)
![Lab 5 Task B selection 4](<Labs RTS/Evidencias lab 5 RTS/Task B/Captura de pantalla_20261107_234003.png>)
![Lab 5 Task B selection 5](<Labs RTS/Evidencias lab 5 RTS/Task B/Captura de pantalla_20261107_234015.png>)
![Lab 5 Task B selection 6](<Labs RTS/Evidencias lab 5 RTS/Task B/Captura de pantalla_20261107_234048.png>)
![Lab 5 Task B selection 7](<Labs RTS/Evidencias lab 5 RTS/Task B/Captura de pantalla_20261107_234101.png>)
![Lab 5 Task B selection 8 — ambiguous 35 ns selection](<Labs RTS/Evidencias lab 5 RTS/Task B/Captura de pantalla_20261107_234114.png>)

</details>

Task B8 displays a `35 ns` selection whose boundaries and semantic event are ambiguous. It is retained for completeness but is not used as quantitative evidence.




2. **Analysis of `calib` Trace Duration Limit (~75 ms):**
* Typing `calib` triggers a massive stream of console output logs and high-frequency UART activity. The CTF trace subsystem uses an in-memory ring buffer to log events before UART transmission at 4 Mbaud. The volume of calibration events quickly exceeds the UART output rate, causing a **trace buffer overflow**. As a result, only the first $\sim 75\text{ ms}$ of trace events are preserved before dropped-event flags are set.


3. **Behavior at the 10th 1 kHz sampling tick ($t \approx 10\text{ ms}$):**
* The canonical kernel source uses a $1\text{ ms}$ sampling period and releases control every tenth sample, so `k_sem_give` occurs every $10\text{ ms}$ ($100\text{ Hz}$), not every $41.66\text{ ms}$ ($24\text{ Hz}$). The trace story is that the scheduler can transfer execution to the higher-priority `control_tid` before returning to lower-priority work. The screenshots are not treated as proof of the previously reported $81.37$–$83.88\ \mu\text{s}$ range unless the relevant event boundaries are explicitly selected.




4. **Why Trace vs. GPIO Measurements Differ (Analysis):**
* **GPIO Analyzer:** Measures physical pin-level instrumentation with substantially less semantic visibility than tracing. The report records a **$2.4\ \mu\text{s}$** visible gap and **$6.0\ \mu\text{s}$** interrupt latency; these values include the firmware's GPIO instrumentation and should not be described as literally overhead-free.
* **Software Tracing (CTF):** Captures event logging hooks inside kernel functions. The trace values (**$30.69\ \mu\text{s} - 32.52\ \mu\text{s}$** for context switches and **$31.50\ \mu\text{s} - 32.52\ \mu\text{s}$** for ISR-to-thread latency) include the software overhead of serializing CTF event records, updating timestamp registers, and executing tracing macros inside kernel schedulers.


#### Task C — The price of looking (Tracing Overhead)

- **Measurement Protocol:** Sampling jitter was measured via GPIO (`instr_samp`) using the Analog Discovery 2 logic analyzer under identical workload conditions (`calib` active) with CTF tracing enabled (`trace.conf`) and disabled baseline.

| Condition | Max Sampling Jitter (AD2 / GPIO) | Observation / Cause |
| --- | --- | --- |
| **Tracing OFF** | **$1.10\ \mu\text{s}$** | Pure RTOS scheduling & tick ISR latency baseline |
| **Tracing ON** | **$3.20\ \mu\text{s}$** | Software perturbation introduced by CTF event logging hooks |
| **Delta ($\Delta$ Overhead)** | **$+2.10\ \mu\text{s}$** | Net timing penalty imposed by the tracing subsystem |

<details>
<summary>Task C — tracing OFF/ON analyzer series (6 screenshots)</summary>

![Lab 5 Task C tracing OFF 1](<Labs RTS/Evidencias lab 5 RTS/Task C/Tracing Off.png>)
![Lab 5 Task C tracing OFF 2](<Labs RTS/Evidencias lab 5 RTS/Task C/Tracing Off(1).png>)
![Lab 5 Task C tracing OFF 3](<Labs RTS/Evidencias lab 5 RTS/Task C/Tracing Off(2).png>)
![Lab 5 Task C tracing ON 1](<Labs RTS/Evidencias lab 5 RTS/Task C/Tracing On.png>)
![Lab 5 Task C tracing ON 2](<Labs RTS/Evidencias lab 5 RTS/Task C/Tracing On(1).png>)
![Lab 5 Task C tracing ON 3](<Labs RTS/Evidencias lab 5 RTS/Task C/Tracing On(2).png>)

</details>

These screenshots distinguish tracing OFF and ON waveform series, but they do not display pulse-statistics maxima or cursor measurements. They do not independently establish the reported maximum jitter values; the numerical table requires the underlying analyzer statistics/export for validation.

1. **Quantification of Overhead:**
   The report records a **$2.10\ \mu\text{s}$** increase (from $1.10\ \mu\text{s}$ to $3.20\ \mu\text{s}$). If confirmed by the missing analyzer statistics, the result remains below the $50\ \mu\text{s}$ jitter limit in **REQ-CTRL-03**. The supplied screenshots alone do not establish those maxima.

2. **Root-Cause Analysis (Why the Observer Perturbs):**
   - **Trace Hook Execution:** Every thread context switch, queue operation, and timer ISR executes inline CTF logging macros that write timestamped event records to memory.
   - **Bus & Memory Contention:** High-frequency binary trace buffer writes compete for internal bus cycles on the ESP32-S3, introducing micro-stalls during instruction fetch and memory access.
   - **UART FIFO Pressures:** High-rate serial streaming over `uart0` consumes additional CPU cycles for UART TX buffer management.

3. **Synthesis (GPIO vs. Tracing):**
   - **GPIO + Logic Analyzer ("The Judge of Numbers"):** Low-intrusion physical observation of instrumented task boundaries. It reports timing at the pins but still includes the cost and placement of GPIO toggles.
   - **CTF Tracing ("The Judge of the Story"):** Intrusive software measurement that reveals thread names, preemption instances, state transitions, and root causes that an external pin analyzer cannot see. The report records $+2.10\ \mu\text{s}$ overhead, but the supplied screenshots do not independently establish that maximum.


### Week 6 — Schedulability, response time, and priority inversion

This section completes the detailed analysis supporting §4 with the data available in this report and in the Week 6 guide. Local analyzer captures are present, but where they are unannotated or omit the required duration, cursor, counter, or comparison case, the corresponding numerical result remains explicitly open.

#### Task A — Fig. 4.13 under RM and EDF

The reproduced periodic set has implicit deadlines:

| Task | $C_i$ | $T_i = D_i$ | RM priority |
| --- | ---: | ---: | --- |
| $\tau_1$ | $2\text{ ms}$ | $5\text{ ms}$ | Higher |
| $\tau_2$ | $4\text{ ms}$ | $7\text{ ms}$ | Lower |

The total utilization is

$$
U = \sum_i \frac{C_i}{T_i}
  = \frac{2}{5} + \frac{4}{7}
  = 0.4 + 0.5714286
  = 0.9714286 \approx 97.14\%.
$$

For two tasks, the Liu & Layland sufficient RM bound is

$$
U_{LL}(2) = 2\left(2^{1/2}-1\right) \approx 0.828427.
$$

Because $0.9714286 > 0.828427$, the utilization-bound test does not guarantee RM schedulability. The hyperbolic sufficient test also fails:

$$
\prod_i\left(1+\frac{C_i}{T_i}\right)
= \left(1+\frac{2}{5}\right)\left(1+\frac{4}{7}\right)
= 1.4 \times 1.5714286
= 2.2 > 2.
$$

The exact RM response-time analysis establishes the miss. For $\tau_1$,

$$
R_1 = C_1 = 2\text{ ms} \le D_1 = 5\text{ ms}.
$$

For $\tau_2$, the fixed-point recurrence is

$$
R_2^{(k+1)} = C_2 + \left\lceil\frac{R_2^{(k)}}{T_1}\right\rceil C_1.
$$

Substitution gives

$$
R_2^{(0)}=4,\qquad
R_2^{(1)}=4+\left\lceil\frac{4}{5}\right\rceil 2=6,\qquad
R_2^{(2)}=4+\left\lceil\frac{6}{5}\right\rceil 2=8\text{ ms}.
$$

Thus $R_2=8\text{ ms}>D_2=7\text{ ms}$. The first schedule shows the same result directly: $\tau_1$ runs from $0$ to $2\text{ ms}$, $\tau_2$ receives only $3\text{ ms}$ from $2$ to $5\text{ ms}$, and the second release of $\tau_1$ runs from $5$ to $7\text{ ms}$. At $t=7\text{ ms}$, $\tau_2$ still has $1\text{ ms}$ of computation pending and misses its first deadline.

Under EDF, the implicit-deadline processor-utilization criterion is both applicable and exact:

$$
U = 0.9714286 \le 1.
$$

Therefore the task set is schedulable under ideal preemptive EDF, despite failing under RM. In the Zephyr experiment, each absolute deadline must be assigned immediately after wake-up and before the thread can be selected using a stale deadline.

![Lab 6 Task A capture 1 — scheduler not identified](<Labs RTS/Evidencias lab 6 RTS/Task A/task a.png>)

![Lab 6 Task A capture 2 — scheduler and duration not identified](<Labs RTS/Evidencias lab 6 RTS/Task A/task a2.png>)

The two Task A captures are present but insufficient/unannotated: they show `Tau_1`/`Tau_2` activity but do not identify RM versus EDF, annotate the RM miss at $t=7\text{ ms}$, show a miss counter, or establish a continuous 60 s EDF run. The theoretical verdict is established; board-level verification remains pending.

#### Task B — Priority inversion and the PI-mutex bound

The established priority order is `sampling_thread` (2), `control_thread` (3), medium-priority background work (`flow_wq`=5, `display_thread`=7, and `telemetry_thread`=8), and `console_thread` (9). Consequently, a binary `k_sem` used as a lock permits the classic unbounded-inversion sequence required by the guide:

1. `console_thread` takes the semaphore and enters the low-priority critical section.
2. `control_thread` preempts it, requests the semaphore, and blocks.
3. A medium-priority thread preempts `console_thread` without using the semaphore.
4. `control_thread` remains blocked until all such interference ends and `console_thread` can run again and release the semaphore.

Using the measured trace execution times for the known periodic tasks, the no-contention response-time recurrence for control is

$$
R_{ctrl}^{(k+1)} = C_{ctrl}
+ \left\lceil\frac{R_{ctrl}^{(k)}}{T_{samp}}\right\rceil C_{samp},
$$

with $C_{ctrl}=82.78\ \mu\text{s}$, $C_{samp}=33.84\ \mu\text{s}$, and $T_{samp}=1000\ \mu\text{s}$. Starting from $R_{ctrl}^{(0)}=82.78\ \mu\text{s}$,

$$
R_{ctrl}^{(1)}
=82.78+\left\lceil\frac{82.78}{1000}\right\rceil 33.84
=116.62\ \mu\text{s},
$$

and the next iteration is unchanged. Therefore the known periodic set bounds the no-contention control response at $116.62\ \mu\text{s}$, below its canonical $10\text{ ms}$ deadline.

Replacing the semaphore with `k_mutex` activates Zephyr priority inheritance: while control is blocked, the console lock owner temporarily inherits control's priority. Medium-priority work can no longer extend the blocking interval, so control is blocked by at most one low-priority critical section of measured width $B$:

$$
R_{ctrl}^{(k+1)} = 82.78 + B
+ \left\lceil\frac{R_{ctrl}^{(k)}}{1000}\right\rceil 33.84
\quad [\mu\text{s}].
$$

For short blocking that keeps the response below one sampling period, this reduces to $R_{ctrl}=116.62+B\ \mu\text{s}$. The canonical control deadline is nevertheless $10\text{ ms}$, not $1\text{ ms}$. At the deadline boundary, ten sampling jobs interfere, giving

$$
B_{max}=10000-82.78-10(33.84)=9578.82\ \mu\text{s}.
$$

<details>
<summary>Task B — local lock captures (3 screenshots)</summary>

![Lab 6 Task B capture 1](<Labs RTS/Evidencias lab 6 RTS/Task B/task b.png>)
![Lab 6 Task B capture 2](<Labs RTS/Evidencias lab 6 RTS/Task B/task b2.png>)
![Lab 6 Task B capture 3](<Labs RTS/Evidencias lab 6 RTS/Task B/task b3.png>)

</details>

The three captures are present but insufficient/unannotated. They do not provide cursors for the low thread's critical-section width $B$, label the no-contention/semaphore/mutex regimes, or demonstrate the full low-takes-lock → high-blocks → medium-preempts sequence. A numerical inversion delay, a numerical mutex response, and empirical closure of ADR-002 cannot yet be claimed. The evidence-supported policy remains: across priorities, use a PI mutex rather than a bare binary semaphore as a lock.

#### Task C — `calib` capacity and the e-stop path

##### `calib`: CPU demand before and after sleeping

Before the source fix, 1000 calls to `k_busy_wait(400)` require

$$
C_{calib,before}=1000\times400\ \mu\text{s}
=400000\ \mu\text{s}=400\text{ ms of CPU}.
$$

For the measured periodic tasks above the console, using the conservative trace values already reported,

$$
U_{known}
=\frac{33.84}{1000}+\frac{82.78}{10000}
+\frac{47.39}{500000}+\frac{62.41}{1000000}
=0.04227519\approx4.23\%.
$$

Those tasks consume

$$
0.04227519\times1000\text{ ms}=42.27519\text{ ms}
$$

per second and leave

$$
1000-42.27519=957.72481\text{ ms}
$$

for lower-priority execution. Against this known subset, the original 400 ms busy-wait fits within the 1 s deadline with

$$
957.72481-400=557.72481\text{ ms}
$$

of analytical slack. This includes the documented $500\text{ ms}$ display and $1\text{ s}$ telemetry periods with their reported trace times, but it is not a verdict for Daniela's proposed load: the periods and execution times of her two tasks are stated as “given in class” and are absent from the supplied material. In addition, `flow_wq` is event-driven without a bounded arrival model, and the console's event-dependent execution demand is not represented as a periodic budget. The Liu & Layland test, hyperbolic test, and final fit decision for Daniela remain open for those reasons.

Under the proposed replacement of `k_busy_wait(400)` with `k_usleep(400)`, the console would no longer consume 400 ms of CPU; it would block during each settle. At the configured 100 µs kernel tick, 400 µs is four ticks, so the requested sleeps contribute a nominal minimum duration of

$$
1000\times400\ \mu\text{s}=400\text{ ms},
$$

plus wake-up and command overhead, while releasing the processor to higher-priority work. At a 1 kHz kernel tick, each 400 µs sleep rounds up to one 1 ms tick:

$$
1000\times1\text{ ms}=1000\text{ ms}.
$$

That consumes the entire 1 s deadline before wake-up, loop, and command overhead, so a 1 kHz tick cannot guarantee completion within 1 s. The post-fix CPU demand must be computed as the traced CPU cost of one settle multiplied by 1000, as required by the guide. That per-settle trace value is not present, and the reported $51.66\ \mu\text{s}$ for `task_console` is not identified as one settle, so it is not substituted as though it were one.

##### E-stop: end-to-end period sum

Before the source fix, the worst-case path waits for sampling to detect the condition, for the next control job to set `duty_pct = 0`, and for the following sampling/PWM step to write the valve pin:

$$
R_{estop,before}
\le T_{samp}+T_{ctrl}+T_{PWM}
=1\text{ ms}+10\text{ ms}+1\text{ ms}
=12\text{ ms}.
$$

This exceeds the Week 6 $5\text{ ms}$ target by $7\text{ ms}$, independently of the low measured CPU utilization. It also shows why utilization alone cannot prove the safety path: the failure is caused by release phasing across tasks, not processor saturation.

Under the proposed move of `duty_pct = 0` into the e-stop branch in `task_sampling`, the PWM write a few lines later would execute in the same sampling job. The inter-task waits would collapse to

$$
R_{estop,after}\le T_{samp}+C_{same\ job}.
$$

Using the reported sampling trace time as the only available execution-time indication,

$$
R_{estop,after}\approx1000\ \mu\text{s}+33.84\ \mu\text{s}
=1033.84\ \mu\text{s}\approx1.034\text{ ms}<5\text{ ms}.
$$

![Lab 6 Task C capture — appears post-fix](<Labs RTS/Evidencias lab 6 RTS/Task C/task c1.png>)

The arithmetic supports ADR-005's design: sleep rather than spin in `calib`, and close the valve in the detecting sampling job. The Task C screenshot is present but insufficient/unannotated: it appears to show a post-fix detection/valve transition, but it has no cursor measurement, no five-trip maximum, and no before capture. Therefore $12\text{ ms}$ and $1.034\text{ ms}$ remain analytical bounds/estimates from documented periods and available sampling timing, not substitutes for physical before/after evidence.


## 4. Schedulability analysis

### Current verdict

| Scope | Analysis | Verdict | Evidence status |
| --- | --- | --- | --- |
| Fig. 4.13 task set | $U=2/5+4/7=0.9714$; RM exact RTA gives $R_2=8\text{ ms}>D_2=7\text{ ms}$; implicit-deadline EDF has $U\le1$ | RM misses; ideal preemptive EDF is schedulable | Theory complete; local captures do not identify RM/EDF or prove the 60 s EDF run |
| Known periodic node subset | Sampling, control, display, and telemetry give $U=0.04227519$ | Known subset uses about 4.23% CPU by the reported trace values | Flow arrivals, console demand, and Daniela's tasks are not bounded, so this is not whole-node utilization |
| Control, no contention | $R_{ctrl}=82.78+\lceil R/1000\rceil33.84=116.62\ \mu\text{s}$ | Below the canonical $10\text{ ms}$ deadline | Analytical; no dedicated measured response supplied |
| Control with PI mutex | $R_{ctrl}=82.78+B+\lceil R/1000\rceil33.84$ | Requires measured $B$; boundary $B\le9578.82\ \mu\text{s}$ for $R\le10\text{ ms}$ | Captures present but do not measure $B$ or label all regimes |
| `calib` and e-stop | Pre-fix `calib` demand: $400\text{ ms}$ CPU; e-stop analytical path: $12\text{ ms}$ before, approximately $1.034\text{ ms}$ after the proposed source fix | Pre-fix e-stop violates REQ-CTRL-02; proposed same-job close is analytically compatible | Before/after maxima, one-settle CPU cost, and five-trip evidence remain missing |

Daniela's additional-load verdict remains open because her two task parameters and a bounded flow-arrival model are not available; console demand is also event-dependent. An RTA formula is documented above, but the required standalone script and output were not supplied and are not claimed. Week 12 CAN frame planning and end-to-end bus analysis are also not yet supported by material in this local unit.

## 5. Functional safety (final project)

The declared safe state is **valve closed**. REQ-CTRL-02 requires overpressure to close the valve in less than $5\text{ ms}$; the proposed same-sampling-job design has an analytical estimate below that deadline. A watchdog design, hardware fail-safe firing trace, and repeated physical e-stop maxima have not been supplied, so final-project functional-safety verification remains open.

## 6. Agent-built components (final project)

No Hub GUI, routing component, `spec/hub-soft.md`, agent implementation log, or component timing measurement is present in this local unit. AI assistance in this update was limited to evidence organization and consistency review, as disclosed in the report header; no measurement is attributed to AI. Agent-built component requirements, deviations, and measured compliance therefore remain pending rather than being invented.
