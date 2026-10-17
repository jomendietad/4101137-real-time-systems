---
marp: true
theme: course
paginate: true
title: Tracing
style: |
  section { --course: #0d9488; }
---

<!-- _class: lead -->
<!-- _paginate: false -->

# Tracing: seeing the scheduler

**Week 5 · demo** · the instrument labs 6 and 7 run on

The analyzer sees pins. The trace sees decisions.

<!--
0:00–1:00. Last week's Task C: a ≈ 4 µs gap between instr_samp falling and
instr_ctrl rising, and we *called* it a context switch. Today we watch it happen.
-->

---

<!-- _class: agenda -->

## Today

1. **Why**: what the analyzer can't see
2. **Pipeline**: from kernel hook to timeline
3. **Setup**: two jacks, three files
4. **Reading**: one preemption, event by event
5. **Cost**: the price of looking
6. **FreeRTOS**: same idea, other tools
7. **Use it**: from lab 6 on

Goal: read a preemption in a trace, and price the tracer.

<!--
1:00–1:30. About 20 minutes of slides, then the live demo on the node. The colored
tag at the top of each slide says which part we're in.
-->

---

<!-- _class: s1 -->
<!-- _header: 1 · Why -->

## Same moment, two instruments

![h:430](img/tr_two_views.svg)

<!--
1:30–3:30. Top: last week's capture. Ask: "What happened in those 4 µs?" They'll
say "a context switch". Bottom: sampling gives control's semaphore, control
becomes ready but can't preempt (priority 3 vs. 2), sampling blocks on tick_q,
and only then does the scheduler switch. On the analyzer, a switch, an ISR and a
slow instruction all look like the same flat line.
-->

---

<!-- _class: s1 -->
<!-- _header: 1 · Why -->

## What only the trace answers

| Question | Analyzer | Trace |
|---|---|---|
| How long did sampling run? | exact | close, but perturbed |
| What filled the 4 µs gap? | — | `k_sem_give`, pend, switch |
| Who ran during `calib`'s 400 ms? | only the pins you toggle | every thread, by name |
| Why did control start late? | — | who was ready, who had the CPU |
| Does measuring change the system? | no | **yes** |

<!--
3:30–5:00. The last row is the whole of part 5. Write the course rule on the
board and leave it there: numbers from the analyzer, story from the trace.
-->

---

<!-- _class: s2 -->
<!-- _header: 2 · Pipeline -->

## From kernel hook to timeline

![h:390](img/tr_pipeline.svg)

Every kernel decision already calls a hook (`sys_port_trace_*`). `CONFIG_TRACING` fills them in.

<!--
5:00–7:00. Without CONFIG_TRACING the hooks are empty macros: zero cost. In async
mode (the default) a hook only copies its event into RAM; the tracing thread, at
the lowest priority, drains the buffer when nothing else wants the CPU. A full
buffer drops the event and counts it: the hook never blocks. SystemView is the
same idea with a debug probe reading the buffer out of RAM.
-->

---

<!-- _class: s2 -->
<!-- _header: 2 · Pipeline -->

## CTF: a binary log with a schema

```text
thread_switched_in: 34 bytes on the wire
  timestamp   8 B   ns since boot
  id          2 B   0x11
  thread_id   4 B   the thread's address
  name       20 B   "sampling_tid"
```

```c
event {
	name = thread_switched_in;
	id = 0x11;
	fields := struct { uint32_t thread_id; ctf_bounded_string_t name[20]; };
};
```

The stream carries **no lengths**: the reader takes each event's size from the **metadata**. One id missing there and every byte after it decodes as garbage.

<!--
7:00–9:00. CTF (Common Trace Format) is also what LTTng writes on Linux. The
metadata (TSDL) is $ZEPHYR_BASE/subsys/tracing/ctf/tsdl/metadata. Hold on to the
last sentence: it bites us on the "Build, capture, view" slide.
-->

---

<!-- _class: s3 -->
<!-- _header: 3 · Setup -->

## Two jacks, two jobs

![h:330](img/tr_jacks.svg)

| Tempting route | Why not on our board |
|---|---|
| SystemView over RTT (J-Link) | Zephyr has no RTT on Espressif chips: Kconfig refuses it |
| CTF over `uart1` | the board routes its TX to GPIO17, our `instr_disp` |

<!--
9:00–10:30. The USB jack is the S3's built-in USB-Serial-JTAG: full-speed USB, no
adapter, no baud rate. Never trace over the console UART: the trace bytes would
mix with printk. Weeks 3–4 used only the UART jack; today both are plugged in.
-->

---

<!-- _class: s3 -->
<!-- _header: 3 · Setup -->

## Turn it on: two files, no C

```ini
# firmware/kernel/trace.conf
CONFIG_TRACING=y
CONFIG_TRACING_CTF=y
CONFIG_TRACING_BACKEND_UART=y
CONFIG_TRACING_HANDLE_HOST_CMD=y   # wait for the capture script's "enable"
CONFIG_TRACING_SYSCALL=n
CONFIG_TRACING_TIMER=n
```

```c
/* firmware/kernel/usb_trace.overlay */
/ { chosen { zephyr,tracing-uart = &usb_serial; }; };
&usb_serial { status = "okay"; };
```

<!--
10:30–11:30. HANDLE_HOST_CMD keeps tracing off until the host sends "enable", so
the capture starts on an event boundary: a capture joined mid-stream starts
mid-event and nothing after it decodes. The two =n lines drop events we don't
read (part 5 shows how little that saves). CONFIG_THREAD_NAME is already on from
week 4: it is what puts names on the lanes.
-->

---

<!-- _class: s3 -->
<!-- _header: 3 · Setup -->

## Build, capture, view

```bash
west build -p -b esp32s3_devkitc/esp32s3/procpu firmware/kernel -- \
  -DEXTRA_CONF_FILE=trace.conf -DEXTRA_DTC_OVERLAY_FILE=usb_trace.overlay
west flash && mkdir -p trace
python $ZEPHYR_BASE/scripts/tracing/trace_capture_uart.py \
  -d /dev/ttyACM0 -b 115200 -o trace/channel0_0     # Ctrl-C to stop
cp $ZEPHYR_BASE/subsys/tracing/ctf/tsdl/metadata trace/
grep -q sleep_ticks trace/metadata || cat firmware/kernel/ctf_sleep_events.tsdl >> trace/metadata
python $ZEPHYR_BASE/scripts/tracing/trace_viewer.py --metadata trace/metadata trace/channel0_0
```

> Zephyr main emits two `k_sleep` events that its own metadata doesn't describe (since Aug 2026). Without the `grep … || cat` line, the trace dies at the first `k_msleep`.

<!--
11:30–13:00. The USB jack ignores -b, but the script requires it. With the pot,
pass -DEXTRA_DTC_OVERLAY_FILE="pot_esp32s3.overlay;usb_trace.overlay".
trace_viewer.py exists on Zephyr main only (added June 2026); `babeltrace2 trace/`
gives a text dump on any version. The metadata gap is upstream commit a672a717cae.
-->

---

<!-- _class: s4 -->
<!-- _header: 4 · Reading -->

## One tick, one preemption

![h:480](img/tr_preemption.svg)

<!--
13:00–16:00. Narrate 1 → 8; the chain is copied from a real capture of this
firmware. Ask:
- "Where is lat_peak_us?" (1 → 5: tick to the thread's first instruction.)
- "Why does the console go to ready, not blocked?" (It was running; a preempted
  thread still wants the CPU.)
- "What changes on every 10th tick?" (Between 5 and 6 sampling gives control_sem;
  at 7 the switch goes to control, not to the console: last week's Task C gap.)
-->

---

<!-- _class: s4 question -->
<!-- _header: 4 · Reading -->

## At event 5, sampling runs. What is the console?

1. Blocked: it waits for `tick_q`
2. Ready: preempted, it still wants the CPU
3. Suspended by the tick ISR until the ISR returns
4. Running on the other core

<!--
16:00–17:30. Vote, one minute in pairs, vote again. Answer: 2. It was running
(spinning in calib) when the tick arrived; preemption moves it to ready. 1 confuses
it with sampling, which is the thread that blocks on tick_q. 3 confuses ISR entry
with a thread state: the ISR is long gone by event 5. 4: the node runs on PROCPU
only until week 9.
-->

---

<!-- _class: s5 -->
<!-- _header: 5 · Cost -->

## The price of looking: bandwidth

| | Per 1 ms tick | Per second |
|---|---|---|
| The node's trace, filtered | ≈ 20 events, ≈ 0.5 KB | ≈ 0.5 MB/s |
| Ring buffer, 2 KB | full in ≈ 4 ms if not drained | |
| A 115200 UART, for scale | | ≈ 11.5 KB/s |

Trace **seconds, not minutes**, and expect **drops**: a lane with a missing switch is a gap, not a bug.

<!--
17:30–19:00. Six events per tick carry a 20-byte thread name (switches, ready,
pend): ≈ 200 of the ≈ 500 bytes. Filtering SYSCALL and TIMER saved ≈ 13 %. A full
buffer loses whole events. Counted from a native_sim capture of this firmware,
median of 100 ticks: 26 events / 600 B unfiltered, 22 / 524 B filtered. The S3
will be close: the 1 kHz tick sets the event mix, not the chip. Whether the USB
jack keeps up is part of the demo.
-->

---

<!-- _class: s5 -->
<!-- _header: 5 · Cost -->

## The price of looking: time

- Each event is formatted with **interrupts locked**: a tick that fires meanwhile waits.
- ≈ 20 of them per tick, around the very latency you're measuring.
- The tracer drains at the **lowest priority**: trace CPU-load figures include it.

> Measure the cost with the instrument that has none: the **analyzer**, tracing on vs. off.

<!--
19:00–20:30. The lock is in CTF_EVENT → irq_lock(). Misconception to anticipate: "trace timestamps are ground truth".
They are the truth as perturbed by tracing. Trace widths and GPIO widths differ by
µs both ways: GPIO includes gpio_pin_set_dt, the trace includes its own formatting.
-->

---

<!-- _class: s5 question -->
<!-- _header: 5 · Cost -->

## The trace says sampling ran 52 µs; the analyzer says 48 µs. What goes in RET §1?

1. 52 µs: the trace sees every instruction
2. 48 µs: the analyzer doesn't perturb the node
3. 50 µs: average the two instruments
4. 52 µs: the larger one is the safe worst case

<!--
20:30–22:00. Vote, pairs, vote. Answer: 2, with "analyzer" in the instrument
column. 1 trusts the trace timestamps as ground truth: they include the tracer's
own formatting under irq_lock. 3 averages a perturbed number with a clean one. 4
sounds conservative but the 4 µs are the tracer's, not sampling's; the margin for
the worst case comes from the measurement protocol, not from a perturbed
instrument.
-->

---

<!-- _class: s6 -->
<!-- _header: 6 · FreeRTOS -->

## What about FreeRTOS?

- Same idea: trace macros (`traceTASK_SWITCHED_IN()`, …) stay empty until a recorder fills them.
- Recorders: **Percepio Tracealyzer** (commercial) or **SEGGER SystemView** (free for non-commercial use).
- On the S3 under ESP-IDF, app_trace streams SystemView over the same USB-JTAG.
- What Zephyr adds: an **open format** (CTF) and free viewers.

<!--
22:00–23:00. Both recorders usually read the target through a debug probe.
-->

---

<!-- _class: takeaway -->
<!-- _header: Verdict -->

## Verdict

> **The analyzer is the witness; the trace is the suspect's own story.** Take numbers from the analyzer and explanations from the trace, and never quote a trace timestamp without a capture beside it.

<!--
23:00–23:30.
-->

---

<!-- _class: s7 -->
<!-- _header: Demo -->

## Demo: one `calib`, traced

1. Traced build: console on the UART jack, capture on the USB jack.
2. Capture, type `calib`, wait 2 s, Ctrl-C.
3. Find the console busy-waiting; zoom on one tick, name events 1–8.
4. A 10th tick: `k_sem_give`, then the switch to control.
5. Count `switched_in` against `switched_out`: did the link drop events?

<!--
23:30–31:00. Dry-run on your own board first:
- /dev/ttyACM0 appears when the USB jack is plugged in.
- trace/channel0_0 grows while the capture runs.
- The viewer's lanes run past the first second (if they stop early, the metadata
  patch is missing).
If the USB jack can't keep up, events drop: show it, that's the bandwidth slide live.
-->

---

<!-- _class: s7 -->
<!-- _header: 7 · Use it -->

## From lab 6 on

| Lab | Use the trace to… |
|---|---|
| 6 · Fig. 4.13 | see τ2 still *ready* when its deadline passes under RM |
| 6 · Task D | see `calib` spin (busy-wait events), then sleep once fixed |
| 7 · inversion | see who holds the lock while control waits |

**Numbers from the analyzer, story from the trace.**

<!--
31:00–33:00. Add trace.conf and usb_trace.overlay to any app; the third file is
the metadata patch. All three live in firmware/kernel/; SOP-05 has the commands.
Then the bridge: the lab's table, predictions due before the boards come out.
-->
