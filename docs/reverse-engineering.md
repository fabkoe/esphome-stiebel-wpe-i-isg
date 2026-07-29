# Stiebel Eltron WPE-I 06 HKW 230 Premium – CAN-Bus Reverse Engineering

Stand: 21.07.2026

## Ausgangslage

- Gerät: **Stiebel Eltron WPE-I 06 HKW 230 Premium** (Sole-Wasser-Wärmepumpe, Inverter, Kühlfunktion)
- Kein ISG vorhanden → kein Modbus/Web-Zugriff möglich
- Ansatz: direktes Mithören/Anfragen auf dem internen **CAN-Bus** (Elster/Kromschröder-Protokoll)
- Hardware: Waveshare ESP32-S3 Industrial Board mit integriertem CAN-Transceiver (SN65HVD230), angeschlossen an Klemme **X1.18 (CAN B – FET/ISG-Anschluss)** des WPM4-Reglers
- Software: ESPHome, Firmware `wpe-i-manifest.yaml`

## Protokoll-Grundlagen

Das Gerät spricht trotz neuerer Inverter-Hardware weiterhin das klassische **Elster/Kromschröder-CAN-Protokoll**, wie es auch in älteren WPL/THZ-Baureihen verwendet wird. Referenztabelle (Community-Reverse-Engineering, nicht offiziell):
`http://juerg5524.ch/data/ElsterTable.inc` (Mirror: `github.com/andig/canprogs`)

### CAN-Bus Parameter
- **Bitrate: 50 kbps** (nicht die üblichen 500 kbps aus der Automotive-Welt!)
- Terminierung: nicht aktiviert (reiner Abgriff/Tap, kein Busende)

### Frame-Aufbau (7 Byte Nutzdaten)

Kurzer Index (Elster-Index ≤ 0xFF):
```
[Cmd][00][Idx][Wert_hi][Wert_lo][00][00]
```

Erweiterter Index (Elster-Index > 0xFF), erkennbar an Byte 2 = 0xFA:
```
[Cmd][00][FA][Idx_hi][Idx_lo][Wert_hi][Wert_lo]
```

Werte sind i. d. R. **big-endian int16**, durch **10 geteilt** für Nachkommastellen (z. B. 277 → 27,7 °C).

### Anfrage vs. Antwort unterscheiden (wichtiger Parser-Fix, 21.07.2026)

**Lese-Anfragen enthalten im Wertefeld immer `0`** und dürfen nicht als Messwert interpretiert werden. Unterscheidung über das untere Halbbyte des Cmd-Bytes (Byte 0):

| Cmd-Halbbyte (unten) | Bedeutung | Beispiele |
|---|---|---|
| `1` | Lese-Anfrage (Wertefeld = 0, ignorieren!) | `0x41`, `0xA1`, `0x91`, `0x31` |
| `2` | Antwort / Schreibbefehl | `0x22`, `0x32` |
| `0` | Broadcast | `0x20`, `0x80`, `0xE0`, `0xA2` |

Ohne diesen Filter sprangen Sensoren sporadisch auf 0, sobald die WPM intern denselben Index angefragt hat – das war auch die Ursache für die zwischenzeitlich beobachteten "Unbekannt (0)"-Anzeigen in Home Assistant. Der vermeintlich "abgelehnte" Komforttemperatur-Schreibversuch vom 21.07. war daher vermutlich falsch interpretiert: Das empfangene `val=0` war eine mitgelesene *Anfrage*, keine Ablehnung.

### Bekannte CAN-Arbitration-IDs (Absender/Modultyp)

| CAN-ID | Modul |
|---|---|
| 0x100 | WPM4-interne Kommunikation (sehr belegt, nicht selbst benutzen!) |
| 0x180 | Kessel/Wärmeerzeuger |
| 0x201 | vermutlich Bedienteil/WPM-Broadcast |
| 0x401 | Raumfernfühler-Modul (FET) |
| 0x480 | Manager |
| 0x500 / 0x700 | Heizmodul (Anfrage) / Fremdgerät (Antwort) – treten paarweise auf |
| 0x514 | IWS / Kälteaggregat (Sensordaten Verdichter, Rücklauf/Vorlauf) |
| 0x601 | Mischermodul |
| 0x680 | **PC / ComfortSoft-Kanal** – für externe Tools reserviert, bei uns unbenutzt → sicher für eigene Anfragen |

### Sentinel-Wert für "nicht verfügbar"

