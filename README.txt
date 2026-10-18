==============================================================================
 RAK3312 + RAK13800 (W5100S) + RAK19018 PoE - Ethernet support for Meshtastic
==============================================================================

Author's note: this patch adds working Ethernet/PoE support to the
Meshtastic "rak3312" board target, which officially has none. It is based
on meshtastic/firmware "develop" branch, commit fad52bba0db68aff926aa02a
55f4d2c4989f68dd (2026-10-17), version string 2.8.2.fad52bb.

This is an UNOFFICIAL, community patch. It is not endorsed by, reviewed by,
or merged into the official Meshtastic project. Use at your own risk. See
"KNOWN ISSUES" and "WHY THIS ISN'T JUST A PULL REQUEST YET" at the bottom
before relying on it for anything important.


------------------------------------------------------------------------------
1. HARDWARE THIS IS FOR
------------------------------------------------------------------------------

  - RAK3312 WisBlock Core Module (ESP32-S3 + Semtech SX1262 LoRa)
  - RAK13800 WisBlock Ethernet Module (WIZnet W5100S-L chip)
  - RAK19018 WisBlock PoE Module (mounts on top of the RAK13800)
  - RAK5005-O WisBlock Base Board (or any base board with the standard
    40-pin WisBlock IO connector)

This is exactly the module combination RAKwireless sells as part of various
"Meshtastic Gateway" starter kits. RAKwireless's own documentation lists
RAK3312 as Ethernet-gateway-capable with this module -- but as of the
commit above, the official Meshtastic firmware's "rak3312" board target
has ZERO Ethernet pin definitions. `network.eth_enabled` is a silent no-op.
This patch is what's actually needed to make the hardware combo work.

If your WisBlock core module is the RAK4631 (nRF52840) instead of the
RAK3312 (ESP32-S3), you do NOT need this patch -- Meshtastic already ships
an official `rak4631_eth_gw` board target with working Ethernet.


------------------------------------------------------------------------------
2. WHAT'S IN THIS FOLDER
------------------------------------------------------------------------------

  firmware-rak3312-ethernet-2.8.2.fad52bb.factory.bin
      The complete, ready-to-flash image (bootloader + partition table +
      app + filesystem, merged into one file). Flash this at offset 0x0
      after a full chip erase. This is what most people want.
      MD5: 9eb3ed702342d64f6be82cad705c1eaa

  firmware-rak3312-ethernet-2.8.2.fad52bb.app-only.bin
      Just the application partition. Use this ONLY if you already have a
      working rak3312 Meshtastic install and just want to update the app
      without touching your existing config/NVS/filesystem. Flash at
      offset 0x10000. Do NOT use this for a first-time flash.
      MD5: ed8f515e728615db6f8af935ce87ff6b

  rak3312-ethernet.patch
      A `git diff` patch with every source change, against the exact
      commit named above. This is the actual, reviewable change -- read
      this if you want to understand precisely what was modified, or if
      you want to rebuild it yourself, or port it to a newer Meshtastic
      version.

  README.txt
      This file.


------------------------------------------------------------------------------
3. QUICK START - JUST FLASH THE BINARY
------------------------------------------------------------------------------

Requirements: Python 3, a USB cable to the RAK3312's USB-C port.

  pip install esptool meshtastic

Put the board in range of the LoRa antenna BEFORE powering it on. Running
an ESP32 LoRa radio without an antenna attached can damage the RF front
end. Then:

  esptool --chip esp32s3 --port COM4 erase-flash
  esptool --chip esp32s3 --port COM4 write-flash 0x0 firmware-rak3312-ethernet-2.8.2.fad52bb.factory.bin

(Replace COM4 with your actual serial port; on Linux/macOS it will be
something like /dev/ttyACM0 or /dev/cu.usbmodemXXXX.)

Full chip erase is required on a first flash, and strongly recommended any
time you are coming from a different firmware build -- Meshtastic/ESP32
devices have been observed to silently fail to persist new settings to
flash after an app-only reflash if the NVS/LittleFS area is not freshly
erased. See section 6 for the full story.

After flashing, the device boots with completely default settings (no
region set, no owner name, Ethernet disabled). Configure it with the
`meshtastic` Python CLI or the Meshtastic Android/iOS/Web app, same as any
other Meshtastic device, with ONE IMPORTANT CAVEAT -- read section 5
("THE REGION-CHANGE BUG") before you set the LoRa region, or your very
first config write may silently vanish on reboot.


