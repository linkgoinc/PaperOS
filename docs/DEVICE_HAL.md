# PaperOS Device HAL

PaperOS is intended to support multiple ESP32-S3 E-Ink devices without forcing applications to know board-specific GPIO assignments, display controllers, touch chips, battery ICs, or optional peripherals.

The Hardware Abstraction Layer (HAL) is the boundary between the PaperOS system core and each supported device.

## Goal

Applications should use stable system APIs:

```text
ui.*
network.*
storage.*
sensor.*
device.*
system.*
```

They should not need to know whether they are running on:

```text
reTerminal Sticky
M5Stack PaperMono
future ESP32-S3 E-Ink hardware
```

## Architecture

```text
Applications
    ↓
PaperOS System API
    ↓
System Services
    ↓
Common HAL Interfaces
    ↓
Board Implementation
    ↓
Physical Hardware
```

The HAL should isolate board-specific details while exposing capabilities and normalized behavior to the rest of PaperOS.

## Proposed Platform Layout

```text
platform/
├── common/
│   ├── display_hal.h
│   ├── touch_hal.h
│   ├── power_hal.h
│   ├── storage_hal.h
│   ├── sensor_hal.h
│   ├── audio_hal.h
│   └── device_caps.h
│
├── sticky/
│   ├── board.cpp
│   ├── display.cpp
│   ├── touch.cpp
│   ├── power.cpp
│   ├── storage.cpp
│   ├── sensors.cpp
│   └── audio.cpp
│
└── papermono/
    ├── board.cpp
    ├── display.cpp
    ├── touch.cpp
    ├── power.cpp
    ├── storage.cpp
    ├── sensors.cpp
    ├── lora.cpp
    ├── nfc.cpp
    └── audio.cpp
```

## Device Capabilities

PaperOS should expose optional hardware through a capability model rather than assumptions.

Possible capability identifiers:

```text
DISPLAY_EINK
TOUCH
WIFI
BLE
MICROSD
IMU
RTC
MICROPHONE
BUZZER
FRONTLIGHT
TEMPERATURE
HUMIDITY
LORA
NFC
RGB_LED
```

Conceptual API:

```cpp
if (device.hasCapability(Capability::LORA)) {
    // enable LoRa app/features
}
```

or at runtime:

```text
device.has("lora")
```

## Capability Examples

### reTerminal Sticky

Expected capabilities:

```text
E-Ink
Touch
Wi-Fi
BLE
microSD
IMU
RTC
Microphone
Temperature
Humidity
Buzzer
Battery Gauge
```

### M5Stack PaperMono

Expected capabilities:

```text
E-Ink
Touch
Wi-Fi
BLE
microSD
IMU
RTC
Microphone
Buzzer
Frontlight
LoRa
NFC
Battery Gauge
```

Exact board mappings should be validated against vendor documentation and hardware during bring-up.

## Board Identity

Each HAL implementation should provide board metadata.

Conceptual structure:

```cpp
struct DeviceInfo {
    const char* id;
    const char* vendor;
    const char* model;
    const char* revision;
};
```

Examples:

```text
seeed.reterminal.sticky
m5stack.papermono
```

This identity is useful for diagnostics and compatibility but applications should prefer capabilities instead of model checks.

Avoid:

```cpp
if (device.model == "PaperMono") {
    ...
}
```

Prefer:

```cpp
if (device.hasCapability(Capability::FRONTLIGHT)) {
    ...
}
```

## Display HAL

The display HAL should hide panel controller and waveform implementation details.

Conceptual interface:

```cpp
class DisplayHAL {
public:
    virtual int width() const = 0;
    virtual int height() const = 0;
    virtual DisplayCapabilities capabilities() const = 0;

    virtual bool wake() = 0;
    virtual bool sleep() = 0;

    virtual bool refresh(
        const Rect& region,
        RefreshMode mode
    ) = 0;

    virtual bool fullRefresh() = 0;
    virtual bool isBusy() const = 0;
};
```

The PaperOS E-Ink compositor sits above this interface.

## Touch HAL

Touch should be normalized into system input events.

Conceptual events:

```text
TOUCH_DOWN
TOUCH_MOVE
TOUCH_UP
TAP
LONG_PRESS
SWIPE
```

Raw controller coordinates should be transformed to logical display orientation before reaching applications.

Conceptual interface:

```cpp
class TouchHAL {
public:
    virtual bool begin() = 0;
    virtual bool read(TouchSample& sample) = 0;
    virtual bool canWakeDevice() const = 0;
};
```

## Power HAL

Power management is especially important for E-Ink devices.

The HAL should expose low-level board functionality while the PaperOS Power Manager owns policy.

Possible interface:

```cpp
class PowerHAL {
public:
    virtual int batteryPercent() = 0;
    virtual int batteryMillivolts() = 0;
    virtual bool isCharging() = 0;

    virtual void prepareForDeepSleep() = 0;
    virtual void configureWakeSources() = 0;
};
```

