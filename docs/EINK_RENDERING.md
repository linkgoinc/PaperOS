# PaperOS E-Ink Rendering Model

PaperOS is designed around E-Ink as a first-class display technology.

The operating system should not treat an E-Ink panel like a conventional LCD. Refresh latency, ghosting, waveform behavior, and power characteristics require a different rendering model.

## Core Principle

Applications should draw **what they want to display**, but they should not directly control **how the physical E-Ink panel refreshes**.

```text
Application
    ↓
Virtual UI Surface
    ↓
Dirty Region Tracking
    ↓
E-Ink Compositor
    ↓
Refresh Scheduler
    ↓
Panel Driver
```

## Why Centralized Refresh Matters

If each app can independently trigger panel refreshes, several problems appear:

- excessive ghosting
- unnecessary full refreshes
- inconsistent visual behavior
- poor battery life
- app transitions that flash unpredictably
- hardware-specific code leaking into applications

PaperOS therefore owns physical refresh policy at the system layer.

## Virtual Surface

Applications render into a virtual framebuffer or drawing surface exposed through the PaperOS UI API.

Examples:

```text
ui.text(...)
ui.image(...)
ui.button(...)
ui.rectangle(...)
```

The app should not call low-level panel methods such as:

```text
refresh()
partialRefresh()
setWaveform()
```

Those operations belong to the display service.

## Dirty Region Tracking

The compositor tracks regions changed since the previous committed frame.

Example:

```text
Before:
┌─────────────────────┐
│ Temperature 24.1 °C │
│ Door        CLOSED  │
└─────────────────────┘

After:
┌─────────────────────┐
│ Temperature 24.2 °C │
│ Door        CLOSED  │
└─────────────────────┘
```

Only the temperature region is dirty.

The compositor may merge nearby dirty rectangles to reduce panel transactions.

## Refresh Modes

PaperOS should expose panel capabilities internally rather than hard-coding one policy.

Potential modes:

```text
FULL
PARTIAL
FAST_PARTIAL
GRAYSCALE
CLEAN
```

Not every panel or controller will support every mode.

The HAL should advertise supported refresh capabilities.

Example:

```text
display.supportsPartialRefresh = true
display.supportsGrayscale = true
display.partialAlignmentX = 8
```

## Refresh Decision Policy

A first implementation can use simple rules.

Example logic:

```text
if app_changed:
    full refresh
else if dirty_area > threshold:
    full refresh
else if partial_count > limit:
    full refresh
else:
    partial refresh
```

Future policies can become more sophisticated based on panel behavior.

## Ghosting Management

PaperOS should maintain a ghosting/partial-refresh budget.

Example state:

```text
partial_refresh_count = 7
partial_area_accumulator = 42%
last_full_refresh = 3 min ago
```

A full refresh may be scheduled after:

- N partial updates
- excessive accumulated changed area
- app transitions
- long periods of repeated fast updates
- visible ghosting heuristics

The exact thresholds should remain panel-specific and configurable.

## App Transitions

Changing apps is a special rendering event.

Initial policy proposal:

```text
same app, small update
→ partial refresh

same app, major layout change
→ partial or full depending on dirty area

app A → Home
→ full refresh

Home → app B
→ full refresh
```

Later, transitions may use optimized partial refresh where safe.

## Refresh Coalescing

Apps may issue several UI changes in a short period.

PaperOS should avoid refreshing after every draw call.

Instead:

```text
app updates widget A
app updates widget B
app updates widget C
      ↓
frame commit
      ↓
one refresh transaction
```

Conceptual API:

```text
ui.beginFrame()
...
ui.endFrame()
```

or an implicit retained-mode UI model.

## E-Ink Event Rate

The OS should discourage high-frequency rendering.

Example application policies:

```text
Clock seconds animation → not recommended
Weather every 10 min    → good
Home Assistant state    → refresh on change
Reader page turn        → immediate
```

Background apps should not cause frequent refresh unless explicitly allowed.

## Grayscale

If the panel supports multiple grayscale levels, the compositor should expose a device-independent grayscale abstraction.

Example:

```text
0 = white
1 = light gray
2 = dark gray
3 = black
```

Apps should not assume a particular panel waveform implementation.

Where grayscale is unavailable, PaperOS may quantize content to black/white.

## Image Rendering

Image rendering may require:

- scaling
- dithering
- grayscale conversion
- clipping
- rotation

These operations should be provided by shared system utilities so each app does not implement its own E-Ink conversion pipeline.

## Rotation

The logical UI coordinate system should be independent from physical panel orientation.

Example:

```text
logical 800 × 480
       ↓
rotation transform
       ↓
physical panel
```

Rotation may be driven by:

- device configuration
- app preference
- IMU orientation

The compositor remains responsible for converting dirty regions correctly.

## Sleep / Wake

E-Ink retains its image without continuous power.

PaperOS should take advantage of this.

Before sleep:

```text
commit final UI state
      ↓
wait panel idle
      ↓
panel sleep
      ↓
ESP32 deep sleep
```

On wake, PaperOS should decide whether the retained image can be reused or requires refresh.

Possible inputs:

- panel power state
- wake duration
- framebuffer persistence
- active app state

## Framebuffer Strategy

ESP32-S3 devices with PSRAM make a full logical framebuffer practical for many E-Ink panels.

However PaperOS should avoid assuming unlimited memory.

Potential strategies:

1. full framebuffer in PSRAM
2. monochrome framebuffer plus temporary grayscale buffers
3. tiled rendering
4. direct rendering for constrained targets

The display service should isolate apps from the selected implementation.

## Panel Driver Boundary

The compositor should communicate with a panel-neutral HAL.

Conceptual interface:

```text
DisplayHAL::width()
DisplayHAL::height()
DisplayHAL::capabilities()
DisplayHAL::refresh(region, mode)
DisplayHAL::fullRefresh()
DisplayHAL::sleep()
DisplayHAL::wake()
DisplayHAL::isBusy()
```

Panel-specific waveform and controller details stay below this boundary.

## Metrics and Debugging

Development builds should expose useful metrics:

```text
partial refresh count
full refresh count
average refresh time
last dirty area
refresh mode
panel busy duration
ghosting budget
```

A developer overlay would make tuning much easier.

## Initial Milestone

For the first reTerminal Sticky implementation, success means:

1. render a simple launcher
2. track dirty regions
3. perform partial refresh for small widget changes
4. force full refresh during app transitions
5. enter display sleep correctly
6. restore UI after device wake

Advanced waveform optimization can follow after stable hardware validation.

## Design Principle

> Applications describe pixels. PaperOS decides how E-Ink should physically refresh them.
