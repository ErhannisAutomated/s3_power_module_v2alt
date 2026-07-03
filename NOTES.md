# power_module v2-alt — session resume notes

Living document; updated at the end of each session so the next session
can pick up from a cold start (after Claude Code context compaction).
Same convention as v1's `../power_module/NOTES.md`.

## RESUME HERE — end of session 2026-07-05

**Charger sheet shipped today** as commit `8c2cb62` on `main`. IP2326 +
support network, 19 components, ERC clean apart from cosmetic
Unspecified/Passive pin-type warnings from the IP2326 library symbol.

### charger.kicad_sch state

- **19 components**: U1 (IP2326, VQFN-24, C2832094), L1 (2.2 µH boost
  inductor), C1 (10 µF Cin @ VBUS_9V), C2/C3 (2× 22 µF @ VSYS), C4
  (10 µF Cout @ VOUT), C5 (100 nF bootstrap), R1 (VSET), TH1 + R2 (NTC
  divider), R3 (BAT_STAT pull-up), D1 + R10 (charge LED, 4.7 kΩ series),
  R4 (TIME_SET), R5 (VIN_UVSET), R6 (VIN_OVSET), **R7 (CON_SEL = 1 kΩ
  to GND → 3S mode)**, R8 (ISET), R9 (EN pull-up to VBUS_9V).
- **Pins 1/2 (D+/D-) and 23/24 (VBATM/VBAT_GND) NC** per REQUIREMENTS
  resolution — PD goes through CH224K on input sheet, BQ76920 handles
  cell balancing.
- **All resistor values are TBD placeholders** (10 k / 100 k / 4.7 k /
  1 k). Verify against IP2326 §10 typical circuit before layout — the
  ratio-based ones (ISET, VSET, VIN_UVSET, VIN_OVSET, TIME_SET) all
  need real calculation, not defaults.
