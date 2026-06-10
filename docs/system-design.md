# Home HVAC - Solar-Optimized Dual-Fuel Control Spec

**Goal:** Maximize self-consumption of on-site solar and minimize grid pull by choosing, hour by hour, between the electric heat pump ("Heat"), the gas furnace ("Aux"), cooling, and off - on a dual-fuel ecobee Smart Thermostat Premium paired with a Tesla Powerwall 3 + solar.

**Status:** Logic locked 2026-05-29. Control-path (how the command reaches the ecobee) still to be verified - see §7.

---

## 1. Reference constants (all tunable numbers in one place)

| Name | Value | Meaning |
|---|---|---|
| `QUALIFY_FORECAST` | **18 kWh** | Solcast day-ahead forecast at/above this ⇒ "qualifying day" (electric allowed) |
| `WAKE_TIME` | **06:30** | Earliest the morning dump may start |
| `BATT_FLOOR` | **5%** | Heat → Aux. Never run the heat pump below this. Universal grid-pull guard |
| `BATT_REENABLE` | **30%** | On a no-cooking day, battery climbing back to this re-enables Heat |
| `SOLAR_EVENING_OFF` | **2 kW** | In the afternoon window, live solar below this ⇒ Heat → Aux for the night |
| `AFTERNOON_ARM` | **13:00** | The `SOLAR_EVENING_OFF` exit is armed only after this (prevents it firing on the morning ramp) |
| `MIN_DWELL` | **15 min** | Minimum time in a mode before another automated switch (anti-chatter) |
| `BASELINE_LOAD` | ~0.7 kW | House baseline draw (reference) |
| `HEAT_PUMP_DRAW` | ~4–5 kW | Heat-pump draw (reference; one ~4 kWh battery dump ≈ 30–60 min) |

---

## 2. Inputs (data sources)

| Signal | Source | Notes |
|---|---|---|
| Battery state-of-charge (%) | Powerwall 3 local gateway (TEDAPI) `/api/system_status/soe` | Read-only, local, free |
| Live solar production (kW) | Powerwall 3 local gateway `/api/meters/aggregates` | Read-only, local, free |
| Day-ahead solar forecast (kWh) | Solcast free tier | Pulled once each morning to set qualifying/non-qualifying |
| Time of day | Local clock | Drives `WAKE_TIME`, `AFTERNOON_ARM` |
| Current thermostat mode | ecobee (via control bridge) | Heat / Aux(emergency) / Cool / Off |

The "month of year" and the static monthly-production spreadsheet are **not used** - the single Solcast forecast test (`>= 18 kWh`) subsumes both season and "is it sunny."

---

## 3. Control states

| State | Meaning | Owner |
|---|---|---|
| **OFF** | System off (midday cooking window) | **Manual only** - automation never turns the system off |
| **COOL** | Air conditioning | **Manual.** When in Cool, automation hands off entirely |
| **AUX** | Gas furnace (ecobee emergency/aux-heat mode) | Automation + manual |
| **HEAT** | Electric heat pump (ecobee normal heat mode) | Automation + manual |

Automation only ever moves the system between **AUX ⇄ HEAT**. It never sets OFF or COOL, and it leaves the system alone whenever the user has put it in COOL.

---

## 4. Qualifying day

```
qualifying_day = (Solcast day-ahead forecast >= 18 kWh)
```
Evaluated once each morning. If **not** qualifying ⇒ gas-only day: automation makes **no** Heat switches; any heating is AUX. (Equivalent to the old winter/rain rule.)

---

## 5. State machine (qualifying day only; system ON and not in COOL)

All automated transitions also require: `qualifying_day == true`, system is ON, mode is not COOL, and `MIN_DWELL` since the last automated switch has elapsed, and Aux is not mid-cycle (don't yank gas off mid-call).

### Enter HEAT (Aux → Heat) - any one of:

- **A. Morning dump** - `time >= WAKE_TIME` AND no Heat window has run yet today AND `battery > BATT_FLOOR`. Starts at *any* charge above the floor (the 30% wake threshold from the first draft is **dropped**).
- **B. No-cooking re-enable (sawtooth)** - system was never turned Off today (Aux running continuously) AND `battery >= BATT_REENABLE` (30%) AND `live_solar >= 2 kW`.
- **C. Afternoon manual re-enable** - user manually turns the system back ON to AUX after ~14:00 → automation upgrades to HEAT (battery is near-full by then).

### Exit HEAT (Heat → Aux) - any one of:

- **Battery floor** - `battery <= BATT_FLOOR` (5%). Ends the morning dump and each sawtooth tooth.
- **Evening** - `time >= AFTERNOON_ARM` AND `live_solar < SOLAR_EVENING_OFF` (2 kW). Ends the last Heat window of the day; system stays AUX overnight.

### Typical qualifying day, narrated

1. **06:30+** battery (from overnight) dumps via HEAT until ~5% → AUX.
2. **Late morning** AUX runs while solar refills the battery.
3a. **No-cooking day:** each time battery passes 30% → HEAT again, down to 5%, repeat (slow ~5–30% sawtooth; safe because afternoon recharge ≈ 1.5 hr, peak ~2pm, cloud-roll-in rare).
3b. **Cooking day:** manual OFF ~noon; manual ON ~2pm to AUX → automation upgrades to HEAT.
4. **Afternoon HEAT** continues until live solar < 2 kW → AUX for the evening/night.

### Non-qualifying day
Automation forces/holds **AUX** whenever heating; never HEAT. Manual OFF/COOL still respected.

---

## 6. Failsafes

- **5% floor is absolute** - never run the heat pump below 5% battery, on any path, including manual-afternoon re-enable.
- **Min dwell 15 min** - no automated mode change within 15 min of the last one.
- **No mid-cycle interruption** - don't switch out of an active heat/gas call; wait for it to idle.
- **Manual override wins** - a manual mode change on the thermostat or app holds until the next day's first automated transition (the morning dump). (Confirm this reset point.)
- **Comfort floor (open):** decide a minimum indoor temperature the system must hold regardless of energy state - see §8.

---

## 7. Control-path dependency (the build risk)

Reading all inputs is solved and free (Powerwall 3 local + Solcast). The open risk is **writing** the mode to the ecobee:

- ecobee's own developer API is **closed to new keys** (since ~2024).
- **HEAT / OFF / COOL / setpoint** are reliably settable through SmartThings or any bridge.
- **AUX (emergency/aux-heat)** - the one command that makes "force gas" work - is hardware-supported on the Premium but historically flaky through the SmartThings → Home Assistant layer (Seam doesn't expose it at all). **This must be tested live on the actual unit before committing to an architecture.**

If the AUX command can't be set reliably, fallbacks: (a) request native/partner ecobee API access from Generac, or (b) restructure around setpoint strategy only. See §8.

---

## 8. Open items before build

1. **Verify the AUX-mode command** on the Premium via SmartThings (§7) - make-or-break.
2. **Manual-override reset point** - confirm a manual change holds only until the next morning dump.
3. **Comfort floor** - minimum indoor temp the system must defend regardless of energy state (e.g., during the noon OFF window on a cold day).
4. **Run environment** - Home Assistant on an always-on box (Pi 5 / mini-PC / HA Green) + companion mobile app is the recommendation; a fully custom phone app needs the same 24/7 backend rebuilt from scratch.

---

*Inputs: Powerwall 3 TEDAPI (local), Solcast (forecast), ecobee Premium (control, bridge TBD). Logic authored from <your name>'s stated operating routine, 2026-05-29.*
