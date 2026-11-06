# Adapting Energy OS to your home

Energy OS runs as Home Assistant packages. You map a short list of placeholder entities to your own hardware, set your own numbers, and start in watch mode.

The mapping below is designed to fit other hardware. It has been run on one home, so expect to adjust. Treat the first week as testing.

## What you need

1. Home Assistant on an always-on machine. A small PC, a mini computer, or a virtual machine all work.
2. A battery and solar integration that reports battery percent and live solar power.
3. A solar forecast integration that reports kWh for today and for the next hour.
4. A thermostat integration that gives you a climate entity.
5. A calendar that Home Assistant can read (optional, with a weekend fallback).
6. A phone with the Home Assistant app (for alerts).

## Install

1. Enable packages once in `configuration.yaml`:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
2. Copy `config/energy_os.yaml` and `config/energy_os_health.yaml` into `<config>/packages/`.
3. Replace every placeholder in the table below, in both files.
4. Run Check configuration (Developer tools, YAML), then restart.
5. Confirm `eos_notify_only` is on. It is the default.
6. Add `config/dashboard-card.example.yaml` as a manual card.

## Entity map

| Placeholder in the files | What it must be | Typical source |
|---|---|---|
| `sensor.battery_percent` | Battery charge, 0 to 100 | Your battery's integration |
| `sensor.solar_power_kw` | Live solar production in kW (divide by 1000 if your sensor reports watts) | Your solar or battery integration |
| `sensor.solar_generated_kwh` | Cumulative energy generated, kWh, only ever rising | Same integration |
| `sensor.solar_forecast_today` | Forecast kWh for today | Forecast.Solar, Solcast, or similar |
| `sensor.solar_forecast_next_hour` | Forecast kWh for the next hour | Same integration |
| `climate.thermostat` | Your thermostat | Ecobee, Nest, Honeywell, or similar |
| `switch.thermostat_aux_only` | A switch that forces gas-only heat | See the dual-fuel note below |
| `number.thermostat_fan_min_on` | Fan minimum minutes per hour | Thermostat integration, if it offers one |
| `sensor.thermostat_temperature` | Temperature at the thermostat | Thermostat integration |
| `sensor.bedroom_a_temperature`, `sensor.bedroom_b_temperature`, `sensor.living_room_temperature` | Remote room sensors | Thermostat integration. Delete the ones you lack |
| `calendar.household` | A calendar with laundry, cleaner and oven events | Google, iCloud or a remote calendar |
| `weather.home` | A weather entity with a temperature attribute | Any weather integration |
| `https://hc-ping.com/YOUR-CHECK-UUID` | Your dead-man's-switch address (keep it secret) | healthchecks.io or similar |

## Variants

1. **No battery.** Create a template sensor that always returns 100 for `sensor.battery_percent`, and set the battery floor to 0.
2. **No calendar.** Point `calendar.household` at an empty calendar. The weekend default applies.
3. **Single fuel (heat pump only, or gas only).** The gas versus heat pump choice does not apply. You can still use the daily gate, the forecast correction, cool control, the fan rules and the health check. The mode logic needs editing. This is on the roadmap.
4. **Thermostats without a gas-only switch.** If your thermostat has no way to force the gas side from software, the heat pump versus gas choice cannot be automated. Verify this first, with a live test, before building on it.
5. **A different latitude.** Edit the month list in `binary_sensor.eos_early_sun_season`, and set `eos_wake_time` and `eos_wake_time_early`. The example assumes a northern-hemisphere site with early summer sun. See the constants table in [`decision-logic.md`](decision-logic.md).

## First week

1. Watch mode on. The logbook shows "WOULD set X" lines.
2. Each day, compare the log with what you would have done by hand.
3. Look at the learned forecast bias. It should settle between its floor and ceiling. A value stuck at the floor means the first seeding did not run.
4. Fix the numbers that cause wrong calls. See the tunables table in [`decision-logic.md`](decision-logic.md).
5. Read [`safe-rollout.md`](safe-rollout.md), then decide when to turn watch mode off.