Board code may need to disable peripherals or power rails in a specific sequence before deep sleep.

That sequence belongs in the board HAL rather than apps.

## Frontlight

Frontlight should be an optional capability.

Conceptual API exposed to the system:

```text
frontlight.available()
frontlight.setBrightness(0..100)
frontlight.setTemperature(...)
```

If a device has no frontlight, the capability is absent.

Apps should degrade gracefully.

## Storage HAL

microSD and internal flash should appear through the PaperOS Storage Service.

The HAL is responsible for board-specific initialization.

The app-facing storage API should not expose raw SD card drivers.

Potential logical namespaces:

```text
/system
/apps
/data/apps/<app-id>
/shared
```

## Sensor HAL

Sensor data should use normalized units.

Examples:

```text
temperature → degrees Celsius
humidity    → percent RH
acceleration → m/s² or normalized SI units
rotation     → standard orientation representation
```

A sensor capability may expose metadata such as sampling limits and wake support.

## IMU

IMU support can provide both raw samples and system-level gestures.

Possible higher-level events:

```text
SHAKE
FLIP
ROTATE
RAISE
TILT_LEFT
TILT_RIGHT
```

Gesture interpretation may belong in a shared service above the raw IMU HAL so behavior remains consistent across devices.

## Audio Input

Microphone hardware should be abstracted through an audio capture API.

Possible app-facing model:

```text
microphone.request()
microphone.start(...)
microphone.read(...)
microphone.stop()
```

The HAL handles device-specific PDM/I2S configuration.

Microphone access should require an explicit app permission.

## Buzzer / Simple Audio Output

A buzzer is not equivalent to a speaker.

PaperOS should expose a simple notification-tone API where appropriate:

```text
system.beep()
system.tone(frequency, duration)
```

Devices without supported audio output simply omit the capability.

## LoRa HAL

PaperMono provides LoRa as an optional device capability.

The first HAL should expose radio primitives rather than force a specific protocol such as LoRaWAN or Meshtastic.

Conceptual operations:

```text
lora.configure(...)
lora.send(...)
lora.receive(...)
lora.rssi()
lora.snr()
```

Higher-level protocols can be implemented as apps or services.

Radio region/frequency constraints must be respected by application and platform policy.

## NFC HAL

NFC should similarly expose useful common operations without coupling apps to one controller.

Potential primitives:

```text
nfc.startDiscovery()
nfc.stopDiscovery()
nfc.readTag()
```

Advanced controller-specific features can be added later if portable abstractions prove insufficient.

## RTC and Wake Sources

RTC support should be exposed to PaperOS scheduling and power services.

The system should be able to schedule a future wake without applications directly programming RTC registers.

Conceptual flow:

```text
App requests wake at 07:00
        ↓
Scheduler
        ↓
Power Manager
        ↓
RTC HAL
        ↓
deep sleep
```

## Event Model

HAL implementations should feed a shared event bus rather than calling application code directly.

Example:

```text
Touch HAL ───────┐
Buttons HAL ─────┤
RTC HAL ─────────┤
IMU Service ─────┤ → System Event Bus → Active App
Battery HAL ─────┤
Network Service ─┘
```

This keeps hardware and app lifecycle decoupled.

## Error Handling

HAL functions should return explicit status values.

PaperOS should be able to distinguish:

```text
NOT_SUPPORTED
NOT_INITIALIZED
BUSY
TIMEOUT
IO_ERROR
INVALID_ARGUMENT
```

Silent failures should be avoided, especially in display and power code.

## Diagnostics

Each platform implementation should provide a device diagnostics screen or structured report.

Useful fields:

```text
board id
hardware revision
ESP-IDF version
free internal RAM
free PSRAM
flash size
SD status
battery percentage
battery voltage
display dimensions
display capabilities
touch status
available capabilities
```

This will be valuable when supporting multiple hardware revisions.

## Hardware Revision Handling

A board model may change components across production revisions.

PaperOS should avoid assuming that one product name always means one exact panel/controller.

Possible strategies:

- compile-time board revision targets
- runtime controller detection
- vendor-provided revision identifiers
- capability probing

Where possible, runtime detection is preferable.

## Bring-Up Checklist

For each new device, the initial HAL validation should cover:

1. boot
2. board identity
3. display full refresh
4. display partial refresh
5. touch calibration/orientation
6. microSD
7. Wi-Fi
8. battery/fuel gauge
9. buttons
10. RTC
11. deep sleep
12. wake source
13. optional sensors
14. optional radio/NFC/frontlight features

## Porting a New Device

The desired long-term process is:

```text
add platform/<device>/
        ↓
implement required HAL interfaces
        ↓
declare capabilities
        ↓
run PaperOS HAL tests
        ↓
existing launcher/apps work
```

A new board should not require changes to Reader, Home Assistant, Notes, Weather, or other portable apps unless it introduces a genuinely new capability.

## Design Principle

> Board-specific implementation belongs below the HAL. Applications should depend on capabilities and stable PaperOS APIs, not GPIO numbers or chip names.
