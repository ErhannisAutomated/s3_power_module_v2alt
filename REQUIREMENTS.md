# Power module v2-alt — requirements (anchoring-free)

This directory is a **from-scratch redesign experiment** running in
parallel with v2 proper. The goal is to test whether a fresh
component + topology pass — done without consulting v1's choices —
produces meaningfully different (and possibly better) decisions
than the copy-revise path.

## Anti-anchoring rule (READ FIRST)

Until the **compare phase** explicitly begins, **do not consult**:

- `../power_module/*.kicad_sch` or `*.kicad_pcb` files
- `../power_module/NOTES.md` (especially the v2 backlog section)
- `../power_module/power_module_bom.csv` or any BOM artifact
- The auto-memory entry `project-power-module` (or any v1-specific
  memory whose body names specific parts/values/topologies)

You **may** consult:

- This file and `TOPOLOGY.md` (the frozen requirements + your own
  in-progress topology notes)
- Reusable workflow memories (PCB workflow, JLCPCB prefs, schematic
  routing, autoplacer params, layer order, etc.)
- Datasheets and JLCPCB part search for parts you are considering
- General KiCAD MCP tooling (`search_jlcpcb_parts`,
  `get_jlcpcb_part`, datasheet lookup)

Phases:

1. **Requirements** — fill in this file from first principles. Done.
2. **Topology + part selection** — fill in `TOPOLOGY.md`.
3. **Compare** — unmask v1; produce `COMPARE.md` head-to-head.

Do not proceed to phase 3 until phases 1 and 2 are complete enough
to compare. Otherwise we end up reverse-engineering v1's choices,
which destroys the experiment.

## Frozen requirements (the problem statement)

These are the original specs that v1 was designed to satisfy.
They define the experiment's playing field, not v1-specific
implementation choices.

### Functional
- **Output:** 12 V regulated rail
- **Output current:** ~5 A continuous (worst-case load TBD — fill in
  derived target below)
- **Battery pack:** 3S Li-ion (3× 18650 cells in series)
- **Charging:** USB-C with PD negotiation (sink role). Must charge from
  a range of sources, not just a full 20 V PD charger — a weaker charger
  (9 V / 15 V PDO, or low current) must still charge the pack, just more
  slowly. (Added 2026-06-17; drives the buck-boost charger choice.)
- **BMS:** cell-level over/under-voltage protection, over-current
  protection, cell balancing
- **External I/O:**
  - **USB-C: charge input only.** PD negotiation (sink role) over the CC
    pins to draw charging power. **No USB data** (D+/D- unused) and **no
    power is sourced out** of the USB-C port. It is purely a power inlet.
  - **12 V rail:** the only power *output*.
  - **I2C breakout:** the only data I/O (telemetry + fuel-gauge readout).

### Derived / to-fill-in (do this FIRST, before topology)

_Filled in phase 1 from first principles + the product decisions in the
"frozen requirements". Values marked **(assumption)** are my proposals,
open to revision; the rest follow from the spec._

- **Worst-case Iout:** 5 A continuous design point (60 W out @ 12 V).
  Peak 6.5 A (1.3×) for ≤100 ms load transients. Converter and output
  copper sized for the 6.5 A peak, thermals for the 5 A continuous.

- **Operating Vin range (3S pack):** 9.0 V (dead, 3.0 V/cell) → 12.6 V
  (full, 4.2 V/cell) as the *normal* operating window. Must still behave
  gracefully (no latch-up / no reverse current) down to the BMS cutoff
  ~8.4 V (2.8 V/cell). **Key topology driver:** the 12 V output sits
  *inside* the input range — input is below Vout near empty and above
  Vout when full — so a pure buck OR pure boost cannot hold 12 V across
  the whole curve. A buck-boost-class topology is effectively forced.

- **Required output ripple:** ≤ 50 mV pk-pk at full load, 20 MHz
  measurement BW. (~0.4 % of 12 V — clean general-purpose rail.)

- **Required transient response:** for a 2.5 A load step (50 %→100 %),
  output deviation ≤ ±600 mV (±5 %), settling < 200 µs. **(assumption —
  no downstream load spec given; chosen as a sane default.)**

- **Quiescent current target:** ship/off (BMS shutdown) < 10 µA; sleep
  (BMS active, converter disabled) < 100 µA; converter enabled at
  no-load Iq < 1 mA. Rationale: 100 µA off a ~2.5 Ah pack ≈ <1 Ah/yr
  self-drain, acceptable shelf life.

