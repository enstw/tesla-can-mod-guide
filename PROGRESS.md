# Install Progress

Status of the Model 3 Highland CAN mod build. See [README.md](./README.md) for the full guide.

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
