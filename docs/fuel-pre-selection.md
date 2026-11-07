# Choosing the fuel in advance: the Heat button problem

On a dual-fuel home (heat pump plus gas furnace), a person can press Heat at the thermostat at any moment. Energy OS wants a Heat press to run the right fuel from the first second: the furnace when solar cannot carry the heat pump, the heat pump when it can. Reacting after the press is too slow on most thermostat clouds. This page explains why, and the mechanism that avoids it.

## Why reacting is too slow

1. Home Assistant learns about the thermostat from the vendor's cloud. The ecobee integration polls it about every 3 minutes. A press therefore reaches the controller between 0 and 3 minutes later. In one test the delay from press to command was about 1 minute 40 seconds.
2. A person pressing Off and then Heat inside one poll looks like a change from gas to heat. Energy OS treats plain Heat as "heat, controller picks the fuel", never as an override, for this reason.
3. A command sent after the press is accepted by the cloud at once and the thermostat follows in seconds. The fault is the detection delay. If the thermostat is calling for heat during that window, the heat pump runs.

Asking the cloud more often is not a fix. It multiplies requests to a service that is not built for it, and the integration's poll interval is not a setting.

## The mechanism: a heat pump lock

Many dual-fuel thermostats have a compressor minimum outdoor temperature (ecobee calls it "compressor minimum temperature"). Below it the heat pump will not run for heating and the furnace does. Energy OS uses it as the fuel switch for the Heat button:

| Lock | Setting (example) | A Heat press runs |
|---|---|---|
| Closed | 65 F | the gas furnace |
| Open | 35 F | the heat pump |

The `Energy OS: heat pump lock follows decision` automation keeps the lock matched to what the controller would pick right now. It closes at once when the decision is gas and opens when the decision has been heat pump for 30 seconds. It re-checks every 10 minutes and at startup, and an unknown decision counts as closed. The controller never starts the heat pump while the lock is still closed, because heat mode with a closed lock runs the furnace first and hands over to the heat pump a couple of minutes after the lock opens. The sawtooth re-enable retries when the lock changes.

It is off by default (`eos_heat_pump_lock_enabled`). Turn it on after you have mapped `number.thermostat_compressor_min_temp` and set the two values for your thermostat.

## What was measured, on one home (ecobee, outdoor 61 F)

1. With the lock closed, a Heat call ran the gas furnace 11 seconds later and the heat pump stayed off.
2. After the lock was opened with the furnace already running, the heat pump took over about 2 minutes later.
3. Not tested: a fresh call with the lock already open, and the lock above the thermostat's upper limit.

## Limits

1. The setting has an upper limit (66 F on ecobee). Above that outdoor temperature the lock cannot hold, and a Heat press is corrected by the gas command within a few minutes.
2. Check how much of your heating happens above that limit before you rely on it. ecobee owners can export runtime history by equipment and outdoor temperature (for example from Beestat). On the home measured here, 1.4 percent of 879 heating hours and about 1 in 100 heating starts happened at 66 F or above over 12 months.
3. A new value takes a few minutes to show in Home Assistant even though the thermostat has it sooner.
4. This depends on a thermostat setting that not every model has. Without it, the fallback is the controller's own correction after detection, or pressing the gas mode on the thermostat.
5. Faster detection is possible with a local protocol such as HomeKit, which pushes changes in seconds. It does not expose the gas-only mode, so it does not replace the lock.
