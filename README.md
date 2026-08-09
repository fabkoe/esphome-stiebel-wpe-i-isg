# WPE-I CAN – Stiebel Eltron WPE-I ohne ISG in Home Assistant

ESPHome-Firmware, die eine **Stiebel Eltron WPE-I 06 HKW 230 Premium**
(Sole-Wasser-Wärmepumpe, Inverter-Generation) **ohne ISG** über den internen
CAN-Bus (Elster/Kromschröder-Protokoll) an Home Assistant anbindet –
lesend *und* teilweise schreibend.

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

## Hardware

- Waveshare **ESP32-S3 Industrial Board** mit integriertem, isoliertem
  CAN-Transceiver (SN65HVD230)
- Anschluss an Klemme **X1.18** (CAN B, FET/ISG-Anschluss) des
  WPM4-Reglers – H → CAN-H, L → CAN-L
- Terminierungs-Jumper **offen** lassen (Abgriff, kein Busende)
- CAN-Pins: TX = GPIO15, RX = GPIO16 · Bitrate: **50 kbps**

⚠️ **Vor Arbeiten am WPM die Anlage allpolig spannungsfrei schalten.**
Nach dem Freischalten kann bis zu 5 Minuten Restspannung anliegen
(Inverter-Kondensatoren).

## Installation

1. Repo klonen, `esphome/secrets.yaml.example` nach
   `esphome/secrets.yaml` kopieren und WLAN-Daten eintragen
2. `esphome/wpe-i-manifest.yaml` mit ESPHome kompilieren und flashen
3. Gerät erscheint in Home Assistant mit allen Sensoren + Steuer-Entities

## Dokumentation

Der komplette Reverse-Engineering-Stand (Protokoll-Grundlagen,
alle bestätigten Elster-Indizes, CAN-ID-Zuordnungen, Menüstruktur des
WPM4, offene TODOs und die bewährte Vorgehensweise für neue Werte)
steht in [`docs/reverse-engineering.md`](docs/reverse-engineering.md).

## Projektstruktur

```
esphome/
  wpe-i-manifest.yaml       # Haupt-Firmware (Sensoren + Steuerung)
  secrets.yaml.example      # Vorlage für WLAN-Zugangsdaten
docs/
  reverse-engineering.md    # Protokoll-Doku, Wertetabellen, TODOs
  wpm4-menue.md             # Menübaum des WPM4-Displays (Coverage-Legende)
  hardware.md               # Board Waveshare ESP32-S3-RS485-CAN (Pinout, Links)
  manuals/                  # lokal gesicherte offizielle Handbücher + Schaltbild
archive/
  wpe-can-sniffer-v1.yaml   # historisch: erster Sniffer (MQTT-basiert)
  wpe-can-sniffer-v2.yaml   # historisch: Sniffer über ESPHome-Logs
```

## Disclaimer

Privates Reverse-Engineering-Projekt ohne Verbindung zu Stiebel Eltron.
Schreibzugriffe auf die Wärmepumpe erfolgen auf eigene Gefahr – falsche
Einstellungen können laut Betriebsanleitung zu Schäden an Wärmepumpe
oder Estrich führen. Alle Indizes wurden an genau einem Gerät
(WPE-I 06 HKW 230 Premium, WPM4 SW 449-10) verifiziert; andere
Baureihen/Softwarestände können abweichen.

## Danksagung / Referenzen

- Elster-Protokolltabelle: [juerg5524.ch](http://juerg5524.ch/data/ElsterTable.inc)
  (Mirror: [andig/canprogs](https://github.com/andig/canprogs))
- [kr0ner/OneESP32ToRuleThemAll](https://github.com/kr0ner/OneESP32ToRuleThemAll)
  als Referenzprojekt für die WPL/THZ-Baureihen
