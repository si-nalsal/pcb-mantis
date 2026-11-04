# Design status

**Snapshot:** 2026-11-04 20:58 WIB · `kicad-cli` 10.0.6 · branch `main` @ `05c2c5a`
**Routing state: `unconnected_items` = 0 · schematic parity = 0 — every functional net is connected.**

Everything on this page was produced by running the commands shown, not estimated.
Re-run them after each milestone — the numbers below are a baseline, not a promise.

```bash
kicad-cli sch erc   --severity-all --format json -o /tmp/svayam_erc.json  SvayamV3_IIK.kicad_sch
kicad-cli pcb drc   --severity-all --schematic-parity --format json \
                    -o /tmp/svayam_drc.json SvayamV3_IIK.kicad_pcb
kicad-cli pcb export stats --units mm -o /tmp/svayam_stats.txt SvayamV3_IIK.kicad_pcb
kicad-cli sch export bom --fields "Reference,Value,Footprint,MPN,LCSC" \
                    --group-by "Value,Footprint" -o /tmp/svayam_bom.csv SvayamV3_IIK.kicad_sch
```

## Stage summary

| Stage | State | Evidence (2026-11-04) |
|---|---|---|
| Schematic capture | ✅ complete, frozen | 290 components / 263 nets |
| ERC | ✅ **0 errors**, 17 warnings | 9 `lib_symbol_mismatch` + 8 `lib_symbol_issues` — all library hygiene, no design error |
| Footprint assignment | ✅ complete | PCB ↔ schematic parity: **0 issues** |
| PCB placement | ✅ finalised | 290 footprints (stats: 195 front / 95 back) |
| Routing | ✅ every functional net connected | 2,918 segments / 424 vias; **204 of 263 nets carry copper**; the other 59 are single-pin `unconnected-(…)` NC nets; `unconnected_items` = **0** |
| DRC | 🔴 **579 findings** (174 error / 405 warning), 2 of them shorts | breakdown below |
| Silkscreen polish | 🔴 pending | 199 `silk_overlap` + 62 `silk_over_copper` + 4 text-rule findings |
| Fabrication release | ⛔ not started | no gerbers published, by design |

## Board metrics (`pcb export stats`)

| Metric | Value |
|---|---:|
| Outline (Edge.Cuts `gr_rect`) | **120.925 × 89.960 mm** (X 41.375–162.300 · Y 94.800–184.760) |
| Stackup thickness | 1.5584 mm (6 layers) |
| Min track width | 0.1000 mm |
| Min track clearance | 0.0899 mm |
| Min drill diameter | 0.1500 mm |
| Pads | 59 TH · 999 SMD · 6 NPTH |
| Through vias | 424 (304 × Ø0.30 · 73 × Ø0.20 · 47 × Ø0.15) |

## Routing state

| Metric | Value |
|---|---:|
| Track segments | 2,918 |
| — by layer | F.Cu 1,639 · B.Cu 746 · In2.Cu 400 · In3.Cu 133 |
| Vias (through-hole) | 424 |
| Copper zones | 8 |
| Nets with pads | 263 |
| Nets with copper | 204 / 263 |
| Nets with no copper | 59 — all `unconnected-(…)` single-pin NC nets |
| Functional nets without copper | **0** |
| Missing connections (`unconnected_items`) | **0** ✅ |

Two `SW1`/`SW2` switches (`PS-5850SVA-3PL` / `-3PN`, footprint
`easyeda2kicad:SW-SMD_G-SWITCH_PS-5850SVX-3PL`) were added on 30 Oct; `SW1` is the
BOOT0 latch, `SW2` the momentary NRST. Their anchor pads are conductive but unassigned,
so they add no nets.

## DRC findings by rule (`pcb drc`, 579 total)

