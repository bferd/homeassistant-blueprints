# Adaptive Fan Speed Control

A Home Assistant blueprint that automatically sets fan speed based on one or more sensors — temperature, humidity, air quality — and/or a thermostat's deviation from setpoint, with configurable cooldown throttling, polling frequency, and unavailable-sensor handling.

## What it does

- Maps each configured sensor's value — temperature, humidity, an air quality reading (CO2, PM2.5, VOC, AQI, etc.) — or a thermostat's deviation from its setpoint, to a fan speed percentage between a configurable min/max range.
- Auto-detects which mode applies to each sensor from its `device_class`, so you only need to fill in the threshold sliders for whichever mode(s) are relevant — the rest are ignored.
- Supports multiple sensors at once (e.g. a thermostat and an air quality sensor in the same room, or temperature and humidity together) — whichever one currently computes the highest fan speed wins. Nothing gets averaged or diluted: a sensor that's satisfied can never suppress a different sensor's more urgent demand.
- Air quality sensors always bypass the cooldown delay, since an air quality event is more time-sensitive than a comfort-oriented one — every other sensor type respects it normally.
- For humidity specifically, supports both directions: more fan as humidity rises (e.g. a bathroom exhaust fan during a shower), or more fan as humidity falls.
- Turns the fan off only once every configured sensor agrees it's satisfied, and (optionally) back on automatically once any one of them crosses back above its threshold.
- Throttles how often speed changes happen (cooldown delay) and how often sensors are even checked (polling frequency), independent of how often the sensors themselves report updates.
- Won't drop a manual fan-on event during an active cooldown — it queues and runs once the cooldown clears, rather than being silently discarded.
- If a sensor becomes unavailable, it's excluded from the calculation rather than treated as a zero — the fan keeps responding to whichever sensors are still healthy, and only freezes at its current speed if every configured sensor goes down at once. An optional companion app notification fires whenever at least one sensor is down (listing which), throttled to once per outage plus a one-time "recovered" notification, if you provide a tracking helper.
- Supports an optional **Ambient Reference Sensor** (e.g. an outdoor temperature sensor, or a home-average temperature/humidity helper) that caps the effective off threshold to what's physically achievable — so if it's 26°C outside, the fan stops trying to push the room below 26°C rather than running at full speed indefinitely toward an impossible target.
- Supports an optional blocking entity (e.g. sleep mode, away mode) with a configurable action — **Pause** (fan coasts at current speed, no adjustments) or **Shut off** (fan turns off immediately when the blocker activates).
- Supports an optional custom trigger input for advanced use — e.g. reacting on every native sensor update instead of polling, or a custom sub-minute polling interval.

## Multiple sensors

The Sensors or Thermostat(s) input accepts any number of `sensor.*` and `climate.*` entities. Each one is evaluated completely independently — its own mode detection, its own threshold sliders, its own computed target speed — and the fan is set to whichever of those targets is currently highest. This is deliberately a max, not an average: averaging would let a sensor that's perfectly happy quietly water down a different sensor's urgent demand, which is exactly the failure mode you don't want from a safety-relevant signal like air quality. Concretely: if a thermostat wants 0% (room's comfortable) and an air quality sensor wants 75% (CO2 is climbing), the fan runs at 75% — the comfortable sensor's "0" never gets averaged in, it just loses the comparison. The reverse holds too: a sensor demanding less doesn't get artificially boosted by one demanding more, and a sensor demanding more never gets capped down by one demanding less, regardless of which sensor that happens to be.

The fan only turns off once *every* configured entity's own off-threshold check agrees it's satisfied. If even one of them still wants the fan on, it stays on at whatever speed the highest-demanding entity currently computes.

Air quality sensors get one more behavioral difference: any configured air quality sensor that's currently active (above its own off threshold) makes the *entire* automation bypass the Change frequency delay cooldown for that check, regardless of the Enable Change Frequency Delay toggle. The reasoning: the cooldown exists to stop comfort-oriented sensors (temperature, humidity) from making jittery, frequent adjustments, which is fine since comfort drift is slow. Air quality can spike fast, and you don't want it stuck waiting out a cooldown that a different sensor's prior adjustment started. Pair this with a fast Update Check Frequency if air quality response time matters to you — the cooldown bypass only helps if the periodic check itself is also running often.

**Limitation:** since mode detection and threshold sliders are per *type*, not per sensor instance, two sensors of the same type (e.g. two separate humidity sensors) share one set of threshold sliders rather than getting independently configurable ones. This covers combining *different* kinds of sensor in one room (a thermostat plus an air quality sensor, or temperature plus humidity) but not "two humidity sensors with two different setpoints" — that would need a much larger redesign (effectively a separate threshold system per sensor slot), which isn't what this blueprint does.

