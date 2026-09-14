# PaperOS Core

The `core/` tree will contain device-independent system services.

Planned modules:

- `app_manager/` — application discovery, lifecycle, launch, suspend, resume, uninstall
- `display/` — virtual surfaces, dirty regions, refresh scheduling, ghosting policy
- `events/` — touch, buttons, timers, system events
- `network/` — Wi-Fi/BLE-facing service APIs
- `permissions/` — app capability and permission checks
- `power/` — deep sleep, wake sources, power locks
- `storage/` — internal storage and microSD service layer
- `update/` — OS OTA and rollback

Core code must not contain board-specific GPIO definitions or peripheral assumptions.
