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
  Skalierung, CAN-ID→Modul-Zuordnung, PoC-Ergebnisse, offene TODOs.
  **Bei jeder neuen Erkenntnis mitpflegen!**
- `docs/wpm4-menue.md` – vollständiger, geräteabgetippter Menübaum des
  WPM4-Displays mit Coverage-Legende (aus der Wissensbasis ausgelagert).
- `docs/hardware.md` – das verbaute Board (Waveshare ESP32-S3-RS485-CAN):
  CAN-Pinout, Transceiver, Terminierung, offizielle Links.
- `docs/manuals/` – lokal gesicherte offizielle Handbücher (WPE-I-Bedienung/
  Installation, ISG-Software-Erweiterung, Board-Schaltbild); die ISG-Modbus-Doku
  liegt als `Modbus Bedienungsanleitung.pdf` im Projekt-Root.
- `archive/wpe-can-sniffer-v1.yaml` / `-v2.yaml` – historische Sniffer
  (MQTT- bzw. ESPHome-Log-basiert), nur Referenz, kein Produktivstand.
- `logs/` ist **nicht im Repo** (per `.gitignore` `*.log` ausgeschlossen).
  Mitschnitte wie `readsensors-verify.log`, in der Doku referenziert,
  liegen nur lokal beim Nutzer.

## Architektur des Manifests (Big Picture)

Das Manifest ist **eine einzige Datei** mit diesen Top-Level-Sektionen
(in Lesereihenfolge): `globals` (u.a. `frame_count`) → `sensor`
(~80 Blöcke) / `text_sensor` → `canbus` mit der `on_frame`-Lambda
(Decoder) → `interval` (Poller) → `button` / `select` / `number`
(Schreib-Entities). Der Datenfluss:

1. **Poller (`interval:`, 60 s):** sendet Lese-**Anfragen** über CAN-ID
   `0x680`, jeweils gestaffelt mit `delay`s.
2. **Decoder (`on_frame:`):** filtert Lese-Anfragen weg
   (`(x[0] & 0x0F) == 0x01`, siehe Sicherheits-/Protokollhinweis),
   zerlegt kurzen/erweiterten Index und Wert, und läuft dann durch eine
   lange `if (idx == … && can_id == …)`-Kette, die per `publish_state`
   die passende Entity füttert.
3. **Schreiben:** `set_action` einer Entity sendet den Write-Frame und
   nach `~250 ms` **sofort eine Lese-Anfrage (Read-back)** desselben
   Index – sonst würde HA den neuen Wert erst beim nächsten 60-s-Poll
   sehen (am Gerät beobachtet).

**Konsequenz für neue Werte:** Ein Lesewert braucht **immer beide
Seiten** – eine Anfrage im `interval:` **und** eine Decode-Zeile in
`on_frame:`. Fehlt eine, bleibt die Entity leer.

### Adressierungs-Formel (Header pro Zielmodul)

Der 2-Byte-Header wird **aus der Ziel-CAN-ID abgeleitet** (kein festes
generisches Präfix):

```
Header-Byte1 = ((can_id >> 3) & 0xF0) | cmd_nibble   (cmd: 1=lesen, 0=schreiben)
Header-Byte2 =  can_id & 0x1F
```

Beispiele: Modul `0x601` → lesen `C1 01` / schreiben `C0 01`;
Manager `0x180` → lesen `31 00` / schreiben `32 00`;
Gesamtsystem `0x480` → lesen `91 00`. Gesendet wird **immer über
`0x680`** (freier PC/ComfortSoft-Kanal), das Ziel steckt nur im Header.

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

**Maßgeblich ist die TODO-/Prioritätensektion oben in
`docs/reverse-engineering.md`** – dort steht der jeweils aktuelle Stand
(bestätigte Lese-/Schreibwerte, offene Ziele, Dashboard-Vorhaben,
PoC-Ergebnis zu `ha-stiebel-control`). Diesen Abschnitt bewusst kurz
halten und nicht mit der Doku doppeln, damit es nur eine Quelle der
Wahrheit gibt.
