# ecobee / Generac native developer API request (for deferred tweak #5 + threshold lever)

**Why:** The SmartThings path can set mode (off/cool/heat/auxheatonly) but CANNOT set two ecobee-native
settings we want: the dual-fuel switchover thresholds, and the fan minimum on-time (fanMinOnTime, the
0/20/55 min-per-hour control behind tweak #5). Only the native ecobee API exposes these. The public
developer portal is closed to new self-serve registrations, so this is a direct ask.

**Where to send:** the contact form at **ecobee.com/en-us/developers**, or reply to ecobee customer
support and ask to be routed to developer relations.  [VERIFY: no direct dev-team email confirmed - the
portal contact form is the safe entry point.]

**Approval odds:** unknown. Some individuals get grandfathered/partner keys; many requests go unanswered.
Low-cost shot, worth firing in parallel; not a dependency.

---

## Subject
Developer API access request - single residential thermostat (personal, non-commercial)

## Body
Hello ecobee Developer team,

I own an ecobee Smart Thermostat Premium on a dual-fuel system (electric heat pump + gas furnace), and
I'm building a personal, non-commercial automation for my own home that selects the heat source based on
my on-site solar production and battery charge, to maximize self-consumption and minimize grid draw.

I understand the public developer portal is no longer accepting new registrations. I'd like to request API
access for a single thermostat on my own account. Specifically I need to:

- read thermostat state and runtime;
- set HVAC mode, including auxHeatOnly (emergency/aux heat);
- set the dual-fuel threshold settings (compressorMinOutdoorTemp, auxHeatMaxOutdoorTemp); and
- set the fan minimum on-time per hour (fanMinOnTime).

Call volume would be light - a handful of reads and at most a few writes per day. This is for my own
residence only, with no resale or commercial use.

Could you let me know whether individual or partner API access is available, and what the process is to
obtain a key?

Thank you,
<your name>

---
**Before sending:** keep it one screen; the concrete endpoint/setting names (auxHeatOnly, fanMinOnTime,
the threshold settings) signal you know the API and aren't a casual request - leave them in.