------------------------------------------------------------------------------
4. PIN MAPPING (if you want to understand or adapt the wiring)
------------------------------------------------------------------------------

These come from the official RAKwireless datasheets for the RAK5005-O base
board, the RAK3312 core module, and the RAK13800 Ethernet module. The
RAK13800 uses the WisBlock base board's SHARED SPI bus (not a per-slot
SPI), plus two WB_IOx pins for reset/interrupt:

  Signal                  WisBlock connector pin    RAK3312 ESP32-S3 GPIO
  ----------------------  ------------------------  ---------------------
  SPI CS   (W5100S CSn)   pin 25 (shared SPI_CS)    GPIO12
  SPI CLK                 pin 26 (shared SPI_CLK)   GPIO13
  SPI MISO                pin 27 (shared SPI_MISO)  GPIO10
  SPI MOSI                pin 28 (shared SPI_MOSI)  GPIO11
  Ethernet RSTn           pin 31 = WB_IO3           GPIO41
  Ethernet INTn           pin 38 = WB_IO6           GPIO39 (unused by
                                                      this patch -- the
                                                      Ethernet library
                                                      polls, it doesn't
                                                      use the IRQ line)

Note: GPIO14 (WB_IO2) is already used by the stock rak3312 board
definition as PIN_3V3_EN, the always-on WisBlock IO-slot power rail. The
RAK13800 draws its power from that shared rail, so this patch does NOT
define a separate PIN_ETH_POWER_EN -- it isn't needed. If you ever port
this to a board where the WisBlock IO slot power rail is NOT already
enabled by firmware, you will need to add that pin yourself (see
`PIN_ETH_POWER_EN` handling already present generically in
src/mesh/eth/ethClient.cpp -- just define the macro in your variant.h).

The RAK13800's INTn/IRQ line is connected but unused: the RAK13800-W5100S
Arduino library (see section 5b) uses polling, not interrupt-driven RX, so
wiring GPIO39 is future-proofing, not a current requirement.


------------------------------------------------------------------------------
5. WHAT THE PATCH ACTUALLY CHANGES (and why)
------------------------------------------------------------------------------

Five files are touched. Full diff in rak3312-ethernet.patch; summary here.

5a) variants/esp32s3/rak3312/variant.h
    Adds the pin defines from section 4, plus:
      #define HAS_ETHERNET 1
      #define ETH_SPI_PORT EthSPI
    HAS_ETHERNET is the master switch Meshtastic's own code checks
    (configuration.h, ethClient.cpp, DeviceMetadata.hasEthernet, etc).
    Without it the Ethernet code in Meshtastic's own tree is compiled out
    entirely for this board and `network.eth_enabled` does nothing.

    IMPORTANT: ETH_SPI_PORT is NOT set to the global `SPI` object, even
    though a naive copy of the RAK11310 (RP2040) board's approach would do
    that. See 5c for why -- this cost real debugging time and is the one
    detail most likely to bite someone porting this patch elsewhere.

5b) variants/esp32s3/rak3312/platformio.ini
    Three additions to [env:rak3312]:

    - `custom_sdkconfig`: CONFIG_ETH_ENABLED=y and
      CONFIG_ARDUINO_SELECTIVE_Ethernet=y. Meshtastic's shared esp32_common
      config (variants/esp32/esp32-common.ini) explicitly DISABLES the
      Arduino "Ethernet" library and the underlying ESP-IDF Ethernet
      support for every ESP32 board by default, with a comment literally
      saying "re-enable on variants that need it". This re-enables it.
      (On Meshtastic firmware versions that predate this sdkconfig
      mechanism -- e.g. the v2.7.26 stable tag -- this section does not
      exist and must be omitted; Arduino's Ethernet lib is not selectively
      excluded on those older builds.)

    - `build_src_filter`: adds `+<mesh/eth/>`. The same esp32_common.ini
      EXCLUDES the entire src/mesh/eth/ directory from the build for every
      ESP32 board by default (`-<mesh/eth/>`). Without re-including it,
      ethClient.cpp (which implements initEthernet()/reconnectETH(), the
      actual DHCP/static-IP/link-state logic) is never compiled, and you
      get a linker error ("undefined reference to initEthernet()") the
      moment HAS_ETHERNET causes something else to call it.

    - `lib_deps`: adds RAKWireless's own Arduino library for this exact
      chip: https://github.com/RAKWireless/RAK13800-W5100S (pinned to tag
      1.0.2, same version the official `rak11310` board target already
      uses in production). This is a fork of the classic Arduino
      "Ethernet" library API (Ethernet.h / EthernetClient / EthernetServer
      / EthernetUDP), adapted for the W5100S chip's quirks -- notably a
      MAC-address-based "did the chip just brown-out and reset itself"
      detection, important because the RAK19018 PoE module can sag under
      load and reset the W5100S while the ESP32-S3 keeps running.

