# PaperOS

> **Install apps, not firmware.**

PaperOS is an open-source application platform for ESP32-S3 E-Ink devices.

The project explores a simple idea: modern E-Ink hardware such as the **Seeed Studio reTerminal Sticky** and **M5Stack PaperMono** is capable of much more than running one dedicated firmware at a time.

Today, switching from an e-reader to a Home Assistant dashboard or another tool usually means flashing a different firmware image. PaperOS aims to replace that model with a persistent system that can install, launch, update, and remove applications without reflashing the device.

```text
Home
├── Reader
├── Home Assistant
├── Notes
├── Weather
├── MQTT
└── Settings
```

## Why PaperOS?

Most embedded E-Ink projects follow this model:

```text
one device
    ↓
one firmware
    ↓
one main use case
```

Examples:

```text
CrossPoint → E-reader
ESPHome    → Home automation
TRMNL      → Dashboard
Custom FW  → One dedicated application
```

That model works well for single-purpose products, but it underuses modern ESP32-S3 hardware and creates unnecessary friction for both developers and users.

PaperOS proposes:

```text
one device
    ↓
one persistent OS
    ↓
installable apps
    ↓
no reflashing for normal app use
```

The long-term goal is a lightweight embedded experience closer to iOS or Android, while remaining optimized for E-Ink, low power, and microcontroller-class hardware.

## First Target Devices

### Seeed Studio reTerminal Sticky

Sticky is planned as the first reference platform because it combines:

- ESP32-S3
- PSRAM
- E-Ink display
- Capacitive touch
- Wi-Fi / BLE
- microSD
- Battery management
- RTC
- IMU
- Microphone
- Temperature/humidity sensor
- Buttons and buzzer
- Developer-friendly ESP-IDF support

### M5Stack PaperMono

PaperMono is the second planned target and adds capabilities such as:

- Frontlight
- NFC
- LoRa
- Larger battery

Supporting two different devices from the beginning helps ensure that PaperOS is a portable application platform rather than a firmware tied to one board.

## Core Architecture

```text
Applications
    ↓
App Runtime
    ↓
System API
    ↓
System Services
    ↓
HAL
    ↓
Sticky / PaperMono
```

The operating system owns shared resources such as display refresh, touch routing, networking, storage, power, application lifecycle, permissions, and updates.

Applications consume stable system APIs rather than directly controlling board-specific hardware.

## Application Model

A PaperOS application should be installable independently from the OS.

PaperOS application packages use the **`.papp`** extension (**PaperOS Application Package**).

Example package:

```text
home-assistant.papp
├── manifest.json
├── app/
│   └── main.lua
├── icon.png
└── resources/
```

Example manifest:

```json
{
  "id": "org.paperos.homeassistant",
  "name": "Home Assistant",
  "version": "0.1.0",
  "runtime": "lua",
  "entry": "app/main.lua",
  "min_os": "0.1.0",
  "permissions": ["network", "storage"]
}
```

## Two Application Tiers

### Sandboxed Apps

Most community applications should run in a lightweight managed runtime such as Lua.

Good candidates include Home Assistant, Weather, MQTT, Calendar, sensor dashboards, utilities, and simple games.

### Trusted Native Apps

Native applications are intended for performance-sensitive or complex workloads such as EPUB rendering, advanced parsers, image processing, NFC tools, and LoRa tools.

A future Reader app could reuse or adapt CrossPoint components while running as an application rather than owning the entire firmware.

## CrossPoint as an App, Not the OS

CrossPoint is an excellent E-Ink reader project. PaperOS does not aim to replace its reading engine.

The preferred model is:

```text
PaperOS
├── Reader
│   └── CrossPoint-derived reader engine
├── Home Assistant
├── Notes
└── Weather
```

This keeps reading functionality focused while allowing the same device to serve multiple roles.

## Hardware Abstraction

Applications should not know which device they are running on.

```text
Application
     ↓
System API
     ↓
Hardware Abstraction Layer
     ↓
Device
```

Applications can query optional capabilities such as frontlight, LoRa, NFC, microphone, or temperature sensors.

## E-Ink-First Rendering

PaperOS should not treat E-Ink like a normal LCD.

```text
App
 ↓
Virtual UI
 ↓
Dirty Region Tracking
 ↓
E-Ink Refresh Scheduler
 ↓
Partial / Full Refresh Decision
 ↓
Display
```

The OS should manage dirty rectangles, partial refresh, full refresh, refresh batching, ghosting control, panel-specific behavior, display sleep/wake, and application transition policy.

Applications should ideally never trigger physical display refresh directly.

## Power Management

Power management belongs to the system, not individual apps.

Apps may request timers, background work, networking, or RTC wake events, but PaperOS decides when the device can enter deep sleep.

## App Installation

PaperOS should eventually support three installation paths:

1. On-device app repository
2. microSD package installation
3. Local web installer

Normal application installation should never require USB flashing.

## Initial Applications

The first three real apps are planned to be:

1. Reader
2. Home Assistant
3. Notes

These intentionally test storage/rendering, networking/touch, and input/storage/microphone.

## Development Roadmap

### Phase 0 — Sticky Hardware Bring-Up

- ESP-IDF baseline
- E-Ink
- Touch
- microSD
- Wi-Fi
- Battery status
- Deep sleep / wake

### Phase 1 — Core System

- App manager
- System navigation
- Display ownership
- Touch event dispatch
- Storage service
- Wi-Fi service
- Settings
- Power manager

### Phase 2 — First Applications

- Reader
- Home Assistant
- Notes

### Phase 3 — Installable Apps

- Package format
- Manifest
- Lua runtime
- App installation/removal
- microSD installer
- Local web installer

### Phase 4 — Native App Loading

Evaluate ESP32-S3 ELF loading for trusted high-performance applications.

### Phase 5 — PaperMono Port

Add PaperMono HAL and optional capabilities such as frontlight, NFC, and LoRa.

### Phase 6 — App Repository

Start with a simple static repository, potentially backed by GitHub Releases.

## Open Source

PaperOS is intended to use the **MIT License**.

Planned open-source scope includes the OS core, HAL, app SDK, app runtime, E-Ink compositor, reference apps, and documentation.

## Why This Could Be Useful for Seeed

PaperOS could allow reTerminal Sticky to serve multiple communities with one hardware platform: e-reader users, Home Assistant users, IoT users, makers, AI/voice app developers, dashboard users, and embedded developers.

Instead of requiring a different firmware for every use case, the community could build applications on top of one shared system.

## Discussion Topics for Seeed

Feedback from the Seeed engineering team would be especially valuable around:

1. Recommended ESP-IDF version for Sticky
2. Display driver and waveform constraints
3. Partial refresh recommendations
4. Touch wake support
5. Deep-sleep best practices
6. Battery/fuel-gauge integration
7. Recommended peripheral power-control sequence
8. Compatibility with the Sticky Playground ecosystem
9. Existing Seeed work related to modular firmware or application runtimes
10. Access to early hardware for development and validation

## Project Principle

> **Applications should be installed, not flashed.**

```text
Home
├── Reader
├── Home Assistant
├── Notes
├── Weather
└── App Store
```

One OS. Multiple devices. Installable applications. No reflashing for normal use.
