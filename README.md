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

Both dew points are derived from temperature and relative humidity by ESPHome's
`dew_point` platform, using the Magnus formula with the Alduchov–Eskridge
coefficients:

```
alpha = ln(RH / 100) + (17.625 · T) / (243.04 + T)
Td    = (243.04 · alpha) / (17.625 − alpha)
```

The platform publishes `NaN` when either input is `NaN` or the humidity falls
outside (0, 100] — it never substitutes a value.

> This used to be a hand-written template sensor with the Magnus–Tetens
> coefficients (17.67 / 243.5) and a clamp of the humidity to a minimum of 0.1 %.
> The clamp turned a 0 % reading into a plausible-looking dew point near −60 °C
> instead of an error. The two coefficient sets differ by less than 0.1 °C in
> this range, which is invisible against a threshold adjustable in 0.5 °C steps.
>
> The platform is also **callback-driven**: it recomputes the moment either
> source publishes. The template version polled on its own 30 s clock,
> independent of the sensors' 30 s clock, so the dew point the control loop read
> could be a full extra interval behind the measurement it came from.

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

   The pause timer is **backdated by a full pause at boot**, so the fan may start
   on the first evaluation after a restart. Without that the timestamp began at
   zero and the guard blocked every start for the first `MIN_PAUSE_TIME_S` — which
   during the watchdog-reset period covered most of the device's uptime. The
   trade-off is accepted deliberately: after a reset the fan can restart roughly a
   boot-time after it stopped, which is shorter than the nominal pause.

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
| 2 | SHT31 (`sht3xd`) | Temperature + humidity, indoor `0x44` and outdoor `0x45` |
| 1 | 20x4 character LCD with PCF8574 I²C backpack | Address `0x27` |
| 1 | DS1307 real-time clock module | Address `0x68`; keeps the clock without network |
| 1 | Relay breakout board | Switches the fan |
| 1 | Rotary encoder with push button | Menu navigation (e.g. KY-040) |

