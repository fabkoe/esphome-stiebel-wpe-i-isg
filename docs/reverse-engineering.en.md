[🇩🇪 Deutsch (vollständig)](reverse-engineering.md) · **🇬🇧 English (overview)**

# Stiebel Eltron WPE-I 06 HKW 230 Premium – CAN bus reverse engineering (overview)

> **This is a condensed English overview.** The German
> [`reverse-engineering.md`](reverse-engineering.md) is the **exhaustive, continuously
> maintained source of truth** – it holds every confirmed index, all verification
> logs, the open TODOs and the Modbus/ISG-emulation planning. When in doubt, that
> file wins.

## Starting point

- Device: **Stiebel Eltron WPE-I 06 HKW 230 Premium** (brine-to-water heat pump,
  inverter, with cooling)
- No ISG present → no Modbus/web access available
- Approach: directly listening/requesting on the internal **CAN bus**
  (Elster/Kromschröder protocol)
- Hardware: Waveshare ESP32-S3-RS485-CAN with integrated CAN transceiver
  (TJA1051T/3, galvanically isolated), connected to terminal **X1.18 (CAN B –
  FET/ISG connection)** of the WPM4 controller (board details:
  [`hardware.md`](hardware.md))
- Software: ESPHome

Despite the newer inverter hardware the device still speaks the classic
**Elster/Kromschröder CAN protocol** used in older WPL/THZ series. Reference table
(community reverse engineering, not official):
`http://juerg5524.ch/data/ElsterTable.inc` (mirror: `github.com/andig/canprogs`).

## Protocol basics

### CAN parameters
- **Bit rate: 50 kbps** (not the usual 500 kbps from the automotive world!)
- Termination: not enabled (pure tap, not a bus end)

### Frame format (7 data bytes)

Short index (Elster index ≤ 0xFF):
```
[Cmd][00][Idx][Val_hi][Val_lo][00][00]
```
Extended index (Elster index > 0xFF), marked by byte 2 = 0xFA:
```
[Cmd][00][FA][Idx_hi][Idx_lo][Val_hi][Val_lo]
```
Values are usually **big-endian int16**, divided by **10** for decimals
(e.g. 277 → 27.7 °C). Heating-curve slope is ÷100.

### Request vs. response (important parser rule)

**Read requests always carry `0` in the value field** and must not be treated as
measurements. Distinguish via the lower nibble of the Cmd byte (byte 0):

| Cmd nibble (low) | Meaning | Examples |
|---|---|---|
| `1` | Read request (value field = 0, ignore!) | `0x41`, `0xA1`, `0x91`, `0x31` |
| `2` | Value report / module answer | `0x22`, `0x32`, `0xD2` |
| `0` | Broadcast **and set/write** (value field valid) | `0x20`, `0x80`, `0xC0` |

A real *write* uses nibble **`0`** (set); the target module then *reports* the new
value with nibble `2`.

### Known CAN arbitration IDs (sender/module)

| CAN ID | Module |
|---|---|
| 0x100 | WPM4-internal comms (very busy – do not use yourself!) |
| 0x180 | boiler/heat generator (a.k.a. manager path for some writes) |
| 0x201 | probably control panel / WPM broadcast |
| 0x401 | room remote-sensor module (FET) |
| 0x480 | manager |
| 0x500 / 0x700 | heating module (request) / peer (answer) – appear in pairs |
| 0x514 | IWS / refrigeration unit (compressor, flow/return sensor data) |
| 0x601 | mixer module |
| 0x680 | **PC / ComfortSoft channel** – reserved for external tools, unused here → safe for our own requests |

### "Not available" sentinel
`-32768` (0x8000) = official Stiebel Eltron value for "object not available"
(also confirmed in the ISG Modbus docs – so it holds across protocols).

### Addressing formula (header per target module)

The 2-byte header is derived **from the target CAN ID** (there is no fixed generic
prefix):
```
Header byte 1 = ((can_id >> 3) & 0xF0) | cmd_nibble   (cmd: 1 = read, 0 = write)
Header byte 2 =  can_id & 0x1F
```
Examples: module `0x601` → read `C1 01` / write `C0 01`; manager `0x180` → read
`31 00` / write `32 00`; whole system `0x480` → read `91 00`. Always **send over
`0x680`** (the free PC/ComfortSoft channel); the target lives only in the header.

