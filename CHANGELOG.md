# Changelog

All notable changes to this patch set are documented here. This version
number tracks the *patch/documentation set* in this repo, independent of
the upstream Meshtastic firmware version it's built against (currently
`2.8.2.326ccb2` — see README.txt).

## v2.0.0 — 2026-10-25

- Rebased onto `develop` @ `95ff510f476e238f60c96d0debb355b2fba69449`
  (2.8.2.326ccb2). Same change as upstream PR
  [meshtastic/firmware#12232](https://github.com/meshtastic/firmware/pull/12232).
- **Breaking:** Ethernet support moved into its own env `rak3312_eth_gw`
  (plain `rak3312` stays stock); binaries renamed accordingly.
- Fix: WiFi and Ethernet shared the WiFi MAC, so Ethernet was unreachable
  while WiFi was on. Ethernet now uses the chip's ESP_MAC_ETH address.
  DHCP reservations for the old MAC need updating.
- Syslog output goes to both the WiFi and the Ethernet syslog client.
- RAK13800-W5100S library 1.0.2 → 1.0.4.
- Factory image now includes the LittleFS filesystem (single write at `0x0`).
- New release `v2.8.2-326ccb2` with prebuilt binaries.

## v1.1.0 — 2026-10-18

- Documented a third, independent finding: config writes over the
  TCP/Ethernet API can silently fail to land once a device has
  accumulated a large NodeDB, because the client disconnects before the
  server finishes replaying cached position/telemetry/status history.
  Not a bug in this patch — see README.txt section 8 for root cause and
  workaround.
- Confirmed and documented: the device runs correctly powered solely via
  the RAK19018 PoE module (no USB connected), and with USB + PoE
  connected simultaneously.
- Added a quick-reference "known issues" table to README.md so visitors
  don't need to read the full README.txt to see what to watch out for.

No changes to the firmware patch or the prebuilt binaries in this
release — `rak3312-ethernet.patch` and the `v2.8.2-fad52bb` release
assets are identical to v1.0.0.

## v1.0.0 — 2026-10-18

- Initial publish: Ethernet/PoE support patch for the Meshtastic
  `rak3312` board target, built and tested against `develop` @
  `fad52bba0db68aff926aa02a55f4d2c4989f68dd` (2.8.2.fad52bb).
- `rak3312-ethernet.patch`, full technical writeup (`README.txt`),
  prebuilt `factory.bin` / `app-only.bin` (release `v2.8.2-fad52bb`).
- Filed upstream as [meshtastic/firmware#12190](https://github.com/meshtastic/firmware/issues/12190).
