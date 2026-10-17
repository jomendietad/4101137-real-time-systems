# Week 9 — The two cores (AMP)
> - **Tech guide:** [SOP-09](sops/sop09_amp.md)
> - **Reading:** [READINGS.md](../READINGS.md), week 9
> - **Module:** 4
> - **Due today:** Problem Set 2.

**From:** Eng. Samuel Cifuentes — *"The S3 has two cores and we're using one. I
want to know whether the second core can buy the control loop a better **worst
case**. Today's talk also owes you an honest answer: when FreeRTOS is the right
call — because some customer will demand it."*

| Stakeholder | Their question | How this session answers it |
|---|---|---|
| **Samuel** | Does the second core improve the tail of the distribution? | Max jitter with and without a dedicated core |
| **Gustavo** | Is the second core worth two images to maintain? | The tail numbers next to the cost, in ADR-003 |

## What you'll measure

Hand in the prediction column before the lab.

| Measurement | Prediction | Your value | Note |
|---|---|---|---|
| Max loop jitter, 1 core, no load | | ____ µs | reference (wk 4) |
| Max loop jitter, 1 core, **with synthetic load** | ____ µs | ____ µs | the load: a cache-thrashing thread |
| Max loop jitter on APPCPU, load on PROCPU | ____ µs | ____ µs | partitioned isolation |
| Averages of all three cases | ____ | ____ | |

## Tasks

### Task A — AMP with sysbuild
- Two images (SOP-09): the **hard loop** (with its instrumentation GPIO) for
  `esp32s3_devkitc/esp32s3/appcpu`; the rest of the node + a **synthetic load**
  for `.../procpu`. No console on the APPCPU (a real limitation — `ets_printf` if
  you must debug): measurement is by GPIO, as always.
- **Evidence:** both images running at once, loop toggle visible.

### Task B — The tail of the distribution
- Usual protocol, ≥ 60 s per case, fill in the table. Compare **maxima**, not
  averages: the talk's thesis is that partitioning doesn't improve the typical
  case — it cuts off the rare spikes.
- **Evidence:** table + the three captures.

### Stretch (optional, not graded) — The flow sensor as a driver
- Turn the YF-S401 reading into a minimal Zephyr driver: its own devicetree
  binding (`vendor,yf-s401`), the sensor API, and the app reaching it through
  `DEVICE_DT_GET`, with no hard-coded pin left in the app.
- **Evidence:** the overlay declaring the sensor, and the app with no flow pin in it.

## What about FreeRTOS?

Here FreeRTOS wins on this board: ESP-IDF ships real SMP on the S3 (a global
scheduler across both cores, with optional per-task pinning), while Zephyr runs the
S3's two cores only as AMP (its SMP support covers the original ESP32, not the S3).
The honest criterion: if your product is "this S3, radio on, both cores squeezed",
ESP-IDF/FreeRTOS is the mature option; if it's "a product family across varied
silicon", Zephyr.

## Deliverables (RET)

- **§3 Week-9 evidence:** the three-case table + a one-sentence reading.
- **§2 ADR-003 — Does the hard loop live on its own core?** Decision backed by the
  tail numbers; include the cost (two images, no console on APPCPU).

## Rubric (100 pts)

| | pts |
|---|---|
| **Execution** — AMP running, both images from one flash (25) · synthetic load on PROCPU, verified on the analyzer (15) | 40 |
| **Evidence** — three-case table with a constant protocol (20) · captures (10) | 30 |
| **Analysis** — maxima vs. averages well read (15) · ADR-003 with the cost acknowledged (15) | 30 |
