# Classic ESP32 (ESP32-D0WD-V3) — headless CSI node

This is a port of the main [ESP32-S3/C6/C5 firmware](README.md) to a plain,
original ESP32. Target chip: ESP32-D0WD-V3, 4 MB flash, no PSRAM assumed.

The firmware builds clean for ESP-IDF v5.4 and runs on real hardware.
CSI capture, UDP reporting, and presence/motion detection are verified.
Long-term stability, breathing rate accuracy, heart rate, and pose remain unverified.
Treat those as experimental.

## What is different from the S3/C6 build

This build is **headless**: no display, no touch, no WASM programmable
sensing. Those features need either an AMOLED panel or PSRAM, and a plain
4 MB classic ESP32 board has neither. Everything else — CSI capture, the
adaptive controller, OTA, the mesh/sync plane, mmWave UART probing — is the
same code as the S3 build.

Classic ESP32 predates WiFi 6 (802.11ax) and is 2.4GHz-only.
Your WiFi network must have a 2.4GHz band (most dual-band routers do).
The firmware already has two CSI setup paths in `main/csi_collector.c`:
a WiFi 6 path for C6/C5 and a plain path for pre-HE chips.
ESP32-S3 already uses that plain path, and classic ESP32 uses it the same way.

## How to build

Set up the ESP-IDF v5.4 toolchain, then from this directory:

```bash
export IDF_TOOLS_PATH=$HOME/.x/tools/espressif
. $HOME/.x/tools/esp-idf/export.sh

cd firmware/esp32-csi-node
idf.py -B build-esp32 -D IDF_TARGET=esp32 -D SDKCONFIG=build-esp32/sdkconfig \
  -D SDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.defaults.esp32" build
```

This uses a separate build directory (`build-esp32/`) so it does not clash
with an S3 or C6 build directory in the same checkout. `build-esp32/` is
gitignored; do not commit it.

Check the binary fits before flashing:

```bash
idf.py -B build-esp32 size
```

## How to flash

Flashing is the user's own step on this port — do not run esptool against a
real board as part of a build task.

Use esptool 5.x. Confirm the port name and the attached chip first:

```bash
python -m esptool --port /dev/cu.usbserial-0001 chip-id
```

Then flash all four images at the offsets `idf.py build` printed (these are
the standard classic-ESP32 offsets — the bootloader loads from `0x1000` here,
not `0x0` like S3/C6, because the ESP32 classic mask ROM looks there):

```bash
python -m esptool --chip esp32 --port /dev/cu.usbserial-0001 --baud 460800 \
  write_flash --flash_mode dio --flash_size 4MB \
  0x1000  build-esp32/bootloader/bootloader.bin \
  0x8000  build-esp32/partition_table/partition-table.bin \
  0xf000  build-esp32/ota_data_initial.bin \
  0x20000 build-esp32/esp32-csi-node.bin
```

Always use the exact offsets printed by your own build (`build-esp32/flash_args`),
in case a future change to the partition table moves them.

## How to provision WiFi

First install dependencies:

```bash
pip install esptool esp-idf-nvs-partition-gen
```

Then use `provision.py`. It writes WiFi credentials to the board over serial NVS.
It is target-generic — just pass the right `--chip` value (or leave it as `auto`,
the default in esptool 5+):

```bash
printf '%s\n' 'YourPass' > ~/.wifi-pass && chmod 600 ~/.wifi-pass
python firmware/esp32-csi-node/provision.py --port /dev/cu.usbserial-0001 \
  --chip esp32 --ssid "YourSSID" --password-file ~/.wifi-pass \
  --target-ip 192.168.1.20
```

On failure, `provision.py` writes a temporary `nvs_config.csv` file to the current
directory containing your WiFi password. Delete it:

```bash
rm -f nvs_config.csv
```

## What is verified and what is not

**Verified:** Firmware builds clean for IDF_TARGET=esp32 with ESP-IDF v5.4 (937,810 bytes in a 1,900,544-byte app partition).
Flashed to ESP32-D0WD-V3 (4 MB, port `/dev/cu.usbserial-0001`).
Board joined 2.4GHz WiFi (channel 6), CSI callbacks ~40 packets/s, 636 UDP packets received on Mac in 25s.
Presence/motion numbers appear in the log.

**Not verified:** Accuracy of presence/motion detection, breathing rate, heart rate, pose.
Long-term stability on real hardware. S3 builds correctly; C6 target not tested.

**By design, not supported:**
- **No display.** `CONFIG_DISPLAY_ENABLE` is off — AMOLED panels need PSRAM this board does not have.
- **No WASM programmable sensing.** `CONFIG_WASM_ENABLE` is off — WASM3 module storage needs PSRAM.
- **No onboard WS2812 LED.** Classic ESP32 modules have no standard addressable LED pin and no GPIO above 39.
