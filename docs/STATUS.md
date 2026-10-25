# Design status

**Snapshot:** 2026-10-25 23:28 WIB · KiCad 10.0.6 · branch `main`, commit `a56f437`

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
| Routing | ✅ every net connected | 203 of 262 nets carry copper; the other 59 are single-pin NC nets; `unconnected_items` = **0** |
| DRC | 🔴 **760 findings** (321 error / 439 warning), 5 of them shorts | see breakdown below |
| Silkscreen polish | 🔴 pending | 254 silk findings — do this last, after copper is final |
| Fabrication release | ⛔ not started | no gerbers published, by design |

## Routing state

| Metric | Value |
|---|---:|
| Track segments | 2,897 |
| Vias (through-hole) | 475 |
| Copper zones | 8 |
| Nets with copper | 203 / 262 |
| Nets with no copper | 59 — all `unconnected-(…)` single-pin NC nets |
| Functional nets without copper | **0** |
| Missing connections (`unconnected_items`) | **0** |

Every open connection present in the 20:52 snapshot was closed in the same day's layout
session. Recorded here for traceability — they were: `GND` (`R1` pad 2 ↔ `J9` pad 6,
`C24` pad 2 ↔ via, and a 0.0109 mm track stub), `+3V3_FMU` (track ↔ `U1` pad `K4`, an MCU
`VDD` pin), `+VBRICK1`, `+VBRICK2`, and `SPI1_MISO` between `U7` pads 10 and 15.

## DRC findings by rule

| Rule | Count | Real? | Action |
|---|---:|---|---|
| `track_width` | 199 | ⚠️ rule gap | 0.10 mm tracks **outside** the `U1_BGA` rule area (151 `F.Cu`, 48 `B.Cu`) vs global 0.127 mm |
| `silk_overlap` | 199 | no | cosmetic — clear with **Autoplace Fields** at the end |
| `lib_footprint_mismatch` | 119 | no | footprint cache drift vs library, geometry unaffected |
| `silk_over_copper` | 55 | no | cosmetic |
| `via_dangling` | 43 | ⚠️ | vias connected on one layer only — track stubs to finish |
| `clearance` | 27 | ⚠️ | BGA fan-out, 0.10–0.118 mm actual vs 0.12 mm rule |
| `hole_clearance` | 24 | ⚠️ | track-to-hole 0.30–0.315 mm vs 0.33 mm rule |
| `starved_thermal` | 19 | ⚠️ | zone thermal spokes (some pads get 1 spoke, min is 2) — refill and re-check |
| `solder_mask_bridge` | 17 | ⚠️ | mask aperture bridging at tight pad pairs — DFM risk |
| `hole_to_hole` | 15 | ⚠️ | drill-to-drill below rule |
| `track_dangling` | 10 | ⚠️ | unterminated track ends |
| `courtyards_overlap` | 7 | ⚠️ | `R84`–`R92` inside their connector courtyards |
| `unconnected_items` | 0 | ✅ | every functional net fully connected |
| `drill_out_of_range` | 6 | 🔴 DFM | 0.15 mm fan-out vias vs 0.20 mm rule floor |
| `shorting_items` | 5 | 🔴 | see below |
| `isolated_copper` | 5 | ⚠️ | copper islands with no net — delete or connect |
| `text_height` / `text_thickness` | 3 / 1 | no | silk text below rule minimum |
| `annular_width` | 2 | ⚠️ | `SW1` PTH pads with 0 annular ring |
| `padstack` | 2 | ⚠️ | pad stack geometry |
| `holes_co_located` | 1 | 🔴 | two holes at the same coordinate |
| `silk_edge_clearance` | 1 | no | cosmetic |

### The five shorts — fix these first

All five share one root cause: a via placed **inside** an `MP` (mechanical) pad of a JST-GH
connector. The `MP` pads carry no net, so the pad and the via count as two different
conductors.

| # | Pad | Via inside the pad | Risk |
|---|---|---|---|
| 1 | `J17` MP left (81.46, 134.73) | `+5V_PERIPH` Ø0.6 at (81.605, 134.725) | ⚠️ tying `MP` to GND would short **+5 V to GND** |
| 2 | `J17` MP left | `+5V_PERIPH` Ø0.7 at (81.83, 136.10) | as #1 |
| 3 | `J17` MP right (88.91, 134.73) | `GND` Ø0.7 at (89.0525, 135.60) | solder wicking through the pad |
| 4 | `J17` MP right | `GND` Ø0.7 at (89.155, 135.10) | as #3 |
| 5 | `J11` MP (104.92, 146.07) | `GND` Ø0.6 at (104.936, 146.071), concentric | via dead-centre in the anchor pad |

**Order matters:** move the five vias out of the pad area (each pad is 1 × 2.8 mm) *first*, and
only then assign `MP` → `GND`. Assigning first turns #1 and #2 into a hard +5 V ↔ GND short.

The genuine signal short found earlier — a `GND` via sitting on the `SPI1_MOSI` track on
`In2.Cu` — was cleared on 2026-10-25: the via moved from (111.916, 111.574) to
(111.500, 111.550), leaving **+0.344 mm** clearance to the track, with the track intact
(14 segments, 2 pads, net still fully connected).

Pad-level audit of the same snapshot (`--schematic-parity`, netlist comparison):

* **0** pads carry a net in the schematic but no net on the PCB.
* **0** functional nets are without copper.
* 49 conductive pads carry no net at all, all mechanical: 28 × `MP` (JST-GH), 6 × NPTH
  (`H1`–`H4`, `J1` shell), 4 × `J10` shield pads (11–14), 11 × `SW1`/`SW2` anchor pads.
  The `J10` shield pads are the only ones worth revisiting — an ungrounded SD shield has no
  ESD return path.

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

1. **Move the 5 vias out of the `MP` pads** — `J17` (both pads) and `J11`; see the shorts table.
2. **Assign `MP` → `GND`** on the JST-GH connectors, and consider `J10` shield pads 11–14.
   Do this only after step 1.
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
