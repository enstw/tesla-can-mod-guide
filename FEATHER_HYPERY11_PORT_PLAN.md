# Feather Hypery11 Port Plan

Updated: 2026-05-17

This file is the resumable implementation plan for running hypery11-derived
behavior on the selected Adafruit Feather RP2040 CAN hardware.

Read first:

1. `install-target.yaml` for the canonical vehicle, hardware, bus, and rollout
   constraints.
2. `PROGRESS.md` for physical install state and open field checks.
3. This file for the firmware implementation plan and test matrix.

## Current Baseline

- Root guide repo: `d94e9bd` (working tree carries the plan-doc updates and
  the `patches/` directory described below).
- `ev-open-can-tools`: `3b0bd91` — upstream is read-only from this workspace
  (no fork). Section 1 / 2 / 3 source changes are captured as a patch in
  `patches/2026-05-11-feather-hypery11-port.patch`. See "Resume Instructions"
  below.
- `flipper-tesla-fsd`: `ae25662` (advanced from `888341d` on 2026-05-11).
- `ev-open-can-tools-plugins`: `1d9686b`.

All four repos were pulled on 2026-05-11. PlatformIO 6.1.19 is installed via
`uv tool install platformio` and `pio` is on PATH at `~/.local/bin/pio`.

## Resume Instructions

The implementation of Sections 1, 2, and 3 lives in
`ev-open-can-tools/` which is gitignored from this root repo and cloned
fresh from the upstream by the AGENTS.md pre-flight. A clean clone of
`ev-open-can-tools` at `3b0bd91` does *not* contain the new handler, the
driver mode API, or the new test suites — those are stored as a patch.

To pick up where this session left off:

```bash
cd ev-open-can-tools
# Confirm the patch applies cleanly against 3b0bd91. If git complains, the
# upstream has moved and the patch needs a manual three-way merge.
git apply --check ../patches/2026-05-11-feather-hypery11-port.patch
git apply ../patches/2026-05-11-feather-hypery11-port.patch

# Sanity-check the four native suites.
pio test -e native
pio test -e native_nag
pio test -e native_can_mode
pio test -e native_hypery11

# Build the Feather firmware to verify the wiring.
pio run -e feather_rp2040_can_hypery11
```

Expected outcome:

- Native tests: 205/205 (132 + 28 + 12 + 33).
- Firmware build: SUCCESS, `firmware.uf2` under
  `.pio/build/feather_rp2040_can_hypery11/`.

If `git apply --check` fails because the upstream has advanced past
`3b0bd91`, the patch is still useful as a reference for the intended
edits — re-port the new files (`include/handlers_hypery11.h`,
`test/test_native_can_mode/`, `test/test_native_hypery11/`) and the
driver mode changes by hand.

## Section 1 Status (2026-05-11)

Section 1 "CAN Mode Control" is implemented and green on native tests.

- `CanMode` enum (`Normal`, `ListenOnly`) and `setMode()`/`mode()` API live on
  base `CanDriver` (`include/drivers/can_driver.h`), with `mode_` stored in the
  base class so all drivers preserve mode across calls.
- `MCP2515Driver` and `ESP32_MCP2515Driver` apply mode to chip via
  `setListenOnlyMode()` / `setNormalMode()`, preserve it through `init()` and
  `setFilters()` (no more silent reset to Normal), and gate `send()` so
  listen-only blocks TX at the driver level.
- `MockDriver` tracks mode, gates TX, records blocked frames separately from
  `sent`, and exposes `setFiltersCalls` for test assertions.
- New native env `native_can_mode` with `test/test_native_can_mode/` covers
  default mode, mode persistence across `setFilters`, send gating, callback
  contract on blocked sends, and switch-back-to-Normal. 12/12 pass.
- Regression: `native_nag` (28/28) and `native` umbrella (132/132) still pass.

Deferred from Section 1:
- TWAI and SAME51 drivers store mode in the base class but do not yet apply it
  to hardware or gate `send()`. Out of scope for the Feather path; revisit if
  ESP32-TWAI builds need listen-only too.

## Sections 2 + 3 Status (2026-05-11)

Hypery11 Feather state and Goal A nag logic are implemented and green on
native tests.

- New `FeatherHypery11Handler` (`include/handlers_hypery11.h`) ported from
  `flipper-tesla-fsd/fsd_logic/fsd_handler.c` — `fsd_handle_nag_killer`,
  `fsd_handle_gtw_car_state`, and the relevant slice of
  `fsd_handle_das_status`. Op mode is read from `driver.mode()` rather than a
  duplicate handler field, so the listen-only gate cannot drift out of sync.