`-32768` (0x8000) = offizieller Stiebel-Eltron-Wert für "Objekt nicht verfügbar" (bestätigt auch in der offiziellen Modbus-Doku des ISG – gilt also protokollübergreifend).

## Bestätigte Werte (Stand 19.07.2026)

| # | Parameter | Elster-Index | CAN-ID(s) | Format | Menüpfad am Display |
|---|---|---|---|---|---|
| 1 | Außentemperatur | `0x000C` | 0x180 / 0x100 | /10, °C | Info→Anlage→Heizung |
| 2 | Warmwasser-Isttemperatur | `0x000E` | 0x180/0x100/0x201 | /10, °C | Info→Anlage→Warmwasser |
| 3 | Warmwasser-Solltemperatur | `0x0013` | 0x100/0x201 | /10, °C | Einstellungen→Warmwasser |
| 4 | Rücklauftemperatur | `0x0016` | 0x514 | /10, °C | Info→Wärmepumpe→Prozessdaten |
| 5 | Vorlauftemperatur WP | `0xFDF3` | 0x700/0x500 | /10, °C | Info→Anlage→Heizung |
| 6 | Raumtemperatur (Fernfühler) | `0x4EC7` | 0x401 | /10, °C | – (Fernfühlermodul) |
| 7 | Raumluftfeuchte (Fernfühler) | `0x4EC8` | 0x401 | /10, % | – (Fernfühlermodul) |
| 8 | Meldungsliste Anlage (Anzahl) | `0x4F0B` | 0x100 | Ganzzahl | Diagnose→Meldungsliste |
| 9 | Meldungsliste Wärmepumpe (Anzahl) | `0x4F0C` | 0x480 | Ganzzahl | Diagnose→Meldungsliste |
| 10 | Betriebsart | `0x4F1B` | 0x100/0x480 | 1-6, siehe unten | Hauptmenü→Betriebsart |
| 11 | Steigung Heizkurve | `0x4F2B` | 0x100/0x601 | /100 | Einstellungen→Heizen→Heizkreis |
| 12 | Komforttemperatur Heizkreis | `0x4EB8` | 0x100/0x601 | /10, °C | Einstellungen→Heizen→Heizkreis |
| 13 | Eco-Temperatur Heizkreis | `0x4EB9` | 0x100/0x601 | /10, °C | Einstellungen→Heizen→Heizkreis |

### Aktives Polling (bestätigt funktionierend)

Werte 3 und 4 werden von der Anlage nur bei Menü-Aufruf/Wertänderung gesendet. Wir fragen sie deshalb selbst aktiv ab, alle 60s, über den PC/ComfortSoft-Kanal (0x680):

```
Anfrage Rücklauf (Index 0x16):   A1 14 16 00 00 00 00
Anfrage WW-Soll (Index 0x13):    41 01 13 00 00 00 00
```

Wichtig: Der Anfrage-Byte-Header (`41 01` vs. `A1 14`) scheint je nach angefragtem Modul/Parameter unterschiedlich zu sein – nicht generisch für jeden Index gültig. Muss ggf. pro neuem Wert einzeln herausgefunden werden (siehe Vorgehen unten).

## Vollständige Menüstruktur des WPM4-Displays

```
HAUPTMENÜ
├── INFO ▸
│   ├── ANLAGE ▸
│   │   ├── RAUMTEMPERATUR    (Ist/Soll/Feuchte/Taupunkt)
│   │   ├── HEIZUNG           (HK1 Ist/Soll, Vorlauf WP/NHZ, Rücklauf, Druck, Volumenstrom)
│   │   ├── WARMWASSER        (Ist/Soll/Volumenstrom)
│   │   ├── KÜHLEN
│   │   └── ELEKTRISCHE NACHERWÄRMUNG
│   ├── WÄRMEPUMPE ▸
│   │   ├── PROZESSDATEN      (3 Unterseiten: Kältekreis-Temperaturen, Drücke, Inverter-Strom/Spannung/Drehzahl)
│   │   ├── WÄRMEMENGE        (VD Heizen/WW Tag+Summe, NHZ Heizen Summe)
│   │   ├── LEISTUNGSAUFNAHME (VD Heizen/WW Tag+Summe)
│   │   ├── LAUFZEIT          (u.a. Passivkühlung)
│   │   └── STARTS
│   └── ENERGIEBILANZ ▸
│       └── GESAMTSYSTEM ▸
│           ├── WÄRMEMENGE
│           ├── STROMVERBRAUCH
│           └── EFFIZIENZ
│
└── DIAGNOSE ▸
    ├── STATUS ANLAGE
    ├── STATUS WÄRMEPUMPE
    ├── ANALYSE WÄRMEPUMPE    (Überhitzung Sauggas Soll/Ist, Verdichterdrehzahlgrenze)
    ├── SYSTEM → BUSTEILNEHMER (Softwareversionen: WPM4, FES, FET1, MFG, WP1)
    ├── INTERNE BERECHNUNG
    ├── MELDUNGSLISTE
    ├── RELAISTEST ANLAGE
    └── RELAISTEST WÄRMEPUMPE
```