**Partial vs. full sensor outages:** if some configured sensors are unavailable but at least one is still healthy, the automation keeps operating off whichever ones are still reporting — an unavailable sensor is excluded from the max calculation, not treated as if it were reporting 0. A notification fires either way (if Notify Device is set) listing which sensor(s) are down. Only when *every* configured sensor is unavailable simultaneously does the automation freeze the fan at its current speed, since at that point there's no data left to compute a response from.

## Detected modes

Which mode applies to a given entity — and which threshold inputs actually do anything for it — is auto-detected per entity, not a separate input you set: **all four modes' inputs are always shown in the UI; blueprints can't conditionally hide one based on another input's value**, so whichever sets don't apply to a particular entity just get ignored for that entity.

- **Climate entity selected → Thermostat Mode.** Deviation Amount and Deviation Off are used, as degrees of deviation from the thermostat's setpoint.
- **Sensor with `device_class: humidity` → Humidity Mode.** Maximum/Minimum/Off Humidity are used, plus the Humidity Direction selector (High or Low — see below).
- **Sensor with an air-quality `device_class`** (`aqi`, `carbon_dioxide`, `carbon_monoxide`, `nitrogen_dioxide`, `nitrogen_monoxide`, `nitrous_oxide`, `ozone`, `pm1`, `pm25`, `pm4`, `pm10`, `sulphur_dioxide`, `volatile_organic_compounds`, `volatile_organic_compounds_parts`) **→ Air Quality Mode.** Maximum/Minimum/Off Air Quality are used. High direction only — more pollutant always means more fan, there's no Low option for this one. Also always bypasses the cooldown delay — see "Multiple sensors" above.
- **Anything else → Temperature Mode (the fallback).** Covers a `device_class: temperature` sensor, but also any sensor with no `device_class` set at all — so a plain temperature sensor needs zero extra configuration, same as before this feature existed.

These used to be a single set of three sliders, reinterpreted depending on what was selected — that turned out to be a real source of bugs (wrong units, wrong slider range, no validation stopping you from setting Min above Max), so each mode gets its own fully separate inputs with their own ranges and defaults.

**A caveat on auto-detection:** since Temperature Mode is the fallback for "nothing else matched," a humidity or air quality sensor that doesn't have its `device_class` attribute set (some custom ESPHome sensors omit this) will silently fall into Temperature Mode and use the wrong sliders, rather than erroring. There's no reliable way to detect this case from inside the automation — a sensor with no `device_class` looks identical whether it's a legitimate plain sensor or a mis-tagged humidity sensor. Worth a quick check in Developer Tools → States that your sensor's `device_class` attribute is what you expect before assuming it's working.

### Humidity Direction

- **High Mode** (default): fan speed increases as humidity rises, and turns off below Off Humidity. This is the shape you want for something like a bathroom exhaust fan reacting to a shower.
- **Low Mode**: fan speed increases as humidity falls instead, and turns off *above* Off Humidity. Useful for the opposite kind of problem — e.g. circulating air when it's gotten too dry.

In both directions, Minimum Humidity must stay numerically smaller than Maximum Humidity — what changes between directions is which end maps to which fan speed (High: Min→min speed, Max→max speed; Low: Min→max speed, Max→min speed), and which side of the range Off Humidity sits on (below Minimum in High Mode, above Maximum in Low Mode). Internally this is done with the same sign-flip technique the blueprint already uses for thermostat Heat/Cool direction, just applied to humidity instead.

### Thermostat mode mechanics

When a thermostat is selected, the direction of the deviation flips automatically with HVAC mode:

- In **Heat** mode, the fan ramps up the further the room runs *below* setpoint.
- In every other mode (Cool, Heat/Cool, Auto, Dry, Fan Only), the fan ramps up the further the room runs *above* setpoint.

Deviation Amount is the deviation (in either direction) that maps to maximum fan speed. Deviation Off is the threshold below which the fan turns off — and it also doubles as the floor of the ramp range, since there's no separate "minimum deviation" input. That's a real simplification compared to the sensor modes: those have three independent thresholds (Off, Min, Max), so there can be a small dead zone between the off threshold and where speed ramping starts. Thermostat mode collapses that into two thresholds, so there's no dead zone — ramping starts immediately above the off threshold.

**Single-setpoint thermostats only.** This relies on the climate entity's `temperature` attribute (a single target). Dual-setpoint thermostats — anything using separate `target_temp_high`/`target_temp_low` instead, typically in a Heat/Cool or Auto mode — don't expose that attribute, so the setpoint value comes back empty. That failure mode is handled gracefully: the same unavailability check used for a missing/unknown sensor also checks whether the thermostat's `temperature` or `current_temperature` attributes are `None`. If either is missing, that entity is excluded from the calculation exactly like an unavailable sensor — leaving the rest of the configured sensors (if any) to drive the fan — rather than computing a deviation against a missing setpoint.

