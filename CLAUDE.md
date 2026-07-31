# CLAUDE.md – Projektkontext für Claude Code

## Was dieses Projekt ist

ESPHome-Firmware, die eine **Stiebel Eltron WPE-I 06 HKW 230 Premium**
Wärmepumpe **ohne ISG** über deren internen CAN-Bus
(Elster/Kromschröder-Protokoll) an Home Assistant anbindet.
Alle CAN-Indizes wurden selbst reverse-engineered, da die
WPE-I-Baureihe (neuere Inverter-Generation) von bestehenden Projekten
nicht abgedeckt wird. **Dies ist das eigene Produktivprojekt** – es
bleibt die maßgebliche Basis (volle Kontrolle, verifizierte
Schreibformate, keine Fremd-Merge-Abhängigkeit).

## Wichtigste Dateien

- `esphome/wpe-i-manifest.yaml` – die Firmware. Ein generischer
  CAN-Frame-Parser in der `on_frame`-Lambda decodiert alle Frames
  und speist die Sensor-/Steuer-Entities.
- `docs/reverse-engineering.md` – **die zentrale Wissensbasis**:
  Protokollformat, alle bestätigten Elster-Indizes mit CAN-IDs und
  Skalierung, CAN-ID→Modul-Zuordnung, Menüstruktur des WPM4-Displays,
  PoC-Ergebnisse, offene TODOs. **Bei jeder neuen Erkenntnis mitpflegen!**

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
- Herkunft: proprietäres Protokoll der Elster/Kromschröder-Regler­
  elektronik, community-dokumentiert über die Elster-Tabelle von
  Jürg Müller (`juerg5524.ch`). WPE-I erweitert diese um Indizes, die
  in der klassischen Tabelle fehlen.

## Sicherheitsregeln (wichtig!)

1. **Schreibzugriffe auf die Wärmepumpe nur nach explizitem Nutzer-OK**
   und nur mit vorher am echten Gerät verifiziertem Format.
   Falsche Werte können laut Betriebsanleitung Wärmepumpe oder
   Estrich beschädigen.
2. Neue Schreib-Entities immer mit konservativen min/max-Grenzen.
3. Not-Betrieb (Betriebsart 6) niemals aktiv setzen/testen.
4. Nach jedem Schreibtest: Display-Kontrolle am Gerät verlangen.

## Arbeitsgrundsätze (für Claude)

Bias in diesem Projekt: **Vorsicht vor Tempo** – am Gerät hängen echte
Kosten (Schäden an WP/Estrich, verfälschte RE-Daten). Bei trivialen
Aufgaben mit Augenmaß.

1. **Erst denken, dann coden.** Annahmen explizit benennen. Bei
   Unklarheit oder mehreren Deutungen: nachfragen, nicht still eine
   Variante wählen. Einfacheren Weg ansprechen, begründet widersprechen.
2. **Einfachheit zuerst.** Minimale Änderung, die das Problem löst.
   Keine spekulativen Features, keine Abstraktion für Einmal-Code,
   kein Error-Handling für unmögliche Fälle.
3. **Chirurgische Änderungen.** Am funktionierenden Manifest nur
   anfassen, was nötig ist. Bestehenden Stil übernehmen, nichts
   „nebenbei verbessern", vorhandenen toten Code nur melden, nicht
   löschen. Jede geänderte Zeile muss sich direkt auf die Anfrage
   zurückführen lassen.
4. **Zielgetrieben arbeiten – Verifikation ist Geräte-Verifikation.**
   Es gibt keine Testsuite; „verifiziert" heißt **Log+Display-Abgleich
   am echten Gerät** (siehe Arbeitsweise unten). Erfolgskriterium vorab
   festlegen: z. B. „Wert erscheint im Log und stimmt mit der
   Display-Anzeige überein". Ein Wert/Schreibformat gilt erst dann als
   bestätigt.

## Arbeitsweise für neue Werte

Bewährter Ablauf (siehe docs/reverse-engineering.md, Abschnitt
"Bewährtes Vorgehen"): ESPHome-Log mitschneiden, Nutzer ändert Wert
am WPM-Display mit notierter Uhrzeit, Log-Frames im Zeitfenster mit
dem Display-Wert abgleichen. Wert gilt erst als "bestätigt", wenn
er einer realen Display-Anzeige zugeordnet wurde.

**RE-Beschleuniger:** Das Framework `bullitt186/ha-stiebel-control`
loggt jeden Frame automatisch mit Elster-Namen + Typ + Skalierung und
markiert Unbekanntes als `INDEX_NOT_FOUND`. Als **beschrifteten
Sniffer** zum Aufspüren offener Werte nutzen; bestätigte Indizes dann
in dieses Projekt übernehmen. (Klon liegt unter
`../ha-stiebel-control-poc`, kein Produktivstand.)

## Entwicklungs-Workflow (PC / esptool – aktuell, nicht HA)

Entwickelt und geflasht wird derzeit **lokal am PC** über die
ESPHome-CLI, nicht über das HA-Add-on.

- **Python:** über den `py`-Launcher **`py -3.12`** ansteuern. Der
  Default `py` zeigt auf 3.14, das ESPHome (noch) nicht unterstützt.
- **Secrets:** `esphome/secrets.yaml` aus `secrets.yaml.example`
  anlegen (WLAN-Daten), nie committen.
- **Flashen (USB):** ESP per Datenkabel anstecken, dann
  `cd esphome; py -3.12 -m esphome run wpe-i-manifest.yaml` und den
  seriellen COM-Port wählen. Erster Build lädt einmalig die
  ESP32-Toolchain (PlatformIO).
- **Nur validieren:** `py -3.12 -m esphome config wpe-i-manifest.yaml`
- **Logs mitlesen:** `py -3.12 -m esphome logs wpe-i-manifest.yaml
  --device COMx` (USB) bzw. `--device <IP>` (über WLAN/API).
- **COM-Port fehlt?** BOOT halten, RESET tippen, BOOT loslassen
  (Bootloader-Modus); ggf. USB-Serial-Treiber (CP210x/CH34x).

## Aktueller Stand / nächste Schritte

Siehe TODO-Sektion in `docs/reverse-engineering.md` (frische Prioritätenliste
oben). Kurzfassung (31.07.2026):
Heizkreis-1- und Kühlen-Menü komplett gelesen + schreibend gerätebestätigt.
Alle `0x601`-Kühlwerte schreibend abgenommen (Raumsoll `0x4F04` log+display,
Steigung/Starttemp/KühlART display-verifiziert). **Schreib-Systematik:**
`0x601`-Modul → `C0 01`, `0x180`-Manager → `32 00` (kein generischer Header!).
Lesend im Manifest **und am Gerät bestätigt**: Prozessdaten, Energiezähler,
Energiebilanz (1–12M + 13–24M + Effizienz), Laufzeiten, Starts, Vorlauf-PD
(`logs/readsensors-verify.log`). `entity_category` (Konfiguration/Diagnose)
gesetzt; Minimal-Temp als Text-Sensor („Aus").
Offen (Prioritäten in docs): Leistung `0x7A40` Schreib-Modul einkreisen;
Display-Name KÜHLEN `0x4F07`; neue Leseziele (Taupunkt, Kühlen-Live-Werte,
E-Nacherwärmung, Heizung-Unterwerte); **ansehnliches HA-Dashboard bauen**.
PoC mit `ha-stiebel-control`
abgeschlossen: Framework liest die WPE-I bei 50 kbps korrekt
(display-verifiziert), taugt aber nur als RE-Werkzeug/Backup, nicht als
Ersatz – Details im PoC-Abschnitt der docs.
