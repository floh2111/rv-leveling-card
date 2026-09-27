# RV Leveling Card

🇬🇧 [English version](README.md)

Eine Lovelace-Karte für Home Assistant: eine Kreuzlibelle (Wasserwaage)
auf einem Fahrzeug-Grundriss – wahlweise **Wohnwagen** (mit Deichsel)
oder **Wohnmobil** (mit Frontscheibe) –, mit der Front nach oben. Zeigt
den aktuellen Neigungswinkel in zwei Achsen (Links/Rechts, Vorne/Hinten)
an und kann per Knopf einen Nullpunkt an einer beliebigen Entity
auslösen (z.B. um den Sensor an der Quelle zu kalibrieren).

| Wohnwagen | Wohnmobil |
|---|---|
| ![Wohnwagen](images/preview-caravan.png) | ![Wohnmobil](images/preview-motorhome.png) |

Funktioniert mit **jeder** Entity-Kombination, die zwei Neigungswinkel
in Grad liefert – unabhängig von Hersteller oder Sensor-Typ.

## Voraussetzungen

Zwei `sensor`-Entities mit einem Neigungswinkel in Grad (positive und
negative Werte, z.B. `-3.2` bis `3.2`):

- Neigung **links/rechts**
- Neigung **vorne/hinten**

Optional ein `button` (oder eine andere Entity mit `button.press`), der
auf der Gegenstelle den Nullpunkt setzt/kalibriert. Die Karte selbst
speichert nichts – ein Druck auf "Nullen" ruft nur `button.press` auf
dieser Entity auf.

## Installation über HACS

1. HACS → oben rechts ⋮ → Benutzerdefinierte Repositories.
2. Repository-URL: `https://github.com/floh2111/rv-leveling-card`,
   Kategorie: **Dashboard**.
3. "RV Leveling Card" installieren, Home Assistant neu starten.
4. Die Ressource wird von HACS automatisch unter
   `/hacsfiles/rv-leveling-card/rv-leveling-card.js` eingebunden
   (Lovelace-Ressource wird von HACS selbst verwaltet).

## Karte hinzufügen

Über den Karten-Editor: "Karte hinzufügen" → "Benutzerdefiniert" →
**RV Leveling Card**. Titel, Fahrzeugtyp und die drei Entity-IDs lassen
sich direkt im visuellen Editor einstellen.

Alternativ per YAML:

```yaml
type: custom:rv-leveling-card
title: Nivellierung
vehicle_type: caravan   # oder: motorhome
entity_lr: sensor.deine_neigung_links_rechts
entity_vh: sensor.deine_neigung_vorne_hinten
entity_zero_button: button.deine_neigung_nullen   # optional
```

## Sprache

Die eigenen UI-Texte der Karte (Labels, der "Nullen"-Button, die
Editor-Felder) folgen deiner Home-Assistant-Spracheinstellung
(`hass.language`) – aktuell Englisch, Deutsch, Französisch, Italienisch,
Niederländisch, Polnisch, Dänisch und Spanisch, Englisch als Fallback
für jede andere Sprache. Beiträge für weitere Sprachen sind
willkommen (siehe `TRANSLATIONS` in `rv-leveling-card.js`).

## Konfigurationsoptionen

| Option | Pflicht | Beschreibung |
|---|---|---|
| `title` | nein | Überschrift der Karte (Standard: "Nivellierung"/"Leveling", folgt der HA-Sprache) |
| `vehicle_type` | nein | `caravan` (Wohnwagen, Standard) oder `motorhome` (Wohnmobil) |
| `entity_lr` | ja | Sensor Neigung links/rechts (°) |
| `entity_vh` | ja | Sensor Neigung vorne/hinten (°) |
| `entity_zero_button` | nein | Button, der bei "Nullen" gedrückt wird |

## Beispiel: eigenen Neigungssensor per ESPHome bereitstellen

Falls du noch keine passenden Entities hast: mit einem ESP32/ESP8266 +
einem MPU6050-Beschleunigungssensor (GY-521-Breakout, wenige Euro, I²C)
lassen sich alle drei benötigten Entities in wenigen Zeilen ESPHome-YAML
bereitstellen. Verkabelung: `VCC`→3V3, `GND`→GND, `SCL`→ein beliebiger
GPIO (hier 22), `SDA`→ein beliebiger GPIO (hier 21).

```yaml
i2c:
  sda: GPIO21
  scl: GPIO22

# Nullpunkt-Offset übersteht einen Neustart, wird aber nicht bei jeder
# Änderung sofort ins Flash geschrieben (spart Schreibzyklen)
preferences:
  flash_write_interval: 5min

globals:
  - id: g_offset_lr
    type: float
    restore_value: true
    initial_value: '0'
  - id: g_offset_vh
    type: float
    restore_value: true
    initial_value: '0'
  - id: g_roh_lr
    type: float
    restore_value: false
    initial_value: '0'
  - id: g_roh_vh
    type: float
    restore_value: false
    initial_value: '0'

sensor:
  - platform: mpu6050
    address: 0x68
    update_interval: 200ms
    accel_x:
      id: accel_x
    accel_y:
      id: accel_y
    accel_z:
      id: accel_z

  # entity_lr der Karte
  - platform: template
    id: neigung_links_rechts
    name: "Neigung links-rechts"
    unit_of_measurement: "°"
    update_interval: 200ms
    lambda: |-
      float roh = atan2(id(accel_y).state, id(accel_z).state) * 180.0 / M_PI;
      id(g_roh_lr) = roh;
      return roh - id(g_offset_lr);

  # entity_vh der Karte
  - platform: template
    id: neigung_vorne_hinten
    name: "Neigung vorne-hinten"
    unit_of_measurement: "°"
    update_interval: 200ms
    lambda: |-
      float roh = atan2(-id(accel_x).state, sqrt(id(accel_y).state * id(accel_y).state + id(accel_z).state * id(accel_z).state)) * 180.0 / M_PI;
      id(g_roh_vh) = roh;
      return roh - id(g_offset_vh);

# entity_zero_button der Karte
button:
  - platform: template
    name: "Neigung nullen"
    on_press:
      - lambda: |-
          id(g_offset_lr) = id(g_roh_lr);
          id(g_offset_vh) = id(g_roh_vh);
      - component.update: neigung_links_rechts
      - component.update: neigung_vorne_hinten
```

Nach dem Flashen erscheinen automatisch `sensor.neigung_links_rechts`,
`sensor.neigung_vorne_hinten` und `button.neigung_nullen` in Home
Assistant (vorausgesetzt, die ESPHome-API-Integration ist eingerichtet)
– genau die drei Entities, die die Karte braucht. Ein Druck auf
"Nullen" merkt sich die aktuelle Lage als neuen Nullpunkt, dauerhaft im
Flash des ESP gespeichert (übersteht auch einen Neustart von Home
Assistant).

## Lizenz

MIT
