# Catnip Soil Sensor for ESPHome

ESPHome external component for the Chirp/Catnip I2C capacitive soil moisture sensor.

Based on the sensor available at: https://github.com/Miceuz/i2c-moisture-sensor

## Features

This component provides the following sensor readings:

- **Capacitance** - Raw capacitance value from the sensor
- **Temperature** - Soil temperature in Celsius
- **Light** - Ambient light level (higher values = darker)

> **Note:** This component exports raw sensor values. To calculate moisture percentage, use Home Assistant template sensors with your own calibration values.

## Hardware

### Sensor Specifications

- **I2C Address:** 0x20 (default, configurable)
- **Supply Voltage:** 3.3V - 5V
- **Interface:** I2C
- **Measurements:** Capacitance, Temperature, Light

### Wiring

Connect the sensor to your ESP device:

| Sensor Pin | ESP Pin |
|------------|---------|
| VCC        | 3.3V or 5V |
| GND        | GND |
| SDA        | SDA (GPIO21 on ESP32) |
| SCL        | SCL (GPIO22 on ESP32) |

## Installation

### 1. Copy Component Files

Clone this repository or copy the `components/catnip_soil_sensor` directory to your ESPHome configuration directory.

```
your-esphome-config/
├── components/
│   └── catnip_soil_sensor/
│       ├── __init__.py
│       ├── sensor.py
│       ├── catnip_soil_sensor.h
│       └── catnip_soil_sensor.cpp
└── your-device.yaml
```
and load the external component with

```
# Load external component
external_components:
  - source:
      type: local
      path: components
```
or skip this step and let ESP home downlaad the repository for you

### 2. Configure ESPHome

Add the external component and sensor configuration to your ESPHome YAML file:

```yaml
# Enable I2C
i2c:
  sda: GPIO21
  scl: GPIO22
  scan: true

# Load external component
external_components:
  - source:
      type: git
      url: https://github.com/jbeker/catnip_soil_sensor
      ref: main
    components: [catnip_soil_sensor]

# Configure sensor
sensor:
  - platform: catnip_soil_sensor
    address: 0x20
    update_interval: 60s

    capacitance:
      name: "Soil Capacitance"

    temperature:
      name: "Soil Temperature"

    light:
      name: "Soil Light Level"
```

## Configuration Variables

### Main Configuration

- **address** (*Optional*, int): I2C address of the sensor. Defaults to `0x20`.
- **update_interval** (*Optional*, Time): Update interval for sensor readings. Defaults to `60s`.

### Sensor Outputs

All sensor outputs are optional. Configure only the sensors you need.

- **capacitance** (*Optional*): Raw capacitance value from the sensor.
  - All options from [Sensor](https://esphome.io/components/sensor/index.html#config-sensor)

- **temperature** (*Optional*): Soil temperature in Celsius.
  - All options from [Sensor](https://esphome.io/components/sensor/index.html#config-sensor)

- **light** (*Optional*): Ambient light level (higher values indicate darker conditions).
  - All options from [Sensor](https://esphome.io/components/sensor/index.html#config-sensor)

## Calculating Moisture Percentage

This component exports raw capacitance values. To calculate moisture percentage, you can use Home Assistant template sensors with your own calibration values.

### Calibration Steps

1. **Measure Dry Value:**
   - Leave the sensor in open air
   - Read the `capacitance` sensor value
   - Note this as your dry value (typically around 290)

2. **Measure Wet Value:**
   - Submerge the sensor in water (only the sensing area, not electronics!)
   - Read the `capacitance` sensor value
   - Note this as your wet value (typically around 600-650)

3. **Create Template Sensor in Home Assistant:**
   ```yaml
   template:
     - sensor:
         - name: "Soil Moisture Percentage"
           unit_of_measurement: "%"
           state: >
             {% set dry_value = 290 %}
             {% set wet_value = 650 %}
             {% set raw = states('sensor.soil_moisture_sensor_soil_capacitance') | float(0) %}
             {% set moisture = ((raw - dry_value) / (wet_value - dry_value) * 100) | round(1) %}
             {{ [0, [moisture, 100] | min] | max }}
   ```

## Example Configuration

See [example.yaml](example.yaml) for a complete configuration example.

## Technical Details

### Sensor Registers

The component communicates with the following sensor registers:

| Register | Function | Data Type |
|----------|----------|-----------|
| 0x00 | Get Capacitance | 16-bit (MSB first) |
| 0x05 | Get Temperature | 16-bit (tenths of °C) |
| 0x03 | Trigger Light Measurement | Write only |
| 0x04 | Get Light Value | 16-bit (MSB first) |
| 0x07 | Get Firmware Version | 8-bit |

### Update Cycle

The component performs measurements in the following order:
1. Read capacitance value
2. Read temperature
3. Trigger light measurement
4. Wait ~1.5 seconds
5. Read light value (on next update cycle)

## Troubleshooting

### Sensor Not Found

If you see "Communication with Catnip Soil Sensor failed!" in the logs:

1. Check wiring connections
2. Verify I2C pins are correct for your ESP board
3. Run I2C scan to detect devices: set `scan: true` in i2c configuration
4. Check if sensor is powered (3.3V or 5V)

### Incorrect Readings

- **Temperature seems wrong:** Sensor reports in tenths of Celsius (252 = 25.2°C)
- **Light values noisy:** This is expected; the light sensor is less precise than other readings
- **Capacitance readings seem off:** Use the calibration method above to determine your sensor's specific dry/wet values

## License

This component is provided as-is for use with ESPHome.

## Credits

- Sensor hardware: [Miceuz/i2c-moisture-sensor](https://github.com/Miceuz/i2c-moisture-sensor)
- ESPHome: [esphome.io](https://esphome.io)