📄 **[Wiring diagram (PDF)](dew-point-ventilation.pdf)** — how the parts above are
connected. See [Pinout](#pinout) for the GPIO assignment in text form.

Notes:

- The two SHT31s are distinguished by their **ADDR pin**: tied to GND gives
  `0x44` (indoor), tied to VDD gives `0x45` (outdoor). Left floating, a board
  falls back to `0x44` and collides with the other sensor — both then answer at
  once, which can wedge SDA and take the whole bus down with it.
- **The DS1307 needs 5 V.** It is specified for 4.5–5.5 V, and with a backup cell
  fitted it switches to battery mode below roughly 1.25 × V<sub>BAT</sub> (~3.8 V
  with a 3 V cell) — in that mode it keeps counting time but **disables its I²C
  interface entirely**. Do not move it onto the 3.3 V rail when adding 3.3 V
  sensors. If you want 3.3 V operation, swap in a DS3231 (see
  [Maintenance notes](#maintenance-notes)).
- The DS1307 breakout carries an AT24C32 EEPROM at `0x50`. This firmware never
  talks to it, but it is a useful **diagnostic probe**: it shares VCC and both bus
  lines with the RTC, so "`0x50` answers, `0x68` does not" isolates a fault to the
  RTC chip itself rather than to power or wiring.
- The PCF8574 backpack has a **backlight jumper**. If it is missing, the display
  shows text but never lights up, and no firmware setting can change that.
- `minimum_chip_revision: "3.1"` is set in the config. The image will refuse to
  boot on an older ESP32 — relevant only if the board is ever replaced.

### Why SHT31 and not DHT22

The unit originally ran two DHT22/AM2302 on GPIO16 and GPIO17. The move to I²C
sensors was worth it on two counts:

- **Accuracy.** DHT22 is specified at ±2…5 % RH and drifts with age, which
  translates to up to ±1.2 °C of dew point error *per sensor*. Against a
  threshold of 1–9 °C that is a large share of the decision criterion. SHT31 is
  ±2 % RH and long-term stable.
- **No interrupt blocking.** The DHT driver read inside an `InterruptLock`,
  roughly 4–5 ms per sensor with interrupts disabled — a recurring blackout for
  the Wi-Fi and I²C ISRs. An I²C sensor removes this entirely.

⚠️ **One behavioural difference matters.** The two drivers report failure
differently:

| Driver | On a failed read |
|--------|------------------|
| `dht` | `publish_state(NAN)` — the sensor visibly goes invalid |
| `sht3xd` | `status_set_warning()` and **returns without publishing** |

The SHT31 therefore keeps its last value indefinitely when it stops answering,
which silently disabled the `isnan()` half of the plausibility check. A dead
sensor would have kept the fan regulating on a frozen reading. This is repaired
with a `timeout` filter on all four values — see
[Safety behaviour](#safety-behaviour). Do not remove those filters.

---

## Pinout

| Signal | GPIO | Notes |
|--------|------|-------|
| I²C SDA | 21 | Five devices, see below |
| I²C SCL | 22 | Bus runs at 100 kHz |
| Relay | 23 | Fan output |
| Encoder CLK | 25 | |
| Encoder DT | 26 | |
| Encoder switch | 27 | `INPUT_PULLUP`, inverted — the KY-040 board only pulls up CLK and DT |

GPIO16 and GPIO17 are free since the DHT22s were replaced.

### I²C bus

| Address | Device |
|---------|--------|
| `0x27` | LCD (PCF8574 backpack) |
| `0x44` | SHT31 indoor (ADDR → GND) |
| `0x45` | SHT31 outdoor (ADDR → VDD) |
| `0x50` | AT24C32 EEPROM on the RTC breakout — unused, diagnostic value only |
| `0x68` | DS1307 RTC |

The bus stays at **100 kHz** because the DS1307 is a 100 kHz part. The SHT31s
would do 1 MHz and the PCF8574 400 kHz; the RTC is the limit. Do not raise the
frequency while it shares the bus.

`timeout` (which maps to `scl_wait_us`, the clock-stretching timeout) is set to
**13 ms**, the maximum ESPHome accepts. It used to be 5 ms, on the then-correct
assumption that nothing on this bus stretches the clock — the SHT31s broke that:
`sht3xd::setup()` reads the serial number through register `0x3780`, explicitly
the clock-stretching variant. If a sensor stretches longer than `scl_wait_us`, the
master aborts mid-byte while the sensor still drives SDA, and a bus left in that
state takes every other device with it — the DS1307 included, whose `setup()`
calls `mark_failed()` on a single bad read and never retries.

Writing 13 ms out explicitly is deliberate: anything larger is silently clamped to
exactly that value, and the warning saying so is easy to miss.

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

Turn the encoder to cycle through pages 0–4. Page 100 is reached only by the OTA
handlers.

| Page | Content |
|------|---------|
| 0 | **Decision page:** current dew point difference, the threshold it is compared against, and the reason for the current fan state |
| 1 | Measurements: temperature, humidity and dew point, indoor (`I`) on the left, outdoor (`O`) on the right |
| 2 | Sensor errors, or "System is fine!" |
| 3 | Wi-Fi status: IP address and signal strength, or a note when disabled/connecting |
| 4 | Reset statistics: restart count and the cause of the last restart |
| 100 | OTA progress, forced to stay backlit for the duration of the update |

Page 0 is the default view because it answers the question the device exists for
— is it ventilating, and if not, why. The other pages show the inputs behind
that answer and are one click away. Previously the first page showed six numbers
from which you had to subtract two dew points in your head and then recall the
setpoint.

Each value on page 1 carries an `I`/`O` prefix. Without it the page was four
unlabelled numbers — the row tells you the quantity, but nothing said which
column was which, and there is no spare row for a header because row 0 belongs to
the status bar.

The encoder clamps at 0 and 4. When adding a page, both bounds in the
`rotary_encoder` handlers have to be moved with it.

### Backlight

The backlight comes on at boot and on **any** user input, and switches off once
*Backlight delay* seconds have passed **with no further input** — including while
the menu is open. Every input path funnels through one script that re-arms the
countdown, so navigating the menu keeps the display lit simply by being used.

The first press or turn on a dark display only wakes the backlight and is
otherwise swallowed, so nobody triggers a menu action blindly. That now applies
inside the menu too, where the old code navigated blind.

> Earlier versions instead froze the light on for as long as the menu was open,
> because menu navigation did not re-arm the timer. That also left the internal
> "display is lit" flag switched off while the display was in fact lit, and the
> next button press was swallowed by the wake-up branch instead of acting.

---

## Menu

Press the encoder button to open the menu, turn to navigate, press to confirm.

```
Mode                          Automatic / On / Off
Settings
  Dew point diff. min.
  Hysteresis
  Indoor min                  e.g. [  8°C]
  Outdoor min                 e.g. [  5°C]
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

The two temperature minimums show their unit on the LCD. That takes a
`value_lambda`: the `format` option is validated against a pattern that rejects
any trailing text, and the menu never looks at the number's
`unit_of_measurement`. Their labels are short because the value is right-aligned
into the same row and eats into the label — `Out. temp. min.` was already being
clipped before the unit was added.

Date and time are read from the RTC when the submenu is opened. On leaving, each
submenu writes back **only the half it edits** — the Date menu the calendar
fields, the Time menu the clock fields — and takes everything else from the
running clock. Writing all six back used to restore the hour, minute and second
captured on entry, setting the clock back by however long you spent in the menu,
and it did so even if nothing was edited.

Impossible dates such as 31 February are rejected on apply, as is a partial
update against an invalid clock.

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

The first four carry a unit, but not the same device class. The two temperature
minimums are readings (`device_class: temperature`); the dew point difference and
the hysteresis are *spans* between two temperatures and use
`device_class: temperature_delta`. Home Assistant converts the two differently,
and only the delta form is correct for a difference. The LCD menu ignores both
fields and needs its own `value_lambda` — see [Menu](#menu).

*Backlight delay* is `internal: True` and therefore never reaches Home Assistant
at all — it is an LCD-only setting and carries no unit.

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
- `Reset Count` — restarts since the last factory reset, counted once per boot
- `Reset Reason` — cause of the last restart, from `esp_reset_reason()`:
  `power-on event`, `software via esp_restart`, `task watchdog`,
  `interrupt watchdog`, `brownout`, …

- `Loop Time` (ms) — how long the main loop takes
- `Heap Free` (bytes) and `Heap Fragmentation` (%)

Read `Reset Count` and `Reset Reason` together, and against the uptime. The count
alone cannot tell an OTA apart from a crash; a rising count at short uptimes with
`task watchdog` as the cause is the signature this device has a documented
history of (see [Networking](#networking)). Both are also on display page 4.

`Loop Time` is the direct measurement of what actually failed during that
period — the main loop stalled while both CPUs sat in IDLE. The whole
measurement table in [Networking](#networking) was assembled without it, by
inference from reset timings. Watch it whenever MQTT is switched on. The two heap
sensors come along nearly free and would catch the other classic cause of late
resets, a slow leak or a fragmenting heap.

Runtime and cycle count survive a restart. A power cut loses whatever run was in
progress, because the preference is only written periodically. A factory reset
clears all of them, including the reset count.

**Other entities**

- Binary sensor: `Fan` (read-only mirror of the relay)
- Select: `Mode` — Automatic / On / Off
- Numbers: the four control settings above
- Switches: `WiFi`, `MQTT`
- Buttons: `Restart`, `Restart (Safe Mode)`, `Factory Reset`
- Text sensors: `ESPHome Version`, `Firmware Version`, `Device Uptime`,
  `Reset Reason`, `SSID`, `IP Address`, `DNS Address`

The relay itself is **internal on purpose**. `control_script` owns it, and a
second writer from Home Assistant would simply be overridden on the next
interval tick. To control the fan from outside, set `Mode` instead.

---

## Networking

### MQTT is disabled at boot

`enable_on_boot: False` is set for MQTT, and the switch uses
`restore_mode: ALWAYS_OFF`. MQTT must be switched on explicitly, either from the
LCD menu (*System → MQTT*) or via Home Assistant. Everything needed to regulate
works without it, so the device comes up in the cheapest possible state.

> ⚠️ **Those two settings must be changed together.** `TemplateSwitch::setup()`
> does not merely publish the restored state, it *acts* on it — calling
> `turn_on()` or `turn_off()`, which fires the switch's action. A switch left at
> `ALWAYS_OFF` therefore runs `mqtt.disable` moments after `enable_on_boot: True`
> brought the client up, and vice versa. The config validates either way; the
> only symptom is MQTT quietly not being in the state you configured.
>
> The same mechanism applies to the **WiFi** switch, which is why it is declared
> `restore_mode: DISABLED`. The schema default is `ALWAYS_OFF`, so without it the
> switch fired a full `wifi.disable()` on every boot — and early, at setup
> priority `HARDWARE - 2` (798) against a WiFi stack that only initialises at
> priority `WIFI` (250). Nothing visibly broke, because `WiFiComponent::setup()`
> then honours `enable_on_boot` and starts the radio anyway, but the round trip
> was pointless. That switch mirrors the radio through a lambda rather than
> owning it, so there is no state worth restoring in the first place.

`discovery: True` is set. Discovery messages are retained, so entities Home
Assistant already knows survive without it — but newly added entities never
arrive on their own while it is off.

The rest of this section is history. The resets described below were traced to
the broker and fixed there; the settings above are now a preference rather than a
workaround. Every time the MQTT connection dropped, the reconnect blocked
ESPHome's main loop long enough for the 5 s task watchdog to reboot the device:

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

### Resolution

**The cause was on the broker side and has been fixed there.** The device-side
configuration above is unchanged, and the network analysis that follows is kept
as a record of what was ruled out — not as an open action item.

#### Investigated and rejected: the broker is reached the long way round

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

Pointing the device at the broker's LAN IP was the obvious candidate fix, and it
is still the better addressing choice — no hairpin, no NAT table, no dependency
on external DNS. It was **not** what resolved the resets, though, so the hairpin
path is documented here as background rather than as the answer.

Note that the measurements above were all taken against the broken broker, and
say nothing about how the device behaves against a healthy one. `discovery` has
since been switched back on without issue. If MQTT is ever enabled at boot as
well, watch the `Loop Time` sensor — it measures the exact failure mode that
produced this table, and none of these rows had it available.

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

The check has two halves, and both are needed.

**1. Range test — catches a sensor that lies.** A `NaN` check alone is not
sufficient: a degraded humidity element still answers with a valid checksum, it
simply reports a wrong value. This unit was seen reporting a constant
**1.0 % RH at 29.5 °C**, which yields a dew point of −32 °C. That passes any
`isnan()` test, and the resulting difference would keep the fan off permanently
without ever raising an error.

- **Indoor:** error if `NaN`, or humidity `< 5 %` or `> 99.5 %`
- **Outdoor:** error if `NaN`, or humidity `< 5 %`

The outdoor sensor deliberately has no upper bound: outdoor air genuinely sits at
100 % RH in fog or rain, and an upper bound would fault the sensor exactly when
it is reading correctly.

> The indoor upper bound of 99.5 % is not risk-free either. A damp cellar can
> genuinely reach 95–100 % RH — the very situation this device exists for. If it
> ever faults a correct reading, raise or drop that bound.

**2. Staleness test — catches a sensor that goes silent.** All four SHT31 values
carry a `timeout: 90s` filter, which publishes `nan` when three consecutive polls
produce nothing. Without it the `NaN` half of the check above is dead code:
`sht3xd` does not publish on a failed read, so a sensor that stops answering
keeps its last plausible value forever and the fan keeps regulating on it. The
`dht` driver it replaced published `NAN` itself, which is what the original check
was written against. See
[Why SHT31 and not DHT22](#why-sht31-and-not-dht22).

When a sensor is flagged, the attention icon appears in the status bar, page 0
reports `Sensor fault` as the reason, and page 2 names the affected sensor.

---

## Building and flashing

Builds run in a Docker container, not on the workstation:

```bash
docker run --rm -v "${PWD}":/config -it \
  esphome/esphome compile dew-point-ventilation.yaml
```

Validation is cheap enough to run anywhere and does not compile anything:

```bash
# validate — resolves substitutions and lambdas, no toolchain needed
esphome config dew-point-ventilation.yaml

# flash over USB
esphome run dew-point-ventilation.yaml --device /dev/cu.usbserial-0001

# flash over the network
esphome upload dew-point-ventilation.yaml --device <ip-address>

# live logs
esphome logs dew-point-ventilation.yaml --device <ip-address>
```

`esphome config` catches schema and substitution errors but **not** C++ errors in
lambdas — those only surface in a real compile. Anything touching a lambda needs
the container build before it is flashed.

A `secrets.yaml` is required with: `wifi_ssid`, `wifi_password`,
`wifi_ap_password`, `api_encryption_key`, `ota_password`, `mqtt_broker`,
`mqtt_username`, `mqtt_password`.

### Logging

The logger runs at **`WARN`**. It used to be `ERROR`, which hid the two things
that mattered most during debugging: the I²C timeout-clamp warning, and sensor
retry warnings. The device logs nothing at `WARN` in normal operation, so the
extra level costs nothing until something is wrong.

The I²C bus scan is logged at `CONFIG` level and is therefore *still* invisible.
Raise the logger to `DEBUG` and set `scan: True` when you need it — that is how
you check which of the five addresses actually answer.

Only the `esphome` OTA platform is configured — the classic push from the
dashboard. (An `http_request` OTA path with a *Firmware Update* button, which
pulled a build from a web server, existed earlier and has been removed.) During
an update the display switches to page 100 and forces the backlight on, so
progress stays visible regardless of the backlight timer.

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
configured. `control_script` only evaluates every 10 s, so faster readings would
never be used, and every poll costs transfers on the same I²C bus the LCD already
keeps busy. (The original reason was the DHT driver's interrupt-locked read; that
cost is gone with the SHT31s, but the interval is still right.) A cellar is
thermally slow; 30 s is still far finer than anything the room does. The visible
cost is that breathing on a sensor takes up to 30 s to register.

**Setup priority is a real hazard in `on_boot`.** Components at the same priority
are set up in registration order, so an `on_boot` block races anything declared
at its own level. This config has been bitten twice:

- A **restoring global** is a component with `setup_priority::HARDWARE` (800). The
  reset counter is therefore incremented from a *second* `on_boot` block at
  priority **700**. At 800 it would race the `load()` that reads the stored
  value — add 1 to the initial 0, then have the restore overwrite it, and the
  count would never grow.
- `backlight_delay_id` is a template number, also at 800, registered after the
  `on_boot` trigger. Its state is still `NaN` during the boot backlight pulse,
  which is why the `delayed_off` filter falls back to a hard-coded default
  instead of casting `NaN` to `uint32_t`.

When adding anything to `on_boot` that reads restored state, put it below 800.

**CGRAM is full.** All eight user character slots are in use: degrees, fan rotor,
no-signal, signal, attention, MQTT connected, MQTT disconnected, and the menu's
back marker. Adding another icon means retiring one. If the absolute humidity
values are ever put on the display, write the unit as `g/m3` rather than
reclaiming a slot for a superscript glyph.

**Time keeping.** The RTC is the primary time source and is read at boot; SNTP
corrects it when the network is available. Both use the `Europe/Berlin` timezone
and must be kept in sync — they share one global timezone setting, and the manual
date/time menu writes to the RTC.

**If the RTC reports a communication error.** `DS1307Component::setup()` calls
`mark_failed()` on a single bad read and never retries, so the verdict is decided
by the very first I²C transaction after boot and only a reboot clears it. Work
through it in this order:

1. **Measure V<sub>CC</sub> at the module.** The DS1307 needs 4.5–5.5 V and
   silently disables its I²C interface in battery-backup mode. This is by far the
   most common cause after rewiring.
2. **Run a bus scan** (`scan: True` plus logger at `DEBUG`). If `0x50` answers but
   `0x68` does not, power and wiring reach the board and the RTC chip itself is
   the problem — which points straight back to step 1.
3. **Check the SHT31 ADDR pins.** A floating ADDR puts both sensors on `0x44`;
   two devices answering at once can wedge SDA and take the RTC down with it.

A **DS3231** is the upgrade path if you want 3.3 V operation: same `0x68`, same
ZS-042 form factor, same `0x50` EEPROM, far more accurate, and it runs from
2.3 V. ESPHome has no `ds3231` time platform, but the DS3231 is layout-compatible
with the DS1307 in the timekeeping registers `0x00`–`0x06`, so `platform: ds1307`
drives it unchanged. The caveat: the `CH` bit in register `0x00` and the century
bit in `0x05` mean something different there — both are read and written as 0,
which is harmless here.
