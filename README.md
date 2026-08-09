# esphome-stiebel-wpe-i-isg – Stiebel Eltron WPE-I ohne ISG in Home Assistant

ESPHome-Firmware, die eine **Stiebel Eltron WPE-I 06 HKW 230 Premium**
(Sole-Wasser-Wärmepumpe, Inverter-Generation) **ohne das ISG-Gateway** direkt
über den internen CAN-Bus (Elster/Kromschröder-Protokoll) an Home Assistant
anbindet – lesend *und* teilweise schreibend, **rein lokal ohne Cloud**.

Das **ISG** (Internet-Service-Gateway) ist Stiebels offizielles,
kostenpflichtiges Zubehör für Modbus/Fernzugriff. Dieses Projekt erschließt
dieselben Anlagenwerte **ohne ISG** über ein ~30-€-Board. Als Ausbaustufe ist
zusätzlich eine **ISG-kompatible Modbus-TCP-Emulation** geplant (Doku, TODO F),
damit bestehende ISG-/Modbus-Integrationen unser Interface ohne Anpassung nutzen
können.

Entstanden durch eigenes Reverse Engineering, da die WPE-I-Baureihe von
bestehenden Projekten wie
[OneESP32ToRuleThemAll](https://github.com/kr0ner/OneESP32ToRuleThemAll)
(WPL/THZ-Baureihen) bisher nicht abgedeckt wird.

## Features

**Lesend (bestätigt):**
Außentemperatur · Warmwasser Ist/Soll · Rücklauf-/Vorlauftemperatur ·
Raumtemperatur & -luftfeuchte (FET-Fernfühler) · Betriebsart ·
Steigung Heizkurve · Komfort-/Eco-Temperatur Heizkreis ·
Meldungslisten-Zähler

**Schreibend (bestätigt):**
- Betriebsart als Home-Assistant-Dropdown (Bereitschaft / Programm /
  Komfort / Eco / Warmwasser)

**Schreibend (experimentell, ungetestet/zurückgestellt):**
- Steigung Heizkurve (`number`-Entity vorhanden, Schreibtest ausstehend)
- Komfort-/Eco-Temperatur (Entities entfernt, Format in Klärung –
  siehe Doku)

### Betriebsmodus & Sicherheit

Gestaffelter, umschaltbarer Zugriff (HA-Auswahl **„Betriebsmodus"**,
Default **1**):

| Modus | Bus-Verhalten | Schreiben |
|---|---|---|
| **0 – Nur lauschen** | reiner Sniffer, **kein** TX vom ESP | nein |
| **1 – Lesen (Poll)** | aktives Pollen der Lesewerte | nein |
| **2 – Vollzugriff** | Lesen + Schreiben | ja |

- **Echter Read-only-Build:** Die Schreib-Entities liegen separat in
  `esphome/wpe-i-writes.yaml`. Wird die Datei nicht eingebunden, ist die
  Firmware **physisch nicht schreibfähig** (Modus 2 hat dann keine Wirkung).
- **Debug-Logging** als HA-Schalter: schaltet den Roh-Frame-Dump im Log zu –
  Normalbetrieb bleibt ruhig.

## Vergleich: ISG vs. dieses Projekt

| | Stiebel **ISG** (offiziell) | **dieses Projekt** |
|---|---|---|
| Anschaffung | kostenpflichtiges Gerät | ~30 € Board + DIY (Verdrahten/Flashen) |
| Home-Assistant-Anbindung | über Modbus-Integration | **nativ** (ESPHome-Entities, sofort) |
| Lesen (Temp, WW, Energie/COP) | ja | ja (verifizierte, wachsende Teilmenge) |
| Schreiben/Steuern | ja (dok. Register) | ja (verifizierte Teilmenge; Rest experimentell) |
| Live-Prozessdaten (Verdichter, Inverter, Kältekreis, Sole) | eingeschränkt | **umfangreich** (mehr als das ISG-Modbus liefert) |
| Cloud/Fernzugriff (ServiceWelt) | ja | nein – **bewusst rein lokal** |
| Modbus-TCP-Kompatibilität | ja (nativ) | geplant (**ISG-Emulation**, TODO F) |
| Read-only-Betrieb erzwingbar | – | ja (Read-only-Build + Betriebsmodus) |
| Offizieller Support / Garantie | ja (Hersteller) | nein (Community, **eigenes Risiko**) |
| Inbetriebnahme | Plug-and-play | Verdrahten + Flashen (Adopt-Flow für Updates) |

> Der Vergleich beruht auf den öffentlichen ISG-Modbus-/Software-Unterlagen
> (extern verlinkt in [`docs/reverse-engineering.md`](docs/reverse-engineering.md))
> und ist nach bestem Wissen erstellt; einzelne ISG-Zeilen
> können je nach ISG-Variante/Softwarestand abweichen. Das ISG bleibt für
> Plug-and-play, Cloud-Fernzugriff und Herstellersupport die einfachere Wahl.

## Hardware

- Waveshare **ESP32-S3-RS485-CAN** (Industrieboard) mit integriertem,
  galvanisch getrenntem CAN-Transceiver (TJA1051T/3) – Details:
  [`docs/hardware.md`](docs/hardware.md)
- Anschluss an Klemme **X1.18** (CAN B, FET/ISG-Anschluss) des
  WPM4-Reglers – H → CAN-H, L → CAN-L
- Terminierungs-Jumper **offen** lassen (Abgriff, kein Busende)
- CAN-Pins: TX = GPIO15, RX = GPIO16 · Bitrate: **50 kbps**

⚠️ **Vor Arbeiten am WPM die Anlage allpolig spannungsfrei schalten.**
Nach dem Freischalten kann bis zu 5 Minuten Restspannung anliegen
(Inverter-Kondensatoren).

## Installation

Die Firmware ist als **ESPHome-Package** aufgebaut: der wiederverwendbare Kern
liegt in `esphome/wpe-i-package.yaml`, die Netz-/Gerätedaten bleiben getrennt.
Es gibt zwei Wege.

### A) Als ESPHome-Package übernehmen (empfohlen)

Sobald das Repo öffentlich ist, meldet sich eine geflashte WPE-I-Firmware im
ESPHome-/Home-Assistant-Dashboard zur Übernahme an („Adopt"). Alternativ das
Package direkt referenzieren – eigene Mini-Config:

```yaml
substitutions:
  name: wpe-i-heatpump
packages:
  wpe_i: github://fabkoe/esphome-stiebel-wpe-i-isg/esphome/wpe-i-package.yaml@main
wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
```

In den ESPHome-`secrets` werden `wifi_ssid`, `wifi_password` **und**
`ota_password` erwartet. Board/Pins sind auf die Waveshare ESP32-S3-RS485-CAN
vorbelegt; für andere Boards die `substitutions` (`board`, `can_tx_pin`,
`can_rx_pin`) überschreiben.

Dieses Package allein ist **read-only** (keine Schreib-Entities). Wer schreiben
will, bindet zusätzlich `wpe-i-writes.yaml` ein:

```yaml
packages:
  wpe_i:        github://fabkoe/esphome-stiebel-wpe-i-isg/esphome/wpe-i-package.yaml@main
  wpe_i_writes: github://fabkoe/esphome-stiebel-wpe-i-isg/esphome/wpe-i-writes.yaml@main
```

### B) Lokal bauen / entwickeln (esptool)

1. Repo klonen, `esphome/secrets.yaml.example` nach
   `esphome/secrets.yaml` kopieren und WLAN- + OTA-Daten eintragen
2. `esphome/wpe-i-manifest.yaml` kompilieren und flashen – die Datei bindet
   `wpe-i-package.yaml` **und** `wpe-i-writes.yaml` ein (Voll-Build). Für einen
   lokalen Read-only-Build die zweite `!include`-Zeile weglassen.
3. Gerät erscheint in Home Assistant mit allen Sensoren + Steuer-Entities

## Dokumentation

Der komplette Reverse-Engineering-Stand (Protokoll-Grundlagen,
alle bestätigten Elster-Indizes, CAN-ID-Zuordnungen, Menüstruktur des
WPM4, offene TODOs und die bewährte Vorgehensweise für neue Werte)
steht in [`docs/reverse-engineering.md`](docs/reverse-engineering.md).

## Projektstruktur

```
esphome/
  wpe-i-package.yaml        # Kern: Sensoren + Poller + Betriebsmodus (read-only, Package-Ziel)
  wpe-i-writes.yaml         # Schreib-Entities (nur Voll-Build; weglassen = read-only)
  wpe-i-manifest.yaml       # schlanke lokale Flash-Config (bindet Package + Writes + WLAN)
  secrets.yaml.example      # Vorlage für WLAN- + OTA-Zugangsdaten
docs/
  reverse-engineering.md    # Protokoll-Doku, Wertetabellen, TODOs
  wpm4-menue.md             # Menübaum des WPM4-Displays (Coverage-Legende)
  hardware.md               # Board Waveshare ESP32-S3-RS485-CAN (Pinout, Links)
archive/
  wpe-can-sniffer-v1.yaml   # historisch: erster Sniffer (MQTT-basiert)
  wpe-can-sniffer-v2.yaml   # historisch: Sniffer über ESPHome-Logs
LICENSE · NOTICE            # Apache-2.0 + Marken-/Herkunftshinweis
CONTRIBUTING.md             # Mitwirken + DCO-Sign-off
```

## Disclaimer

**Privates Reverse-Engineering-Projekt, keinerlei Verbindung zu Stiebel Eltron.**
„Stiebel Eltron", „WPE-I", „WPM" und weitere Namen sind Marken ihrer jeweiligen
Inhaber und werden hier nur zur Beschreibung der Kompatibilität genannt.

Nutzung **auf eigene Gefahr, ohne jede Gewährleistung** (siehe Lizenz). Im
Einzelnen:

- ⚠️ **Sachschaden:** Falsche Schreibwerte können laut Betriebsanleitung zu
  Schäden an Wärmepumpe oder Estrich führen. Schreibzugriffe nur bewusst und mit
  verstandenem Format nutzen. Lesen ist unkritisch, Schreiben nicht.
- ⚡ **Elektrische Gefahr:** Arbeiten am WPM/CAN-Bus nur durch fachkundige
  Personen und an der **allpolig spannungsfrei geschalteten** Anlage
  (Restspannung bis ~5 min).
- 🔬 **Ein-Geräte-Basis:** Alle Indizes wurden an **genau einem** Gerät
  (WPE-I 06 HKW 230 Premium, WPM4 SW 449-10) verifiziert. Andere Baureihen oder
  Softwarestände können abweichen – vor dem Schreiben am eigenen Gerät prüfen.
- 🔌 **Kein Support/keine Haftung** für Folgen aus Nachbau, Fehlkonfiguration
  oder abweichender Hardware. Ein Eingriff kann Garantie-/Gewährleistungs-
  ansprüche gegenüber dem Hersteller berühren.

## Lizenz

Veröffentlicht unter der **Apache-2.0-Lizenz** – siehe [`LICENSE`](LICENSE)
(inkl. Patent-Grant) und [`NOTICE`](NOTICE) für den Marken-/Herkunftshinweis.
Beiträge laufen über den **DCO-Sign-off** – siehe
[`CONTRIBUTING.md`](CONTRIBUTING.md).
*(Copyright-Zeile in `NOTICE` noch mit deinem Namen füllen.)*

## Danksagung / Referenzen

- Elster-Protokolltabelle: [juerg5524.ch](http://juerg5524.ch/data/ElsterTable.inc)
  (Mirror: [andig/canprogs](https://github.com/andig/canprogs))
- [kr0ner/OneESP32ToRuleThemAll](https://github.com/kr0ner/OneESP32ToRuleThemAll)
  als Referenzprojekt für die WPL/THZ-Baureihen
