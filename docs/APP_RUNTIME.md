# PaperOS App Runtime

PaperOS needs an application runtime that allows apps to be installed, launched, suspended, updated, and removed independently from the base firmware.

The runtime is one of the core differences between PaperOS and a traditional embedded firmware project.

Traditional model:

```text
change app
   ↓
rebuild firmware
   ↓
flash device
```

PaperOS target:

```text
install app package
   ↓
register app
   ↓
launch at runtime
```

## Design Goals

The runtime should:

- support installable apps
- keep apps independent from board-specific GPIO and drivers
- support lightweight sandboxed apps
- support trusted native apps where performance matters
- allow apps to be unloaded to recover memory
- expose a stable PaperOS API
- work within ESP32-S3 memory constraints
- remain compatible across Sticky and PaperMono

## Proposed Runtime Model

PaperOS initially uses two execution tiers:

```text
App Runtime
├── Lua Runtime
│   └── sandboxed/community apps
└── Native ELF Runtime
    └── trusted/high-performance apps
```

This avoids forcing every app into either interpreted code or unrestricted native code.

## Tier 1: Lua Runtime

Lua is the preferred first runtime for community apps.

Good candidates:

- Home Assistant
- Weather
- MQTT
- Calendar
- Sensor dashboards
- Utilities
- Small games
- Device status tools

### Why Lua

Lua offers:

- small runtime footprint
- small application packages
- fast development iteration
- dynamic loading from filesystem
- simple C/C++ binding model
- controlled API exposure
- easier sandboxing than unrestricted native apps

Example app:

```lua
function on_open()
    ui.title("Weather")
end

function on_render()
    local weather = http.get("https://example/api")
    ui.text(20, 60, weather.temperature .. " °C")
end

function on_suspend()
    storage.set("last_open", time.now())
end
```

The application does not access ESP-IDF directly.

It only receives PaperOS APIs.

## Lua Sandbox

By default, community applications should not receive unrestricted access to:

- filesystem root
- raw sockets
- GPIO
- SPI
- I2C
- ESP-IDF system APIs
- flash partitions
- NVS namespaces belonging to other apps

Instead, PaperOS exposes selected modules.

Possible modules:

```text
ui
network
storage
system
device
sensor
notification
time
```

Optional modules become available only with permissions and compatible hardware:

```text
microphone
bluetooth
lora
nfc
```

## Example Lua API

Conceptual examples:

```lua
ui.text(20, 40, "Hello")
ui.button("Turn light on", on_click)

local r = network.http_get(url)

storage.write("settings.json", data)

if device.has("temperature") then
    local t = sensor.temperature()
end

system.request_wake_after(900)
```

The exact API should remain small in v0.1.

## Runtime Context

Each sandboxed app receives an isolated logical context.

Conceptually:

```text
AppContext
├── app ID
├── granted permissions
├── private storage path
├── runtime state
├── event queue
└── resource handles
```

This context is owned by the App Manager, not the app itself.

## App Lifecycle

Initial lifecycle:

```text
installed
    ↓
load
    ↓
on_open
    ↓
active
    ↓
on_suspend
    ↓
suspended / unloaded
    ↓
resume
    ↓
on_resume
```

Closing an app may result in complete runtime destruction.

This is useful on ESP32-S3 because memory can be reclaimed aggressively.

## Memory Strategy

PaperOS should not attempt to keep all installed applications resident.

Recommended model:

```text
Launcher resident
System services resident
Foreground app resident
Everything else stored on flash/SD
```

When switching apps:

```text
App A
 ↓
on_suspend()
 ↓
save logical state
 ↓
destroy runtime
 ↓
free heap / PSRAM
 ↓
load App B
```

This model trades small launch latency for predictable memory usage.

It is especially appropriate for E-Ink, where instant 60 FPS app switching is not required.

## Lua Memory Limits

PaperOS should eventually support per-app memory budgets.

Example concept:

```text
App heap limit: 512 KB
```

The exact limit should be based on real Sticky and PaperMono measurements.

A runaway Lua app should fail locally rather than exhausting the entire operating system if practical.

## Tier 2: Native ELF Runtime

Some applications need native performance or need to reuse existing C/C++ projects.

Examples:

- Reader / CrossPoint-derived engine
- EPUB parsing
- Image processing
- Complex font/layout code
- NFC protocol tools
- LoRa protocol tools

For these cases PaperOS can investigate ESP32-S3 ELF loading.

Conceptual flow:

```text
reader.papp
       ↓
reader.elf
       ↓
ELF Loader
       ↓
PaperOS Native ABI
       ↓
System services
```

## Native Apps Are Trusted

ESP32-S3 does not provide process isolation equivalent to Linux or Android.

Therefore native applications should initially be considered trusted code.

They should be used for:

- built-in apps
- officially reviewed apps
- performance-critical community apps after review

Untrusted third-party code should prefer the Lua runtime.

## Native ABI

Native apps should link against a narrow PaperOS SDK rather than arbitrary system internals.

Example exported symbols:

```text
paper_ui_draw_text()
paper_ui_invalidate()
paper_http_get()
paper_storage_open()
paper_device_has()
paper_app_exit()
paper_request_wake()
```

PaperOS should avoid exposing raw HAL objects where possible.

This reduces coupling and improves portability.

## ABI Versioning

Native applications require an ABI compatibility strategy.

The package manifest should eventually include something like:

