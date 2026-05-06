# 🛁 HA Hot Tub Controller

A fully featured Home Assistant smart hot tub controller built around an **Octopus Cosy tariff**, a **Zigbee smart plug**, **solar production monitoring**, and a **custom OLED/touch sensor unit** built on a Wemos D1 Mini running ESPHome.

The system automatically heats the tub during cheap electricity windows, pauses during peak hours, and fires up when surplus solar is available — all while exposing a polished standalone HTML dashboard and a physical wall-mounted control unit next to the tub.

---

## Features

- **Tariff-aware scheduling** — Octopus Cosy windows (Cheap 04:00–07:00 / 13:00–16:00 / 22:00–00:00, Peak 16:00–19:00) built in; each cheap window independently toggleable
- **Solar heating** — turns on automatically when solar production exceeds a configurable watt threshold, with a 5-minute debounce to filter cloud-cover flapping
- **Solar temperature boost** — automatically raises the target temperature to 40 °C when solar is above 2 kW (so the tub runs a full heat cycle rather than cycling on/off at 38 °C); reverts to 38 °C when solar drops below 1.5 kW
- **Peak block** — hard-forces the tub off 16:00–19:00 regardless of other conditions
- **Physical override** — capacitive touch sensor on the OLED unit activates 30/60/90 min override with countdown display
- **Dashboard override** — override also controllable from the web dashboard with configurable duration
- **OLED status display** — 128×64 OLED shows water temperature in standby; HH:MM:SS countdown during override
- **iOS push notifications** — daily, weekly, and monthly energy/cost reports to iPhone
- **Energy & cost tracking** — server-side `utility_meter` entities accumulate kWh per tariff (Cheap / Standard / Peak / Solar) continuously in HA — accurate regardless of restarts or whether the dashboard is open
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
| Display | SSD1309 2.42\" 128×64 OLED (I2C, SSD1306-compatible driver) |
| Temperature sensor | DS18B20 waterproof probe, 1000mm cable, OneWire on D5 |
| Touch sensor | TTP223 capacitive touch, signal on D6 |
| Enclosure | IP-rated waterproof box, clear acrylic lid, UV resin potted |
| Power | 12VDC → 5VDC buck converter |

### Wiring
| Signal | D1 Mini Pin | GPIO |
|---|---|---|
| I2C SCL (OLED) | D1 | GPIO5 |
| I2C SDA (OLED) | D2 | GPIO4 |
| DS18B20 Data | D5 | GPIO14 (4.7kΩ pull-up to 3.3V) |
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
│       ├── hottub_helpers.yaml         input_number, input_boolean,
│       │                               input_select, timer helpers
│       ├── hottub_sensors.yaml         utility_meter energy tracking,
│       │                               template sensors, binary sensors
│       ├── hottub_automations.yaml     Smart controller, override, tariff
│       │                               switcher, peak block, daily store,
│       │                               ESPHome bridge automations
│       ├── hottub_solar_automations.yaml  Solar trigger (on/off with 5-min
│       │                               debounce) and solar boost (target
│       │                               temp raise/revert)
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

## Deployment

These files deploy to `/config/` on the HA instance:

| Repo file | HA path |
|---|---|
| `packages/hottub/hottub_helpers.yaml` | `/config/packages/hottub/hottub_helpers.yaml` |
| `packages/hottub/hottub_sensors.yaml` | `/config/packages/hottub/hottub_sensors.yaml` |
| `packages/hottub/hottub_automations.yaml` | `/config/packages/hottub/hottub_automations.yaml` |
| `packages/hottub/hottub_solar_automations.yaml` | `/config/packages/hottub/hottub_solar_automations.yaml` |
| `packages/hottub/hottub_notifications.yaml` | `/config/packages/hottub/hottub_notifications.yaml` |
| `esphome/hottub-temperature.yaml` | `/config/esphome/hottub-temperature.yaml` |
| `www/hottub/dashboard.html` | `/config/www/hottub/dashboard.html` |

`configuration.yaml.example` shows the `packages:` block that must exist in HA's `configuration.yaml`.

The font `ConsidermevexedRegular-ExLe.ttf` is not in the repo — it must be downloaded separately and placed at `/config/esphome/fonts/`.

---

## Installation

### 1. Home Assistant Package Files

Copy the `packages/hottub/` folder to `/config/packages/hottub/` on your HA instance.

Add the packages block to your `configuration.yaml`:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

