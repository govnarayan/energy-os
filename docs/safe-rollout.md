# Safe rollout

Energy OS can turn heating and cooling on and off. The question for any control system is not whether it works on a normal day. It is what the system does when it is wrong, and how a person takes control back.

## Five safety layers

1. **Watch mode.** With `eos_notify_only` on, the full decision engine runs against live data and logs "WOULD set X" instead of acting. Run it for several days before it touches the thermostat.
2. **Battery floor.** The heat pump never runs below the battery floor. This applies on every path, including after a manual override, because forced heat can drain a battery to near zero.
3. **Manual override.** If a person changes the thermostat mode by hand, the automation suspends until the next daily reset. It does not fight the operator.
4. **Minimum dwell time.** Automated switches wait a minimum number of minutes after the last one, to protect the equipment.
5. **Master switch.** `eos_automation_enabled` stops the control automations and hands control back.

## Turning watch mode off

1. Run watch mode for at least a week that includes a bright day, a dark day and a weekend.
2. Confirm the log matches what you would have done.
3. Confirm the health check has passed every day.
4. Turn watch mode off on a day when you can watch the thermostat.
5. Watch the first full day. Keep the master switch within reach.

## Watch: the daily health check

1. The check runs at 10:00 and tests that the automations are loaded and on, the thermostat answers, the forecast is fresh, and the learned bias is not stuck at its floor.
2. A pass pings a dead-man's-switch service. The service alerts you if a day passes with no ping, which also catches a machine that is off or asleep.
3. A failure raises a notification on the phone, and in Home Assistant.

Treat the ping address like a password. Anyone who has it can send a ping and hide an outage.

## Keeping the host running

1. Set the machine to never sleep.
2. Set it to start Home Assistant after a power loss or restart.
3. If Home Assistant runs in a virtual machine, start the virtual machine at login, and shut it down cleanly when you can.
4. Keep an automatic backup of Home Assistant, and a copy outside the virtual machine. Backups hold secrets, so keep them out of shared cloud folders.
5. Phone alerts only work if the app's notify action is set. Check it after any restart.

## When something is wrong

1. A rule did not fire: open the automation and read its trace. A stop at "notify only" is expected in watch mode.
2. The thermostat shows unavailable: reload its integration from Settings, Devices and services. Wait a few minutes after any restart first.
3. The forecast is empty or old: check the forecast integration's call limit. Many have a daily cap.
4. Anything else: Settings, System, Logs, search for the package name.
5. Last resort: restore the last known good backup.
