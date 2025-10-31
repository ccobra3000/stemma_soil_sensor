# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an ESPHome custom component for the Adafruit STEMMA Soil Sensor, which uses the Seesaw protocol over I2C to read capacitive moisture and temperature data.

## Architecture

### Component Structure

The component consists of three main files:

1. **`__init__.py`** - Package initialization
   - Defines the component namespace for ESPHome
   - Minimal file required for Python package structure

2. **`sensor.py`** - ESPHome sensor platform integration
   - Registers `stemma_soil_sensor` as a sensor platform
   - Defines configuration schema with temperature and moisture sensor options
   - Handles I2C device registration (default address 0x36)
   - Configurable polling interval (default 300s)
   - Links YAML configuration to C++ sensor objects via setter methods

3. **`stemma_soil_sensor.h`** - C++ implementation (header-only)
   - Implements `StemmaSoilSensor` class inheriting from `PollingComponent` and `i2c::I2CDevice`
   - Handles I2C communication with the Seesaw protocol using ESPHome's I2C interface
   - Exposes two optional ESPHome `Sensor` entities via setter methods
   - Includes proper error handling and hardware validation

### Communication Protocol

The component implements the Adafruit Seesaw protocol for I2C communication:
- **Device I2C Address:** `0x36` (configurable in YAML)
- **Hardware ID:** `0x36` (verified during setup)
- **Module Base Addresses:**
  - `SEESAW_STATUS_BASE` (0x00): Status and system functions
  - `SEESAW_TOUCH_BASE` (0x0F): Capacitive touch/moisture readings
- **Key Registers:**
  - `SEESAW_STATUS_HW_ID`: Hardware identification
  - `SEESAW_STATUS_SWRST`: Software reset
  - `SEESAW_STATUS_TEMP`: Temperature reading
  - `SEESAW_TOUCH_CHANNEL_OFFSET`: Base address for capacitive readings

### Data Flow

1. **Setup:** Performs software reset, waits 500ms, verifies hardware ID (marks component as failed if validation fails)
2. **Update Loop:** At configurable interval (default 300s), reads temperature in Celsius and capacitive moisture value
3. **Publishing:** Sensor values published to ESPHome's sensor framework with proper null checks
   - Temperature in °C (1 decimal place by default)
   - Moisture in pF (0 decimal places by default)

### I2C Implementation Details

- **Interface:** Uses ESPHome's `i2c::I2CDevice` base class (not Arduino Wire)
- **Read Operations:** Uses ESPHome's I2C read/write methods with proper error handling
- **Timing:** Uses configurable delays (default 125μs for reads, 1000μs for sensor data)
- **Retry Logic:** `read_moisture()` retries up to 5 times if invalid data (0xFFFF) received
- **Error Handling:** All I2C operations return bool for success/failure; component marks itself as failed on setup errors

## Development

This component is designed to be used as an external component in ESPHome configurations.

### YAML Configuration

```yaml
external_components:
  - source:
      type: local
      path: custom_components
    components: [ stemma_soil_sensor ]

i2c:
  sda: GPIO21
  scl: GPIO22
  scan: true

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

Both `temperature` and `moisture` are optional - you can configure one or both sensors.

### Testing

The component does not use standard build/test commands since it's integrated into ESPHome's build system. Testing is done by:
1. Adding the component to an ESPHome device YAML configuration
2. Running `esphome compile your-device.yaml` to verify compilation
3. Running `esphome upload your-device.yaml` to test on hardware