## Noch offene Werte (Kandidaten vorhanden, nicht final bestätigt)

| Screen | Angezeigte Werte | CAN-Kandidat | Status |
|---|---|---|---|
| Prozessdaten S.1 | Verdampfer 21,6°C / Verdichtereintritt 28,0°C / Heißgas 27,6°C | – | offen |
| Prozessdaten S.2 | Ölsumpf 37,8°C / ND 9,52bar / HD 9,41bar / Volumenstrom | – | offen |
| Prozessdaten S.3 | Strom/Spannung Inverter, Ist/Solldrehzahl Verdichter | – | vermutlich WPE-I-spezifisch, nicht in Standard-Elster-Tabelle |
| Wärmemenge (WP) | VD Heizen Summe 15,2 MWh etc. | `0x4F9A`/`0x4EFB` auf 0x514 | Wert-Bestätigung ausstehend |
| Leistungsaufnahme | VD Heizen Summe 2,033 MWh etc. | `0x0805` auf 0x514 | Wert-Bestätigung ausstehend |
| Laufzeit/Starts | Passivkühlung 4000h | `0x025A`/`0x0259` | Wert-Bestätigung ausstehend |
| Mischermodul-Wert | – | `0x4EB4` auf 0x601, ~19,1-19,2°C | vermutlich Vorlauf HK2 |
| Status-Flags | – | `0x0074` auf 0x480 (val=1), `0x4EB3` auf 0x401/0x100 (val=1) | Bedeutung unklar (Betriebsstatus?) |

## Bewährtes Vorgehen für neue Werte

1. **Sniffer-Log laufen lassen** (ESPHome-Logs live via `esphome logs` oder HA-Add-on-Log-Fenster)
2. **Am WPM-Display gezielt navigieren**, dabei Uhrzeit(en) grob im Kopf behalten
3. **Fotos vom Display machen** – iPhone-Dateinamen enthalten UTC-Zeitstempel (`YYYYMMDD_HHMMSSmmm_iOS.heic`), lokal = UTC+2 (Sommerzeit)
4. Frames im Log-Zeitfenster der Fotos mit den angezeigten Werten abgleichen (Format meist `Wert × 10` als int16 big-endian)
5. Bei Bedarf: aktives Polling für den neuen Index testen (immer zuerst über CAN-ID `0x680`, nie über belegte IDs wie `0x100`)

## Nächste Ziele: Heizkurve, Kühlkurve, Betriebsart (inkl. Schreibzugriff)

Zusätzlich zum reinen Auslesen soll hier künftig auch **verstellt** werden können (nicht nur gelesen). Das erfordert einen zusätzlichen Vorsichts-Schritt gegenüber den bisherigen Werten.

### Betriebsart ✅ bestätigt

**Index `0x4F1B`, CAN-ID 0x100 und 0x480** (zweifach bestätigt, gleicher Wert auf beiden IDs beim Umschalten Komfort→Eco→Komfort).

Kodierung ist **1-indiziert** (nicht 0-indiziert wie ursprünglich vermutet):

| Wert | Betriebsart | Status |
|---|---|---|
| 1 | Bereitschaftsbetrieb | ✅ bestätigt |
| 2 | Programmbetrieb | vermutet (Musterfortsetzung) |
| 3 | Komfortbetrieb | ✅ bestätigt (mehrfach) |
| 4 | Eco-Betrieb | ✅ bestätigt |
| 5 | Warmwasserbetrieb | ✅ bestätigt |
| 6 | Not-Betrieb | vermutet (Musterfortsetzung, bewusst nicht aktiv getestet – Notfallmodus aktiviert elektrische Zusatzheizung) |

Ursprüngliche Kandidaten `0x0176` (BETRIEBS_STATUS) und `0x0062` (WAERMEPUMPEN_STATUS) waren beide falsch – tauchten im Log nicht auf.

