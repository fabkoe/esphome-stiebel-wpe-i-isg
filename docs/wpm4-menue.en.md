[🇩🇪 Deutsch](wpm4-menue.md) · **🇬🇧 English**

# WPM4 display menu structure (WPE-I 06 HKW 230 Premium)

> Split out from [`reverse-engineering.md`](reverse-engineering.md); the CAN
> indices, scalings and verification logs still live there.

Menu structure transcribed from the device and spell-corrected (as of
2026-07-31, "Expert" view). The values originally shown are snapshots from the
time of writing and not necessarily current; omitted here. `->` on the device =
editable. **Menu still incomplete** (program, diagnostic sub-items, operating-mode
selection only partially captured).

> Note: the tree keeps the **original German on-device labels** (that's what the
> display shows), so you can still navigate on the unit. Headings, legend,
> annotations and the notes below are in English.

**Coverage legend:** ✅ sensor entity in the manifest · ✏️ writable entity
present · 📄 index known/documented but entity still missing (index in
parentheses) · ❓ open (no index).

```
HAUPTMENÜ
├── INFO
│   ├── ANLAGE
│   │   ├── RAUMTEMPERATUR / FET1
│   │   │   ├── Isttemperatur              ✅
│   │   │   ├── Solltemperatur             ❓
│   │   │   ├── Raumfeuchte                ✅
│   │   │   └── Taupunkttemperatur         ❓  ← relevant for cooling (screed/condensate)
│   │   ├── HEIZUNG
│   │   │   ├── Außentemperatur            ✅
│   │   │   ├── Isttemperatur HK 1         ❓
│   │   │   ├── Solltemperatur HK 1        ❓
│   │   │   ├── Vorlaufisttemperatur WP    ✅
│   │   │   ├── Vorlaufisttemperatur NHZ   ❓
│   │   │   ├── Rücklaufisttemperatur WE   ✅
│   │   │   ├── Festwertsolltemperatur WE  ❓
│   │   │   ├── Heizungsdruck              ❓
│   │   │   ├── Volumenstrom               ❓
│   │   │   └── Anlagenfrost               ❓  (= frost-protection setting)
│   │   ├── WARMWASSER
│   │   │   ├── Isttemperatur              ✅
│   │   │   ├── Solltemperatur             ✅
│   │   │   └── Volumenstrom               ❓
│   │   ├── KÜHLEN
│   │   │   ├── Isttemperatur              ❓
│   │   │   ├── Solltemperatur             ❓
│   │   │   ├── Isttemperatur KK 1         ❓
│   │   │   └── Solltemperatur KK 1        ❓
│   │   └── ELEKTRISCHE NACHERWÄRMUNG
│   │       ├── Bivalenztemperatur HZG     ❓
│   │       ├── Einsatzgrenze HZG          ❓
│   │       ├── Bivalenztemperatur WW      ❓
│   │       └── Einsatzgrenze WW           ❓
│   ├── WÄRMEPUMPE
│   │   ├── PROZESSDATEN
│   │   │   ├── Rücklauftemperatur             ✅
│   │   │   ├── Vorlauftemperatur              ✅ (0x01D6)
│   │   │   ├── Verdampfertemperatur           ✅
│   │   │   ├── Verdichtereintrittstemperatur  ✅
│   │   │   ├── Heißgastemperatur              ✅
│   │   │   ├── Verflüssigertemperatur         ✅
│   │   │   ├── Ölsumpftemperatur              ✅
│   │   │   ├── Druck Niederdruck              ✅
│   │   │   ├── Druck Hochdruck                ✅
│   │   │   ├── WP Wasser Volumenstrom         ✅
│   │   │   ├── Strom Inverter                 ✅
│   │   │   ├── Spannung Inverter              ✅
│   │   │   ├── Istdrehzahl Verdichter         ✅
│   │   │   ├── Solldrehzahl Verdichter        ✅
│   │   │   ├── Rücklauftemperatur Wärmequelle ✅
│   │   │   ├── Vorlaufisttemperatur Wärmeq.   ✅
│   │   │   ├── Wärmequellendruck              ✅
│   │   │   └── Leistung Wärmequellenpumpe     ✅
│   │   ├── WÄRMEMENGE
│   │   │   ├── VD Heizen Tag              📄 (daily value)
│   │   │   ├── VD Heizen Summe            ✅
│   │   │   ├── VD Warmwasser Tag          📄
│   │   │   ├── VD Warmwasser Summe        ✅
│   │   │   ├── NHZ Heizen Summe           ✅
│   │   │   └── NHZ Warmwasser Summe       ✅
│   │   ├── LEISTUNGSAUFNAHME
│   │   │   ├── VD Heizen Tag              📄
│   │   │   ├── VD Heizen Summe            ✅
│   │   │   ├── VD Warmwasser Tag          📄
│   │   │   └── VD Warmwasser Summe        ✅
│   │   ├── LAUFZEIT
│   │   │   ├── VD Heizen                  ✅ (0x4EFB)
│   │   │   ├── VD Warmwasser              ✅ (0x4EFD)
│   │   │   ├── NHZ 1                      ✅ (0x0259)
│   │   │   ├── NHZ 2                      ✅ (0x025A)
│   │   │   ├── NHZ 1/2                    ✅ (0x0805)
│   │   │   └── Passivkühlung              ✅ (0x4F9A)
│   │   └── STARTS
│   │       └── Verdichter                ✅ (0x4EF0/0x4EF1)
│   └── ENERGIEBILANZ / GESAMTSYSTEM
│       ├── WÄRMEMENGE
│       │   ├── Heizen 1–24 h             ❓ (daily value)
│       │   ├── Heizen 1–12 M             ✅
│       │   ├── Heizen 13–24 M            ✅ (0x502C)
│       │   ├── Warmwasser 1–24 h         ❓
│       │   ├── Warmwasser 1–12 M         ✅
│       │   └── Warmwasser 13–24 M        ✅ (0x5030)
│       ├── STROMVERBRAUCH
│       │   ├── Heizen 1–24 h             ❓
│       │   ├── Heizen 1–12 M             ✅
│       │   ├── Heizen 13–24 M            ✅ (0x5032)
│       │   ├── Warmwasser 1–24 h         ❓
│       │   ├── Warmwasser 1–12 M         ✅
│       │   └── Warmwasser 13–24 M        ✅ (0x5036)
│       └── EFFIZIENZ
│           ├── Heizen 1–12 M             ✅ (0x501E)
│           ├── Warmwasser 1–12 M         ✅ (0x5022)
│           ├── Warmwasser 13–24 M        ✅ (0x503A)
│           └── übrige 1–24 h / 13–24 M   ❓  (remaining periods)
├── DIAGNOSE
│   ├── Status Anlage
│   ├── Status Wärmepumpe
│   ├── Analyse Wärmepumpe   (suction-gas superheat target/actual, compressor speed limit)
│   └── System → Busteilnehmer (SW versions: WPM4, FES, FET1, MFG, WP1)
├── PROGRAMM
├── EINSTELLUNGEN
│   ├── Ansicht -> Experte   (view -> Expert)
│   ├── ALLGEMEIN            (time/date, DST, language, contrast, brightness, touch)
│   ├── FAVORITEN
│   ├── HEIZEN
│   │   ├── HEIZKREIS 1
│   │   │   ├── Komforttemperatur ->      ✏️
│   │   │   ├── ECO-Temperatur ->         ✏️
│   │   │   ├── Minimaltemperatur ->      ✏️
│   │   │   ├── Steigung Heizkurve ->     ✏️
│   │   │   ├── Ansicht Heizkurve         (display only)
│   │   │   └── Raumeinfluss ->           ✏️
│   │   ├── GRUNDEINSTELLUNG
│   │   │   ├── Sommerbetrieb             ❓
│   │   │   ├── Vorlaufanteil Heizkreis ->❓
│   │   │   ├── Maximale Rücklauftemp ->  ❓
│   │   │   ├── Maximale Vorlauftemp ->   ❓
│   │   │   ├── Festwertbetrieb ->        ❓
│   │   │   └── Frostschutz ->            📄 (= Anlagenfrost / system frost)
│   │   ├── Pumpenzyklen
│   │   └── ELEKTRISCHE NACHERWÄRMUNG     (bivalence temp/operating limit HZG, stages, delay)  ❓
│   ├── WARMWASSER
│   │   ├── WARMWASSERTEMPERATUREN        (comfort/ECO temperature ->)  ❓
│   │   ├── GRUNDEINSTELLUNG              (mode, hysteresis, stages, learning fn, HW power, max flow, anti-legionella)  ❓
│   │   ├── ELEKTRISCHE NACHERWÄRMUNG     (bivalence temp/operating limit WW ->)  ❓
│   │   └── ZIRKULATION                   (request ->)  ❓
│   └── KÜHLEN
│       ├── KÜHLKREIS 1
│       │   ├── Kühlkreis ->              ✏️  (0x4F08)
│       │   ├── Raumsolltemperatur ->     ✏️  (0x4F04)
│       │   ├── Kühlart (Gebläse/Fläche)  ✏️  (0x4F05; fan/surface)
│       │   ├── Steigung Kühlkurve ->     ✏️  (0x4FB9)
│       │   └── Starttemperatur ->        ✏️  (0x4FBE)
│       └── GRUNDEINSTELLUNG
│           ├── Leistung ->               📄 (0x7A40 read; write module open)
│           └── Hysterese Vorlauftemp ->  ✏️  (0x4F00)
└── INBETRIEBNAHME
```

**Observations for cross-checking:**
- **Read-only additions (2026-07-31) added to the manifest + confirmed on the
  device** (`logs/readsensors-verify.log`): runtimes (6), starts (1),
  process-data flow `0x01D6`, remaining energy balance (13–24-month heat/power +
  efficiency `0x501E/0x5022/0x503A`). All 14 sensors publish digit-exact against
  the known display values (e.g. VD heating 5125 h, starts 8605, heat heating
  13–24M 7502 kWh, efficiency 7.56). Sensors `entity_category diagnostic`, polled
  over 0x680 (IWS `A1 14` or manager `91 00`).
- **Dew-point temperature (FET1)** still has no index – the most interesting open
  value for cooling safety (condensate/screed).
- **The "cooling" switch `0x4F07`** cannot be unambiguously located in this
  transcript; "Kühlkreis -> EIN" under Kühlkreis 1 is `0x4F08`. The display name
  of `0x4F07` remains open.
- **Operating-mode selection** does not appear in the transcript (probably under
  PROGRAMM) – the entity exists nonetheless (`0x4F1B`).