| Rule | Count | Real? | Action |
|---|---:|---|---|
| `silk_overlap` | 199 | no | cosmetic — clear with **Autoplace Fields** at the end |
| `lib_footprint_mismatch` | 119 | no | footprint cache drift vs library, geometry unaffected |
| `track_width` | 97 | ⚠️ rule gap | 0.10 mm tracks **outside** the `U1_BGA` rule area vs global 0.127 mm (was 199) |
| `silk_over_copper` | 62 | no | cosmetic |
| `clearance` | 26 | ⚠️ | tightest is BGA fan-out, 0.0899 mm actual (was 28) |
| `hole_clearance` | 15 | ⚠️ | track-to-hole below rule (was 22) |
| `solder_mask_bridge` | 14 | ⚠️ DFM | mask aperture bridging at tight pad pairs |
| `track_dangling` | 14 | ⚠️ | unterminated track ends (was 10) |
| `hole_to_hole` | 11 | ⚠️ | drill-to-drill below rule (was 14) |
| `via_dangling` | 6 | ⚠️ | vias connected on one layer only (was 43) |
| `starved_thermal` | 4 | ⚠️ | zone spokes below minimum — refill and re-check (was 19) |
| `drill_out_of_range` | 3 | 🔴 DFM | 0.15 mm fan-out vias outside the `U1_BGA` rule area (was 6) |
| `text_height` / `text_thickness` | 3 / 1 | no | silk text below rule minimum |
| `courtyards_overlap` | 2 | ⚠️ | 2 pairs inside connector courtyards (was 7) |
| `shorting_items` | 2 | 🔴 | see below |
| `isolated_copper` | 1 | ⚠️ | zone `+3V3_FMU` on `In3.Cu` has an island — connect or delete |
| `unconnected_items` | **0** | ✅ | every functional net fully connected |
| `holes_co_located` | 0 | ✅ | cleared (was 1) |

### The two remaining shorts

Both are the same class and are **not live shorts** — the `MP` pad carries no net, so no
current flows where it should not. They become a hard **+5 V / GND short the moment `MP`
is assigned to `GND`**, so move the via first, assign second.

| # | Pad | Via inside the pad | Risk |
|---|---|---|---|
| 1 | `J18` `MP` (B.Cu) | `GND` via, F.Cu→B.Cu | ⚠️ tying `MP` → `GND` closes the short |
| 2 | `J13` `MP` (B.Cu) | `GND` via, F.Cu→B.Cu | as #1 |

The three earlier `J17` / `J11` shorts were cleared on 2026-10-25 (commit `a63baa4`) by
moving the vias out of the pad area, and the genuine signal short (a `GND` via on the
`SPI1_MOSI` track on `In2.Cu`) was cleared the same day.

Pad-level audit (`--schematic-parity`, netlist comparison):

* **0** pads carry a net in the schematic but no net on the PCB.
* **0** functional nets are without copper.
* **55** pads carry no net at all, of which 6 are NPTH (`H1`–`H4`, `J1` shell) and **49 are
  conductive**: 28 × `MP` (14 JST-GH connectors × 2), 4 × `J10` shield pads (11–14), 17 ×
  unnamed anchor pads on `U3` (4), `U4` (4) and `U11` (9).
  The `J10` shield pads are the only ones worth revisiting — an ungrounded SD shield has no
  ESD return path.

## History — unroute global run #3 (2026-10-30 13:54) — **superseded**

On user request, all global inter-component routes were removed on every copper layer so
components could be re-arranged, while local connections and their vias were kept
(decision log **D-054/D-055/D-056**). That run left 1,328 segments / 347 vias.

**That is no longer the working state:** the board in this snapshot carries **2,918
segments / 424 vias**, so it has been re-routed (or restored) since. The section is kept
only to document the procedure and its rollback path.

