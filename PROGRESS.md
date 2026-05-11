# Install Progress

Status of the Model 3 Highland CAN mod build. See [README.md](./README.md) for the full guide.

Canonical target configuration: [install-target.yaml](./install-target.yaml).

Primary goals:

1. **Nag suppression** — first active feature after listen-only bring-up.
1. **FSD region unlock** — unlock the regional UI gate for this VIN's paid FSD entitlement.
1. **Telemetry disable evaluation** — research only for now; default decision remains disabled.

**Vehicle firmware:** 2026.14.3 (observed 2026-05-05; previously 2026.2.9.3).

**Current handoff:** 2026-05-10. Hypery11 behavior remains the target, but hypery11 currently supports Flipper Zero and ESP32 targets, not Feather RP2040. The selected Feather hardware requires a logic-only port onto the existing `ev-open-can-tools` Feather RP2040/MCP2515 framework, not a port of the Flipper app or ESP32 dashboard. Resume firmware implementation from [`FEATHER_HYPERY11_PORT_PLAN.md`](./FEATHER_HYPERY11_PORT_PLAN.md).

## Done

1. **Tesla Gen 2 cable (Enhance Auto)** — acquired.
1. **Power chain wired and bench-tested**: 12V input → MP1584EN buck converter → 5V → USB-C male → Adafruit Feather RP2040 CAN.
1. **Wiring strategy decided**: keep the Gen 2 cable intact; mate a passive 6-pin breakout adapter with the Commander-side connector instead of cutting it. README updated to reflect this.
1. **TERM jumper cut** on the Feather RP2040 CAN. Verified: H↔L across the screw terminals reads ~32 kΩ (unpowered, no wiring) — clean cut, terminator isolated.
1. **Cabling complete**:
   1. Commander adapter red/blue → MP1584EN IN+/IN− → 5V out → Feather USB/GND.
   1. Commander adapter green/white → Feather screw terminal H/L (first attempt).
   1. Yellow + black left disconnected and insulated for now.

   **Open question — adapter pair mapping:** `pinout-labeled.png` labels green/white as Body Bus and yellow/black as Other Bus. The restored `x170-enhanced.png` labels X179 pin 13/14 as Chassis CAN; hypery11 documents X179 pin 13/14 as Bus 6 mixed forwarding. For this repo's goals, target the pair that maps to X179 pin 13/14 and verify by sniffing for `0x370`, `0x39B`, `0x3FD`, and ideally `0x7FF`. Try up to 4 combos: {green/white, yellow/black} × {H/L, L/H}.
1. **Hardware comparison clarified**:
   1. Flipper Zero + Electronic Cats CAN Add-On is hypery11's reference/upstream path.
   1. ESP32 + CAN is supported by hypery11 now and is the best path for web UI, bus discovery, and a practical permanent install.
   1. Feather RP2040 CAN remains the selected physical hardware, but hypery11 behavior requires a Feather/RP2040 port.
1. **Feather port architecture selected**:
   1. Keep the existing `ev-open-can-tools` `feather_rp2040_can` PlatformIO environment and MCP2515 driver.
   1. Add a hypery11-derived `CarManagerBase` handler instead of porting the Flipper app or ESP32 dashboard.
   1. Start with a minimal nag-only build: `0x370` EPAS echo, `0x39B` DAS-aware gate, and `0x318` OTA TX pause.
   1. Add true MCP2515 listen-only boot support before any vehicle connection or active TX test.
1. **Feather implementation handoff captured**:
   1. Added a dedicated resumable implementation plan in [`FEATHER_HYPERY11_PORT_PLAN.md`](./FEATHER_HYPERY11_PORT_PLAN.md).
   1. Clarified that Goal A should use compiled hypery11-derived C++ feature logic first, not the ESP32/SPIFFS JSON plugin path.
   1. Captured the native, golden-reference, bench CAN, and vehicle bring-up test matrix.
   1. Recorded the local build blocker: `pio` was not found on the current shell `PATH`; check for a venv or install/activate PlatformIO before firmware builds.

## In progress

1. **Implement the Feather logic-only port before vehicle TX**:
   1. Start with explicit listen-only/normal mode control in the existing Feather MCP2515 path.
   1. Add a `CanFrame` adapter or native handler for the needed hypery11 logic.
   1. Implement Goal A first: nag suppression only, with DAS-aware gating and organic torque variation.
   1. Defer `0x3FD` region unlock, `0x7FF` Ban Shield, and `0x3F8` telemetry-disable code until nag-only operation is proven.

## Not started

