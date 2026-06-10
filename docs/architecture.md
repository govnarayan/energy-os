# Step 2 - Home Assistant build layout

**Dependency:** assumes the AUX command passed step 1 (`verify-aux-control.md`). If it didn't, the SmartThings path gives you Heat/Off/Cool only and you need the native API (`ecobee-api-request.md`) for full gas control.

**Shape of it:** Home Assistant is the always-on engine (reads Powerwall + Solcast, runs the §5 state machine, sends mode commands). The HA **companion mobile app** is your phone UI - live status + manual-override buttons - so you get the "phone app" feel with zero front-end code. A phone can't be the engine (it sleeps); the engine has to be the always-on box regardless of UI.

---

## Hardware

Because the Powerwall is read via **Teslemetry (cloud)**, the HA box no longer needs to touch the Powerwall's Wi-Fi - so almost any always-on machine works, including one you already own. (The dual-network requirement that previously argued for a Pi 5 is gone.)

| Option | Notes |
|---|---|
| **A machine you already own** (mini-PC, NAS that runs Docker, always-on desktop, spare laptop) - cheapest | Run HA OS in a VM or HA Container. ~$0. Fine for the 2–3 day test and for production if it stays powered on. |
| **HA Green appliance** | Turnkey HA, single Ethernet - perfectly fine now that no Powerwall-Wi-Fi bridging is needed. Verify current price before buying. |
| **Raspberry Pi 5** | Works, but currently ~$150–250 for a kit (2026 memory shortage) and no longer advantaged here - the dual-network reason to prefer it no longer applies. |

---

## Integrations

1. **Powerwall - battery % + live solar (via Teslemetry cloud).**
   - Local TEDAPI is **off the table for you**: PW3 firmware needs the gateway Wi-Fi password, which is inside the sealed unit and unavailable.
   - Use **Teslemetry** (cloud, ~$25/yr): sign into your own Tesla account, authorize **only the Powerwall**. Native HA integration, ~1-minute polling - ample for this loop, and you've already confirmed both values are visible. (Note: "command credits" don't apply - we only read the Powerwall; all control goes to the ecobee.)
   - You need two sensors: battery state-of-charge (%) and solar power (watts → `/1000` to kW). The exact entity IDs appear once the integration is added - confirm them in Developer Tools → States and paste them into `solar_hvac.yaml` (replace the `SITE_` placeholders).

2. **Solcast - day-ahead forecast.** HACS integration `BJReplay/ha-solcast-solar`, free tier. Gives tomorrow's predicted kWh → sets `qualifying_day`.

3. **SmartThings - the ecobee control bridge.** Core HA integration (OAuth). Gives a climate entity for **Heat / Off / Cool / setpoint**. For **AUX/emergency heat**, if the climate entity doesn't expose it (known HA limitation), send the command straight to the SmartThings API with a `rest_command` (`setThermostatMode` → `auxheatonly`), bypassing HA's climate abstraction - this is the documented workaround.

---

## Helpers (so every constant is editable from the app, no code edits)

- `input_number`: `qualify_forecast` (18), `batt_floor` (5), `batt_reenable` (30), `solar_evening_off` (2), `min_dwell_min` (15)
- `input_datetime`: `wake_time` (06:30), `afternoon_arm` (13:00)
- `input_boolean`: `qualifying_day`, `heat_ran_today` (morning-dump-once flag), `cooking_day` (no-cooking sawtooth vs cooking manual-on path), `automation_enabled` (master kill-switch)
- `template sensor`: `battery_soc`, `solar_kw`, `solcast_tomorrow_kwh`

---

## The logic - one Node-RED flow (recommended over a YAML automation; easier for stateful sawtooth)

**Daily reset (06:00):** pull Solcast → `qualifying_day = solcast_tomorrow_kwh >= qualify_forecast`; set `heat_ran_today = false`.

**Triggers:** `battery_soc` crossing 5 / 30; `solar_kw` crossing 2; clock 06:30 and 13:00; ecobee mode change (to detect a manual On→Aux).

**Guards on every automated switch:** `automation_enabled` on; `qualifying_day` true; system not Off/Cool; `min_dwell` elapsed since last auto-switch; Aux not mid-cycle.

**Enter HEAT (→ heat-pump) when any of:**
- **A. Morning dump** - time ≥ `wake_time` AND `heat_ran_today` false AND `battery_soc` > `batt_floor` → Heat; set `heat_ran_today = true`.
- **B. No-cooking re-enable** - `cooking_day` false AND `battery_soc` ≥ `batt_reenable` AND `solar_kw` ≥ 2 → Heat.
- **C. Cooking re-enable** - ecobee manually turned On→Aux AND time ≥ 14:00 → Heat.

**Exit HEAT (→ AUX / gas) when any of:**
- `battery_soc` ≤ `batt_floor` (5%) - ends the morning dump and each sawtooth tooth.
- time ≥ `afternoon_arm` AND `solar_kw` < `solar_evening_off` (2 kW) - ends the day; stays Aux overnight.

**Non-qualifying day:** force/hold Aux whenever heating; never Heat.

**Command mapping:** Heat → `climate.set_hvac_mode: heat`; Aux → the SmartThings `rest_command` (`auxheatonly`). Never issue Heat when `battery_soc` ≤ 5% (absolute floor, all paths).

---

## Phone UI (companion app dashboard)

- **Status:** current mode, battery %, live solar kW, today's Solcast kWh, a `qualifying_day` badge.
- **Manual override:** Force Heat / Force Aux / Auto toggle, and the master `automation_enabled` switch.
- **History graph:** battery % + mode over the day, to eyeball the sawtooth and tune the thresholds.

---

## Build order

1. HA on your chosen always-on box.
2. Add the **Teslemetry** integration; authorize only the Powerwall; confirm `battery_soc` + `solar_kw` update, and copy the real entity IDs into `solar_hvac.yaml`.
3. Solcast → `qualifying_day` toggling correctly each morning.
4. SmartThings → confirm Heat/Off/Cool **and** the AUX `rest_command` both actually move the furnace/heat pump.
5. Build the Node-RED flow; run it in "notify-only" mode (logs the decision, doesn't switch) for a few days; then enable switching.
6. Dashboard + override buttons.

Step 5's notify-only shakedown is the cheap insurance - watch it call the right moves against your real battery/solar curves before you let it touch the thermostat.
