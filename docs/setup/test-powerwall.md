# Test the Powerwall 3 read on your laptop (no Pi needed) - ~15 min, $0

Proves the project's one unproven link (local PW3 read) before you buy any hardware.

## What you need first
The **Gateway Wi-Fi password** - printed on the QR sticker on your Powerwall 3 / Gateway.
NOT your Tesla app/account password - those do not work for the local TEDAPI.

## Steps
1. On your laptop, join the Powerwall's own Wi-Fi. The network looks like `TeslaPW_XXXXX`
   (or `TEG-XXXX`). That adapter loses internet while joined - expected; PW3 firmware
   25.10.1+ requires a direct connection to this AP.

2. Confirm you can reach the gateway:
   ```
   curl -k --head https://192.168.91.1
   ```
   Any HTTP response = reachable. (Self-signed cert, hence `-k`.)

3. Install and run pypowerwall:
   ```
   python -m venv pwtest
   # Windows:        pwtest\Scripts\activate
   # macOS / Linux:  source pwtest/bin/activate
   pip install pypowerwall
   python -m pypowerwall tedapi -gw_pwd YOUR_GATEWAY_WIFI_PASSWORD
   ```

## Reading the result
- **Prints live battery % (SOC) and solar power** -> SUCCESS. These are the exact two
  values the automation reads. The riskiest piece is proven; hardware is now just
  "where to run HA 24x7."
- **"Access Denied / check your Gateway Password"** -> wrong password. Use the QR-sticker
  Gateway Wi-Fi password, not the Tesla account one (case-sensitive).
- **Can't reach 192.168.91.1** -> not actually on the Powerwall Wi-Fi; rejoin and retry.

Paste the output back and I'll confirm what we're seeing and pull out the SOC + solar fields.
