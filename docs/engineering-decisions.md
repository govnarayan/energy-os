# Engineering decisions and tradeoffs

This documents the main design decisions and the constraints behind them. The control logic is not
trivial: a multi-mode state machine that balances live battery charge, live solar, a day-ahead
forecast, indoor temperature, and manual control, with safety floors. Each decision below is a real
choice with a real alternative that was rejected.

## 1. Prove the make-or-break unknown before building anything

The risk that could have ended the project: could software tell a dual-fuel system to use the gas
furnace on command? Reading sensors and setting heat, cool, or off is straightforward. Forcing the
gas side (auxiliary heat) is the specific hard part, and ecobee's own developer API has been closed
to new keys since about 2024.

What I did: rather than assume, I ran one live API call through SmartThings (the remaining open
path) that set the thermostat to aux-heat-only, and confirmed the furnace physically fired. Only
then did I write any decision logic.

Why it matters: the most expensive failure is building on an assumption that turns out false. A few
minutes of verification removed the largest risk in the project.

## 2. A blocker that became a simplification

The plan: read the Powerwall's battery and solar locally, free and without a cloud dependency.

The wall: on current Powerwall 3 firmware, local access is gated behind a gateway password printed
inside the sealed unit. The installer would not share it (warranty), and the tooling the helpdesk
pointed to is for installers, not customers.

The reframe: that answer was about installer configuration tooling, a different question from
reading my own data. The customer path is Tesla's official cloud API, reached through a small bridge
service (Teslemetry) authorized with my own Tesla login. No password, no opening the unit, no
warranty issue.

The payoff: going cloud also removed a hardware constraint. Local reads would have forced the
controller's host onto the Powerwall's private Wi-Fi. The cloud read let it run anywhere, which
reduced the hardware question to "any always-on machine."

## 3. Designing for safe rollout, not only correctness

This system can turn heating on and off. Getting the logic 80 percent right and learning the rest by
watching it misbehave was not acceptable, so the design has three deliberate safety layers:

- A notify-only mode that runs the full decision engine against live data and logs the action
  instead of taking it, a multi-day dry run before it touches the thermostat.
- A hard floor that never lets the heat pump run below 5 percent battery, on every code path,
  regardless of the higher-level logic.
- A manual override that detects a by-hand mode change and suspends the automation until the next
  morning, so it does not fight the operator.

Why it matters: the question on a control system is not whether it works when everything is normal,
but what it does when it is wrong and how control is recovered.

## 4. Recognizing a capability ceiling and routing around it

I wanted the system to vary the fan's minimum runtime (0, 20, or 55 minutes per hour) by state, to
stop the fan circulating cooking smells when the system is off.

The finding: that setting (fanMinOnTime) is native to the ecobee but not exposed by the SmartThings
API in use. The older integrations that exposed it were discontinued in 2023, and a live scan of the
device's capabilities confirmed only the coarse fan mode is available.

The decision: rather than ship an approximation, I documented the exact limitation, deferred the
feature to the roadmap, and drafted a request for the native API, which is the only thing that can
set it. The four enhancements that were possible shipped; the one that was not is flagged, not faked.

Why it matters: distinguishing "hard" from "not currently possible on this surface," and stating
which it is, avoids shipping something that looks finished but is not.

## 5. Treating every input as a claim, not a fact

When the solar forecast read about 20 percent high, the easy move would have been to distrust the
tool. Instead I separated the question: forecasting error or configuration error? It was
configuration. The forecast service's azimuth convention differs from the installer's plan-set, and
it models AC capacity where the array is panel-limited. (The azimuth convention was easy to get
backwards from memory; the fix was checking it against the service's own published definition rather
than assuming.)

Then the second question: does the 20 percent even matter? The forecast feeds a binary gate ("is
today above 18 kWh?"). On a 40 kWh summer day a 20 percent error never flips the decision; it only
matters on a marginal 18 to 23 kWh winter day. So I logged it to tune later rather than let a
precision problem block a system that did not need precision there.

Why it matters: accuracy is worth chasing where it changes a decision. Knowing where on the input
the tolerance is tight is more useful than blanket precision.

## 6. Picking the right environment at kickoff

The instinct for "an app that controls my house" is to build an app. I did not. Every input I needed
(Powerwall, forecast, thermostat) already had a Home Assistant integration, and the logic is a
single declarative package. A custom app would have meant re-implementing all of those integrations
and standing up a 24/7 backend, more work for a worse result. The right tool was the one that
already talked to all the hardware.

Why it matters: the highest-leverage decision on a build is often what not to build.
