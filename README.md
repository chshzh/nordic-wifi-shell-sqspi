# Nordic Wi-Fi Shell — sQSPI (Zephyr MSPI Driver) Backend

[![Build](https://github.com/chshzh/nordic-wifi-shell-sqspi/actions/workflows/build.yml/badge.svg)](https://github.com/chshzh/nordic-wifi-shell-sqspi/actions/workflows/build.yml)

A port of the NCS Wi-Fi Shell sample to the **nRF54LM20DK + hardware-modified nRF7002 EB-II**, replacing the SPI bus with a **soft QSPI (sQSPI) peripheral** implemented in the FLPR VPR co-processor.

This is a reference implementation demonstrating that the nRF7002 Wi-Fi radio can be driven over Quad SPI from an nRF54LM20DK, unlocking higher bus throughput (tested at 32 MHz) compared to the default single-bit SPI.

- **Evaluator** — grab the pre-built firmware from the [Releases](https://github.com/chshzh/nordic-wifi-shell-sqspi/releases) page and follow the [Evaluator Quick Start](#evaluator-quick-start) below. No build environment needed.
- **Developer** — clone, apply patches, build from source; see [Developer Info](#developer-info).

---

## Evaluator Quick Start

> Evaluator path — no build environment needed. Requires one hardware modification. ~15 minutes total.

### Step 1 — Hardware modification: SPI → sQSPI

The nRF7002 EB-II has solder bridges that route the nRF7002 QSPI data lines either to the SPI path (SB1–SB4, closed by default) or to the QSPI path (SB5–SB8, open by default). The modification cuts SB1–SB4 and closes SB5–SB8.

**Solder bridge modification:**

| Bridge | Before | After | Signal |
|--------|--------|-------|--------|
| SB1    | closed | **cut**    | SPI MOSI / DQ0 (old single-wire path) |
| SB2    | closed | **cut**    | SPI MISO / DQ1 (old single-wire path) |
| SB3    | closed | **cut**    | SPI SCK |
| SB4    | closed | **cut**    | SPI CS / BUCKEN path |
| SB5    | open   | **solder** | QSPI DQ0 → expansion header pin 18 (P2.2) |
| SB6    | open   | **solder** | QSPI DQ1 → expansion header pin 20 (P2.4) |
| SB7    | open   | **solder** | QSPI DQ2 → expansion header pin 19 (P2.3) |
| SB8    | open   | **solder** | QSPI CS0 → expansion header pin 21 (P2.5) |

Reference: [nRF7002 EB-II QSPI strap guide](https://docs.nordicsemi.com/bundle/ug_nrf7002_eb2/page/UG/nrf7002_EK/hw_wifi_strap_qspi.html)

**After soldering — configure the nRF54LM20DK with Board Configurator:**

Open **nRF Connect for Desktop → Board Configurator**, select your nRF54LM20DK, and verify:

1. **External Memory (QSPI)** — set to **Disabled**. The Port 2 SDP pins (P2.0–P2.5) used by sQSPI are shared with the on-board external flash in the default DK configuration. Disabling external memory frees these pins for the sQSPI peripheral.

2. **VDD_IO** — set to **3.3 V** (default is 1.8 V). At 32 MHz, the SPI signals must transition within a ~15.6 ns half-clock period. The 3.3 V swing provides the drive current needed to charge and discharge the connector parasitic capacitance in time; 1.8 V is insufficient at this speed.

> These settings are stored in the DK's onboard configuration and persist across reboots.

### Step 2 — Flash the firmware

Download `nordic-wifi-shell-sqspi-nrf54lm20dk-nrf7002ebii-ncs3.3.0.hex` from the [Latest Release](https://github.com/chshzh/nordic-wifi-shell-sqspi/releases/latest).

**Option A — GUI (nRF Connect for Desktop):** *(recommended)*

Open **nRF Connect for Desktop → Programmer**, select your nRF54LM20DK, add the `.hex` file, and click **Erase & Write**.

**Option B — CLI (nrfutil):**

```sh
# Recover the device first (required on first flash or after mass erase)
nrfutil device recover

# Program the hex file
nrfutil device program \
  --firmware nordic-wifi-shell-sqspi-nrf54lm20dk-nrf7002ebii-ncs3.3.0.hex \
  --options chip_erase_mode=ERASE_ALL
```

### Step 3 — Connect to the console

Open a serial terminal on **VCOM1** at **115200 baud**. On the nRF54LM20DK, the application console (UART20, P1.16/P1.17) is enumerated as **VCOM1** — the second COM port.

You should see the Zephyr and NCS boot log followed by the shell prompt `uart:~$`.

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


### Development Env Setup

> Make sure [Hardware modification](#step-1--hardware-modification-spi--sqspi) has been done.

#### Add to an existing NCS v3.3.0 installation

```sh
cd /opt/nordic/ncs/v3.3.0   # your existing NCS workspace root

git clone https://github.com/chshzh/nordic-wifi-shell-sqspi.git

west config manifest.path nordic-wifi-shell-sqspi
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
west flash -d nordic-wifi-shell-sqspi/build --recover
```

### Serial Monitor

Connect to **VCOM1** (UART20, P1.16/P1.17) at **115200 baud** — this is the second COM port enumerated by the nRF54LM20DK.

### Developer Notes

- **UART20 / VCOM1**: The default Zephyr console (UART20, P1.16/P1.17) is used — no redirect needed. On the nRF54LM20DK, UART20 is enumerated as VCOM1. UART20 pins P1.16–P1.19 are not used by the nRF7002 EB-II expansion header nexus (which maps gpio1.2, 1.3, 1.4, 1.5, 1.6, 1.7, 1.13 only), so there is no pin conflict.
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
| `zephyr` | `0002-zephyr-noup-modules-nrf_wifi-bus-mspi_if-fix-hl_read.patch` | Fix `hl_read` for per-word addressed reads |

### Key design decisions

**VPR barrier synchronization** — The FLPR VPR may miss a `TASKS_TRIGGER` when it arrives immediately after completing the previous transfer. The fix issues `__DSB()` before the first trigger (ensures handshake memory writes are visible to the VPR) and re-fires the trigger at 100 K, 500 K, and 2 M busy-wait iterations if the VPR has not acknowledged.

**DMA timeout recovery** — If a DMA transfer does not complete within the nRF Wi-Fi driver timeout, `nrf_sqspi_abort()` disables the VPR core and clears all DMA events, returning the peripheral to idle. The MSPI driver retries up to 2 times before propagating an error.

**Memory reservation** — The top 16 KB of app-core SRAM is reserved for the sQSPI soft peripheral register file and DMA buffers. `cpuapp_sram` is reduced to 496 KB in the board overlay.

**VDD_IO 3.3 V requirement** — At 32 MHz the SPI half-clock period is ~15.6 ns, leaving little margin for signal edges to settle across the connector. The 3.3 V swing provides the drive current needed to charge connector parasitic capacitance within this budget; 1.8 V (the DK default) is insufficient at this speed. The `NRF_DRIVE_E0E1` extra drive mode in pinctrl complements this.

---

## Performance Results

Measured on NCS v3.3.0 with `overlay-zperf.conf` + `overlay-high-performance.conf`, 5 GHz (802.11ac), 3 runs × 5 s UDP zperf. Mac iperf endpoint at 192.168.75.216.

| Device | Bus | Clock | TX (Mbps) | RX (Mbps) |
|--------|-----|-------|-----------|----------|
| nRF54LM20DK + nRF7002 EB-II | sQSPI (FLPR VPR) | 32 MHz | 10.79 / 10.96 / 10.81 | 10.11 / 10.32 / 9.97 |
| nRF7002DK (nRF5340) | QSPI | 32 MHz | 15.63 / 15.70 / 15.60 | 13.82 / 13.67 / 13.68 |

> The sQSPI path adds ~2–3 Mbps overhead vs native QSPI, reflecting the FLPR VPR handshake and DMA round-trip cost. Both targets achieve ≫ the SPI@8MHz baseline of ~4–5 Mbps.

---

## License

`nordic-wifi-shell-sqspi/` application code is licensed under **LicenseRef-Nordic-5-Clause**.
Modified upstream files in `/nrfxlib`,`nrf/` and `zephyr/` retain their original licenses.