```json
{
  "runtime": "elf",
  "abi": "paperos-1"
}
```

PaperOS can reject native apps built for an incompatible ABI.

## Reader as a Native App

The Reader is the best initial native-runtime test.

Proposed architecture:

```text
PaperOS
   ↓
Reader App
   ↓
CrossPoint-derived components
   ↓
PaperOS display/storage/input APIs
```

The long-term goal is not necessarily to run the entire existing CrossPoint firmware unchanged.

Instead, reusable parts can be adapted behind PaperOS APIs.

Likely reusable areas include:

- EPUB parsing
- library management
- typography
- page layout
- progress tracking

System ownership remains with PaperOS.

## Event Model

Apps should receive normalized events rather than polling hardware.

Candidate events:

```text
TouchDown
TouchUp
Tap
LongPress
ButtonPress
NetworkChanged
BatteryChanged
Timer
Resume
Suspend
CapabilityChanged
```

Example Lua model:

```lua
function on_event(event)
    if event.type == "tap" then
        -- handle input
    end
end
```

This allows the same app to work across different touch controllers and button mappings.

## UI Runtime

Apps should not draw directly to an E-Ink panel driver.

The runtime should expose UI primitives that render into the PaperOS virtual surface.

Example:

```text
app UI calls
    ↓
UI tree / canvas
    ↓
PaperOS compositor
    ↓
refresh scheduler
```

Initial UI primitives might include:

```text
text
image
line
rectangle
button
list
card
progress
```

A full widget toolkit is not required for v0.1.

## Permissions

Runtime access is constrained by the package manifest.

Example:

```json
{
  "permissions": [
    "network",
    "storage"
  ]
}
```

The runtime builds the API environment from granted permissions.

For example, an app without microphone permission should not receive a microphone API object at all.

## Capability Filtering

Runtime availability also depends on device capabilities.

Example:

```json
{
  "capabilities": ["lora"]
}
```

On Sticky:

```text
not compatible
```

On PaperMono:

```text
launchable
```

## App Crashes

A sandboxed app error should return the user safely to the launcher.

Example:

```text
Lua exception
    ↓
log error
    ↓
release app resources
    ↓
show crash message
    ↓
return Home
```

The system should avoid rebooting the entire device for normal script errors.

Native app crashes are harder to isolate and may still cause a system reset in early versions.

This is another reason to treat native apps as trusted.

## Watchdog Policy

Apps should not be allowed to block the system indefinitely.

Possible safeguards:

- event execution timeout
- runtime watchdog
- cooperative yield points
- network operation timeout
- memory quota

The exact design should remain simple initially.

## App Data

Apps should have a private data directory.

Example:

```text
/data/apps/org.paperos.weather/
```

The runtime maps app-relative storage calls into that directory.

Example:

```lua
storage.write("settings.json", data)
```

becomes internally:

```text
/data/apps/org.paperos.weather/settings.json
```

Apps should not see another app's private directory unless a future shared-data API explicitly permits it.

## Runtime Services

Some APIs are better implemented as shared asynchronous services.

Example:

```text
HTTP Service
Wi-Fi Service
Storage Service
Notification Service
Time Service
```

An app requests work and receives a completion event.

This reduces duplicated network stacks and centralizes power decisions.

## Background Tasks

Background execution should be intentionally limited.

PaperOS is not trying to emulate Android background processes.

Preferred model:

```text
scheduled event
    ↓
wake device
    ↓
load app/task
    ↓
perform short work
    ↓
persist result
    ↓
unload
    ↓
sleep
```

This model is much better suited to E-Ink and battery-powered ESP32 hardware.

## Runtime Registry

The App Manager should maintain a runtime registry.

Conceptually:

```text
RuntimeManager
├── LuaRuntime
└── ElfRuntime
```

App launch flow:

```text
read manifest
    ↓
runtime = "lua"
    ↓
RuntimeManager.get("lua")
    ↓
create context
    ↓
load entry point
    ↓
launch
```

This keeps the architecture open to future runtimes without changing the package system.

## Possible Future Runtimes

PaperOS may later evaluate:

```text
WASM
JavaScript
MicroPython
```

These should not be part of the initial implementation unless a concrete use case justifies the memory and maintenance cost.

## Developer Workflow

A lightweight developer experience is important.

Target flow:

```text
edit app
   ↓
package .papp
   ↓
upload to paperos.local
   ↓
launch
   ↓
inspect logs
```

No firmware rebuild should be needed for a normal Lua app.

## Logging

Each app should have tagged logs.

Example:

```text
[org.paperos.weather] request started
[org.paperos.weather] request completed
```

A developer console can expose recent logs over USB or the local web interface.

## Initial v0.1 Scope

The first runtime milestone should be deliberately narrow:

1. one foreground app at a time
2. Lua runtime only
3. basic UI API
4. private app storage
5. network API
6. lifecycle callbacks
7. install/uninstall from filesystem
8. safe return to launcher on Lua errors

Only after this works reliably should ELF/native app loading be added.

## Success Criteria

The runtime concept is proven when a user can:

```text
boot PaperOS
   ↓
install Home Assistant.papp
   ↓
launch it
   ↓
return Home
   ↓
launch Reader
   ↓
return Home
   ↓
uninstall Home Assistant
```

without reflashing or rebuilding the operating system.

That is the core application model PaperOS is trying to achieve.
