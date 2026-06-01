# fbb_ws63 — QEMU fork

This is a **QEMU-oriented fork** of the HiSilicon **fbb_ws63** WS63 C SDK
(upstream: <https://gitcode.com/HiSpark/fbb_ws63>), tracking the small changes
needed to build firmware that boots on the **[ws63-qemu](https://github.com/sanchuanhehe/ws63-qemu)**
emulator (no real WS63 hardware required).

## What runs on ws63-qemu today

- **`flashboot` / `loaderboot`** — boot to UART output (clock bring-up).
- **`ws63-liteos-app`** — boots LiteOS, prints formatted logs, creates its
  subsystem tasks and reaches **`cpu 0 entering scheduler`**. With the BT/WiFi
  tasks cut (below) it idles cleanly in the scheduler with no crash.

ws63-qemu implements the HiSilicon **"xlinx" custom RISC-V ISA** that the vendor
GCC emits, models the WS63 peripherals (UART/TIMER/GPIO/SFC/TCXO/intc/…), and
intercepts mask-ROM calls — so the unmodified vendor-compiled firmware runs.
See the ws63-qemu repo for details.

## QEMU-specific changes in this fork

- **`src/build/config/target_config/ws63/config.py`** — the BT (`BGLE_TASK_EXIST`,
  `BTH_TASK_EXIST`) and WiFi (`WIFI_TASK_EXIST`) task-creation defines are
  commented out for the `ws63-liteos-app` targets. Those subsystems' deep init
  depends on on-chip ROM data / RF calibration / hardware that QEMU cannot model
  (no ROM/efuse dump), so their tasks fault; cutting them lets the app boot
  cleanly to the scheduler. Re-enable them to build the full stack for hardware.

## Building for QEMU

```bash
cd src
python3 build.py ws63-liteos-app -c -ninja      # vendor toolchain is bundled
# -> output/ws63/acore/ws63-liteos-app/ws63-liteos-app.elf
```

Then run it on ws63-qemu:

```bash
qemu-system-riscv32 -M ws63 -nographic -serial mon:stdio \
    -kernel ws63-liteos-app.elf
```

## Boundaries (not emulator bugs)

The full BT/WiFi/flash/NV bring-up needs real chip data that isn't shippable
(mask-ROM data structures, efuse calibration, the flash NV partition). These are
physical boundaries, not register-layout issues — see ws63-qemu's
`docs/xlinx-isa.md` and `docs/alignment-analysis.md`.
