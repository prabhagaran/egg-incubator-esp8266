# Egg Incubator — ESP8266 Firmware

A compact incubator controller firmware for ESP8266-based boards (NodeMCU, Wemos D1 mini, etc.).  
This project implements temperature sensing, heater control, alarms, a simple OLED UI with rotary encoder input, Wi‑Fi/NTP support and persistent settings stored in EEPROM.

> NOTE: This README describes the current implementation (what's done) and the suggested future scope.

---

## Table of contents
- [Features implemented](#features-implemented)
- [Hardware](#hardware)
- [Wiring / Pins](#wiring--pins)
- [Software dependencies](#software-dependencies)
- [Quick start (build & flash)](#quick-start-build--flash)
- [Configuration & defaults](#configuration--defaults)
- [User interface (how to use)](#user-interface-how-to-use)
- [Safety & limits](#safety--limits)
- [EEPROM layout](#eeprom-layout)
- [Future scope / roadmap](#future-scope--roadmap)
- [Contributing](#contributing)
- [License](#license)

---

## Features implemented

- Temperature sensing
  - DS18B20 (OneWire + DallasTemperature). Non-blocking conversion (setWaitForConversion(false)) with filtering.
  - Raw + filtered temperature handling and sensor validity checks.

- Heater control
  - AUTO mode (hysteresis-based) and MANUAL mode.
  - Stress protection (minimum ON / OFF times to protect the heater).
  - Hard safety cutoff to immediately turn heater off above an absolute max temperature.

- Alarms
  - Alarm types: sensor fault, over-temperature, under-temperature.
  - Alarm finite-state machine with ACTIVE and LATCHED states (latched alarms force heater off).
  - Blink/visual indicator for critical latched alarms.
  - Ability to ACK latched alarm from the Home screen after condition is resolved.

- Persistence
  - Settings saved in EEPROM (versioned). Safe defaults and validation on load.
  - Dirty-flush commit (debounced writes) to reduce EEPROM wear.

- Time & incubation tracking
  - NTP sync (configTime) to compute incubation day and hatch date.
  - Incubation start date/time editing and day calculation (21-day incubations supported as default).

- UI
  - SSD1306 OLED driver (Adafruit_SSD1306).
  - Rotary encoder + push button (ezButton) for navigation and parameter editing.
  - Multiple screens: Home, Menu, Incubation, Status, Wi‑Fi, Settings, Temperature edit & confirm, Alarms.
  - Visual indicators for Wi‑Fi, time sync, heater state, and alarm icon.

- Wi‑Fi
  - WiFiManager for simple provisioning (config portal).

- Input validation & constraints
  - Runtime and EEPROM validation for setpoints, safe ranges, hysteresis and logical constraints (min < max).
  - UI-enforced live constraints when editing temperature values.

---

## Hardware

Minimum components
- ESP8266 board (NodeMCU, Wemos D1 mini, etc.)
- DS18B20 temperature sensor
- OLED display (SSD1306 compatible, I2C)
- Rotary encoder with push button
- MOSFET / relay / driver for heater control (properly sized)
- Optional: LEDs / buzzer for alarms

---

## Wiring / Pins (as implemented in code)

- CLK_PIN — D7 (rotary encoder CLK)
- DT_PIN — D6 (rotary encoder DT)
- SW_PIN  — D5 (rotary encoder switch / button)
- HEATER_PIN — D0 (heater control output)
- ONE_WIRE_BUS — D4 (DS18B20)
- OLED I2C using Wire on (D2=SDA, D1=SCL) with address 0x3D

Adjust pins to your board if required — verify in `setup()` and definitions.

---

## Software dependencies

Install these Arduino/PlatformIO libraries:
- ESP8266WiFi
- WiFiManager
- ESP8266HTTPClient
- WiFiClientSecure
- ezButton
- Wire (built-in)
- Adafruit_GFX
- Adafruit_SSD1306
- OneWire
- DallasTemperature
- EEPROM (built-in)

Recommended environment: Arduino IDE or PlatformIO with an ESP8266 board selected.

---

## Quick start (build & flash)

1. Install the listed libraries.
2. Select your ESP8266 board and correct COM/flash settings.
3. Compile & flash the firmware.
4. On first boot the WiFiManager AP will appear as `EggIncubator_Setup` — connect and configure your Wi‑Fi.
5. Wait for NTP sync (or set time via menu) before starting incubation for accurate day counting.

---

## Configuration & defaults

Default values are set in the code and restored on EEPROM version mismatch:
- Set temperature: 37.5 °C
- Hysteresis: 0.3 °C
- Heater mode: AUTO
- Max safe temp: 39.5 °C
- Min safe temp: 35.0 °C
- Alarms: enabled

Runtime constraints enforced (current code):
- Target setTemp constrained to 30.0 — 40.0 °C
- Max safe temp constrained to 38.0 — 42.0 °C
- Min safe temp constrained to 30.0 — 37.0 °C
- Hysteresis constrained to 0.1 — 1.0 °C
- HARD_MAX_TEMP = 45.0 °C (absolute cutoff)

(If you want a wider set temperature range, the validation/constrain values and EEPROM defaults need updating — see "Future scope".)

---

## User interface (how to use)

- Rotate encoder to move selection / change values.
- Press encoder button to select / advance / confirm.
- Home screen shows time, temp, heater status, incubation day and hatch date.
- Press from Home to open the Main Menu:
  - Incubation: Start / Info / Reset
  - Status: two pages with live metrics
  - WiFi: provisioning and status
  - Settings: Set Temperature, Heater Mode, Hysteresis, Alarms
- Editing temperatures: go to Settings → Set Temperature. Rotate to change Target / Max / Min. Press to move to next field, then Confirm.
- Alarm ACK: when an alarm is latched, press button on Home screen to attempt ACK (only clears if condition is resolved).

---

## Safety & limitations

- The firmware includes multiple safety layers, but wiring and hardware decisions are critical. Always:
  - Use properly rated heater drivers (SSR/MOSFET/relay) and fuse protection.
  - Provide a separate hardware high‑temperature thermal fuse / mechanical thermostat as a failsafe.
  - Test carefully with no eggs installed before actual use.
- The HARD_MAX_TEMP is a last-resort software cutoff — do not rely on it exclusively for safety.

---

## EEPROM layout (addresses used)

- ADDR_EEPROM_VERSION (0) — uint8_t
- ADDR_SET_TEMP (1) — float (4 bytes)
- ADDR_HYSTERESIS (5) — float
- ADDR_HEATER_MODE (9) — uint8_t
- ADDR_MANUAL_STATE (10) — uint8_t
- ADDR_INCUBATION_STARTED (11) — uint8_t
- ADDR_INCUBATION_EPOCH (12) — uint32_t (4 bytes)
- ADDR_MAX_SAFE_TEMP (16) — float
- ADDR_MIN_SAFE_TEMP (20) — float
- ADDR_ALARMS_ENABLED (24) — uint8_t

The code performs validation and falls back to defaults if values are out-of-range or version mismatch occurs.

---

## Future scope / roadmap (suggested improvements)

Short-term (easy / high impact)
- Extend configurable set temperature range (update validation constraints and UI ranges).
- Add calibration offset for the DS18B20 (configurable).
- Make NTP server and GMT offset configurable via Wi‑Fi config portal.
- Provide an option to set temperature units (Celsius / Fahrenheit) and do unit conversion in UI.

Medium-term
- Add humidity sensing and control (DHT22 / SHTxx + humidifier control + turning schedule).
- Add egg-turning mechanism control and schedule.
- Add OTA (Over-The-Air) firmware updates via Wi‑Fi.
- Implement a simple web UI / REST API for remote monitoring and setpoint changes.
- Add logging of temperature & events to SPIFFS / LittleFS / external SD for diagnostics.

Long-term / advanced
- Implement PID temperature control (instead of simple hysteresis) with autotune.
- Add push notifications (Telegram / Pushbullet / Email) for critical alarms.
- Add secure cloud integration and authentication.
- Support multiple temperature probes and sensor redundancy / voting.
- Add unit tests / CI and hardware-in-the-loop test harness.

---

## Contributing

Contributions welcome — please:
1. Fork the repository.
2. Make changes on a feature branch.
3. Open a pull request describing the change and rationale.
4. Keep safety and hardware implications in mind for any control logic changes.

---

## License

Specify a license for this repository (e.g. MIT). Add license file in the repo root.

---