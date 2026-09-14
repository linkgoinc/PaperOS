# PaperOS Power Management

Power management is a system responsibility in PaperOS.

An E-Ink device can retain content on screen without continuously powering the display, so the operating system should aggressively minimize active time while preserving a responsive user experience.

The design goal is:

> **Apps may request resources and wakeups, but PaperOS decides when hardware stays awake.**

## Goals

PaperOS power management should:

- preserve long battery life
- centralize control of Wi-Fi, BLE, display, sensors, and optional radios
- allow applications to request future wakeups
- survive deep sleep cleanly
- restore the previous user context after wake
- avoid one misbehaving app keeping the whole device awake
- expose battery-aware policies to the system UI

## Power States

Initial system states:

```text
ACTIVE
IDLE
SUSPENDING
DEEP_SLEEP
WAKING
```

### ACTIVE

The screen is interactive and at least one foreground app is active.

Typical resources:

```text
CPU        on
Touch      on
Display    awake
Wi-Fi      on only when needed
BLE        optional
Sensors    on-demand
```

### IDLE

The device is still interactive but no recent input has occurred.

Possible behavior:

- reduce CPU frequency
- stop unnecessary polling
- disconnect Wi-Fi after a grace period
- dim or disable frontlight where available
- prepare background state for sleep

### SUSPENDING

PaperOS coordinates services before deep sleep.

```text
notify foreground app
        ↓
persist app state
        ↓
flush filesystem
        ↓
stop network services
        ↓
power down optional peripherals
        ↓
sleep E-Ink controller if appropriate
        ↓
configure wake sources
```

### DEEP_SLEEP

ESP32-S3 enters deep sleep.

The E-Ink panel continues displaying the last image without active refresh power.

### WAKING

PaperOS restores the minimum required services, determines the wake reason, and resumes the relevant system or app context.

## Wake Sources

Supported wake sources depend on device capabilities.

Candidate wake sources:

```text
power button
physical buttons
touch interrupt
RTC alarm
ESP32 timer
IMU interrupt
charger / USB event
LoRa interrupt
NFC interrupt
```

Not every device needs to implement every wake source.

The HAL should expose available wake capabilities.

Example:

```text
power.hasWakeSource("touch")
power.hasWakeSource("rtc")
power.hasWakeSource("imu")
```

## Application Wake Requests

Apps should not directly configure ESP32 wake registers.

Instead they request a wake event from PaperOS.

Conceptual API:

```text
system.requestWakeAt(timestamp)
system.requestWakeAfter(seconds)
system.cancelWake(id)
```

PaperOS merges wake requests from all apps and system services.

Example:

```text
Weather requests 09:00
Calendar requests 08:55
System maintenance requests 03:00
```

PaperOS chooses the next required wake event and configures hardware accordingly.

## Background Execution

PaperOS should avoid an Android-style always-running background model.

Instead, background work should be event-driven and short-lived.

Recommended model:

```text
wake
 ↓
run background task
 ↓
update data / notification
 ↓
refresh display only if required
 ↓
sleep again
```

This keeps the system compatible with E-Ink and battery-powered usage.

## Network Policy

Wi-Fi is one of the highest-power subsystems on an ESP32-S3 device.

PaperOS should own connection lifetime.

Apps request networking through the system API:

```text
network.request()
network.httpGet()
network.httpPost()
```

The OS can then:

- reuse an existing connection
- batch requests from multiple apps
- disconnect after an idle timeout
- reject low-priority background networking at critically low battery

Apps should not independently initialize or destroy the Wi-Fi stack.

## BLE Policy

BLE should also be system-managed.

Potential modes:

```text
off
scan-on-demand
connected
advertising
```

PaperOS may suspend BLE during deep sleep unless the board supports a specific low-power wake strategy.

## Optional Radios

PaperMono may expose LoRa and NFC.

These should be represented as system-managed resources:

```text
radio.lora.acquire()
radio.lora.release()

nfc.acquire()
nfc.release()
```

The OS should power down or place these peripherals into low-power mode when not actively used.

