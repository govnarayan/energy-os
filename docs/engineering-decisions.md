# Engineering decisions

Ten choices that shaped Energy OS, each with the risk it removes. The controller balances live battery charge, live solar, a day-ahead forecast, indoor temperature and manual control, inside hard safety limits.

## 1. Prove the make-or-break unknown first

The risk that could end the project: can software tell a dual-fuel system to use the gas furnace on command? Reading sensors and setting heat, cool or off is routine. Forcing the gas side is the hard part, and some thermostat vendors close their developer programs to new users.

The approach: run one live test that sets the gas-only mode and confirm the furnace physically fires. Write decision logic only after that.

Why it matters: the most expensive failure is building on an assumption that turns out false.

## 2. Treat a blocked path as a question, not a wall

A blocker often answers a different question from the one you asked. A vendor refusing local access to a battery answered "can an installer tool do this", not "can I read my own data". The customer path was the vendor's official cloud interface, reached through a bridge service with the owner's own login.

Why it matters: the cloud route also removed a hardware constraint. The controller can run on any always-on machine.

## 3. Design for safe rollout, not only correctness

A system that switches heating cannot be 80 percent right and learn the rest by misbehaving. The design has five layers: watch mode, a battery floor, manual override, minimum dwell time and a master switch. See [`safe-rollout.md`](safe-rollout.md).

Why it matters: the test of a control system is what it does when it is wrong and how control is recovered.

## 4. Name a capability ceiling and route around it

Some settings are native to the thermostat but not exposed by every interface. When an interface lacks one, the right move is to document the limit and use a different interface, not ship an approximation that looks finished.

## 5. Treat every input as a claim, not a fact

When a solar forecast reads high, separate two questions: is the forecast wrong, or is the configuration wrong? Then ask whether the error even changes the decision. A forecast that feeds a threshold only matters near the threshold.

Why it matters: accuracy is worth chasing where it changes a decision.

## 6. Replace a fixed gate with a computed one

The first version asked "is today's forecast above a fixed number of kWh?". Real days showed the forecast running high and the house never reaching surplus on days the gate approved. The fix was a gate computed per day: the corrected forecast must beat what the house will use that day, including laundry and cooking days from the calendar.

Why it matters: a fixed number encodes one season and one household. A computed one adapts to both.

## 7. Learn the forecast's error from the roof

A forecast service models a generic roof. The meter measures the real one. Comparing the two every night and keeping a moving average gives a correction that follows seasons, shading and sun angle without anyone retuning.

## 8. Make failure loud

An automatic system fails quietly: a restart leaves a stale switch, an integration hangs, a forecast stops updating. A daily check plus a dead-man's-switch ping turns each of those into a message.

## 9. Pick the right environment at kickoff

The instinct for "an app that controls my house" is to build an app. Every input already had a Home Assistant integration, and the logic is one declarative package. A custom app would mean re-implementing the integrations and running a 24/7 backend, for a worse result.

Why it matters: the highest-leverage decision on a build is often what not to build.

## 10. Keep personal values out of the framework

Every number that depends on one household is a tunable with guidance on how to set it. Every entity id is a placeholder with a mapping table. This keeps the framework reusable and keeps one home's data out of a public repository.