- Handler state: `otaInProgress`, `dasApState`, `dasHandsOnState`
  (default 0xFF = unseen → conservative echo), `dasSeen`, `nagKillerActive`,
  `dasGate`, `organicTorque`, and counters (`nagEchoCount`,
  `nagBlockedListenOnly`, `nagBlockedOta`, `epasSeenCount`, `gtwSeenCount`,
  `dasSeenCount`).
- Periodic Serial status emitter (`emitStatusIfDue`) wired from
  `app.h::appLoop` prints one line every ~1 s on the firmware build:
  `[STATUS] t=<ms> mode=<LISTEN|ACTIVE> total=<n> 370=<n> 318=<n> 39B=<n>
  dasSeen=<0|1> dasHO=<n> ota=<0|1> echo=<n> blkLO=<n> blkOTA=<n>`. Compiles
  to a no-op on `NATIVE_BUILD`. This is what makes listen-only bring-up
  distinguishable from "no frames seen at all"; capture with
  `pio device monitor -e feather_rp2040_can_hypery11 -b 115200 --raw |
  tee logs/sniff-$(date +%Y%m%d-%H%M%S).log`.
- 0x370 echo path matches hypery11: clear handsOnLevel bits 7:6 and force
  level=1, counter +1 (lower nibble, wraps), checksum
  `sum(b0..b6) + 0x70 + 0x03`.
- Organic torque: per-instance xorshift32, random walk in raw 2150-2290
  (~1.00-2.40 Nm), grip pulses in raw 2330-2370 (~2.80-3.20 Nm) every
  125-224 frames for 3-5 frames.
- `app.h` SelectedHandler dispatch adds `FEATHER_HYPERY11`. `appSetup()`
  now applies `setMode(CanMode::ListenOnly)` before `init()` when
  `LISTEN_ONLY_BOOT` is defined, then preserves it through `setFilters()`
  (Section 1 contract).
- New env `native_hypery11` runs 36 tests covering Test Matrix items 3-14
  (OTA gate, DAS parse, EPAS hands-on eligibility, DAS gate states 0/2/8,
  unseen DAS conservative echo, listen-only suppression via driver mode,
  nag-killer toggle, counter wrap, output checksum, torque envelope,
  organic vs fixed torque, output ID/DLC, ignored unknown IDs, short DLC,
  per-ID seen counters, `emitStatusIfDue` native no-op). 36/36 pass.
- New firmware env `feather_rp2040_can_hypery11` selects
  `DRIVER_MCP2515 + FEATHER_HYPERY11 + LISTEN_ONLY_BOOT`.

Regression on 2026-05-11: `native` (132/132), `native_nag` (28/28),
`native_can_mode` (12/12), `native_hypery11` (36/36) — 208/208 across the four
suites.

Firmware build sanity-check: `pio run -e feather_rp2040_can_hypery11` succeeded
on 2026-05-11. The Feather build links cleanly with the new SelectedHandler
dispatch + LISTEN_ONLY_BOOT block + status emitter (RAM 3.6 %, Flash 0.9 %,
76 864 bytes). `firmware.uf2` is produced under
`ev-open-can-tools/.pio/build/feather_rp2040_can_hypery11/`.
Note: the dedicated hypery11 env intentionally omits `extra_scripts =
pre:scripts/platformio_sync_profile.py` because that script enforces the
upstream `platformio_profile.h` workflow (LEGACY / HW3 / HW4 selection) which
does not yet know about `FEATHER_HYPERY11`. All defines come straight from
the env's `build_flags` instead.

Deferred from Sections 2 + 3:
- Section 4 serial *command* interface (`MODE LISTEN/ACTIVE`, `NAG ON/OFF`,
  `LOG`). The Section 4 *print* path (periodic `[STATUS]`) is already
  shipped; only the input/parser side is still pending.
- Section 5 build flag refinement (currently `FEATHER_HYPERY11` is the only
  switch; the per-feature flags `DAS_AWARE_GATE_ENABLED` /
  `ORGANIC_TORQUE_ENABLED` are runtime toggles on the handler instead of
  compile flags).
- On-device log persistence. `Serial.printf` is a live USB stream — nothing
  on the Feather stores history across reboots. Durable capture happens at
  the laptop side via the `tee` pipe shown above. If post-install historical
  state becomes needed, plan calls for adding LittleFS + a `LOG` serial
  command to replay the buffer over USB on demand.
- Section 3 task 1's "act on raw value 2 only" interpretation note: the
  handler treats raw values 0/1/3 as not-installing and raw 2 as installing.
  This matches `fsd_handle_gtw_car_state` exactly.

## Bring-Up Power and Diagnostics Cabling

The Feather has one USB-C port and the current install routes car
12 V → MP1584EN buck → 5 V → USB-C male into that port (see
`PROGRESS.md` Done item 5). During bring-up you also need a laptop on
the same USB-C port to capture status logs. Three options, in order of
recommended use:

