[🇩🇪 Deutsch](README.md) · **🇬🇧 English**

# esphome-stiebel-wpe-i-isg – Stiebel Eltron WPE-I without ISG in Home Assistant

ESPHome firmware that connects a **Stiebel Eltron WPE-I 06 HKW 230 Premium**
(brine-to-water heat pump, inverter generation) to Home Assistant **without the
ISG gateway**, directly over its internal CAN bus (Elster/Kromschröder protocol) –
reading *and* partially writing, **fully local, no cloud**.

The **ISG** (Internet Service Gateway) is Stiebel's official, paid accessory for
Modbus/remote access. This project exposes the same system values **without an
ISG** using a ~€30 board. As a future stage, an **ISG-compatible Modbus-TCP
emulation** is planned (docs, TODO F) so that existing ISG/Modbus integrations can
use our interface unchanged.

Built from independent reverse engineering, because the WPE-I series is not yet
covered by existing projects such as
[OneESP32ToRuleThemAll](https://github.com/kr0ner/OneESP32ToRuleThemAll)
(WPL/THZ series).

## What it can do

**Reading** (~60 sensor entities, grouped as Everyday / Configuration / Diagnostic):

- **Temperatures:** outside, hot water actual/target, flow/return,
  room temperature & humidity (FET remote sensor)
- **Settings:** operating mode, heating curve (slope, comfort/eco temperature,
  room influence, minimum temperature), cooling (room target, cooling curve,
  start temperature, hysteresis, power, cooling type)
- **Live process data (refrigerant circuit):** evaporator, compressor-inlet,
  hot-gas, condenser, oil-sump temperature, low/high pressure, flow rate,
  inverter current/voltage, compressor actual/target speed
- **Heat source (brine):** flow/return, pressure, pump power
- **Energy & efficiency:** heat quantities and power consumption (compressor +
  booster heater, heating/hot water), yearly balance months 1–12 & 13–24, COP
- **Runtimes & starts:** compressor (heating/hot water), booster heater, passive
  cooling, compressor starts · plus fault-list counters

**Writing / control** (full build & operating mode "Full access" only):

- **Operating mode** as an HA dropdown (standby / program / comfort / eco / hot water)
- **Heating circuit:** heating-curve slope, comfort/eco temperature, room
  influence, minimum temperature
- **Cooling:** room target temperature, cooling-curve slope, start temperature,
  hysteresis, cooling circuit / cooling type / "cooling" switch

The write values were confirmed on one device via log + display; some are still
experimental – formats, limits and status are in
[`docs/reverse-engineering.md`](docs/reverse-engineering.md).

## How it works

The ESP32 sits as a silent tap on the WPM4 controller's internal CAN bus and
speaks the Elster/Kromschröder protocol (50 kbps, 7-byte frames):

1. **Listening:** a decoder splits each frame (index + value) and feeds the
   matching Home Assistant entity.
2. **Active polling:** values the WPM doesn't send on its own are polled every
   60 s over the free PC/ComfortSoft channel (CAN ID `0x680`) – collision-free.
3. **Writing:** control entities send a write frame + an immediate read-back so
   the new value shows up in HA right away – only in "Full access" mode.

All indices and scalings were reverse-engineered and are documented in
[`docs/reverse-engineering.md`](docs/reverse-engineering.md).

## Operating mode & read-only

Staggered, switchable access (HA selector **"Betriebsmodus"**, default **1**):

| Mode | Bus behaviour | Writing |
|---|---|---|
| **0 – Listen only** | pure sniffer, **no** TX from the ESP | no |
| **1 – Read (poll)** | actively polls the read values | no |
| **2 – Full access** | read + write | yes |

- **True read-only build:** the write entities live separately in
  `esphome/wpe-i-writes.yaml`. If that file isn't included, the firmware is
  **physically unable to write** (mode 2 then has no effect).
- **Debug logging** as an HA switch: enables the raw-frame dump in the log –
  normal operation stays quiet.

## Comparison: ISG vs. this project

| | Stiebel **ISG** (official) | **this project** |
|---|---|---|
| Cost | paid device | ~€30 board + DIY (wiring/flashing) |
| Home Assistant integration | via Modbus integration | **native** (ESPHome entities, instantly) |
| Reading (temp, HW, energy/COP) | yes | yes (verified, growing subset) |
| Writing/control | yes (documented registers) | yes (verified subset; rest experimental) |
| Live process data (compressor, inverter, refrigerant circuit, brine) | limited | **extensive** (more than ISG Modbus provides) |
| Cloud/remote access (ServiceWelt) | yes | no – **intentionally local-only** |
| Modbus-TCP compatibility | yes (native) | planned (**ISG emulation**, TODO F) |
| Enforceable read-only operation | – | yes (read-only build + operating mode) |
| Official support / warranty | yes (manufacturer) | no (community, **at your own risk**) |
| Commissioning | plug-and-play | wiring + flashing (adopt flow for updates) |

> The comparison is based on the public ISG Modbus/software documents (linked
> externally in [`docs/reverse-engineering.md`](docs/reverse-engineering.md)) and
> made to the best of our knowledge; individual ISG rows may vary by ISG variant/
> firmware. The ISG remains the easier choice for plug-and-play, cloud remote
> access and manufacturer support.

## Hardware

- Waveshare **ESP32-S3-RS485-CAN** (industrial board) with an integrated,
  galvanically isolated CAN transceiver (TJA1051T/3) – details:
  [`docs/hardware.md`](docs/hardware.md)
- Connected to terminal **X1.18** (CAN B, FET/ISG connection) of the
  WPM4 controller – H → CAN-H, L → CAN-L
- Leave the termination jumper **open** (tap, not a bus end)
- CAN pins: TX = GPIO15, RX = GPIO16 · bit rate: **50 kbps**

⚠️ **De-energize the system on all poles before working on the WPM.**
After disconnection, residual voltage may persist for up to 5 minutes
(inverter capacitors).

## Installation

The firmware is built as an **ESPHome package**: the reusable core lives in
`esphome/wpe-i-package.yaml`, the network/device data stays separate. There are
two ways.

### A) Adopt as an ESPHome package (recommended)

Once the repo is public, a flashed WPE-I firmware announces itself in the
ESPHome/Home Assistant dashboard for adoption ("Adopt"). Alternatively, reference
the package directly – your own mini config:

```yaml
substitutions:
  name: wpe-i-heatpump
packages:
  wpe_i: github://fabkoe/esphome-stiebel-wpe-i-isg/esphome/wpe-i-package.yaml@v1.0.0
wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
```

The ESPHome `secrets` need `wifi_ssid`, `wifi_password` **and** `ota_password`.
Board/pins are preset for the Waveshare ESP32-S3-RS485-CAN; for other boards
override the `substitutions` (`board`, `can_tx_pin`, `can_rx_pin`).

This package alone is **read-only** (no write entities). To enable writing,
additionally include `wpe-i-writes.yaml`:

```yaml
packages:
  wpe_i:        github://fabkoe/esphome-stiebel-wpe-i-isg/esphome/wpe-i-package.yaml@v1.0.0
  wpe_i_writes: github://fabkoe/esphome-stiebel-wpe-i-isg/esphome/wpe-i-writes.yaml@v1.0.0
```

### B) Build/develop locally (esptool)

1. Clone the repo, copy `esphome/secrets.yaml.example` to
   `esphome/secrets.yaml` and fill in Wi-Fi + OTA data
2. Compile and flash `esphome/wpe-i-manifest.yaml` – it includes
   `wpe-i-package.yaml` **and** `wpe-i-writes.yaml` (full build). For a local
   read-only build, omit the second `!include` line.
3. The device appears in Home Assistant with all sensors + control entities

## Documentation

The full reverse-engineering state (protocol basics, all confirmed Elster
indices, CAN-ID mappings, the WPM4 display menu tree, open TODOs and the proven
procedure for new values) is in
[`docs/reverse-engineering.md`](docs/reverse-engineering.md).

## Project structure

```
esphome/
  wpe-i-package.yaml        # core: sensors + poller + operating mode (read-only, package target)
  wpe-i-writes.yaml         # write entities (full build only; omit = read-only)
  wpe-i-manifest.yaml       # thin local flash config (includes package + writes + Wi-Fi)
  secrets.yaml.example      # template for Wi-Fi + OTA credentials
docs/
  reverse-engineering.md    # protocol docs, value tables, TODOs
  wpm4-menue.md             # WPM4 display menu tree (coverage legend)
  hardware.md               # Waveshare ESP32-S3-RS485-CAN board (pinout, links)
archive/
  wpe-can-sniffer-v1.yaml   # historic: first sniffer (MQTT-based)
  wpe-can-sniffer-v2.yaml   # historic: sniffer via ESPHome logs
LICENSE · NOTICE            # Apache-2.0 + trademark/attribution notice
CONTRIBUTING.md             # contributing + DCO sign-off
```

## Important (please read briefly)

Private hobby/reverse-engineering project, **not affiliated with Stiebel Eltron** –
use at your own risk, no warranty.

- **Reading is harmless, writing is not.** Wrong write values can, per the manual,
  damage the heat pump or the screed – that's why read-only is the default and you
  enable writing deliberately.
- **Power off:** only work on the WPM/CAN bus with the system de-energized on all
  poles (residual voltage up to ~5 min).
- **One device:** everything was tested on exactly one WPE-I 06 HKW 230 (WPM4
  449-10) – on a different series/firmware it may differ.

## License & contributing

Apache-2.0 (see [`LICENSE`](LICENSE)). Contributions welcome – sign off your
commits via DCO (`git commit -s`), details in [`CONTRIBUTING.md`](CONTRIBUTING.md).

## Acknowledgements / references

- Elster protocol table: [juerg5524.ch](http://juerg5524.ch/data/ElsterTable.inc)
  (mirror: [andig/canprogs](https://github.com/andig/canprogs))
- [andig/goelster](https://github.com/andig/goelster) – Elster CAN in Go, a good
  reference for index scalings
- [kr0ner/OneESP32ToRuleThemAll](https://github.com/kr0ner/OneESP32ToRuleThemAll)
  – reference project for the WPL/THZ series
- [bullitt186/ha-stiebel-control](https://github.com/bullitt186/ha-stiebel-control)
  – labelled frame logger, helped track down open values