## Ambient Reference Sensor

The Ambient Reference Sensor input accepts an optional single temperature or humidity sensor that represents the ambient conditions physically limiting what the fan can achieve. When set, the effective off threshold for matching sensor types is adjusted so the fan stops trying once the room reaches ambient, rather than continuing to run toward a setpoint that ambient conditions make impossible.

**Temperature / Thermostat** — the most natural use case is an outdoor temperature sensor for a fan pulling in fresh air from outside. If it's 28°C outside, no amount of fan speed will cool a room below 28°C, so the fan turns off when the room reaches 28°C rather than thrashing at 100% indefinitely. A home-average temperature helper (HA's "Combine the state of several sensors" group helper, set to arithmetic mean) works the same way as an indoor ambient reference.

**Humidity** — same idea. If home average humidity is 60%, a bathroom exhaust fan can't reduce the bathroom below 60% no matter how hard it runs. Set the home average humidity helper as the ambient reference and the fan turns off once the bathroom reaches 60%.

The direction is handled automatically per mode:
- Temperature sensors and Thermostat Cool mode: `effective_off = max(configured_off, ambient)` — fan stops when room reaches the higher of the two
- Thermostat Heat mode: `effective_off = max(configured_off, setpoint − ambient)` in deviation space — fan stops when the room reaches ambient from below
- Humidity High mode: `effective_off = max(configured_off, ambient)` — same as temperature
- Humidity Low mode: inverted automatically via the existing sign-flip — fan stops when room rises to ambient from below

If the ambient sensor goes unavailable, the correction silently disables itself and the automation falls back to the configured off thresholds as-is. Air quality sensors are not affected — there's no meaningful physical ceiling for an air quality reading in the same sense.

## Blocking entity

The blocking entity input disables the automation when the entity is `on`. Two modes are available via the Blocking Entity Action input:

- **Pause** — the automation stops making adjustments. The fan stays at whatever speed it was running when the blocker activated.
- **Shut off** — the fan turns off immediately when the blocker activates and stays off until it deactivates.

In Shut off mode, the fan turns off on the next periodic check cycle rather than instantly, since the blocker entity isn't a trigger by default. For an immediate response (e.g. you want the fan off the moment Quiet Time turns on at midnight), add a state trigger on your blocker entity to the Custom Trigger Override input — set platform to **state**, entity to your blocker entity, and **to** to `on`. That fires the automation the instant the blocker activates, which immediately hits the shutoff branch.

## Installation

