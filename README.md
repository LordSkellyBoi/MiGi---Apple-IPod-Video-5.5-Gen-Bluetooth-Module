# MiGi – Bluetooth module for the iPod Video 5.5G (Rockbox)

Adds Bluetooth audio output to an iPod Video running Rockbox. An STM32G031 drives a
Feasycom **BT1058** (A2DP source) and a **PCM1822** ADC, talks to the iPod over the dock's
Apple Accessory Protocol (iAP) serial port, and shows a Bluetooth menu on the iPod through a
small Rockbox plugin.

Features
- Scan, pair, connect, disconnect and **forget** devices from the iPod screen
- Auto-connect to the most recently used device (periodic scan, can be switched off in the menu)
- Bluetooth on/off (switches the BT1058 power via the AP22802)
- Codec toggle (SBC / aptX family – only codecs the BT1058 can send as a *source*)
- Play / pause / next / previous from the speaker or headphone buttons (works with the Hold switch on)
- Saved settings and device list survive power cycles (last 2 KB flash page)

## Repository contents

| File | What it is |
|---|---|
| `rockbox-btlink.patch` | **The one Rockbox patch** (iAP "BT link" lingo + plugin API + `btpicker` plugin) |
| `main.c`, `stm32g0xx_it.c` | STM32 firmware user code (CubeMX project) |
| `ipod_ping.py`, `rb_probe.lua` | Optional bench helpers |

`btlink.patch`, `btlink_playback.patch` and `btpicker2.patch` are **obsolete** – use only
`rockbox-btlink.patch`.

## 1. Rockbox side

The patch was made against the Rockbox tree used for development (build `dbcee0deae`,
Rockbox "master"). It touches:

- `apps/iap/iap-btlink.c/.h` (new) – private lingo `0x1E`, frame queues, and direct playback handling
- `apps/iap/iap-core.c`, `apps/SOURCES` – hook into the iAP dispatcher / periodic task
- `apps/plugin.h`, `apps/plugin.c` – `PLUGIN_API_VERSION` 283, adds `iap_btlink_send/recv`
- `apps/plugins/btpicker.c` (new), `apps/plugins/SOURCES`, `apps/plugins/CATEGORIES`

Build (Linux, macOS or a GitHub Codespace – Windows needs WSL2 or a Codespace):

```sh
git clone https://git.rockbox.org/rockbox.git && cd rockbox
patch -p1 < /path/to/rockbox-btlink.patch      # or: git apply /path/to/rockbox-btlink.patch
tools/rockboxdev.sh --prefix=$HOME/cross       # choose "a" (arm-elf-eabi); once only
export PATH=$HOME/cross/bin:$PATH
mkdir build && cd build
../tools/configure                              # choose the iPod Video target, Normal build
make -j4 && make zip
```

Unzip `rockbox.zip` into the root of the iPod's `.rockbox` parent (as with a normal Rockbox install).
If the patch does not apply cleanly on a newer Rockbox, the hunks are small; re-apply them by hand.

**Important:** the firmware and the plugin must come from the *same build*. After rebuilding,
copy the new `rockbox.ipod` **and** the whole `.rockbox` folder (the plugin is
`.rockbox/rocks/apps/btpicker.rock`). If you edit `btpicker.c`, edit it in the source tree
(`apps/plugins/`), not in `build/`.

Run the plugin from *Plugins → Applications → btpicker*.

## 2. STM32 side

- MCU: STM32G031F8P6. Generate the project with CubeMX (`IPod_Bluetooth.ioc`), keep the
  `USER CODE` blocks, and replace `main.c` / `stm32g0xx_it.c` contents accordingly.
- USART2 = BT1058 (PA2 TX, PA3 RX, 115200 8N1). USART1 = iPod (PB6 TX, PB7 RX, **19200**).
- AP22802 power switch enable = PA0. **The I2S peripheral stays disabled** – the BT1058 is the I2S master.
- Firmware switches at the top of `main.c`:

