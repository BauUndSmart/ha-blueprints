# Raumheizung – Einrichtung und Funktionsweise

Zurück zur [Übersicht](../README.md)

---

## Inhalt

1. [Grundidee](#grundidee)
2. [Vor dem Import: Helfer anlegen](#vor-dem-import-helfer-anlegen)
3. [Der Einrichtungsdialog, Feld für Feld](#der-einrichtungsdialog-feld-für-feld)
4. [Wie die Logik entscheidet](#wie-die-logik-entscheidet)
5. [Sollwert-Profile](#sollwert-profile)
6. [Vorlauf und Nachlauf](#vorlauf-und-nachlauf)
7. [Fensterlogik im Detail](#fensterlogik-im-detail)
8. [Sensor-Wächter und Notbetrieb](#sensor-wächter-und-notbetrieb)
9. [Manueller Eingriff](#manueller-eingriff)
10. [Meldungen](#meldungen)
11. [Beispiele](#beispiele)
12. [Bekannte Grenzen](#bekannte-grenzen)
13. [FAQ](#faq)

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
| **Vorlaufzeit** | nein | Minuten. Siehe [Vorlauf und Nachlauf](#vorlauf-und-nachlauf). 0 = aus. |
| **Vorlaufzeit – Helfer** | nein | `input_number` **in Minuten**. Überschreibt die Zahl darüber. |
| **Nachlauf** | nein | Prozent der Vorlaufzeit. 0 = aus. |
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

### 3 · Temperaturen

Pro Stufe zwei Felder:

- **Sollwert** – die Zahl, die gilt
- **Helfer** – ein `input_number`. Ist er gesetzt, gilt **nur** er und die Zahl wird
  ignoriert. Nie beides, nie addiert.

Dass es den Helfer gibt, ist die wichtigste Einstellung im ganzen Blueprint, und sie
fällt am leichtesten unter den Tisch. Siehe [Sollwert-Profile](#sollwert-profile).

| Stufe | Default |
|---|---|
| Komfort | 21 °C |
| Eco/Nacht | 18 °C |
| Abwesend | 17 °C |
| Urlaub | 15 °C |
| Frei 1 / Frei 2 | 20 °C |
| Frostschutz | 7 °C (Untergrenze und Fensteraktion) |

**Sonderbereich.** Standard begrenzt alle Sollwerte auf 5–30 °C. Das Opt-in öffnet
0–45 °C für Spezialfälle wie Terrarium, Anzucht oder frostfreies Gewächshaus. Es hebt
damit auch den Schutz gegen Zahlendreher auf.

> Bei Fußbodenheizung sind Raumsollwerte über etwa 26 °C praktisch nicht erreichbar.
> Die zulässige Oberflächentemperatur im Aufenthaltsbereich liegt bei rund 29 °C.

### 4 · Stufen-Zuordnung

Dieser Abschnitt ist zugeklappt, und das ist Absicht: In fast jedem Raum steht hier
dasselbe. Trag ihn einmal aus, dann übernimm ihn.

Pro Stufe zwei Felder:

- **Zustände der Modus-Quelle** – bei welchen Zuständen die Stufe greift
- **Kalender-Stichwort** – bei welchem Termin die Stufe greift, siehe
  [Kalender](#kalender)

Beide beantworten dieselbe Frage aus zwei Richtungen: „bei welchem Signal gelte ich".
Der Zustand beschreibt das Jetzt, der Termin einen geplanten Zeitraum.

Lässt du alles leer, gilt nur der Zeitplan – also Komfort und Eco.

| Stufe | Typische Zustände |
|---|---|
| Komfort | `Zuhause` · `disarmed`, `armed_home` |
| Eco/Nacht | `Nacht` · `armed_night` |
| Abwesend | `Abwesend` · `armed_away` |
| Urlaub | `Urlaub` · `armed_vacation` |
| Frei 1 / Frei 2 | frei, z. B. `Gäste` – inaktiv, solange keine Zustände eingetragen sind |

> **Schreibweise beachten.** Die Zustände müssen exakt passen, inklusive Groß- und
> Kleinschreibung. Prüfen in *Entwicklerwerkzeuge → Zustände*.
>
> Und weil dieser Abschnitt in jedem Raum gleich ist: Ein Tippfehler hier ist ein
> Tippfehler in **allen** Räumen. Einmal sorgfältig, dann übernehmen.

**Unbekannter Zustand.** Steht die Modus-Quelle auf etwas, das in keiner Liste steht
(z. B. `arming` bei einer Alarmanlage), hält der Blueprint die letzte Stufe. Dauert das
länger als das Zeitlimit, greift die Ersatzstufe und es gibt eine Meldung.

Warum die letzte Stufe und nicht einfach Eco? Weil Zwischenzustände normal sind und
die Heizung nicht bei jedem Scharfschalten springen soll. Wichtig: „Letzte Stufe"
meint nur die Stufen der Modus-Quelle. Ein Fenster-Frostschutz kann dort nie hängen
bleiben.

### 5 · Zeitplan

| Variante | Kann Wochentage | Wann |
|---|---|---|
| **Zeitplan-Helfer** | ja | der Normalfall |
| **Uhrzeit-Helfer** | nein | wenn es jeden Tag gleich ist |
| beides leer | – | dauerhaft Komfort |

Sind beide gesetzt, gewinnt der Zeitplan-Helfer.

#### Kalender

Ein Zeitplan beantwortet „wann ist es üblicherweise warm". Ein Kalender beantwortet
„wann ist es diesmal anders". Das ist der Unterschied zur Modus-Quelle, die nur das
Jetzt kennt: Ein Termin hat Anfang und Ende, lässt sich Wochen vorher eintragen und
kann von Leuten gepflegt werden, die Home Assistant nie öffnen.

Typische Fälle: eine Feier, Besuch über Weihnachten, eine Dienstreise, oder die
Buchung einer Ferienwohnung.

Welcher Termin welche Stufe auslöst, legst du über die **Kalender-Stichwörter** in
Abschnitt 4 fest. Steht das Stichwort im Titel des Termins, gilt diese Stufe, solange
der Termin läuft. Groß- und Kleinschreibung sind egal, Teiltreffer reichen.

| Stichwörter | Wirkung |
|---|---|
| alle leer | **jeder** Termin im Kalender zählt als Komfort |
| eines gesetzt | nur Termine mit diesem Wort lösen diese Stufe aus |
| mehrere treffen | Urlaub → Abwesend → Frei 1 → Frei 2 → Eco → Komfort, der erste gewinnt |

Der leere Fall ist Absicht: Wer einen eigenen Kalender „Heizung" anlegt, trägt dort
Termine ein und muss sonst nichts konfigurieren.

Ein laufender Termin gewinnt gegen die Modus-Quelle. Heizperiode, Fenster, Notbetrieb
und manueller Eingriff gewinnen weiterhin gegen den Kalender.

> **Grenze:** Eine Kalender-Entität in Home Assistant zeigt immer nur **einen** Termin
> – den laufenden, und wenn keiner läuft, den nächsten. Überlappende Termine lassen
> sich damit nicht auflösen. Wer das braucht, nimmt getrennte Kalender.

### 6 · Fenster

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

### 7 · Manueller Eingriff

| Variante | Wann sinnvoll |
|---|---|
| Bis zum nächsten Zeitplan-Schaltpunkt (Default) | der Normalfall |
| Feste Dauer | wenn ein Eingriff verlässlich ablaufen soll |
| Bis zum nächsten Moduswechsel | wenn er bis zum Verlassen des Hauses gelten soll |
| Nicht zulassen | vermietete Einheiten, Räume ohne Eingriffsrecht |

### 8 · Sensor-Wächter & Notbetrieb

| Feld | Default | Hinweis |
|---|---|---|
| Sensor ausgefallen nach | 3 h | Zigbee-Sensoren melden oft nur bei Änderung – nicht zu knapp |
| Plausibel von / bis | 5 / 35 °C | außerhalb gilt der Sensor als defekt |
| Notbetrieb: Taktzyklus | 30 min | nur mit Notbetrieb-Aktor |
| Regler nicht erreichbar: melden nach | 15 min | ignoriert kurze Funk-Aussetzer |
| Befehls-Quittierung | an | siehe unten |

### 9 · Meldungen

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
| 5 | Kalender | Stufe des laufenden Termins |
| 6 | Hausmodus | Urlaub / Abwesend / Frei 1 / Frei 2 |
| 7 | Zeitplan | Komfort / Eco |
| 8 | Untergrenze | Frostschutz als Minimum |

Vorlauf und Nachlauf stehen **nicht** in dieser Kette. Sie ändern nur, *welche* Stufe
auf Ebene 5 bis 7 gilt – nie, ob Fenster, Notbetrieb oder Heizperiode greifen.

**Fenster steht bewusst über dem Sensor-Wächter.** Bei offenem Fenster soll auch im
Notbetrieb nicht geheizt werden.

**Der Blueprint schreibt nur, wenn sich etwas ändert.** Weicht der Sollwert am Regler
um weniger als 0,1 K vom Ziel ab, passiert nichts. Das schont Batterien und Funknetz.

### Wann er läuft

- beim Start von Home Assistant und nach einem Neuladen der Automationen
- bei jeder Änderung an Regler, Sensor, Fenster, Modus-Quelle, Zeitplan, Kalender,
  Vorlauf-Helfer, Heizperiode
  oder einem Sollwert-Helfer
- alle 15 Minuten zur Sicherheit – das heilt verpasste Ereignisse

---

## Sollwert-Profile

Jede Stufe hat zwei Felder: eine Zahl und einen Helfer. Ein `input_number`-Helfer kann
von **beliebig vielen Räumen gleichzeitig** benutzt werden, und darin steckt der
ganze Trick. Du entscheidest pro Raum und pro Stufe neu, ob der Wert lokal gilt oder
von außen kommt.

### Einfamilienhaus

Vier hausweite Helfer anlegen und in jedem Raum identisch eintragen:

```
input_number.soll_eco          18
input_number.soll_abwesend     17
input_number.soll_urlaub       15
input_number.soll_frostschutz   7
```

Damit bleibt pro Raum **genau eine** Zahl, über die du wirklich nachdenken musst:
Komfort. Schlafzimmer 18, Wohnzimmer 21, Bad 23.

Dass Eco im Schlafzimmer 18,0 und im Wohnzimmer 17,5 sein soll, ist eine Frage, die
sich niemand ernsthaft stellt. Komfort schon.

### Mehrere Wohneinheiten oder Mieter

Pro Einheit ein eigenes Profil:

```
input_number.komfort_eg          21,0
input_number.komfort_wg_oben     20,5
input_number.komfort_oma         23,0
```

Jeder Raum der jeweiligen Einheit zeigt auf seinen Helfer. Der Mieter ruft an und will
ein halbes Grad mehr — du drehst **eine** Zahl, alle seine Räume folgen, die anderen
Einheiten merken nichts.

Für gewerbliche Vermietung funktioniert es genauso: ein Profil pro Mietpartei.

### Einzelne Räume dürfen trotzdem abweichen

Das ist kein Entweder-Oder auf Hausebene. Das Bad einer Einheit darf eine feste 23
haben, während die anderen sieben Räume am gemeinsamen Helfer hängen. Lass dort
einfach das Helfer-Feld leer und trag die Zahl ein.

### ⚠️ Keine Climate-Gruppe als Regler

Es liegt nahe, statt vieler Instanzen eine Gruppe aller Heizkörper einer Einheit als
Regler einzutragen. **Tu das nicht.**

Erstens kann der Group-Helfer in Home Assistant gar keine `climate`-Entitäten – nur
die alten YAML-Gruppen oder eine Zusatzintegration aus HACS.

Zweitens, und das ist der eigentliche Grund: Der Blueprint liest vom Regler die
Ist-Temperatur, braucht einen Raumsensor für den Wächter und hat eigene
Fensterkontakte. **Fünf Heizkörper in fünf Räumen haben keine gemeinsame
Raumtemperatur und kein gemeinsames Fenster.** Die Fensterlogik würde sinnlos, die
Abfallerkennung Unsinn, und der Sensor-Wächter überwachte einen Mittelwert, den es
nirgends gibt.

Der Blueprint steht auf einem Satz: **eine Instanz pro Regelzone.** Ein Mieter ist
keine Regelzone, ein Raum ist eine.

> **Den Sollwert teilen – nie die Instanz.**

### Was sich so nicht teilen lässt

Nur Zahlen. Die Felder in [4 · Stufen-Zuordnung](#4--stufen-zuordnung) sind Textlisten
und damit nicht über eine Entität teilbar — die stehen in jedem Raum einzeln drin.
Deshalb ist dieser Abschnitt zugeklappt und deshalb steht dort die Warnung, ihn einmal
sorgfältig auszufüllen.

---

## Vorlauf und Nachlauf

Eine Fußbodenheizung, die erst um sieben anfängt, ist um sieben nicht warm. Sie ist um
elf warm. Der Vorlauf verschiebt den Start so weit nach vorn, dass der Raum zum
Umschaltzeitpunkt die Temperatur schon hat.

Der Nachlauf ist dasselbe rückwärts: rechtzeitig aufhören und die gespeicherte Wärme
auslaufen lassen, statt bis zur letzten Minute zu heizen und danach wegzuwerfen.

### Woher der Zeitpunkt kommt

Vorlauf und Nachlauf brauchen einen **bekannten Zeitpunkt in der Zukunft**. Es gibt
genau zwei Quellen:

| Quelle | Zeitpunkt | Beispiel |
|---|---|---|
| **Zeitplan-Helfer** | nächster Wechsel | morgen 06:00 von Eco auf Komfort |
| **Kalender** | Beginn des nächsten Termins | Anreise Ferienwohnung |
| **Kalender** | Ende des laufenden Termins | Rückkehr aus dem Urlaub |

Die Modus-Quelle hat keine Zukunft – ein Auswahl-Helfer weiß nicht, dass du morgen
heimkommst. Ohne Zeitplan und ohne Kalender bleibt die Vorlaufzeit deshalb wirkungslos.
Sie ist dann nicht kaputt, es gibt nur nichts, worauf sie vorlaufen könnte.

Stehen beide Quellen an, gewinnt der **frühere** Zeitpunkt. Ist der vorbei, rückt der
nächste nach.

### Wann genau umgeschaltet wird

Der Blueprint vergleicht die Sollwerte der aktuellen und der kommenden Stufe:

| Richtung | Es greift | Fenster |
|---|---|---|
| kommende Stufe ist **wärmer** | Vorlauf | Vorlaufzeit vor dem Zeitpunkt |
| kommende Stufe ist **kälter** | Nachlauf | Nachlaufzeit vor dem Zeitpunkt |
| gleiche Stufe | nichts | – |

Beide können sich nie gleichzeitig auslösen, weil die Richtung eindeutig ist.

### Die Zahl oder der Helfer

Die Vorlaufzeit ist eine Eigenschaft des Raums, nicht des Termins. Grobe Hausnummern:

| Heizung | Vorlauf | Nachlauf bei 50 % |
|---|---|---|
| Estrich-Fußbodenheizung | 360–720 min | 180–360 min |
| Trockenbau-Fußbodenheizung | 120–240 min | 60–120 min |
| Heizkörper | 30–60 min | 15–30 min |

Du kannst die Minuten pro Raum eintragen – oder einen `input_number`-Helfer wählen,
den sich mehrere Räume teilen. Üblich ist einer für alle Fußbodenheizungen und einer
für alle Heizkörper. Dann drehst du an einer Stelle statt an zwanzig.

> ⚠️ **Der Helfer muss in Minuten sein**, nicht in Stunden. Bei den
> Temperatur-Helfern ist °C selbsterklärend, hier nicht.

Ist ein Helfer eingetragen, gilt **nur** der Helfer. Die Zahl im Feld darüber wird
dann ignoriert, nie addiert.

Der Nachlauf ist ein **Prozentsatz der Vorlaufzeit**, keine eigene Zahl. Auskühlen
geht schneller als Aufheizen, und so skaliert er automatisch mit, wenn du am
Vorlauf-Helfer drehst. 0 = aus, 50 ist ein brauchbarer Einstieg.

### Warum das zusammenpasst

Dein Urlaubsfall, Schritt für Schritt:

1. Im Kalender steht „Urlaub" bis Sonntag 14:00.
2. Solange der Termin läuft, gilt die Stufe Urlaub – 15 °C.
3. Der Blueprint sieht das Ende des Termins als Umschaltzeitpunkt und weiß, was danach
   gilt: das, was Modus-Quelle und Zeitplan ohne Kalender ergeben hätten.
4. Bei 720 Minuten Vorlauf beginnt der Estrich Samstag um 02:00 zu heizen.
5. Sonntag 14:00 ist der Raum warm, nicht erst Montag.

Für eine Ferienwohnung läuft es andersherum: Der Buchungszeitraum steht im Kalender,
der Vorlauf sorgt dafür, dass bei Anreise geheizt ist, und zwischen den Buchungen
fällt das Haus auf die Grundstufe zurück.

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

### Türen in der Kontaktliste

Der Abfall-Weg kennt **keine Zeitschwelle**. In einem ruhigen Raum ist das richtig:
1,5 K Abfall heißt, dass jemand ernsthaft lüftet, egal wie lange das Fenster schon auf
ist.

Hängt dagegen eine viel benutzte Tür in der Kontaktliste – Haustür, Terrassentür –,
bedeutet derselbe Abfall nur, dass jemand nach Hause gekommen ist. Bei Frost reichen
dafür zwei Minuten, besonders wenn der Raumsensor in der Nähe hängt.

**In solchen Räumen soll die Zeitschwelle führen.** Setz die beiden Abfallschwellen
hoch, etwa auf **3,0 K** für den Hinweis und **5,0 K** für den Eingriff. Die
Zeitschwellen bleiben, wie sie sind – sie erwischen den Fall, auf den es ankommt: die
Tür, die offen stehen geblieben ist.

### Die Uhr überlebt Neustarts und Funklöcher

Der Blueprint merkt sich den Öffnungszeitpunkt im Status-Helfer (`wo`) und rechnet die
Dauer aus dem **früheren** von zwei Werten: dem gemerkten Zeitpunkt und dem letzten
Zustandswechsel des Kontakts.

Das ist nötig, weil Home Assistant den letzten Zustandswechsel beim Neustart auf die
Startzeit setzt. Ohne das Gedächtnis würde die Eskalation nach jedem Update bei null
anfangen – und ein dauerhaft offenes Fenster erreicht die Eingriffsschwelle dann nie.

Aus demselben Grund gilt ein Kontakt, der `unavailable` oder `unknown` meldet, **nicht
als geschlossen**:

| Zustand der Kontakte | Fenster gilt als | Gedächtnis |
|---|---|---|
| mindestens einer `on` | offen | bleibt, Dauer ab dem früheren Zeitpunkt |
| alle `off` | zu | wird gelöscht |
| einer `unavailable` / `unknown` | nicht offen, aber auch nicht zu | **bleibt erhalten** |

Ein kurzer Funkaussetzer oder ein Neustart löscht die Uhr also nicht mehr. Kommt der
Kontakt als `on` zurück, läuft die Eskalation vom echten Öffnungszeitpunkt weiter.

> **Der Preis dafür:** Wird das Fenster geschlossen *und* wieder geöffnet, während der
> Kontakt nicht erreichbar war, zählt der Blueprint zu lange. Das ist die bewusst
> gewählte Richtung – lieber einmal zu früh melden als eine offene Tür im Januar
> übersehen.

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

### Beispiel 5: Ferienwohnung nach Buchungskalender

Zwischen den Gästen steht das Haus auf 15 °C. Zur Anreise soll es warm sein, ohne
dass jemand daran denken muss.

```
Regler:              climate.fewo_wohnzimmer
Heiztyp:             Fußbodenheizung
Vorlaufzeit-Helfer:  input_number.vorlauf_fussbodenheizung   (720 = 12 Stunden)
Nachlauf:            50 %

Kalender:            calendar.buchungen
Komfort-Stichwort:   (leer – jede Buchung zählt)
Urlaub – Sollwert:   15 °C
Modus-Quelle:        (leer)
Zeitplan:            (leer)
Ersatzstufe:         Urlaub
```

Weil keine Modus-Quelle und kein Zeitplan gesetzt sind, fällt das Haus außerhalb der
Buchungen auf die Grundstufe zurück. Ein laufender Buchungstermin hebt es auf
Komfort, und der Vorlauf startet den Estrich zwölf Stunden vor Anreise.

Beim Auschecken greift derselbe Mechanismus rückwärts: Sechs Stunden vor Buchungsende
hört die Heizung auf und die Restwärme reicht bis zur Abreise.

> Trägt der Vermieter die Buchungen ohnehin in einen Kalender ein, den Home Assistant
> lesen kann, ist das die komplette Konfiguration. Siehe auch
> [datenschutz.md](datenschutz.md), wenn fremde Haushalte mit in der Installation
> hängen.

---

## Bekannte Grenzen

- **Ohne Status-Helfer gibt es kein Gedächtnis über einen Neustart hinaus.** Das trifft
  auch die Fensteruhr: Sie fängt dann nach jedem Neustart wieder bei null an. Wer
  Updates regelmäßig einspielt, sollte den Helfer anlegen.
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
