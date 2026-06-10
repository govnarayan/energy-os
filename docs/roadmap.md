# Home HVAC - running list of tweaks for the final build

Open items to revisit after the notify-only test proves the core logic. Not blockers for the test.

## 1. Solcast forecast accuracy (flagged 2026-06-09)
**Symptom:** `forecast_today` read 43.67 kWh on a partly-cloudy day; user's real expectation ~35.
A 6.16 kW DC array physically can't exceed ~37 kWh/day (~6-6.5 kWh/kW/day ceiling), so 43.67 means
Solcast is modeling a larger system than exists.

**Confirmed NOT the cause (user entered these correctly):**
- Azimuth = 109 is correct under Solcast's convention (North 0, West positive, East negative;
  compass 251 deg WSW -> -251+360 = +109).
- Capacity entered as DC 6.16 / AC 11.5 (both correct values).

**Suspected cause (unverified - DO NOT assert):** Solcast scaling off the 11.5 AC figure instead of
the 6.16 DC limit, and/or a too-low loss factor (default ~10-14%; near-0 would inflate). Plus genuine
same-day weather optimism (cloud cover not reflected on the flagged morning).

**WHY IT MATTERS (user's point):** irrelevant on clear summer days (forecast 40s, threshold 18 -
decision never flips). It matters on **winter/rainy/marginal days where the true forecast is ~18-23** -
a 20% over-read could wrongly label a sub-threshold day as qualifying (run electric when it shouldn't).

**Fix options:** (a) check Solcast site's loss factor + which capacity it scales from; reduce the
modeled capacity toward 6.16 until clear-day forecast lands ~35. (b) Use Solcast "tuned" forecasts
(feed actual production to self-calibrate). (c) Recalibrate the qualify threshold to the observed bias.
**Verification:** during the notify-only watch, log forecast-vs-actual daily production for a week;
if Solcast is systematically ~X% high, it's config; if it scatters around actual, it's normal scatter.

## 2. SmartThings OAuth token refresh
Replace the 24h personal-access-token (manual re-mint each day) with a durable SmartThings OAuth
app + auto-refresh so the system runs 24x7 without the daily token chore. Do before going live.

## 3. Phone dashboard
Build the companion-app dashboard: current mode, battery %, live solar, today's forecast,
qualifying-day badge, + Force Heat / Force Gas / Auto buttons and the master kill switch.

## 4. Permanent-host decision + uptime hardening
Currently on the Windows PC (VM). Decide permanent home (PC vs Mac). Apply auto-start (day0-setup.md
Appendix A) and, if Mac, the no-sleep / no-auto-update / FileVault-off+auto-login hardening.
Migrate via Settings -> System -> Backups if moving hosts.

## 5. Fan minimum-runtime control (deferred 2026-06-09 - needs native ecobee API)
Want: fan min-runtime 0 min/hr when system off (stop cooking-fume circulation), 20 min/hr when heating/cooling
with no solar (incl. overnight aux), 55 min/hr when solar > 0. Fan MODE stays auto throughout.
**Blocker:** this is the ecobee `fanMinOnTime` setting (valid 0-55). The current SmartThings API exposes only
fan MODE (auto/on/circulate), NOT fanMinOnTime - the old Groovy handlers that could were killed in 2023, and
this device's capability list has no fan-minutes control. Only the **native ecobee API** can set it.
**Path:** included in the Generac/ecobee API request (see ecobee-api-request.md). Until granted, fan is
unmanaged. Build #1-4 (override, cool, etc.) proceed on SmartThings now.

## 6. Cool trigger boundary: >74 vs >=74 (logged 2026-06-09)
Spec is "cool if indoor temp >= 74"; the continuous "cool when warm" automation uses a numeric_state
`above: hvac_cool_temp` trigger, which fires only at strictly **>74** (74.0 exactly won't cool until 75).
The manual-on decision path already uses >=74 correctly, so it's just the continuous trigger that's off-by-one.
Fix when convenient: set the trigger to `above: 73` (fires at 74+) OR use a template trigger on
`>= cool_temp`. Minor; only matters at an exact 74.0 hold. (Also note: numeric_state triggers fire on a
threshold CROSSING, not on already-sitting-at-the-value when the automation loads.)
