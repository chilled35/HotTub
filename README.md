# 🛁 HA Hot Tub Controller

A fully featured Home Assistant smart hot tub controller built around an **Octopus Cosy tariff**, a **Zigbee smart plug**, **solar production monitoring**, and a **custom OLED/touch sensor unit** built on a Wemos D1 Mini running ESPHome.

The system automatically heats the tub during cheap electricity windows, pauses during peak hours, and fires up when surplus solar is available — all while exposing a polished standalone HTML dashboard and a physical wall-mounted control unit next to the tub.

---

## Features

- **Tariff-aware scheduling** — Octopus Cosy windows (Cheap 04:00–07:00 / 13:00–16:00 / 22:00–00:00, Peak 16:00–19:00) built in
- **Solar heating** — turns on automatically when solar production exceeds a configurable watt threshold
- **Peak block** — hard-forces the tub off 16:00–19:00 regardless of other conditions
- **Physical override** — capacitive touch sensor on the OLED unit activates 30/60/90 min override with countdown display
- **Dashboard override** — override also controllable from the web dashboard with configurable duration
- **OLED status display** — 128×64 OLED shows current tariff, water temperature, and countdown timer during override
- **iOS push notifications** — daily, weekly, and monthly energy/cost reports to iPhone
- **Energy & cost tracking** — per-tariff kWh split (Cheap / Standard / Peak / Solar) using a rolling 31-day WebSocket history calculator; immune to HA restarts
- **Voltage monitoring** — alerts if plug voltage falls outside 210–250V
- **Custom HTML dashboard** — standalone panel with live WebSocket connection; no Lovelace cards required

---

## Hardware

### Hot Tub Control
| Component | Details |
|---|---|
| Smart plug | Zigbee plug with energy monitoring (power, voltage, energy sensors) |
| Entity | `switch.appliance_hottub` |

### OLED Controller Unit (wall-mounted, weatherproof)
| Component | Details |
|---|---|
| MCU | Wemos D1 Mini (ESP8266) |
| Display | SSD1309 2.42" 128×64 OLED (I2C, SSD1306-compatible driver) |
| Temperature sensor | DS18B20 waterproof probe, 1000mm cable, OneWire on D5 |
| Touch sensor | TTP223 capacitive touch, signal on D6 |
| Enclosure | IP-rated waterproof box, clear acrylic lid, UV resin potted |
| Power | 12VDC → 5VDC buck converter |

### Wiring
| Signal | D1 Mini Pin | GPIO |
|---|---|---|
| I2C SCL (OLED) | D1 | GPIO5 |
| I2C SDA (OLED) | D2 | GPIO4 |
| DS18B20 Data | D5 | GPIO14 (internal pull-up) |
| TTP223 Signal | D6 | GPIO12 (inverted) |

---

## Software Stack

| Layer | Technology |
|---|---|
| Home Automation | Home Assistant |
| ESP Firmware | ESPHome |
| HA Package | YAML packages (`packages/hottub/`) |
| Dashboard | Standalone HTML + HA WebSocket API |
| Notifications | HA mobile companion app (iOS) |

---

## Repository Structure

```
ha-hottub-controller/
│
├── esphome/
│   └── hottub-temperature.yaml         ESPHome firmware for D1 Mini
│
├── packages/
│   └── hottub/
│       ├── hottub_helpers.yaml         All input_number, input_boolean,
│       │                               input_select, timer helpers
│       ├── hottub_sensors.yaml         Template sensors and binary sensors
│       ├── hottub_automations.yaml     Smart controller, override, tariff
│       │                               switcher, peak block, daily store,
│       │                               ESPHome bridge automations
│       └── hottub_notifications.yaml   Daily / weekly / monthly iOS push
│
├── www/
│   └── hottub/
│       └── dashboard.html              Standalone HTML dashboard
│
├── configuration.yaml.example          packages: block to add to HA config
└── README.md
```

---

## Installation

### 1. Home Assistant Package Files

Copy the `packages/hottub/` folder to `/config/packages/hottub/` on your HA instance.

Add the packages block to your `configuration.yaml`:

```yaml
homeassistant:
  packages:
    hottub: !include_dir_named packages/hottub
```

Restart Home Assistant.

### 2. Dashboard

Copy `www/hottub/dashboard.html` to `/config/www/hottub/dashboard.html`.

Access at: `http://your-ha-ip:8123/local/hottub/dashboard.html`

On first load the dashboard will prompt for your HA URL and a long-lived access token. These are stored in `localStorage`.

### 3. ESPHome Firmware