**Nebenbefund:** `0x0074` auf CAN-ID 0x480 (val=1) ändert sich NICHT mit der Betriebsart (war in Komfort und Eco identisch) – kein Betriebsart-Indikator, Bedeutung weiterhin offen.

### Schreiben (Betriebsart setzen)

Schreib-Format gefunden (KNX-User-Forum, Autor der Elster-Tabelle selbst), am Beispiel eines anderen Parameters (`PROGRAMMSCHALTER`, Index `0x0112`):

```
CAN-ID 0x680:  32 00 FA 01 12 02 00
```
- Byte0 `0x32`: unteres Halbbyte `2` = **Schreiben** (Lesen war bisher immer `1`)
- Byte2-4 `FA 01 12`: erweiterter Index `0x0112`
- Byte5-6 `02 00`: Wert (dort little-endian dokumentiert – **Achtung, Endianness ist pro Elster-Index individuell**, nicht global einheitlich; unsere bisherigen Lese-Antworten waren durchweg big-endian)

**Offene Unsicherheiten für unseren Fall (Index `0x4F1B`):**
- Ob Byte0 `0x32` auch für unseren Index/unser Gerät passt, oder ob das erste Halbbyte eine Art Ziel-Adressierung ist
- Byte-Reihenfolge des Werts (big- vs. little-endian) für diesen spezifischen Index

**Sicherer Test implementiert:** Button "Betriebsart Test-Rueckschreiben" im Manifest – schreibt bewusst nur den aktuell bekannten Wert zurück (keine echte Änderung), um zu prüfen, ob die WPM das Telegramm überhaupt akzeptiert (sichtbar an erneutem Broadcast auf `idx=0x4F1B` im Log, ohne Fehler). Erst danach sollte ein echter Schreib-Service mit variablem Zielwert gebaut werden.

### ✅ Schreiben erfolgreich verifiziert (19.07.2026)

Echter Wechseltest mit festen Zielwerten (unabhängig vom aktuellen Zustand):

```
Gesendet: Ziel=Komfort (3)  →  821ms später bestätigt: idx=0x4F1B val=3  ✓ Display wechselte sichtbar
Gesendet: Ziel=Eco (4)      →  829ms später bestätigt: idx=0x4F1B val=4  ✓ Display wechselte sichtbar
```

Keine CAN-Fehler, konsistentes Timing bei beiden Tests. **Schreib-Format bestätigt:**
```
CAN-ID 0x680:  32 00 FA 4F 1B 00 XX     (XX = Zielwert 1-6)
```

Im Manifest umgesetzt als `select`-Entity **"Betriebsart setzen"** (Home Assistant Dropdown) mit den 5 unbedenklichen Optionen (Not-Betrieb bewusst ausgeschlossen). Dropdown wird beim Lesen automatisch mit dem tatsächlichen Zustand synchronisiert.

### Heizkurve / Kühlkurve

Keine öffentlich dokumentierten Indizes gefunden – wie erwartet per Foto+Log-Abgleich selbst ermittelt.

Menüpfade (laut Betriebs-/Installationsanleitung):
- `Einstellungen → Heizen → Heizkreis` → `STEIGUNG HEIZKURVE`, `NIVEAU`/`KOMFORT TEMPERATUR`
- `Einstellungen → Kühlen` → entsprechende Kühlkurven-Parameter (nur bei HKW-Modellen mit Kühlfunktion sichtbar)

**Warnhinweis aus der Betriebsanleitung:** *"Falsche Einstellungen können zur Beschädigung der Wärmepumpe oder des Estrichs führen."* Eine zu hoch eingestellte Heizkurve kann dazu führen, dass Thermostat-/Zonenventile schließen und der Mindestvolumenstrom im Heizkreis unterschritten wird.

#### Steigung Heizkurve ✅ Lesen bestätigt (19.07.2026)

```
0,40 → 0,45  →  id=0x100/0x601 idx=0x4F2B val=40 → val=45
0,45 → 0,40  →  val=40
```

**Index `0x4F2B`, CAN-ID 0x100 und 0x601** (Mischermodul – die Heizkurve steuert die Mischerregelung, macht Sinn). Skalierung `/100` (nicht `/10` wie bei Temperaturen).

**Schreiben:** noch nicht separat verifiziert, aber im Manifest bereits als `number`-Entity **"Steigung Heizkurve setzen"** umgesetzt (gleiches Schreib-Format wie bei Betriebsart, `32 00 FA 4F 2B ...`). Grenzen bewusst konservativ auf 0,1–2,5 gesetzt. **Ersten Schreibtest bewusst klein und mit Log+Display-Kontrolle durchführen**, bevor der Wert als vertrauenswürdig gilt.

