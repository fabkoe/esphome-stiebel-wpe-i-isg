# CLAUDE.md – Projektkontext für Claude Code

## Was dieses Projekt ist

ESPHome-Firmware, die eine **Stiebel Eltron WPE-I 06 HKW 230 Premium**
Wärmepumpe **ohne ISG** über deren internen CAN-Bus
(Elster/Kromschröder-Protokoll) an Home Assistant anbindet.
Alle CAN-Indizes wurden selbst reverse-engineered, da die
WPE-I-Baureihe von bestehenden Projekten nicht abgedeckt wird.

## Wichtigste Dateien

- `esphome/wpe-i-manifest.yaml` – die Firmware. Ein generischer
  CAN-Frame-Parser in der `on_frame`-Lambda decodiert alle Frames
  und speist die Sensor-/Steuer-Entities.
- `docs/reverse-engineering.md` – **die zentrale Wissensbasis**:
  Protokollformat, alle bestätigten Elster-Indizes mit CAN-IDs und
  Skalierung, CAN-ID→Modul-Zuordnung, Menüstruktur des WPM4-Displays,
  offene TODOs. **Bei jeder neuen Erkenntnis mitpflegen!**

## Protokoll-Kurzreferenz (Details in docs/)

- CAN 50 kbps, 7-Byte-Frames
- Kurzer Index: `[Cmd][00][Idx][Wert_hi][Wert_lo][00][00]`
- Erweiterter Index (Byte2=0xFA): `[Cmd][00][FA][Idx_hi][Idx_lo][Wert_hi][Wert_lo]`
- Cmd-Byte unteres Halbbyte: `1`=Lese-Anfrage (Wertefeld=0, NIE als
  Messwert werten!), `2`=Antwort/Schreiben, `0`=Broadcast
- Werte: big-endian int16, Temperaturen /10, Heizkurve /100
- `-32768` (0x8000) = "nicht verfügbar"
- Eigene Anfragen/Schreibbefehle IMMER über CAN-ID `0x680` senden
  (PC/ComfortSoft-Kanal, unbenutzt) – NIE über 0x100 (stark belegt,
  Kollisionsgefahr)

## Sicherheitsregeln (wichtig!)

1. **Schreibzugriffe auf die Wärmepumpe nur nach explizitem Nutzer-OK**
   und nur mit vorher am echten Gerät verifiziertem Format.
   Falsche Werte können laut Betriebsanleitung Wärmepumpe oder
   Estrich beschädigen.
2. Neue Schreib-Entities immer mit konservativen min/max-Grenzen.
3. Not-Betrieb (Betriebsart 6) niemals aktiv setzen/testen.
4. Nach jedem Schreibtest: Display-Kontrolle am Gerät verlangen.

## Arbeitsweise für neue Werte

Bewährter Ablauf (siehe docs/reverse-engineering.md, Abschnitt
"Bewährtes Vorgehen"): ESPHome-Log mitschneiden, Nutzer ändert Wert
am WPM-Display mit notierter Uhrzeit, Log-Frames im Zeitfenster mit
dem Display-Wert abgleichen. Wert gilt erst als "bestätigt", wenn
er einer realen Display-Anzeige zugeordnet wurde.

## Aktueller Stand / nächste Schritte

Siehe TODO-Sektion in `docs/reverse-engineering.md`. Kurzfassung:
13 Werte lesend bestätigt, Betriebsart schreibend bestätigt (select
in HA), Parser-Fix für Anfrage/Antwort-Unterscheidung frisch drin
(muss noch am Gerät verifiziert werden), Komforttemperatur-Schreibtest
sollte mit dem Fix wiederholt werden, Kühlkurve + Prozessdaten +
Energiewerte noch offen.
