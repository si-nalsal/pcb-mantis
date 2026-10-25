# Fabrication

**This folder intentionally contains no gerbers yet.** The board is not release-ready — see
[`docs/STATUS.md`](../docs/STATUS.md). Publishing gerbers for a revision with 6 DRC shorts
would be worse than publishing nothing.

## Release gate

All of the following must be true before any file in this folder is produced:

| # | Gate | Check |
|---|---|---|
| 1 | `kicad-cli pcb drc` reports **0 errors** | warnings reviewed individually |
| 2 | `kicad-cli sch erc` still reports **0 errors** | 17 known cosmetic warnings accepted |
| 3 | PCB ↔ schematic parity is 0 | `--schematic-parity` |
| 4 | Every BOM line has an MPN and an LCSC number | `docs/bom.csv` |
| 5 | Drill sizes and min track/space match the chosen fab tier | `kicad-cli pcb export stats` |
| 6 | Mounting holes, courtyards and board outline reviewed in 3D | Pcbnew 3D viewer |
| 7 | Git tag cut for the released revision | `git tag -a v1.0-rc1` |

## Export commands

Run from the repository root, output into this folder:

```bash
# Gerbers — all copper, mask, silk, and the board outline
kicad-cli pcb export gerbers \
  --layers F.Cu,In1.Cu,In2.Cu,In3.Cu,In4.Cu,B.Cu,F.Paste,B.Paste,F.Silkscreen,B.Silkscreen,F.Mask,B.Mask,Edge.Cuts \
  --no-x2 --subtract-soldermask \
  -o ./fabrication/gerbers/ SvayamV3_IIK.kicad_pcb

# Drill files (Excellon, with plated/non-plated split)
kicad-cli pcb export drill \
  --format excellon --excellon-separate-th --generate-map --map-format gerberx2 \
  -o ./fabrication/gerbers/ SvayamV3_IIK.kicad_pcb

# Pick-and-place, one file per side
kicad-cli pcb export pos --side front --format csv --units mm \
  -o ./fabrication/SvayamV3_IIK-pos-front.csv SvayamV3_IIK.kicad_pcb
kicad-cli pcb export pos --side back  --format csv --units mm \
  -o ./fabrication/SvayamV3_IIK-pos-back.csv  SvayamV3_IIK.kicad_pcb

# BOM
kicad-cli sch export bom \
  --fields "Reference,Value,Footprint,MPN,LCSC" \
  --labels "Refs,Value,Footprint,MPN,LCSC" \
  --group-by "Value,Footprint" \
  -o ./fabrication/bom.csv SvayamV3_IIK.kicad_sch

# IPC-D-356 netlist (useful for bare-board electrical test)
kicad-cli pcb export ipcd356 -o ./fabrication/SvayamV3_IIK.d356 SvayamV3_IIK.kicad_pcb

# Mechanical: STEP model for enclosure work
kicad-cli pcb export step --subst-models -o ./fabrication/SvayamV3_IIK.step SvayamV3_IIK.kicad_pcb
```

## Intended fab process

| Parameter | Choice |
|---|---|
| House | JLCPCB |
| Layers | 6 |
| Stackup | `JLC06161H-3313` (see [`docs/STACKUP.md`](../docs/STACKUP.md)) |
| Thickness | 1.6 mm nominal (design stores 1.5584 mm) |
| Copper | 1 oz outer / 0.5 oz inner |
| Surface finish | ENIG |
| Solder mask | both sides, tented vias (`tenting front/back = yes` in board setup) |
| Silkscreen | both sides, function labels only |

## Open DFM questions for the fab house

1. **0.15 mm mechanical drills** on the `U1_BGA` fan-out — supported at which tier?
2. **0.10 mm tracks / 0.10 mm spaces** in the BGA escape — within the 6-layer process minimum?
3. **Via tenting** on both sides with no plugging — acceptable, or is via-in-pad/plugging
   required anywhere?
4. **NPTH 3.2 mm mounting holes** — plain, or with a keep-out ring?

Answers belong in this file before the first order is placed.

## What gets committed

`fabrication/**` is whitelisted in `.gitignore` for `*.zip`, `*.gbr`, `*.drl`, `*.csv` and
`*.pdf`, so released artefacts can be committed and tagged. Do not commit intermediate plots
from `output/` — that folder is ignored on purpose.
