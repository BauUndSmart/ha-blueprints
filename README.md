# Home Assistant Blueprints von Bau & Smart

Blueprints, die ich für mein eigenes Haus gebaut habe und die deshalb am lebenden
Patienten entstanden sind. Mehr dazu auf [bauundsmart.com](https://bauundsmart.com) (sooon).

---

## Raumheizung

**Datei:** `automations/raumheizung.yaml`

Eine Instanz pro Raum. Der Blueprint steuert eine `climate`-Entität – nie den Aktor
direkt. Das Takten übernimmt der Regler, egal ob Thermostatkopf, `generic_thermostat`
aus Home-Assistant-Core oder ein PID-Thermostat.

Die aktuelle Version ist noch WIP und wird noch weitere Verbesserungen erhalten. Wer aber die Vorabversion ziehen möchte, have fun:

[![Blueprint importieren](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://github.com/BauUndSmart/ha-blueprints/blob/main/automations/raumheizung.yaml)

### Was er macht

- **Zwei Heiztyp-Pfade:** Fußbodenheizung (träge, zweistufige Fensterlogik) und
  Heizkörper (Fenster wirkt sofort), dazu „nur manuell“ für mechanische Thermostate
- **Stufen:** Komfort, Eco/Nacht, Abwesend, Urlaub und zwei frei benennbare Stufen –
  jede mit Sollwert und optionalem Helfer
- **Modus-Quelle frei wählbar:** ein Auswahl-Helfer, ein `select` oder direkt die
  Alarmanlage. Zu jeder Stufe trägt man ein, bei welchen Zuständen sie gilt
- **Fensterlogik, nach Außentemperatur gestaffelt:** je kälter, desto früher die
  Meldung. Bei der Fußbodenheizung erst ein Hinweis, erst später ein Eingriff
- **Sensor-Wächter mit Notbetrieb:** erkennt tote, veraltete und unplausible Sensoren
- **Befehls-Quittierung:** prüft, ob der Regler den Sollwert übernommen hat, und
  sendet bei Bedarf erneut – besonders nützlich bei Zigbee-Thermostatköpfen
- **Manueller Eingriff:** vier Varianten, von „bis zum nächsten Schaltpunkt“ bis
  „gar nicht zulassen“
- **Meldungen** mit eigener Benachrichtigungs-Aktion und Ruhezeiten

### Prioritätenkette

Die höhere Ebene gewinnt immer. Am Ende steht genau ein Zielzustand.

| Prio | Ebene | Ergebnis |
|---|---|---|
| 1 | Heizperiode aus | Heizung aus bzw. Frostschutz |
| 2 | Fenster offen | gewählte Fensteraktion |
| 3 | Sensor-Wächter | Notbetrieb |
| 4 | Manueller Eingriff | Nutzer-Sollwert bis Ablauf |
| 5 | Hausmodus | Urlaub / Abwesend / Frei 1 / Frei 2 |
| 6 | Zeitplan | Komfort (aktiv) / Eco (inaktiv) |
| 7 | Untergrenze | Frostschutz gilt immer als Minimum |

### Leitprinzipien

1. **Core reicht, HACS wird empfohlen.** Der Blueprint läuft mit reinem Home Assistant.
2. **Wir empfehlen, du entscheidest.** Jede Funktion hat einen Default, fast alles ist
   abschaltbar.
3. **Optional heißt optional.** Fehlende Fensterkontakte brechen nichts – die Logik wird
   übersprungen, bis der Sensor eingetragen ist.
4. **Kein YAML.** Alles über die Oberfläche, bis auf den Regler selbst.
5. **Ausfallsicher.** Jeder Fehlerfall hat ein definiertes Verhalten.

### Voraussetzungen

- Home Assistant **2026.9** oder neuer
- Pro Raum eine `climate`-Entität
- Empfohlen: Companion-App (für Meldungen), Status-Helfer, Außentemperatur-Sensor

### Vorher anlegen

Ein Blueprint kann keine Helfer erzeugen. Je nachdem, was du nutzen willst:

| Helfer | Typ | Wofür |
|---|---|---|
| Heizperiode | `input_boolean` | hausweit, einmal |
| Modus-Quelle | `input_select` | hausweit oder je Wohneinheit |
| Sollwerte | `input_number` | je Stufe, wenn am Dashboard änderbar |
| Zeitplan | `schedule` | je Wohneinheit oder Raum |
| **Status-Helfer** | `input_text`, **Maximallänge 255** | je Raum |

> ⚠️ **Der Status-Helfer braucht Maximallänge 255.** Home Assistant legt `input_text`
> standardmäßig mit 100 Zeichen an – das reicht nicht, und der Status wird dann
> stillschweigend abgeschnitten.

Ausführlich: **[docs/raumheizung.md](docs/raumheizung.md)**

### Weitere Doku

| Datei | Inhalt |
|---|---|
| [docs/raumheizung.md](docs/raumheizung.md) | Einrichtung Feld für Feld, Funktionsweise, FAQ, Grenzen |
| [docs/fussbodenheizung-regler.md](docs/fussbodenheizung-regler.md) | Welcher Regler für die Fußbodenheizung – Core oder PID |
| [docs/ersatzsensor.md](docs/ersatzsensor.md) | Mehrere Temperaturquellen zu einem robusten Sensor verbinden |
| [docs/ausfallsicherheit.md](docs/ausfallsicherheit.md) | Was passiert, wenn Home Assistant selbst ausfällt |
| [docs/datenschutz.md](docs/datenschutz.md) | Mehrparteienhäuser |

---

## Geplant

- **Rollladensteuerung** – nach demselben Muster
- **Lichtsteuerung** – sobald überall Präsenzmelder hängen

Alle drei sollen dieselben hausweiten Helfer und denselben Meldungsweg nutzen.

---

## Lizenz

MIT – siehe [LICENSE](LICENSE). Nutzen, anpassen und weitergeben ausdrücklich erwünscht.

## Haftungsausschluss

Die Blueprints steuern eine Heizung. Fehlfunktionen können zu ausgekühlten oder
überheizten Räumen führen. Nutzung auf eigene Verantwortung – prüfe die Einstellungen,
bevor du dich im Winter darauf verlässt, und sichere dich auf Geräteebene ab
(siehe [docs/ausfallsicherheit.md](docs/ausfallsicherheit.md)).
