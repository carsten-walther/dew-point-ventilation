# Dew Point Ventilation

ESPHome firmware for a dew-point controlled ventilation unit. It compares the
dew point indoors and outdoors and runs an extractor fan only while outside air
is actually drier in absolute terms — the condition under which ventilating
removes moisture instead of adding it.

The device is self-contained: it regulates, displays and can be configured
entirely from its own LCD and rotary encoder. Wi-Fi, Home Assistant and MQTT are
optional conveniences, not requirements.

---

## Table of contents

- [How it works](#how-it-works)
- [Why the fan is doing what it does](#why-the-fan-is-doing-what-it-does)
- [Hardware](#hardware)
- [Pinout](#pinout)
- [Display](#display)
- [Menu](#menu)
- [Settings](#settings)
- [Home Assistant / MQTT](#home-assistant--mqtt)
- [Networking](#networking)
- [Safety behaviour](#safety-behaviour)
- [Building and flashing](#building-and-flashing)
- [Maintenance notes](#maintenance-notes)

---

## How it works

### Why dew point and not relative humidity

Comparing relative humidity indoors and outdoors is misleading, because relative
humidity depends on temperature. Cold outdoor air at 90 % RH can hold far less
water than warm cellar air at 60 % RH. The dew point is a temperature-independent
measure of how much water the air actually contains, so it is the correct basis
for deciding whether ventilating dries the room out or wets it further.

### Dew point calculation

Both dew points are derived from temperature and relative humidity using the
Magnus formula:

```
gamma = ln(RH / 100) + (17.67 · T) / (243.5 + T)
Td    = (243.5 · gamma) / (17.67 − gamma)
```

The relative humidity is clamped to a minimum of 0.1 % so `log()` never receives
zero.

### Control algorithm

Evaluated every 10 s, in this order — each step can abort the ones below it:

1. **Mode override.** `On` forces the fan on, `Off` forces it off. Only
   `Automatic` continues to the checks below.
2. **Sensor plausibility.** If either sensor reads implausibly (see
   [Safety behaviour](#safety-behaviour)), the fan is switched off and control
   stops. Regulating on bad data is worse than not regulating.
3. **Temperature limits.** If the indoor temperature is below *Indoor temperature
   minimum*, or the outdoor temperature below *Outdoor temperature minimum*, the
   fan is switched off. This keeps the room from being cooled down and protects
   against running the fan in freezing conditions.
4. **Dew point difference with hysteresis.**

```
delta = dew point indoors − dew point outdoors

delta > minimum + hysteresis   →  switch fan ON
delta < minimum                →  switch fan OFF
otherwise                      →  keep current state
```

The band between `minimum` and `minimum + hysteresis` is the hysteresis: inside
it nothing changes, which prevents the relay from chattering when the difference
hovers around the threshold.

5. **Minimum runtime and pause.** Step 4 may not stop a fan that has been running
   for less than `MIN_RUN_TIME_S`, nor start one that has been off for less than
   `MIN_PAUSE_TIME_S`. This protects the motor and the relay contacts from short
   cycling.

   Steps 1–3 deliberately ignore these limits and switch off immediately.
   Contact protection must never outrank safety: a faulty sensor or a
   temperature limit has to stop the fan on the spot.

### Why the fan is doing what it does

Every path through the control script — including the ones that abort early —
records a reason code. It is published as the `Fan State Reason` text sensor and
shown on display page 0:

| Reason | Meaning |
|--------|---------|
| `Ventilating` | Fan running, outside air is drier |
| `Diff too small` | Dew point difference below the threshold |
| `Indoor too cold` | Indoor temperature under its minimum |
| `Outdoor too cold` | Outdoor temperature under its minimum |
| `Sensor fault` | A sensor reads implausibly |
| `Manual: On` / `Manual: Off` | Mode override active |
| `Min runtime hold` | Wants to stop, minimum runtime not reached yet |
| `Min pause hold` | Wants to start, minimum pause not over yet |

Without this, the only outward signal was `Fan: OFF` — identical for a manual
stop, a cold night, a broken sensor and a difference that is simply too small.

---

## Hardware

| Qty | Part | Notes |
|-----|------|-------|
| 1 | ESP32 development board | The config targets `board: esp32dev` and the ESP-IDF framework |
| 2 | DHT22 / AM2302 | Temperature + humidity, indoor and outdoor |
| 1 | 20x4 character LCD with PCF8574 I²C backpack | Address `0x27` |
| 1 | DS1307 real-time clock module | Address `0x68`; keeps the clock without network |
| 1 | Relay breakout board | Switches the fan |
| 1 | Rotary encoder with push button | Menu navigation (e.g. KY-040) |

📄 **[Wiring diagram (PDF)](dew-point-ventilation.pdf)** — how the parts above are
connected. See [Pinout](#pinout) for the GPIO assignment in text form.

Notes:

- The DS1307 breakout usually carries an AT24C32 EEPROM at `0x50`. It shows up on
  an I²C scan but is not used by this firmware.
- The PCF8574 backpack has a **backlight jumper**. If it is missing, the display
  shows text but never lights up, and no firmware setting can change that.
- `minimum_chip_revision: "3.1"` is set in the config. The image will refuse to
  boot on an older ESP32 — relevant only if the board is ever replaced.

### Optional: SHT31 instead of DHT22

The YAML contains commented-out `sht3xd` blocks as a drop-in replacement, using
the same entity IDs so nothing downstream changes. Reasons to consider it:

- **Accuracy.** DHT22 is specified at ±2…5 % RH and drifts with age, which
  translates to up to ±1.2 °C of dew point error *per sensor*. Against a
  threshold of 1–9 °C that is a large share of the decision criterion. SHT31 is
  ±2 % RH and long-term stable.
- **No interrupt blocking.** The DHT driver reads inside an `InterruptLock`,
  roughly 4–5 ms per sensor with interrupts disabled. An I²C sensor removes this
  entirely.

SHT31 supports addresses `0x44` and `0x45` via its ADDR pin, so both sensors fit
on the existing bus alongside the LCD and RTC.

---

## Pinout

| Signal | GPIO | Notes |
|--------|------|-------|
| DHT22 indoor | 16 | Obsolete when switching to `sht3xd` |
| DHT22 outdoor | 17 | Obsolete when switching to `sht3xd` |
| I²C SDA | 21 | LCD `0x27`, RTC `0x68`, EEPROM `0x50` |
| I²C SCL | 22 | Bus runs at 100 kHz |
| Relay | 23 | Fan output |
| Encoder CLK | 25 | |
| Encoder DT | 26 | |
| Encoder switch | 27 | `INPUT_PULLUP`, inverted — the KY-040 board only pulls up CLK and DT |

The I²C bus stays at **100 kHz** because the DS1307 is a 100 kHz part. Do not
raise the frequency while the RTC shares the bus.

---

## Display

### Status bar

Row 0 is drawn on every page:

```
col  0    5         13    17 18 19
     HH:MM           !     %  o  ~
```

| Column | Meaning |
|--------|---------|
| 0–4 | Time from the RTC, `--:--` until it delivers a valid time |
| 13 | Attention icon on a failed OTA or an implausible sensor; blank when healthy |
| 17 | Fan rotor icon while the fan runs; blank while stopped |
| 18 | MQTT: blank = disabled, hollow disc = enabled but not connected, filled disc = broker connected |
| 19 | Wi-Fi: signal or no-signal glyph |

MQTT is deliberately three-valued rather than two-valued like Wi-Fi, so that
"switched on but not reaching the broker" is distinguishable at a glance.

### Pages

Turn the encoder to cycle through pages 0–3. Page 100 is reached only by the OTA
handlers.

| Page | Content |
|------|---------|
| 0 | **Decision page:** current dew point difference, the threshold it is compared against, and the reason for the current fan state |
| 1 | Indoor values on the left, outdoor on the right: temperature, humidity, dew point |
| 2 | Wi-Fi status: IP address and signal strength, or a note when disabled/connecting |
| 3 | Sensor errors, or "System is fine!" |
| 100 | OTA progress, forced to stay backlit for the duration of the update |

Page 0 is the default view because it answers the question the device exists for
— is it ventilating, and if not, why. The other pages show the inputs behind
that answer and are one click away. Previously the first page showed six numbers
from which you had to subtract two dew points in your head and then recall the
setpoint.

### Backlight

The backlight turns on for *Backlight delay* seconds on boot and on any user
interaction, then switches off again. While the menu is open it stays on.

The first press or turn on a dark display only wakes the backlight and is
otherwise swallowed, so nobody triggers a menu action blindly.

---

## Menu

Press the encoder button to open the menu, turn to navigate, press to confirm.

```
Mode                          Automatic / On / Off
Settings
  Dew point diff. min.
  Hysteresis
  In. temp. min.
  Out. temp. min.
  Back
System
  Date
    Day / Month / Year
  Time
    Hour / Minute / Second
  Backlight delay
  WiFi                        on/off
  MQTT                        on/off
  Restart
  Factory Reset
  Back
Exit Menu
```

Date and time are read from the RTC when the submenu is opened and written back
on leaving it. Impossible dates such as 31 February are rejected on apply.

---

## Settings

| Setting | Range | Step | Default | Meaning |
|---------|-------|------|---------|---------|
| Minimum dew point difference | 1.0 … 9.0 °C | 0.5 | 5.0 | How much drier outside air must be before ventilating |
| Hysteresis | 0.0 … 5.0 °C | 0.5 | 1.0 | Dead band above the minimum; prevents relay chatter |
| Indoor temperature minimum | 5 … 25 °C | 1 | 8 | Below this the fan stays off so the room is not cooled further |
| Outdoor temperature minimum | −30 … 20 °C | 1 | 5 | Below this the fan stays off |
| Backlight delay | 3 … 180 s | 1 | 10 | How long the display stays lit after an interaction |

All settings survive a restart. The date/time helper values are scratch values
and are not persisted.

### Compile-time constants

Not adjustable at runtime; change them in the `substitutions` block and reflash:

| Constant | Default | Meaning |
|----------|---------|---------|
| `MIN_RUN_TIME_S` | 300 s | Minimum time the fan keeps running before the dew point logic may stop it |
| `MIN_PAUSE_TIME_S` | 180 s | Minimum pause before it may start again |

They are constants rather than menu entries to keep the menu focused on the
settings that are actually worth changing during operation.

---

## Home Assistant / MQTT

Exposed entities:

**Sensors**

| Indoor | Outdoor | General |
|--------|---------|---------|
| Indoor Temperature | Outdoor Temperature | Delta Dew Point |
| Indoor Humidity | Outdoor Humidity | Absolute Humidity Delta |
| Indoor Dew Point | Outdoor Dew Point | WiFi Signal (dBm) / (%) |
| Indoor Absolute Humidity | Outdoor Absolute Humidity | |

**Diagnostics**

- `Fan State Reason` — why the fan is on or off, see
  [Why the fan is doing what it does](#why-the-fan-is-doing-what-it-does)
- `Fan Runtime` (hours) — whether the settings actually do anything; never
  running and always running are both worth noticing
- `Fan Switch Cycles` — wear indicator for the relay contacts

Runtime and cycle count survive a restart. A power cut loses whatever run was in
progress, because the preference is only written periodically.

**Other entities**

- Binary sensor: `Fan` (read-only mirror of the relay)
- Select: `Mode` — Automatic / On / Off
- Numbers: the four control settings above
- Switches: `WiFi`, `MQTT`
- Buttons: `Restart`, `Restart (Safe Mode)`, `Factory Reset`, `Firmware Update`
- Text sensors: `ESPHome Version`, `Firmware Version`, `Device Uptime`, `SSID`,
  `IP Address`, `DNS Address`

The relay itself is **internal on purpose**. `control_script` owns it, and a
second writer from Home Assistant would simply be overridden on the next
interval tick. To control the fan from outside, set `Mode` instead.

---

## Networking

### MQTT is disabled at boot

`enable_on_boot: False` is set for MQTT, and the switch uses
`restore_mode: ALWAYS_OFF`. MQTT must be switched on explicitly, either from the
LCD menu (*System → MQTT*) or via Home Assistant.

This is not a preference, it is the outcome of a measurement. Every time the MQTT
connection drops, the reconnect blocks ESPHome's main loop long enough for the
5 s task watchdog to reboot the device. Observed behaviour:

| Configuration | Result |
|---------------|--------|
| MQTT with discovery | watchdog reset every 2.5–5 minutes |
| `discovery: False` | one reset ~5 min after boot, then stable |
| MQTT start delayed by 90 s | no effect — reset ~1 min after MQTT connected |
| `keepalive: 60s` | 18 min, then 1 min 40 s — no reliable improvement |
| MQTT disabled | 50+ minutes clean |

On the device the trigger appears as
`tcp_read error, errno=Connection reset by peer` (socket errno 104), followed by
about 90 s of reconnect attempts and then the watchdog.

### Root cause: the broker is reached the long way round

The broker runs on the **same local network** as this device, but is addressed by
its public DynDNS name. The evidence:

```
mqtt.carstenwalther.de → mqtt-walther.dynv6.net → 92.206.232.135
public IP of the network the device sits in     → 92.206.232.135   (identical)
source address Mosquitto logs for this device   → 92.206.232.135
```

Every packet therefore travels device → router → NAT → back into the LAN →
broker. That the broker sees the WAN address instead of a LAN address proves the
router is source-NATing the hairpin. Consumer routers handle this badly: small
NAT tables, aggressive idle timeouts, and it is by far the least reliable path in
the house.

Mosquitto's own log confirms the consequence — the dominant disconnect reason is
`has exceeded timeout` (~35 occurrences in one afternoon) against ~9 protocol
errors.

**The fix is to point the device at the broker's LAN IP** instead of the DynDNS
name. No hairpin, no NAT table, no dependency on external DNS — and the
`Couldn't resolve IP address` warnings disappear with it.

`ALWAYS_OFF` is deliberate: after a watchdog reset the device must come back in
the state known to be stable, not the one that caused the reset.

> **Not the cause: a duplicate `client_id`.** The broker log does contain
> `already connected, closing old connection`, but every occurrence comes from
> the *same* source address, immediately followed by the same client
> reconnecting. That is this device racing its own stale session after a silent
> drop, not a second client. ESPHome derives the ID as device name + full MAC
> (`dew-point-ventilation-ec64c98652d0`), which does not collide by accident.
> The `mqtt_client.h` comment claiming truncation to 23 characters is stale —
> the buffer is `MAX_NAME_WITH_SUFFIX_SIZE = 128`.

### Home Assistant API

The API stays enabled. It is not implicated in the resets: the 50-minute clean
run above had the API active. ESPHome offers no runtime enable/disable for the
API component, so unlike Wi-Fi and MQTT it cannot be put behind a menu switch.

`reboot_timeout: 0s` is set for both API and MQTT — this device regulates on its
own and must not restart merely because Home Assistant or the broker is
unreachable.

---

## Safety behaviour

The fan is switched off:

- on shutdown (`on_shutdown`, priority 700 — early enough that the relay is still
  controllable),
- before a firmware update (`on_update`), so the fan is never left energised
  across a flash,
- on an OTA error,
- at boot, since the relay has no `restore_mode` and must start from a defined
  state,
- whenever a sensor reads implausibly.

### Sensor plausibility

A `NaN` check alone is not sufficient. A degraded DHT22 humidity element still
answers with a valid checksum — it simply reports a wrong value. This unit was
seen reporting a constant **1.0 % RH at 29.5 °C**, which yields a dew point of
−32 °C. That passes any `isnan()` test, and the resulting difference would keep
the fan off permanently without ever raising an error.

The check therefore includes a range test:

- **Indoor:** error if `NaN`, or humidity `< 5 %` or `> 99.5 %`
- **Outdoor:** error if `NaN`, or humidity `< 5 %`

The outdoor sensor deliberately has no upper bound: outdoor air genuinely sits at
100 % RH in fog or rain, and an upper bound would fault the sensor exactly when
it is reading correctly.

> The indoor upper bound of 99.5 % is not risk-free either. A damp cellar can
> genuinely reach 95–100 % RH — the very situation this device exists for. If it
> ever faults a correct reading, raise or drop that bound.

When a sensor is flagged, the attention icon appears in the status bar, page 0
reports `Sensor fault` as the reason, and page 3 names the affected sensor.

---

## Building and flashing

```bash
# validate
esphome config dew-point-ventilation.yaml

# compile
esphome compile dew-point-ventilation.yaml

# flash over USB
esphome run dew-point-ventilation.yaml --device /dev/cu.usbserial-0001

# flash over the network
esphome upload dew-point-ventilation.yaml --device <ip-address>

# live logs
esphome logs dew-point-ventilation.yaml --device <ip-address>
```

A `secrets.yaml` is required with: `wifi_ssid`, `wifi_password`,
`wifi_ap_password`, `api_encryption_key`, `ota_password`, `mqtt_broker`,
`mqtt_username`, `mqtt_password`.

The firmware can also be updated from the device itself via the *Firmware Update*
button, which pulls a build over HTTP. During any update the display switches to
page 100 and forces the backlight on, so progress stays visible.

### Framework

The config uses **ESP-IDF**, not Arduino. With the Arduino framework the
libsodium stub pulled in by arduino-esp32 shadows `esphome/libsodium`, which
breaks the noise-c build required by `api: encryption`. Consequence: no
Arduino-only APIs (`String`, `WiFi.*`) may be used in lambdas.

---

## Maintenance notes

**Display refresh cost.** A full 20x4 redraw is 84 bytes × 2 nibbles × 3
single-byte I²C transfers = 504 transfers, measured at roughly 170 ms on the
100 kHz bus. `lcd_base` redraws everything unconditionally, so this is paid on
every update. `update_interval` is set to 1 s to keep the main loop load under
20 %. Menu navigation and OTA progress stay responsive because both call
`update()` directly.

**Sensor intervals.** The climate sensors run at 30 s, not the 5 s originally
configured. `control_script` only evaluates every 10 s, so faster readings were
never used, and the DHT driver's interrupt-locked read is worth doing less often.
A cellar is thermally slow; 30 s is still far finer than anything the room does.
The visible cost is that breathing on a sensor takes up to 30 s to register.

**CGRAM is full.** All eight user character slots are in use: degrees, fan rotor,
no-signal, signal, attention, MQTT connected, MQTT disconnected, and the menu's
back marker. Adding another icon means retiring one. If the absolute humidity
values are ever put on the display, write the unit as `g/m3` rather than
reclaiming a slot for a superscript glyph.

**Time keeping.** The RTC is the primary time source and is read at boot; SNTP
corrects it when the network is available. Both use the `Europe/Berlin` timezone
and must be kept in sync — they share one global timezone setting, and the manual
date/time menu writes to the RTC.
