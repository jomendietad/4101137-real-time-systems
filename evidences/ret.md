# RET — Timing Evidence Report

* **Team:** `Johan Sebastian Mendieta Dilbert, Jeronimo Novoa Giraldo`
* **Boards:** Nucleo-L476RG (STM32L476RG) / ESP32-S3 DevKitC (esp32s3_devkitc_esp32s3_procpu)

**Living document** — updated every week; handed in at the workshop (week 8) and
at the close (week 16).

**House rule:** *"show me the trace"* — every timing claim cites a measurement.

## 1. The system and its task set

Requirements first — one sentence each, EARS style (*when/while `<condition>`, the
system shall `<response>` within `<deadline>*`) — then the task that implements them:

| ID | Requirement |
| --- | --- |
| REQ-CTRL-01 | While the system is powered on, the superloop shall execute `task_sampling()` with a nominal sampling period of $1000\ \mu\text{s}$ and a maximum jitter of $\le 50\ \mu\text{s}$. |
| REQ-CTRL-02 | While irrigation control is active, the control loop (`task_control()`) shall run every $10\text{ ms}$ with a maximum execution deadline of $1000\ \mu\text{s}$. |
| REQ-FLOW-03 | When flow meter pulse interrupts occur, `task_flow_batch()` shall process accumulated edge events within $500\ \mu\text{s}$ of the main loop iteration. |
| REQ-CONS-04 | When CLI input is received over UART, `task_console()` shall parse the command stream without blocking the main loop for more than $200\ \mu\text{s}$. |
| REQ-SAFE-05 | When measured pressure exceeds the threshold, the system shall assert the valve cutoff signal within $10\text{ ms}$ ($1\text{ control tick}$). |
| REQ-PERF-06 | While the system operates under nominal workload, the loop tick backlog (`backlog_peak`) shall not exceed $2\text{ ticks}$ ($2000\ \mu\text{s}$). |

| Task | Req. | Type (H/F/S) | Period | Deadline | Measured $C_i$ | How it was measured |
| --- | --- | --- | --- | --- | --- | --- |
| `task_sampling` | REQ-CTRL-01 | Hard | $1\text{ ms}$ | $1\text{ ms}$ | $0.75\ \mu\text{s}$ `[GPIO]` / $33.84\ \mu\text{s}$ `[Trace]` | GPIO D3 + Logic Analyzer (S3 @ 100 MHz) / CTF Trace |
| `task_control` | REQ-CTRL-02 | Hard | $10\text{ ms}$ | $1\text{ ms}$ | $1.35\ \mu\text{s}$ `[GPIO]` / $82.78\ \mu\text{s}$ `[Trace]` | GPIO D4 + Logic Analyzer / CTF Trace |
| `task_flow_batch` | REQ-FLOW-03 | Firm | Event | $500\ \mu\text{s}$ | $12.81\ \mu\text{s}$ `[GPIO]` | GPIO D5 + Logic Analyzer |
| `task_console` | REQ-CONS-04 | Soft | Polled | $200\ \mu\text{s}$ | $5794.55\ \mu\text{s}$ `[GPIO]` / $51.66\ \mu\text{s}$ `[Trace]` | GPIO D6 + Logic Analyzer / CTF Trace |
| `task_display` | N/A | Soft | Polled | N/A | $47.39\ \mu\text{s}$ `[Trace]` | GPIO D7 + Logic Analyzer / CTF Trace |
| `task_telemetry` | N/A | Soft | Polled | N/A | $62.41\ \mu\text{s}$ `[Trace]` | GPIO D8 + Logic Analyzer / CTF Trace |

*Note: Instrument annotation bracket `[GPIO]` indicates physical pin toggling measurement; `[Trace]` indicates software kernel instrumentation hook measurement via CTF.*

## 2. ADRs

### ADR-001 — Node Architecture: Multithreaded Kernel

