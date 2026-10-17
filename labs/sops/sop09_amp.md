# SOP-09 — Two images on the two cores (sysbuild / AMP)

- **Serves:** [lab09_amp.md](../lab09_amp.md)
- **Hardware:** ESP32-S3 DevKitC + logic analyzer.

Steps, checks, and troubleshooting only — the *why* lives in the lab.

> **Draft** — commands pending validation on course hardware.

## 1. The reference sample first

Prove the AMP plumbing before touching your code:

```bash
west build -p -b esp32s3_devkitc/esp32s3/procpu \
    zephyr/samples/drivers/ipm/ipm_esp32 --sysbuild
west flash && west espressif monitor
```

**Verify:** the monitor shows PROCPU booting and messages relayed from the APPCPU
image — both cores alive from one `west flash`.

## 2. Project layout for the lab

Two apps, one sysbuild:

```
amp_node/
├── app_procpu/          # rest of the node + synthetic load
│   ├── CMakeLists.txt
│   ├── prj.conf         # CONFIG_IPM=y: PROCPU starts the APPCPU only with IPM on
│   ├── app.overlay      # &ipm0 { status = "okay"; };
│   └── sysbuild.cmake
└── app_appcpu/          # hard loop + its instrumentation GPIO, nothing else
    ├── CMakeLists.txt
    └── prj.conf
```

`app_procpu/sysbuild.cmake` registers the second image and builds it first, as the
`ipm_esp32` sample does (PROCPU's image needs the APPCPU image's load address):

```cmake
ExternalZephyrProject_Add(
    APPLICATION app_appcpu
    SOURCE_DIR ${APP_DIR}/../app_appcpu
    BOARD esp32s3_devkitc/esp32s3/appcpu
)
add_dependencies(app_procpu app_appcpu)
sysbuild_add_dependencies(CONFIGURE app_procpu app_appcpu)
```

The first argument of both lines is the PROCPU app's `project()` name.

## 3. Build, flash, run

```bash
west build -p -b esp32s3_devkitc/esp32s3/procpu app_procpu --sysbuild
west flash
```

**Verify:** PROCPU console comes up; the APPCPU loop's GPIO toggles on the analyzer.
That toggle is the only APPCPU health signal you get.

## 4. Debugging the APPCPU

There is **no Zephyr console on the APPCPU**. If you must print during bring-up:

```c
extern int ets_printf(const char *fmt, ...);
ets_printf("appcpu alive: %d\n", counter);   /* ROM UART, bring-up only */
```

Remove it before measuring — it perturbs the loop. Measurement is by GPIO, as always.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| APPCPU GPIO never toggles | Second core not started | PROCPU starts it only with IPM enabled: `CONFIG_IPM=y` and `&ipm0 { status = "okay"; };` in the **procpu** image; confirm sysbuild built both (`build/app_appcpu/` exists) |
| `west flash` writes only one image | Built without `--sysbuild` | Rebuild with `--sysbuild`; flash from the sysbuild build dir |
| Build fails resolving the appcpu image | Path/board typo in `sysbuild.cmake` | `SOURCE_DIR` relative to `${APP_DIR}`; board is `.../esp32s3/appcpu` |
| Both cores fight over a pin | Same GPIO claimed in both DTs | Give each image its own overlay; keep pin maps disjoint |
| APPCPU crashes silently | Shared-memory/IPM misconfig, or heavy driver on APPCPU | Keep the APPCPU image minimal (loop + GPIO only); anything fancy stays on PROCPU |
| `ets_printf` output interleaves garbage | Both cores writing the ROM UART | Print from one core at a time; delete the prints before measuring |
