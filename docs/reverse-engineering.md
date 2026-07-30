# Stiebel Eltron WPE-I 06 HKW 230 Premium – CAN-Bus Reverse Engineering

Stand: 29.07.2026

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
| `2` | Wert-Meldung / Antwort des Moduls | `0x22`, `0x32`, `0xD2` |
| `0` | Broadcast **und Setzen/Schreiben** (Wertefeld gültig) | `0x20`, `0x80`, `0xE0`, `0xA2`, `0xC0` |

**Präzisierung (30.07.2026, Heizkurven-Schreib-Sniff):** Ein echter
*Schreibbefehl* nutzt Halbbyte **`0`** (Setzen), nicht `2`. Das WPM setzte die
Heizkurve mit `C0 01 FA 4F 2B <wert>`; das Zielmodul *meldete* den neuen Wert
danach mit Halbbyte `2` (`22 00 FA 4F 2B <wert>`). Die frühere Zuordnung
„`2` = Schreibbefehl" war also ungenau – `2` ist die Melde-/Antwortrichtung,
`0` die Schreib-/Broadcast-Richtung. (Der bisher für Betriebsart funktionierende
`0x32`-Write mit Halbbyte `2` bleibt ein noch nicht final erklärter Sonderfall –
für das Mischermodul war eindeutig `C0`/Halbbyte `0` nötig.)

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
| Prozessdaten S.1–S.4 | Kältekreis, Drücke, Inverter, Wärmequelle | 0x514-Block | ✅ **komplett bestätigt** – siehe Abschnitt „Prozessdaten" |
| Wärmemenge / Leistungsaufnahme (WP) | VD/NHZ Heizen+WW Summen | `0x091a–0x0931` | ✅ bestätigt – **Achtung: Display-Werte kommen von 0x514 (IWS), nicht 0x500!** Siehe „Energiewerte / Zähler" |
| Laufzeit/Starts | VD/NHZ/Passivkühlung/Starts | 0x514-Block | ✅ **komplett bestätigt** – siehe Abschnitt „Laufzeiten & Starts" |
| Mischermodul-Wert | – | `0x4EB4` auf 0x601, ~19,1-19,2°C | vermutlich Vorlauf HK2 |
| Status-Flags | – | `0x0074` = **EVU_SPERRE_AKTIV** (geklärt); `0x4EB3` auf 0x401/0x100 (val=1) | 0x0074 geklärt, 0x4EB3 offen |

## Bewährtes Vorgehen für neue Werte

1. **Sniffer-Log laufen lassen** (ESPHome-Logs live via `esphome logs` oder HA-Add-on-Log-Fenster)
2. **Am WPM-Display gezielt navigieren**, dabei Uhrzeit(en) grob im Kopf behalten
3. **Fotos vom Display machen** – iPhone-Dateinamen enthalten UTC-Zeitstempel (`YYYYMMDD_HHMMSSmmm_iOS.heic`), lokal = UTC+2 (Sommerzeit)
4. Frames im Log-Zeitfenster der Fotos mit den angezeigten Werten abgleichen (Format meist `Wert × 10` als int16 big-endian)
5. Bei Bedarf: aktives Polling für den neuen Index testen (immer zuerst über CAN-ID `0x680`, nie über belegte IDs wie `0x100`)
6. **Schreib-Telegramme werden jetzt mitgeloggt** (seit 29.07.2026): die drei Schreib-Aktionen (Betriebsart-Testbutton, Betriebsart-Select, Heizkurve-Number) geben ihr gesendetes Frame als `TX id=0x680 raw=..` aus. So lässt sich das selbst gesendete Telegramm direkt mit dem vom Display gesnifften vergleichen. Die ~40 Poll-Sends loggen bewusst KEINE Rohbytes (Log-Spam; ihre Antworten werden ohnehin geloggt).

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

**Nebenbefund (geklärt 29.07.2026):** `0x0074` auf CAN-ID 0x480 ändert sich NICHT mit der Betriebsart – es ist **EVU_SPERRE_AKTIV** (Standard-Elster-Index, vom Sniffer als `on` gelesen; val=1 = Sperre aktiv). Kein Betriebsart-Indikator. Ggf. per Display gegenprüfbar.

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

> **Nachtrag (30.07.2026): Ziel-Adressierung bestätigt.** Byte0/Byte1 kodieren
> tatsächlich das Zielmodul: `byte0 = ((Ziel>>3)&0xF0) | Halbbyte`, `byte1 =
> Ziel & 0x1F`. Für Betriebsart (0x480) hat der `0x32`-Write empirisch
> funktioniert; für die Heizkurve (Mischermodul 0x601) war das der Grund des
> Fehlschlags – dort ist `C0 01` nötig (siehe Heizkurven-Schreib-Sniff und die
> präzisierte Cmd-Halbbyte-Tabelle oben). Werte waren bei allen bestätigten
> Indizes big-endian.

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