* **Context:** In Weeks 2 and 3, the cooperative superloop architecture suffered from severe temporal interference. Synchronous blocking operations such as `calib` (1000 iterations of 400 µs busy-waits in `task_console`) stalled execution in `main()`, causing the control period to stretch from 10 ms to 102 ms (and up to >1000 ms), while sampling jitter exceeded 14000 µs.
* **Decision:** Migrate all application tasks to a fully preemptive multi-threaded model on the Zephyr RTOS kernel, assigning Rate-Monotonic priorities (`sampling_thread`=2, `control_thread`=3, `flow_wq`=5, `display_thread`=7, `telemetry_thread`=8, `console_thread`=9) and deploying a dedicated workqueue for flow pulse processing.
* **Justification (with numbers):** The A/B comparison confirms complete temporal isolation under load. Under `calib`, sampling jitter dropped from $14200.0\ \mu\text{s}$ (superloop) to **$3.2\ \mu\text{s}$** (kernel RTOS), and the maximum control period was restored from $102.0\text{ ms}$ to **$10.02\text{ ms}$**. The measured context-switch overhead between sampling and control is **$2.4\ \mu\text{s}$**, representing an overall CPU capacity usage penalty of **$0.312\%$**. Max stack utilization across user threads remained at 50% (`console_tid` at 1024 / 2048 bytes).
* **Status:** Accepted.

## 3. Evidence by week

Each entry cites the `REQ`(s) it verifies.

### Week 1 — Bring-up: the team, the board, the toolchain

#### Iteration Loop Measurement Matrix

| Quantity | Prediction | Your value | Target |
| --- | --- | --- | --- |
| Edit $\rightarrow$ build $\rightarrow$ flash $\rightarrow$ message on the console, one member's laptop | $35.0\text{ s}$ | **$27.4\text{ s}$** | $< 60\text{ s}$ |
| Same, incremental build (one line changed, no `-p`) | $10.0\text{ s}$ | **$6.8\text{ s}$** | — |
| Members with `blinky` running from their own laptop | — | **$2 / 2$** | all |

#### Toolchain & Board Bring-Up Verification (Tasks A, B & C)

1. **Cross-Check on `nucleo_l476rg` & `native_sim` (Task A):**
* Both team members (`Johan Sebastian Mendieta Dilbert` and `Jeronimo Novoa Giraldo`) independently verified the Zephyr workspace (`~/rts`) and toolchain by building and executing `zephyr/samples/basic/blinky` first on the host simulator (`west build -p -b native_sim zephyr/samples/basic/blinky`) and subsequently on hardware (`west build -p -b nucleo_l476rg zephyr/samples/basic/blinky && west flash`).
* **Hardware Evidence:** On-board user LED (`LD2`, `PA5`) toggled at $1\text{ Hz}$ ($500\text{ ms}$ ON / $500\text{ ms}$ OFF) on the Nucleo-L476RG after flashing from each member's laptop.


2. **Serial Console & Build Cycle Breakdown (Task B):**
* Connected to the ST-Link Virtual COM port at `115200 8N1` (`minicom` / `tio`) after flashing `zephyr/samples/hello_world`.
* Edited the print string in `src/main.c` and measured two complete edit-to-console cycles:
* **Pristine cycle (`west build -p` + `west flash`):** **$27.4\text{ s}$** total ($23.1\text{ s}$ CMake/Devicetree/kernel full compile + $4.3\text{ s}$ OpenOCD flash and target reset), satisfying the $< 60\text{ s}$ stakeholder requirement.
* **Incremental cycle (`west build` without `-p` + `west flash`):** **$6.8\text{ s}$** total ($2.5\text{ s}$ Ninja incremental recompile and link + $4.3\text{ s}$ flash and target reset), achieving a **$4.03\times$** reduction in iteration latency.





```text
*** Booting Zephyr OS build v4.1.0 ***
Hello World! SoilSense Control Team (Johan & Jeronimo) - nucleo_l476rg

```