#### Komforttemperatur / Eco-Temperatur Heizkreis ✅ bestätigt (19.07.2026)

| Index | Wert | Bedeutung |
|---|---|---|
| `0x4EB8` | 200 → 20,0°C | Komforttemperatur (Ziel-Raumtemperatur im Komfortbetrieb) |
| `0x4EB9` | 161 → 16,1°C | Eco-Temperatur (Ziel-Raumtemperatur im Eco-Betrieb) |

Beide `/10`-skaliert, gleiche CAN-IDs wie Steigung Heizkurve (0x100/0x601). **Nur lesend** im Manifest.

**Schreibversuch am 21.07.2026 – Bewertung revidiert:** Ursprünglich als "abgelehnt" interpretiert (Anlage schien mit `val=0` zu antworten). Nach dem Parser-Fix (siehe Protokoll-Grundlagen) stellte sich heraus: Das `val=0` war eine mitgelesene *Lese-Anfrage* der WPM, keine Ablehnung – unser Parser hat damals Anfragen und Antworten nicht unterschieden. Ob der Schreibversuch tatsächlich abgelehnt wurde oder schlicht ignoriert, ist damit wieder offen. Die tatsächliche Einstellung am Gerät blieb in jedem Fall unverändert (Display-Kontrolle bestätigt). Schreib-Entities bleiben vorerst entfernt; ein erneuter, sauberer Test (mit korrektem Parser) ist jetzt aber deutlich aussichtsreicher.

#### Noch offene Bonus-Kandidaten

| Index | Wert | Vermutung |
|---|---|---|
| `0x4EA7` | -28672 | unklar, ungewöhnliches Format |
| `0x4EA4` | 0 | unklar |

#### Kühlkurve

Noch nicht getestet – gleiches Vorgehen wie bei Steigung Heizkurve, im Menü `Einstellungen → Kühlen`.

### Geplantes Vorgehen (Lesen) – bereits erfolgreich angewendet für Betriebsart & Steigung Heizkurve

1. Log läuft, aktuellen Wert am Display fotografieren
2. Wert leicht ändern, Uhrzeit merken
3. Zurückstellen, Uhrzeit erneut merken
4. Log + Fotos abgleichen → liefert zuverlässig den Lese-Index

### Geplantes Vorgehen (Schreiben) – zusätzliche Vorsicht nötig

Im Unterschied zu reinen Leseanfragen (bisher nur `A1`/`41`-Header über CAN 0x680 verwendet) braucht Schreiben einen echten **Set-Befehl**, dessen genaues Byte-Format wir noch nicht kennen. Geplantes, vorsichtiges Vorgehen:

1. **Erst Lese-Index bestätigen** (siehe oben)
2. **Natürliches Set-Telegramm mitschneiden**, während der Wert manuell am Display geändert wird (wie beim WW-Soll-Test 45→40°C) – daraus das exakte Byte-Format ableiten, das die WPM selbst beim Setzen nutzt
3. **Erst danach** einen Schreib-Service in ESPHome ergänzen (z. B. als `number`/`select`-Entity in Home Assistant), mit sinnvollen Min/Max-Grenzen im Code gegen Fehleingaben
4. Betriebsart-Wechsel testweise nur kurz (Sekunden), nicht dauerhaft auf "Bereitschaft" stehen lassen – pausiert sonst Heizen/Kühlen/Warmwasser

## TODOs

### Als Nächstes (priorisiert)
- [ ] **Parser-Fix verifizieren (21.07.)** – neu flashen, prüfen dass keine Sensoren mehr sporadisch auf 0/Unbekannt springen (Anfrage-Frames mit Cmd-Halbbyte 1 werden jetzt gefiltert)
- [ ] **Komforttemperatur-Schreibtest wiederholen** – mit korrektem Parser; der alte "Fehlschlag" war vermutlich eine Fehlinterpretation (mitgelesene Anfrage statt Ablehnung)
- [ ] **Aktive Betriebsart-Abfrage reparieren** – `41 01 FA 4F 1B` wird aktuell mit `-32768` auf 0x201 beantwortet (falsches Zielmodul); Betriebsart-Wert kommt derzeit nur über Broadcasts bei echten Wechseln. Anderes Anfrage-Header-Muster nötig
- [x] **Betriebsart auslesen** – Index `0x4F1B` bestätigt (Bereitschaft=1, Komfort=3, Eco=4, Warmwasser=5 alle verifiziert)
  - [ ] Programmbetrieb (erwartet: 2) verifizieren
  - [ ] Not-Betrieb (erwartet: 6) – bewusst nicht aktiv testen (Notfallmodus)