5c) src/mesh/eth/ethClient.cpp
    Two kinds of change:

    (i) A new SPI peripheral, `EthSPI`, declared and begin()'d with the
        Ethernet pins from section 4:

          #ifdef RAK3312
          SPIClass EthSPI(HSPI);
          #endif
          ...
          #elif defined(RAK3312)
          ETH_SPI_PORT.begin(ETH_SCLK_PIN, ETH_MISO_PIN, ETH_MOSI_PIN, PIN_ETHERNET_SS);
          #endif

        WHY THIS MATTERS: the straightforward thing to do (and what this
        patch originally tried) is `#define ETH_SPI_PORT SPI` -- reuse the
        one global default SPIClass instance, same as the RAK11310 board
        does. On ESP32-S3, this does NOT work, and the failure is
        completely silent: Meshtastic's LoRa radio code already calls
        `SPI.begin(LORA_SCK, LORA_MISO, LORA_MOSI, LORA_CS)` during boot,
        on pins 5/3/6/7. Arduino-ESP32's SPIClass::begin() silently NO-OPS
        if the bus is already initialized -- a second call with DIFFERENT
        pins (13/10/11/12 for the Ethernet chip) does not actually move
        the underlying SPI peripheral's GPIO matrix routing. The result:
        Ethernet.begin() runs, DHCP times out, and the log says
        "Ethernet shield was not found" -- because the W5100S is, at the
        hardware level, still wired to the LoRa radio's pins, not its own.
        The ESP32-S3 has multiple independent SPI peripherals (the Arduino
        core calls them FSPI/HSPI); this patch gives Ethernet its own
        (HSPI) so both devices get properly pinned.

        If you are porting this patch to a different ESP32 board, or a
        newer/older Arduino-ESP32 core, re-verify this specifically: watch
        the boot log for TWO distinct `SPI.begin(...)` / SPI init lines
        with the correct, DIFFERENT pin sets, not one.

    (ii) Renaming four globals and two functions that collide at LINK TIME
         the moment a board has both HAS_WIFI and HAS_ETHERNET at once
         (every existing Ethernet-capable Meshtastic board is Ethernet-
         *only*; RAK3312 is WiFi-AND-Ethernet-capable, so this is a new
         combination nobody hit before):

           ntpUDP          -> ethNtpUDP
           timeClient      -> ethTimeClient
           syslogClient    -> ethSyslogClient
           syslog          -> ethSyslog
           initApiServer() -> initEthSocketApiServer()
           deInitApiServer()-> deInitEthSocketApiServer()

         src/mesh/wifi/WiFiAPClient.cpp and src/mesh/api/WiFiServerAPI.cpp
         define globals/functions with the EXACT same names for the WiFi
         path. Historically fine, because no board ever linked both files.
         RAK3312 links both (it has WiFi hardware too), and the linker
         fails with "multiple definition of ...". The fix only touches the
         eth-specific files; WiFiAPClient.cpp / WiFiServerAPI.* are
         completely untouched, so every other board's build is unaffected.

5d) src/mesh/api/ethServerAPI.h and .cpp
    Just the declaration/definition side of the initApiServer/
    deInitApiServer rename from 5c(ii) above.


------------------------------------------------------------------------------
6. THE CONFIG-PERSISTENCE DEAD END (read this if settings "don't save")
------------------------------------------------------------------------------

