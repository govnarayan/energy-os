# Decision logic

This describes what `config/energy_os.yaml` does, using the names of its tunables. Set the numbers from your own meter. The file's values are examples.

## Modes

The controller moves between four thermostat modes:

| Mode | Meaning |
|---|---|
| `auxheatonly` | Gas furnace heat. The default, and the safe fallback |
| `heat` | Electric heat pump |
| `cool` | Air conditioning |
| `off` | Thermostat off |

## The daily gate

Each morning (the daily reset) the controller decides whether the day can run electric heat or cooling on solar.

```
needed   = base load + heat-pump run
           + laundry extra (if a laundry day)
           + cooking extra (if a heavy-cooking day)
           + manual margin
expected = solar forecast for today x learned bias
qualifying day = expected >= needed
```

1. **Laundry and cooking days** come from the household calendar. Event titles with "laundry", "cleaner" or "oven" switch them on. If the calendar does not answer, a weekend default applies.
2. **The learned bias** is actual production divided by forecast. Every night the controller compares the generation meter with the forecast and updates the bias with a moving average. Dark days are skipped because the ratio is noise there.
3. **After a restart** the gate is re-evaluated, so a stale switch cannot carry over.

## The target mode

One template sensor, `eos_target_mode`, holds the answer to "what should the thermostat be doing now":

1. **Cool** if it is already cool and indoor is at or above `eos_cool_hold_temp`, or if solar is viable, indoor is at or above the cool setpoint, and outdoor is warm enough.
2. **Heat pump** if heat is eligible and indoor is below the cool setpoint.
3. **Gas** otherwise.

Two helper conditions feed it:

1. **Solar viable:** a qualifying day, battery above the re-enable level, and live solar at or above the evening cutoff.
2. **Heat eligible:** solar viable, or the morning battery-dump window: a qualifying day, battery above the re-enable level, after the earliest start time, before the afternoon arm time, and the corrected next-hour forecast at or above the dump threshold.

Once cool is engaged it stays cool until a person changes it.

## The automations

| Automation | What it does |
|---|---|
| Daily reset | Sets load-day flags and the qualifying-day switch. Clears the day's flags |
| Seed learned values | Gives the bias a neutral starting value the first time |
| Re-evaluate gate on start | Recomputes the qualifying day a few minutes after a restart |
| Snapshot generation at midnight | Records the generation meter for the bias calculation |
| Learn forecast bias | Updates the bias at the end of the day |
| Morning dump | Moves from gas to heat pump once a day when the battery can cover the gap before solar ramps up |
| Battery floor | Moves heat to gas when the battery falls to the floor |
| Sawtooth re-enable | Moves back to heat pump when battery and solar recover, after a minimum dwell time |
| Cool when warm | Moves to cool when indoor is warm and solar is viable |
| Flag manual off | Notes that a person turned the thermostat off |
| Detect manual override | Suspends the automation until the next reset when a person changes the mode |
| Manual-on decision | Picks the right mode when a person turns the thermostat back on |
| Evening transition | Moves heat to gas after the afternoon arm time when solar drops below the cutoff |
| Fan minutes | Sets fan minutes per hour: low when off, normal when active, higher when hot |

## The sequence on a typical solar day

1. Early morning: gas heat. The gate has said yes.
2. When the battery can cover the gap before solar ramps up: the morning dump moves to heat pump.
3. Midday: gas or heat pump, depending on battery level and solar. Dwell time stops rapid switching.
4. Warm afternoon: cool when warm takes over.
5. Evening: solar falls below the cutoff and the evening transition returns to gas.

On a non-qualifying day the controller makes no electric switches. It does not override a manual choice, with one exception: the battery floor always applies.

## Tunables

| Name | What it controls | How to choose it |
|---|---|---|
| `eos_base_load_kwh` | Daily use with no heat pump or big loads | The median day on your meter without those loads |
| `eos_hp_load_kwh` | Heat-pump energy on a heating day | Compare a heating day with a mild day |
| `eos_laundry_load_kwh` | Extra use on laundry days | Compare laundry days with others |
| `eos_oven_load_kwh` | Extra use on heavy-cooking days | Same method |
| `eos_qualify_forecast` | Extra safety margin on the gate | Zero until the log shows wrong calls |
| `eos_dump_next_hour_kwh` | How much solar must be coming before the morning dump | Raise it if the battery empties before solar arrives |
| `eos_batt_floor` | Battery level the heat pump never goes below | Your outage reserve |
| `eos_batt_reenable` | Battery level that allows heat pump again | Above the floor, with a gap, so it does not flip back and forth |
| `eos_solar_evening_off` | Solar level below which the day is over | The production that no longer covers a heat pump |
| `eos_min_dwell_min` | Minimum minutes between automated switches | Long enough to protect the equipment |
| `eos_cool_temp` | Indoor temperature that triggers cooling | Your comfort limit |
| `eos_cool_hold_temp` | Indoor temperature below which cooling stops | A few degrees under the trigger, so it does not cycle |
| `eos_outdoor_cool_min` | Outdoor temperature below which auto-cool stays off | Your mild-weather threshold |
| `eos_fan_off_min`, `eos_fan_on_min`, `eos_fan_hot_min` | Fan minutes per hour in three states | Comfort and air-quality preference |
| `eos_wake_time`, `eos_wake_time_early`, `eos_afternoon_arm` | Earliest morning action (normal and early-sun season) and when the evening logic arms | Your household's day and your sunrise |

## Constants in the logic

These are fixed in the file as examples. Review them for your site.

| Where | Value | Change it when |
|---|---|---|
| `eos_early_sun_season` | Months 6 to 10 | You are in another latitude or hemisphere |
| Daily reset times | 05:00 in season, 06:00 otherwise | Sunrise or your gate timing differs |
| `eos_indoor_temp` | Night is 22:00 to wake time, bedrooms only | Your sensors or sleep hours differ |
| Weekend default | Saturday and Sunday count as laundry and heavy-cooking days | Your calendar is reliable, or your big-load days differ |
| Learn forecast bias | Average weight 0.25, range 0.5 to 1.05, dark day under 8 kWh | Your forecast service runs low, or your roof is shaded |
