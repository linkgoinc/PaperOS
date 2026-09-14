# reTerminal Sticky Platform

This directory will contain the PaperOS Hardware Abstraction Layer implementation for Seeed Studio reTerminal Sticky.

Initial bring-up targets:

- display
- capacitive touch
- microSD
- Wi-Fi / BLE integration points
- buttons
- buzzer
- RTC
- IMU
- microphone
- temperature/humidity sensor
- battery/fuel gauge
- deep sleep and wake sources

The platform layer should expose capabilities through PaperOS interfaces instead of leaking board-specific GPIO or driver details into applications.