While bringing this patch up, config changes (region, modem preset, owner
name, etc.) appeared to not persist across a reboot -- you'd set something,
read it back in the same session, reboot, and it would be back to firmware
defaults. This cost a lot of debugging time and is NOT caused by this
patch. Recording it here so nobody re-discovers it the hard way:

  - It reproduces on a completely STOCK, unpatched `develop` branch
    checkout of meshtastic/firmware, on this exact hardware, with no
    Ethernet code involved at all.
  - It does NOT reproduce on the v2.7.26.54e0d8d stable tag.
  - So: something between 2.7.26 and current `develop` introduced a
    regression in settings persistence (at least for this board/chip).
    This patch does not fix that regression -- it's an unrelated, pre-
    existing upstream issue. If you hit it, try:
      1) A full chip erase before your first config write (section 3).
      2) Waiting several seconds after `commitSettingsTransaction()` /
         before power-cycling -- some of what looked like "it didn't
         persist" was actually "the write hadn't finished flushing yet".
      3) If it still doesn't stick, you've hit the same upstream
         regression; building against the 2.7.26 tag will avoid it, but
         then you hit a SEPARATE problem -- see section 7.


------------------------------------------------------------------------------
7. THE REGION-CHANGE BUG (read this BEFORE your first config write)
------------------------------------------------------------------------------

This one you WILL hit, and it's specific to a fresh/factory-reset device,
so read this before you touch the region setting.

Symptom: you connect to a freshly-flashed device, open a settings
transaction, set lora.region (e.g. to EU_868) together with anything else,
commit the transaction, reboot -- and NONE of it is saved. Not just the
region: the whole batch.

Root cause (confirmed by reading AdminModule.cpp and NodeDB.cpp, and by
capturing the live device debug log during the failure): the VERY FIRST
time a device's LoRa region is changed away from UNSET, Meshtastic mints a
brand-new PKI keypair and, since the device's node number is defined as
CRC32(public key), the device's OWN NODE NUMBER CHANGES, mid-transaction:

  Old node num 0x756f8c0c now 0x1669dc58, remove stale identity

If this happens while a multi-step settings transaction is open (i.e. you
called beginSettingsTransaction(), then writeConfig(), and now you're
about to send commitSettingsTransaction() as a SEPARATE packet), that
commit packet is still addressed to the OLD node number. The device's
own routing layer only accepts packets addressed to its CURRENT node
number, so the commit is silently dropped. The deferred disk write that
commit was supposed to trigger never happens. Nothing crashes, nothing
logs an error visible without the debug log API -- it just quietly
doesn't save.

This is a one-time situation: it can only happen on the single occasion a
device's region is UNSET and you're setting it for the first time. Once
region is set, the node number is stable and ordinary begin/commit
transactions work completely normally, forever after.

WORKAROUND (confirmed working): set the region ONCE, by itself, with NO
settings transaction wrapped around it:

  meshtastic --port COM4 --set lora.region EU_868

(No --begin-edit / --commit-edit, no other --set in the same breath.)
Wait ~10-15 seconds, then reconnect and verify `region` reads back
correctly before doing anything else. After that, proceed completely
normally -- transactions, multiple fields at once, whatever -- for
everything else (modem preset, Ethernet/WiFi toggles, owner name, ...).

This is an upstream Meshtastic design wrinkle, not something this patch
introduces or fixes. It would affect ANY board/config combination hitting
"region changes identity" + "that happens inside a transaction" + "the
client addresses packets by a node-number it cached before the change".
Scripted/CLI usage (as above) is far more likely to hit it than the
official phone app, which is presumably why this hasn't been reported
widely.


------------------------------------------------------------------------------
8. KNOWN ISSUES
------------------------------------------------------------------------------

  - STATIC IP MODE IS BROKEN. Setting `network.address_mode` to STATIC and
    filling in network.ipv4_config (ip/gateway/subnet/dns) results in the
    device coming up with Local IP 0.0.0.0 -- the static values are
    correctly saved to config (verified via `meshtastic --info`) but are
    not correctly applied by the `Ethernet.begin(mac, ip, dns, gateway,
    subnet)` call. This looks like a bug in the RAK13800-W5100S Arduino
    library's static-IP code path specifically (the DHCP path, which this
    patch exercises heavily, works perfectly). WORKAROUND: use DHCP
    (`address_mode` = DHCP, the default) and set a DHCP reservation for
    the device's MAC address on your router/DHCP server instead of a
    static IP on the device. The Ethernet interface's MAC address is
    visible via `meshtastic --info` (look for the `macaddr` field on the
    device's own node entry) -- nobody has investigated the W5100S library
    source far enough yet to find/fix the real bug.

  - Flash usage on the merged factory image is ~95% of the 16MB OTA
    app partition's budget for this board (2,302,551 / 2,424,832 bytes at
    time of writing). There is not a lot of headroom left for this board
    to grow further without either trimming something else out of the
    build or revisiting the partition table.

  - This is built from `develop` (pre-release, unstable-by-definition)
    branch, not a tagged stable release, because of the section 6
    regression. Expect bugs unrelated to this patch that are simply
    "develop branch is develop branch".

  - The RAK13800's INTn (interrupt) line is wired per the pin table in
    section 4 but not actually used -- the library polls. If a future
    revision wants interrupt-driven RX, GPIO39 is where it's connected.


