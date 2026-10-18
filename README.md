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

## Details

See **[README.txt](README.txt)** for the pin mapping, every code change and why,
the two upstream bugs found along the way (with workarounds), known issues and
rebuild instructions.

## License

GPLv3, same as [meshtastic/firmware](https://github.com/meshtastic/firmware). See [LICENSE](LICENSE).
