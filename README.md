# RV Leveling Card

🇩🇪 [Deutsche Version](README.de.md)

A Lovelace card for Home Assistant: a cross-level (spirit level) drawn
on a vehicle floor plan – either a **caravan** (with tow hitch) or a
**motorhome** (with windshield) – with the front pointing up. Shows the
current tilt angle on two axes (left/right, front/back) and can trigger
a zero-point calibration on any entity via a button press (e.g. to
calibrate the sensor at its source).

| Caravan | Motorhome |
|---|---|
| ![Caravan](images/preview-caravan.png) | ![Motorhome](images/preview-motorhome.png) |

Works with **any** entity combination that provides two tilt angles in
degrees – independent of manufacturer or sensor type.

## Requirements

Two `sensor` entities with a tilt angle in degrees (positive and
negative values, e.g. `-3.2` to `3.2`):

- **Left/right** tilt
- **Front/back** tilt

Optionally a `button` (or any other entity supporting `button.press`)
that sets/calibrates the zero point on the source device. The card
itself stores nothing – pressing "Zero" just calls `button.press` on
that entity.

## Installation via HACS

1. HACS → top right ⋮ → Custom repositories.
2. Repository URL: `https://github.com/floh2111/rv-leveling-card`,
   Category: **Dashboard**.
3. Install "RV Leveling Card", restart Home Assistant.
4. The resource is automatically registered by HACS under
   `/hacsfiles/rv-leveling-card/rv-leveling-card.js` (the Lovelace
   resource is managed by HACS itself).

## Adding the card

Via the card editor: "Add card" → "Custom" → **RV Leveling Card**.
Title, vehicle type and the three entity IDs can be set directly in the
visual editor.

Or via YAML:

```yaml
type: custom:rv-leveling-card
title: Leveling
vehicle_type: caravan   # or: motorhome
entity_lr: sensor.your_tilt_left_right
entity_vh: sensor.your_tilt_front_back
entity_zero_button: button.your_tilt_zero   # optional
```

## Language

The card's own UI text (labels, the "Zero" button, the editor fields)
follows your Home Assistant language setting (`hass.language`) –
currently English and German, with English as the fallback for any
other language. Contributions adding more languages are welcome (see
`TRANSLATIONS` in `rv-leveling-card.js`).

## Configuration options

| Option | Required | Description |
|---|---|---|
| `title` | no | Card heading (default: "Leveling"/"Nivellierung", follows the HA language) |
| `vehicle_type` | no | `caravan` (default) or `motorhome` |
| `entity_lr` | yes | Left/right tilt sensor (°) |
| `entity_vh` | yes | Front/back tilt sensor (°) |
| `entity_zero_button` | no | Button pressed by the card's "Zero" button |

## Example: providing your own tilt sensor via ESPHome

If you don't have suitable entities yet: an ESP32/ESP8266 with an
MPU6050 accelerometer (GY-521 breakout board, a couple of dollars, I²C)
can provide all three required entities in a few lines of ESPHome YAML.
Wiring: `VCC`→3V3, `GND`→GND, `SCL`→any GPIO (22 here), `SDA`→any GPIO
(21 here).

```yaml
i2c:
  sda: GPIO21
  scl: GPIO22

# The zero-point offset survives a reboot, but isn't written to flash
# on every single change (saves flash write cycles)
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
  - id: g_raw_lr
    type: float
    restore_value: false
    initial_value: '0'
  - id: g_raw_vh
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

  # card's entity_lr
  - platform: template
    id: tilt_left_right
    name: "Tilt left-right"
    unit_of_measurement: "°"
    update_interval: 200ms
    lambda: |-
      float raw = atan2(id(accel_y).state, id(accel_z).state) * 180.0 / M_PI;
      id(g_raw_lr) = raw;
      return raw - id(g_offset_lr);

  # card's entity_vh
  - platform: template
    id: tilt_front_back
    name: "Tilt front-back"
    unit_of_measurement: "°"
    update_interval: 200ms
    lambda: |-
      float raw = atan2(-id(accel_x).state, sqrt(id(accel_y).state * id(accel_y).state + id(accel_z).state * id(accel_z).state)) * 180.0 / M_PI;
      id(g_raw_vh) = raw;
      return raw - id(g_offset_vh);

# card's entity_zero_button
button:
  - platform: template
    name: "Zero tilt"
    on_press:
      - lambda: |-
          id(g_offset_lr) = id(g_raw_lr);
          id(g_offset_vh) = id(g_raw_vh);
      - component.update: tilt_left_right
      - component.update: tilt_front_back
```

After flashing, `sensor.tilt_left_right`, `sensor.tilt_front_back` and
`button.zero_tilt` automatically show up in Home Assistant (assuming
the ESPHome API integration is set up) – exactly the three entities the
card needs. Pressing "Zero" remembers the current position as the new
zero point, stored permanently in the ESP's flash (survives a Home
Assistant restart too).

## License

MIT