3. **Team RET Initialization & Initial Task Set (Task C):**
* Initialized the team repository under Git version control with `ret.md` (`Header + §1`), defining the initial SoilSense Control task set (`task_sampling` at $1\text{ ms}$, `task_control` at $10\text{ ms}$, event-driven `task_flow_batch`, and polled soft tasks `task_console`, `task_display`, and `task_telemetry` mapped to **REQ-CTRL-01** through **REQ-PERF-06**) with unmeasured execution times initialized to `____` ahead of Week 2 hardware instrumentation.

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
│ [1] INICIO: main() -> Entrada al bucle mientras(1)            │
└──────────────────────────────┬────────────────────────────────┘
│
▼
┌───────────────────────────────────────────────────────────────┐
│ [2] task_console()        (Conmutación GPIO D3 HIGH -> LOW)   │
└──────────────────────────────┬────────────────────────────────┘
│
▼
┌───────────────────────────────────────────────────────────────┐
│ [3] task_display()        (Conmutación GPIO D4 HIGH -> LOW)   │
└──────────────────────────────┬────────────────────────────────┘
│
▼
┌───────────────────────────────────────────────────────────────┐
│ [4] task_telemetry()      (Conmutación GPIO D5 HIGH -> LOW)   │
└──────────────────────────────┬────────────────────────────────┘
│
▼
┌───────────────────────────────────────────────────────────────┐
│ [5] DECISIÓN: ¿ atomic_get(&ticks_pending) > 0 ?              │
└──────────────┬─────────────────────────────────┬──────────────┘
│ SÍ                                             │ NO
▼                                                │
┌──────────────────────────────┐                 │
│ atomic_dec(&ticks_pending)   │                 │
└──────────────┬───────────────┘                 │
│                                                │
▼                                                │
┌──────────────────────────────┐                 │
│ [6] task_sampling()          │                 │
│      (GPIO D6 HIGH -> LOW)   │                 │
└──────────────┬───────────────┘                 │
│                                                │
▼                                                │
┌──────────────────────────────┐                 │
│ [7] ¿ ++control_div >= 10 ?  │                 │
└──────┬───────────────┬───────┘                 │
│ SÍ               │ NO                          │
▼                  │                             │
┌────────────────────┐ │                         │
│ [8] task_control() │ │                         │
│   (GPIO D7 H -> L) │ │                         │
└──────┬─────────────┘ │                         │
│                │                               │
└───────┬────────┘                               │
│                                                │
└───────────┬─────────────────────┘
│
▼
┌───────────────────────────────────────────────────────────────┐
│ [9] task_flow_batch()     (Conmutación GPIO D8 HIGH -> LOW)   │
└──────────────────────────┬────────────────────────────────────┘
│
▼
┌───────────────────────────────────────────────────────────────┐
│ [10] FIN DE ITERACIÓN -> Retorno a [1]                         │
└───────────────────────────────────────────────────────────────┘


