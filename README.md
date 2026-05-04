# Tesla Model 3 Highland — RP2040 CAN Wiring Guide

> 2024 Model 3 Highland (HW4) · Adafruit Feather RP2040 CAN (MCP2515) · Nag Suppression

## Shopping List

| Item | Notes |
|---|---|
| [Enhance Auto Gen 2 Cable](https://www.enhauto.com/products/tesla-gen-2-cable) | Select variant **"Model 3 Highland 2024+"** (€13.39). Plugs into X179 on the vehicle side. The other end has a proprietary S3XY Commander connector — left **intact** here, mated with a passive 6-pin breakout adapter (see Adapter Wiring below) so the cable stays reusable. |
| 6-pin breakout adapter | Mates with the Commander-side connector and breaks out to colored pigtails (red, blue, green, white, yellow, black). Lets you tap CAN + power without cutting the Gen 2 cable. |
| MP1584EN Fixed 5V 3A Buck Converter | Fixed 5V output, ~22×17mm. Input 4.5–28V, no adjustment needed. The RP2040 board runs on 5V and has no built-in 12V regulator (unlike the S3XY Commander which regulates 12V internally), so an external step-down converter is required. Wire OUT+/OUT− to a USB-C breakout board or directly to the Feather's USB/GND pads. |
| [Adafruit Feather RP2040 CAN](https://www.adafruit.com/product/5724) | MCP2515 CAN controller. Has screw terminal for CAN-H / CAN-L. |
| Heat shrink tubing (flat width ≥ 75mm) | Insulates and dust-proofs the board. Shrinks to fit ~50mm width. |

## Board Preparation

> **⛔ Cut the TERM jumper!**
>
> The Feather RP2040 CAN has an onboard 120Ω termination resistor. The vehicle's CAN bus already has its own termination. A second resistor will cause communication errors.
>
> Find the solder jumper labeled **TERM** on the top of the board and cut the trace between the pads with a hobby knife.

## Adapter Wiring

The Enhance Auto Gen 2 Cable has a **proprietary 6-pin connector** on the Commander end. Rather than cutting it (which makes the cable single-use and non-resaleable), this guide uses a passive **breakout adapter** that mates with that connector and exposes the same 6 pins as colored pigtails.

The cable itself carries 6 lines on the Commander connector — 2 power + 2 CAN buses (Body + Chassis):

![Enhance Auto Gen 2 cable pinout](images/pinout-labeled.png)
*Commander-side connector — Pin 1: 12V+, Pin 2: GND, Pins 3–4: Body Bus pair, Pins 5–6: Other (Chassis) Bus pair*

The breakout adapter mates face-to-face with this connector. Adapter pin order, **right to left** (rightmost mates with cable Pin 1):

| Adapter wire | Cable pin | Signal | Connect to |
|---|---|---|---|
| **Red** | 1 | 12V+ | MP1584EN **IN+** |
| **Blue** | 2 | GND | MP1584EN **IN−** |
| **Green** | 3 | Body CAN (H or L)* | Board screw terminal **H** or **L** |
| **White** | 4 | Body CAN (the other) | Board screw terminal **L** or **H** |
| **Yellow** | 5 | Chassis CAN | Leave disconnected |
| **Black** | 6 | Chassis CAN | Leave disconnected |

\* H vs L within the green/white pair is not yet known. Verify on first power-up — the TERM jumper is cut, so swapping if traffic doesn't appear at 500 kbit/s is safe.

### Verify before plugging into the car

Take 5 minutes with a multimeter before energizing anything:

1. **Power orientation** (cable plugged into car, car awake): DC volts red→chassis ≈ **12V**; blue should show continuity to chassis. If red reads 0V instead, the adapter is reversed end-to-end and the wire-to-pin mapping above flips.
1. **CAN pair grouping** (car off 8–10 min): resistance green↔white ≈ **60Ω**, yellow↔black ≈ **60Ω**. Confirms the two buses are paired as expected.

## X179 Connector (Vehicle Side)

The cable handles this for you, but for reference, the X179 connector pinout:

![X179 connector pinout diagram](images/x170-enhanced.png)
*X179 connector — Vehicle side and Commander side pinout*

| Pin | Signal |
|---|---|
| 1 | +12V (VCC) |
| 9 | Body CAN-H |
| 10 | Body CAN-L |
| 13 | Chassis CAN-H |
| 14 | Chassis CAN-L |
| 20 | GND |

## Installation Steps

### Step 1: Mate the adapter

Plug the 6-pin breakout adapter into the Commander-side connector of the Gen 2 cable. The Gen 2 cable stays intact. You now have 6 colored pigtails to work with — see [Adapter Wiring](#adapter-wiring).

### Step 2: Access the X179 connector

The X179 is in the **passenger side footwell**, behind the right panel trim. Pull the panel gently — it pops off, no tools required.

![X179 connector location](images/x179-connector.png)
*X179 connector location — passenger side footwell, right panel*

Video walkthrough: [Enhance Auto installation video](https://youtube.com/watch?v=ifwJNZgykVI)

### Step 3: Wire the board

Connect the adapter's pigtails to the board and MP1584EN converter:

1. **Green** (Body CAN) → board screw terminal **H** (try this polarity first; swap with white if no traffic)
1. **White** (Body CAN) → board screw terminal **L**
1. **Red** (12V+) → MP1584EN **IN+**
1. **Blue** (GND) → MP1584EN **IN−**
1. MP1584EN **OUT+** → USB-C breakout **VBUS** (or Feather **USB** pad)
1. MP1584EN **OUT−** → USB-C breakout **GND** (or Feather **GND** pad)
1. USB-C breakout → Feather USB-C port (if using breakout board)
1. Leave **Yellow** and **Black** (Chassis Bus) disconnected and insulated

> **Tip:** The easiest approach is to solder the MP1584EN output to a USB-C breakout board, then plug into the Feather with a short USB-C cable. Alternatively, solder directly to the Feather's **USB** and **GND** pads on the back of the board.

![Completed wiring](images/connected-setup.png)
*Completed setup — Feather RP2040 CAN + DC/DC converter, ready to plug in*

### Step 4: Wrap with heat shrink tubing

The Feather RP2040 CAN has exposed solder joints on the bottom side. Wrapping the board prevents short circuits against metal surfaces in the footwell and keeps dust out.

1. Connect all wires first (CAN-H, CAN-L, USB-C power)
1. Slide a flat heat shrink tube (≥ 75mm flat width) over the board, with wires exiting from both ends
1. Use a heat gun to shrink — it will conform tightly to the board
1. Seal any gaps at the wire exits with a dab of hot glue

### Step 5: Plug into the car

Plug the cable's vehicle-side connector into the X179 port — it clicks into place. The board powers on automatically when the car wakes up.

### Step 6: Flash the firmware

If you haven't already flashed the firmware:

1. Hold **BOOTSEL** on the board while connecting to USB
1. A drive named **RPI-RP2** appears
1. Drag `firmware.uf2` onto the drive
1. Board reboots automatically — done

Or via PlatformIO:

```bash
source .venv/bin/activate && pio run -e feather_rp2040_can -t upload
```

### Step 7: Tuck and reassemble

Place the board and DC/DC converter behind the footwell panel. Snap the trim panel back into place. No visible wires.

## Important Notes

> **Termination resistor:** You must cut the **TERM** jumper on the Feather RP2040 CAN before connecting. Failing to do so will cause CAN bus communication errors.

> **Car must be off during wiring:** Turn the car off and wait 8–10 minutes for all CAN buses and relays to shut down before plugging/unplugging connectors.

> **FSD on HW4 + firmware 2026.2.9.x:** FSD activation is currently **not working** on this firmware version. Nag suppression (bit 19 on CAN ID 1021 mux 1) works independently and is unaffected.

## Pre-Install Checklist

- [ ] 6-pin breakout adapter mated with Gen 2 cable's Commander connector (cable left intact)
- [ ] Adapter orientation verified with multimeter: red ≈ 12V, blue → GND
- [ ] CAN pair grouping verified: green↔white ≈ 60Ω, yellow↔black ≈ 60Ω (car off 8–10 min)
- [ ] TERM jumper cut on Feather RP2040 CAN board
- [ ] Firmware flashed (`firmware.uf2`)
- [ ] Green wired to board screw terminal H, White to L (swap if no traffic at 500 kbit/s)
- [ ] Red wired to MP1584EN IN+, Blue to IN−
- [ ] MP1584EN OUT+/OUT− connected to Feather (via USB-C breakout or direct solder)
- [ ] Yellow and Black (Chassis Bus) left disconnected and insulated
- [ ] Board wrapped in heat shrink tubing, wire exits sealed
- [ ] Car fully powered off for 8–10 minutes before plugging in

## Firmware Configuration

Current `sketch_config.h` settings:

```cpp
#define DRIVER_MCP2515   // Adafruit Feather RP2040 CAN
#define HW4              // 2024 Model 3 Highland

// Optional features (uncomment to enable):
// #define ISA_SPEED_CHIME_SUPPRESS
// #define EMERGENCY_VEHICLE_DETECTION
// #define FORCE_FSD
```

**Active features:** Nag suppression (always on), Summon enable (always on)
**Optional:** ISA speed chime suppress, emergency vehicle detection

## Sources

- [EV Open Can Mod](https://github.com/ev-open-can-tools/ev-open-can-tools)
- [Enhance Auto — Gen 2 Cable](https://www.enhauto.com/products/tesla-gen-2-cable)
- [Enhance Auto — X179 Installation Video](https://youtube.com/watch?v=ifwJNZgykVI)
- [Adafruit — Feather RP2040 CAN Documentation](https://learn.adafruit.com/adafruit-rp2040-can-bus-feather)
- [Tesla — Model 3 Electrical Reference](https://service.tesla.com/docs/Model3/ElectricalReference/)

---
*Generated for Jay's 2024 Model 3 Highland (HW4) · Firmware 2026.2.9.3 · April 2026*
