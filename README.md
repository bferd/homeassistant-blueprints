# homeassistant-blueprints

Home Assistant automation blueprints by [bferd](https://github.com/bferd).

---

## Blueprints

### 🌡 Adaptive Fan Speed Control

Automatically sets fan speed based on one or more sensors — temperature,
humidity, air quality — and/or a thermostat's deviation from setpoint. Supports
multiple sensors simultaneously with max-wins combination, per-type threshold
sliders, air quality cooldown bypass, partial sensor degradation, and
unavailable-sensor notifications.

[![Import blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Fbferd%2Fhomeassistant-blueprints%2Fmain%2Fadaptive_fan_speed_control%2Fadaptive_fan_speed_control.yaml)

📄 [Full documentation](adaptive_fan_speed_control/README.md) · 📣 [Community forum post](https://community.home-assistant.io/t/adaptive-fan-speed-control-based-on-temperature-and-speed-range/678152)

---

### 🔄 ESPHome Device Auto Bulk Update

Automatically handles ESPHome device firmware updates on a schedule, with
actionable notifications (Update Now / Cancel), configurable time windows,
sequential or bulk update modes, and a cooldown to prevent re-triggering
immediately after updates complete.

[![Import blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Fbferd%2Fhomeassistant-blueprints%2Fmain%2Fesphome_auto_bulk_update%2Fesphome_auto_bulk_update.yaml)

📄 [Full documentation](esphome_auto_bulk_update/README.md) · 📣 [Community forum post](https://community.home-assistant.io/t/esphome-device-auto-bulk-update/997537)

---

## License

[MIT](LICENSE)