- [x] **Betriebsart-Schreiben getestet und bestätigt** – select-Entity in HA verfügbar, Komfort/Eco per CAN-Schreibbefehl erfolgreich verifiziert (Display wechselt sichtbar)
- [x] **Steigung Heizkurve auslesen** – Index `0x4F2B` bestätigt (0,40→0,45→0,40 verifiziert)
  - [ ] Steigung Heizkurve schreiben verifizieren (number-Entity vorhanden, Testschreiben mit Log+Display-Kontrolle noch ausstehend)
  - [ ] Schreibformat für Komforttemperatur/Eco-Temperatur (0x4EB8/0x4EB9) herausfinden – erster Versuch mit `32 00 FA ...` schlug fehl (Anlage antwortete val=0), Gerät blieb unverändert. Evtl. anderer Byte0-Header oder andere Zieladressierung nötig, ähnlich wie bei WW-Soll (`41 01` statt `A1 14`) herausgefunden
  - [ ] Niveau/Komfort-Temperatur der Heizkurve auslesen (eigener Index, noch offen)
  - [ ] Bonus-Kandidaten zuordnen: `0x4EA7`, `0x4EA4` auf 0x601 (Komforttemperatur `0x4EB8` und Eco-Temperatur `0x4EB9` bereits bestätigt)
- [ ] **Kühlkurve auslesen** – analog unter Einstellungen→Kühlen
- [ ] **Set-Telegramm-Format für Kühlkurve ableiten** – Schreib-Format `32 00 FA ...` bereits für Betriebsart bestätigt, vermutlich generisch übertragbar
- [ ] **Schreib-Service in ESPHome ergänzen** (number/select-Entities) – erst nachdem Set-Format bestätigt ist, mit Min/Max-Grenzen im Code

### Prozessdaten / Energie (aus vorheriger Session offen)
- [ ] Prozessdaten S.1 bestätigen: Verdampfer, Verdichtereintritt, Heißgas
- [ ] Prozessdaten S.2 bestätigen: Ölsumpf, Druck ND/HD, Volumenstrom
- [ ] Prozessdaten S.3 bestätigen: Strom/Spannung Inverter, Ist-/Solldrehzahl Verdichter (vermutlich WPE-I-spezifisch, ggf. nicht in Standard-Elster-Tabelle)
- [ ] Wärmemenge (Wärmepumpe) bestätigen – Kandidaten `0x4F9A`/`0x4EFB` auf 0x514
- [ ] Leistungsaufnahme bestätigen – Kandidat `0x0805` auf 0x514
- [ ] Laufzeit/Starts bestätigen – Kandidaten `0x025A`/`0x0259`
- [ ] Mischermodul-Wert (`0x4EB4` auf 0x601, ~19,1–19,2°C) einer konkreten Bedeutung zuordnen (vermutlich Vorlauf HK2)
- [ ] Status-Flags klären: `0x0074` auf 0x480, `0x4EB3` auf 0x401/0x100 (beide val=1 gesehen – Betriebsstatus?)

### Sonstiges / Housekeeping
- [ ] Meldungsliste (aktuell 14 Einträge) inhaltlich prüfen – harmlose Altmeldungen oder aktive Störungen?
- [ ] Aktives Polling-Intervall (aktuell 60s) ggf. anpassen/pro Wert individualisieren
- [ ] Manifest ggf. auf Pull Request Richtung `OneESP32ToRuleThemAll` vorbereiten, sobald Wertesatz stabil ist (WPE-I-Manifest existiert dort noch nicht)

## Hilfreiche externe Referenzen



- **Elster-Protokolltabelle:** `http://juerg5524.ch/data/ElsterTable.inc`
- **OneESP32ToRuleThemAll** (Referenzprojekt, andere WP-Baureihen): `github.com/kr0ner/OneESP32ToRuleThemAll`
- **Stiebel Eltron Modbus-Doku** (andere Adressierung, aber gleiche Sentinel-Werte/Kategorien): offizielle ISG-Modbus-Anleitung
- **pail23/stiebel_eltron_isg_component** (Modbus, braucht ISG – nicht direkt nutzbar, aber gute Namens-Roadmap)
