# Roadmap

Energy OS is the framework. Home Energy Copilot is the product built on it. This lists what turns one working home into something that fits many.

## Now

1. The operate layer runs in watch mode on one home.
2. The design method is documented.
3. The health layer runs daily with a dead-man's-switch ping.

## Next: make the operate layer portable

1. Test the entity map on a second home with different hardware. Record what breaks.
2. Handle single-fuel homes: heat pump only, gas only, and electric resistance.
3. Replace the hard-coded morning and evening logic with settings for latitude and tariff hours.
4. Add a no-battery mode as a first-class option instead of a workaround.
5. Move from a package you edit by hand to a guided setup that scans the home's devices and fills the map.
6. Test the heat pump lock with a fresh Heat call while the lock is already open, and on a second thermostat model.
7. Evaluate local detection of thermostat changes (for example HomeKit) to cut the detection delay on thermostats whose cloud is polled every few minutes.

## Next: the design layer as a tool

1. A guided intake covering what installers ask and what they do not.
2. A month-by-month and hour-by-hour model from utility interval data.
3. A proposal checker that applies the nine questions to a real quote.
4. A tariff library, starting with one utility, with each rate sourced and dated.

## Later: the watch layer as a service

1. A hosted health page and alerts, so a failure never depends on the home's own network.
2. Fleet-level learning: forecast corrections shared across homes with similar roofs and climates.
3. A clear before-and-after report: what the home would have spent, and what it spent.

## Known limits today

1. Tested on one home. The portability claims are designs, not results.
2. The thermostat must allow software control of the gas side for the fuel choice to work.
3. Savings depend on roof, rates, equipment and habits. The framework makes no promise of a number.
4. Control stays in watch mode by default, and a person decides when to turn it off.
