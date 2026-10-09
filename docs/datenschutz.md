# Datenschutz in Mehrparteienhäusern

Zurück zur [Übersicht](../README.md) · [Hauptdoku](raumheizung.md)

---

Steuerst du nur deine eigene Wohnung, ist dieses Kapitel für dich nicht relevant.

Sobald aber **andere Haushalte** mit in der Installation hängen – eine
Einliegerwohnung, Mieter, Angehörige mit eigenem Hausstand – ändert sich die Lage.
Dann verarbeitest du Daten über Menschen, die nicht du sind.

*Hinweis: Das hier ist eine praktische Einordnung aus Betreibersicht, keine
Rechtsberatung.*

---

## Was die Heizungssteuerung über Menschen verrät

Die erfassten Daten wirken harmlos. In der Zeitreihe sind sie es nicht:

| Datum | Was man daraus ablesen kann |
|---|---|
| Fensterkontakte | wann jemand aufsteht, lüftet, das Haus verlässt |
| Raumtemperatur | wann jemand zu Hause ist, wann geschlafen wird |
| Manuelle Eingriffe am Thermostat | wann jemand aktiv ist, welche Räume genutzt werden |
| Heizzeiten | der komplette Tagesrhythmus |

Aus Fensterkontakt plus Temperaturverlauf lässt sich ein ziemlich genaues
Anwesenheitsprofil ableiten. Dafür muss niemand böswillig sein – es entsteht
nebenbei.

Hinzu kommt: Als Administrator kannst du jeden einzelnen Wert ohnehin einsehen. Das
lässt sich technisch kaum verhindern. Umso wichtiger ist, wie man damit umgeht.

---

## Was ich empfehle

### 1. Transparenz statt technischer Sperre

Sag den Bewohnern, was erfasst wird und wofür. Das ist der wirksamste Punkt, und es
ist ein leichtes Gespräch: Es geht um Heizkosten und Frostschutz, nicht um Kontrolle.

Was dazugehört:

- welche Sensoren in ihrer Wohnung hängen
- dass Temperatur und Fensterzustand aufgezeichnet werden
- wofür das genutzt wird
- dass sie es ablehnen können

### 2. Meldungen an die richtigen Leute

Der Blueprint hat ein eigenes Meldeziel pro Instanz. Nutze das: Die Meldung „Fenster
offen" aus einer fremden Wohnung gehört an deren Bewohner, nicht an dich.

Beim Betreiber landen nur Dinge, die das Haus betreffen: Regler nicht erreichbar,
Sensor ausgefallen, Frostgefahr.

### 3. Aufbewahrungsdauer bewusst wählen

Home Assistant speichert standardmäßig 10 Tage. Für die Heizungssteuerung selbst
reicht weit weniger – der Blueprint braucht überhaupt keine Historie, er arbeitet
immer mit dem aktuellen Zustand.

Für Fensterkontakte fremder Wohnungen kannst du die Aufzeichnung deutlich kürzen oder
ganz abschalten:

```yaml
recorder:
  exclude:
    entities:
      - binary_sensor.wg2_fenster_wohnzimmer
```

Die Steuerung funktioniert weiter, nur die Langzeithistorie entfällt.

### 4. Dashboards trennen

Wenn Bewohner eigene HA-Zugänge haben, bekommen sie ein Dashboard mit ihren eigenen
Räumen. Das ist auch praktischer als eine Übersicht über das ganze Haus.

### 5. Keine Heizkostenabrechnung daraus bauen

Home Assistant kann Verbräuche mitloggen, und das ist für eigene Auswertungen
praktisch. Eine rechtlich belastbare Heizkostenabrechnung ersetzt es nicht – dafür
braucht es geeichte Zähler und ein geregeltes Verfahren.

---

## Was der Blueprint selbst speichert

Nur zwei Dinge, und beide liegen in Home Assistant:

1. **Den Status-Helfer** mit Stufe, Override und Fensterzeit. Das ist ein
   Momentzustand, keine Historie.
2. **Was der Recorder ohnehin aufzeichnet** – also genau das, was du oben
   konfigurierst.

Es gibt keine Cloud, keine Telemetrie, keinen externen Dienst.

---

## Kurzfassung

| | |
|---|---|
| Bewohner informieren | ✅ immer |
| Meldungen an die jeweilige Einheit | ✅ |
| Aufbewahrung bewusst wählen | ✅ |
| Getrennte Dashboards | ✅ wenn es eigene Zugänge gibt |
| Heizkostenabrechnung daraus | ❌ |
