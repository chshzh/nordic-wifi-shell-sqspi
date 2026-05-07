# Nordic Wi-Fi Shell — sQSPI (Zephyr MSPI Driver) Backend

[![Build](https://github.com/chshzh/nordic-wifi-shell-sqspi/actions/workflows/build.yml/badge.svg)](https://github.com/chshzh/nordic-wifi-shell-sqspi/actions/workflows/build.yml)
[![License](https://img.shields.io/badge/License-Apache--2.0-blue.svg)](LICENSE)
[![NCS](https://img.shields.io/badge/NCS-v3.3.0-skyblue)](https://developer.nordicsemi.com/nRF_Connect_SDK/doc/latest/nrf/index.html)

A port of the NCS Wi-Fi Shell sample to the **nRF54LM20DK + hardware-modified nRF7002 EB-II**, replacing the SPI bus with a **soft QSPI (sQSPI) peripheral** implemented in the FLPR VPR co-processor.

This is a reference implementation demonstrating that the nRF7002 Wi-Fi radio can be driven over Quad SPI from an nRF54LM20DK, unlocking higher bus throughput (tested at 32 MHz) compared to the default single-bit SPI.

- **Evaluator** — grab the pre-built `merged.hex` from the [Releases](https://github.com/chshzh/nordic-wifi-shell-sqspi/releases) page and follow the [Quick Start](#evaluator-quick-start) below. No build environment needed.
- **Developer** — clone, apply patches, build from source; see [Developer Info](#developer-info).

---

## Evaluator Quick Start

> Pre-built path — no build environment needed. ~5 minutes.
> **Requires the hardware modification** (cut SB1–4, solder SB5–8). See [Hardware Modification](#hardware-modification-spi--sqspi).

### Step 1 — Flash the firmware

Download `merged.hex` from the [Latest Release](https://github.com/chshzh/nordic-wifi-shell-sqspi/releases/latest), then open **nRF Connect for Desktop → Programmer**, select your nRF54LM20DK, add the `.hex` file, and click **Erase & Write**.

Or flash via command line (requires a local NCS toolchain):

```sh
west flash -d nordic-wifi-shell-sqspi/build --recover --dev-id <SERIAL>
```

### Step 2 — Connect to the console

Open a serial terminal on **VCOM0** (the first COM port enumerated by the DK) at **115200 baud with hardware flow control enabled** (RTS/CTS).

> **Why VCOM0?** The nRF7002 EB-II expansion header occupies some Port 1 GPIO lines that
> conflict with UART20 (the default console UART). UART30 (Port 0, P0.6/P0.7) is used
> instead and maps to VCOM0. This is standard NCS behaviour — the unmodified `nrf7002eb2`
> shield applies the same redirect.

Recommended tools:
- **nRF Connect for Desktop → Serial Terminal** — select VCOM0, 115200 baud, flow control ON
- `python3 nordicsemi_uart_monitor.py --port /dev/cu.usbmodem... --baud 115200`

You should see the Zephyr boot log followed by the shell prompt `uart:~$`.

### Step 3 — Connect to Wi-Fi

```sh
uart:~$ wifi scan
uart:~$ wifi connect -s <SSID> -k 1 -p <password>
uart:~$ wifi status
```

`-k 1` selects WPA2-PSK. After a successful connect, `wifi status` shows the assigned IP address.

---

## Project Overview

### Introduction

The standard nRF7002 EB-II shield uses a single-bit SPI bus routed through the nRF54LM20DK's expansion header. This project replaces that bus with a 4-bit QSPI path — physically wired via solder-bridge modifications to the EB-II — and drives it using Nordic's sQSPI soft peripheral (VPR firmware + MSPI API).

The application firmware is the upstream `nrf/samples/wifi/shell` sample with an MSPI bus backend added. Everything above the bus layer is unchanged.

### Supported hardware

| Board | Build target | Notes |
|-------|--------------|-------|
| nRF54LM20DK + **modified** nRF7002 EB-II | `nrf54lm20dk/nrf54lm20a/cpuapp` + `-DSHIELD=nrf7002eb2_mspi` | Requires HW mod — see Part 1 |

> **SPI reference**: the unmodified EB-II + standard `-DSHIELD=nrf7002eb2` still works. This project targets the sQSPI path only.

### Features

- **32 MHz Quad SPI** bus to nRF7002 (1-4-4 mode: single-wire command, quad addr+data)
- **VPR/FLPR soft peripheral** — no dedicated QSPI hardware block required
- Full **Zephyr MSPI API** driver (`nordic,nrf-sqspi` compatible)
- All standard Wi-Fi shell commands: `wifi scan`, `wifi connect`, `wifi status`, etc.
- Hardened VPR barrier synchronization with graduated re-trigger and DMA timeout recovery
- Automatic retry (up to 2×) on barrier or DMA timeout without upper-layer visibility
- Stable connection over 20+ consecutive boot/connect cycles

---

## Hardware Modification: SPI → sQSPI

The nRF7002 EB-II routes the nRF7002 QSPI pins to the expansion header via solder bridges. By default (SB1–SB4 populated), SPI signals pass through; after the modification (SB5–SB8 populated instead), four QSPI data lines plus CS are exposed.

### Solder bridge map

| Bridge | Default | After mod | Signal |
|--------|---------|-----------|--------|
| SB1    | closed  | **cut**   | SPI MOSI / DQ0 (single-wire path) |
| SB2    | closed  | **cut**   | SPI MISO / DQ1 (single-wire path) |
| SB3    | closed  | **cut**   | SPI SCK |
| SB4    | closed  | **cut**   | SPI CS / nRF7002 BUCKEN path |
| SB5    | open    | **solder** | QSPI DQ0 → expansion header pin 18 |
| SB6    | open    | **solder** | QSPI DQ1 → expansion header pin 20 |
| SB7    | open    | **solder** | QSPI DQ2 → expansion header pin 19 |
| SB8    | open    | **solder** | QSPI CS0 → expansion header pin 21 |

### Pin mapping after modification

After soldering, the sQSPI driver connects to these nRF54LM20DK Port 2 (SDP) pins:

| EB-II signal | nRF54LM20DK pin | Expansion header |
|-------------|-----------------|------------------|
| DQ0         | P2.2            | pin 18 |
| DQ1         | P2.4            | pin 20 |
| DQ2         | P2.3            | pin 19 |
| DQ3         | P2.0            | pin 16 |
| SCK         | P2.1            | pin 17 |
| CS0         | P2.5            | pin 21 |

Power control (BUCKEN, IOVDD_CTRL) and HOST_IRQ remain on Port 1 via the expansion header and are unchanged from the standard shield.

### Reference

Official QSPI strap documentation:
<https://docs.nordicsemi.com/bundle/ug_nrf7002_eb2/page/UG/nrf7002_EK/hw_wifi_strap_qspi.html>

---

## Patching nrf, nrfxlib, and zephyr

This project requires patches to two upstream NCS repos: `nrf/` (MSPI driver) and `nrfxlib/` (sQSPI library). These patches are **not** upstreamed — apply them to a fresh NCS v3.3.0 checkout as described in [Applying the patches](#applying-the-patches) below.

### New files (added by patches)

| File | Description |
|------|-------------|
| `nrf/drivers/mspi/mspi_sqspi.c` | Zephyr MSPI bus driver wrapping the sQSPI soft peripheral |
| `zephyr/modules/nrf_wifi/bus/mspi_if.c` | MSPI bus backend for the nRF70 Wi-Fi driver |
| `zephyr/modules/nrf_wifi/bus/mspi_if.h` | Header for mspi_if.c |
| `zephyr/dts/bindings/mspi/nordic,nrf70-mspi.yaml` | DTS binding for the MSPI bus node |
| `zephyr/dts/bindings/mspi/nordic,nrf7002-mspi.yaml` | DTS binding for the nRF7002 MSPI device node |
| `zephyr/boards/shields/nrf7002eb2_mspi/` | Shield definition (overlay + board overlay + Kconfig) |

### Modified upstream files

| File | Change |
|------|--------|
| `nrfxlib/softperipheral/include/softperipheral_regif.h` | Added `__DSB()` before trigger, graduated re-trigger at 100K/500K/2M spins, timeout guard at 10M spins |
| `nrfxlib/softperipheral/sQSPI/src/nrf_sqspi.c` | Added `nrf_sqspi_abort()`, `#undef` to suppress CTRLR0 redefinition warning |
| `nrfxlib/softperipheral/sQSPI/include/nrf_sqspi.h` | Added `nrf_sqspi_abort()` declaration |
| `zephyr/modules/nrf_wifi/bus/Kconfig` | Added `NRF70_ON_MSPI` Kconfig option |
| `zephyr/modules/nrf_wifi/bus/CMakeLists.txt` | Added mspi_if.c to build when `NRF70_ON_MSPI` |
| `zephyr/modules/nrf_wifi/bus/device.c` | Added MSPI bus registration path |
| `zephyr/modules/nrf_wifi/bus/rpu_hw_if.c` | Added MSPI wakeup command path |
| `zephyr/Kconfig.nrfwifi` | Added `NRF70_ON_MSPI` symbol |

### Key design decisions

**VPR barrier synchronization** — the FLPR VPR sometimes misses a TASKS_TRIGGER when it arrives immediately after finishing the previous barrier. The fix uses a graduated re-trigger: the host re-fires the task trigger at 100K, 500K, and 2M busy-wait iterations. The host also issues a `__DSB()` before the first trigger to ensure handshake memory writes are visible to the VPR.

**DMA timeout recovery** — if a DMA transfer does not complete within the timeout window (nRF Wi-Fi driver default), `nrf_sqspi_abort()` disables the core and clears all DMA events, returning the peripheral to idle. The MSPI driver retries the transfer up to 2 times before propagating the error.

**Memory layout** — the top 16 KB of app-core SRAM (`0x2007c000–0x2007ffff`) is reserved for sQSPI firmware and register file. `cpuapp_sram` is reduced to 496 KB in the board overlay.

---

## Changes to the Wi-Fi Shell Sample

The base sample is `nrf/samples/wifi/shell` taken verbatim. The following files were added to `nordic-wifi-shell-sqspi/`:

| File | Purpose |
|------|---------|
| `boards/nrf54lm20dk_nrf54lm20a_cpuapp.conf` | Board Kconfig: flash/ZMS/settings, log buffer, random MAC |
| `patches/nrfxlib/*.patch` | VPR barrier hardening + `nrf_sqspi_abort()` |
| `patches/nrf/*.patch` | MSPI driver retry and DMA abort recovery |
| `patches/zephyr/*.patch` | MSPI bus backend, DTS bindings, shield files |

### Board config (`boards/nrf54lm20dk_nrf54lm20a_cpuapp.conf`)

Key options added:
- `CONFIG_WIFI_CREDENTIALS=y` — credential storage
- `CONFIG_FLASH=y`, `CONFIG_ZMS=y`, `CONFIG_SETTINGS=y` — NVS-backed settings
- `CONFIG_LOG_BUFFER_SIZE=8192` — larger log ring for sQSPI diagnostic messages
- `CONFIG_WIFI_RANDOM_MAC_ADDRESS=y` — workaround for boards with blank OTP MAC (`FF:FF:FF:FF:FF:FF`)

### UART console

Console output is redirected from UART20 (conflicts with EB-II expansion header) to **UART30** with hardware flow control. UART30 maps to **VCOM0** on the nRF54LM20DK (Port 0, pins P0.6/P0.7). Connect at **115200 baud, hardware flow control (RTS/CTS)**.

---

## Applying the patches

The patches under `patches/` are `git format-patch` exports that apply on top of the
unmodified NCS v3.3.0 tags. Use `git am` to apply them to a fresh workspace.

### From a fresh NCS v3.3.0 west workspace

```sh
# 1. Initialise workspace (or reuse an existing NCS v3.3.0 checkout)
west init -m https://github.com/nrfconnect/sdk-nrf --mr v3.3.0 ~/ncs
cd ~/ncs && west update

# 2. Clone this app alongside the NCS repos
git clone https://github.com/chshzh/nordic-wifi-shell-sqspi
```

### Apply patches

From the **NCS workspace root** (`~/ncs` in the example above):

```sh
# nrfxlib — VPR barrier hardening and nrf_sqspi_abort()
git -C nrfxlib am nordic-wifi-shell-sqspi/patches/nrfxlib/*.patch

# nrf — MSPI driver retry and DMA abort recovery
git -C nrf am nordic-wifi-shell-sqspi/patches/nrf/*.patch

# zephyr — MSPI bus backend, DTS bindings, nrf7002eb2_mspi shield
git -C zephyr am nordic-wifi-shell-sqspi/patches/zephyr/*.patch
```

Each command applies a single patch commit on top of the v3.3.0 tag. If any patch fails to apply, the working tree will be in mid-`am` state; run `git -C <repo> am --abort` to reset.

### Verify

After applying, build the firmware to confirm everything compiles cleanly:

```sh
nrfutil sdk-manager toolchain launch --ncs-version=v3.3.0 -- \
  west build -b nrf54lm20dk/nrf54lm20a/cpuapp -p \
  -d nordic-wifi-shell-sqspi/build \
  nordic-wifi-shell-sqspi \
  -- -DSHIELD=nrf7002eb2_mspi
```

---

## Developer Info

### Build

```sh
nrfutil sdk-manager toolchain launch --ncs-version=v3.3.0 -- \
  west build -b nrf54lm20dk/nrf54lm20a/cpuapp -p \
  -d nordic-wifi-shell-sqspi/build \
  nordic-wifi-shell-sqspi \
  -- -DSHIELD=nrf7002eb2_mspi
```

### Flash

```sh
# nRF54LM20DK requires --recover for first flash or after mass erase
west flash -d nordic-wifi-shell-sqspi/build --recover --dev-id <SERIAL>
```

### Connect

```sh
# UART30 = VCOM0 on nRF54LM20DK (Port 0, pins P0.6/P0.7)
# Port: /dev/cu.usbmodem... or /dev/ttyACM1 on Linux
# Baud: 115200, HW flow control ON
```

### Connect to Wi-Fi

```sh
uart:~$ wifi connect -s <SSID> -k 1 -p <password>
# -k 1 = WPA2-PSK
uart:~$ wifi status
```

### Project structure

```text
nordic-wifi-shell-sqspi/
├── CMakeLists.txt            ← upstream (unmodified)
├── prj.conf                  ← upstream (unmodified)
├── sysbuild.conf             ← upstream (unmodified)
├── boards/
│   └── nrf54lm20dk_nrf54lm20a_cpuapp.conf   ← board-specific Kconfig
├── patches/
│   ├── nrfxlib/              ← patch for nrfxlib (barrier + abort)
│   ├── nrf/                  ← patch for nrf (MSPI driver retry)
│   └── zephyr/               ← patch for zephyr (MSPI backend + shield)
└── README.md                 ← this file
```

Upstream modified files live under `nrfxlib/`, `nrf/`, and `zephyr/` — see Part 2.

---

## License

`nordic-wifi-shell-sqspi/` application code is licensed under **Apache-2.0**.  
Modified upstream files in `nrfxlib/` retain **LicenseRef-Nordic-5-Clause**.  
Modified upstream files in `zephyr/` and `nrf/` retain their original licenses.
