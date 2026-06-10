# Day-0 setup - Home Assistant on a Windows computer you own

Goal: stand up HA, wire the three data sources, drop in the package, and reach the
notify-only shakedown - in one sitting. No furnace control happens until you flip
`notify_only` off at the end.

## 0. Before you start
- The PC must stay **always-on and not sleep** while testing (and after, if this is the
  permanent home). Settings -> System -> Power -> Screen & sleep -> set sleep to **Never**.
- **Pause auto-updates for the test:** Settings -> Windows Update -> **Pause updates** (1 week)
  so a nightly reboot doesn't interrupt the run. (In notify-only a reboot is harmless anyway -
  it just leaves a gap in the Logbook - so this is for continuity, not safety.)
- Free resources for the VM: ~2 GB RAM, ~32 GB disk, 2 CPU cores.

> **Moving to the Mac (or a dedicated box) later?** You won't rebuild. Settings -> System ->
> Backups -> create a backup; on the new host, flash a fresh HA OS and **restore that backup**.
> Config, all three integrations, the package, and history come with it. Only the 20-minute
> base OS install is redone. So building here now is not throwaway work.

## 1. Install VirtualBox
Download from virtualbox.org and install (accept the network-driver prompt).

## 2. Get the Home Assistant OS image
From https://www.home-assistant.io/installation/windows/ download the **VirtualBox (.vdi)**
image, then **unzip** it to a folder you'll keep (the VM boots from this file).

## 3. Create the VM  (the EFI + bridged steps are the ones people miss)
New -> Name "Home Assistant", ISO = None, Type **Linux**, Version **Oracle Linux (64-bit)**.
- Memory: **2048 MB+**,  CPUs: **2+**
- **Enable EFI / UEFI (same thing):** tick the **Enable EFI** or **UEFI** box - it may be in
  the VM creation wizard, otherwise in VM Settings -> System -> Motherboard. HAOS only boots in
  UEFI mode - miss this and it won't start.
- Disk: choose **Use an existing virtual hard disk file** -> select the unzipped **.vdi**.
- Network: Settings -> Network -> **Attached to: Bridged Adapter** -> Name = the adapter you
  use for internet. (Bridged gives HA its own IP so the phone app and your browser can reach it.)
- **Auto-start after reboots: SKIP THIS FOR THE TEST.** Just start the VM by hand each time
  (open VirtualBox, click "Home Assistant", click **Start**) and leave the PC on. When you
  later want it running 24/7 on its own, follow **Appendix A** at the bottom - plain steps.

## 4. Boot + onboard
Start the VM. After a few minutes, browse to **http://homeassistant.local:8123** (or the IP
shown in the VM console). Create your account. Note the IP - you'll use it for the phone app.

## 5. Add the File editor add-on (to edit config from the UI)
Settings -> Add-ons -> Add-on Store -> **File editor** -> Install -> Start. (Optional: also
"Terminal & SSH".) This lets you edit `configuration.yaml`, `secrets.yaml`, and the package.

## 6. Wire the three data sources
**a. Teslemetry (Powerwall):** Settings -> Devices & Services -> Add Integration -> Teslemetry
   -> sign in, authorize **only the Powerwall**. Then Developer Tools -> States, filter by your
   site name, and copy the two entity IDs: battery **percentage_charged** and **solar_power**.
**b. Solcast (forecast):** install HACS first (follow https://hacs.xyz/docs/setup/download -
   needs a GitHub login), then in HACS add **BJReplay/ha-solcast-solar**, then add the Solcast
   integration with your free API key. Note the "forecast tomorrow" sensor's exact entity ID.
**c. ecobee:** nothing to install - the package calls SmartThings directly with your token.

## 7. Install the package
1. configuration.yaml - add (once):
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
2. Create folder `/config/packages/` and copy **solar_hvac.yaml** into it (File editor).
3. secrets.yaml - add your 24h SmartThings token:
   ```yaml
   smartthings_bearer: "Bearer xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
   ```
4. In solar_hvac.yaml, replace the **SITE_** placeholders with the real Teslemetry entity IDs
   from step 6a, and confirm the Solcast sensor name from 6b.
5. Developer Tools -> YAML -> **Check Configuration** (fix anything flagged) -> **Restart**.

## 8. Start the shakedown (still no furnace control)
Turn ON `input_boolean.hvac_automation_enabled` AND `input_boolean.hvac_notify_only`.
In notify-only the scripts only **log** "WOULD set HEAT/AUX" to the Logbook - they don't switch.

## Day-0 go / no-go checklist (run these now)
- [ ] `sensor.hvac_battery_soc` shows a live number (matches the Tesla app).
- [ ] `sensor.hvac_solar_kw` shows a live number.
- [ ] `sensor.ecobee_mode_raw` shows one of: off / heat / cool / auxheatonly.
- [ ] `sensor.hvac_forecast_tomorrow` shows a kWh number.
- [ ] Control round-trip: briefly turn `notify_only` OFF, run `script.hvac_set_aux`, watch
      `sensor.ecobee_mode_raw` flip to `auxheatonly`, then `script.hvac_set_heat` -> `heat`,
      then set the thermostat back to where it was. (Summer/no heat demand = mode flips, no
      furnace run.) Turn `notify_only` back ON.

If all five pass, the system is fully wired and you're in the 2-3 day notify-only watch.

## During the 2-3 day test
- Each morni


---

## Appendix A - Make Home Assistant start automatically (only when you go permanent)

You do NOT need this for the 2-3 day test. During the test, just open VirtualBox, click
"Home Assistant", click **Start**, and leave the PC on.

When you want HA to run on its own 24/7, this makes Windows launch it every time the PC starts -
quietly in the background, so there's no window to accidentally close. (You still open it the
same way, in your web browser at http://homeassistant.local:8123.)

Do this once:

1. Hold the **Windows key** and tap **R**. A small "Run" box appears.
2. Type **shell:startup** and press **Enter**. A folder opens. (Anything placed in this folder
   runs automatically whenever Windows starts.)
3. **Right-click** an empty spot in that folder -> **New** -> **Shortcut**.
4. In the "Type the location of the item" box, copy-paste this line EXACTLY, including the
   quotation marks:

   `"C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" startvm "Home Assistant" --type headless`

5. Click **Next**.
6. Type a name: **Start Home Assistant**. Click **Finish**.
7. That's it. To test it: restart the PC. Once it's back and you're signed in, wait about
   2 minutes, then open http://homeassistant.local:8123 in your browser - HA should load
   without you touching VirtualBox.

Notes:
- This runs after you sign in to Windows. For a truly hands-off reboot (nobody at the PC), also
  turn on Windows auto-login - ask me and I'll give you those click-by-click steps too.
- To stop HA by hand: open VirtualBox, click "Home Assistant", choose **Close -> Power Off**.
- If step 4 gives a "file not found" error, VirtualBox is installed in a different folder - tell
  me and I'll give you the corrected line.
