# Tesla Model 3 Highland — CAN Mod Guide

> 2024 Model 3 Highland (HW4) · X179 pin 13/14 target bus · Feather RP2040 CAN hardware selected · hypery11 behavior target, Feather port pending

## Goals

This repo is organized around three vehicle goals, in rollout order:

1. **Nag suppression** — first active feature after listen-only validation. Target is `0x370` EPAS counter+1 echo, gated by `0x39B` DAS hands-on state where available, with organic torque variation. Lower detection risk than entitlement/config changes, but still an active CAN modification.
1. **FSD region unlock** — for this VIN's paid permanent FSD entitlement that is region-locked in Taiwan. Target is `0x3FD` bit 46 + bit 60 on HW4, only after the hardware is proven stable and ban-defense behavior is understood.
1. **Telemetry disable evaluation** — research goal, not a default active feature. Candidate target is `0x3F8` bit 43 (`UI_enableTripTelemetry=0`), but current decision is to keep it disabled unless intentionally testing offline with SIM removed and WiFi disabled.

Non-goal: Acceleration Boost is already paid and active on this VIN; no CAN unlock is needed for it.

Canonical target configuration lives in [`install-target.yaml`](./install-target.yaml). Use it as the source of truth for vehicle, hardware, firmware status, bus-selection assumptions, and build-flag semantics.

## Shopping List

