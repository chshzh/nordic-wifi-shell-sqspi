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

### Step 2 — Flash the firmware

Download `merged.hex` from the [Latest Release](https://github.com/chshzh/nordic-wifi-shell-sqspi/releases/latest), then open **nRF Connect for Desktop → Programmer**, select your nRF54LM20DK, add the `.hex` file, and click **Erase & Write**.

Or flash via command line (requires a local NCS toolchain):

```sh
west flash -d nordic-wifi-shell-sqspi/build --recover 
```

### Step 3 — Connect to the console

Open a serial terminal on **VCOM0** (the first COM port enumerated by the DK) at **115200 baud** with **hardware flow control enabled** (RTS/CTS).

> **NOTE** The firmware redirects the Zephyr shell from UART20 to UART30. UART30 maps to VCOM0 on the nRF54LM20DK (Port 0, P0.6/P0.7). This is consistent with the standard unmodified `nrf7002eb2` shield behaviour, which due to potentioal conflict when using with nRF54L15DK. For nRF54LM20DK+nRF7002EB-II combination, UART20 default pins (P1.16–P1.19) are **not** used by the nRF7002 EB-II expansion header nexus — the redirect to UART30 is inherited from the upstream `nrf7002eb2` shield for nRF54LM20.

Recommended tools:
- **nRF Connect for Desktop → Serial Terminal** — select VCOM0, 115200 baud, flow control ON
- `python3 nordicsemi_uart_monitor.py --port /dev/cu.usbmodem... --baud 115200`

You should see the Zephyr boot log followed by the shell prompt `uart:~$`.

### Step 4 — Connect to Wi-Fi

```sh
uart:~$ wifi scan
uart:~$ wifi connect -s <SSID> -k 1 -p <password>
uart:~$ wifi status
```

`-k 1` selects WPA2-PSK. After a successful connect, `wifi status` shows the assigned IP address and signal strength.

---

## Developer Info

### Project Structure

```text
nordic-wifi-shell-sqspi/
├── CMakeLists.txt            ← upstream (unmodified)
├── prj.conf                  ← upstream (unmodified)
├── sysbuild.conf             ← upstream (unmodified)
├── west.yml                  ← manifest pinned to NCS v3.3.0
├── .github/workflows/
│   └── build.yml             ← CI build + rolling release
├── boards/
│   └── nrf54lm20dk_nrf54lm20a_cpuapp.conf   ← board-specific Kconfig
└── patches/
    ├── nrfxlib/              ← VPR barrier hardening + nrf_sqspi_abort()
    ├── nrf/                  ← MSPI driver retry and DMA abort recovery
    └── zephyr/               ← MSPI bus backend, DTS bindings, nrf7002eb2_mspi shield
```


### Workspace Setup


This project is a [Workspace Application](https://docs.nordicsemi.com/bundle/ncs-latest/page/nrf/dev_model_and_contributions/adding_code.html#workflow_4_workspace_application_repository_recommended). The [west.yml](west.yml) manifest pins to NCS v3.3.0:

```yaml
- name: nrf
  url: https://github.com/nrfconnect/sdk-nrf
  revision: v3.3.0
  import: true
```

#### Method 1 (Preferred) — Add to an existing NCS v3.3.0 installation

```sh
cd /opt/nordic/ncs/v3.3.0   # your existing NCS workspace root

git clone https://github.com/chshzh/nordic-wifi-shell-sqspi.git

west config manifest.path nordic-wifi-shell-sqspi
west update
```

#### Method 2 — Fresh workspace via CLI

```sh
west init -m https://github.com/chshzh/nordic-wifi-shell-sqspi --mr main ncs-sqspi
cd ncs-sqspi
west update
```

### Patching nrf, nrfxlib, and zephyr

Three patches must be applied to upstream NCS repos after `west update`. Apply them from the **NCS workspace root**:

```sh
# nrfxlib — VPR barrier hardening and nrf_sqspi_abort()
git -C nrfxlib am nordic-wifi-shell-sqspi/patches/nrfxlib/*.patch

# nrf — MSPI driver retry and DMA abort recovery
git -C nrf am nordic-wifi-shell-sqspi/patches/nrf/*.patch

# zephyr — MSPI bus backend, DTS bindings, nrf7002eb2_mspi shield
git -C zephyr am nordic-wifi-shell-sqspi/patches/zephyr/*.patch
```

Each command applies a single commit on top of the v3.3.0 tag. If a patch is already applied (e.g., rebuilding after a cache hit), `git am` will fail; run `git -C <repo> am --abort` to reset, then confirm with `git -C <repo> log --oneline`.

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

### Serial Monitor

Connect to VCOM0 at **115200 baud, hardware flow control ON**:

### Developer Notes

- **UART30 only**: UART30 (VCOM0, P0.6/P0.7) is used as the console. UART20 is disabled, inherited from the upstream `nrf7002eb2` shield for nRF54LM20. (UART20 default pins P1.16–P1.19 are not part of the expansion header nexus; the redirect is upstream shield behaviour.)
- **BUTTON3 unavailable**: The standard `nrf7002eb2` shield removes BUTTON3 (hardware pin conflict on nRF54LM20DK). The sQSPI shield inherits this.
- **Random MAC**: `CONFIG_WIFI_RANDOM_MAC_ADDRESS=y` is enabled to work around boards with blank OTP MAC (`FF:FF:FF:FF:FF:FF`).
- **Memory layout**: Top 16 KB of app-core SRAM (`0x2007c000–0x2007ffff`) is reserved for the sQSPI soft peripheral firmware and register file.

---

## Design Notes

### Patches applied

| Repo | Patch | Description |
|------|-------|-------------|
| `nrfxlib` | `0001-nrf-noup-softperipheral-sqspi-harden-VPR-barrier-syn.patch` | `__DSB()` before trigger, graduated re-trigger at 100 K / 500 K / 2 M spins, 10 M spin timeout |
| `nrf` | `0001-nrf-noup-drivers-mspi-sqspi-add-retry-and-DMA-abort-.patch` | MSPI driver: up to 2× retry on barrier/DMA timeout; calls `nrf_sqspi_abort()` on failure |
| `zephyr` | `0001-nrf-noup-wifi-nrf70-add-MSPI-bus-backend-for-sQSPI-o.patch` | MSPI bus backend for nRF70 Wi-Fi, DTS bindings, `nrf7002eb2_mspi` shield definition |

### Key design decisions

**VPR barrier synchronization** — The FLPR VPR may miss a `TASKS_TRIGGER` when it arrives immediately after completing the previous transfer. The fix issues `__DSB()` before the first trigger (ensures handshake memory writes are visible to the VPR) and re-fires the trigger at 100 K, 500 K, and 2 M busy-wait iterations if the VPR has not acknowledged.

**DMA timeout recovery** — If a DMA transfer does not complete within the nRF Wi-Fi driver timeout, `nrf_sqspi_abort()` disables the VPR core and clears all DMA events, returning the peripheral to idle. The MSPI driver retries up to 2 times before propagating an error.

**Memory reservation** — The top 16 KB of app-core SRAM is reserved for the sQSPI soft peripheral register file and DMA buffers. `cpuapp_sram` is reduced to 496 KB in the board overlay.

**VDD_IO 3.3 V requirement** — At 32 MHz the SPI half-clock period is ~15.6 ns, leaving little margin for signal edges to settle across the connector. The 3.3 V swing provides the drive current needed to charge connector parasitic capacitance within this budget; 1.8 V (the DK default) is insufficient at this speed. The `NRF_DRIVE_E0E1` extra drive mode in pinctrl complements this.

---

## License

`nordic-wifi-shell-sqspi/` application code is licensed under **Apache-2.0**.
Modified upstream files in `nrfxlib/` retain **LicenseRef-Nordic-5-Clause**.
Modified upstream files in `zephyr/` and `nrf/` retain their original licenses.
