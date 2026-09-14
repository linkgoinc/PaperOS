# PaperOS Architecture

This document describes the initial technical direction for PaperOS.

## Layers

```text
Applications
    ↓
App Runtime
    ↓
PaperOS System API
    ↓
System Services
    ↓
Hardware Abstraction Layer
    ↓
Device
```

## Responsibilities

### Applications

User-facing functionality such as Reader, Home Assistant, Notes, Weather, and device-specific tools.

### App Runtime

Executes installable applications. Initial direction:

- Lua for sandboxed/community applications
- Native ELF for trusted high-performance applications

### System API

Stable boundary exposed to applications.

Candidate APIs:

```text
ui.*
network.*
storage.*
sensor.*
device.*
system.*
```

### System Services

Shared system ownership of:

- App lifecycle
- Display refresh
- Touch routing
- Wi-Fi / BLE
- Storage
- Power
- Permissions
- OTA updates
- Notifications

### HAL

Board-specific implementation for:

- Seeed Studio reTerminal Sticky
- M5Stack PaperMono
- Future ESP32-S3 E-Ink devices

## E-Ink Compositor

The compositor is a core PaperOS component.

Apps should render into an abstract UI surface. The compositor owns physical E-Ink updates and decides when to use partial or full refresh.

Future considerations:

- dirty-region merging
- refresh coalescing
- ghosting counters
- per-panel capabilities
- transition policies
- sleep/wake state restoration

## App Lifecycle

Proposed lifecycle:

```text
install
→ register
→ launch
→ active
→ suspend
→ resume
→ close
→ uninstall
```

Apps should be able to persist state before suspension or deep sleep.

## Capability Model

Optional device capabilities are exposed through the system rather than assumed by apps.

Examples:

```text
frontlight
lora
nfc
microphone
temperature
imu
rtc
```

This allows one app package to run across multiple boards while adapting to available hardware.

## Initial Runtime Strategy

PaperOS should support two application classes:

### Sandboxed Apps

A lightweight managed runtime such as Lua is the preferred initial path for community apps. This makes installation simple and limits direct hardware access.

### Trusted Native Apps

Performance-sensitive apps may use native ESP32-S3 code loaded separately from the core OS. ELF loading is one candidate mechanism to evaluate.

## Hardware Abstraction

The HAL should present a stable capability-oriented interface rather than exposing board GPIOs to applications.

Example conceptual calls:

```text
device.has("lora")
device.has("frontlight")
display.drawText(...)
storage.open(...)
network.httpGet(...)
system.requestWake(...)
```

Board-specific implementations live under `platform/`.

## Initial Targets

### reTerminal Sticky

First reference implementation and primary bring-up target.

### M5Stack PaperMono

Second target used to validate portability and add optional features such as LoRa, NFC, and frontlight.