1. Copy `esphome/hottub-temperature.yaml` to `/config/esphome/`
2. Copy the `ConsidermevexedRegular-ExLe.ttf` font file to `/config/esphome/fonts/`  
   (Download from [FontSpace — ConsiderMeVexed by Chequered Ink](https://www.fontspace.com/considermevexed-font-f19295))
3. Edit the YAML to set your WiFi credentials in `secrets.yaml`
4. Flash via the ESPHome dashboard (USB first flash, OTA thereafter)
5. Adopt the device in HA: Settings → Devices & Services → ESPHome

### 4. Notifications

Update `notify.mobile_app_jasons_iphone` in `hottub_notifications.yaml` to match your device name, found at Developer Tools → Services → search `notify.mobile_app`.

---

## Entity Reference

### Key Entities (must exist in your HA)

| Entity | Purpose |
|---|---|
| `switch.appliance_hottub` | Zigbee plug — controls tub power |
| `sensor.appliance_hottub_power` | Running wattage (W) |
| `sensor.appliance_hottub_voltage` | Supply voltage (V) |
| `sensor.appliance_hottub_energy` | Cumulative energy (kWh) |
| `sensor.power_meter` | Solar production (kW) |

### Created by this package

| Entity | Purpose |
|---|---|
| `sensor.hot_tub_electricity_rate` | Current Cosy tariff rate (p/kWh) |
| `sensor.hot_tub_today_cost` | Today's running cost (pence) |
| `sensor.hot_tub_week_cost` | This week's cost (pence) |
| `sensor.hot_tub_month_cost` | This month's cost (pence) |
| `binary_sensor.hot_tub_cheap_rate_active` | True during Cosy cheap windows |
| `binary_sensor.hot_tub_peak_block_active` | True 16:00–19:00 |
| `binary_sensor.hot_tub_solar_sufficient` | True when solar ≥ threshold |
| `binary_sensor.hot_tub_voltage_alert` | True if voltage outside 210–250V |
| `input_boolean.hottub_schedule_solar` | Enable solar heating schedule |
| `input_boolean.hottub_schedule_cheap` | Enable cheap rate schedule |
| `input_boolean.hottub_in_use_override` | Manual override active |
| `input_select.hottub_control_mode` | Current mode (Solar/Cheap Rate/Override/Peak Block/Off) |
| `timer.hottub_override_timer` | Dashboard override countdown |

### ESPHome entities (D1 Mini)

| Entity | Purpose |
|---|---|
| `sensor.hottub_temperature_temperature` | DS18B20 water temperature |
| `sensor.hottub_temperature_wifi_signal` | ESP WiFi signal (dBm) |
| `binary_sensor.hottub_temperature_override_active` | Physical override countdown active |
| `binary_sensor.hottub_temperature_touch_sensor` | Touch sensor state |

---

## Smart Controller Logic

The main automation (`hottub_smart_controller`) runs every 5 minutes and on key state changes. It evaluates conditions in strict priority order:

```
Priority 1 — PEAK BLOCK    (16:00–19:00 and override not active)  → OFF
Priority 2 — OVERRIDE      (input_boolean.hottub_in_use_override)  → ON
Priority 3 — SOLAR         (solar_sufficient AND schedule_solar)   → ON
Priority 4 — CHEAP RATE    (cheap_rate_active AND schedule_cheap)  → ON
Default    — ALL OTHER                                              → OFF
```

---

## Override System

Override can be activated two ways:

**Physical (touch sensor):**
- Tap 1: 30 min countdown starts, tub on
- Tap 2: extend to 60 min
- Tap 3: extend to 90 min (maximum)
- Tap 4: cancel, tub off
- OLED shows MM:SS countdown
- ESPHome `binary_sensor.hottub_temperature_override_active` bridges to HA

**Dashboard:**
- Toggle button activates override for `hottub_override_duration` hours (default 3h)
- HA `timer.hottub_override_timer` counts down, cancels automatically
- iOS notification sent on expiry

---

## Octopus Cosy Tariff Windows

| Period | Times | Default rate |
|---|---|---|
| Cheap | 04:00–07:00 | ~12p/kWh |
| Standard | 07:00–13:00 | ~24p/kWh |
| Cheap | 13:00–16:00 | ~12p/kWh |
| Peak | 16:00–19:00 | ~38p/kWh |
| Standard | 19:00–22:00 | ~24p/kWh |
| Cheap | 22:00–00:00 | ~12p/kWh |

Rates are configurable via `input_number.hottub_rate_*` — update them to match your exact contract.

---

## Dashboard

The dashboard is a self-contained single HTML file that connects to HA via the WebSocket API. No server-side rendering, no Lovelace, no custom components required — just drop it in `/config/www/hottub/` and open it in a browser.

**Sections:**
- Temperature — live water temp, target, voltage alert
- Mode — current control mode badge
- Solar — live production vs threshold
- Electricity rate — current tariff and rate
- Energy & cost — Today / This Week / This Month with tariff split (Cheap / Standard / Peak / Solar ☀️ FREE)
- Daily breakdown — last 7 days kWh and cost
- Schedule toggles — Solar / Cheap Rate / Standard Rate
- Manual override — duration slider + activate button
- Settings — solar threshold, override duration, tariff rates

---

## Configuration

All user-facing settings are in `hottub_helpers.yaml` and exposed on the dashboard Settings panel:

| Setting | Default | Description |
|---|---|---|
| Solar threshold | 2000 W | Minimum solar production to activate solar heating |
| Override duration | 3 h | Dashboard override auto-cancel time |
| Cheap rate | 12.0 p/kWh | Your Octopus Cosy cheap rate |
| Standard rate | 24.0 p/kWh | Your Octopus Cosy standard rate |
| Peak rate | 38.0 p/kWh | Your Octopus Cosy peak rate |

---

## Notes

- `sensor.power_meter` is expected to report solar production in **kW** (not W). The solar sufficient binary sensor multiplies by 1000 to compare against the W threshold.
- The energy history calculator in the dashboard uses a rolling 31-day lookback window to avoid the month-boundary zero bug (data resets to 1st of new month before old month data is fully available).
- The TTP223 touch sensor is wired with `inverted: true` in ESPHome — it idles HIGH and pulls LOW on touch.
- The font `ConsidermevexedRegular-ExLe.ttf` is not included in this repository as it is a third-party font. Download it free from FontSpace (link above).
