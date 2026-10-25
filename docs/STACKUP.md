# Layer stackup

Source of truth: `SvayamV3_IIK.kicad_pcb` → `(setup (stackup …))`. The values below are
copied from the board file, not from a fab calculator, and sum to the `(thickness 1.5584)`
stored in `(general)`.

Target process: **JLCPCB 6-layer, `JLC06161H-3313`, 1.6 mm nominal, ENIG**.

## Physical stack (top → bottom)

| # | Layer | Function | Material | Thickness | εr | tan δ |
|---|---|---|---|---:|---:|---:|
| 1 | `F.SilkS` | Top silkscreen | — | — | — | — |
| 2 | `F.Paste` | Top stencil | — | — | — | — |
| 3 | `F.Mask` | Top solder mask | — | 0.010 mm | — | — |
| 4 | `F.Cu` | Signal / power / components | Copper | 0.035 mm (1 oz) | — | — |
| 5 | dielectric 1 | Prepreg | 3313 | 0.0994 mm | 4.10 | 0.00 |
| 6 | `In1.Cu` | Plane (GND) | Copper | 0.0152 mm (0.5 oz) | — | — |
| 7 | dielectric 2 | Core | FR4 | 0.550 mm | 4.60 | 0.02 |
| 8 | `In2.Cu` | Signal / routing | Copper | 0.0152 mm | — | — |
| 9 | dielectric 3 | Prepreg | 2116 | 0.1088 mm | 4.16 | 0.02 |
| 10 | `In3.Cu` | Signal / routing | Copper | 0.0152 mm | — | — |
| 11 | dielectric 4 | Core | FR4 | 0.550 mm | 4.60 | 0.02 |
| 12 | `In4.Cu` | Plane (GND) | Copper | 0.0152 mm | — | — |
| 13 | dielectric 5 | Prepreg | 3313 | 0.0994 mm | 4.10 | 0.02 |
| 14 | `B.Cu` | Signal / power / components | Copper | 0.035 mm (1 oz) | — | — |
| 15 | `B.Mask` | Bottom solder mask | — | 0.010 mm | — | — |
| 16 | `B.Paste` | Bottom stencil | — | — | — | — |
| 17 | `B.SilkS` | Bottom silkscreen | — | — | — | — |
| | **Total** | | | **1.5584 mm** | | |

Board setup also stores `(copper_finish "ENIG")` and `(dielectric_constraints yes)`, so KiCad's
impedance calculator uses these εr / thickness values rather than generic FR4 defaults.

## Layer roles

| Layer | Role |
|---|---|
| `F.Cu` | Component side — BGA escape, most signal routing, servo rail pour |
| `In1.Cu` | GND plane (uninterrupted under the MCU and the sensor zone) |
| `In2.Cu` | Horizontal signal routing |
| `In3.Cu` | Vertical signal routing |
| `In4.Cu` | GND plane / return path |
| `B.Cu` | Back-side components, remaining signals, `+VSERVO` distribution |

Rule of thumb applied in this design: never route across a plane split; every high-speed net
has an unbroken reference plane on the layer directly below it.

## Trace width and current

IPC-2221 basis, 1 oz outer copper, 10 °C rise, with 30–50 % margin added for manufacturing
tolerance (reference: PCB Design Tutorial Rev. A, track-width table, and IPC-2221 §6.2):

| Current | 1 oz outer | 1 oz inner | 2 oz outer |
|---|---:|---:|---:|
| 0.5 A | 0.15 mm | 0.25 mm | 0.10 mm |
| 1.0 A | 0.30 mm | 0.50 mm | 0.20 mm |
| 2.0 A | 0.70 mm | 1.20 mm | 0.40 mm |
| 3.0 A | 1.20 mm | 2.00 mm | 0.70 mm |
| 5.0 A | 2.50 mm | 4.00 mm | 1.50 mm |

Applied to this board:

| Rail | Design width | Rationale |
|---|---|---|
| `+VSERVO` (rail) | **copper zone**, not a trace | 16 channels × 2 A stall cannot be carried by any sane trace; the connector shell and the external BEC are the real limits |
| Servo channel stub (V+) | 1.0 mm default, 1.5–1.8 mm if high-torque | 2 A basis + margin; JST-XH contact rating is 3 A (AWG22) |
| GND return | equal to the V+ stub | return path must match the source |
| `+5V_IN`, `VIN_BB`, `+5V_PERIPH` | 0.7 mm (`PWR_MAIN` / `PWR`) | ≈2 A continuous, power-budget load is ~570 mA |
| PWM / signal | 0.15–0.25 mm | no current requirement |
| BGA fan-out | 0.10 mm | escape only, inside the `U1_BGA` rule area |

## Net classes (from `SvayamV3_IIK.kicad_pro`)

| Class | Clearance | Track | Via (dia / drill) |
|---|---:|---:|---:|
| `Default` | 0.15 mm | 0.15 mm | 0.70 / 0.30 mm |
| `GND` | 0.20 mm | 0.20 mm | 0.70 / 0.30 mm |
| `PWR` | 0.20 mm | 0.30 mm | 0.70 / 0.30 mm |
| `PWR_MAIN` | 0.20 mm | 0.70 mm | 1.00 / 0.50 mm |
| `CAN` | 0.20 mm | 0.20 mm | 0.70 / 0.30 mm |
| `USB` | 0.20 mm | 0.20 mm, diff pair 0.20 / 0.25 mm gap | 0.70 / 0.30 mm |

## Custom DRC rules (`SvayamV3_IIK.kicad_dru`)

* Global: track width ≥ 0.127 mm (outer), ≥ 0.09 mm (inner); track spacing 0.127 mm (outer),
  0.09 mm (inner).
* Rule area **`U1_BGA`**: 0.10 mm tracks, 0.12 mm clearance (outer) / 0.13 mm (inner),
  0.15 mm hole, hole-to-hole 0.45 mm, track-to-hole 0.20 mm — the 0.65 mm-pitch UFBGA
  fan-out cannot be escaped under the global rules.
* Last (highest priority) rule: relaxed clearance for `PWR`/`PWR_MAIN` inside the BGA area.

**Known gap:** the `U1_BGA` rule area does not cover the whole escape region, so DRC still
flags 199 fan-out tracks at 0.10 mm against the global 0.127 mm rule. See
[STATUS.md](STATUS.md).