| Switch | Release value | Meaning |
|---|---|---|
| `BT_PROVISION` | 1 | Check/write BT1058 settings at boot (read first, write only if different) |
| `BT_NAME` / `BT_PROFILE` / `BT_AUTOCONN` | `"MiGi"` / `"655699"` / `"5"` | Module name, profiles, auto-connect |
| `IPOD_IAP`, `IAP_BAUD` | 1, 19200 | iPod link |
| `PLAYBACK_VIA_LINK` | 1 | Speaker buttons are executed inside Rockbox (needed for Hold switch) |
| `SETTINGS_FLASH` | 1 | Persist settings at `0x0800F800` (keep the last flash page free of code) |
| `BOOT_CONNECT`, `SPEAKER_MAC` | 0 | Optional fixed-speaker fallback; not needed with auto-connect |
| `IAP_SELFTEST`, `BT_SELFTEST` | **0** | Bench self-tests – leave off |

Flash with ST-LINK (SWD). Debug variables (`dbg_*`) can be watched in the debugger.

## 3. Wiring

BT1058 ↔ STM32: UART cross-over (STM32 TX → BT1058 RX and vice versa). Audio path (analog version):
iPod headphone/line → PCM1822 → I2S → BT1058. BT1058 is configured `AT+AUXCFG=3`, `AT+I2SCFG=33` (master, 48 kHz, 24-bit).

Dock connector (pin names are from the iPod's point of view):

| Dock pin | Function | Connect to |
|---|---|---|
| 11 | Serial GND | GND |
| 12 | iPod TX | STM32 PB7 (USART1 RX) |
| 13 | iPod RX | STM32 PB6 (USART1 TX) |
| 18 | 3.3 V accessory power | optional supply |
| 21 | Accessory identify | resistor to GND (~500 kΩ; 510 kΩ works) |

PCM1822 notes: DREG, AREG and VREF must **not** be tied to GND – decouple them with capacitors.
Disconnect any USB-UART adapter TX from the module RX line when the STM32 is running.

## 4. Link protocol (STM32 ↔ plugin)

iAP frame: `FF 55 <len> <lingo 0x1E> <cmd> <data…> <checksum>`.

Plugin → STM32: `0x10` open menu, `0x11` select item id, `0x12` back, `0x13` close, `0x14` tick.
STM32 → plugin: `0x90` begin, `0x91` header (line, text), `0x92` item (id, flags, text), `0x93` end,
`0x94` exit, `0x95` toast, `0xA0` playback (1 play, 2 pause, 3 next, 4 prev – executed directly by Rockbox).

The plugin is a generic menu client: new menu items/features are added in STM32 firmware only,
with no Rockbox rebuild.

## 5. Known limitations

- **Codecs:** the BT1058 supports LDAC/AAC only as a *sink*. As a source it sends SBC and aptX/HD/AD.
- **Not lossless:** analog hop + lossy Bluetooth codec. A direct digital I2S tap removes the first.
- **Volume** with the analog PCM1822 input is lower than the iPod's aux output.
- Do not change `AT+AUDMODE` from 0 (LE Audio mode breaks I2S input and scanning).
- Rockbox must be the running firmware; stock Apple firmware is unsupported.

## 6. Troubleshooting

| Symptom | Likely cause |
|---|---|
| Plugin says "waiting for module" | Firmware/plugin version mismatch, dock wiring, or pin-21 resistor |
| STM32 gets no replies from BT1058 | Cold joint, or a USB-UART adapter still driving the RX line |
| No audio | PCM1822 DREG/AREG/VREF grounded; I2S clock not present |
| Speaker buttons do nothing with Hold on | `PLAYBACK_VIA_LINK` off or patch missing the `0xA0` handler |
| AT commands time out right after boot | The BT1058 needs time to boot; firmware waits with `BT_WaitReady` |
