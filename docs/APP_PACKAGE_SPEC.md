# PaperOS App Package Specification

This document defines the initial package model for installable PaperOS applications.

The design goal is simple:

> **Applications should be installed, not flashed.**

A PaperOS app package should be self-contained, versioned, portable across supported devices, and installable without rebuilding the operating system.

## Package Extension

Proposed extension:

```text
.paperapp
```

Example:

```text
home-assistant.paperapp
```

The package can initially use a ZIP-compatible container format.

## Package Layout

A sandboxed application might look like:

```text
home-assistant.paperapp
├── manifest.json
├── app/
│   └── main.lua
├── icon.png
└── resources/
```

A native application might look like:

```text
reader.paperapp
├── manifest.json
├── app/
│   └── reader.elf
├── icon.png
├── fonts/
└── resources/
```

## Manifest

Every package must contain `manifest.json` at the package root.

Example:

```json
{
  "id": "org.paperos.homeassistant",
  "name": "Home Assistant",
  "version": "0.1.0",
  "runtime": "lua",
  "entry": "app/main.lua",
  "min_os": "0.1.0",
  "permissions": [
    "network",
    "storage"
  ],
  "capabilities": [],
  "background": false
}
```

## Required Fields

### `id`

Globally unique application identifier.

Recommended format:

```text
reverse.domain.appname
```

Example:

```text
org.paperos.weather
```

### `name`

Human-readable application name.

### `version`

Semantic version string.

Example:

```text
1.2.0
```

### `runtime`

Initial runtime values:

```text
lua
elf
```

Possible future runtimes:

```text
wasm
javascript
```

### `entry`

Path to the application entry point inside the package.

Examples:

```text
app/main.lua
app/reader.elf
```

### `min_os`

Minimum compatible PaperOS version.

## Permissions

Apps should explicitly declare access to system resources.

Initial permission candidates:

```text
network
storage
microphone
bluetooth
sensors
notifications
background
lora
nfc
```

Permissions are granted through PaperOS system APIs. Apps should not directly access board-specific drivers or GPIO.

Example installation screen:

```text
Home Assistant requires:

✓ Network
✓ Local Storage

[ Install ]
```

## Hardware Capabilities

Apps may require optional device hardware.

Example:

```json
{
  "capabilities": [
    "lora"
  ]
}
```

If the current device does not expose the required capability, PaperOS should mark the application as incompatible rather than installing it.

Candidate capabilities include:

```text
frontlight
lora
nfc
microphone
temperature
humidity
imu
rtc
```

Capabilities and permissions are different concepts:

- **Capability**: whether the hardware exists.
- **Permission**: whether the app may use a resource.

## Installation Flow

Proposed flow:

```text
download / discover package
        ↓
read manifest
        ↓
validate package
        ↓
check OS version
        ↓
check hardware capabilities
        ↓
show requested permissions
        ↓
install files
        ↓
register application
        ↓
show icon in launcher
```

## Storage Layout

A possible installed application layout:

```text
/apps/
├── org.paperos.reader/
│   ├── manifest.json
│   ├── app/
│   └── resources/
├── org.paperos.homeassistant/
└── org.paperos.weather/
```

Application-private data should be stored separately from application binaries.

Example:

```text
/data/apps/org.paperos.homeassistant/
```

This allows applications to be upgraded without deleting user data.

## Application Lifecycle

Initial lifecycle states:

```text
installed
    ↓
launching
    ↓
active
    ↓
suspended
    ↓
active
    ↓
closing
```

Applications should receive lifecycle callbacks where supported.

Conceptual API:

```text
on_install()
on_open()
on_suspend()
on_resume()
on_close()
on_uninstall()
```

Before deep sleep, PaperOS may request applications to persist state.

## Updates

Applications should update independently of the operating system.

```text
Weather 1.0
   ↓
download 1.1 package
   ↓
validate
   ↓
replace application files
   ↓
preserve app data
   ↓
Weather 1.1
```

Rollback support can be considered later for critical applications.

## Package Integrity

Initial implementation may use:

- package size validation
- SHA-256 digest
- manifest validation

A later public app repository can add signed packages.

Possible model:

```text
package
   ↓
SHA-256
   ↓
signature
   ↓
trusted repository key
```

The signing model should remain optional for local developer packages.

## Developer Installation

PaperOS should support local development without an app store.

Recommended developer paths:

1. microSD package install
2. local web upload
3. USB development tooling

Example:

```text
paperos.local
    ↓
Developer
    ↓
Upload .paperapp
```

## Runtime Notes

### Lua

Lua is the preferred initial runtime for third-party/community apps because it provides:

- small package sizes
- fast development iteration
- controlled API exposure
- easier isolation than native code

### ELF

ELF is intended for trusted, performance-sensitive applications.

Examples:

- Reader
- complex parsers
- image processing

Native apps should link against the PaperOS SDK rather than directly taking ownership of hardware.

## Open Questions

The following remain intentionally undecided for early prototyping:

- final archive/container format
- package compression
- signature format
- dependency declarations
- shared libraries
- background execution policy
- application memory quotas
- runtime ABI compatibility strategy

These should be validated against real ESP32-S3 memory and storage constraints before the format is frozen.

## Design Principle

The package system should make this possible:

```text
Store
 ↓
Home Assistant
 ↓
Install
 ↓
Home
├── Reader
├── Home Assistant
└── Notes
```

without rebuilding or reflashing PaperOS.
