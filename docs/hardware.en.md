[🇩🇪 Deutsch](hardware.md) · **🇬🇧 English**

# Hardware: Waveshare ESP32-S3-RS485-CAN

The board used in this project. This file bundles the device documentation; the
official schematic is linked at Waveshare (see sources), not mirrored in the repo
for copyright reasons.

## Sources (official)

- **Product page:** `https://www.waveshare.com/esp32-s3-rs485-can.htm` (SKU 32154)
- **Wiki (docs/demos/resources):** `https://www.waveshare.com/wiki/ESP32-S3-RS485-CAN`
- **Schematic:** `https://files.waveshare.com/wiki/ESP32-S3-RS485-CAN/ESP32-S3-RS485-CAN-Schematic.pdf`
- ESP32-S3 datasheet + Technical Reference Manual: via the wiki section
  "Resources → Datasheets" (Espressif).

## Key facts

- MCU **ESP32-S3** (WiFi/BLE), industrial control board, pin pitch **2.0 mm**
- **CAN transceiver: NXP `TJA1051T/3/1J`** (per schematic), **galvanically
  isolated** (power isolation via DC-DC `B0505LS-1W`, optocoupler isolation)
- CAN protection: TVS array `SM24CANB-02HTG`
- Onboard **120 Ω CAN termination**, switchable via jumper
- RS485 (isolated) also present – **not used in this project**

## Pin assignment (from wiki)

| Function | Signal | GPIO |
|---|---|---|
| CAN (TWAI) | TX | **GPIO15** |
| CAN (TWAI) | RX | **GPIO16** |
| RS485 (unused) | TXD | GPIO17 |
| RS485 (unused) | RXD | GPIO18 |
| RS485 (unused) | TX-Enable | GPIO21 |

The CAN pins **GPIO15/GPIO16** match the manifest configuration
(`esphome/wpe-i-manifest.yaml`) and the README.

> Board confirmed by the user as the one installed (2026-08-09). The earlier
> `SN65HVD230` claim in README/docs was wrong and corrected to `TJA1051T/3`.

## Connection to the heat pump

See README (section "Hardware"): terminal **X1.18** (CAN B, FET/ISG connection)
of the WPM4, H→CAN-H, L→CAN-L. **Leave the termination jumper open** – the ESP is
a tap on the bus, not a bus end. Bit rate **50 kbps**.

## Baud rates (wiki FAQ)

- CAN (TWAI): theoretically up to 1 Mbps freely in software (fixed **50 kbps** here).
- RS485/UART: theoretically up to 5 Mbps; due to isolation/protection ≈ 115200 bps
  is recommended in practice (irrelevant to this project).

## Flashing / bootloader (wiki FAQ)

If the COM port is missing or flashing fails: **hold BOOT, tap RESET at the same
time, release BOOT** (bootloader mode). Matches the usual ESP32 flashing workflow.
