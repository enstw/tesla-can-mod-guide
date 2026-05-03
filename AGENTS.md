# Tesla CAN Mod Guide

## Project Overview

This workspace contains tools and documentation for modifying and analyzing CAN bus traffic on supported electric vehicles (primarily Tesla). It consists of three main components:

1. **Root Directory (`/`)**: Contains a wiring and installation guide (`README.md`) specifically for using the Adafruit Feather RP2040 CAN (MCP2515) with a 2024 Model 3 Highland (HW4).
2. **`ev-open-can-tools/`**: The core C++ firmware project. It sits on the vehicle's CAN bus, monitors frames, and applies real-time changes. It supports multiple boards (ESP32, RP2040, M5Stack, Waveshare) and can optionally host a local web interface (ESP32 dashboard) for configuration, plugins, and diagnostics.
3. **`ev-open-can-tools-plugins/`**: A collection of JSON configuration files representing plugins that define specific CAN bus modifications.

**Core Technologies:**
- **Language**: C/C++ (Arduino Framework)
- **Build System**: PlatformIO
- **Microcontrollers**: ESP32 variants, RP2040, SAMD51
- **CAN Interfaces**: TWAI (ESP32 native), MCP2515 (SPI)
- **Key Libraries**: ArduinoJson

## Building and Running

The core firmware code is located in the `ev-open-can-tools/` directory. All build commands should be executed from within that directory.

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

*   **Pre-flight Check**: Before starting any work, verify that both the `ev-open-can-tools` and `ev-open-can-tools-plugins` repositories are cloned and their latest changes are pulled. If they are not cloned, clone them from their respective origins:
    *   `git clone https://github.com/ev-open-can-tools/ev-open-can-tools.git`
    *   `git clone https://github.com/ev-open-can-tools/ev-open-can-tools-plugins.git`
*   **Configuration**: Target build environments are defined in `platformio.ini`. Custom configuration and board preferences are managed via scripts and a local profile system.
*   **Versioning**: The project uses Semantic Versioning. The current version is tracked in the `ev-open-can-tools/VERSION` file.
*   **Changelog**: Changes should be documented in `ev-open-can-tools/CHANGELOG.md`. Ongoing work must be added to the `Unreleased` section before merging.
*   **Plugin Architecture**: Features are dynamically injected into the CAN stream via JSON-based plugins, which keeps the core C++ logic abstracted from specific vehicle hacks.
*   **Safety Warning**: The codebase modifies safety-critical vehicle networks. Testing must be rigorous.

## Key Directories & Files

*   **`README.md` (root)**: The primary hardware wiring and installation guide for the Model 3 Highland.
*   **`ev-open-can-tools/platformio.ini`**: The master build configuration file containing all supported boards and native testing environments.
*   **`ev-open-can-tools/src/` & `ev-open-can-tools/include/`**: The C++ source code and headers for the firmware, drivers, and web dashboard.
*   **`ev-open-can-tools/test/`**: Contains Python and C++ native tests.
*   **`ev-open-can-tools-plugins/`**: Houses the JSON files used to define CAN frame modifications for various vehicle versions (HW3, HW4, Legacy).