# Ersatzsensor: mehrere Quellen zu einem robusten Wert

Zurück zur [Übersicht](../README.md) · [Hauptdoku](raumheizung.md)

---

## Warum das die wichtigste Vorarbeit ist

Ein Thermostat ohne Messwert regelt nicht. Je nach Regler bleibt es dann im letzten
Zustand stehen – und wenn der „heizen" war, läuft der Heizkreis stundenlang
ungeregelt weiter.

Der Blueprint hat dafür einen Wächter mit Notbetrieb. Der ist aber die *zweite*
Verteidigungslinie. Die erste ist viel einfacher: **Gib jedem Raum mehr als eine
Temperaturquelle.** Dann fällt gar nicht erst alles aus.

---

## Die Lösung: Helfer „Sensorwerte kombinieren"

Home Assistant kann das von Haus aus, ohne Code.

*Einstellungen → Geräte & Dienste → Helfer → Erstellen → **Sensorwerte kombinieren***

| Feld | Wert |
|---|---|
| Name | z. B. „Wohnzimmer Temperatur" |
| Eingangsentitäten | alle Temperatursensoren des Raums |
| Statistikeigenschaft | **Arithmetisches Mittel** |
| Nachkommastellen | 1 oder 2 |

Den fertigen Helfer trägst du dann beim Regler als Temperatursensor ein und im
Blueprint als Raumtemperatursensor.

### Warum das robust ist

**Der Helfer ignoriert Quellen ohne Wert automatisch.** Fällt ein Sensor aus, rechnet
er mit den verbleibenden weiter. Es braucht keine Fallback-Logik, keine Templates,
keine Bedingungen.

Fallen *alle* Quellen aus, hat der Helfer keinen Wert mehr. Genau dann greift der
Sensor-Wächter im Blueprint.

---

## Welche Quellen eignen sich

| Quelle | Taugt als | Anmerkung |
|---|---|---|
| Zigbee-Temperatursensor | Hauptquelle | günstig, genau, lokal |
| Netatmo / WLAN-Sensor | Hauptquelle | genau, hängt aber an einer Cloud oder Bridge |
| **Wandpanel** (NSPanel o. ä.) | Ersatzquelle | **misst durch Eigenwärme oft zu warm** |
| Bewegungsmelder mit Temperatursensor | Ersatzquelle | oft nur grob, aber besser als nichts |
| Heizkörper-Thermostatkopf | Notnagel | misst direkt am Heizkörper, also zu warm |

### Die Panel-Falle

Wandpanels erwärmen sich selbst. In meinem Wohnzimmer maßen zwei Panels konstant
0,2 bis 1,5 K zu warm. Weil sie zwei Drittel des Mittelwerts stellten, war der ganze
Raumwert um 0,6 K zu hoch – und die Heizung hat entsprechend zu wenig geheizt.

**Faustregel:** Panels nur in den Mittelwert nehmen, wenn es sonst gar keine zweite
Quelle gäbe. Lieber ein leicht schiefer Wert als gar keiner – aber nicht, wenn es
bessere gibt.

> **Vergleichen lohnt sich.** Stell Panel und echten Sensor nebeneinander in
> *Entwicklerwerkzeuge → Zustände*. Bei mehr als 1 K Unterschied: Panel raus, oder in
> dessen Firmware einen Temperatur-Offset setzen.

---

## Zwei Integrationen für denselben Sensor

Manche Geräte erscheinen doppelt, etwa einmal direkt und einmal über HomeKit. Beide in
den Helfer zu legen ist sinnvoll: Fällt eine *Integration* aus, liefert die andere
weiter.

Zwei Dinge sollte man dabei wissen:

- Es bleibt **derselbe physische Sensor**. Ist die Batterie leer, fallen beide weg.
- Im Mittelwert **zählt dieser Sensor doppelt**. Das kann man wollen (er gleicht dann
  ein ungenaues Panel stärker aus) – bewusst entscheiden sollte man es.

---

## Migration von YAML-Template-Sensoren

Hast du deine Mittelwerte schon als Template-Sensoren in der YAML, lassen sie sich
ohne Bruch auf Helfer umstellen. Der Trick: **Gib dem neuen Helfer dieselbe
Entitäts-ID wie dem alten Sensor.** Dann merken Regler, Automationen und Dashboards
nichts – und sogar der Verlauf läuft nahtlos weiter.

### Ablauf

1. **Helfer anlegen**, zunächst unter einem Zwischennamen (`… neu`).
2. **Prüfen**, ob er einen plausiblen Wert liefert.
3. **Template-Block aus der YAML löschen.**
4. **Template-Entitäten neu laden** (*Entwicklerwerkzeuge → YAML*). Kein Neustart nötig.
5. **Verwaiste Einträge entfernen.** Der alte Sensor steht jetzt als `unavailable` in
   der Entitätenliste. Diesen Eintrag löschen, sonst ist die Entitäts-ID belegt.
6. **Helfer umbenennen** auf die alte Entitäts-ID und den alten Anzeigenamen.
7. **Kontrollieren**, dass der Regler wieder eine Ist-Temperatur sieht.

> ⚠️ **Schritt 3 bis 6 gehören in eine Sitzung.** Dazwischen hat der Regler keinen
> Messwert. Bei einer trägen Fußbodenheizung sind ein paar Minuten egal, ein paar
> Tage nicht.

### Die Alternative ohne Zeitdruck

Statt der ID-Übernahme kannst du auch den Regler auf den neuen Helfer umbiegen. Dann
gibt es keine Lücke, die Entitäts-IDs sind aber neu und der Verlauf beginnt von vorn.

| | ID-Übernahme | Regler umbiegen |
|---|---|---|
| Lücke ohne Messwert | wenige Minuten | keine |
| Entitäts-IDs | bleiben | ändern sich |
| Verlauf | läuft weiter | beginnt neu |
| Aufwand am Regler | keiner | eine Zeile pro Regler |

---

## Kontrolle

Nach der Einrichtung prüfen:

```jinja
{{ states('sensor.wohnzimmer_temperatur') }}
{{ state_attr('climate.wohnzimmer', 'current_temperature') }}
```

Beide Werte sollten übereinstimmen. Steht beim Regler `None`, kommt der Sensor nicht
an.
