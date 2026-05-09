# Feather Hypery11 Port Plan

Updated: 2026-05-10

This file is the resumable implementation plan for running hypery11-derived
behavior on the selected Adafruit Feather RP2040 CAN hardware.

Read first:

1. `install-target.yaml` for the canonical vehicle, hardware, bus, and rollout
   constraints.
2. `PROGRESS.md` for physical install state and open field checks.
3. This file for the firmware implementation plan and test matrix.

## Current Baseline

- Root guide repo: `fc56a7c`
- `ev-open-can-tools`: `3b0bd91`
- `flipper-tesla-fsd`: `888341d`
- `ev-open-can-tools-plugins`: `1d9686b`

Reference repos were present and `git pull` reported up to date on
2026-05-10.

The selected host is `ev-open-can-tools` environment `feather_rp2040_can`,
which targets the Adafruit Feather RP2040 CAN with MCP2515/MCP25625 using the
Arduino framework.

Important current limitation: hypery11 does not ship a Feather RP2040 firmware
artifact. The Feather path must be a logic-only port on top of
`ev-open-can-tools`, not a direct Flipper app port or ESP32 dashboard port.

## Goal

Implement a Feather firmware build that can safely start in passive sniffing
mode, validate the X179 target bus, and then run Goal A nag suppression using
hypery11-derived logic.

Initial active scope:

1. True MCP2515 listen-only boot/sniff mode.
2. `0x370` EPAS nag echo.
3. `0x39B` DAS-aware nag gate.
4. Organic torque variation.
5. `0x318` OTA-in-progress TX pause.
6. USB serial status/log/control surface at 115200 baud.

Explicitly deferred:

1. `0x3FD` FSD region unlock.
2. `0x7FF` Ban Shield.
3. `0x3F8` telemetry-disable code path.
4. ESP32 WiFi dashboard parity.
5. Runtime JSON plugin upload/install on Feather.

## Plugin Strategy

There are two different plugin-like concepts in this workspace:

1. hypery11 behavior modules in `flipper-tesla-fsd`, implemented as C logic.
2. `ev-open-can-tools-plugins` JSON rules, processed today by the ESP32
   dashboard/SPIFFS path.

For the first Feather vehicle build, implement hypery11 behavior as compiled
C++ feature modules behind a `CarManagerBase` handler. Do not depend on the
existing JSON plugin loader for nag suppression. The JSON `Beta/nag-killer`
plugin is useful as a reference for the basic counter echo, but it is not
expressive enough for DAS-aware gating, OTA pause, or organic torque variation.

Feather JSON plugin support can be added later by refactoring the existing
plugin engine behind portable storage and control abstractions:

1. Split JSON rule parsing/application from ESP32 dashboard code.
2. Replace direct `SPIFFS` use with a storage interface.
3. Add an RP2040 LittleFS implementation for plugin files.
4. Add a no-filesystem build option for fixed compiled plugins.
5. Add USB serial commands to list/install/remove/toggle plugins.
6. Merge handler filter IDs with plugin filter IDs on Feather.
7. Keep plugin TX blocked while listen-only or OTA-in-progress is active.

That later work should not block Goal A.

## Implementation Checklist

### 1. CAN Mode Control

Files:

- `ev-open-can-tools/include/drivers/can_driver.h`
- `ev-open-can-tools/include/drivers/mcp2515_driver.h`
- other concrete drivers as needed for compile compatibility

Tasks:

1. Add a driver mode API, for example `setMode(CanMode mode)`.
2. Support at least `Normal` and `ListenOnly`.
3. Make `MCP2515Driver::init()` and `setFilters()` preserve the requested mode
   instead of always returning to normal mode.
4. Ensure listen-only prevents TX at both driver and handler levels.
5. Add native tests for mode gating using `MockDriver`.

Reason: current MCP2515 setup returns to normal mode after init/filter setup,
so the first vehicle connection cannot yet be guaranteed passive.

### 2. Hypery11 Feather State

Files:

- new handler/header under `ev-open-can-tools/include/`
- optionally small helper module under `ev-open-can-tools/src/`
- native tests under `ev-open-can-tools/test/`

Tasks:

1. Add a Feather/HW4 state struct carrying:
   - op mode: listen-only or active
   - OTA-in-progress flag from `0x318`
   - latest DAS hands-on state from `0x39B`
   - nag enabled flag
   - echo counters and last status
2. Default to listen-only for first boot.
3. Default telemetry-disable and FSD region unlock to off.
4. Keep state independent from the Flipper UI and ESP32 dashboard.

### 3. Goal A Nag Logic

Reference:

- `flipper-tesla-fsd/fsd_logic/fsd_handler.c`
- `flipper-tesla-fsd/fsd_logic/fsd_handler.h`
- current simple handler in `ev-open-can-tools/include/handlers.h`

Tasks:

1. Parse `0x318` byte 6 bits 1:0. Only raw value `2` means OTA installing and
   must block TX.
2. Parse `0x39B`:
   - `das_ap_state = (byte1 >> 4) & 0x0F`
   - `das_hands_on_state = (byte5 >> 2) & 0x0F`