## Display Power Policy

The E-Ink compositor and power manager must cooperate.

Before sleep:

```text
finish pending refresh
       ↓
wait for panel BUSY completion
       ↓
optionally enter panel deep sleep
       ↓
power down display rail if safe
```

On wake:

```text
restore display power
       ↓
initialize controller if needed
       ↓
decide whether framebuffer is still valid
       ↓
refresh only if necessary
```

Different E-Ink controllers may require different sequences, so the HAL must own panel-specific details.

## Frontlight Policy

Where supported, frontlight is a system capability.

Recommended behavior:

- off during deep sleep
- optional automatic timeout
- brightness restored after wake
- warm/cool channel state persisted by PaperOS
- applications may request a preferred level, but system settings remain authoritative

## Sensor Policy

Sensors should not remain active by default.

Examples:

```text
Temperature/Humidity → periodic sample
IMU                  → interrupt-driven wake where possible
Microphone           → only active with foreground permission
```

Continuous polling should be avoided unless explicitly required.

## App Lifecycle Before Sleep

Before entering deep sleep, the foreground app should receive an event such as:

```text
on_suspend(reason = "deep_sleep")
```

The app should save only its own logical state.

Example Reader state:

```text
book_id
chapter
page
scroll position
```

The app should not attempt to preserve hardware state.

Hardware state belongs to PaperOS and the HAL.

## System State Persistence

PaperOS should persist only the minimum state required to resume.

Candidate data:

```text
last foreground app
launcher page
app resume token
frontlight preference
pending notifications
scheduled wake metadata
```

State may live in:

- RTC memory for short-lived data
- NVS for durable system settings
- filesystem for larger app state

## Battery Service

The operating system should expose a normalized battery API independent of the underlying fuel-gauge IC.

Conceptual API:

```text
battery.percent()
battery.voltage()
battery.isCharging()
battery.timeEstimate()
```

Not all HAL implementations need to provide every metric.

## Battery Policies

Suggested policy tiers:

```text
Normal      > 20%
Low         10–20%
Critical    < 10%
```

Possible behavior:

### Low

- reduce background refresh frequency
- reduce network polling
- notify user

### Critical

- disable nonessential background apps
- disable frontlight automatically
- avoid unnecessary full E-Ink refreshes
- preserve app state and enter deep sleep sooner

These thresholds should remain configurable.

## Power Locks

Some operations temporarily require the system to remain awake.

Example API:

```text
PowerLock lock("firmware-update")
```

Valid use cases:

- OTA update
- filesystem operation
- active voice recording
- E-Ink refresh
- user-visible network transaction

Locks must be reference-counted and visible in diagnostics so a leaked lock can be identified.

## Diagnostics

PaperOS should expose useful developer metrics:

```text
current power state
wake reason
active power locks
Wi-Fi uptime
last sleep duration
battery percent
estimated current draw if available
```

A developer page can make power debugging significantly easier.

## Device-Specific HAL Responsibilities

The platform layer should implement:

```text
prepareForSleep()
configureWakeSources()
enterDeepSleep()
getWakeReason()
setPeripheralPower()
readBatteryState()
```

For reTerminal Sticky, this layer can use its documented battery, sensor, RTC, and peripheral controls.

For PaperMono, the implementation can additionally manage frontlight, LoRa, NFC, and its device-specific power rails.

## Initial Implementation Strategy

Version 0.1 should remain deliberately simple:

1. foreground-only app model
2. inactivity sleep timer
3. button/timer wake
4. system-owned Wi-Fi
5. battery percentage
6. app suspend callback
7. deep sleep / resume to launcher or last app

Only after real measurements should PaperOS add more complex background scheduling.

## Design Principle

A PaperOS app should never need to know how the ESP32 enters deep sleep.

The app says:

```text
I need to wake at 08:00.
```

PaperOS decides:

```text
how to store state
which RTC/timer to use
which peripherals to shut down
when to disconnect Wi-Fi
how to restore the system
```

That separation is essential to keep apps portable and battery-efficient.
