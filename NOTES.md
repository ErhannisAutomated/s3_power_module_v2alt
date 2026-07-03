# power_module v2-alt — session resume notes

Living document; updated at the end of each session so the next session
can pick up from a cold start (after Claude Code context compaction).
Same convention as v1's `../power_module/NOTES.md`.

## RESUME HERE — end of session 2026-07-04

**Two things happened today.** (1) `bms.kicad_sch` populated (27
components, 21 nets — BQ76920 + 3S cell holder + external FET stack
+ cell-sense R/C/TVS network + I²C header). (2) A new MCP tool
**`autoplace_schematic`** was built in `KiCAD-MCP-Server` because
hand-laying the dense sense-R + differential-cap cluster ran into
label-stub collisions repeatedly. The tool ran end-to-end but with
caveats — see below.

### bms.kicad_sch state

- **27 components placed and wired** (labels + no-connect for U1.11
  NC pin). ERC clean apart from 2 errors: `BAT+` and `GND` global
  labels not driven yet (charger + input sheets will provide/consume).
- **Placement is SPARSE** (~245×155 mm, filling the sheet) rather than
  compact (~46×105 target). Autoplacer's staged-anneal recipe was
  slightly over-spread; the layout is functional but not visually
  polished. Autoplacer refinement is task #61.
- **Same architecture pattern as v1**: `BAT-` (pack pre-Rsns) → `R9`
  (5 mΩ shunt) → BQ76920 VSS+SRN + Q1(DSG).S; Q1.D + Q2.D = `MID`;
  Q2.S = global `GND`. FET Q1/Q2 are 30V/20A_NFET placeholders (same
  as buckboost — pick one MPN that covers both when we lock parts).
- **v2 additions vs v1**: 15V TVS on VC1/VC2/VC3 cell-sense lines
  (D1/D2/D3), 100nF differential cell-sense caps (v1 had 1nF —
  probably too small per TI EVM), 5-pin I²C+ALERT breakout header.
- **buckboost.kicad_sch review** from yesterday still open. User
  looked at it (session start) and said layout will need autoplacer
  before it reads cleanly. Same fix applies once the autoplacer is
  tuned.

### autoplace_schematic MCP tool

- **Files changed**: `KiCAD-MCP-Server/python/kicad_interface.py`
  (`_handle_autoplace_schematic` + dispatch entry) and
  `src/tools/placement.ts` (TS registration). `npm run build` +
  `/mcp` restart to load.
- **Handler wraps** `PLACER.load → set_params → recipe → apply`. Takes
  `dryRun`, `rewire`, `previewPath`, plus compact-profile knobs
  (`repulsionK`, `attractionK`, `polarityK`, `rotationK`,
  `initialTemperature`) and full recipe overrides.