- **Cross-sheet nets** exported as globals: VBUS_9V (from input), BAT+
  (to bms + buckboost — same rail as BQ76920's pack side), GND.

### Layout lesson learned — schematic pin-collision

**Vertical spacing between passive components on the same X must be
≥15.24 mm** (or offset X between adjacent neighbors). The 7.62 mm pin
span + 2.54 mm label stubs mean 10 mm spacing bridges nets — the two
stubs meet in the middle at ~y=(y_a + y_b)/2, and any two labels
placed there merge onto the same electrical net. Killed the first
charger draft; had to nuke-and-restart the sheet with a 15.24 mm grid.

Same lesson applies to future dense sheets. If you *need* tight
vertical packing, alternate X positions between adjacent components.

### All four sheets shipped + full-project ERC clean

- **input.kicad_sch** (`5b82f16`, main): 16-pin USB-C receptacle
  (charge-only), CH224K strapped for 9 V PDO via 24 kΩ CFG1 → GND,
  SRV05-4 ESD on CC1/CC2, SMAJ12A TVS on VBUS_RAW, 600 Ω ferrite between
  VBUS_RAW and VBUS_9V. Sheet ERC clean.
- **buckboost.kicad_sch** re-drafted (`768c2f2`, main): the yesterday
  draft had the same too-tight vertical stacking bug as the first
  charger draft — the label stubs bridged BAT+ ↔ GND ↔ VCC ↔
  LM5176_VCC/BIAS ↔ COMP ↔ RT into one 31-pin mega-net. Nuke-and-
  restart at 15.24 mm grid fixed it. Also resolved the yesterday-TBDs:
  CS/CSG via Rds(on) sensing (dropped R8 shunt), SLOPE = 10 kΩ
  placeholder, PGOOD/DITH NC, MODE to GND for auto PFM/PWM.
- **Full-project ERC** (`768c2f2`): 0 errors, 34 warnings — all
  cosmetic. Design is schematic-clean.

### Still to do

- **Autoplacer #61** (bug fix) — apply/rewire silently strips
  labels+wires + compact profile still over-spreads. Blocks the
  autoplace pass on charger/input/buckboost (#63) and the label→wire
  conversion (#64) after that.
- **#63 Autoplace pass** — all four sheets are laid on a manual
  15.24 mm grid, not autoplaced. Also, the bms autoplace from
  yesterday over-spread to 245×155 mm (target ~46×105 per compact_v3).
  Once #61 lands, re-run on all four. Blocked by #61.
- **#64 Label → routed wire conversion** — every net on every sheet
  is currently `connect_pins(style="label")`, i.e. label per pin, not
  routed. Per `[[feedback_schematic_routing]]` the default is
  `style="auto"` with labels only as fallback (and always for power
  nets). Re-run each net through `connect_pins(style="auto")` after
  #63 stabilises coords. Blocked by #63.
- **Buckboost polish** — R5 SLOPE value TBD per LM5176 §9.3.7 (need
  final Fsw/L1 pick). Q1-Q4 still generic 30V/20A_NFET placeholders;
  final MPN pick due before layout (candidates in yesterday's NOTES
  block: AO4406 / AON7418 / CSD17559Q5). R1/R2 FB divider is exact
  110k/10k → 12.0 V; verify at real load.
- **Charger polish** — config-R values (VSET / NTC / BAT_STAT / LED
  series / TIME_SET / VIN_UVSET / VIN_OVSET / ISET / EN) are TBD
  placeholders; verify against IP2326 §10 typical circuit before
  layout. CON_SEL (R7 = 1 kΩ) is locked for 3S.
- **PCB drafting** — annotate_schematic + BOM + sync_schematic_to_board
  → freerouting. Not yet started.

### Charger sheet — still open

- **Config resistor values are TBD placeholders** (VSET, NTC pull-up,
  BAT_STAT pull-up, LED series R, TIME_SET, VIN_UVSET, VIN_OVSET, ISET,
  EN pull-up). Verify against IP2326 §10 typical reference circuit
  before layout. CON_SEL (R7 = 1 kΩ → GND) is locked for 3S mode.

## RESUME HERE — end of session 2026-07-04 (previous)

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

### Schematic workflow-checklist (current best-known state)

Two-phase workflow. Phase 1 gets the sheet to ERC-clean; phase 2
polishes it to look like a real schematic. Phase 2 is currently
blocked on autoplacer bug #61 — until that's fixed, phase 1 is what
ships.

**Phase 1 — get to ERC-clean:**

1. Place all components at ≥15.24 mm vertical spacing per column
   (see `[[feedback_schematic_vertical_spacing]]`). Tighter than
   that on the same X bridges nets via label stubs.
2. Wire every net with `connect_pins(style="label")`. This is the
   RELIABLE-BUT-UGLY fallback: one label per pin, no real wires.
   Use it here because auto routing on unrefined placement often
   crosses symbols. For dense pin clusters that fail, fall back
   to per-pin `add_schematic_net_label(componentRef=…, pinNumber=…)`.
   Add `no_connect` for intentional NC pins.
3. Add `labelType="global_label"` for cross-sheet nets (BAT+, GND,
   VOUT_12V, VBUS_9V, VSYS, etc.) at existing local-label positions
   so BFS merges them.
4. Add `power:PWR_FLAG` on any global net whose only source is a
   Passive pin (Cin pos-terminal, cell-holder terminal, etc.) —
   ERC's "Input Power pin not driven" needs a Power_Output somewhere
   on the net.
5. `run_erc` to validate. Expect cosmetic warnings (Unspecified/
   Passive pin-type mismatches, local↔global label pairs — both
   intentional).

**Phase 2 — polish to real schematic (BLOCKED on #61 today):**

6. `autoplace_schematic` with compact params + reduced iterations.
   Blocks on #61: the apply/rewire flow silently strips labels+wires
   on real sheets, and the compact profile still over-spreads.
7. After autoplace stabilises coords, re-run each signal net through
   `connect_pins(style="auto")` per `[[feedback_schematic_routing]]`
   to convert labels → routed wires. Power nets stay as labels via
   the built-in `powerNets` behaviour.
8. `run_erc` again to catch any bridges the auto-router introduced.

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
