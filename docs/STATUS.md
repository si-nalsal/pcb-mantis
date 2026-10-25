# Design status

**Snapshot:** 2026-10-25 · KiCad 10.0.6 · branch `master`, commit at time of writing 15d59f5

Everything on this page was produced by running the commands shown, not estimated.
Re-run them after each milestone — the numbers below are a baseline, not a promise.

```bash
kicad-cli sch erc   --severity-all -o /tmp/erc.rpt SvayamV3_IIK.kicad_sch
kicad-cli pcb drc   --severity-all --schematic-parity -o /tmp/drc.rpt SvayamV3_IIK.kicad_pcb
kicad-cli pcb export stats -o /tmp/stats.txt SvayamV3_IIK.kicad_pcb
kicad-cli sch export netlist --format kicadxml -o /tmp/net.kicadxml SvayamV3_IIK.kicad_sch
```

## Stage summary

| Stage | State | Evidence |
|---|---|---|
| Schematic capture | ✅ complete, frozen | 290 components / 262 nets / 1,005 nodes |
| ERC | ✅ **0 errors**, 17 warnings | all 17 are library-symbol cosmetics |
| Footprint assignment | ✅ complete | PCB ↔ schematic parity: 0 issues |
| PCB placement | ✅ finalised | 290 footprints (190 front / 100 back) |
| Routing | 🟡 functionally complete | 203 of 262 nets carry copper; the other 59 are single-pin NC nets — but **7 connections still have gaps** |
| DRC | 🔴 **765 findings**, 6 of them shorts | see breakdown below |
| Silkscreen polish | 🔴 pending | 254 silk findings — do this last, after copper is final |
| Fabrication release | ⛔ not started | no gerbers published, by design |

## Routing state

| Metric | Value |
|---|---:|
| Track segments | 2,882 |
| Vias (through-hole) | 475 |
| Copper zones | 8 |
| Nets with copper | 203 / 262 |
| Nets with no copper | 59 — all `unconnected-(…)` single-pin NC nets |
| Missing connections (`unconnected_items`) | **7** |

The seven gaps are not "forgotten nets" — they are short track-to-track, pad-to-via and
track-to-pad breaks at joints:

| # | Net | Between |
|---|---|---|
| 1 | `GND` | pad 2 of `R1` ↔ pad 6 of `J9` |
| 2 | `GND` | pad 2 of `C24` ↔ via |
| 3 | `GND` | track 0.0109 mm ↔ track 1.30 mm (`F.Cu`) |
| 4 | `+3V3_FMU` | track on `B.Cu` ↔ pad `K4` of `U1` |
| 5 | `+VBRICK1` | track ↔ track (`F.Cu`) |
| 6 | `+VBRICK2` | track ↔ track (`F.Cu`) |
| 7 | `SPI1_MISO` | track ↔ track (`F.Cu`) |

Fix: select the two endpoints in Pcbnew and use **Route → Finish / drag the joint**; each is
sub-millimetre work.

## DRC findings by rule

| Rule | Count | Real? | Action |
|---|---:|---|---|
| `track_width` | 199 | ⚠️ rule gap | 0.10 mm tracks **outside** the `U1_BGA` rule area (151 `F.Cu`, 48 `B.Cu`) vs global 0.127 mm |
| `silk_overlap` | 199 | no | cosmetic — clear with **Autoplace Fields** at the end |
| `lib_footprint_mismatch` | 119 | no | footprint cache drift vs library, geometry unaffected |
| `silk_over_copper` | 55 | no | cosmetic |
| `via_dangling` | 46 | ⚠️ | vias connected on one layer only — track stubs to finish |
| `clearance` | 27 | ⚠️ | BGA fan-out, 0.10–0.118 mm actual vs 0.12 mm rule |
| `hole_clearance` | 24 | ⚠️ | track-to-hole 0.30–0.315 mm vs 0.33 mm rule |
| `starved_thermal` | 19 | ⚠️ | zone thermal spokes (some pads get 1 spoke, min is 2) — refill and re-check |
| `solder_mask_bridge` | 17 | ⚠️ | mask aperture bridging at tight pad pairs — DFM risk |
| `hole_to_hole` | 15 | ⚠️ | drill-to-drill below rule |
| `track_dangling` | 11 | ⚠️ | unterminated track ends |
| `courtyards_overlap` | 7 | ⚠️ | `R84`–`R92` inside their connector courtyards |
| `unconnected_items` | 7 | 🔴 | see table above |
| `drill_out_of_range` | 6 | 🔴 DFM | 0.15 mm fan-out vias vs 0.20 mm rule floor |
| `shorting_items` | 6 | 🔴 | see below |
| `isolated_copper` | 5 | ⚠️ | copper islands with no net — delete or connect |
| `text_height` / `text_thickness` | 3 / 1 | no | silk text below rule minimum |
| `annular_width` | 2 | ⚠️ | `SW1` PTH pads with 0 annular ring |
| `padstack` | 2 | ⚠️ | pad stack geometry |
| `holes_co_located` | 1 | 🔴 | two holes at the same coordinate |
| `silk_edge_clearance` | 1 | no | cosmetic |

