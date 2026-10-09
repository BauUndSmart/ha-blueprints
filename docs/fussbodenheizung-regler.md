# Welcher Regler für die Fußbodenheizung?

Zurück zur [Übersicht](../README.md) · [Hauptdoku](raumheizung.md)

---

Der Blueprint steuert eine `climate`-Entität. Bei Heizkörpern mit smartem
Thermostatkopf *ist* der Kopf bereits diese Entität – da gibt es nichts zu entscheiden.

Bei der Fußbodenheizung sieht es anders aus: Dort hängt am Verteiler ein Relais
(z. B. ein Shelly), und das ist erstmal nur ein Schalter. Damit daraus ein Thermostat
wird, braucht es etwas dazwischen.

---

## Die Optionen

| | Weg A: `generic_thermostat` | Weg B: PID-Thermostat |
|---|---|---|
| Herkunft | Home Assistant Core | HACS |
| Einrichtung | Oberfläche | YAML |
| Regelprinzip | Hysterese (an/aus) | Taktung per Pulsweite |
| Genauigkeit bei FBH | ausreichend | besser |
| Außentemperatur-Kompensation | nein | ja |

---

## Weg A: `generic_thermostat` (Core)

Der eingebaute Weg. Kein Zusatz nötig, Einrichtung komplett über die Oberfläche:

*Einstellungen → Geräte & Dienste → Helfer → Erstellen → Generisches Thermostat*

| Feld | Wert |
|---|---|
| Heizgerät | dein Relais (`switch.*`) |
| Temperatursensor | der Raumsensor |
| Kalt-/Warm-Toleranz | je 0,3 K |
| Mindestlaufzeit | 10 min |
| Abkühlzeit | 10 min |

**Wie es regelt:** unter Sollwert minus Toleranz an, über Sollwert plus Toleranz aus.

**Der Haken bei trägen Fußbodenheizungen:** Wenn das Relais abschaltet, ist der Estrich
noch warm und gibt weiter Wärme ab. Die Temperatur steigt also über den Abschaltpunkt
hinaus. Umgekehrt dauert es nach dem Einschalten, bis überhaupt etwas ankommt.

Mit Mindestlaufzeit und Abkühlzeit lässt sich das abfedern. Perfekt wird es nicht, für
viele Installationen reicht es aber gut.

---

## Weg B: PID-Thermostat (HACS)

Ein PID-Regler schaltet nicht einfach an und aus, sondern berechnet, wie lange die
Heizung innerhalb eines Zeitfensters laufen soll. Statt „voll an oder ganz aus" also
zum Beispiel „35 % der nächsten Stunde".

Genau das passt zu einer Fußbodenheizung.

**Verbreitet ist [HASmartThermostat](https://github.com/ScratMan/HASmartThermostat).**
Es gibt daneben Forks und Alternativen – der Blueprint funktioniert mit allen, weil er
nur die `climate`-Entität anspricht.

Beispielkonfiguration:

```yaml
climate:
  - platform: smart_thermostat
    name: Wohnzimmer FBH (PID)
    unique_id: wohnzimmer_fbh_pid
    heater: switch.fussbodenheizung_wohnzimmer
    target_sensor: sensor.wohnzimmer_temperatur_mittel
    outdoor_sensor: sensor.aussentemperatur
    min_temp: 5
    max_temp: 26
    target_temp_step: 0.5
    pwm: "01:00:00"
    kp: 10
    ki: 0.005
    kd: 500
    keep_alive:
      minutes: 30
    sensor_stall: "06:00:00"
    output_safety: 5
```

### Die zwei Parameter, die wirklich zählen

```yaml
sensor_stall: "06:00:00"   # Sensor meldet 6 h nichts -> Sicherheitsmodus
output_safety: 5           # dann 5 % Heizleistung
```

**Ohne diese beiden friert der Regler bei Sensorausfall seinen letzten Zustand ein.**
Stand er dabei auf „heizen", läuft der Heizkreis ungeregelt weiter, bis es jemand
merkt. Das ist kein theoretisches Szenario – genau so ist es mir passiert.

> **Achtung, eine wichtige Unterscheidung:** `sensor_stall` greift, wenn ein Sensor
> *stehenbleibt*. Ob es auch greift, wenn die Sensor-Entität *komplett verschwindet*
> (z. B. weil ein Template-Sensor aus der Konfiguration entfernt wurde), ist nicht
> garantiert. Deshalb hat der Blueprint einen eigenen Sensor-Wächter – als zweite
> Verteidigungslinie.

### Die PID-Parameter

| Parameter | Wirkung | Startwert für FBH |
|---|---|---|
| `kp` | Reaktion auf die aktuelle Abweichung | 10 |
| `ki` | Ausgleich der Restabweichung über Zeit | 0.005 |
| `kd` | Dämpfung bei schnellen Änderungen | 500 |
| `ke` | Außentemperatur-Kompensation | 0.6 bei guter Dämmung |
| `pwm` | Länge eines Taktfensters | 1 h |

**Tuning in Kurzform:** Bleibt der Raum dauerhaft unter dem Sollwert, `ki` erhöhen.
Schießt er über, `ki` senken. Reagiert er zu langsam, `kp` erhöhen.

Nach jeder Änderung mindestens einen halben Tag beobachten. Eine Fußbodenheizung
braucht Stunden, bis man eine Wirkung sieht – wer stündlich nachjustiert, dreht im
Kreis.

Die Autotune-Funktion würde ich nicht nutzen. Sie ist laut Projekt selbst nicht
zuverlässig, und bei einer trägen Fußbodenheizung dauert ein Durchlauf sehr lange.

---

## Was der Blueprint vom Regler erwartet

Nur Standard-Verhalten:

| Erwartung | Warum |
|---|---|
| Attribut `temperature` (Sollwert) | um Änderungen zu erkennen und zu schreiben |
| Attribut `current_temperature` | für die Anzeige |
| Attribut `hvac_action` | um zu sehen, ob gerade geheizt wird |
| `climate.set_temperature` | zum Setzen |
| `climate.set_hvac_mode` mit `off` | optional, nur wenn Fensteraktion „aus" gewählt ist |

Kann ein Regler kein `off`, verwendet der Blueprint automatisch den
Frostschutz-Sollwert.

---

## Und wenn der Regler seine eigene Fensterlogik mitbringt?

Manche Regler bringen Fenstererkennung, Presets und Sicherheitsmodus selbst mit. Das
überschneidet sich mit dem Blueprint.

**Lösung:** Fensteraktion auf „nur melden" stellen. Dann macht der Regler die
Abschaltung, und der Blueprint kümmert sich um Stufen, Zeitplan und Meldungen.
