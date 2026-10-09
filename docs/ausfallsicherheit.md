# Ausfallsicherheit

Zurück zur [Übersicht](../README.md) · [Hauptdoku](raumheizung.md)

---

Der Blueprint fängt viel ab. Eines kann er nicht: sich selbst retten. Fällt Home
Assistant aus, läuft keine Automation mehr – auch diese nicht.

Deshalb gibt es drei Ebenen, und nur die erste steckt im Blueprint.

---

## Ebene 1: Der Blueprint

| Fall | Verhalten |
|---|---|
| Raumsensor tot, veraltet oder unplausibel | Notbetrieb, je nach Reglertyp (Fall A–F in der Hauptdoku) |
| Modus-Quelle in unbekanntem Zustand | letzte Stufe, nach Zeitlimit Ersatzstufe, dazu Meldung |
| Regler nimmt einen Befehl nicht an | erneut senden, dann kritische Meldung |
| Regler nicht erreichbar | kritische Meldung nach der Meldefrist |
| Home Assistant startet neu | vollständige Neubewertung |
| Ein Ereignis geht verloren | spätestens nach 15 Minuten korrigiert |

**Mit Status-Helfer** überstehen auch ein laufender manueller Eingriff, die letzte
Stufe und die Fenster-Abfallerkennung einen Neustart.

---

## Ebene 2: Das Gerät

**Die wichtigste Regel: Kein Aktor darf bei einem Ausfall unbegrenzt „an" oder
unbegrenzt „aus" bleiben.**

### Thermostatköpfe

Die meisten Zigbee-Köpfe regeln bei Funkverlust über ihren internen Sensor weiter.
Das ist nicht genau, aber es ist eine Regelung – der Raum wird weder eiskalt noch
überhitzt.

Viele Köpfe haben zusätzlich einen eigenen Wochenplan im Gerät. Wenn deiner das kann,
hinterlege dort einen vernünftigen Grundplan. Er greift nur, wenn sonst nichts mehr
greift.

### Relais (Shelly und ähnliche)

Hier liegt das eigentliche Risiko: Ein Relais ist stur. Es bleibt, wo es ist.

**Mindestens das einstellen:**

*Shelly-Weboberfläche → Einstellungen → Verhalten beim Einschalten*

| Einstellung | Bewertung |
|---|---|
| Letzten Zustand wiederherstellen | gefährlich – ein „an" bleibt „an" |
| **Immer aus** | sicher, aber kalt |
| Immer an | nur bei Frostgefahr, und dann bewusst |

**Besser: Auto-Aus-Timer.** Shellys können einen Ausgang nach einer festen Zeit von
allein abschalten. Setz ihn etwas länger als dein längstes normales Taktfenster, zum
Beispiel 90 Minuten bei einem Taktzyklus von einer Stunde.

Im Normalbetrieb merkst du davon nichts, weil der Regler ohnehin vorher schaltet.
Bricht die Verbindung ab, schaltet der Shelly nach spätestens 90 Minuten selbst ab –
statt die ganze Nacht durchzuheizen.

> Bei meinem Vorfall lief ein Heizkreis acht Stunden ungeregelt. Mit Auto-Aus-Timer
> wären es 90 Minuten gewesen.

---

## Ebene 3: Die Infrastruktur

### Überwachung von außen

Ein Monitoring-Dienst auf einem *anderen* Gerät prüft, ob Home Assistant erreichbar
ist, und meldet sich, wenn nicht. Das ist der einzige Weg, einen HA-Ausfall überhaupt
mitzubekommen. Die Heizung ist der schlechteste Melder dafür, denn sie fällt lautlos
aus – man merkt es erst, wenn es kalt wird.

### Stromversorgung

Eine kleine USV für HA und Netzwerk-Hardware überbrückt kurze Ausfälle. Das löst
keinen Dauerausfall, verhindert aber die häufigsten unsauberen Abstürze.

### Sicherungen

Vor jedem größeren Umbau an der Heizungssteuerung eine Sicherung anlegen. Ein
`climate`-Block in der YAML ist schnell kaputtgeschrieben und im Winter schlecht zu
debuggen.

---

## Checkliste vor der Heizsaison

- [ ] Jeder Raum hat mindestens zwei Temperaturquellen → [ersatzsensor.md](ersatzsensor.md)
- [ ] Jeder Shelly hat ein definiertes Verhalten beim Einschalten
- [ ] Jeder Shelly hat einen Auto-Aus-Timer
- [ ] Die PID-Regler haben `sensor_stall` und `output_safety` gesetzt
- [ ] Thermostatköpfe haben einen Grundplan im Gerät
- [ ] Der Blueprint hat ein funktionierendes Meldeziel
- [ ] Externes Monitoring meldet, wenn HA weg ist
- [ ] Jeder Raum hat einen Status-Helfer (Maximallänge 255)
- [ ] Eine aktuelle Sicherung existiert

---

## Einen Fehler erkennen, bevor es kalt wird

Zwei Fragen, die sich schnell beantworten lassen:

**Sieht jeder Regler eine Ist-Temperatur?**

```jinja
{% for e in states.climate %}
{{ e.entity_id }} = {{ e.attributes.get('current_temperature', 'KEIN WERT') }}
{% endfor %}
```

Steht irgendwo `KEIN WERT` oder `None`, ist genau dieser Raum ungeregelt.

**Läuft ein Heizkreis zu lange?**

Schau im Verlauf nach, ob ein Aktor über viele Stunden durchgehend „an" steht. Ein
gesunder Heizkreis taktet.