| Rule | Value (run #3) |
|---|---|
| Removed | connected track groups > 12.0 mm — 79 groups, 1,185 segments |
| Removed per layer | F.Cu −639 · In2.Cu −275 · B.Cu −182 · In3.Cu −89 |
| Removed vias | 109 |
| Kept | 304 local groups (≤ 12 mm) + `USB_D±`; 0 groups > 12 mm remained |
| Zones | 8 pours untouched |

Rollback for that run: `SvayamV3_IIK.kicad_pcb.bak_unroute_20261030_135451` (byte-exact
pre-unroute copy), or `git restore SvayamV3_IIK.kicad_pcb` back to `a63baa4`.

## Drill capability

DRC is written against a **0.20 mm minimum drill** rule, but **47 fan-out vias use
0.15 mm**. The 3 `drill_out_of_range` findings are the ones outside the `U1_BGA` rule
area; the rest are waived by that area's `hole_size (min 0.15 mm)`.

Before ordering: confirm the fab tier supports 0.15 mm mechanical drills on a 6-layer
stack, or bump the fan-out vias to 0.20 mm. Cost/tier question, not a KiCad question.

## Symbol-library state — the reason for the 8 `lib_symbol_issues`

`sym-lib-table` currently registers **two** entries that cannot both be satisfied:

| Entry | URI | State |
|---|---|---|
| `Svayam_Symbol` | `${KIPRJMOD}/Library/Svayam/Svayam_Symbol` | ❌ path does not exist — no `Svayam_Symbol.kicad_sym` in the repo |
| `Svayam` | `${KIPRJMOD}/Library/Svayam/Svayam.kicad_sym` | ✅ exists, but holds **only** `PS-5850SVA-3PL`, `PS-5850SVA-3PN`, `PS-5824SVB-6PL` |

So the 8 ERC warnings resolve against `Svayam`:

```
7 × [lib_symbol_issues] not found in symbol library 'Svayam'
      FM25CL64B · BMP388 · BMI088 · BMI270 · TPS2120YFPR · RT9080-33GJ5 ×2
1 × [lib_symbol_issues] Symbol 'tps63070' not found in symbol library 'tps63070'
      (the tps63070 entry points at Library/tps63070.pretty — a footprint dir)
```

Symbols are embedded in the `.kicad_sch`, so plots and BOMs are correct. The durable fix
is to put the 7 sensor/regulator symbols into `Svayam.kicad_sym` (or drop the dead
`Svayam_Symbol` entry and re-home them), then repair the `tps63070` entry:

```diff
-(lib (name "tps63070") (type "KiCad") (uri "${KIPRJMOD}/Library/tps63070.pretty") ...)
+(lib (name "tps63070") (type "KiCad") (uri "${KIPRJMOD}/Library/Svayam/Svayam_Symbol/TPS63070RNMR.kicad_sym") ...)
```

Do **not** follow this with *Tools → Update Symbols from Library* — that path previously
exploded this project from 56 to 249 warnings.

## BOM readiness (`sch export bom`)

| Metric | Value |
|---|---:|
| BOM line items (grouped by value + footprint) | 88 |
| Component references covered | 277 |
| Excluded by `in_bom no` | 13 — `H1`–`H4` (mounting holes) + `R93`–`R101` (9 × 10k in `connector.kicad_sch`) |
| Line items with an MPN | 39 |
| Line items **without** an MPN | **49** |
| Line items with an LCSC part number | **2** — `SW1` C5155594 · `SW2` C5155595 |

The `MPN` field is populated for the active parts; passives are still blank. `LCSC` is now
used on the two switches — filling it for the remaining 86 lines is what turns this BOM
into a one-click JLCPCB order. **Decide whether `R93`–`R101` should really be out of the
BOM** — they are populated-class 10k resistors sitting next to connectors that *are* in
the BOM.

## Next steps, in order

1. **Move the 2 vias out of the `J18` and `J13` `MP` pads** — see the shorts table.
2. **Assign `MP` → `GND`** on the JST-GH connectors, and consider `J10` shield pads 11–14.
   Do this only after step 1.
3. **Place `H1`–`H4`** — the four M3 mounting holes are still parked at X = 223.775 mm,
   outside the board outline (right edge 162.300 mm).
4. **Enlarge the `U1_BGA` rule area** so it covers the full escape region, or raise the
   global minimum track width to 0.10 mm. Re-run DRC.
5. **Refill zones**, re-check `starved_thermal` (4) and the `+3V3_FMU` `isolated_copper`.
6. **Silkscreen pass** — Autoplace Fields, then fix `silk_over_copper` manually. Do this last.
7. **Decide the drill tier** — keep 0.15 mm fan-out vias (advanced/costlier) or bump to 0.20 mm.
8. **Merge the symbol libraries** (`Svayam_Symbol` vs `Svayam`) and fill the 49 missing MPNs
   plus the `LCSC` column.
9. **Then, and only then**, export gerbers with the commands in
   [`fabrication/README.md`](../fabrication/README.md).
