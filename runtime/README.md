# PaperOS Runtime

PaperOS is designed around installable applications rather than monolithic firmware.

The initial runtime plan has two tiers.

## Sandboxed runtime

A lightweight managed runtime, initially Lua, for most third-party/community applications.

Typical apps:

- Home Assistant
- Weather
- MQTT
- Calendar
- dashboards
- utilities

Sandboxed apps should only access APIs explicitly exposed by PaperOS.

## Trusted native runtime

A native loading path, potentially based on ESP32-S3 ELF loading, for trusted performance-sensitive applications.

Typical apps:

- Reader
- complex parsers
- image processing
- advanced NFC/LoRa tools

Native apps should still consume PaperOS system services rather than directly owning display, power, storage, or navigation.
