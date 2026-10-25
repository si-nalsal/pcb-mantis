# Design decisions and rationale

Every decision below was made against a named reference — a datasheet, an application note,
an industry standard, or the PCB Design Tutorial Rev. A. Nothing here is an unverified
assumption. Where a number is a calculation, the inputs are shown so it can be re-checked.

---

## 1. Power input: two batteries, one mux

`POWER1` (`J9`) and `POWER2` (`J16`) each feed `U10` (TPS2120), a priority power mux with
automatic switchover. Rationale: a flight controller should survive losing one battery lead
mid-flight; the mux does the hand-over in hardware without firmware involvement.

`U10`'s current limit is set by `R24` = 33 kΩ.

## 2. 5 V rail: buck-boost, not buck

`U11` (TPS63070) is a buck-boost converter, chosen because the input (`+VBRICK1/2`) can sit
above *or* below 5 V depending on cell count and state of charge. The feedback divider
`R27`/`R28` is set for **4.99 V**. `R25` is marked DNP — it exists only as a tuning option.

Layout rule applied (TI SLVA139, "layout for SMPS"): input and output capacitors sit as close
to the IC pins as the footprint allows, and the switching loop area is minimised by keeping
the inductor adjacent to `U11`.

## 3. Nine protected peripheral ports

Every external connector passes through its own TPS2553-Q1 current-limited switch
(`U16`–`U24`). A shorted GPS or telemetry cable must not take down the 5 V rail.

Trip current is set per port by `RILIM` (TPS2553-Q1 datasheet, `ILIM = K / RILIM`):

| Port | Limiter | `RILIM` | Trip window |
|---|---|---|---:|
| GPS | `U22` | R90 = 56 kΩ | 422 – 522 mA |
| TELEM1 | `U16` | R84 = 33 kΩ | 723 – 859 mA |
| TELEM2 | `U17` | R85 = 33 kΩ | 723 – 859 mA |
| SERIAL4 | `U18` | R86 = 75 kΩ | 314 – 397 mA |
| SERIAL5 | `U19` | R87 = 75 kΩ | 314 – 397 mA |
| I2C | `U20` | R88 = 110 kΩ | 213 – 277 mA |
| CAN | `U21` | R89 = 47 kΩ | 505 – 616 mA |
| CAN2 | `U24` | R92 = 47 kΩ | 505 – 616 mA |
| SPI | `U23` | R91 = 110 kΩ | 213 – 277 mA |

`R89`/`R92` were changed from 75 kΩ to 47 kΩ so a passive CAN splitter (two 120 Ω
terminations plus stub) cannot trip the limiter.

## 4. `+VSERVO` is fail-last

The servo rail has **no on-board source** and **no series fuse**. Power is fed externally, the
way Pixhawk-class controllers do it.

Reasoning: if a BEC fails, a rail that is *sourced* by the flight controller on the same
copper as the servo bank gives the failure a path into the FMU. Feed-only topology keeps the
failure on the airframe side. Protection that *is* present: `D8` (SMAJ6.0A TVS) plus
`C78` (220 µF bulk) and 16 × 1 µF per channel.

Design caveat, recorded deliberately: because the rail has no fuse, a dead-short on a servo
cable is cleared by the external BEC's own protection, not by the board.

## 5. Sensor redundancy, in separate packages

| Quantity | Primary | Secondary |
|---|---|---|
| IMU | `U7` BMI088 | `U8` BMI270 |
| Barometer | `U5` BMP388 | `U6` DPS310XTSA1 |
| Magnetometer | `U9` BMM150 | — |

Two IMUs of *different* silicon means a firmware bug or a part-specific erratum affecting one
model does not blind both. The sensor block gets its own LDO domain (`+3V3_SENS` from `U13`)
and a local copper pour, and is placed away from the switching regulator — the usual
"keep the barometer out of the hot air of the buck converter" rule.

## 6. Silkscreen = function, not value

The assembled board must be repairable years later by someone who did not design it.

* Silkscreen carries **function labels** only: `Channel 1`…`Channel 16`, `TELEM1`, `GPS`,
  `SBUS`, `CAN`, `I2C`, `SPI`, `POWER1`, `POWER2`, `SAFETY`, `BUZZER`, `ADC`.
* **Component values are hidden from silkscreen and moved to `F.Fab`**, so they remain
  readable in the editor and in fab drawings without printing 88 line items on a 125 mm board.
* Every connector reference designator is visible.

## 7. MCU escape (BGA 0.65 mm)

The STM32H743IIK6 is a UFBGA-201 at 0.65 mm pitch. A via cannot be placed on the pad grid, so
the escape is **dog-bone**: vias offset **0.46 mm diagonally** from each pad — half the
diagonal pitch, `sqrt(2) × 0.65 / 2` — which is the geometry recommended by ST's
"Example of PCB design rules (0.65 mm pitch BGA)" table in the STM32H743 datasheet.

Consequence: a dedicated rule area (`U1_BGA`) relaxes track width to 0.10 mm, clearance to
0.12 mm (outer), and drill to 0.15 mm **inside that region only**. Everywhere else the global
rules apply.

## 8. ESD and connector practice

* USB-C (`J1`) is protected by `U15` (USBLC6-2SC6), placed at the connector.
* CAN transceivers `U3`/`U4` (TJA1051TK-3) sit next to their connectors, on the far side of
  the board from the sensor zone.
* Mounting holes `H1`–`H4` are **isolated** (non-plated, no net) rather than tied to GND.
  Rationale: carbon/composite airframes and grounded metal standoffs are both in scope; an
  isolated hole cannot create a ground loop through the frame.
* TVS on both battery inputs (`D4`, `D5`) and the servo rail (`D8`).

## 9. Deliberately not implemented

| Item | Decision |
|---|---|
| Per-rail current monitoring (INA180) | Shelved — telemetry value did not justify board area |
| External watchdog (TPL5010 → NRST) | Proposed, not adopted |
| Per-channel servo ESD | Shelved — the keyed JST-XH connector is considered adequate |
| Series fuse on `+VSERVO` | Rejected — see §4 |
| Split ground planes | Rejected — single solid ground; analog/digital separation is done by placement and the `+3V3_SENS` domain, not by cutting the plane |

---

## Reference list

1. **PCB Design Tutorial, Rev. A** — David L. Jones. Track width/current table, copper pour
   and silkscreen practice. <https://www.alternatezone.com/electronics/files/PCBDesignTutorialRevA.pdf>
2. **IPC-2221B** — Generic Standard on Printed Board Design (conductor spacing, current
   capacity).
3. **STM32H743II datasheet** — STMicroelectronics. UFBGA-201 mechanical data, 0.65 mm pitch
   PCB design rules table.
4. **TPS2553-Q1 datasheet** — Texas Instruments. `RILIM` → trip current relationship.
5. **TPS63070 datasheet** — Texas Instruments. Feedback divider and buck-boost layout.
6. **TPS2120 datasheet** — Texas Instruments. Power mux operation and `ILM` selection.
7. **SLVA139** — Texas Instruments, "AN-1149 Layout Guidelines for Switching Power Supplies".
8. **JST XH series datasheet** — 3 A per contact rating (AWG22), used to size servo stubs.
9. **BMI088 / BMI270 / BMP388 / DPS310 / BMM150** — Bosch Sensortec datasheets, layout
   recommendations for the sensor zone.
