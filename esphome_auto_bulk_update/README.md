# ESPHome Device Auto Bulk Update

A Home Assistant blueprint that automatically handles ESPHome device firmware
updates so you don't have to — with actionable notifications, a configurable
time window, sequential or bulk update modes, and a cooldown to prevent
re-triggering immediately after updates complete.

[![Import blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Fbferd%2Fhomeassistant-blueprints%2Fmain%2Fesphome_auto_bulk_update%2Fesphome_auto_bulk_update.yaml)

📣 [Community forum post](https://community.home-assistant.io/t/esphome-device-auto-bulk-update/997537)

---

## What it does

- Monitors all ESPHome device firmware update entities and triggers when the
  number of pending updates reaches a configurable threshold.
- Sends an actionable notification listing which devices will be updated, with
  **Update Now** and **Cancel** buttons.
- Waits a configurable delay before proceeding — tap **Update Now** to skip the
  wait, or **Cancel** to abort entirely.
- Updates all devices at once, or one at a time sequentially with a configurable
  delay between each.
- Sends a completion notification confirming which devices were updated.
- Waits out a configurable cooldown period before the automation can trigger
  again.
- Includes a daily sweep trigger at the start of the update window to catch
  anything the template trigger may have missed (e.g. if updates were already
  pending when the previous run completed, the rising-edge trigger won't re-fire
  until the count drops and rises again — the daily sweep covers that gap).

## Requirements

- ESPHome integration installed and configured in Home Assistant.
- Home Assistant Companion App installed on your notification device.
- Firmware update entities must be **enabled** for each ESPHome device you want
  included — Settings → Devices & Services → ESPHome → your device → enable the
  Firmware Update entity. Devices without this entity enabled are invisible to
  the blueprint.

## Installation

1. Enable firmware update entities for each ESPHome device you want covered
   (Settings → Devices & Services → ESPHome → your device → Firmware Update
   entity → Enable).
2. Click the import button above, or go to Settings → Automations & Scenes →
   Blueprints → Import Blueprint and paste the raw URL to
   `esphome_auto_bulk_update.yaml` in this repo.
3. Create a new automation from the imported blueprint.
4. Configure the options to your preference — no YAML editing required.

## Flow

1. Triggers when the count of pending-update devices exceeds the threshold, or
   at the daily sweep time at the start of the update window.
2. Checks that the current time is within the configured update window and
   that the device count condition is still met.
3. Sends an actionable notification listing devices to be updated with
   **Update Now** and **Cancel** buttons.
4. Waits for the configured delay, listening for a button response.
5. If **Cancel** is tapped — sends a cancellation notification and stops.
6. If **Update Now** is tapped, or the delay expires with no response — proceeds
   with updates.
7. Updates all devices at once or sequentially depending on the mode selected.
8. Sends a completion notification listing updated devices.
9. Waits out the cooldown period before the automation can trigger again.

## Configuration

| Input | Description |
|---|---|
| Device Update Threshold | How many devices need a pending update before the automation triggers (1–20) |
| Delay Before Updating | How many minutes to wait before pushing updates — tap Update Now to skip (0–60 min) |
| Cooldown After Update | Minutes to wait after updates complete before the automation can trigger again (0–120 min) |
| Update Window Start | Earliest time of day updates can be triggered; also the time of the daily sweep |
| Update Window End | Latest time of day updates can be triggered |
| Notification Device | Companion app device to notify before updates are pushed |
| Update Mode | All at once, or one at a time sequentially |
| Delay Between Sequential Updates | Minutes to wait between each device in sequential mode (0–10 min) |

## Tips, gotchas, and known limitations

- If you dismiss the notification without tapping a button, the automation waits
  out the full delay and then proceeds with updates automatically.
- On Android, if the action buttons aren't visible, try long-pressing or
  expanding the notification.
- On iOS, long press the notification to reveal the action buttons.
- Sequential mode is gentler on your network and the ESPHome add-on —
  recommended if you have more than a handful of devices.
- If an update fails on one device in sequential mode, the automation continues
  to the next device (`continue_on_error: true`).
- The template trigger fires on a **rising edge** — it fires once when the count
  first crosses the threshold, not on every subsequent check. If updates are
  already pending when the previous run ends (e.g. new updates arrived during
  the cooldown), the trigger won't re-fire until the count drops to zero and
  rises again. The daily sweep at the window start covers this gap.
- `mode: single` — if the automation is already running, a second trigger is
  dropped rather than queued. This prevents double-runs during the cooldown.
- Make sure your update window is wide enough to accommodate the delay plus the
  total update time for all your devices.

## Changelog

| Version | Date | Changes |
|---|---|---|
| v1.1.0 | 2026-03-20 | Removed optional second notification device (caused errors when left blank) |
| v1.0.0 | 2026-03-20 | Initial release |

## Credits

Blueprint by [bferd](https://github.com/bferd).
