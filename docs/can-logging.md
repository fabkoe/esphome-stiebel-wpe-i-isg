# CAN-Langzeit-Mitschnitt für Reverse-Engineering (MQTT → InfluxDB → Grafana)

Ziel: CAN-Nachrichten und **Wertänderungen über längere Zeit** aufzeichnen und
auswerten, um neue Elster-Indizes zu entschlüsseln. Der ESP32 schreibt **keine**
Datenbank selbst (zu wenig Flash/RAM, Flash-Verschleiß) – er **streamt** die
Frames per MQTT raus, ein Host speichert sie in einer Zeitreihen-DB.

```
 WPM4-CAN ──► ESP32 (ESPHome)
                 │  on_frame-Lambda dekodiert idx/val
                 │  RE-Sniffer (optional, #ifdef USE_MQTT)
                 ▼
            MQTT  Topic: wpe-i/can   (InfluxDB line protocol)
                 ▼
            Telegraf (mqtt_consumer, data_format=influx)
                 ▼
            InfluxDB v2  Bucket: wpe_i
                 ▼
            Grafana  (Verläufe, Änderungs-Tabelle)
```

Der native ESPHome-API-Kanal zu Home Assistant bleibt parallel aktiv – die
bekannten Sensoren landen weiterhin normal in HA. MQTT ist **nur** der
RE-Firehose.

## Zwei Modi

| Switch (in HA) | Was wird gesendet | Standard | Last |
|---|---|---|---|
| **RE Sniffer (changes)** | eine Zeile pro Index, **nur wenn sich der Wert ändert** | AN | gering |
| **RE Sniffer RAW (all frames)** | eine Zeile für **jeden** wertführenden Frame | AUS | hoch – nur kurze, gezielte Captures |

„Changes" ist der Alltagsmodus fürs RE: Du änderst am Display einen Wert und
siehst in Grafana sofort, welcher `idx`/`can_id` sich bewegt hat. RAW ist für
vollständige Mitschnitte kurzer Zeitfenster.

> Hinweis: Lese-**Anfragen** (Cmd-Halbbyte `1`, Wertefeld `0`) werden – wie im
> Haupt-Parser – vor dem Sniffer herausgefiltert. Der Mitschnitt enthält also
> nur wertführende Frames (Meldungen/Broadcasts/Schreibbefehle).

## 1. Firmware aktivieren

1. `mqtt_*`-Schlüssel in `esphome/secrets.yaml` eintragen (Vorlage:
   `secrets.yaml.example`).
2. Im lokalen Manifest die Sniffer-Zeile einkommentieren:
   ```yaml
   packages:
     wpe_i:         !include wpe-i-package.yaml
     wpe_i_writes:  !include wpe-i-writes.yaml
     wpe_i_sniffer: !include wpe-i-sniffer.yaml
   ```
3. Flashen: `py -3.12 -m esphome run wpe-i-manifest.yaml`.

Ohne diesen Include ist **kein** MQTT im Build und der Sniffer-Codeblock wird
per `#ifdef USE_MQTT` komplett verworfen – der Release-/Adopt-Build bleibt
unverändert.

## 2. Host-Stack (Home-Assistant-Add-ons)

Alle vier als HA-Add-on verfügbar (oder standalone/Docker):

- **Mosquitto broker** – MQTT-Broker (liefert `mqtt_*`-Zugangsdaten).
- **InfluxDB** – Zeitreihen-DB; Org + Bucket `wpe_i` + API-Token anlegen.
- **Telegraf** – Brücke MQTT → InfluxDB. Config: `tools/telegraf-wpe-i.conf`
  (Platzhalter `YOUR_*` ersetzen). Da der ESP bereits Line-Protocol sendet,
  genügt `data_format = "influx"` – kein Feld-/Tag-Mapping nötig.
- **Grafana** – Visualisierung; InfluxDB als Datenquelle einbinden.

## 3. MQTT-Payload / Schema

Topic: `wpe-i/can`, ein Frame = eine Zeile (InfluxDB line protocol):

```
wpe_can,can_id=180,idx=000C val=277i,delta=2i,cmd=34i,changed=1i,hex="22000C0115..."
```

- **Tags:** `can_id`, `idx` (jeweils hex, ohne `0x`)
- **Fields:** `val` (int16), `delta` (val − vorheriger Wert), `cmd` (rohes
  Byte 0), `changed` (1/0), `hex` (Rohframe)
- **Zeitstempel:** keiner im Payload → Telegraf/InfluxDB stempeln die
  Empfangszeit (Bus-Latenz vernachlässigbar).

