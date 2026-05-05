# Tesla CAN Mod Guide

## Project Overview

This workspace is a goal-oriented install and research guide for a 2024 Model 3 Highland HW4 using X179 CAN access. The canonical target configuration is [`install-target.yaml`](./install-target.yaml); read it before changing firmware, build flags, wiring assumptions, or rollout docs.

Important current state:

- Selected physical install hardware is **Adafruit Feather RP2040 CAN (MCP2515)**.
- Desired firmware behavior is **hypery11 CAN logic**.
- hypery11 currently supports **Flipper Zero** and **ESP32** targets, not Feather RP2040. Do not assume a hypery11 Feather `firmware.uf2` exists.
- The target vehicle bus is **X179 pin 13/14 / Bus 6 mixed forwarding**. The restored image labels pin 13/14 as Chassis CAN; hypery11 labels it Bus 6 mixed forwarding. Verify by frame visibility, not by Body/Chassis naming alone.

Project goals, in rollout order:

1. **Nag suppression**: first active feature after listen-only validation.
2. **FSD region unlock**: unlock the regional UI gate for this VIN's paid permanent FSD entitlement in Taiwan.
3. **Telemetry disable evaluation**: research only by default; `0x3F8` bit 43 remains disabled for normal use unless explicitly testing offline.

The workspace has four main components:

1. **Root Directory (`/`)**: Hardware wiring, install guide, progress tracking, and risk decisions for the Model 3 Highland.
2. **`flipper-tesla-fsd/`**: hypery11 reference firmware and research source for nag suppression, AP-First, Ban Shield, FSD region-unlock behavior, and telemetry-disable research.
3. **`ev-open-can-tools/`**: Reference C++ firmware project for CAN handlers, tests, drivers, and plugin architecture. Useful for comparison, but not the default firmware target for this install plan.
4. **`ev-open-can-tools-plugins/`**: Reference JSON plugins that define CAN frame modifications.

**Core Technologies:**
- **Language**: C/C++ (Arduino Framework)
- **Build System**: PlatformIO
- **Microcontrollers**: ESP32 variants, RP2040, SAMD51
- **CAN Interfaces**: TWAI (ESP32 native), MCP2515 (SPI)
- **Key Libraries**: ArduinoJson

## Building and Running

The desired install target is hypery11 behavior on the chosen hardware. Current supported choices are Flipper Zero + CAN add-on or ESP32 + CAN hardware. Feather RP2040 requires a port before it can carry hypery11-derived behavior. If the work is about the actual vehicle install, do not silently switch to stock `ev-open-can-tools` firmware; first confirm that fallback with the user.

The `ev-open-can-tools/` commands below are reference commands for that upstream project only.

**PlatformIO Commands:**

*   **Build a specific environment:**
    ```bash
    cd ev-open-can-tools
    pio run -e <environment_name>
    # Example: pio run -e esp32_ext_mcp2515
    # Example: pio run -e feather_rp2040_can
    ```
*   **Build and upload (flash) to a connected board:**
    ```bash
    cd ev-open-can-tools
    pio run -e <environment_name> -t upload
    ```
*   **Run Native Unit Tests:**
    ```bash
    cd ev-open-can-tools
    pio test -e native
    # To run a specific test suite: pio test -e <native_env_name>
    ```

*(Note: Ensure PlatformIO is installed and accessible in your environment. If using a Python venv, activate it first: `source .venv/bin/activate`)*

## Development Conventions

*   **Pre-flight Check**: Before starting any work, verify that the following reference repositories are cloned inside this guide repo root (all gitignored) and their latest changes are pulled. If a repo is not cloned, clone it from its origin:
    *   `git clone https://github.com/ev-open-can-tools/ev-open-can-tools.git`
    *   `git clone https://github.com/ev-open-can-tools/ev-open-can-tools-plugins.git`
    *   `git clone https://github.com/hypery11/flipper-tesla-fsd.git` — primary firmware/research reference for the current goals; also contains `enhauto-re/` reverse-engineering notes on the Enhance Auto cable that are useful for Commander connector pinout questions.
*   **Configuration**: Target build environments are defined in `platformio.ini`. Custom configuration and board preferences are managed via scripts and a local profile system.
*   **Versioning**: The project uses Semantic Versioning. The current version is tracked in the `ev-open-can-tools/VERSION` file.
*   **Changelog**: Changes should be documented in `ev-open-can-tools/CHANGELOG.md`. Ongoing work must be added to the `Unreleased` section before merging.
*   **Plugin Architecture**: `ev-open-can-tools` features can be dynamically injected into the CAN stream via JSON-based plugins, but the current plugin schema is not yet enough for hypery11-quality nag suppression because DAS-aware gating and organic torque variation need stateful behavior.
*   **Safety Warning**: The codebase modifies safety-critical vehicle networks. Testing must be rigorous.

## Key Directories & Files

*   **`README.md` (root)**: The primary goal-oriented hardware wiring and installation guide for the Model 3 Highland.
*   **`PROGRESS.md` (root)**: Current physical install state, firmware target, staged rollout plan, and risk decisions.
*   **`install-target.yaml` (root)**: Canonical vehicle, hardware, firmware status, target-bus decision, build-semantics, and rollout configuration for future agents.
*   **`flipper-tesla-fsd/`**: hypery11 reference for the target behavior; use it to verify current hardware support before writing install instructions.
*   **`ev-open-can-tools/platformio.ini`**: The master build configuration file containing all supported boards and native testing environments.
*   **`ev-open-can-tools/src/` & `ev-open-can-tools/include/`**: The C++ source code and headers for the firmware, drivers, and web dashboard.
*   **`ev-open-can-tools/test/`**: Contains Python and C++ native tests.
*   **`ev-open-can-tools-plugins/`**: Houses the JSON files used to define CAN frame modifications for various vehicle versions (HW3, HW4, Legacy).
