# SOP-05 — CTF tracing on the node, over the USB jack

- **Serves:** [lab05_tracing.md](../lab05_tracing.md)
- **Hardware:** ESP32-S3 DevKitC with **both** jacks plugged in: console on the UART
  jack, trace on the USB jack (the S3's built-in USB-Serial-JTAG). No probe, no adapter.
- **Firmware:** [firmware/kernel/](../../firmware/kernel/) or your week-4 node, plus
  its three tracing files: `trace.conf`, `usb_trace.overlay`, `ctf_sleep_events.tsdl`.

Steps, checks and troubleshooting only; the *why* is in the
[week-5 deck](../presentations/05_tracing.md).

## 1. Build and flash the traced node

Copy the three tracing files into your app if it isn't `firmware/kernel`, then:

```bash
west build -p -b esp32s3_devkitc/esp32s3/procpu firmware/kernel -- \
  -DEXTRA_CONF_FILE=trace.conf -DEXTRA_DTC_OVERLAY_FILE=usb_trace.overlay
west flash
```

With the pot: `-DEXTRA_DTC_OVERLAY_FILE="pot_esp32s3.overlay;usb_trace.overlay"`.

**Verify:** `ls /dev/ttyACM*` lists the USB jack (usually `/dev/ttyACM0`), and the
console on the UART jack still answers `status`.

## 2. Capture

```bash
mkdir -p trace
python $ZEPHYR_BASE/scripts/tracing/trace_capture_uart.py \
  -d /dev/ttyACM0 -b 115200 -o trace/channel0_0
```

The script sends `enable`, so the stream starts on an event boundary. Do what you
want to trace (e.g. type `calib` on the console), wait 1–2 s, then Ctrl-C. The USB
jack ignores the baud rate; the script still requires `-b`.

**Verify:** `ls -l trace/channel0_0` grows while the capture runs.

## 3. Metadata

```bash
cp $ZEPHYR_BASE/subsys/tracing/ctf/tsdl/metadata trace/
grep -q sleep_ticks trace/metadata || cat firmware/kernel/ctf_sleep_events.tsdl >> trace/metadata
```

The second line adds two `k_sleep` events the pinned Zephyr emits but doesn't
describe; without them decoding stops at the first `k_msleep`.

**Verify:** `grep -c sleep_ticks trace/metadata` prints a number above 0.

## 4. View

```bash
python $ZEPHYR_BASE/scripts/tracing/trace_viewer.py --metadata trace/metadata trace/channel0_0
python $ZEPHYR_BASE/scripts/tracing/trace_viewer.py --metadata trace/metadata --text trace/channel0_0 > trace.txt
```

For a graphical timeline, open the `trace/` directory in Eclipse TraceCompass
(File → Open Trace). `babeltrace2 trace/` gives a text dump on any Zephyr version.

**Verify:** the lanes carry your thread names (`sampling_tid`, `control_tid`, …) and
run past the first second; `trace.txt` shows `thread_switched_in` /
`thread_switched_out` events with increasing timestamps.

## 5. Tracing off, for the overhead row

Rebuild without `trace.conf` and `usb_trace.overlay` (the plain week-4 build) and
measure sampling jitter on the analyzer with the usual protocol.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| No `/dev/ttyACM*` | USB jack not plugged, or charge-only cable | Plug the jack marked USB with a data cable |
| `Permission denied` on `/dev/ttyACM0` | User not in `dialout` | `sudo usermod -aG dialout $USER`, log out and in |
| Lanes stop after a fraction of a second | Metadata without the sleep events | Step 3, second line |
| Everything after some point decodes as garbage | Capture joined mid-event, or an event id missing from the metadata | Restart the capture (the script re-sends `enable`); re-copy the metadata from the pinned Zephyr |
| A switch in without its switch out | Ring buffer full: whole events dropped | Expected at 1 kHz: trace seconds, not minutes; treat the gap as a gap |
| Threads shown as addresses | `CONFIG_THREAD_NAME` off | It is on in `firmware/kernel/prj.conf`; check your own `prj.conf` |
| Trace bytes in the console | Tracing pointed at the console UART | Keep `usb_trace.overlay`: tracing goes to `usb_serial`, never `uart0` |
