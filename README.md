**🇩🇪 Deutsch** · [🇬🇧 English](README.en.md)

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

## Was es kann

**Lesen** (~60 Sensor-Entities, gruppiert nach Alltag / Konfiguration / Diagnose):

- **Temperaturen:** Außen, Warmwasser Ist/Soll, Vorlauf/Rücklauf,
  Raumtemperatur & -luftfeuchte (FET-Fernfühler)
- **Einstellungen:** Betriebsart, Heizkurve (Steigung, Komfort-/Eco-Temperatur,
  Raumeinfluss, Minimaltemperatur), Kühlen (Raumsoll, Kühlkurve, Starttemperatur,
  Hysterese, Leistung, Kühlart)
- **Live-Prozessdaten (Kältekreis):** Verdampfer-, Verdichtereintritts-,
  Heißgas-, Verflüssiger-, Ölsumpftemperatur, Nieder-/Hochdruck, Volumenstrom,
  Inverter-Strom/-Spannung, Verdichter Ist-/Solldrehzahl
- **Wärmequelle (Sole):** Vor-/Rücklauf, Druck, Pumpenleistung
- **Energie & Effizienz:** Wärmemengen und Stromverbrauch (Verdichter +
  Nacherwärmung, Heizen/Warmwasser), Jahresbilanz 1–12 & 13–24 Monate, COP
- **Laufzeiten & Starts:** Verdichter (Heizen/WW), Nacherwärmung, Passivkühlung,
  Verdichter-Starts · plus Meldungslisten-Zähler

**Schreiben / Steuern** (nur im Voll-Build & Betriebsmodus „Vollzugriff"):

- **Betriebsart** als HA-Dropdown (Bereitschaft / Programm / Komfort / Eco /
  Warmwasser)
- **Heizkreis:** Steigung Heizkurve, Komfort-/Eco-Temperatur, Raumeinfluss,
  Minimaltemperatur
- **Kühlen:** Raumsolltemperatur, Steigung Kühlkurve, Starttemperatur,
  Hysterese, Kühlkreis / Kühlart / „Kühlen"-Schalter

Die Schreibwerte wurden an einem Gerät per Log + Display bestätigt; einzelne
sind noch experimentell – Formate, Grenzen und Status stehen in
[`docs/reverse-engineering.md`](docs/reverse-engineering.md).

## Wie es funktioniert

Der ESP32 hängt als stiller Abgriff am internen CAN-Bus des WPM4-Reglers und
spricht das Elster/Kromschröder-Protokoll (50 kbps, 7-Byte-Frames):

1. **Mithören:** Ein Decoder zerlegt jeden Frame (Index + Wert) und füttert die
   passende Home-Assistant-Entity.
2. **Aktiv abfragen:** Werte, die die WPM nicht von selbst sendet, pollt der ESP
   alle 60 s über den freien PC/ComfortSoft-Kanal (CAN-ID `0x680`) – kollisionsfrei.
3. **Schreiben:** Steuer-Entities senden einen Write-Frame + sofortigen
   Read-back, damit der neue Wert direkt in HA erscheint – nur im Modus „Vollzugriff".

Alle Indizes und Skalierungen sind selbst reverse-engineered und in
[`docs/reverse-engineering.md`](docs/reverse-engineering.md) dokumentiert.

## Betriebsmodus & Read-only

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
  wpe_i: github://fabkoe/esphome-stiebel-wpe-i-isg/esphome/wpe-i-package.yaml@v1.0.0
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
  wpe_i:        github://fabkoe/esphome-stiebel-wpe-i-isg/esphome/wpe-i-package.yaml@v1.0.0
  wpe_i_writes: github://fabkoe/esphome-stiebel-wpe-i-isg/esphome/wpe-i-writes.yaml@v1.0.0
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
  wpe-i-sniffer.yaml        # optionaler RE-Firehose: CAN → MQTT (InfluxDB/Grafana)
  wpe-i-manifest.yaml       # schlanke lokale Flash-Config (bindet Package + Writes + WLAN)
  secrets.yaml.example      # Vorlage für WLAN- + OTA- (+ optional MQTT-) Zugangsdaten
docs/
  reverse-engineering.md    # Protokoll-Doku, Wertetabellen, TODOs
  can-logging.md            # CAN-Langzeit-Mitschnitt (MQTT → InfluxDB → Grafana)
  wpm4-menue.md             # Menübaum des WPM4-Displays (Coverage-Legende)
  hardware.md               # Board Waveshare ESP32-S3-RS485-CAN (Pinout, Links)
tools/
  telegraf-wpe-i.conf       # Telegraf-Config: MQTT-Firehose → InfluxDB v2
archive/
  wpe-can-sniffer-v1.yaml   # historisch: erster Sniffer (MQTT-basiert)
  wpe-can-sniffer-v2.yaml   # historisch: Sniffer über ESPHome-Logs
LICENSE · NOTICE            # Apache-2.0 + Marken-/Herkunftshinweis
CONTRIBUTING.md             # Mitwirken + DCO-Sign-off
```

## Wichtig (bitte kurz lesen)

Privates Bastel-/Reverse-Engineering-Projekt, **nichts mit Stiebel Eltron zu tun** –
Nutzung auf eigene Gefahr, ohne Gewähr.

- **Lesen ist harmlos, Schreiben nicht.** Falsche Schreibwerte können laut
  Handbuch die Wärmepumpe oder den Estrich beschädigen – darum ist Read-only der
  Default, Schreiben schaltest du bewusst zu.
- **Strom aus:** am WPM/CAN-Bus nur an der spannungsfrei geschalteten Anlage
  arbeiten (Restspannung bis ~5 min).
- **Ein Gerät:** alles an genau einer WPE-I 06 HKW 230 (WPM4 449-10) getestet –
  bei anderer Baureihe/Firmware kann's abweichen.

## Lizenz & Mitmachen

Apache-2.0 (siehe [`LICENSE`](LICENSE)). Beiträge sind willkommen – Commits kurz
per DCO signieren (`git commit -s`), Details in [`CONTRIBUTING.md`](CONTRIBUTING.md).

## Danksagung / Referenzen

- Elster-Protokolltabelle: [juerg5524.ch](http://juerg5524.ch/data/ElsterTable.inc)
  (Mirror: [andig/canprogs](https://github.com/andig/canprogs))
- [andig/goelster](https://github.com/andig/goelster) – Elster-CAN in Go, gute
  Referenz für Index-Skalierungen
- [kr0ner/OneESP32ToRuleThemAll](https://github.com/kr0ner/OneESP32ToRuleThemAll)
  – Referenzprojekt für die WPL/THZ-Baureihen
- [bullitt186/ha-stiebel-control](https://github.com/bullitt186/ha-stiebel-control)
  – beschrifteter Frame-Logger, half beim Aufspüren offener Werte
