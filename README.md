# Meshtastic Ethernet/PoE support for RAK3312

Unofficial community patch that adds working Ethernet/PoE support to the
Meshtastic `rak3312` board target (ESP32-S3 WisBlock core), which has none
upstream. For anyone running a RAK3312-based "Meshtastic Gateway" kit who wants
`network.eth_enabled` to actually do something.

> Not endorsed by or merged into the official Meshtastic project. Built against
> the `develop` branch. Use at your own risk.

## Hardware

- RAK3312 WisBlock Core Module (ESP32-S3 + SX1262)
- RAK13800 WisBlock Ethernet Module (WIZnet W5100S)
- RAK19018 WisBlock PoE Module
- RAK5005-O WisBlock Base Board (or any base board with the standard 40-pin IO connector)

Using a **RAK4631** instead? You don't need this, use the official `rak4631_eth_gw` target.

## Quick start

Download `firmware-rak3312-ethernet-2.8.2.fad52bb.factory.bin` from the
[latest release](https://github.com/jyxbk/meshtastic-rak3312-ethernet/releases/latest).
Attach the LoRa antenna before powering the board. Then:

```sh
pip install esptool meshtastic
esptool --chip esp32s3 --port COM4 erase-flash
esptool --chip esp32s3 --port COM4 write-flash 0x0 firmware-rak3312-ethernet-2.8.2.fad52bb.factory.bin
```

Replace `COM4` with your serial port (e.g. `/dev/ttyACM0` on Linux).

**Before your first config write**, set the LoRa region on its own, outside
any settings transaction, or your first settings batch may silently be lost:

```sh
meshtastic --port COM4 --set lora.region EU_868
```

## Contents

| File | Purpose |
| --- | --- |
| `firmware-rak3312-ethernet-2.8.2.fad52bb.factory.bin` ([release](https://github.com/jyxbk/meshtastic-rak3312-ethernet/releases/latest)) | Full image, flash at `0x0` after full erase |
| `firmware-rak3312-ethernet-2.8.2.fad52bb.app-only.bin` ([release](https://github.com/jyxbk/meshtastic-rak3312-ethernet/releases/latest)) | App partition only, flash at `0x10000` for updates |
| `rak3312-ethernet.patch` | Source diff against meshtastic/firmware `fad52bba0db68aff926aa02a55f4d2c4989f68dd` |
| `README.txt` | Full technical writeup |

Upstream discussion: [meshtastic/firmware#12190](https://github.com/meshtastic/firmware/issues/12190)

## Status

Confirmed working: DHCP, link detection, the TCP API on port 4403, NTP.
Confirmed working powered **solely via the RAK19018 PoE module** (no USB
connected), and with USB + PoE connected simultaneously.

## Known issues (quick list -- see README.txt for detail)

| # | Issue | Workaround |
| --- | --- | --- |
| 1 | First-ever `lora.region` change (from UNSET) mid-settings-transaction can silently fail to save | Set region alone, outside any transaction, first (README.txt §7) |
| 2 | On `develop` (not v2.7.26), config writes have been observed to not persist, unrelated to this patch | Full chip erase before first config write; not fixed by this patch (README.txt §6) |
| 3 | Static IP (`network.address_mode = STATIC`) comes up as `0.0.0.0` -- looks like a RAK13800-W5100S library bug | Use DHCP + a router-side DHCP reservation instead (README.txt §9) |
| 4 | Config writes over the TCP/Ethernet API can silently fail to land once a device has a large NodeDB (client disconnects before the server works through its position/telemetry replay backlog) | Write config over USB, or add explicit delays around the write if scripting over TCP (README.txt §8) |

## Details

See **[README.txt](README.txt)** for the pin mapping, every code change and why,
all four issues above in full (with workarounds and root causes), and
rebuild instructions.

## License

GPLv3, same as [meshtastic/firmware](https://github.com/meshtastic/firmware). See [LICENSE](LICENSE).
