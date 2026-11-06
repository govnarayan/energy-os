# Reading a Powerwall 3 without a subscription

Energy OS needs two live readings from the battery system: charge percent and solar power. This note compares the ways to get them from a Tesla Powerwall 3 and records the free route that was tested.

## The options

| Route | Cost | Needs | Works when the cover stays closed |
|---|---|---|---|
| Local read over your network (community integration, or Home Assistant's Teslemetry local mode) | Free | The gateway Wi-Fi password, from a QR label inside the Powerwall's cover | No. Tesla and many installers treat opening it as voiding the warranty |
| Teslemetry (cloud) | Paid subscription | A Tesla login | Yes |
| Tesla Fleet API through Home Assistant's Tesla Fleet integration | Free for energy data, with a $10 monthly credit from Tesla | A Tesla developer app, a hosted public key, a Tesla login | Yes |

If your installer can give you the gateway password, the local read is the best option: no cloud, fast updates. If it cannot, the Fleet API is free and needs a one-time setup.

## Fleet API setup, tested on a Powerwall 3

1. Create a Tesla developer application at developer.tesla.com for yourself ("just for me"). Use the grant type Authorization Code and Machine-to-Machine. Select one scope: Energy Product Information. Skip billing.
2. Set the allowed redirect to `https://my.home-assistant.io/redirect/oauth`.
3. Host the public key Home Assistant gives you. A key host such as FleetKey.net (run by Teslemetry) works and needs no domain of your own. Its page gives you a domain and tells you the origin to enter in the Tesla app. If you prefer your own domain, serve the key at `/.well-known/appspecific/com.tesla.3p.public-key.pem` on a subdomain.
4. In Home Assistant, add the Tesla Fleet integration. Paste the client ID and client secret, sign in to Tesla, approve, enter your key domain, then submit. Registration can fail for a few minutes until Tesla sees the hosted key.
5. Skip the virtual key step. It is only for vehicles.
6. Rename the new device so entity names stay generic, then map `sensor.<device>_percentage_charged`, `sensor.<device>_solar_power` and `sensor.<device>_solar_generated` in the entity table.

## What to know

1. Live readings refreshed about every 50 seconds in the test, close to the paid service.
2. The Fleet solar generated sensor is a daily total that resets at local midnight. The learning step in `energy_os.yaml` reads this kind of sensor directly.
3. A key host run by another company is a dependency. If it disappears, move the key to your own domain and register again.
4. Tesla may disable an application that has no payment method on file. Home Assistant's documentation calls billing optional. If it happens, add a card. Energy data itself is free.
5. This is a community-tested path, not a Tesla-supported one. Interfaces can change.
