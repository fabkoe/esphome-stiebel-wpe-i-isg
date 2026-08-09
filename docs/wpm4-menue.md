# Menüstruktur des WPM4-Displays (WPE-I 06 HKW 230 Premium)

> Ausgelagert aus [`reverse-engineering.md`](reverse-engineering.md); die
> CAN-Indizes, Skalierungen und Verifikations-Logs stehen weiterhin dort.

Vom Gerät abgetippte, rechtschreibkorrigierte Menüstruktur (Stand 31.07.2026,
Ansicht „Experte"). Angezeigte Werte im Original sind Momentaufnahmen vom
Aufschreiben und nicht zwingend aktuell; hier weggelassen. `->` am Gerät =
veränderbar. **Menü noch unvollständig** (Programm, Diagnose-Unterpunkte,
Betriebsart-Auswahl nur teilweise erfasst).

**Coverage-Legende:** ✅ Sensor-Entity im Manifest · ✏️ schreibbare Entity
vorhanden · 📄 Index bekannt/dokumentiert, aber Entity fehlt noch (Index in
Klammern) · ❓ offen (kein Index).

```
HAUPTMENÜ
├── INFO
│   ├── ANLAGE
│   │   ├── RAUMTEMPERATUR / FET1
│   │   │   ├── Isttemperatur              ✅
│   │   │   ├── Solltemperatur             ❓
│   │   │   ├── Raumfeuchte                ✅
│   │   │   └── Taupunkttemperatur         ❓  ← relevant fürs Kühlen (Estrich/Kondensat)
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
│   │   │   └── Anlagenfrost               ❓  (= Frostschutz-Einstellung)
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
│   │   │   ├── VD Heizen Tag              📄 (Tageswert)
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
│       │   ├── Heizen 1–24 h             ❓ (Tageswert)
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
│           └── übrige 1–24 h / 13–24 M   ❓
├── DIAGNOSE
│   ├── Status Anlage
│   ├── Status Wärmepumpe
│   ├── Analyse Wärmepumpe   (Überhitzung Sauggas Soll/Ist, Verdichterdrehzahlgrenze)
│   └── System → Busteilnehmer (SW-Versionen: WPM4, FES, FET1, MFG, WP1)
├── PROGRAMM
├── EINSTELLUNGEN
│   ├── Ansicht -> Experte
│   ├── ALLGEMEIN            (Zeit/Datum, Sommerzeit, Sprache, Kontrast, Helligkeit, Touch)
│   ├── FAVORITEN
│   ├── HEIZEN
│   │   ├── HEIZKREIS 1
│   │   │   ├── Komforttemperatur ->      ✏️
│   │   │   ├── ECO-Temperatur ->         ✏️
│   │   │   ├── Minimaltemperatur ->      ✏️
│   │   │   ├── Steigung Heizkurve ->     ✏️
│   │   │   ├── Ansicht Heizkurve         (nur Anzeige)
│   │   │   └── Raumeinfluss ->           ✏️
│   │   ├── GRUNDEINSTELLUNG
│   │   │   ├── Sommerbetrieb             ❓
│   │   │   ├── Vorlaufanteil Heizkreis ->❓
│   │   │   ├── Maximale Rücklauftemp ->  ❓
│   │   │   ├── Maximale Vorlauftemp ->   ❓
│   │   │   ├── Festwertbetrieb ->        ❓
│   │   │   └── Frostschutz ->            📄 (= Anlagenfrost)
│   │   ├── Pumpenzyklen
│   │   └── ELEKTRISCHE NACHERWÄRMUNG     (Bivalenztemp/Einsatzgrenze HZG, Stufen, Verzögerung)  ❓
│   ├── WARMWASSER
│   │   ├── WARMWASSERTEMPERATUREN        (Komfort-/ECO-Temperatur ->)  ❓
│   │   ├── GRUNDEINSTELLUNG              (Betrieb, Hysterese, Stufen, Lernfkt., WW-Leistung, Max-Vorlauf, Antilegionellen)  ❓
│   │   ├── ELEKTRISCHE NACHERWÄRMUNG     (Bivalenztemp/Einsatzgrenze WW ->)  ❓
│   │   └── ZIRKULATION                   (Anforderung ->)  ❓
│   └── KÜHLEN
│       ├── KÜHLKREIS 1
│       │   ├── Kühlkreis ->              ✏️  (0x4F08)
│       │   ├── Raumsolltemperatur ->     ✏️  (0x4F04)
│       │   ├── Kühlart (Gebläse/Fläche)  ✏️  (0x4F05)
│       │   ├── Steigung Kühlkurve ->     ✏️  (0x4FB9)
│       │   └── Starttemperatur ->        ✏️  (0x4FBE)
│       └── GRUNDEINSTELLUNG
│           ├── Leistung ->               📄 (0x7A40 lesend; Schreibmodul offen)
│           └── Hysterese Vorlauftemp ->  ✏️  (0x4F00)
└── INBETRIEBNAHME
```

**Beobachtungen zum Abgleich:**
- **Nur-Lese-Zuwächse (31.07.2026) ins Manifest + am Gerät bestätigt**
  (`logs/readsensors-verify.log`): Laufzeit (6), Starts (1), Prozessdaten-Vorlauf
  `0x01D6`, Energiebilanz-Rest (13–24-M-Wärme/Strom + Effizienz
  `0x501E/0x5022/0x503A`). Alle 14 Sensoren publishen digit-genau gegen die
  bekannten Display-Werte (z. B. VD Heizen 5125 h, Starts 8605, Wärme Heizen
  13–24M 7502 kWh, Effizienz 7,56). Sensoren `entity_category diagnostic`, Poll
  über 0x680 (IWS `A1 14` bzw. Manager `91 00`).
- **Taupunkttemperatur (FET1)** noch ohne Index – der für die Kühl-Sicherheit
  (Kondensat/Estrich) interessanteste offene Wert.
- **KÜHLEN-Schalter `0x4F07`** ist in dieser Abschrift nicht eindeutig zu
  verorten; „Kühlkreis -> EIN" unter Kühlkreis 1 ist `0x4F08`. Display-Name von
  `0x4F07` bleibt offen.
- **Betriebsart-Auswahl** taucht in der Abschrift nicht auf (vermutlich unter
  PROGRAMM) – Entity existiert dennoch (`0x4F1B`).