## 4. Grafana – Startpunkte (Flux)

Alle Änderungen der letzten Stunde, gruppiert nach Index:
```flux
from(bucket: "wpe_i")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "wpe_can" and r._field == "val")
  |> group(columns: ["can_id", "idx"])
```

„Welcher Index hat sich gerade bewegt?" (Tabelle, nach Zeit sortiert):
```flux
from(bucket: "wpe_i")
  |> range(start: -15m)
  |> filter(fn: (r) => r._measurement == "wpe_can" and r._field == "delta")
  |> filter(fn: (r) => r._value != 0)
  |> sort(columns: ["_time"], desc: true)
```

## 5. RE-Workflow

1. RAW **kurz** an, alles andere ruhig lassen, am Display **einen** Wert ändern.
2. In der Grafana-Tabelle den `idx`/`can_id` mit passendem `delta` ablesen.
3. Kandidaten in `reverse-engineering.md` eintragen, Skalierung (`/10`, `/100`)
   aus dem Displaywert ableiten, dann als Sensor ins Manifest mappen.
4. RAW wieder aus, „changes" reicht für die Dauerbeobachtung.

## 6. Womit anfangen – konkrete RE-Ziele

Aus den offenen Punkten in [`reverse-engineering.md`](reverse-engineering.md).
Vorgehen jeweils: „RE Sniffer (changes)" an lassen, ggf. für ein enges
Zeitfenster RAW an, den genannten **Menüpfad** am WPM-Display aufrufen bzw. den
Wert ändern, dann in der Grafana-„delta≠0"-Tabelle den `idx`/`can_id` ablesen.

**Gut zum Einstieg (Kandidat/CAN-ID bereits bekannt):**

| Ziel | Kandidat | Vorgehen mit dem Firehose |
|---|---|---|
| Mischermodul-Wert (vmtl. Vorlauf HK2) | `idx=4EB4` auf `can_id=601` (~19,1–19,2 °C) | direkt in Grafana filtern und mit HK2-Vorlauf am Display vergleichen |
| Offener 0x401-Wert | `idx=4EB3` auf `401`/`100` (val=1) | beobachten, wann er kippt; Bedeutung aus Kontext |
| Ist-/Solldrehzahl Verdichter | `idx=06EB` / `06EC` | **bei laufendem** Verdichter mitschneiden (im Test standen beide auf 0) |
| Leistung Kühlen-Grundeinst. | `idx=7A40` auf Modul `480` | Lesewert bestätigen; Schreib-Modul separat per No-Op einkreisen |

**Neue Werte finden (Kandidat noch unbekannt → RAW + Menü-Klick):**

- **Kühlen-Live-Werte** – Info→Anlage→Kühlen: Ist/Soll, KK1 Ist/Soll.
- **Taupunkt FET1 / Raum-Taupunkt** (= ISG-Reg 506) – relevant fürs Kühlen.
- **Heizung-Unterwerte** – HK1 Ist/Soll, Vorlauf NHZ, Festwertsoll.
- **Warmwasser-Einstellungen** – Komfort/Eco WW, Hysterese, Stufen.
- **Gemittelte/gedämpfte Außentemperatur** (= C-Punkt „gemittelte AT").
- **Meldungsliste** (14 Einträge) inhaltlich zuordnen – Altmeldung vs. aktiv.

**Abgeleitete Größen (kein neuer Index, aus vorhandenen rechnen):**

- **Spreizung Heizung (ΔT)** = Vorlauf WP (`FDF3`) − Rücklauf (`0016`).
- **Momentane therm. Heizleistung** ≈ `V̇·ΔT·0,0697` – braucht Volumenstrom.
- **Momentaner COP** = P_therm / P_el – **P_el momentan fehlt noch**.

Der Firehose ersetzt hier vor allem das mühsame Foto-/Log-Zeitfenster-Abgleichen:
Der `delta`-Sprung beim Menü-Klick zeigt den Index sofort. Bestätigte Indizes
dann wie gewohnt als Sensor ins Manifest mappen (Skalierung `/10` bzw. `/100`).

## Grenzen / Sicherheit

- Reiner **Lesebetrieb** – der Sniffer sendet nichts auf den CAN-Bus.
- RAW auf dem stark belegten Bus (v. a. `0x100`) erzeugt viele Nachrichten;
  Bucket-Retention in InfluxDB begrenzen, wenn dauerhaft aktiv.
- Kein Persistenzrisiko am Gerät: Es wird nichts auf dem ESP gespeichert.
