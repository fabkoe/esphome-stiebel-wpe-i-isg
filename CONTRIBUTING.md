**🇩🇪 Deutsch** · [🇬🇧 English](CONTRIBUTING.en.md)

# Mitwirken

Beiträge sind willkommen – Issues, Werte-Bestätigungen an anderen WPE-I-/
WPM-Geräten, neue Elster-Indizes, Doku, Dashboard.

## Lizenz & Herkunftsnachweis (DCO)

Das Projekt steht unter der **Apache-2.0-Lizenz** (siehe [`LICENSE`](LICENSE)).
Mit einem Beitrag stimmst du zu, dass er unter derselben Lizenz veröffentlicht
wird.

Wir nutzen den **Developer Certificate of Origin (DCO)** statt eines CLA:
Bestätige die Herkunft deines Beitrags, indem du jeden Commit mit `-s`
signierst (fügt eine `Signed-off-by`-Zeile mit deinem Namen/E-Mail an):

```bash
git commit -s -m "…"
```

Der DCO-Text: https://developercertificate.org/ – kurz: du versicherst, dass du
den Beitrag beisteuern darfst und er unter die Projektlizenz gestellt werden
darf.

## Sicherheit zuerst (wichtig)

Am Gerät hängen echte Kosten: falsche **Schreibwerte** können laut
Betriebsanleitung Wärmepumpe oder Estrich beschädigen.

- Neue **Schreib**-Formate erst nach Verifikation am echten Gerät
  (Log **und** Display-Abgleich) als „bestätigt" markieren.
- Neue Schreib-Entities immer mit **konservativen** min/max-Grenzen.
- **Not-Betrieb** (Betriebsart 6) niemals aktiv setzen/testen.
- Read-only-/Lese-Beiträge sind unkritisch und der einfachste Einstieg.

Details zum Protokoll und zur bewährten Vorgehensweise:
[`docs/reverse-engineering.md`](docs/reverse-engineering.md).
