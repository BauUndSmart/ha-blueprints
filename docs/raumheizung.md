# Raumheizung – Einrichtung und Funktionsweise

Zurück zur [Übersicht](../README.md)

---

## Inhalt

1. [Grundidee](#grundidee)
2. [Vor dem Import: Helfer anlegen](#vor-dem-import-helfer-anlegen)
3. [Der Einrichtungsdialog, Feld für Feld](#der-einrichtungsdialog-feld-für-feld)
4. [Wie die Logik entscheidet](#wie-die-logik-entscheidet)
5. [Fensterlogik im Detail](#fensterlogik-im-detail)
6. [Sensor-Wächter und Notbetrieb](#sensor-wächter-und-notbetrieb)
7. [Manueller Eingriff](#manueller-eingriff)
8. [Meldungen](#meldungen)
9. [Beispiele](#beispiele)
10. [Bekannte Grenzen](#bekannte-grenzen)
11. [FAQ](#faq)

---

## Grundidee

**Der Blueprint steuert eine `climate`-Entität, nie den Aktor direkt.**

Das klingt nach einem Detail, ist aber die wichtigste Entscheidung im ganzen Projekt.
Eine Automation ist ein schlechter Regler: Sie kann nicht takten, nicht ausschwingen
und nicht aus der Vergangenheit lernen. Ein Thermostat kann das.

Also übernimmt der Blueprint das *Was* – welcher Sollwert soll jetzt gelten, und darf
überhaupt geheizt werden? Das *Wie* bleibt beim Regler.

Die Folge: Es ist egal, ob dahinter ein Zigbee-Thermostatkopf, ein
`generic_thermostat` aus Core oder ein PID-Thermostat aus HACS steckt. Der Blueprint
funktioniert mit allem, was eine `climate`-Entität hat.

**Eine Instanz pro Regelzone.** Meist ist das ein Raum. Manchmal nicht: Wenn ein Regler
zwei Heizkreise schaltet, die zusammen einen durchgehenden Bereich heizen (bei mir
Küche und Eingangsflur), ist das *eine* Zone und damit *eine* Instanz.

---

## Vor dem Import: Helfer anlegen

Ein Blueprint kann keine Helfer erzeugen. Alles unten ist optional – lass weg, was du
nicht brauchst.

### Hausweit, einmal für alle Räume

| Helfer | Typ | Beispiel |
|---|---|---|
| Heizperiode | `input_boolean` | „Heizperiode" |
| Modus-Quelle | `input_select` | „Hausmodus" mit Zuhause / Nacht / Abwesend / Urlaub |

> Du hast schon eine Alarmanlage? Dann brauchst du keine eigene Modus-Quelle. Trag
> stattdessen die `alarm_control_panel`-Entität ein, siehe
> [Beispiel 3](#beispiel-3-alarmanlage-als-modus-quelle).

### Je Wohneinheit

Hast du mehrere Wohneinheiten, bekommt jede eine eigene Modus-Quelle und einen eigenen
Zeitplan. Die Räume einer Einheit wählen dann denselben Helfer aus – und schon regelt
jede Wohnung für sich.

### Je Raum

| Helfer | Typ | Wofür |
|---|---|---|
| Sollwerte | `input_number` | nur, wenn jemand sie am Dashboard oder Panel ändern soll |
| **Status-Helfer** | `input_text` | **Maximallänge auf 255 setzen** |

### Der Status-Helfer

Eine Automation hat zwischen zwei Durchläufen kein Gedächtnis. Der Status-Helfer ist
ihr Notizzettel. Er merkt sich:

- die letzte Stufe aus der Modus-Quelle
- einen laufenden manuellen Eingriff und wie lange er gilt
- die Temperatur beim Öffnen des Fensters (für die Abfall-Erkennung)
- welche Meldung zuletzt ging (damit sie sich nicht wiederholt)

**Ohne Status-Helfer läuft alles**, der Raum fängt nach einem Neustart nur „frisch" an:
Ein manueller Eingriff ist weg, und die Fenster-Abfallerkennung beginnt neu.

> ⚠️ **Maximallänge 255.** Home Assistant legt `input_text` standardmäßig mit
> **100 Zeichen** an. Der Status braucht etwa 120. Bei 100 wird er stillschweigend
> abgeschnitten und ist dann unbrauchbar.
>
> *Einstellungen → Geräte & Dienste → Helfer → Erstellen → Text* →
> unter „Erweiterte Einstellungen" die Maximallänge auf **255** setzen.

Den Inhalt sollst du nicht bearbeiten. Lesen lohnt sich aber: Daran siehst du bei der
Fehlersuche sofort, *warum* ein Raum gerade tut, was er tut.

```json
{"v":1,"st":"komfort","ov":23.5,"ou":1791560000,"os":"komfort","wo":1791531392,"wt":21.8,"us":0,"n":""}
```

| Schlüssel | Bedeutung |
|---|---|
| `v` | Format-Version |
| `st` | letzte Stufe |
| `ov` / `ou` / `os` | manueller Sollwert / gültig bis / Stufe beim Eingriff |
| `wo` / `wt` | Fenster offen seit / Temperatur beim Öffnen |
| `us` | unbekannter Modus-Zustand seit |
| `n` | zuletzt gesendete Meldung |

---

## Der Einrichtungsdialog, Feld für Feld

### 1 · Raum & Regelung

| Feld | Pflicht | Erklärung |
|---|---|---|
| **Regler** | ja | Die `climate`-Entität. Auch beim Heiztyp „nur manuell" auszuwählen, sie wird dann aber nicht angesteuert. |
| **Heiztyp** | ja | Entscheidet über die Fensterlogik – siehe unten. |
| **Raumtemperatursensor** | nein | Ist-Wert für Wächter und Fenster-Abfallerkennung. Leer = beides aus. |
| **Aktor für Notbetrieb** | nein | **Nur bei Software-Reglern.** Siehe [Notbetrieb](#sensor-wächter-und-notbetrieb). |
| **Status-Helfer** | nein | Siehe oben. |

### 2 · Haus

Diese drei wählst du in *jeder* Instanz gleich aus.

| Feld | Leer bedeutet |
|---|---|
| **Heizperiode** | immer Heizperiode |
| **Außentemperatur** | mittlere Fensterschwelle, fester Notbetriebs-Anteil |
| **Modus-Quelle** | nur Zeitplan, also Komfort und Eco |

### 3 · Stufen & Sollwerte

Für jede Stufe gibt es drei Felder:

- **Sollwert** – die Zahl, die gilt
- **Helfer** – überschreibt die Zahl, wenn gesetzt
- **Zustände der Modus-Quelle** – bei welchen Zuständen die Stufe greift

| Stufe | Default | Typische Zustände |
|---|---|---|
| Komfort | 21 °C | `Zuhause` · `disarmed`, `armed_home` |
| Eco/Nacht | 18 °C | `Nacht` · `armed_night` |
| Abwesend | 17 °C | `Abwesend` · `armed_away` |
| Urlaub | 15 °C | `Urlaub` · `armed_vacation` |
| Frei 1 / Frei 2 | 20 °C | frei, z. B. `Gäste` – inaktiv, solange keine Zustände eingetragen sind |
| Frostschutz | 7 °C | – (Untergrenze und Fensteraktion) |

> **Schreibweise beachten.** Die Zustände müssen exakt passen, inklusive Groß- und
> Kleinschreibung. Prüfen in *Entwicklerwerkzeuge → Zustände*.

**Unbekannter Zustand.** Steht die Modus-Quelle auf etwas, das in keiner Liste steht
(z. B. `arming` bei einer Alarmanlage), hält der Blueprint die letzte Stufe. Dauert das
länger als das Zeitlimit, greift die Ersatzstufe und es gibt eine Meldung.

Warum die letzte Stufe und nicht einfach Eco? Weil Zwischenzustände normal sind und
die Heizung nicht bei jedem Scharfschalten springen soll. Wichtig: „Letzte Stufe"
meint nur die Stufen der Modus-Quelle. Ein Fenster-Frostschutz kann dort nie hängen
bleiben.

**Sonderbereich.** Standard begrenzt alle Sollwerte auf 5–30 °C. Das Opt-in öffnet
0–45 °C für Spezialfälle wie Terrarium, Anzucht oder frostfreies Gewächshaus. Es hebt
damit auch den Schutz gegen Zahlendreher auf.

> Bei Fußbodenheizung sind Raumsollwerte über etwa 26 °C praktisch nicht erreichbar.
> Die zulässige Oberflächentemperatur im Aufenthaltsbereich liegt bei rund 29 °C.

### 4 · Zeitplan

| Variante | Kann Wochentage | Wann |
|---|---|---|
| **Zeitplan-Helfer** | ja | der Normalfall |
| **Uhrzeit-Helfer** | nein | wenn es jeden Tag gleich ist |
| beides leer | – | dauerhaft Komfort |

Sind beide gesetzt, gewinnt der Zeitplan-Helfer.

### 5 · Fenster

| Feld | Default | Gilt für |
|---|---|---|
| Fenstersensoren | – | alle, mehrere möglich („eines offen" = offen) |
| Fensteraktion | Frostschutz | alle |
| Öffnungs-/Schließverzögerung | 60 s / 30 s | Heizkörper |
| Hinweisschwellen (3×) | 30 / 15 / 10 min | Fußbodenheizung |
| Hinweis bei Abfall | 1,5 K | Fußbodenheizung |
| Eingriff: Faktor | 2× | Fußbodenheizung |
| Eingriff bei Abfall | 3,0 K | Fußbodenheizung |

**Fensteraktion „nur melden"** ist richtig bei mechanischen Thermostaten und bei
Köpfen mit eigener Fenster-auf-Erkennung – sonst arbeiten zwei Systeme gegeneinander.

### 6 · Manueller Eingriff

| Variante | Wann sinnvoll |
|---|---|
| Bis zum nächsten Zeitplan-Schaltpunkt (Default) | der Normalfall |
| Feste Dauer | wenn ein Eingriff verlässlich ablaufen soll |
| Bis zum nächsten Moduswechsel | wenn er bis zum Verlassen des Hauses gelten soll |
| Nicht zulassen | vermietete Einheiten, Räume ohne Eingriffsrecht |

### 7 · Sensor-Wächter & Notbetrieb

| Feld | Default | Hinweis |
|---|---|---|
| Sensor ausgefallen nach | 3 h | Zigbee-Sensoren melden oft nur bei Änderung – nicht zu knapp |
| Plausibel von / bis | 5 / 35 °C | außerhalb gilt der Sensor als defekt |
| Notbetrieb: Taktzyklus | 30 min | nur mit Notbetrieb-Aktor |
| Regler nicht erreichbar: melden nach | 15 min | ignoriert kurze Funk-Aussetzer |
| Befehls-Quittierung | an | siehe unten |

### 8 · Meldungen

Statt eines festen Ziels trägst du hier **deine eigene Benachrichtigungs-Aktion** ein.
Damit funktioniert der Blueprint mit der Companion-App genauso wie mit Telegram,
Signal, einem Sprachausgabe-Skript oder einer dauerhaften Meldung im Dashboard.

Verfügbare Variablen:

| Variable | Inhalt |
|---|---|
| `meldung` | der Text |
| `meldung_titel` | „Heizung <Raumname>" |
| `meldung_kritisch` | `true` / `false` |
| `raumname` | der Bereichsname des Reglers |

Beispiel für die Companion-App:

```yaml
action: notify.mobile_app_dein_telefon
data:
  title: "{{ meldung_titel }}"
  message: "{{ meldung }}"
```

---

## Wie die Logik entscheidet

Bei jedem Durchlauf wird die ganze Kette neu bewertet. Die höhere Ebene gewinnt.

| Prio | Ebene | Ergebnis |
|---|---|---|
| 1 | Heizperiode aus | Heizung aus, sonst Frostschutz |
| 2 | Fenster offen | gewählte Fensteraktion |
| 3 | Sensor-Wächter | Notbetrieb |
| 4 | Manueller Eingriff | Nutzer-Sollwert bis Ablauf |
| 5 | Hausmodus | Urlaub / Abwesend / Frei 1 / Frei 2 |
| 6 | Zeitplan | Komfort / Eco |
| 7 | Untergrenze | Frostschutz als Minimum |

**Fenster steht bewusst über dem Sensor-Wächter.** Bei offenem Fenster soll auch im
Notbetrieb nicht geheizt werden.

**Der Blueprint schreibt nur, wenn sich etwas ändert.** Weicht der Sollwert am Regler
um weniger als 0,1 K vom Ziel ab, passiert nichts. Das schont Batterien und Funknetz.

### Wann er läuft

- beim Start von Home Assistant und nach einem Neuladen der Automationen
- bei jeder Änderung an Regler, Sensor, Fenster, Modus-Quelle, Zeitplan, Heizperiode
  oder einem Sollwert-Helfer
- alle 15 Minuten zur Sicherheit – das heilt verpasste Ereignisse

---

## Fensterlogik im Detail

### Heizkörper

Fenster auf, nach der Öffnungsverzögerung greift die Fensteraktion. Fenster zu, nach
der Schließverzögerung wird die Kette neu bewertet – kein blindes „zurück auf vorher".

Die Verzögerungen filtern kurzes Öffnen und zitternde Kontakte.

### Fußbodenheizung

Die Fußbodenheizung wegen zehn Minuten Stoßlüften abzuschalten bringt nichts, dafür
ist sie viel zu träge.

**Das eigentliche Problem ist ein anderes:** Der Regler sieht die Temperatur fallen und
heizt stärker. Bei einem PID-Regler läuft der Integral-Anteil hoch. Machst du das
Fenster zu, überheizt der Raum – und zwar stundenlang, weil der Estrich die Wärme erst
abgeben muss.

Deshalb zwei Stufen:

| Stufe | Bedingung | Folge |
|---|---|---|
| **Hinweis** | Fenster offen **und** Heizung aktiv **und** (Zeitschwelle **oder** 1,5 K Abfall) | nur Meldung |
| **Eingriff** | doppelte Zeitschwelle **oder** 3,0 K Abfall | Fensteraktion greift |

### Warum die Zeitschwelle von der Außentemperatur abhängt

Eine physikalische Berechnung aus Raumgröße, Estrichträgheit und Luftwechsel wäre
Overengineering – die Eingangsdaten hat niemand sauber. Die Raumgröße ist auch
weniger entscheidend, als man denkt: Die Luft ist in wenigen Minuten getauscht, was
dauert, ist das Auskühlen von Wänden und Boden.

Was wirklich zählt, sind zwei messbare Größen:

1. **Wie kalt ist es draußen.** Je kälter, desto schneller wirkt das Lüften und desto
   teurer wird es: über 10 °C 30 min, 0–10 °C 15 min, unter 0 °C 10 min.
2. **Wie stark fällt die Temperatur wirklich.** Das misst der Raum selbst und deckt
   Größe und Trägheit automatisch ab.

Was zuerst eintritt, gilt.

---

## Sensor-Wächter und Notbetrieb

Ein Sensor gilt als ausgefallen, wenn er:

- `unavailable` oder `unknown` ist,
- seit mehr als X Stunden nichts gemeldet hat, **oder**
- einen unplausiblen Wert liefert (außerhalb 5–35 °C).

### Was dann passiert, hängt vom Reglertyp ab

Der Blueprint erkennt den Typ **ohne Zusatzfeld**:

> Ist ein „Aktor für Notbetrieb" eingetragen, gilt der Regler als **Software-Regler**.
> Ohne Aktor gilt er als **Gerät**.

| Fall | Situation | Reaktion |
|---|---|---|
| A | Sensor weg, Regler ist ein **Gerät** | Der Kopf regelt über seinen internen Sensor weiter · Meldung |
| B | Sensor weg, **Software-Regler** mit eigener Schutzfunktion | Der Regler macht den Notbetrieb selbst · Meldung |
| C | Sensor weg, **Software-Regler**, Aktor eingetragen | Blueprint schaltet den Regler aus und taktet den Aktor · Meldung |
| D | **Regler** nicht erreichbar | Nach der Meldefrist eine kritische Meldung |
| E | **Software-Regler** `unavailable`, Aktor erreichbar | Takten · Meldung |
| F | **Aktor** nicht erreichbar | Keine Aktion möglich · Meldung |

### Kennlinie für den Taktbetrieb

Zyklus 30 min (einstellbar), linear interpoliert:

| Außen | ≥ 15 °C | 10 °C | 5 °C | 0 °C | −5 °C | ≤ −10 °C |
|---|---|---|---|---|---|---|
| Einschaltanteil | 0 % | 15 % | 25 % | 35 % | 42 % | 50 % |

Ohne Außensensor: fest 25 %.

### Befehls-Quittierung

Zigbee-Thermostatköpfe schlafen viel und verschlucken gern einen Befehl. Mit
eingeschalteter Quittierung prüft der Blueprint nach jedem Sollwert-Befehl, ob der
Regler ihn übernommen hat. Wenn nicht, sendet er nach 2 Minuten erneut und wartet
weitere 3.

---

## Manueller Eingriff

Dreht jemand am Thermostatkopf, am Panel oder im Dashboard, merkt der Blueprint das und
respektiert es.

Technisch unterscheidet er die eigene Änderung von einer fremden am **Kontext** der
Zustandsänderung. Änderungen, die die Automation selbst ausgelöst hat, zählen nicht als
manueller Eingriff.

**Ein Eingriff wird überstimmt von:** Fensteraktion, Notbetrieb und Heizperiode. Nach
deren Ende gilt wieder der normale Zielwert.

> Ohne Status-Helfer überlebt ein Eingriff keinen Neustart.

---

## Meldungen

| Meldung | Kritisch |
|---|---|
| Fenster offen und Fußbodenheizung läuft (Hinweis) | nein |
| Fensteraktion hat gegriffen | nein |
| Fenster offen bei unter 0 °C außen | **ja** |
| Sensor ausgefallen, Notbetrieb aktiv | **ja** |
| Modus-Quelle unbekannt, Ersatzstufe greift | **ja** |
| Regler nicht erreichbar | **ja** |

In der Ruhezeit kommt nur Kritisches durch. Jede Meldung wird nur einmal gesendet,
bis sich die Lage ändert – es gibt also keine Dauerbeschallung.

---

## Beispiele

### Beispiel 1: Fußbodenheizung mit PID

```
Regler:        climate.wohnzimmer_fbh_pid
Heiztyp:       Fußbodenheizung
Raumsensor:    sensor.wohnzimmer_temperatur_mittel
Notbetriebs-Aktor: switch.fussbodenheizung_wohnzimmer   ← Software-Regler!
Status-Helfer: input_text.heizung_status_wohnzimmer
Modus-Quelle:  input_select.hausmodus
  Komfort:  Zuhause          21 °C
  Eco:      Nacht            18 °C
  Abwesend: Abwesend         17 °C
  Urlaub:   Urlaub           15 °C
Fenster:       binary_sensor.wohnzimmer_strasse, binary_sensor.wohnzimmer_terasse
Fensteraktion: Frostschutz
```

### Beispiel 2: Heizkörper mit Zigbee-Thermostatkopf

```
Regler:        climate.schlafzimmer_trv
Heiztyp:       Heizkörper
Raumsensor:    sensor.schlafzimmer_temperatur
Notbetriebs-Aktor: (leer)                              ← Gerät!
Fenster:       binary_sensor.schlafzimmer_fenster
Fensteraktion: Frostschutz
Quittierung:   an
```

### Beispiel 3: Alarmanlage als Modus-Quelle

```
Modus-Quelle:  alarm_control_panel.haus
  Komfort:  disarmed, armed_home
  Eco:      armed_night
  Abwesend: armed_away
  Urlaub:   armed_vacation
  Frei 1:   armed_custom_bypass           (z. B. „Gäste")
Unbekannter Zustand: letzte Stufe, Limit 60 min, Ersatzstufe Eco
```

`arming`, `pending` und `triggered` stehen in keiner Liste – damit fängt die
Letzte-Stufe-Regel genau die Zwischenzustände ab.

### Beispiel 4: Handtuchheizkörper ohne smarten Aktor

```
Regler:        climate.bad_handtuch   (wird nicht angesteuert)
Heiztyp:       Nur manuell
Fenster:       binary_sensor.bad_fenster
Fensteraktion: Nur melden
```

---

## Bekannte Grenzen

- **Ohne Status-Helfer gibt es kein Gedächtnis über einen Neustart hinaus.**
- **Der Blueprint kann Home Assistant nicht ersetzen.** Fällt HA aus, regelt niemand
  mehr – siehe [ausfallsicherheit.md](ausfallsicherheit.md).
- **Ein externer Sensor im Thermostatkopf** (eingespeist über Zigbee2MQTT oder
  Better Thermostat) muss auf *dieser* Ebene einen Rückfall auf den internen Sensor
  bekommen. Der Blueprint kann das nicht für alle Köpfe generisch lösen.
- **Die Uhrzeit-Variante des Zeitplans kennt keine Wochentage.** Bewusst so – wer den
  Unterschied braucht, nimmt den Zeitplan-Helfer.
- **Nur Heizen.** Kühlen ist nicht abgedeckt.

---

## FAQ

**Ich habe noch keine Fensterkontakte. Kann ich trotzdem starten?**
Ja. Lass das Feld leer. Die ganze Fensterlogik wird dann übersprungen. Sobald die
Sensoren da sind, trägst du sie nach – mehr ist nicht nötig.

**Muss ich für jeden Raum einen Zeitplan anlegen?**
Nein. Ein Zeitplan kann von beliebig vielen Räumen ausgewählt werden.

**Was, wenn mein Thermostatkopf schon eine eigene Fenstererkennung hat?**
Fensteraktion auf „nur melden" stellen. Sonst arbeiten zwei Systeme gegeneinander.

**Warum steht der Regler manchmal auf `heat`, heizt aber nicht?**
Weil die Raumtemperatur über dem Sollwert liegt. `heat` heißt „bereit zu heizen", nicht
„heizt gerade". Ob gerade geheizt wird, steht im Attribut `hvac_action`.

**Kann ich mehrere Heizkörper in einem Raum steuern?**
Ja, auf zwei Wegen: entweder eine Instanz pro Kopf, oder du fasst die Köpfe vorher
zu einer Gruppe zusammen und steuerst die Gruppe mit einer Instanz.

**Der Blueprint überschreibt meine Änderung am Kopf sofort wieder.**
Dann steht der manuelle Eingriff auf „nicht zulassen", oder es greift gerade eine
höhere Priorität: Fenster offen, Heizperiode aus oder Notbetrieb.

**Wie finde ich heraus, warum ein Raum gerade tut, was er tut?**
Status-Helfer anschauen. Darin steht die aktive Stufe, ein laufender Eingriff und
die Fensterzeit. Dazu die Ablaufverfolgung der Automation (*Automation → ⋮ →
Ablaufverfolgung*).