------------------------------------------------------------------------------
9. HOW TO REBUILD THIS YOURSELF
------------------------------------------------------------------------------

  git clone https://github.com/meshtastic/firmware meshtastic-firmware
  cd meshtastic-firmware
  git checkout fad52bba0db68aff926aa02a55f4d2c4989f68dd
  git apply /path/to/rak3312-ethernet.patch
  pip install -U platformio            # needs PlatformIO Core >= 6.2.0
  pio run -e rak3312

  Output appears under .pio/build/rak3312/ -- the file you want is
  firmware-rak3312-<version>.factory.bin

Notes if you hit problems:

  - Windows users: the ESP-IDF/Arduino-ESP32 toolchain installs very long,
    deeply-nested paths (some components bundle ESP-Matter/ESP-IDF trees
    well over 200 characters deep). If PlatformIO's package install step
    fails with a Python "FileNotFoundError" deep in a temp extraction
    path, that's the Windows MAX_PATH (260 char) limit, not a real error.
    Fix: set a short PlatformIO core directory before building, e.g.:
      (PowerShell)  $env:PLATFORMIO_CORE_DIR = "C:\pio"
    and rerun `pio run`.

  - The patch applies cleanly to the exact commit named above. If you're
    applying it to a different commit/tag, see section 6/7 for why some
    commits (anything at/before v2.7.26) will also need you to DELETE the
    `custom_sdkconfig` block this patch adds to platformio.ini -- that
    PlatformIO feature plus the Arduino-Ethernet-selective-compilation
    mechanism it switches on didn't exist yet on 2.7.26, and the project
    file will fail to parse if you leave it in. On 2.7.26 you will also
    hit a SEPARATE problem: the pinned RAK13800-W5100S library version
    doesn't compile cleanly against the older Arduino-ESP32 core that
    release's platformio.ini points at (`ethServerPort` ends up an
    abstract class -- `Server::begin(uint16_t)` never gets overridden).
    That combination (old tag + this Ethernet library) was not resolved;
    this patch is built and tested against `develop` only.


------------------------------------------------------------------------------
10. WHY THIS ISN'T JUST A PULL REQUEST YET
------------------------------------------------------------------------------

Several reasons someone should sanity-check before upstreaming this:

  - It's built against `develop`, which moves fast; by the time you read
    this the exact line numbers in the patch may not apply cleanly to the
    latest commit (though the underlying fixes/reasoning should still be
    correct and easy to re-apply by hand).
  - The config-persistence regression in section 6 is a separate, real
    upstream bug that someone should properly root-cause and report
    independently of this Ethernet work.
  - The static-IP bug in section 8 needs someone to actually dig into the
    RAK13800-W5100S library's source rather than work around it.
  - This was developed and tested on exactly one physical device. Full
    functionality was confirmed both with USB and PoE connected at the
    same time and powered solely by the RAK19018 PoE module over the
    Ethernet cable (no USB connected). It has not been tested across
    multiple units, power-cycle counts, or long-term uptime.

Test it, poke at the known issues, and feel free to open this as a PR
against meshtastic/firmware yourself -- just please mention the above so
reviewers go in with realistic expectations.


------------------------------------------------------------------------------
Pin mapping and library choice sourced from official RAKwireless datasheets
(RAK5005-O, RAK3312, RAK13800) and the existing, shipped `rak11310` and
`rak4631_eth_gw` Meshtastic board targets, which use the same RAK13800
module and the same RAK13800-W5100S Arduino library on different MCUs.
==============================================================================