1. **Bring-up: power from laptop only.** Unplug the buck's USB-C male
   from the Feather, tape it off, plug the laptop in instead. X179 only
   supplies CAN signals on green/white, which still reach the screw
   terminals; ignition ON makes frames flow regardless of the Feather's
   power source. Zero soldering, zero backfeed risk. Use this for the
   first listen-only plug-in and any polarity-swap iterations.

2. **Permanent install: car 5 V → BAT pad, USB-C free for laptop.** The
   Feather RP2040 CAN has a BAT pad / JST connector that goes through
   the onboard ORing diodes; USB-C and BAT can both be powered without
   fighting because the higher voltage wins through separate diodes.
   Cut off the buck's USB-C male, solder the buck's +5 V/GND to BAT/GND.
   After this rewire the diagnostics workflow is "plug laptop in any
   time, no power juggling." This is the right end-state once Goal A is
   proven.

3. **Both at once on USB-C (not recommended).** A USB-C data-only dongle
   or a galvanic USB isolator (e.g. Adafruit 4090) lets the buck's 5 V
   stay wired through the USB-C while the laptop sees data only. CC pin
   negotiation makes the cheap kludge fragile, and the isolator is at
   the same cost/effort as option 2. Skip unless option 2 is blocked
   for some reason.

The plan's earlier "Power note" was already directionally right but did
not spell out the BAT-pad rewire; option 2 is the implementation of it.

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
   - EPAS hands-on level is not `1`. Upstream intent is to act on level `0`
     (nag imminent) and `3` (escalated alarm); the implementation skips only
     level `1` (hands detected), so level `2` also passes through.
   - DAS hands-on state is not `0` and not `8`
4. Build the echo:
   - ID `0x370`
   - DLC 8
   - clear hands-on bits and set level `1`
   - counter = source lower nibble + 1
   - checksum = sum bytes 0..6 plus `0x70 + 0x03`
   - torque random walk about 1.00-2.40 Nm
   - grip pulses about 2.80-3.20 Nm (centered ~3.00 Nm, raw 2350 ± 20) every
     5-9 seconds, lasting 3-5 frames
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

Wireless access (Bluefruit LE UART Friend, ordered 2026-05-16): the same
`[STATUS]` print path and the command parser above must read/write a
`Stream` that is both USB `Serial` *and* hardware `Serial1`. An Adafruit
Bluefruit LE UART Friend wired to `Serial1` (free — MCP2515 is on SPI)
then exposes status + `MODE/NAG/LOG` over BLE so the car-embedded Feather
can be reached without opening the trim panel. Keep TX gating identical
on both transports (listen-only / OTA-installing still block CAN TX
regardless of which link issued `MODE ACTIVE`). See `PROGRESS.md`
Decisions and `install-target.yaml` `hardware.diagnostics`.

NeoPixel status indicator (accepted 2026-05-17): drive the Feather's
onboard NeoPixel from the same state the `[STATUS]` line reports — red =
no frames seen, amber = frames seen but no target IDs (`0x370`/`0x39B`),
green = target IDs locked; encode listen-only vs active distinctly (e.g.
slow breathe vs solid). This makes the in-car 4-combo X179
polarity/pair sweep doable by glancing at the board instead of a laptop
screen. It is a coarse field aid only — it cannot show which CAN ID is
present, so it does not replace `[STATUS]` or the serial logs. Compile
to a no-op on `NATIVE_BUILD` like `emitStatusIfDue`.

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

Sections 1, 2, and 3 are done at the native-test level *and* the firmware
build links cleanly. The next session should pick from:

1. **Section 4 — serial runtime interface.** Add a tiny command parser to
   `appLoop()` (or a sibling helper) for `STATUS`, `MODE LISTEN`,
   `MODE ACTIVE`, `NAG ON`, `NAG OFF`, `LOG`. `STATUS` should print
   `driver.mode()`, `epasSeenCount`, `nagEchoCount`,
   `nagBlockedListenOnly`, `nagBlockedOta`, `otaInProgress`, `dasSeen`,
   `dasApState`, `dasHandsOnState`. Native-test the parser without CAN
   hardware. Mirror the print path and parser to `Serial1` for the
   Bluefruit LE UART Friend (see Section 4 "Wireless access"), and drive
   the onboard NeoPixel from the same state (see Section 4 "NeoPixel
   status indicator") so the in-car X179 polarity sweep needs no laptop.
2. **Bench CAN tests.** Per the existing Test Matrix "Bench CAN Tests"
   section, validate the firmware against a second CAN node before any
   vehicle TX.
3. **Section 5 build-flag cleanup.** Move runtime toggles
   (`dasGate`, `organicTorque`) behind compile flags so disabled features
   shrink flash, and align flag names with `install-target.yaml`.