**Full restart** is required after first install — `utility_meter` entities need a restart to initialise. After restarting, manually trigger the **Hot Tub — Tariff Switcher** automation once from the Automations UI to set the correct starting tariff on the new meters.

### 2. Dashboard

Copy `www/hottub/dashboard.html` to `/config/www/hottub/dashboard.html`.

Access at: `http://your-ha-ip:8123/local/hottub/dashboard.html`

On first load the dashboard will prompt for your HA URL and a long-lived access token. These are stored in `localStorage`.

### 3. ESPHome Firmware

1. Copy `esphome/hottub-temperature.yaml` to `/config/esphome/`
2. Copy the `ConsidermevexedRegular-ExLe.ttf` font file to `/config/esphome/fonts/`
   (Download from [FontSpace — ConsiderMeVexed by Chequered Ink](https://www.fontspace.com/considermevexed-font-f19295))
3. Add WiFi credentials and passwords to `secrets.yaml`
4. Update the DS18B20 `address:` in the YAML to match your sensor (scan the bus on first flash to find it)
5. Flash via the ESPHome dashboard (USB first flash, OTA thereafter)
6. Adopt the device in HA: Settings → Devices & Services → ESPHome

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
| `sensor.appliance_hottub_energy` | Cumulative energy (kWh, resets at midnight) |
| `sensor.power_meter` | Solar production (**kW** — multiplied ×1000 internally) |

### Created by this package — Helpers

| Entity | Purpose |
|---|---|
| `input_number.hottub_target_temp` | Target water temperature (°C) |
| `input_number.hottub_solar_threshold` | Min solar production to activate solar heating (W) |
| `input_number.hottub_override_duration` | Dashboard override auto-cancel time (h) |
| `input_number.hottub_rate_cheap` | Cheap tariff rate (p/kWh) |
| `input_number.hottub_rate_standard` | Standard tariff rate (p/kWh) |
| `input_number.hottub_rate_peak` | Peak tariff rate (p/kWh) |
| `input_number.hottub_yesterday_energy` | Yesterday's kWh snapshot (saved at 23:55) |
| `input_number.hottub_yesterday_cost` | Yesterday's cost snapshot in pence (saved at 23:55) |
| `input_boolean.hottub_schedule_enabled` | Master on/off gate for all scheduled heating |
| `input_boolean.hottub_schedule_solar` | Enable solar heating |
| `input_boolean.hottub_schedule_cheap_0407` | Enable cheap rate 04:00–07:00 window |
| `input_boolean.hottub_schedule_cheap_1316` | Enable cheap rate 13:00–16:00 window |
| `input_boolean.hottub_schedule_cheap_2200` | Enable cheap rate 22:00–00:00 window |
| `input_boolean.hottub_schedule_standard` | Enable standard rate heating |
| `input_boolean.hottub_in_use_override` | Manual override active |
| `input_select.hottub_control_mode` | Current mode (Solar / Cheap Rate / Override / Peak Block / Off) |
| `timer.hottub_override_timer` | Dashboard override countdown |

### Created by this package — Sensors

| Entity | Purpose |
|---|---|
| `sensor.hot_tub_electricity_rate` | Current Cosy tariff rate (p/kWh) with `tariff` and `next_change` attributes |
| `sensor.hot_tub_today_total_kwh` | Today's total kWh (all tariffs combined) |
| `sensor.hot_tub_today_cost` | Today's running cost (pence) |
| `sensor.hot_tub_week_cost` | This week's cost (pence) |
| `sensor.hot_tub_month_cost` | This month's cost (pence) |
| `binary_sensor.hot_tub_cheap_rate_active` | True during Cosy cheap windows |
| `binary_sensor.hot_tub_peak_block_active` | True 16:00–19:00 |
| `binary_sensor.hot_tub_solar_sufficient` | True when solar ≥ threshold |
| `binary_sensor.hot_tub_voltage_alert` | True if voltage outside 210–250V |

### Created by this package — Utility Meters

Each meter has four tariff slots: `cheap`, `standard`, `peak`, `solar`. The active tariff is switched by the tariff switcher automation at every Cosy boundary and when solar crosses the threshold.

| Entity pattern | Cycle |
|---|---|
| `sensor.hottub_energy_daily_<tariff>` | Resets daily |
| `sensor.hottub_energy_weekly_<tariff>` | Resets weekly (Monday) |
| `sensor.hottub_energy_monthly_<tariff>` | Resets monthly (1st) |

The corresponding select entities (`select.hottub_energy_daily/weekly/monthly`) control the active tariff slot.

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
Priority 0 — MASTER GATE   (hottub_schedule_enabled OFF, override not active) → OFF
Priority 1 — PEAK BLOCK    (16:00–19:00 and override not active)              → OFF
Priority 2 — OVERRIDE      (input_boolean.hottub_in_use_override ON)          → ON
Priority 3 — SOLAR         (solar_sufficient AND schedule_solar)               → ON
Priority 4 — CHEAP RATE    (cheap_rate_active AND schedule_cheap_*)            → ON
Default    — ALL OTHER                                                          → OFF
```

The master gate (`input_boolean.hottub_schedule_enabled`) must be ON for any scheduled heating. Manual override bypasses it intentionally.

---

## Solar Heating

Solar heating is handled by a dedicated automation file (`hottub_solar_automations.yaml`) with four automations.

### Solar Trigger (on/off)

| Automation | Behaviour |
|---|---|
| `hottub_solar_trigger_on` | Powers tub on when solar production has been above the threshold for **5 continuous minutes**. Checks master gate and override. |
| `hottub_solar_trigger_off` | Powers tub off when solar drops below the threshold for 5 minutes. Only acts when mode is Solar. |

The 5-minute debounce prevents rapid cycling during patchy cloud cover.

### Solar Boost (target temperature)

When solar is plentiful the tub can reach its target temperature quickly and begin short on/off cycles. The solar boost prevents this by raising the target while solar is active:

| Automation | Trigger | Action |
|---|---|---|
| `hottub_solar_boost_target_on` | Solar > 2 kW for 5 min | Raises `input_number.hottub_target_temp` to **40 °C** |
| `hottub_solar_boost_target_off` | Solar < 1.5 kW for 5 min | Reverts `input_number.hottub_target_temp` to **38 °C** |

The 0.5 kW hysteresis (on at 2 kW, off at 1.5 kW) prevents flapping on days with variable cloud. Both automations also trigger on HA start and automation reload so the correct target is applied immediately without waiting for the next threshold crossing.

---

## Energy & Cost Tracking

Energy is tracked server-side using HA's `utility_meter` integration — no dashboard needs to be open. Three meters (daily / weekly / monthly) each have four tariff slots:

- **Solar** — takes priority when `binary_sensor.hot_tub_solar_sufficient` is ON
- **Cheap** — 04:00–07:00, 13:00–16:00, 22:00–00:00
- **Peak** — 16:00–19:00
- **Standard** — all other hours

The **Tariff Switcher** automation switches the active tariff at every Cosy boundary and whenever solar output crosses the threshold. Template sensors multiply accumulated kWh by the configurable rates to produce cost figures in pence (divide by 100 for £).

---

## Override System

Override can be activated two ways:

**Physical (touch sensor):**
- Tap 1: 30 min countdown starts, tub on
- Tap 2: extend to 60 min
- Tap 3: extend to 90 min (maximum)
- Tap 4: cancel, tub off
- OLED shows HH:MM:SS countdown
- ESPHome `binary_sensor.hottub_temperature_override_active` bridges to HA

**Dashboard:**
- Toggle button activates override for `hottub_override_duration` hours (default 3h)
- HA `timer.hottub_override_timer` counts down, cancels automatically
- iOS notification sent on expiry

Both paths converge on `input_boolean.hottub_in_use_override`.

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
- Temperature — live water temp with arc gauge, target, source indicator
- Mode — current control mode badge with description
- Solar — live production vs threshold
- Cosy Tariff — current rate and tariff name, 24-hour colour-coded timeline with live position marker
- Energy & Cost — Today / This Week / This Month tabs with per-tariff kWh split and last-7-days daily breakdown
- Schedule toggles — Solar / Cheap 04–07 / Cheap 13–16 / Cheap 22–00 / Standard Rate, each independently switchable
- Manual override — activate button with countdown timer
- Settings — solar threshold, override duration sliders

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
- The solar boost on/off thresholds (2 kW / 1.5 kW) are hardcoded in `hottub_solar_automations.yaml`. The solar trigger threshold is separately configurable via `input_number.hottub_solar_threshold`.
- The DS18B20 is read at 10-bit resolution (0.25°C steps, 187ms conversion) rather than 12-bit to avoid WiFi interrupt corruption of the OneWire bus on the ESP8266.
- The TTP223 touch sensor is wired with `inverted: true` in ESPHome — it idles HIGH and pulls LOW on touch.
- The font `ConsidermevexedRegular-ExLe.ttf` is not included in this repository as it is a third-party font. Download it free from FontSpace (link above).