3. On `0x370`, echo only when:
   - nag feature enabled
   - not listen-only
   - not OTA installing
   - EPAS hands-on level is `0` or `3`
   - DAS hands-on state is not `0` and not `8`
4. Build the echo:
   - ID `0x370`
   - DLC 8
   - clear hands-on bits and set level `1`
   - counter = source lower nibble + 1
   - checksum = sum bytes 0..6 plus `0x70 + 0x03`
   - torque random walk about 1.00-2.40 Nm
   - grip pulses about 3.10-3.30 Nm every 5-9 seconds
5. Rate-limit serial/log output so runtime does not block CAN processing.

### 4. Serial Runtime Interface

Files:

- likely `ev-open-can-tools/include/app.h`
- optional new serial command helper

Minimum commands:

1. `STATUS` prints mode, frame counts, target-frame seen flags, last DAS state,
   OTA flag, and nag echo count.
2. `MODE LISTEN` switches to listen-only and disables TX.
3. `MODE ACTIVE` switches to normal mode only after explicit command.
4. `NAG ON` and `NAG OFF` toggle nag echo logic.
5. `LOG` dumps the in-memory ring buffer.

Live log capture from a development machine:

```bash
cd ev-open-can-tools
pio device monitor -e feather_rp2040_can -b 115200 --raw | tee ../logs/feather-$(date +%Y%m%d-%H%M%S).log
```

Power note: avoid feeding the board from car buck 5V and laptop USB 5V at the
same time unless the USB setup prevents 5V backfeed. Use laptop-only power for
bench tests, or use a data-only USB arrangement during in-car logging.

### 5. Build Flags

The exact macro names can still be chosen, but the semantics should match
`install-target.yaml`:

1. Feather RP2040 CAN board.
2. MCP2515 driver.
3. HW4 vehicle profile.
4. listen-only boot enabled.
5. nag killer enabled.
6. DAS-aware gate enabled.
7. organic torque variation enabled.
8. FSD region unlock disabled for Goal A.
9. telemetry disable disabled.

Avoid silently switching the install target to stock `ev-open-can-tools`
behavior. If stock fallback is used, call it out explicitly.

## Test Matrix

### Native Unit Tests

Add or extend native tests for:

1. Driver mode API defaults to listen-only for the Feather target.
2. `setFilters()` does not force normal mode.
3. `0x318` raw update state `2` blocks all TX.
4. `0x318` raw states `0`, `1`, and `3` do not set OTA-installing.
5. `0x39B` parser extracts DAS AP and hands-on state correctly.
6. `0x370` hands-on `0` produces one echo when active and allowed.
7. `0x370` hands-on `3` produces one echo and clears level to `1`.
8. `0x370` hands-on `1` produces no echo.
9. DAS hands-on state `0` produces no echo.
10. DAS hands-on state `8` produces no echo.
11. Unseen DAS state `0xFF` echoes conservatively.
12. Output counter wraps lower nibble correctly.
13. Output checksum is correct.
14. Organic torque stays inside the planned normal and pulse ranges.
15. Serial command parser changes mode/toggles without requiring CAN hardware.

Expected commands once PlatformIO is available:

```bash
cd ev-open-can-tools
pio test -e native_nag
pio test -e native
```

Known local blocker as of this handoff: `pio` was not found on the current
shell `PATH`. Before firmware work, check for a repo venv or install/activate
PlatformIO:

```bash
which pio
find . -maxdepth 3 -type f -name activate
```

### Golden Reference Tests

Create a small set of sample frames and compare Feather output against
hypery11 behavior for:

1. `0x318` OTA state handling.
2. `0x39B` DAS state parsing.
3. `0x370` echo shape, counter, checksum, and hands-on bit behavior.

Do not require byte-for-byte torque values if the PRNG implementation differs;
assert the allowed torque ranges and pulse cadence instead.

### Bench CAN Tests

Use a second CAN node or analyzer before any vehicle TX:

1. Replay target frames at 500 kbit/s.
2. Confirm listen-only mode emits zero frames on the bus.
3. Switch to active bench mode by serial command.
4. Confirm only expected `0x370` echo frames are emitted.
5. Inject `0x318` OTA installing and confirm TX stops.
6. Confirm serial `STATUS` and live log output stay responsive.

### Vehicle Bring-Up

Do not begin vehicle active TX until native and bench tests pass.

1. Connect in listen-only only.
2. Verify target bus by frame visibility, not name:
   - required: `0x370`
   - required: `0x39B`
   - required before later region work: `0x3FD`
   - ideal: `0x7FF`
3. If target frames are not visible, try:
   - green/white normal polarity
   - green/white swapped
   - yellow/black normal polarity
   - yellow/black swapped
4. Save serial logs from each attempt.
5. Only enable active nag after target frames are visible and stable.

## Next Concrete Step

Implement CAN mode support first. The next coding session should start with:

1. Update `CanDriver` with mode semantics.
2. Update `MCP2515Driver` to support and preserve listen-only.
3. Update `MockDriver` for tests.
4. Add tests proving filter setup does not force active TX mode.

Only after this lands should the hypery11-derived nag handler be added.
