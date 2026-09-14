# PaperOS SDK

The SDK will define the stable application-facing API.

Candidate namespaces:

```text
ui.*
network.*
storage.*
sensor.*
device.*
system.*
```

Goals:

- hide board-specific hardware details
- keep app APIs small and stable
- expose optional hardware through capability checks
- support both sandboxed and trusted native applications
- document lifecycle, memory, and power expectations clearly

Example capability checks:

```text
device.has("frontlight")
device.has("lora")
device.has("nfc")
device.has("microphone")
device.has("temperature")
```
