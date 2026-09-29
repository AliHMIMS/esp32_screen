# SmallTV desk screen: flashing smalltv-mod + clawdmeter

Notes from reflashing a cheap Temu "smart weather clock" desk screen (time, weather,
custom GIF, web UI) with [smalltv-mod](https://github.com/giovi321/smalltv-mod), and
driving its Claude usage meter from a MacBook with
[clawdmeter-daemon](https://github.com/giovi321/clawdmeter-daemon).

## Hardware

Despite the listing (and the folder name), it is **not an ESP32**:

| | |
|---|---|
| MCU | ESP8266EX, 26 MHz crystal, 160 MHz |
| Flash | 4 MB (manufacturer `0x20`, device `0x4016`) |
| USB | WCH CH340 USB-serial (`VID_1A86&PID_7523`) on the USB-C port |
| Screen | 1.54" 240x240 ST7789 |
| Model | GeekMagic SmallTV / SmallTV-ultra style clone |

The smalltv-mod README says the ESP8266 SmallTV has no USB-serial chip and needs OTA
or a UART header. This unit has a CH340 built in, so it can be backed up and flashed
directly over the USB-C cable with esptool (auto-reset works, no button needed). That
also sidesteps the SmallTV-ultra "Not Enough Space" OTA problem, so the two-step
`smalltv-mod-loader.bin` install is not needed.

## Tools (Windows)

1. **CH340 driver.** Out of the box Windows lists the device as `USB Serial` with
   problem code 28 (no driver). Install WCH's `CH341SER.ZIP` from
   <https://www.wch-ic.com/downloads/CH341SER_ZIP.html> (WHQL signed):

   ```powershell
   pnputil /add-driver CH341SER\CH341SER.INF /install
   ```

   The device then shows up as `USB-SERIAL CH340 (COMx)`; here it was **COM8**.

2. **esptool.** The standalone Windows build from
   <https://github.com/espressif/esptool/releases> (used v5.4.0), so no Python needed.

## Identify the chip

```powershell
esptool --port COM8 flash-id
```

```
Detected chip type... ESP8266
Chip type:          ESP8266EX
Features:           Wi-Fi, 160MHz
Crystal frequency:  26MHz
Detected flash size: 4MB
```

## Back up the stock firmware

GeekMagic does not publish its stock images, so take a full dump before anything else:

```powershell
esptool --port COM8 --baud 460800 read-flash 0x0 0x400000 backup\stock-backup.bin
```

`verify-flash` against the dump right afterwards **fails**, and that is expected: the
stock firmware boots between the two commands and the ESP8266 SDK rewrites its
system-parameter sectors (`0x3FD000`-`0x3FFFFF`, RF calibration and WiFi config) on
every boot. Reading the flash twice and diffing showed only those three 4 KB sectors
changed; everything else matched byte for byte.

> The dump is **not committed** (see `.gitignore`). It is proprietary firmware, and
> the SDK config sectors hold the saved WiFi SSID and password in plain text.

## Flash smalltv-mod

Download `smalltv-mod-firmware.bin` from the
[releases](https://github.com/giovi321/smalltv-mod/releases) (v2.16.0 at the time) and
check its SHA-256 against the digest GitHub shows for the asset. Then erase and write
the image at `0x0`:

```powershell
esptool --port COM8 --chip esp8266 erase-flash
esptool --port COM8 --chip esp8266 --baud 460800 write-flash 0x0 smalltv-mod-firmware.bin
```

```
Wrote 780352 bytes (523185 compressed) at 0x00000000 in 12.4 seconds (503.9 kbit/s).
Verifying written data...
Hash of data verified.
```

Boot log on the serial port at 115200 baud (the garbage line before it is the ROM
bootloader talking at 74880 baud, which is normal):

```
smalltv-mod 2.16.0
[boot] reset reason: External System
[boot] settings
[boot] display
[boot] net
[boot] web
[boot] modes
[boot] done
```

## First boot

The device opens a **`SmallTV-Setup`** captive portal. Join it, pick the home WiFi,
then open the device's IP in a browser to switch between the stock/crypto ticker, the
Claude usage meter, and the plane radar. Later updates can go over WiFi from the web
UI's System tab.

## Claude usage meter: clawdmeter-daemon on macOS

[clawdmeter-daemon](https://github.com/giovi321/clawdmeter-daemon) reads Claude Code's
5-hour and 7-day usage and pushes it to the screen over HTTP.

```bash
git clone https://github.com/giovi321/clawdmeter-daemon
cd clawdmeter-daemon
./install.sh          # creates a .venv, installs deps, registers autostart
```

Pointing it at the screen:

```bash
python clawdmeter_daemon.py --push                  # find the SmallTV via mDNS
python clawdmeter_daemon.py --push-to 192.168.1.50  # or a fixed IP / hostname
```

- Settings (transport, push targets) persist in `~/.clawdmeter-daemon.json`; the tray
  icon's "Configure push targets..." menu edits them.
- `CLAWDMETER_PUSH_URL` takes the same target list as `--push-to`.
- Autostart is a per-user LaunchAgent at
  `~/Library/LaunchAgents/com.giovi321.clawdmeter.plist` (runs `--tray` at login).

Switch the screen to the usage meter mode in its web UI and the bars update as the
daemon pushes.

## Restore stock firmware

```powershell
esptool --port COM8 write-flash 0x0 backup\stock-backup.bin
```