## Confirmed values (reading)

| Parameter | Elster index | CAN ID(s) | Format |
|---|---|---|---|
| Outside temperature | `0x000C` | 0x180 / 0x100 | ÷10, °C |
| Hot-water actual temp | `0x000E` | 0x180/0x100/0x201 | ÷10, °C |
| Hot-water target temp | `0x0013` | 0x100/0x201 | ÷10, °C |
| Return temperature | `0x0016` | 0x514 | ÷10, °C |
| Flow temperature (HP) | `0xFDF3` | 0x700/0x500 | ÷10, °C |
| Room temperature (remote sensor) | `0x4EC7` | 0x401 | ÷10, °C |
| Room humidity (remote sensor) | `0x4EC8` | 0x401 | ÷10, % |
| Fault list, system (count) | `0x4F0B` | 0x100 | integer |
| Fault list, heat pump (count) | `0x4F0C` | 0x480 | integer |
| Operating mode | `0x4F1B` | 0x100/0x480 | 1–6 (see below) |
| Heating-curve slope | `0x4F2B` | 0x100/0x601 | ÷100 |
| Comfort temp, heating circuit | `0x4EB8` | 0x100/0x601 | ÷10, °C |
| Eco temp, heating circuit | `0x4EB9` | 0x100/0x601 | ÷10, °C |

Beyond these, the firmware reads the full **process data** (refrigerant circuit,
inverter, brine), **energy counters + COP** (12- and 13–24-month balance),
**runtimes and compressor starts** – see the German file for the complete index
list, scalings and per-value verification logs.

**Operating mode `0x4F1B`** encoding (1-indexed): 1 = standby, 2 = program,
3 = comfort, 4 = eco, 5 = hot water, 6 = emergency. **Never actively set/test
emergency mode (6)** – it switches on the electric booster heater.

## Writing (summary)

Writes are module-specific:
- **Mixer module `0x601`** (heating/cooling curve, comfort/eco, room influence,
  min temp, cooling values): `C0 01 FA <idx> <hi> <lo>`, read-back `C1 01 …`.
- **Manager module `0x180`** (operating mode `0x4F1B`, "cooling" switch `0x4F07`,
  hysteresis `0x4F00`): `32 00 FA <idx> …`.

`32 00 …` is **not** generic – pick the path per target module. Confirmed write
values (device-verified via log + display) include operating mode, heating-curve
slope, comfort/eco temperature, room influence, minimum temperature and the
cooling-circuit values; details, limits and open items in the German file.

## Safety

- Wrong write values can, per the manual, **damage the heat pump or the screed**
  (e.g. a too-high heating curve can starve the circuit's minimum flow).
- New write formats must be verified on the real device (log + display) before
  being trusted; new write entities get conservative min/max limits.
- **Emergency mode (6) is never set/tested actively.**

## Operating mode & read-only (firmware)

A staggered, switchable access mode (`select` "Betriebsmodus", global `g_mode`):
0 = listen only (no TX), 1 = read/poll, 2 = full access (writing). The poller is
gated by `g_mode >= 1`, every write by `g_mode >= 2`. Write entities also live in a
separate file (`wpe-i-writes.yaml`) so a build without it is physically read-only.

## Proven procedure for new values

1. Run a sniffer log (`esphome logs`), keep rough timestamps.
2. Change the value at the WPM display; photograph it (timestamps help).
3. Match frames in the log window to the shown value (usually `value × 10`,
   int16 big-endian).
4. If needed, actively poll the new index – always via CAN ID `0x680` first.
5. For writes: capture the natural set telegram the WPM sends, derive the exact
   byte format, only then add a write entity with sensible min/max limits.

---

For everything else – the complete confirmed index tables, all device-verification
logs, the WPM4 menu coverage, the open TODOs and the planned ISG-compatible
Modbus-TCP emulation – see the German
[`reverse-engineering.md`](reverse-engineering.md).