```

#### Measurements Table

| Measurement | Your value | Reference / Verified REQ |
| --- | --- | --- |
| Actual sampling period (nominal 1 kHz): average | **997.89 µs** | $1000\ \mu\text{s}$ (Verifica **REQ-CTRL-01**) |
| Sampling jitter: max over $\ge 30\text{ s}$ | **5581.46 µs** | Course baseline |
| ISR $\rightarrow$ loop-service latency (`task_flow_batch`) | **12.81 µs** (Prom) / **29.42 µs** (Máx) | Verifica **REQ-FLOW-03** |
| Sampling jitter, max, **with the flash cache off** | **6798.46 µs** | Gap vs row 2 is the ART accelerator |
| Sampling jitter **with the blocking command active** | **6663.79 µs** | Violación de **REQ-CONS-04** y **REQ-CTRL-01** |
| `backlog_peak`, idle $\rightarrow$ during the blocking command | **1 $\rightarrow$ 6 ticks** | Verifica **REQ-PERF-06** |

#### Root-Cause Analysis & Observations

* **Impact of Disabling Flash Cache (`nocache.conf`):** Disabling the ART Accelerator and instruction prefetch buffer forces the STM32L476RG CPU to introduce wait states on every Flash fetch. This increases task execution times $C_i$ (e.g., `task_sampling` increases from $4.94\ \mu\text{s}$ to $7.26\ \mu\text{s}$), widening the loop pass duration and increasing worst-case sampling jitter.
* **Degradation under Blocking Command (`calib`):** In a non-preemptive superloop, tasks execute cooperatively to completion. The calibration routine inside `task_console()` (`firmware/superloop/src/console.c`) performs a synchronous blocking delay. While trapped in this function, the CPU cannot return to `main()` to evaluate `ticks_pending` or execute `task_sampling()`, causing $1\text{ kHz}$ timer ticks to backlog and severely degrading period determinism.

#### Baseline Reading

> *El superloop cooperativo garantiza un tiempo de ejecución eficiente para tareas de muestreo periódicas en estado de reposo, pero presenta una severa falta de aislamiento temporal que degrada el determinismo del sistema ante cualquier operación de entrada/salida o comando bloqueante.*

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

4. **Shared State Access and Atomic Memory Operations:**
State variables `pressure_mv`, `estop`, and `setpoint_mv` are accessed across `sampling_thread` and `main`. On 32-bit architectures like the Xtensa LX7 (ESP32-S3), naturally aligned 32-bit scalar memory reads and writes are single-cycle atomic operations at the hardware memory bus level, preventing torn reads or writes across threads without requiring explicit mutex locks.

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
* **Overhead Calculation:** At nominal operational workload (1000 sampling thread activations/s + 100 control thread activations/s + ~200 background thread switches/s ≈ 1300 context switches/s):

$$\text{CPU Overhead} = 1300\text{ switches/s} \times 2.4\ \mu\text{s/switch} = 3120\ \mu\text{s/s} \approx 0.312\%$$

The scheduling context-switch overhead accounts for less than **0.32%** of total CPU capacity.

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

#### Measurement Comparison Matrix (Trace vs. GPIO)

| Measurement | Prediction | Your value (Trace / Analyzer) | Contrast / Reference |
| --- | --- | --- | --- |
| Context switch per the trace | $15.0\ \mu\text{s}$ | **$30.69\ \mu\text{s} - 32.52\ \mu\text{s}$**

 | vs. GPIO (wk 4): **$2.4\ \mu\text{s}$** |
| Tick ISR → sampling start per the trace | $10.0\ \mu\text{s}$ | **$31.50\ \mu\text{s} - 32.52\ \mu\text{s}$**

 | vs. `lat_peak_us`: **$6.0\ \mu\text{s}$** |
| Per-thread CPU load (%) in normal operation | 100% total | **`sampling_tid`: 0.81%** <br>

<br> **`control_tid`: 0.20%** <br>

<br> **`console_tid`: 0.30%** <br>

<br> **`display_tid`: <0.05%** | sums $\approx 100\%$ with `idle` (**98.64%**) and tracer |
| Tracing overhead on max sampling jitter | $2.0\ \mu\text{s}$ | **$2.10\ \mu\text{s}$** ($\text{ON: } 3.20\ \mu\text{s} \text{ vs. OFF: } 1.10\ \mu\text{s}$) | analyzer, tracing ON vs. OFF |

#### The Story of One Preemption (Task B)

1. **Preemption Execution Sequence:**
* **Tick ISR:** Hardware clock tick triggers `ISR: Tick / Hardware` at $t = 8024.90\text{ ms}$.


* **Queue Posting (`k_msgq_put`):** The ISR executes `k_msgq_put`, placing a message into the queue and unblocking `sampling_tid`.


* **Console Preempted:** `console_tid` (running at priority 9) is immediately preempted by the RTOS scheduler and transitions from `RUNNING` to `READY` state.


* **Sampling Execution:** `sampling_tid` (priority 2) enters `RUNNING` state and executes for **$34.02\ \mu\text{s} - 34.17\ \mu\text{s}$**.


* **Queue Pending (`k_msgq_get`):** `sampling_tid` finishes processing and calls `k_msgq_get`, blocking itself on an empty queue (`PENDING` state).


* **CPU Return:** Control yields back to `console_tid`, resuming CLI parsing.




2. **Analysis of `calib` Trace Duration Limit (~75 ms):**
* Typing `calib` triggers a massive stream of console output logs and high-frequency UART activity. The CTF trace subsystem uses an in-memory ring buffer to log events before UART transmission at 4 Mbaud. The volume of calibration events quickly exceeds the UART output rate, causing a **trace buffer overflow**. As a result, only the first $\sim 75\text{ ms}$ of trace events are preserved before dropped-event flags are set.


3. **Behavior at the 10th Tick ($t \approx 41.66\text{ ms}$):**
* At the 10th system tick ($10 \times 4.166\text{ ms} = 41.66\text{ ms}$), the timer calls `k_sem_give` to release the control loop semaphore. Instead of returning directly to `console_tid`, the scheduler transfers CPU execution to **`control_tid`** (running at $24\text{ Hz}$), which executes its control algorithm for **$81.37\ \mu\text{s} - 83.88\ \mu\text{s}$** before returning to lower-priority tasks.




4. **Why Trace vs. GPIO Measurements Differ (Analysis):**
* **GPIO Analyzer:** Measures physical pin level changes. It captures pure execution boundaries without software overhead, yielding context-switch gaps of **$2.4\ \mu\text{s}$** and interrupt latency of **$6.0\ \mu\text{s}$**.
* **Software Tracing (CTF):** Captures event logging hooks inside kernel functions. The trace values (**$30.69\ \mu\text{s} - 32.52\ \mu\text{s}$** for context switches and **$31.50\ \mu\text{s} - 32.52\ \mu\text{s}$** for ISR-to-thread latency) include the software overhead of serializing CTF event records, updating timestamp registers, and executing tracing macros inside kernel schedulers.


#### Task C — The price of looking (Tracing Overhead)

- **Measurement Protocol:** Sampling jitter was measured via GPIO (`instr_samp`) using the Analog Discovery 2 logic analyzer under identical workload conditions (`calib` active) with CTF tracing enabled (`trace.conf`) and disabled baseline.

| Condition | Max Sampling Jitter (AD2 / GPIO) | Observation / Cause |
| --- | --- | --- |
| **Tracing OFF** | **$1.10\ \mu\text{s}$** | Pure RTOS scheduling & tick ISR latency baseline |
| **Tracing ON** | **$3.20\ \mu\text{s}$** | Software perturbation introduced by CTF event logging hooks |
| **Delta ($\Delta$ Overhead)** | **$+2.10\ \mu\text{s}$** | Net timing penalty imposed by the tracing subsystem |

1. **Quantification of Overhead:**
   Enabling CTF tracing increases worst-case sampling jitter by **$2.10\ \mu\text{s}$** (from $1.10\ \mu\text{s}$ to $3.20\ \mu\text{s}$). While the sampling thread continues to meet its hard real-time requirement ($\text{jitter} \le 50\ \mu\text{s}$, verifying **REQ-CTRL-01**), the software instrumentation introduces measurable temporal perturbation.

2. **Root-Cause Analysis (Why the Observer Perturbs):**
   - **Trace Hook Execution:** Every thread context switch, queue operation, and timer ISR executes inline CTF logging macros that write timestamped event records to memory.
   - **Bus & Memory Contention:** High-frequency binary trace buffer writes compete for internal bus cycles on the ESP32-S3, introducing micro-stalls during instruction fetch and memory access.
   - **UART FIFO Pressures:** High-rate serial streaming over `uart0` consumes additional CPU cycles for UART TX buffer management.

3. **Synthesis (GPIO vs. Tracing):**
   - **GPIO + Logic Analyzer ("The Judge of Numbers"):** Non-intrusive physical measurement. It provides exact, unperturbed timing numbers ($1.10\ \mu\text{s}$ jitter, $2.4\ \mu\text{s}$ context switch) without altering firmware behavior.
   - **CTF Tracing ("The Judge of the Story"):** Intrusive software measurement. While it adds a small timing overhead ($+2.10\ \mu\text{s}$), it reveals thread names, exact preemption instances, state transitions, and root causes that an external pin analyzer cannot see inside the kernel.


### Week 6 — Schedulability, response time, and priority inversion

This section completes the analytical part of §4 with the data available in this report and in the Week 6 guide. Where the guide requires analyzer captures or parameters supplied separately in class, the corresponding numerical result remains explicitly open rather than being inferred from unavailable evidence.

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

Therefore the task set is schedulable under ideal preemptive EDF, despite failing under RM. In the Zephyr experiment, each absolute deadline must be assigned immediately after wake-up and before the thread can be selected using a stale deadline. The guide requires analyzer evidence of the RM miss and a 60 s EDF run with no misses; neither capture nor a measured miss counter is present in the supplied report, so the theoretical verdict is established but the board-level evidence remains pending.

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

and the next iteration is unchanged. Therefore the known periodic set bounds the no-contention control response at $116.62\ \mu\text{s}$, below its $1000\ \mu\text{s}$ deadline.

Replacing the semaphore with `k_mutex` activates Zephyr priority inheritance: while control is blocked, the console lock owner temporarily inherits control's priority. Medium-priority work can no longer extend the blocking interval, so control is blocked by at most one low-priority critical section of measured width $B$:

$$
R_{ctrl}^{(k+1)} = 82.78 + B
+ \left\lceil\frac{R_{ctrl}^{(k)}}{1000}\right\rceil 33.84
\quad [\mu\text{s}].
$$

If the converged response remains below one sampling period, this reduces to

$$
R_{ctrl}=116.62+B\quad [\mu\text{s}],
$$

and meeting the $1000\ \mu\text{s}$ control deadline requires $B\le883.38\ \mu\text{s}$ in that interval. The required GPIO width for the low thread's critical section, the three annotated captures, and measured control responses for the semaphore and mutex cases are not present in the supplied material. A numerical inversion delay, a numerical mutex response, and an empirical ADR-002 therefore cannot be claimed yet. The evidence-supported policy is nevertheless clear: shared resources used across priorities require a PI mutex, never a bare binary semaphore.

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
=0.03384+0.008278
=0.042118\approx4.21\%.
$$

Those tasks consume

$$
0.042118\times1000\text{ ms}=42.118\text{ ms}
$$

per second and leave

$$
1000-42.118=957.882\text{ ms}
$$

for lower-priority execution. Against this known subset, the original 400 ms busy-wait fits within the 1 s deadline with

$$
957.882-400=557.882\text{ ms}
$$

of analytical slack. This is not a verdict for Daniela's proposed load: the periods and execution times of her two tasks are stated as “given in class” but are absent from both supplied files. In addition, `flow_wq` is event-driven and the display and telemetry periods are not documented, so their demand cannot be added without inventing parameters. The Liu & Layland test, hyperbolic test, and final fit decision for Daniela remain open for that reason.

After replacing `k_busy_wait(400)` with `k_usleep(400)`, the console no longer consumes 400 ms of CPU; it blocks during each settle. At the configured 100 µs kernel tick, 400 µs is four ticks, so the requested sleeps contribute a nominal minimum duration of

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

After moving `duty_pct = 0` into the e-stop branch in `task_sampling`, the PWM write a few lines later executes in the same sampling job. The inter-task waits collapse to

$$
R_{estop,after}\le T_{samp}+C_{same\ job}.
$$

Using the reported sampling trace time as the only available execution-time indication,

$$
R_{estop,after}\approx1000\ \mu\text{s}+33.84\ \mu\text{s}
=1033.84\ \mu\text{s}\approx1.034\text{ ms}<5\text{ ms}.
$$

The arithmetic supports ADR-005's source-level design: sleep rather than spin in `calib`, and close the valve in the detecting sampling job. However, the report does not contain the required five-trip before/after maxima, valve GPIO captures, or an e-stop-specific WCET measurement. Therefore $12\text{ ms}$ and $1.034\text{ ms}$ are analytical bounds/estimates from the documented periods and available sampling timing, not substitutes for the missing physical evidence.


## 4. Schedulability analysis

U = ΣC_i/T_i with **measured** C_i; test used (RM / hyperbolic / EDF); RTA as a
script with its output; blocking B_i if there are mutexes. (Formulas: READINGS.md,
"The math that does get used".)

## 5. Functional safety (final project)

Declared safe state (failure ⇒ valve closed), watchdog, and the evidence of the
fail-safe firing.
