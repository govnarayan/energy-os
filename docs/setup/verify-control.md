# Step 1 - Verify the AUX (force-gas) command on your ecobee Premium

> **RESULT - PASSED (2026-06-05).** SmartThings device `YOUR_DEVICE_ID` ("Thermostat").
> `supportedThermostatModes = ["off","cool","heat","auxheatonly"]`. API round-trip `off → auxheatonly → heat → off` all read back correctly; furnace audibly fired on `auxheatonly` (confirmed manually via the app with raised setpoint). **Confirmed mode strings: gas = `auxheatonly`, heat pump = `heat`, cooling = `cool`, off = `off`.** Full spec is buildable on SmartThings; native ecobee API is no longer required for the flip-based logic.

**Why this is the only real risk.** Reading battery + solar (Powerwall 3 local) and setting **Heat / Off / Cool / setpoint** are all reliable on every path. The single command the whole automation hinges on is putting the Premium into **emergency / auxiliary-heat mode** (= gas furnace only, heat pump locked out). The ecobee developer API is closed to new keys, so the realistic bridge is **SmartThings** - and whether *that* can set emergency heat on your unit is the make-or-break test. Run this before building anything.

The decisive test is the **raw SmartThings API**, not "does it show up in Home Assistant" (HA's climate layer is known to drop the emergency-heat mode even when SmartThings has it - the build doc works around that separately).

---

## Layer 0 - App level (≈5 min)

1. Install the **SmartThings** app, sign in.
2. **Add device → by brand → ecobee → Thermostat**, sign into your ecobee account, authorize. (This is the one-time account link; needs your hands.)
3. Open the thermostat tile. Look for a **mode selector** that includes **Emergency Heat** / **Auxiliary Heat**.
   - **Not present** → SmartThings can't drive AUX on your unit. Stop here; the native-API request (step 3 doc) or a Hubitat hub become the path.
   - **Present** → select it, and **physically confirm the gas furnace runs, not the heat pump** (listen/feel the vents; furnace air is hotter, no outdoor-unit hum). Then continue to Layer 1.

## Layer 1 - API level (≈10 min) - the actual proof

1. Generate a token at **account.smartthings.com/tokens** with device scopes (list/read/execute). SmartThings personal access tokens are now short-lived - fine for a one-time test; the page shows the current expiry.
2. Run these (replace `$TOKEN` and `DEVICE_ID`):

```bash
# Find the ecobee and its deviceId
curl -s -H "Authorization: Bearer $TOKEN" https://api.smartthings.com/v1/devices | grep -i -A2 ecobee

# Read the modes your unit actually exposes
curl -s -H "Authorization: Bearer $TOKEN" \
  https://api.smartthings.com/v1/devices/DEVICE_ID/components/main/capabilities/thermostatMode/status
# -> look at supportedThermostatModes; you want one of: "emergencyHeat", "emergency", or "auxHeatOnly"

# Force gas (use the exact value you saw above)
curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  https://api.smartthings.com/v1/devices/DEVICE_ID/commands \
  -d '{"commands":[{"capability":"thermostatMode","command":"setThermostatMode","arguments":["emergencyHeat"]}]}'

# Return to the heat pump
curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  https://api.smartthings.com/v1/devices/DEVICE_ID/commands \
  -d '{"commands":[{"capability":"thermostatMode","command":"setThermostatMode","arguments":["heat"]}]}'
```

## Pass / fail

| Outcome | Meaning | Next |
|---|---|---|
| `supportedThermostatModes` lists an emergency value **AND** the POST flips the thermostat to emergency/aux **AND** the furnace fires | **GREEN** - full spec is buildable on SmartThings | Go to Home Assistant build |
| App shows emergency heat but API rejects the value | Likely token scope or wrong mode string - retry with the exact value from the status call | Re-test |
| Emergency mode absent from app **and** `supportedThermostatModes` | SmartThings can't do AUX on your Premium | Pursue native API (step 3 doc) or a Hubitat hub (its Ecobee Suite exposes `emergency()` → `auxHeatOnly`) |

## My prediction (with the uncertainty stated plainly)

**Moderate-to-high confidence this works via the SmartThings API on a Premium.** Basis: the Premium hardware supports emergency heat (2H/2C + 2-stage aux); the SmartThings `thermostatMode` capability includes the emergency/`auxHeatOnly` value; and the documented failures were the **Essential** model (HA core issue #155382) and HA's **climate-entity layer**, not the SmartThings API itself on a Premium. What I have **not** seen is a Premium-specific, confirmed API success - that gap is exactly what this test closes. Don't take the green light as given until the furnace physically fires on the POST.

**Offer:** once the ecobee is linked and you have a token, I can run the GET/POST steps with you and read back what `supportedThermostatModes` returns.
