# Contributing to PaperOS

Thanks for your interest in PaperOS.

PaperOS is currently in the architecture and hardware bring-up stage. Contributions are welcome, especially around reTerminal Sticky, M5Stack PaperMono, ESP-IDF, E-Ink rendering, low-power design, and application runtimes.

## Current priorities

1. reTerminal Sticky hardware bring-up
2. Common Hardware Abstraction Layer
3. E-Ink compositor and refresh policy
4. App lifecycle and launcher
5. Lua application runtime
6. Reader, Home Assistant, and Notes reference apps
7. PaperMono port

## Contribution areas

You can help with:

- Board support and peripheral drivers
- Display and touch integration
- Deep sleep and wake behavior
- Battery/fuel gauge support
- E-Ink partial refresh experiments
- App runtime design
- App SDK design
- Documentation
- Reference applications

## Design rules

Please keep these principles in mind:

- Applications should be installed, not flashed.
- Applications should not directly own hardware.
- Device-specific code belongs in the HAL/platform layer.
- E-Ink refresh should be managed centrally by the OS.
- Power management should remain a system responsibility.
- Optional hardware should be exposed through capabilities.
- Avoid assumptions that only work on one board.

## Proposed workflow

Before implementing a major architectural change, please open an issue describing:

- the problem
- the proposed approach
- affected layers
- memory/power implications
- whether the change remains portable across devices

Small fixes and documentation improvements can be submitted directly as pull requests.

## Code style

The exact formatting rules will be finalized once the first ESP-IDF skeleton is committed. Until then:

- prefer clear, small interfaces
- avoid global hardware ownership inside apps
- keep board-specific GPIO and driver code under `platform/`
- document memory ownership and lifecycle for long-lived services

## License

By contributing, you agree that your contributions will be licensed under the MIT License used by this repository.