- **Thermal envelope:** free-air, **no heatsink**, design ambient 40 °C
  (operating 25–40 °C). PCB copper-pour cooling only. At 60 W out a 92 %
  converter dissipates ~5 W — hard to shed on bare copper at 40 °C — so
  this sets an **efficiency floor of ~93–95 %** at mid-Vin / full load,
  and makes spreading switch + inductor + diode losses across copper a
  real layout constraint.

- **Mechanical: (assumption)** the **3S 18650 cells dominate the
  outline** — a flat 3-up holder is ~65 mm (cell length + contacts) ×
  ~57 mm (3 × ~19 mm cell pitch). **Decision: cells and circuitry on
  opposite sides** — THT 18650 holder(s) on the **top**, all SMT
  electronics on the **bottom**. This:
  1. shrinks the outline to roughly the cell footprint (~**75 × 65 mm**,
     4-layer) instead of cells-beside-electronics (~110 × 70 mm);
  2. shortens the high-current pack / cell-tap runs — the BMS AFE sits
     directly under the holder contacts;
  3. separates the switching converter (a heat source) from the
     heat-sensitive cells.
  Crucially it keeps assembly cheap: the **SMT stays single-sided**
  (bottom only) and the holder is THT, so we avoid the double-sided-SMT
  cost step while still using both faces. USB-C on one short edge;
  I2C/telemetry header + 12 V output terminal on an accessible edge;
  4× M3 mounting holes at corners.
  **Thermal caveat:** keep the converter's hot copper *laterally* offset
  from the cell zone — heat conducts through the board, and Li-ion
  derates / ages above ~45 °C. **Open** to a real enclosure / cell-holder
  part constraint if one exists.

- **Certification target:** FCC/CE **Class B** as an aspirational target,
  *not* mandatory (basement/personal use). Design in the EMC best
  practices the lessons-learned demand (USB-C ESD array, VBUS CM
  choke/ferrite, low-inductance switching return paths), but do **not**
  add cost/complexity purely to guarantee a lab pass — scale filtering
  back if it becomes a real burden.

- **Cost target:** no hard BOM-$ ceiling. Selection rule (per user):
  Basic vs Extended is a minor lever; the real cost cliff is parts that
  force **Standard** PCBA (hand-load / off-line feeders) over **Economic**
  assembly. Prefer parts compatible with JLCPCB **Economic** assembly;
  accept Extended freely; treat "needs Standard assembly" as a strong
  negative. Optimize for low-qty (1–10 boards) total landed cost.

### Lessons-learned constraints (requirements, not solutions)

These are real constraints we learned from v1 + EMC review. They
shape what *kinds* of solutions are acceptable, without dictating
which specific solution to pick.

- **USB-C connector requires ESD + EMC filtering** within ~25 mm of
  the connector (ferrite or CM choke on VBUS; TVS/ESD protection on the
  exposed connector pins). Lack of this on v1 was the single biggest EMC
  finding. **v2-alt note:** since USB-C is charge-input only with no data
  (D+/D- unused), ESD focus is on **VBUS and the CC pins** (CC carry the
  PD negotiation and are exposed at the plug). D+/D- protection is
  optional — only worthwhile if a combined ESD array covers them for
  free / for connector-pin robustness.
- **Switching-converter return paths** must have low-inductance
  GND continuity at every layer transition. Either GND-on-both-sides
  or GND↔PWR caps within ~5 mm of the via. Affects stackup choice.
- **MLCC DC-bias derating** must be accounted for when sizing output
  caps (a 22 µF 25 V X5R at 12 V loses 40-60% of its capacitance).
- **JLCPCB Basic-parts preference** for cost + assembly availability.
- **Trace width must match netclass ampacity** (POWER_4A or whatever
  the equivalent ends up being — sizing per IPC-2152 at the chosen
  copper weight). v1 had 33 DRC track_width violations from this
  going un-checked during routing.

- **Assembly-tier cost cliff:** at JLCPCB the meaningful BOM cost step
  is **Economic vs Standard** PCBA, not Basic vs Extended parts.
  Extended parts are usually only marginally pricier; a part that forces
  the whole board onto the Standard assembly line (or hand-loading) is
  the expensive outcome. Prefer Economic-assembly-compatible packages.

(Add more as you discover them in the topology phase.)
