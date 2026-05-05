# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A Home Assistant smart hot tub controller. No build system — files are deployed directly to a running HA instance. The stack is:
- **Home Assistant YAML packages** — automations, sensors, helpers, notifications
- **ESPHome firmware** (`hottub-temperature.yaml`) — D1 Mini wall unit (OLED + DS18B20 temp + TTP223 touch)
- **Standalone HTML dashboard** (`dashboard.html`) — connects to HA via WebSocket API, no Lovelace

## Deployment

These files deploy to `/config/` on the HA instance:

| Repo file | HA path |
|---|---|
| `packages/hottub/hottub_helpers.yaml` | `/config/packages/hottub/hottub_helpers.yaml` |
| `packages/hottub/hottub_sensors.yaml` | `/config/packages/hottub/hottub_sensors.yaml` |
| `packages/hottub/hottub_automations.yaml` | `/config/packages/hottub/hottub_automations.yaml` |
| `packages/hottub/hottub_notifications.yaml` | `/config/packages/hottub/hottub_notifications.yaml` |
| `esphome/hottub-temperature.yaml` | `/config/esphome/hottub-temperature.yaml` |
| `www/hottub/dashboard.html` | `/config/www/hottub/dashboard.html` |

`configuration.yaml.example` shows the `packages:` block that must exist in HA's `configuration.yaml`.

The font `ConsidermevexedRegular-ExLe.ttf` is not in the repo — it must be downloaded separately and placed at `/config/esphome/fonts/`.

## Validating Changes

In HA: **Developer Tools → YAML → Check Configuration** before restarting.

For package-only changes (no new entities), use **Developer Tools → YAML → Reload Automations / Reload Template Entities / Reload Input Helpers** instead of a full restart.

ESPHome changes require flashing the D1 Mini via the ESPHome dashboard.

## Architecture

### Smart Controller Priority Cascade

`hottub_smart_controller` (automation #7) runs every 5 min and on key state changes. Conditions evaluate top-down — first match wins:

```
1. PEAK BLOCK    (16:00–19:00, override not active)  → OFF
2. OVERRIDE      (input_boolean.hottub_in_use_override) → ON
3. SOLAR         (hot_tub_solar_sufficient AND schedule_solar) → ON
4. CHEAP RATE    (hot_tub_cheap_rate_active AND schedule_cheap) → ON
   DEFAULT                                            → OFF
```

The master gate `input_boolean.hottub_schedule_enabled` must be ON for any scheduled heating — override (automations 4, 9, 10) bypasses this gate.

### Energy / Cost Tracking

Cost tracking does **not** use HA utility meters. Instead, `calculateCosts()` in `dashboard.html` queries `sensor.appliance_hottub_energy` history over a rolling 31-day window via the HA WebSocket API, then sorts each kWh into cheap/standard/peak/solar buckets based on the timestamp's hour. Results are written back to `input_number.hottub_*_kwh` entities so HA template sensors can expose them. This design survives HA restarts and avoids the month-boundary reset bug.

`sensor.power_meter` reports solar production in **kW** — the solar sufficient binary sensor multiplies by 1000 to compare against the watt threshold.

### Override System (Two Entry Points)

**Physical (D1 Mini touch sensor):**
- Taps cycle through 30/60/90 min countdown → cancel
- ESPHome `switch.hottub_temperature_manual_override` bridges to HA via automations 9 & 10
- OLED shows MM:SS countdown

**Dashboard:**
- Activates `input_boolean.hottub_in_use_override` for `hottub_override_duration` hours
- `timer.hottub_override_timer` auto-cancels and sends iOS push notification on expiry

Both paths converge on `input_boolean.hottub_in_use_override`.

### Key External Entity Dependencies

These entities must already exist in HA (provided by Zigbee plug + solar inverter):

| Entity | Source |
|---|---|
| `switch.appliance_hottub` | Zigbee plug |
| `sensor.appliance_hottub_power` | Zigbee plug |
| `sensor.appliance_hottub_voltage` | Zigbee plug |
| `sensor.appliance_hottub_energy` | Zigbee plug (cumulative kWh) |
| `sensor.power_meter` | Solar inverter (reports in **kW**) |

Notification service `notify.mobile_app_jasons_iphone` in `hottub_notifications.yaml` is device-specific — update to match the target device.

## Tariff Windows (Octopus Cosy)

| Window | Times |
|---|---|
| Cheap | 04:00–07:00, 13:00–16:00, 22:00–00:00 |
| Peak | 16:00–19:00 |
| Standard | all other times |

Rates are stored in `input_number.hottub_rate_*` and configurable from the dashboard Settings panel.