1. Wrap the Feather in heat shrink, seal wire exits.
1. Plug into X179 (passenger footwell, right panel) with the car powered off for 8–10 minutes.
1. Confirm target CAN traffic in listen-only/sniffer mode; swap H/L polarity or move to the other adapter pair if no 500 kbit/s traffic or target frames are visible.
1. Tuck behind the trim panel and reassemble.

## Vehicle situation & feature plan

- **Vehicle:** 2024 Model 3 Highland HW4 (Long Range AWD), firmware 2026.14.3.
- **Paid entitlements on this VIN:**
  1. **FSD** — paid permanent purchase, currently region-locked (Taiwan). UI/region gate is the blocker, not entitlement — so `0x3FD` bit injection (HW4: bit 46 + bit 60) is the relevant unlock path.
  1. **Acceleration Boost** — Long Range AWD upgraded to 0–100 km/h <4 s. Already active in software; no CAN mod needed. Listed for completeness so future feature work doesn't accidentally try to "unlock" something already on.
- **Risk context:** Tesla's April 2026 ban wave specifically targeted region-unlock users in EU/UK/CN/JP/KR. Taiwan is in the same detection cluster. A VIN ban on a paid permanent FSD is a much larger loss than for monthly subscribers — strongly favours stealth-first rollout.

### Layered rollout plan (post-cabling, post-firmware-flash)

1. **Listen-only first** — verify wiring and 500 kbit/s X179 pin 13/14 / Bus 6 mixed-forwarding traffic without ever transmitting.
1. **Goal A: nag suppression only** — `0x370` EPAS counter+1 echo, gated by `0x39B` `DAS_autopilotHandsOnState` (states 2–7, 9–10), with organic torque variation (xorshift32 random walk 1.00–2.40 Nm + grip pulses 3.10–3.30 Nm every 5–9 s). Drive on this for at least a few weeks to confirm no server reaction.
1. **Add Ban Shield** — `0x7FF` healthy snapshot retransmit. Must be enabled *before* `0x3FD` region unlock so it captures pre-modification baseline of `GTW_carConfig`.
1. **Goal B: add region unlock** — `0x3FD` bit 46 + bit 60.
1. **Goal C: evaluate telemetry disable only if deliberately testing offline** — candidate `0x3F8` bit 43 (`UI_enableTripTelemetry=0`), with SIM removed and WiFi disabled. Keep off for normal use.
1. **Hold TLSSC Restore (`0x331` byte[0]=0x1B) in reserve** — only deploy if Tesla VIN-bans us and removes TLSSC.

### Decisions (with reasoning)

- **Telemetry Off (`0x3F8` bit 43): not enabled by default.** Researched 2026-05-05. hypery11 labels this beta and warns it may itself be a ban signal. Tesla documentation confirms vehicle data can be stored locally and later accessed or transmitted; disabling one UI trip-telemetry flag should not be treated as a reliable privacy or anti-ban boundary. For this paid permanent FSD VIN, normal use still needs OTAs, service, and connectivity, so Telemetry Off remains an offline-only experiment, not part of the staged active rollout.
- **X179 pin 13/14 / Bus 6 mixed forwarding is the target bus.** The restored image labels pin 13/14 as Chassis CAN and pin 9/10 as Body CAN; hypery11 labels 13/14 as Bus 6 mixed forwarding. Frame visibility is the deciding test, not the Body/Chassis name.
- **Adapter approach over cable cut.** Keeps the Gen 2 cable resaleable; adapter color-to-X179-bus mapping still needs field verification (see Done item 5).
- **Feather firmware path corrected.** Do not assume a hypery11 Feather `firmware.uf2` exists today. Hypery11 supports Flipper and ESP32 now; Feather requires a port or a deliberate stock-firmware fallback.
- **Feather port should be logic-only.** The existing `ev-open-can-tools` app loop already feeds filtered `CanFrame` messages into a `CarManagerBase` handler on top of the Feather MCP2515 driver. The right port is to replace/upgrade the current simple `NagHandler` with hypery11-derived logic, not to bring over Flipper UI code, ESP32 WiFi/dashboard code, or preferences infrastructure.
- **Bring-up uses laptop USB power, not the car buck.** Decided 2026-05-11. The Feather has one USB-C port and the current install routes car 12 V → MP1584EN → 5 V → USB-C male into it. For the first plug-in we need the laptop on that same port for status logs. Cleanest answer is to unplug the buck's USB-C, tape it off, and let the laptop power the Feather; X179 still supplies CAN signals on green/white regardless of where the 5 V comes from. After Goal A is proven, the permanent install rewires the buck's +5 V to the Feather's BAT pad (onboard ORing diodes handle USB-C-vs-BAT), freeing USB-C for diagnostics. See `FEATHER_HYPERY11_PORT_PLAN.md` "Bring-Up Power and Diagnostics Cabling".