#### Bonus-Kandidaten ✅ zugeordnet (30.07.2026, `logs/zusäzlicheparameter-write-test.log`)

Per Display-Sniff im HEIZKREIS-1-Menü eindeutig identifiziert (beide auf
Modul 0x601, Schreiben mit `C0 01 FA <idx>`):

| Index | Bedeutung | Skalierung | Gerätebereich (F8/F9) | Besonderheit |
|---|---|---|---|---|
| `0x4EA7` | **MINIMAL TEMPERATUR (HK)** | ÷10 → °C | 10,0–30,0 °C | „Aus" = Sonderwert `0x9000` (−28672) |
| `0x4EA4` | **RAUMEINFLUSS (HK)** | ×1 → % | 0–100 % | Gerät rastet auf 5er-Schritte |

Display-Frames: `C0 01 FA 4E A7 00 64` (=10,0 °C), `C0 01 FA 4E A7 90 00`
(=„Aus"), `C0 01 FA 4E A4 00 0A` (=10 %). Modul bestätigt jeweils mit
`22 00 FA 4E A7/A4 <wert>`. **Nebenerkenntnis:** Beim Öffnen eines Edit-
Screens sendet das Modul die Grenzen als `F8`=Min / `F9`=Max desselben Index
(z. B. `22 00 F8 4E A7 00 64` = Min 10,0 °C, `22 00 F9 4E A7 01 2C` = Max
30,0 °C) – nützlicher genereller RE-Trick zum Ablesen von Wertebereichen.

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

## PoC: Framework `bullitt186/ha-stiebel-control` (29.07.2026)

Getestet, ob das Fremd-Framework (ESPHome, gleiches Elster-Protokoll,
Zielbaureihen WPL/WPF) mit der WPE-I funktioniert. Firmware v2.1.1 auf
die eigene Hardware geflasht, **Bitrate von 20 auf 50 kbps** geändert
(einzige zwingende Anpassung), am Bus mitgelesen.

**Ergebnis: Fundament kompatibel.** Der generische Decoder liest die
WPE-I bei 50 kbps korrekt – **display-verifiziert**: `AUSSENTEMP 32.4 °C`,
`SPEICHERISTTEMP 38.2 °C` (WW-Ist), `EVU_SPERRE_AKTIV on`, Uhrzeit. Damit
sind 50 kbps + Protokoll + Frame-Format erneut bestätigt.

**Aber nicht plug-and-play – zwei Lücken:**

1. **Kein Anfrage/Antwort-Filter (Framework-Bug, alle Modelle betroffen).**
   Frames wie `OTHER (0x00): AUSSENTEMP: 0.0` = mitgelesene Lese-Anfragen
   (Wertefeld 0), die das Framework nicht von Antworten unterscheidet →
   Werte flattern auf 0. Genau der Fix, den unser Manifest über das
   Cmd-unteres-Halbbyte (`1`=Anfrage) schon hat. Fundstelle:
   `ha-stiebel-control.h`, `processCanMessage()` (prüft `msg[0]` nie).
2. **WPE-I-Indizes fehlen.** Nur **4 von 13** bestätigten Indizes sind in
   der `ElsterTable.h` (Außentemp, WW-Ist, WW-Soll, Rücklauf). Der ganze
   **`0x4Exx`/`0x4Fxx`-Block (8 Werte)** – Betriebsart, Heizkurve,
   Komfort/Eco-Temp, FET-Raumtemp/-feuchte, Meldungslisten – **fehlt**.
   Zusätzlich: `0xFDF3` (WP-Vorlauf) ist als `STATUS_MULTIFUNKTIONSAUSGANG`
   **falsch belegt** (unskaliert), und die Betriebsart wird als
   `PROGRAMMSCHALTER 0x0112` gepollt statt über unser `0x4F1B`.

**Draft:** `../ha-stiebel-control-poc/esphome/ha-stiebel-control/signal_requests_wpei.h`
(nach `wpf10`-Muster, ungetestet) inkl. Vorschlag für die
ElsterTable-Ergänzungen.

**Aufwand:** reiner Code ~3–4 h (8 ElsterTable-Einträge, `0xFDF3` lösen,
Anfrage-Filter, `wpei.yaml`, optional CanMember 0x401 für FET). Der
Zeitfaktor ist die Geräteverifikation der neuen Indizes im Framework-Pfad.

**Entscheidung / Strategie:**
- **Produktivbasis bleibt dieses `wpe-i-manifest.yaml`** (Kontrolle,
  verifizierte Schreibformate, Anfrage-Filter, WPE-I-Indizes vorhanden).
- Framework = **RE-Werkzeug** (beschrifteter Sniffer + ElsterTable-Lookup
  für offene Werte) und **Community-Backup**, kein Ersatz.
- Beitragswege (gestaffelt): (1) **Anfrage/Antwort-Bugfix als PR** –
  klein, modellübergreifend, hohe Annahmechance. (2) **WPE-I-Modell als
  PR** erst nach Geräteverifikation (v. a. `0xFDF3` + FET-Adressierung).

## Energiewerte / Zähler ✅ bestätigt (29.07.2026)

Aufgespürt mit dem beschrifteten Sniffer (`ha-stiebel-control`), der diesen
Block bereits pollt. Zuordnung **display-verifiziert** über zwei unabhängige
Wege (Magnitude + COP-Gegencheck aus `Info → Energiebilanz → Gesamtsystem`,
siehe unten). Die alten Kandidaten (`0x4F9A`/`0x0805` auf 0x514) sind damit
**verworfen**.

**Modul: Heizmodul, Antwort auf CAN-ID `0x500`** (paarweise mit 0x700).
**⚠️ Nachträgliche Korrektur:** dieselben Indizes existieren auch auf der
**IWS (0x514)** mit **abweichenden Werten – die IWS entspricht dem Display**
(inkl. NHZ-Zählern). Siehe Korrektur-Abschnitt weiter unten.
Erweiterte Indizes (`0xFA`-Frames). Jeder Zähler ist auf **mehrere Register
gesplittet** (Stiebel-Standard):

- Summen: `…_SUM_MWH` (ganze MWh) **+** `…_SUM_KWH` (0–999 kWh, Nachkomma)
  → realer Wert `kWh = SUM_MWH × 1000 + SUM_KWH`
- Tageswerte: `…_TAG_KWH` (ganze kWh) **+** `…_TAG_WH` (0–999 Wh)

`2WE` = „zweiter Wärmeerzeuger" = die elektrische **NHZ**-Nacherwärmung.

| Zweig | MWh-Index (Summe) | kWh-Index (Summe) | TAG (kWh / Wh) |
|---|---|---|---|
| Wärmeertrag Heizen (VD) | `0x0931` | `0x0930` | `0x092f` / `0x092e` |
| Wärmeertrag WW (VD) | `0x092d` | `0x092c` | `0x092b` / `0x092a` |
| Wärmeertrag NHZ Heizen (2WE) | `0x0929` | `0x0928` | `0x0927` / `0x0926` |
| Wärmeertrag NHZ WW (2WE) | `0x0925` | `0x0924` | `0x0923` / `0x0922` |
| El. Aufnahme Heizen (VD) | `0x0921` | `0x0920` | `0x091f` / `0x091e` |
| El. Aufnahme WW (VD) | `0x091d` | `0x091c` | `0x091b` / `0x091a` |

**Live gelesene Ist-Werte (18:28–18:30, nur MWh-Register):** WAERMEERTRAG_HEIZ
= 14, _WW = 9, EL_AUFNAHME_HEIZ = 2, _WW = 2 MWh. TAG-Burst (bei Screen-Aufruf
broadcastet): WW-Wärme heute 6,966 kWh, WW-Strom heute 1,255 kWh → Tages-COP ~5,5.

**Verifikation (Display, `Info → Energiebilanz → Gesamtsystem`, 29.07.2026):**
Der Screen ist in sich konsistent (WW 13–24 M: 4,00 MWh Wärme ÷ 4,18 COP =
0,957 MWh Strom = abgelesene 0,956 MWh ✓). Die all-time-Summen des Sniffers
liegen erwartungsgemäß knapp über den 24-Monats-Fenstern (Heizen 14 ≥ 13,68;
WW 9 ≥ 8,28; El-Heiz 2 ≥ 1,84; El-WW 2 ≥ 1,95). COP Heizen 14/2 = 7,0 (Display
7,3–7,6), WW 9/2 = 4,5 (Display 4,2–5,5). Passt.

**Bonus-Screen noch offen:** `Info → Energiebilanz → Gesamtsystem` selbst
(Gesamtwärmemenge/-stromverbrauch/-effizienz je 1–12 M / 13–24 M) sind
**eigene** Indizes (Gesamtsystem inkl. NHZ, nach Zeitfenster) – noch nicht
zugeordnet, wären aber schöne HA-Sensoren. Kandidaten: `et_double_val`/
`et_triple_val`-Einträge in der ElsterTable.

**✅ Im Produktiv-Manifest umgesetzt und gerätebestätigt (29.07.2026):**
Sechs kombinierte Energiesensoren (`SUM_MWH`×1000 + `SUM_KWH`) als HA-Entities
(`device_class energy`, `total_increasing`). Live gelesen nach OTA-Flash:

| Zähler | MWh-Reg | kWh-Reg | kombiniert |
|---|---|---|---|
| Wärmemenge Heizen VD | 14 | 616 | 14 616 kWh |
| Wärmemenge WW VD | 9 | 386 | 9 386 kWh |
| Wärmemenge Heizen/WW NHZ (2WE) | 0 | 0 | 0 kWh (NHZ nie genutzt) |
| Stromaufnahme Heizen VD | 2 | 240 | 2 240 kWh |
| Stromaufnahme WW VD | 2 | 294 | 2 294 kWh |

**Request-Header gelöst:** Heizmodul-Indizes (0x500) über unseren 0x680-Kanal
mit `A1 00 FA <idx_hi> <idx_lo> 00 00` abfragen. `A1 00` = `generate_read_id(0x500)`
(unteres Halbbyte `1` = Lese-Anfrage). Antwort kommt auf `id=0x500`. Verifiziert,
weil der Sniffer identisch auf 0x680 sendet und die WPE-I antwortet.

**Nebenbefund:** Die **Tageszähler** (`…_TAG_KWH`/`…_TAG_WH`, z. B. `0x092A/0x092B`,
`0x091A/0x091B`) werden auf **CAN-ID 0x514** (IWS) beantwortet/gebroadcastet,
nicht auf 0x500. WW heute 6,966 kWh / Strom heute 1,255 kWh (mit Display-TAG
konsistent). Für künftige Tageswert-Sensoren dort pollen.

**Build-Hinweis:** Das Manifest muss mit `framework: type: esp-idf` gebaut
werden – `arduino` lieferte unter ESPHome 2026.7.3 / IDF 5.5.5 einen
Linkfehler in `libwpa_supplicant.a` (vorkompilierte WLAN-Blobs inkompatibel).

### ⚠️ Wichtige Korrektur (29.07.2026 abends): Index-Raum ist modulspezifisch!

Beim Prozessdaten-Sniff kam heraus: **dieselben `0x09xx`-Indizes liefern auf
0x500 (Heizmodul) andere Werte als auf 0x514 (IWS)** – und die
**Display-Anzeige entspricht den IWS-Werten**:

| Zähler (Summe) | 0x500 (Heizmodul) | 0x514 (IWS) = Display |
|---|---|---|
| Wärme Heizen VD | 14.616 | **15.216** MWh |
| Wärme WW VD | 9.386 | 9.421 (Display 9.427, lief gerade) |
| Wärme NHZ Heizen (2WE) | 0 | **3.214** MWh |
| Wärme NHZ WW (2WE) | 0 | **0.367** MWh |
| El. Heizen VD | 2.240 | **2.033** MWh |
| El. WW VD | 2.294 | 2.202 MWh |

Die NHZ-Zähler existieren also nur auf der IWS (das Heizmodul meldet 2WE=0,
obwohl die NHZ real 378 h gelaufen ist). Warum die Heizmodul-Zähler abweichen
(Heizen −600 kWh, El +207 kWh), ist offen – **Referenz ist die IWS.**

**Konsequenz fürs Manifest:** Energie-Summenpolls auf die IWS umstellen –
Request-Header **`A1 14`** (bereits durch den Rücklauf-Poll verifiziert,
IWS antwortet mit Cmd `D2` auf CAN-ID 0x514), Decode auf `can_id == 0x514`.

## Prozessdaten ✅ komplett bestätigt (29.07.2026)

Alle Werte der 4 Prozessdaten-Screens (`Info → Wärmepumpe → Prozessdaten`)
liegen auf **CAN-ID 0x514 (IWS)**, Broadcast (Cmd `0x20`) beim Screen-Aufruf.
Foto+Log-Abgleich während laufender WW-Bereitung (Werte bewegten sich sichtbar
konsistent mit dem Display):

| Display | Index | Skalierung | Beleg (Display ↔ Log) |
|---|---|---|---|
| Rücklauftemperatur | `0x0016` | /10 °C | 37,9 ↔ 348–379 (bekannter Index) |
| Vorlauftemperatur | `0x01D6` | /10 °C | 44,0 ↔ 440–449 (= Elster `WPVORLAUFIST`) |
| Verdampfertemperatur | `0x07A9` | /10 °C | 16,9 ↔ 166–170 |
| Verdichtereintrittstemp. | `0x06D9` | /10 °C | 26,7 ↔ 259–303 |
| Heißgastemperatur | `0x0265` | /10 °C | 49,5 ↔ 488–513 |
| Verflüssigertemperatur | `0x0A39` | /10 °C | 38,9 ↔ 355–389 |
| Ölsumpftemperatur | `0x0A37` | /10 °C | 378/389 (alte Notiz „37,8 °C" passt) |
| Druck Niederdruck | `0x07A7` | /100 bar | 6,73 ↔ 654–695 |
| Druck Hochdruck | `0x07A6` | /100 bar | 17,42 ↔ 1729–1746 |
| WP Wasservolumenstrom | `0x02E2` | /10 l/min | 7,0 ↔ 70–80 |
| Strom Inverter | `0x06B2` | /10 A | 3,8 ↔ 38–40 |
| Spannung Inverter | `0x06B1` | /10 V | 224,0 ↔ 2240 |
| Ist-/Solldrehzahl Verdichter | `0x06EB` / `0x06EC` | Hz direkt | 58/55 ↔ 55–60 (welcher Ist ist: offen) |
| Rücklauftemp. Wärmequelle | `0x4FA6` | /10 °C | 13,3 ↔ 133/136 |
| Vorlauftemp. Wärmequelle | `0x4FA7` | /10 °C | 16,9 ↔ 169/172 |
| Wärmequellendruck | `0x4FA8` | /10 bar | 1,5 ↔ 15 |
| Leistung Wärmequellenpumpe | `0x4FA9` | /10 % | 40,6 ↔ 405–425 |

Nebenbefunde: `0xFDF4` (0x700) läuft deckungsgleich mit `0x0016` → sehr
wahrscheinlich Rücklauf auf dem Heizmodul-Kanal. `0xFDF3` (WP-Vorlauf) stieg
während der WW-Bereitung live 29,3→45,2 °C – dynamische Bestätigung.
`0x063B` (0x500, 43–58) vermutlich ebenfalls eine Drehzahl.

## Laufzeiten & Starts ✅ komplett bestätigt (29.07.2026)

Ebenfalls 0x514-Broadcasts. **Die alten Kandidaten-Vermutungen waren alle
falsch beschriftet** – `0x4F9A` ist nicht Wärmemenge, sondern Passivkühlung;
`0x0805` nicht Leistungsaufnahme, sondern NHZ-Laufzeit:

| Display | Index | Beleg |
|---|---|---|
| VD Heizen 5126 h | `0x4EFB` | 5125 ✓ |
| VD Warmwasser 2434 h | `0x4EFD` | 2434 ✓ exakt |
| NHZ 1 / NHZ 2 | `0x0259` / `0x025A` | 23/23 ✓ |
| NHZ 1/2 378 h | `0x0805` | 378 ✓ exakt |
| Passivkühlung 4223 h | `0x4F9A` | 4223 ✓ exakt |
| Verdichter-Starts 8595 | `0x4EF0` (×1000) + `0x4EF1` | 8 + 595 ✓ (Split-Register) |

(`0x4EFC`, `0x4F06`, `0x07FC–0x0809` außer `0x0805` melden `-32768` =
nicht verfügbar.)

## Gesamtsystem-Energiebilanz (Jahresfenster) – Block `0x50xx` auf 0x480

Aufgespürt durch Rohframe-Mitschnitt beim Navigieren von
`Info → Energiebilanz → Gesamtsystem` (Produktiv-Firmware loggt alle Frames).
Der komplette Screen liegt im **`0x50xx`-Block auf CAN-ID 0x480 (Manager)**,
Struktur wie der Energieblock: Energie als MWh+kWh-Paar, Effizienz als
einzelner ×100-Wert. **11 Werte digit-genau gegen die Display-Fotos bestätigt.**

**Zeitfenster 1–12 M** (rollend, im Manifest als 4 Sensoren umgesetzt):

| Wert | MWh-idx | kWh-idx | Display |
|---|---|---|---|
| Wärmemenge Heizen | `0x5007` | `0x5008` | 6,17 MWh |
| Wärmemenge WW | `0x500F` | `0x5010` | 4,283 MWh |
| Stromverbrauch Heizen | `0x5013` | `0x5014` | 0,817 MWh |
| Stromverbrauch WW | `0x501B` | `0x501C` | 0,991 MWh |
| Effizienz Heizen (÷100) | `0x501E` | – | 7,56 |
| Effizienz WW (÷100) | `0x5022` | – | 4,32 |

**Zeitfenster 13–24 M** (bestätigt, noch nicht als Sensor umgesetzt):

| Wert | MWh-idx | kWh-idx | Display |
|---|---|---|---|
| Wärmemenge Heizen | `0x502C` | `0x502D` | 7,502 MWh |
| Wärmemenge WW | `0x5030` | `0x5031` | 4,00 MWh |
| Stromverbrauch Heizen | `0x5032` | `0x5033` | 1,025 MWh |
| Stromverbrauch WW | `0x5036` | `0x5037` | 0,956 MWh |
| Effizienz WW (÷100) | `0x503A` | – | 4,18 |

Rollende Fenster → im Manifest `state_class measurement` (nicht
`total_increasing`, da der Wert sinken kann). COP wird bewusst **nicht**
gelesen, sondern HA-seitig aus Wärme/Strom berechnet.

**Request-Header ✅ gerätebestätigt (29.07.2026):** Manager-Indizes (0x480)
über 0x680 mit `91 00 FA 50 <idx_lo> 00 00` (`91 00` = `generate_read_id(0x480)`).
Der Manager antwortet auf unseren aktiven 0x680-Poll auf `id=0x480`. Live nach
Flash gelesen: Wärme Heizen 6179 kWh, WW 4283 kWh, Strom Heizen 817 kWh,
WW 991 kWh – deckt sich digit-genau mit dem Display (HA-COP daraus 7,56 / 4,32).

**Offen (nicht im 1–12-M-Scope):** COP Heizen 13–24 M (`val=732`, ~`0x5038`,
nicht erfasst) und einige Nachbar-Indizes (`0x500D/E`=7.953, `0x5019/A`=1.446,
`0x5021`=550 – evtl. „Gesamt"-/Lifetime-Zeilen oder kombinierter COP). Für die
Zuordnung bräuchte es einen korrelierten Sniff (Display-Zeile ↔ Frame).

## TODOs

### ⏭️ Session-Übergabe (Stand 30.07.2026 mittags, `logs/hk-parameter-write-test.log`)
**HEIZKREIS-1-Menü als Entities vollständig – vier weitere 0x601-Writes
gerätebestätigt:**

| Write | HA gesetzt | TX (0x680) | Modul-Antwort (0x601) | Status |
|---|---|---|---|---|
| Komforttemperatur `0x4EB8` | 25,0 °C | `C0 01 FA 4E B8 00 FA` | `D2 00 FA 4E B8 00 FA` | ✅ bestätigt |
| Eco-Temperatur `0x4EB9` | 20,1 °C | `C0 01 FA 4E B9 00 C9` | `D2 00 FA 4E B9 00 C9` | ✅ bestätigt |
| Minimal-Temp `0x4EA7` (Temp) | 10 / 16 °C | `.. A7 00 64` / `00 A0` | Poll `22 00 FA 4E A7 00 A0` | ✅ bestätigt |
| Raumeinfluss `0x4EA4` | 10 / 18 % | `.. A4 00 0A` / `00 12` | `D2 .. 00 0A` / `00 14` (=20!) | ✅ bestätigt |

Erkenntnisse aus dem Log:
- **Read-back nach Write** (Leseanfrage direkt hinterher) eingebaut; Modul
  antwortet ~0,8 s später mit Cmd **`D2`** (nicht `22`; `22` sind die
  60-s-Poll-Antworten – Parser wertet beide gleich). Steigung Heizkurve hatte
  noch keinen Read-back → Übernahme dauerte ~47 s bis zum Poll; jetzt ebenfalls
  Read-back eingebaut.
- **Raumeinfluss rastet am Gerät auf 5** (18 % gesendet → 20 % gespeichert,
  `D2 .. 00 14`). number-Entity daher auf `step: 5`.
- **Minimal-Temp „Aus"**: Schalter entfernt, „Aus" jetzt im Schieber integriert
  (0 = Aus/`0x9000`, 10–30 = Temperatur). Der **Lese-Pfad** ist bestätigt
  (Display auf „Aus" → `22 00 FA 4E A7 90 00` → Sensor NaN / Schieber 0). Das
  „Aus"-Frame im Log kam aber von **id=0x100** (WPM-Display selbst), nicht von
  unserem 0x680-Write.

**➡️ Einziger offener nächster Schritt:** Nach Reflash noch verifizieren:
(1) **Minimal-Temp „Aus" als HA-Write** – Schieber auf 0 → erwartet `TX id=0x680
raw=C0 01 FA 4E A7 90 00`, Display muss „Aus" zeigen. (2) Steigung-Read-back +
Raumeinfluss `step:5` gegenprüfen (unkritisch).

### Als Nächstes (priorisiert)
- [ ] **Parser-Fix verifizieren (21.07.)** – neu flashen, prüfen dass keine Sensoren mehr sporadisch auf 0/Unbekannt springen (Anfrage-Frames mit Cmd-Halbbyte 1 werden jetzt gefiltert)
- [x] **Komforttemperatur-Schreibtest** – überholt: Schreibformat geklärt (`C0 01`), Entities gebaut, Geräte-Test siehe Session-Übergabe oben
- [ ] **Aktive Betriebsart-Abfrage reparieren** – `41 01 FA 4F 1B` wird aktuell mit `-32768` auf 0x201 beantwortet (falsches Zielmodul); Betriebsart-Wert kommt derzeit nur über Broadcasts bei echten Wechseln. Anderes Anfrage-Header-Muster nötig
- [x] **Betriebsart auslesen** – Index `0x4F1B` bestätigt (Bereitschaft=1, Komfort=3, Eco=4, Warmwasser=5 alle verifiziert)
  - [ ] Programmbetrieb (erwartet: 2) verifizieren
  - [ ] Not-Betrieb (erwartet: 6) – bewusst nicht aktiv testen (Notfallmodus)
- [x] **Betriebsart-Schreiben getestet und bestätigt** – select-Entity in HA verfügbar, Komfort/Eco per CAN-Schreibbefehl erfolgreich verifiziert (Display wechselt sichtbar)
- [x] **Steigung Heizkurve auslesen** – Index `0x4F2B` bestätigt (0,40→0,45→0,40 verifiziert). Lese-Header `C1 01` (= `generate_read_id(0x601)`) am 29.07. **gerätebestätigt** (war zuvor nur abgeleitet).
  - [x] **Steigung Heizkurve SCHREIBEN gerätebestätigt (30.07., `logs/heizkurve-write-*.log`).** Unser Write von 0x680 mit `C0 01 FA 4F 2B <hi> <lo>` änderte 0,50→0,40 – **am WPM4-Display verifiziert** (`STEIGUNG HEIZKURVE 0.40`), TX-Log `C0 01 FA 4F 2B 00 28` vorhanden. Format per Display-Sniff hergeleitet: das WPM setzt mit einem einzelnen Frame `C0 01 FA 4F 2B <wert>` (Nibble **0**=Setzen, Modul-Adressierung `((0x601>>3)&0xF0)|0 = C0`, `0x601&0x1F = 01`). Der frühere `32 00`-Fehlschlag hatte ZWEI Fehler: falsches Modul (0x480) UND falsches Nibble (2 statt 0). Geräte-Gültigkeitsbereich laut `F8`/`F9`-Frames: **0,20–3,00**.
    - [ ] Optional: number-Entity-Untergrenze von 0,1 auf 0,2 anheben (Gerätemin), damit keine unter-Bereich-Werte gesendet werden
  - [x] **Komfort-/Eco-Temperatur schreiben (0x4EB8/0x4EB9) – gerätebestätigt (30.07., `logs/hk-parameter-write-test.log`).** `C0 01 FA 4E B8 00 FA` (25,0 °C) bzw. `C0 01 FA 4E B9 00 C9` (20,1 °C) über 0x680, Modul antwortet `D2 00 FA 4E B8/B9 <wert>`, Display übernimmt. Format (°C×10) aus dem Heizkurven-Write abgeleitet, Grenzen konservativ (Komfort 10–28, Eco 5–24 °C).
  - [x] Schreibformat für Komforttemperatur/Eco-Temperatur (0x4EB8/0x4EB9) geklärt – war Header-Problem (`32 00` falsch), korrekt ist `C0 01` (siehe oben). Entities gebaut.
  - [ ] Niveau/Komfort-Temperatur der Heizkurve auslesen (eigener Index, noch offen)
  - [x] Bonus-Kandidaten `0x4EA7`/`0x4EA4` auf 0x601 zugeordnet: **MINIMAL TEMPERATUR** (÷10, „Aus"=0x9000) bzw. **RAUMEINFLUSS** (×1 %). Siehe Abschnitt „Bonus-Kandidaten zugeordnet".
    - [x] Read-Sensoren + Schreib-Entities gebaut. Schalter „Minimal Temperatur aktiv" wieder **entfernt** (Commit a2dda79); „Aus" jetzt im Schieber „Minimal Temperatur setzen" integriert: 0 = Aus (`0x9000`), 10–30 °C = Temperatur, Werte <10 fängt die Set-Logik als „Aus" ab. Raumeinfluss `step:5` (Gerät rastet auf 5).
    - [x] **Min-Temp (Temperatur) + Raumeinfluss gerätebestätigt** (30.07., `logs/hk-parameter-write-test.log`): `C0 01 FA 4E A7 00 A0` = 16 °C, `C0 01 FA 4E A4 00 0A` = 10 % (18 % → Gerät speichert 20 %). Display übernimmt.
    - [ ] **Minimal-Temp „Aus" als HA-Write noch offen**: nur der Lese-Pfad ist bestätigt (Display-Aus → `0x9000` → Sensor NaN/Schieber 0). Der HA-Schreibweg (Schieber 0 → `TX 0x680 C0 01 FA 4E A7 90 00`) wurde noch nicht getestet.
- [ ] **Kühlkurve auslesen** – analog unter Einstellungen→Kühlen
- [ ] **Set-Telegramm-Format für Kühlkurve ableiten** – ACHTUNG: `32 00 FA ...` ist NICHT generisch. Es funktioniert nur fürs Betriebsart-Modul (0x480); beim Mischermodul (0x601, Heizkurve) blieb es wirkungslos (siehe Heizkurve-Schreiben oben). Schreib-Adressierung ist modulspezifisch und ≠ Lese-Header → pro Zielmodul per Display-Sniff bestätigen
- [ ] **Schreib-Service in ESPHome ergänzen** (number/select-Entities) – erst nachdem Set-Format bestätigt ist, mit Min/Max-Grenzen im Code

### Prozessdaten / Energie (aus vorheriger Session offen)
- [x] **Prozessdaten S.1–S.4 komplett bestätigt** (29.07.) – alle auf 0x514, siehe Abschnitt „Prozessdaten"
  - [x] Prozessdaten-Sensoren ins Manifest übernommen (16 Sensoren, aktiver `A1 14`-Poll über 0x680, gerätebestätigt in HA)
  - [ ] Ist- vs. Solldrehzahl auflösen (`0x06EB`/`0x06EC`) – bei **laufendem** Verdichter prüfen (im Test standen beide auf 0)
- [x] **Wärmemenge + Leistungsaufnahme (WP) bestätigt** – Block `0x091a–0x0931`, siehe Abschnitt „Energiewerte / Zähler"
  - [x] `SUM_KWH`-Register mitpollen und mit `SUM_MWH` kombinieren – im Manifest umgesetzt (6 Sensoren)
  - [x] Request-Header für 0x500-Ziel über 0x680 ermittelt (`A1 00 FA …`)
  - [x] **Manifest-Fix umgesetzt: Energie-Summenpolls von 0x500 auf 0x514 (IWS)** – Header `A1 14`, Decode `0x514`. In HA gerätebestätigt: Wärme Heizen 15.216, NHZ Heizen 3.214, NHZ WW 367, El Heizen 2.033 kWh = Display-Werte
  - [x] Gesamtsystem-Screen zugeordnet und im Manifest (4 Sensoren, 1–12M, gerätebestätigt)
  - [ ] Optional: Tageszähler-Sensoren (`…_TAG_*` auf 0x514) ergänzen, falls Tageswerte gewünscht
- [x] **Laufzeiten + Starts komplett bestätigt** (29.07.) – `0x4EFB`/`0x4EFD` (VD), `0x0259`/`0x025A` (NHZ1/2), `0x0805` (NHZ1/2 gemeinsam), `0x4F9A` (Passivkühlung), `0x4EF0`/`0x4EF1` (Starts, Split). Siehe Abschnitt „Laufzeiten & Starts"
  - [ ] Optional: Laufzeit-/Start-Sensoren ins Manifest (diagnostic, langsames Intervall)
- [ ] Mischermodul-Wert (`0x4EB4` auf 0x601, ~19,1–19,2°C) einer konkreten Bedeutung zuordnen (vermutlich Vorlauf HK2)
- [x] `0x0074` geklärt = **EVU_SPERRE_AKTIV** (Standard-Elster, val=1=aktiv; ändert sich nicht mit Betriebsart – konsistent)
  - [ ] `0x4EB3` auf 0x401/0x100 (val=1) noch offen

### Sonstiges / Housekeeping
- [ ] Meldungsliste (aktuell 14 Einträge) inhaltlich prüfen – harmlose Altmeldungen oder aktive Störungen?
- [ ] Aktives Polling-Intervall (aktuell 60s) ggf. anpassen/pro Wert individualisieren
- [ ] Manifest ggf. auf Pull Request Richtung `OneESP32ToRuleThemAll` vorbereiten, sobald Wertesatz stabil ist (WPE-I-Manifest existiert dort noch nicht)

## Hilfreiche externe Referenzen



- **Elster-Protokolltabelle:** `http://juerg5524.ch/data/ElsterTable.inc`
- **OneESP32ToRuleThemAll** (Referenzprojekt, andere WP-Baureihen): `github.com/kr0ner/OneESP32ToRuleThemAll`
- **Stiebel Eltron Modbus-Doku** (andere Adressierung, aber gleiche Sentinel-Werte/Kategorien): offizielle ISG-Modbus-Anleitung
- **pail23/stiebel_eltron_isg_component** (Modbus, braucht ISG – nicht direkt nutzbar, aber gute Namens-Roadmap)
