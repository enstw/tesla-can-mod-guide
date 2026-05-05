# Install Progress

Status of the Model 3 Highland CAN mod build. See [README.md](./README.md) for the full guide.

**Vehicle firmware:** 2026.14.3 (observed 2026-05-05; previously 2026.2.9.3).

## Done

1. **Tesla Gen 2 cable (Enhance Auto)** — acquired.
1. **Power chain wired and bench-tested**: 12V input → MP1584EN buck converter → 5V → USB-C male → Adafruit Feather RP2040 CAN.
1. **Wiring strategy decided**: keep the Gen 2 cable intact; mate a passive 6-pin breakout adapter with the Commander-side connector instead of cutting it. README updated to reflect this.
1. **TERM jumper cut** on the Feather RP2040 CAN. Verified: H↔L across the screw terminals reads ~32 kΩ (unpowered, no wiring) — clean cut, terminator isolated.
1. **Cabling complete**:
   1. Commander adapter red/blue → MP1584EN IN+/IN− → 5V out → Feather USB/GND.
   1. Commander adapter green/white → Feather screw terminal H/L (first attempt).
   1. Yellow + black left disconnected and insulated for now.

   **Open question — body vs chassis:** `pinout-labeled.png` labels middle pair (green/white) as Body Bus; `x170-enhanced.png` Commander-side diagram suggests rightmost pair (yellow/black) is Body. To be resolved by field test — try up to 4 combos: {green/white, yellow/black} × {H/L, L/H}.

## In progress

1. **Flash `firmware.uf2`** to the Feather (BOOTSEL → drag-drop, or `pio run -e feather_rp2040_can -t upload`).

## Not started

1. Wrap the Feather in heat shrink, seal wire exits.
1. Plug into X179 (passenger footwell, right panel) with the car powered off for 8–10 minutes.
1. Confirm CAN traffic on the Feather; swap green/white at the Feather screw terminal if no 500 kbit/s traffic.
1. Tuck behind the trim panel and reassemble.

## Vehicle situation & feature plan

- **Vehicle:** 2024 Model 3 Highland HW4 (Long Range AWD), firmware 2026.14.3.
- **Paid entitlements on this VIN:**
  1. **FSD** — paid permanent purchase, currently region-locked (Taiwan). UI/region gate is the blocker, not entitlement — so `0x3FD` bit injection (HW4: bit 46 + bit 60) is the relevant unlock path.
  1. **Acceleration Boost** — Long Range AWD upgraded to 0–100 km/h <4 s. Already active in software; no CAN mod needed. Listed for completeness so future feature work doesn't accidentally try to "unlock" something already on.
- **Risk context:** Tesla's April 2026 ban wave specifically targeted region-unlock users in EU/UK/CN/JP/KR. Taiwan is in the same detection cluster. A VIN ban on a paid permanent FSD is a much larger loss than for monthly subscribers — strongly favours stealth-first rollout.

### Layered rollout plan (post-cabling, post-firmware-flash)

1. **Listen-only first** — verify wiring and 500 kbit/s body-bus traffic without ever transmitting (MCP2515 hardware listen-only register).
1. **Nag suppression only** — `0x370` EPAS counter+1 echo, gated by `0x39B` `DAS_autopilotHandsOnState` (states 2–7, 9–10), with organic torque variation (xorshift32 random walk 1.00–2.40 Nm + grip pulses 3.10–3.30 Nm every 5–9 s). Drive on this for at least a few weeks to confirm no server reaction.
1. **Add Ban Shield** — `0x7FF` healthy snapshot retransmit. Must be enabled *before* `0x3FD` region unlock so it captures pre-modification baseline of `GTW_carConfig`.
1. **Add region unlock** — `0x3FD` bit 46 + bit 60.
1. **Hold TLSSC Restore (`0x331` byte[0]=0x1B) in reserve** — only deploy if Tesla VIN-bans us and removes TLSSC.

### Decisions (with reasoning)

- **Telemetry Off (`0x3F8` bit 43): not enabled.** Researched 2026-05-05. Once flipped, the bit-flip event likely persists on MCU/Gateway flash for the life of the vehicle (per IEEE Spectrum / forensic-research consensus on Tesla gateway log retention). Provides damage-limitation if connectivity ever returns, but does not enable safe reconnection — and our threat model (paid permanent FSD on Highland that will need OTAs and may need service visits) makes "permanent SIM-out, no WiFi, no service, no OTA" infeasible. Organic torque variation is the cheaper mitigation.
- **Adapter approach over cable cut.** Keeps the Gen 2 cable resaleable; pinout body/chassis still TBD by field test (see Done item 5).
