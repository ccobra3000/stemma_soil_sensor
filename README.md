# ESPHome STEMMA Soil Sensor Component

A custom ESPHome component for the [Adafruit STEMMA Soil Sensor](https://www.adafruit.com/product/4026), which uses the Seesaw I2C protocol to measure soil moisture via capacitive sensing and temperature.

## Credits

This component is based on the original code by cerebrate: https://gist.github.com/cerebrate/92d607153f0fd2c0f62cbd8aeed9f6b9

It has been refactored and enhanced to work as a proper ESPHome sensor platform with improved error handling, timing adjustments, and better integration with ESPHome's infrastructure.

## Features

- **Temperature Sensing**: Reads soil temperature in Celsius (can be converted to Fahrenheit in YAML)
- **Moisture Sensing**: Capacitive moisture measurement in pF (picofarads)
- **Configurable Polling**: Set custom update intervals (default: 300 seconds)
- **I2C Communication**: Uses ESPHome's I2C interface with proper error handling
- **Hardware Validation**: Verifies device hardware ID during setup
- **Retry Logic**: Automatic retries for moisture readings to ensure stable values

## Hardware

- **Sensor**: [Adafruit STEMMA Soil Sensor](https://www.adafruit.com/product/4026)
- **Protocol**: Seesaw I2C
- **Default I2C Address**: 0x36
- **Connections**:
  - VCC → 3.3V or 5V
  - GND → Ground
  - SDA → I2C Data
  - SCL → I2C Clock

## Installation

### 1. Copy Component Files

Place the component files in your ESPHome configuration directory:

```
config/
├── your-device.yaml
└── custom_components/
    └── stemma_soil_sensor/
        ├── __init__.py
        ├── sensor.py
        └── stemma_soil_sensor.h
```

### 2. Configure ESPHome YAML

Add the component to your device configuration:

```yaml
# Load the external component
external_components:
  - source:
      type: local
      path: custom_components
    components: [ stemma_soil_sensor ]

# Configure I2C bus
i2c:
  sda: GPIO21
  scl: GPIO22
  scan: true  # Optional: scan for I2C devices on boot

# Add the sensor
sensor:
  - platform: stemma_soil_sensor
    address: 0x36  # Optional, defaults to 0x36
    update_interval: 60s  # Optional, defaults to 300s
    temperature:
      name: "Soil Temperature"
      filters:
        - lambda: return x * 9.0/5.0 + 32.0;  # Optional: Convert to Fahrenheit
    moisture:
      name: "Soil Moisture"
```

### 3. Compile and Upload

```bash
esphome compile your-device.yaml
esphome upload your-device.yaml
```

## Configuration Options

### Platform Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `address` | hex | `0x36` | I2C address of the sensor |
| `update_interval` | time | `300s` | How often to poll the sensor |

### Sensor Options

Both `temperature` and `moisture` are optional and support all standard ESPHome sensor options:

| Sensor | Unit | Decimals | Description |
|--------|------|----------|-------------|
| `temperature` | °C | 1 | Soil temperature in Celsius |
| `moisture` | pF | 0 | Capacitive moisture reading |

## Usage Examples

### Basic Configuration

```yaml
sensor:
  - platform: stemma_soil_sensor
    temperature:
      name: "Soil Temperature"
    moisture:
      name: "Soil Moisture"
```

### Temperature Only

```yaml
sensor:
  - platform: stemma_soil_sensor
    temperature:
      name: "Soil Temperature"
```

### Custom Update Interval

```yaml
sensor:
  - platform: stemma_soil_sensor
    update_interval: 30s
    temperature:
      name: "Soil Temperature"
    moisture:
      name: "Soil Moisture"
```

### With Filters and Calibration

```yaml
sensor:
  - platform: stemma_soil_sensor
    temperature:
      name: "Soil Temperature"
      filters:
        - lambda: return x * 9.0/5.0 + 32.0;  # Convert to Fahrenheit
    moisture:
      name: "Soil Moisture"
      unit_of_measurement: "%"
      filters:
        - calibrate_linear:
            - 200 -> 0    # Dry
            - 2000 -> 100  # Wet
        - lambda: |-
            if (x < 0.0) return 0.0;
            if (x > 100.0) return 100.0;
            return x;
```

### Multiple Sensors

```yaml
sensor:
  - platform: stemma_soil_sensor
    address: 0x36
    temperature:
      name: "Plant 1 Temperature"
    moisture:
      name: "Plant 1 Moisture"

  - platform: stemma_soil_sensor
    address: 0x37  # Second sensor with modified address
    temperature:
      name: "Plant 2 Temperature"
    moisture:
      name: "Plant 2 Moisture"
```

## Troubleshooting

### Sensor Not Found

If you see errors like "I2C hardware NACK detected":

1. Verify wiring connections (SDA, SCL, VCC, GND)
2. Check that the sensor appears in I2C scan (enable `scan: true` in i2c config)
3. Verify the I2C address (default is 0x36)
4. Ensure adequate power supply (3.3V or 5V)

### Invalid Hardware ID

If setup fails with "Invalid hardware ID":

- The component expects hardware ID 0x36
- Verify you have a genuine Adafruit STEMMA Soil Sensor
- Check I2C communication is working (try I2C scan)

### Timing Issues

The Seesaw protocol requires specific timing delays. This component uses:
- 1000ms delay after reset
- 2000μs-3000μs delays for sensor reads
- Automatic retries for moisture readings

If you still experience issues, you may need to adjust these values in `stemma_soil_sensor.h`.

## Technical Details

### Seesaw Protocol

The component communicates using Adafruit's Seesaw protocol:
- **Status Base Address**: 0x00 (system functions, temperature)
- **Touch Base Address**: 0x0F (capacitive sensing)

### Register Map

| Register | Function |
|----------|----------|
| 0x00, 0x01 | Hardware ID (returns 0x36) |
| 0x00, 0x04 | Temperature (4 bytes, signed 32-bit fixed point) |
| 0x00, 0x7F | Software reset |
| 0x0F, 0x10 | Capacitive reading channel 0 (2 bytes) |

## Development

See [CLAUDE.md](CLAUDE.md) for detailed architecture documentation and development guidance.

## License

This component inherits the license of the original gist code. Please refer to the original author's licensing terms.

## Contributing

Contributions are welcome! Please submit issues or pull requests on the repository.

## See Also

- [Adafruit STEMMA Soil Sensor Product Page](https://www.adafruit.com/product/4026)
- [Adafruit Seesaw Guide](https://learn.adafruit.com/adafruit-seesaw-atsamd09-breakout)
- [ESPHome Custom Components Documentation](https://esphome.io/custom/custom_component.html)
- [Original Code Gist](https://gist.github.com/cerebrate/92d607153f0fd2c0f62cbd8aeed9f6b9)