| Item | Notes |
|---|---|
| [Enhance Auto Gen 2 Cable](https://www.enhauto.com/products/tesla-gen-2-cable) | Select variant **"Model 3 Highland 2024+"** (€13.39). Plugs into X179 on the vehicle side. The other end has a proprietary S3XY Commander connector — left **intact** here, mated with a passive 6-pin breakout adapter (see Adapter Wiring below) so the cable stays reusable. |
| 6-pin breakout adapter | Mates with the Commander-side connector and breaks out to colored pigtails (red, blue, green, white, yellow, black). Lets you tap CAN + power without cutting the Gen 2 cable. |
| MP1584EN Fixed 5V 3A Buck Converter | Fixed 5V output, ~22×17mm. Input 4.5–28V, no adjustment needed. The RP2040 board runs on 5V and has no built-in 12V regulator (unlike the S3XY Commander which regulates 12V internally), so an external step-down converter is required. Wire OUT+/OUT− to a USB-C breakout board or directly to the Feather's USB/GND pads. |
| [Adafruit Feather RP2040 CAN](https://www.adafruit.com/product/5724) | Selected permanent-install hardware. MCP2515 CAN controller with screw terminal for CAN-H / CAN-L. Note: hypery11 does not currently ship a Feather RP2040 target; using this board for hypery11 behavior requires a port. |
| Optional ESP32 discovery/firmware hardware | For immediate hypery11 ESP32 support and web UI, use M5Stack ATOM Lite + ATOMIC CAN Base, LILYGO T-CAN485, Waveshare ESP32-S3-RS485-CAN, or generic ESP32 + CAN per `flipper-tesla-fsd/esp32/README.md`. |
| Heat shrink tubing (flat width ≥ 75mm) | Insulates and dust-proofs the board. Shrinks to fit ~50mm width. |

## Board Preparation

> **⛔ Cut the TERM jumper!**
>
> The Feather RP2040 CAN has an onboard 120Ω termination resistor. The vehicle's CAN bus already has its own termination. A second resistor will cause communication errors.
>
> Find the solder jumper labeled **TERM** on the top of the board and cut the trace between the pads with a hobby knife.

## Adapter Wiring

The Enhance Auto Gen 2 Cable has a **proprietary 6-pin connector** on the Commander end. Rather than cutting it (which makes the cable single-use and non-resaleable), this guide uses a passive **breakout adapter** that mates with that connector and exposes the same 6 pins as colored pigtails.

The cable itself carries 6 lines on the Commander connector — 2 power + 2 CAN pairs:

![Enhance Auto Gen 2 cable pinout](images/pinout-labeled.png)
*Commander-side connector photo labels pins 3–4 as Body Bus and pins 5–6 as Other Bus. The restored X179 diagram labels X179 pin 13/14 as Chassis CAN, while hypery11 documents pin 13/14 as Bus 6 mixed forwarding. Treat color-to-bus mapping as unconfirmed until sniffed.*

The breakout adapter mates face-to-face with this connector. Adapter pin order, **right to left** (rightmost mates with cable Pin 1):

| Adapter wire | Cable pin | Signal | Connect to |
|---|---|---|---|
| **Red** | 1 | 12V+ | MP1584EN **IN+** |
| **Blue** | 2 | GND | MP1584EN **IN−** |
| **Green** | 3 | Candidate CAN pair A (H or L)* | Test as target pair first only if it carries `0x370`, `0x39B`, `0x3FD`, and `0x7FF` |
| **White** | 4 | Candidate CAN pair A (the other) | Test with green |
| **Yellow** | 5 | Candidate CAN pair B (H or L)* | Test if pair A does not show target frames |
| **Black** | 6 | Candidate CAN pair B (the other) | Test with yellow |

\* H vs L within each pair is not yet known. Verify in listen-only/sniffer mode — the TERM jumper is cut, so swapping polarity if traffic does not appear at 500 kbit/s is safe.

### Verify before plugging into the car

Take 5 minutes with a multimeter before energizing anything:

1. **Power orientation** (cable plugged into car, car awake): DC volts red→chassis ≈ **12V**; blue should show continuity to chassis. If red reads 0V instead, the adapter is reversed end-to-end and the wire-to-pin mapping above flips.
1. **CAN pair grouping** (car off 8–10 min): resistance green↔white ≈ **60Ω**, yellow↔black ≈ **60Ω**. Confirms the two buses are paired as expected.
1. **Target pair identification** (listen-only/sniffer): the target pair should show `0x370`, `0x39B`, `0x3FD`, and ideally `0x7FF`. For this project, frame visibility matters more than whether the pair is named Body or Chassis.

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

For this project, the target bus is **X179 pin 13/14**. The restored diagram labels it Chassis CAN, while hypery11's current hardware docs call it **Bus 6 mixed forwarding** and recommend it because it carries the frames needed for nag suppression and FSD-region work. Do not treat pin 13/14 as pure Body CAN.

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

1. Connect the candidate pair believed to map to X179 pin 13/14 / Bus 6 mixed forwarding to the board screw terminal **H/L**.
1. If no 500 kbit/s traffic or no target frames are visible, swap H/L polarity. If still wrong, move to the other candidate pair and repeat.
1. **Red** (12V+) → MP1584EN **IN+**
1. **Blue** (GND) → MP1584EN **IN−**
1. MP1584EN **OUT+** → USB-C breakout **VBUS** (or Feather **USB** pad)
1. MP1584EN **OUT−** → USB-C breakout **GND** (or Feather **GND** pad)
1. USB-C breakout → Feather USB-C port (if using breakout board)
1. Leave the unused CAN pair disconnected and insulated

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

### Step 6: Select and flash firmware

Current status: **hypery11 does not ship a Feather RP2040 build today**. It supports Flipper Zero and ESP32 targets. The selected Feather hardware can only run hypery11 behavior after a Feather/RP2040 port exists.

Valid paths:

1. **Fastest hypery11 path:** use an ESP32-supported board (`m5stack-atom`, `esp32-lilygo`, `waveshare-s3-can`, or `esp32-mcp2515`) for bus discovery and/or install.
1. **Reference hypery11 path:** use Flipper Zero + Electronic Cats CAN Bus Add-On.
1. **Keep Feather hardware:** port hypery11 logic to Feather RP2040 and then flash the generated `firmware.uf2` by BOOTSEL drag-drop.
1. **Fallback only:** flash stock `ev-open-can-tools` `feather_rp2040_can`, knowing it does not match the desired hypery11 feature behavior.

### Step 7: Tuck and reassemble

Place the board and DC/DC converter behind the footwell panel. Snap the trim panel back into place. No visible wires.

## Important Notes

> **Termination resistor:** You must cut the **TERM** jumper on the Feather RP2040 CAN before connecting. Failing to do so will cause CAN bus communication errors.

> **Car must be off during wiring:** Turn the car off and wait 8–10 minutes for all CAN buses and relays to shut down before plugging/unplugging connectors.

> **FSD on HW4 + firmware 2026.2.9.x – 2026.14.x:** CAN-injection cannot *grant* FSD entitlement on these firmware versions — the AP ECU verifies VIN-level entitlement against Tesla servers independently of CAN. For VINs that **already** have a paid FSD entitlement but are region-locked (e.g. Taiwan), `0x3FD` bit injection can still unlock the regional UI gate — at the risk of triggering a server-side VIN ban (the April 2026 ban wave specifically targeted these region-unlock users; ban persists across account transfers and re-subscriptions). Nag suppression is mechanically separate from this entitlement/region gate, but it is still an active CAN modification with nonzero detection and safety risk.

## Pre-Install Checklist

- [ ] 6-pin breakout adapter mated with Gen 2 cable's Commander connector (cable left intact)
- [ ] Adapter orientation verified with multimeter: red ≈ 12V, blue → GND
- [ ] CAN pair grouping verified: green↔white ≈ 60Ω, yellow↔black ≈ 60Ω (car off 8–10 min)
- [ ] TERM jumper cut on Feather RP2040 CAN board
- [ ] Firmware path selected: ESP32 hypery11 now, Flipper hypery11 now, Feather hypery11 port, or stock Feather fallback
- [ ] Target CAN pair verified by sniffing for `0x370`, `0x39B`, `0x3FD`, and ideally `0x7FF`
- [ ] Red wired to MP1584EN IN+, Blue to IN−
- [ ] MP1584EN OUT+/OUT− connected to Feather (via USB-C breakout or direct solder)
- [ ] Unused CAN pair left disconnected and insulated
- [ ] Board wrapped in heat shrink tubing, wire exits sealed
- [ ] Car fully powered off for 8–10 minutes before plugging in

## Firmware Target

Selected permanent-install hardware: **Adafruit Feather RP2040 CAN** with MCP2515, TERM jumper cut.

Desired firmware behavior: **hypery11 CAN logic** for:

| Goal | CAN surface | Default stance |
|---|---|---|
| Nag suppression | `0x370` EPAS echo, ideally gated by `0x39B` DAS hands-on state | First active feature after listen-only validation |
| FSD region unlock | `0x3FD` HW4 bit 46 + bit 60 | Deferred until wiring, nag-only operation, and ban-defense behavior are proven |
| Telemetry disable | `0x3F8` bit 43 | Disabled by default; research/evaluation only |

Current firmware status:

| Path | Status | Use |
|---|---|---|
| Flipper Zero + Electronic Cats CAN Add-On | Supported by hypery11 now | Reference/upstream path |
| ESP32 + CAN hardware | Supported by hypery11 now | Best for web UI, bus discovery, and practical install |
| Feather RP2040 CAN | Port pending | Best physical fit only after hypery11 behavior is ported |
| Stock `ev-open-can-tools` Feather build | Builds UF2 now | Fallback only; not equivalent to hypery11 |

Required runtime posture:

- Boot or start in listen-only mode first.
- Keep Telemetry Off disabled for normal use.
- On 2026.14.x, use AP-First behavior before any `0x3FD` region-unlock injection.
- If Ban Shield is used, let it learn a healthy `0x7FF` baseline before enabling region unlock.

## Sources

- [FSD CAN Mod — Active Repos Tracker](https://fsdcanmod.com/#active-repos)
- [EV Open Can Mod](https://github.com/ev-open-can-tools/ev-open-can-tools)
- [hypery11 Flipper Zero CAN Mod](https://fsdcanmod.com/repos/hypery11-flipper-zero/)
- [Enhance Auto — Gen 2 Cable](https://www.enhauto.com/products/tesla-gen-2-cable)
- [Enhance Auto — X179 Installation Video](https://youtube.com/watch?v=ifwJNZgykVI)
- [Adafruit — Feather RP2040 CAN Documentation](https://learn.adafruit.com/adafruit-rp2040-can-bus-feather)
- [Tesla — Model 3 Electrical Reference](https://service.tesla.com/docs/Model3/ElectricalReference/)

---
*Generated for Jay's 2024 Model 3 Highland (HW4) · Firmware 2026.14.3 (observed 2026-05) · April 2026*
