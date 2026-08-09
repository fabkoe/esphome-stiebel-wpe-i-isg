# Hardware: Waveshare ESP32-S3-RS485-CAN

Das im Projekt verwendete Board. Diese Datei bündelt die Geräte-Doku; das
offizielle Schaltbild liegt lokal unter
[`manuals/Waveshare_ESP32-S3-RS485-CAN_Schematic.pdf`](manuals/Waveshare_ESP32-S3-RS485-CAN_Schematic.pdf).

## Quellen (offiziell)

- **Produktseite:** `https://www.waveshare.com/esp32-s3-rs485-can.htm` (SKU 32154)
- **Wiki (Doku/Demos/Ressourcen):** `https://www.waveshare.com/wiki/ESP32-S3-RS485-CAN`
- **Schaltbild:** `https://files.waveshare.com/wiki/ESP32-S3-RS485-CAN/ESP32-S3-RS485-CAN-Schematic.pdf`
  (lokal gesichert, s.o.)
- ESP32-S3 Datenblatt + Technical Reference Manual: über die Wiki-Sektion
  „Resources → Datasheets" (Espressif).

## Eckdaten

- MCU **ESP32-S3** (WiFi/BLE), industrielles Steuerboard, Pin-Raster **2,0 mm**
- **CAN-Transceiver: NXP `TJA1051T/3/1J`** (laut Schaltbild), **galvanisch
  getrennt** (Power Isolation über DC-DC `B0505LS-1W`, Optokoppler-Isolation)
- CAN-Schutzbeschaltung: TVS-Array `SM24CANB-02HTG`
- Onboard **120 Ω CAN-Terminierung**, per Jumper zu-/abschaltbar
- RS485 (isoliert) zusätzlich vorhanden – **im Projekt nicht genutzt**

## Pinbelegung (aus Wiki)

| Funktion | Signal | GPIO |
|---|---|---|
| CAN (TWAI) | TX | **GPIO15** |
| CAN (TWAI) | RX | **GPIO16** |
| RS485 (ungenutzt) | TXD | GPIO17 |
| RS485 (ungenutzt) | RXD | GPIO18 |
| RS485 (ungenutzt) | TX-Enable | GPIO21 |

Die CAN-Pins **GPIO15/GPIO16** decken sich mit der Manifest-Konfiguration
(`esphome/wpe-i-manifest.yaml`) und dem README.

> Board vom Nutzer als das verbaute bestätigt (09.08.2026). Frühere Angabe
> `SN65HVD230` in README/Doku war falsch und wurde auf `TJA1051T/3` korrigiert.

## Anschluss an die Wärmepumpe

Siehe README (Abschnitt „Hardware"): Klemme **X1.18** (CAN B, FET/ISG-Anschluss)
des WPM4, H→CAN-H, L→CAN-L. **Terminierungs-Jumper offen lassen** – der ESP
hängt als Abgriff am Bus, nicht als Busende. Bitrate **50 kbps**.

## Baudraten (Wiki-FAQ)

- CAN (TWAI): theoretisch bis 1 Mbps frei per Software (hier fest **50 kbps**).
- RS485/UART: theoretisch bis 5 Mbps, wegen Isolation/Schutz praktisch
  ≈ 115200 bps empfohlen (im Projekt irrelevant).

## Flashen / Bootloader (Wiki-FAQ)

Wenn der COM-Port fehlt oder das Flashen fehlschlägt: **BOOT gedrückt halten,
gleichzeitig RESET tippen, BOOT loslassen** (Bootloader-Modus). Deckt sich mit
dem Workflow in CLAUDE.md.