1. Settings → Automations & Scenes → Blueprints → Import Blueprint
2. Paste the raw URL to `adaptive_fan_speed_control.yaml` in this repo, or use the import button at the top of the [community forum post](https://community.home-assistant.io/t/adaptive-fan-speed-control/1014995)
3. Create a new automation from the blueprint and fill in your fan, sensors or thermostat, and preferences

## Configuration

| Input | Description |
|---|---|
| Sensors or Thermostat(s) | One or more `sensor.*` (temperature, humidity, or air quality) and/or single-setpoint `climate.*` entities. Mode is auto-detected per entity — see "Detected modes" above. Combined via max-wins — see "Multiple sensors" above. |
| Fan | The fan entity to control |
| Minimum / Maximum Fan Speed | The speed range the fan will be set to |
| Minimum / Maximum Temperature | Temperature Mode only — absolute degrees used to map to fan speed. Ignored otherwise. |
| Off Temperature | Temperature Mode only — below this, the fan turns off (and optionally back on above it). Ignored otherwise. |
| Humidity Direction | Humidity Mode only — High (more fan as humidity rises) or Low (more fan as humidity falls). Ignored otherwise. |
| Minimum / Maximum Humidity | Humidity Mode only — which end maps to which fan speed depends on Humidity Direction. Ignored otherwise. |
| Off Humidity | Humidity Mode only — below Minimum in High Mode, above Maximum in Low Mode. Ignored otherwise. |
| Minimum / Maximum Air Quality | Air Quality Mode only — reading mapped to fan speed, in whatever unit your sensor reports. Ignored otherwise. |
| Off Air Quality | Air Quality Mode only — below this, the fan turns off. Ignored otherwise. |
| Deviation Amount | Thermostat Mode only — degrees of deviation from setpoint that maps to maximum fan speed. Ignored otherwise. |
| Deviation Off | Thermostat Mode only — degrees of deviation below which the fan turns off; also the floor of the ramp range. Ignored otherwise. Must be smaller than Deviation Amount. |
| Enable auto fan on | Whether the fan turns back on automatically once it crosses back above the off threshold |
| Update Check Frequency | How often to re-check and consider an adjustment (5–60 seconds) |
| Custom Trigger Override (optional) | Add your own trigger(s) — e.g. a state trigger on the sensor for every-update reactivity, or a custom sub-minute time pattern |
| Change frequency delay | Minimum time between actual speed adjustments, in seconds (30 to 1200, with the minute equivalent shown in each dropdown option) |
| Enable Change Frequency Delay | Toggle the above cooldown on or off. Turn off temporarily while testing — manually nudging a thermostat and watching the fan respond every check — since the cooldown otherwise sleeps through several checks at once, which can make speed changes appear to skip straight to a much later value instead of stepping through the ramp. |
| Minimum / Maximum percentage change | Bounds on how much the fan speed can change per adjustment |
| Blocking entity (optional) | If this entity is "on", the automation pauses or shuts off the fan depending on Blocking Entity Action |
| Blocking Entity Action | What to do when the blocker is active — Pause (fan coasts) or Shut off (fan turns off). For instant shutoff, also add a state trigger on the blocker to Custom Trigger Override — see "Blocking entity" above. |
| Ambient Reference Sensor (optional) | A temperature or humidity sensor representing ambient conditions (e.g. outdoor temp, home average humidity) — caps the effective off threshold so the fan stops trying once it reaches ambient. See "Ambient Reference Sensor" above. |
| Notify Device (optional) | Companion app device to notify if the sensor/thermostat becomes unavailable or unreadable |
| Notify State Helper (optional) | An `input_boolean` you create — tracks notification state so the unavailable notification fires once per outage instead of every check, and enables a one-time "recovered" notification. Leave blank for the old every-check behavior with no recovery notification. |

## Known limitations

- The Custom Trigger Override is additive, not exclusive — anything added there runs alongside the periodic check, not instead of it. Blueprints can't conditionally disable one static trigger based on another input's value.
- Dual-setpoint thermostats (separate heat/cool targets) aren't supported when a thermostat is selected — see "Thermostat mode mechanics" above for what happens instead.
- Mode is `queued` with a max of 2 (current run + 1 queued) rather than unbounded, so a very fast custom override trigger combined with a long cooldown can't build an unbounded backlog — excess trigger events beyond that are dropped (logged as a warning) rather than queued indefinitely.
- All four modes' input sets always show in the UI regardless of which entities are selected, since blueprints can't conditionally hide inputs.
- Threshold sliders are shared per detected type, not per sensor instance — two sensors of the same type in the list (e.g. two humidity sensors) can't have independently configured thresholds. See "Multiple sensors" above.
- Mode detection happens via each sensor's `device_class` attribute, and there's no way to surface "this is the mode I detected for each one" inside the blueprint's own configuration form — blueprint inputs are static and can't be templated against another input's current value, confirmed against HA's own blueprint schema docs (and there's an open, unimplemented feature request for exactly this on HA's GitHub). If you want confirmation of which mode got picked per entity, that only becomes visible after the automation is saved and has run at least once — e.g. via its trace, not during setup.
- Requires a reasonably recent Home Assistant Core release for the `select`/`device`/`entity` selector filter syntax used in the inputs.

## Changelog

| Version | Changes |
|---|---|
| v1.1.0 | Added Ambient Reference Sensor (outdoor temp / home average) to cap the effective off threshold at what's physically achievable. Added Blocking Entity Action toggle (Pause vs Shut off). Cooldown delay now uses `wait_template` instead of `delay` so it exits immediately if the blocker activates mid-wait, enabling near-instant fan shutoff when combined with a Custom Trigger Override on the blocker entity. |
| v1.0.0 | Initial release — multi-sensor support, Temperature / Humidity / Air Quality / Thermostat modes, max-wins combination, AQ cooldown bypass, partial sensor degradation, notification throttle with recovery notification, change frequency delay toggle. |

## Credits

This blueprint builds on prior work from the Home Assistant community:

- [lennon101](https://community.home-assistant.io/u/lennon101) — original author of [Adaptive fan speed control based on temperature and speed range](https://community.home-assistant.io/t/adaptive-fan-speed-control-based-on-temperature-and-speed-range/678152), which this repo forks and extends.
- [MickW69](https://community.home-assistant.io/u/mickw69) — original idea and code via [Multi-speed Fan Control Based on Temperature](https://community.home-assistant.io/t/multi-speed-fan-control-based-on-temperature/552322), credited by lennon101 as the basis for the original blueprint.

All credit for the original concept and groundwork goes to the above. Bugs introduced in this fork are mine.

---

## Support

If this project saved you some time or you just want to say thanks, a coffee is always appreciated!
 
[![Buy Me A Coffee](https://img.shields.io/badge/Buy%20Me%20A%20Coffee-support-%23FFDD00?style=for-the-badge&logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/schrothdotca)


