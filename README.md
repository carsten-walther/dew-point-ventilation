# Dew Point Ventilation

## How it works

The two DHT22 sensors determine the temperatures and relative humidity for indoors and outdoors.

The difference in humidity is calculated from the measured values and the relay for deactivating/activating the fan is switched depending on this.

The settings for switching hysteresis, minimum outdoor and indoor temperatures can be made via the LCD menu and the rotary encoder.

The device is configured so that it also works without a WLAN connection or MQTT/Home Assistant API.

## Requirements

* 1x Wemos D1 mini
* 2x DHT22
* 1x 20x4 LCD-Display with I2C Interface
* 1x Relais Breakout board
* 1x 360° Rotary Encoder with Push-Button
  
## Input/Outout MQTT/Home Assistant API

- Delta Dew Point 

### Outdoors

- Temperature
- Humidity
- Absolute Humidity

### Indoors

- Temperature
- Humidity
- Absolute Humidity

### In general

- Uptime
- Wifi signal
- Connection status

### Number Inputs

These settings can also be adjusted via the LCD menu.

- Minimum dew point difference at which the relay switches
- Distance from switch-on and switch-off point
- Minimum indoor temperature at which the ventilation is activated
- Minimum outside temperature at which the ventilation is activated
- Backlight delay
- Mode: Automatic, On, Off

### Buttons/Selects

- Reset
- Restart
- Mode: Automatic, On, Off