- **Known caveats** (task #61):
  1. Rewire step in `apply` claims success ("34 pairs wired") but
     leaves the file with NO labels/wires. Had to re-wire the bms
     sheet manually with `connect_pins` after autoplace. Symptom on
     disk: 0 wires + 0 labels after apply. Bug likely in
     `rewire_session` — the rewire result dict doesn't reflect what
     ends up in the file.
  2. Layout is looser than the memory's compact_v3 profile suggests
     (`[[feedback-autoplacer-params]]`). Even with the passthrough
     fix, the recipe's spread stage fans components out to fill the
     sheet. Might need lower `repulsionK` or tighter sheet bounds.
  3. **First call sometimes times out**. The 30 s MCP call ceiling
     is enough for the ~330-iter recipe + apply on ~30 components,
     but only just. Reducing iterations (`clusterIters=40,
     spreadStages=5, polarizeStages=1, settleIters=40,
     itersPerStage=5`) keeps it well under.
- **Preview mode**: pass `dryRun=true` + `previewPath` to write the
  annealed positions to a sibling file for visual review. Preview
  file uses `strip_connections=False` so labels stay from the
  original — expect visually noisy render (labels at old positions,
  components at new). Useful for coord-check, not final review.

### v1 workflow-checklist for schematic autoplace (proven pattern)

The bms flow that worked:

1. Place all components at rough starting coords (no need to hand-
   optimise — autoplacer will move them; just avoid pin-coord
   collisions by spacing >7.62 mm apart).
2. Wire everything with `connect_pins(style="label")` — one call per
   net. For failures (e.g. dense pin clusters), fall back to
   per-pin `add_schematic_net_label` with `componentRef` + `pinNumber`.
   Add `no_connect` for intentional NC pins.
3. Add global labels for cross-sheet nets (BAT+, GND, etc.) via
   `add_schematic_net_label(labelType="global_label")`.
4. Run `autoplace_schematic` with compact params + reduced iterations.
5. **After autoplace, re-run step 2 to restore labels/wires** — the
   current apply/rewire flow strips them silently (#61).
6. `run_erc` to validate.

## RESUME HERE — end of session 2026-07-03 (previous)

**Anti-anchoring experiment is closed; this project is now the "build v2"
schematic pass** using the architecture v2-alt landed on. Read the three
project docs before touching the schematic: `REQUIREMENTS.md` (frozen
specs), `TOPOLOGY.md` (all architecture decisions + JLCPCB parts),
`COMPARE.md` (v1 head-to-head). Referencing v1 (`../power_module/`) is
now allowed — the compare phase is over.

### What's done

- **Project skeleton** (`d130f39`): `.kicad_pro`, root sheet, 4 empty
  sub-sheets (`input`, `charger`, `bms`, `buckboost`). Sheet plan is a
  2×2 block-diagram layout on the root; **no hierarchical sheet pins**
  — all cross-sheet nets travel via global labels (GND, BAT+, VBUS_9V,
  VOUT_12V). Keeps sub-sheets uncluttered.
- **Libraries** (`d130f39`): `libs/power_module_lib.{kicad_sym,pretty,
  3dshapes}` copied from v1 and extended with **IP2326** (LCSC
  C2832094) auto-generated via `easyeda2kicad --full` — symbol +
  VQFN-24 footprint + 3D step/wrl. Registered in project-scoped
  `sym-lib-table` / `fp-lib-table` using `${KIPRJMOD}` for portability.
  LM5176PWP, BQ76920PW, and CH224K come from stock KiCad libs.
- **buckboost.kicad_sch drafted** (`c68b8a7`): LM5176 + 4 external
  N-FETs + inductor + full support network. 25 components, 17 named
  nets, ERC not run yet (defer until all four sheets are populated
  since power labels are cross-sheet).

### Still empty (in-priority order for next session)

1. **bms.kicad_sch** — BQ76920PW AFE (TSSOP-20), external
   charge+discharge N-FET stack in pack return path, cell holder
   (`power_module_lib:BH-18650-B5BA016` × 1 or × 3 depending on holder
   pick), cell-sense network **1 kΩ + 100 nF + ~15 V ESD/TVS per line
   on VC1/VC2/VC3** (per REQUIREMENTS lessons-learned), I²C breakout
   header. Estimated ~30 components + ~40 wires. Roughly same scale as
   buckboost.
2. **charger.kicad_sch** — IP2326 (`power_module_lib:IP2326`, VQFN-24)
   with the datasheet §10 typical reference circuit: 2.2 µH boost
   inductor, 10 µF Cin, **2× 22 µF ceramic on VSYS pins 19/20**, 10 µF
   Cout, RVSET/RISET/ROV/RUV/ROT setup resistors, LED indicator, NTC,
   **CON_SEL to GND via 1 kΩ for 3S mode**, **VBATM/VBAT_GND (pins
   23/24) LEFT FLOATING** to disable IP2326's 2-cell balancer.
   Estimated ~20 components; smaller than buckboost because most FETs
   are integrated.
3. **input.kicad_sch** — 16-pin USB-C receptacle (charge-only; D± not
   routed), **SRV05-4** on CC1/CC2, **~12 V VBUS TVS** (SMAJ12A-class
   works since input is capped at 9.5 V), ferrite/CM choke, **CH224K**
   (stock `Interface_USB:CH224K`, LCSC C970725) strapped to request
   **9 V PDO** on CC pins. Output goes to IP2326 VIN. Estimated ~15
   components; smallest sheet.

### Then

4. **ERC** across the whole design (`run_erc` after all sheets
   populated so global labels resolve). Expect complaints about the
   TBDs left on the buckboost sheet — resolve them at this stage.
5. **Commit initial schematic**, close task #59 in the task list.
6. Later: annotate_schematic + BOM export + PCB drafting (all deferred).

### buckboost.kicad_sch — what to review + refine before moving on

Rendered image is in the `c68b8a7` commit thread. Notes:

- **FET pick (Q1–Q4) needs a real MPN.** v1 uses `FDS9926A` (dual N,
  20 V/4 A) which is undersized for v2's 5 A continuous / 6.5 A peak.
  Look for a **30 V / ≥ 20 A N-FET** on JLCPCB Basic or low-cost
  Extended — candidates worth checking: **AO4406** (30 V/20 A, SOIC-8,
  ~8 mΩ), **AON7418** (30 V/30 A, DFN 3×3, ~4 mΩ), or **CSD17559Q5**
  (~4 mΩ). Whatever gets picked, update Q1..Q4's `Value` field and
  set an `LCSC` property.
- **CS/CSG (pins 16/15) currently unconnected.** LM5176 needs
  high-side inductor current sense for its control loop. Two options:
  (a) wire R8 (currently placed but unwired) inline with the inductor
  and route CS/CSG across R8 — accurate but adds a shunt loss, or
  (b) switch to **Rds(on) sensing** by tying CS to Q2's drain and CSG
  to Q2's source (uses the LS FET's on-resistance as the sense
  element — free, less accurate). Datasheet allows either. Option (b)
  is what most low-cost designs do.
- **SLOPE (pin 7) — needs a resistor R5.** Tying to GND disables
  slope comp which risks sub-harmonic oscillation above 50 % duty.
  Pick per LM5176 §9.3.7. Value depends on Fsw + inductor.
- **PGOOD (pin 17) is currently floating** — either add an LED to
  VCC/BIAS via a resistor, or leave floating.
- **All resistor values are LM5176-EVM starting points.** Verify with
  a real design pass once FET pick is locked:
  - R1/R2 FB divider: 110k / 10k → 12.0 V from Vref=1.0 V (exact)
  - R3 COMP: 20k
  - C8 COMP: 4.7 nF
  - C7 SS: 10 nF
  - R4 RT: 40.2 k → ~500 kHz Fsw
  - R6 EN pull-up: 100 k to BAT+
  - L1: 6.8 µH — should verify ripple vs Fsw at chosen output current
  - C1/C2 Cin: 22 µF each; check DC-bias derating at 12.6 V (see
    REQUIREMENTS lessons-learned)
  - C10/C11 Cout: 22 µF each; check DC-bias derating

### Nice-to-have polish

- Component positions on the buckboost sheet are approximate — the
  render is legible but some labels overlap ("BAT+" near VIN pins,
  "VOUT_12V" repeated on FB divider and ISNS). Minor cleanup, can be
  done in KiCad GUI after ERC clears.
- `LM5176_VCC` and `LM5176_BIAS` net names were chosen to avoid
  collision with the stock KiCad `VCC` power symbol (which is a
  global-scope symbol that would electrically merge the LM5176's
  internal 5 V rail with the BQ76920's 3.3 V REGOUT — bad).

### Files / commits

- `e3db9cf` — initial planning package (REQUIREMENTS + TOPOLOGY +
  COMPARE from prior sessions).
- `d130f39` — project skeleton + libs + IP2326 + root block diagram.
- `c68b8a7` — buckboost draft (this session).
- `.mcp.json`, `.claude/`, `*.kicad_prl` gitignored per v1 convention.

### Task state

Live task list (`TaskList`) has:
- Completed: #51 skeleton, #52 libs, #53 IP2326 symbol, #54 root
  block diagram, #55 buckboost.
- Pending: #56 bms, #57 charger, #58 input, #59 ERC + final commit.