### The six shorts — fix these first

| # | Nets | Where | Cause |
|---|---|---|---|
| 1–4 | `+5V_PERIPH` / `GND` vs *no net* | pad `MP` of `J17`, pad `MP` of `J11` | the JST-GH mechanical pads carry no net, so any via nearby shorts to them |
| 5 | `GND` ↔ `SPI1_MOSI` | via at (111.92, 111.57) ↔ 22.8 mm track on `In2.Cu` | **electrically real short** — the SPI trace passes through a GND via |
| 6 | *no net* ↔ `GND` | pad `MP` of `J11` | as #1–4 |

Two ways out of the `MP` problem:

* assign the mechanical pads to `GND` in the footprint (they are plated through-holes on the
  connector shell — tying them to ground is the intended use), or
* move the offending vias out of the pad radius.

The `SPI1_MOSI` / `GND` short must be re-routed — a GND via sitting on a signal track.

## Drill capability

DRC is written against a **0.20 mm minimum drill** rule, but 56 fan-out vias use **0.15 mm**
(`drill_out_of_range` × 6 are the ones that fall outside the `U1_BGA` rule area; the rest are
waived by that area's `hole_size (min 0.15mm)`).

Before ordering: confirm the fab tier actually supports 0.15 mm mechanical drills on a 6-layer
stack, or bump the fan-out vias to 0.20 mm. This is a cost/tier question, not a KiCad question.

## Known library-table issues

`ERC` reports 9 warnings that are *not* design errors but *are* a portability smell:

```
8 ×  [lib_symbol_issues]  "The current configuration does not include the symbol library 'Svayam'"
1 ×  [lib_symbol_issues]  "Symbol 'tps63070' not found in symbol library 'tps63070'"
```

Cause, in `sym-lib-table`:

1. The custom library is registered as **`Svayam_Symbol`**, but every schematic symbol
   references it as **`Svayam`**.
2. The **`tps63070`** entry points at `Library/tps63070.pretty` — a *footprint* directory,
   not a symbol library file.

Symbols are embedded in the `.kicad_sch`, so plots and BOMs are correct. The fix (not applied
here, because the library configuration is frozen during layout) is two lines in
`sym-lib-table`:

```diff
-(lib (name "Svayam_Symbol") (type "KiCad") (uri "${KIPRJMOD}/Library/Svayam/Svayam_Symbol") ...)
+(lib (name "Svayam")        (type "KiCad") (uri "${KIPRJMOD}/Library/Svayam/Svayam_Symbol") ...)

-(lib (name "tps63070") (type "KiCad") (uri "${KIPRJMOD}/Library/tps63070.pretty") ...)
+(lib (name "tps63070") (type "KiCad") (uri "${KIPRJMOD}/Library/Svayam/Svayam_Symbol/TPS63070RNMR.kicad_sym") ...)
```

Do **not** follow this with *Tools → Update Symbols from Library* — that path previously
exploded this project from 56 to 249 warnings.

## BOM readiness

| Metric | Value |
|---|---:|
| BOM line items (grouped by value + footprint) | 88 |
| Components covered | 290 placements |
| Line items with an MPN | 39 |
| Line items **without** an MPN | **49** |
| Line items with an LCSC part number | 0 |

The `MPN` field is populated in the schematic for the active parts; passives and a few
mechanical parts are still blank, and the `LCSC` field is unused. Filling `LCSC` for all 88
lines is what turns this BOM into a one-click JLCPCB order.

## Next steps, in order

1. **Clear the 6 shorts** (assign `MP` pads of `J11`/`J17` to GND; re-route `SPI1_MOSI`).
2. **Close the 7 gaps** — drag each joint.
3. **Enlarge the `U1_BGA` rule area** so it covers the full escape region, or raise the global
   minimum track width to 0.10 mm. Re-run DRC.
4. **Place `H1`–`H4`** — the four M3 mounting holes are still parked at X = 223.78 mm, outside
   the board outline.
5. **Refill zones**, re-check `starved_thermal` and `isolated_copper`.
6. **Silkscreen pass** — Autoplace Fields, then fix `silk_over_copper` manually. Do this last.
7. **Decide the drill tier** — keep 0.15 mm fan-out vias (advanced/costlier) or bump to 0.20 mm.
8. **Fill the 49 missing MPNs** and the whole `LCSC` column.
9. **Then, and only then**, export gerbers with the commands in
   [`fabrication/README.md`](../fabrication/README.md).
