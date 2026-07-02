# Power module v2-alt — topology + component selection

**Anti-anchoring rule still applies:** do not consult v1 files,
v1 NOTES.md, or v1-specific memory until the compare phase.
See `REQUIREMENTS.md`.

All LCSC numbers / stock / price verified against the local JLCPCB DB
on 2026-06-17 (see MCP-issues note: use `search_jlcpcb_parts`, NOT
`get_database_stats`). Per the cost rule in REQUIREMENTS, Extended is
fine; we only avoid parts that would force Standard PCBA.

## Power architecture

Conversion path from a 3S pack (9.0–12.6 V) to a regulated 12 V output.

**The governing fact (from REQUIREMENTS):** 12 V sits *inside* the input
window — above the pack when nearly empty, below it when full — so the
converter must both **boost** (low SoC) and **buck** (high SoC) to hold
12 V. That eliminates pure-buck and pure-boost, and an LDO (needs Vin >
Vout always). SEPIC works but is less efficient and needs a coupled
inductor + coupling cap. Two-stage (boost-to-intermediate then buck)
doubles the loss budget — unacceptable against our ~93–95 % free-air
efficiency floor. **A single-stage 4-switch synchronous buck-boost is
the natural answer.**

### Decision:
**Single-stage 4-switch synchronous buck-boost.**
- **Primary: TI LM5176 controller + 4 external N-FETs.**
  `LM5176PWPR` — **C442493**, Extended, HTSSOP-28-EP (thermal pad),
  4.2–55 V in, ~$1.57–2.23, 5497 stock.
- **Compact alternative: TI TPS55288 (integrated FETs).**
  `TPS55288RPMR` — **C2864583**, Extended, VQFN-26, 2.7–36 V in,
  16 A switch, ~$1.50–2.48, 7969 stock. I²C-configurable output.

### Why:
The free-air / no-heatsink / 40 °C constraint is the deciding factor.
At 60 W out, ~5 W of loss has to leave through bare copper. The LM5176
**controller** lets us pick efficient external FETs and **spread the
dissipation** across four packages + the inductor instead of cooking one
QFN — and it comfortably clears the 93–95 % efficiency floor (LM5176
designs hit 95–97 % at these voltages). The TPS55288 is the tidy,
cheaper, fewer-parts option and is genuinely attractive for a bench
build, but it concentrates the full loss in a single VQFN-26 — borderline
without a heatsink at 5 A continuous. **Pick LM5176; keep TPS55288 as the
fallback if board area gets tight and a thermal sim says the QFN survives
derated.** Switching freq ~300–400 kHz (loss vs. size balance; lets us
use a physically reasonable ~6.8–10 µH inductor).

## BMS architecture

Options: integrated BMS+charger single-chip; separate AFE + charger;
fully discrete (FETs + comparator).

### Decision:
**Separate I²C analog front-end + external FET pair.**
`BQ76920` (`BQ7692003PWR` — **C601650**, Extended, TSSOP-20, ~$1.04–1.75,
2730 stock). 3–5S; per-cell OV/UV, OCD/SCD, **cell balancing**, integrated
coulomb counter; I²C. Drives an external **dual N-channel** charge +
discharge FET stack in the pack return path (MPN TBD — ~30 V, low-RDSon,
e.g. a dual-NFET in 3×3 or a pair of singles sized for 6.5 A).

### Why:
The requirement explicitly calls out I²C telemetry + fuel-gauge readout,
so an **I²C AFE is mandated by the spec**, not a luxury. BQ76920 covers
3S directly, does protection *and* balancing in one chip, and its coulomb
counter + the charger's ADC (below) together satisfy "fuel-gauge readout"
**without** a dedicated gauge IC (an optional add later for better SoC).
A single-chip BMS+charger for 3S with PD is essentially unavailable on
JLCPCB; fully-discrete loses balancing + telemetry. The AFE+charger split
is the sweet spot.

## Charging

USB-C PD (sink) charging a 3S pack (charge to 12.6 V).

### Decision: (UPDATED 2026-07-02 — datasheet-validated, PD trigger added)
**IP2326 autonomous boost charger + CH224K USB-C PD trigger.** Host-less,
no MCU. Datasheet validation (2026-07-02, IP2326 rev V1.11) confirms every
core assumption **except** PD compatibility: IP2326 does legacy BC1.2/QC
fast-charge negotiation via **D+/D-**, not USB-C PD via CC. To satisfy the
REQUIREMENTS "USB-C PD sink" clause, we add a small PD trigger.
- **Charger:** Injoinic **IP2326** (**C2832094**, Extended, QFN-24-EP 4×4,
  ~$0.30–0.62, ~13 k stock) — autonomous 2S/3S sync **boost** charger,
  integrated power MOSFETs, LED status (on=charging, off=full, blink=fault),
  NTC, adjustable CV / ICC / UV / OV / timeout. 3S CV = **12.6 V** exact
  (RVSET=NC), CON_SEL to GND via 1 kΩ selects 3S. ICHG = 90 000/RISET →
  RISET = 60 kΩ for 1.5 A. Efficiency 91.5 % (5 V→12 V/1 A) / **94 % at
  9 V→12 V/1 A**. Only external parts: 2.2 µH L, 10 µF Cin, 2×22 µF C on
  VSYS pins, 10 µF Cout, and setup resistors.
- **PD trigger:** WCH **CH224K** (**C970725**, Extended, ESSOP-10,
  ~$0.33–0.62, ~10 k stock). Strapped to request **9 V PDO** via CC. Sits
  between USB-C VBUS and IP2326's VIN. Falls back to 5 V default if the
  source doesn't do PD (plain USB-C 5 V still charges, just slower).
  Also validates why v1 used the same family (CH224K@20V); the trigger
  concept is exactly right, just re-targeted for a boost charger.
- **VBATM / VBAT_GND (pins 23/24) left FLOATING** to disable IP2326's
  2-cell balancer — BQ76920 does 3-cell balancing on its own, no conflict.
- **Dropped:** BQ25703A (needs an MCU — its charge regs boot at 0),
  CYPD3177 (overkill PDO ladder), BQ24610 (buck-only), IP2721 (no 9 V
  output), IP2368 (low stock + unneeded 20 V PD).

### Why:
The driving requirement is **charge from weak sources**, not I²C. A
**boost** charger fed 5–9 V inherently charges the 12.6 V pack from *any*
USB supply — even the weakest 5 V port works — a cleaner answer to the
weak-charger problem than a buck-boost that still wants ≥9 V PD. IP2326 is
**autonomous** (no MCU, matching v1's host-less philosophy) and cheap with
deep stock. The CH224K PD trigger closes the "USB-C PD sink" gap in the
IP2326 spec: without it, a modern USB-C PD-only charger would stay at 5 V
default (charge works but slow); with it, the CC-pin PD handshake requests
9 V and IP2326 runs at its 94 %-efficient sweet spot. Telemetry stays
firmware-free: **low-battery via BQ76920 I²C** to the breakout host,
**"full" via LED** off IP2326's LED pin. Cost of this whole path: no 20 V
fast-charge (1.5 A cap) — fine for 18650s at 0.5 C.

### ⚠️ Revision note (2026-06-17) — the charger does NOT need I²C
User clarified the telemetry needs, which undercuts part of the BQ25703A
rationale:
- **"Low battery" (discharging)** is reported by the **BMS (BQ76920)**
  over I²C — a *different chip* from the charger, present in both v1 and
  v2-alt. The charger's I²C was never the source of battery state.
- **"Fully charged" (charging)** only needs an **LED** off a charger
  status pin — no I²C required.

So the charger's I²C is not a feature we need — and for **BQ25703A it's a
liability**: its charge-voltage/current registers power up at **0**, so it
**won't charge at all without an I²C host writing them**. Choosing it
silently forces an **MCU** onto the board — which v1 deliberately avoided
(v1 is host-less; the I²C breakout just exposes the BMS to whatever
external host plugs in). The real requirement driving the charger is
*buck-boost (charge from weak sources)*, **not** I²C.

**→ Prefer an autonomous (host-less) boost/buck-boost charger with a
status pin for the "full" LED.** Candidates found on JLCPCB:
- **IP2326** (**C2832094**, Extended, VQFN-24, ~$0.30–0.62, 13 303 stock)
  — autonomous **2–3S boost charger** from 5–9.5 V USB, **built-in cell
  balancing** + protection, status outputs. Charges a 3S pack from plain
  5 V/9 V (no PD chip strictly required); ~1.5 A ≈ 0.5 C of a 3 Ah 18650,
  exactly our target rate. Trade: ignores 20 V PD (no fast-charge), 1.5 A
  cap. Datasheet check needed (true 3S? input source detection? balancing
  overlap with the BQ76920?).
- **IP2368** (**C5203723**, Extended, QFN-48, ~$1.5–2.7, **only 357
  stock**) — autonomous 2–6S **buck-boost**, up to 25 V in, 5 A; uses
  20 V PD for fast charge. Bigger, lower stock, more complex.

**Decision (host-less vs MCU): RESOLVED 2026-06-17 → host-less.** User
chose the autonomous charger (no MCU); open question #7 closed. IP2326 is
the charger of record (see the updated Decision above). IP2326/IP2368 were
the two candidates; IP2326 wins on stock, cost, simplicity, and our
0.5 C / 1.5 A target making its lack of 20 V fast-charge irrelevant.

## Stackup

### Decision:
**4-layer: F.Cu (signal + power) / In1.Cu GND / In2.Cu PWR / B.Cu
(signal + secondary pour)** — the project default. Reasons it's right
here: the switching converter needs an unbroken GND reference plane
directly under it (In1) for low-inductance return; the high-current pack
/ 12 V rails get the In2 PWR plane + wide top pours. Per the lessons
learned, place **GND↔PWR stitching/decoupling caps within ~5 mm** of
every layer transition on the switch-node return path, and keep a GND
pour on B.Cu beneath the converter so return current has copper on both
outer faces. 1 oz copper baseline; **bump high-current rails to 2 oz or
size widths per IPC-2152** (sized in the POWER netclass, below).

## Protection

### Decisions:
- **USB-C ESD (CC + any sense pins):** `SRV05-4` (**C7420376**,
  Preferred, SOT-23-6, ~$0.05, 654 k stock) on the CC1/CC2 (and unused
  D± left protected/grounded). Within 25 mm of the connector.
  *Note: D± unused (charge-only port) per REQUIREMENTS — ESD focus is
  CC + VBUS.*
- **VBUS surge/TVS:** SRV05 (5 V clamp) doesn't protect VBUS. Since input
  is now capped at ≤9.5 V (no 20 V PD), a **~12 V working-voltage uni-dir
  TVS** (SMAJ12A-class) suffices — *lower* rating than the earlier 24 V
  plan, a side benefit of the 9 V-max charger. + ferrite/CM choke for EMC.
  (MPN TBD.)
- **Reverse-polarity (battery side):** ✅ RESOLVED 2026-07-02 — **rely on
  BQ76920 sense-line filtering + fault detection; no ideal-diode
  controller, no series pack-side FET.** Rationale: cells are *internal*,
  so the realistic reverse-polarity event is a backwards 18650 insertion,
  not a reversed pack terminal. Ideal-diode controllers block reverse
  *current* but a reversed cell doesn't cause reverse current — it just
  drives VC pins negative. The right protection is the standard TI ref-
  design network on each cell-sense line, which:
  1. **1 kΩ series R + 100 nF cap** on each cell-tap sense line
     (VC1/VC2/VC3 for 3S; VC0 is pack ground). Bounds fault current into
     the AFE's internal clamps to ~4 mA at 4.2 V.
  2. **Small ~15 V-class ESD/TVS to GND on VC1/VC2/VC3** (3× parts,
     belt-and-suspenders ~$0.15; standoff above 12.6 V pack max, well
     below the AFE's abs-max; keeps the AFE alive under transient events
     + reversed-cell insertion).
  3. **BQ76920 refuses to enable its FET stack** when it sees an
     impossible cell-voltage pattern → the reversed cell is a contained
     fault: module doesn't power on, no LEDs, user pulls the cell.

  Not protected: running the module with the pack disconnected but a load
  hanging on the 12 V output (n/a — not a reverse-polarity case).
- **Output (12 V) protection:** TVS clamp (e.g. SMAJ15A-class) at the
  output terminal + the converter's own cycle-by-cycle current limit for
  over-current; optional polyfuse.

## Component selection

| Block | Part / MPN | LCSC | Lib | Pkg | Key spec | Notes |
|---|---|---|---|---|---|---|
| Buck-boost ctrl | LM5176PWPR | C442493 | Extended | HTSSOP-28-EP | 4.2–55 V, BB | primary converter |
| (alt) integ. BB | TPS55288RPMR | C2864583 | Extended | VQFN-26 | 16 A sw, I²C | compact fallback |
| BMS AFE | BQ7692003PWR | C601650 | Extended | TSSOP-20 | 3–5S, I²C, bal | + ext dual NFET |
| Charger | **IP2326** | C2832094 | Extended | QFN-24-EP 4×4 | 2/3S boost, autonomous | 3S CV 12.6 V; RISET=60k→1.5A; VBATM/VBAT_GND float |
| **9 V PD trigger** | **CH224K** | **C970725** | Extended | ESSOP-10 | 5/9/12/15/20 V PD 3.0 sink | strap for 9 V; falls back to 5 V; on CC of USB-C |
| USB-C ESD | SRV05-4 | C7420376 | Preferred | SOT-23-6 | 4-ch ESD | CC + sense |
| ~~dropped~~ | BQ25703A / CYPD3177 / BQ24610 | — | — | — | — | MCU-path (BQ25703A) + buck-only (BQ24610) rejected |

### To finalize (specced, MPN pending):
- **Main inductor:** ~6.8–10 µH, Isat ≥ 8 A, low DCR, shielded (size by
  the LM5176 ripple calc at chosen Fsw).
- **Converter FETs (×4):** ~30–40 V N-ch, low RDSon, SO-8/PowerPAK,
  gate-charge matched to LM5176 driver.
- **BMS FETs (×2 / dual):** ~30 V N-ch, ≥6.5 A, low RDSon.
- **BMS cell-sense network (3 sense lines VC1/VC2/VC3 for 3S):** 1 kΩ
  series R (0603) + 100 nF cap (0402) per line + one small ~15 V ESD/TVS
  (0402/SOD-323) to GND per line. Pick a single ESD MPN reused across
  the three lines; ~16–18 V standoff (above 12.6 V pack max, below the
  BQ76920 VC-pin abs-max).
- **Charger:** IP2326 integrates the power MOSFETs — **no external charger
  FETs**. External parts: L=2.2 µH, Cin=10 µF, C_VSYS = 2×22 µF ceramic
  close to pins 19/20, Cout=10 µF, plus RVSET/RISET/ROV/RUV/ROT setup
  resistors. All confirmed against datasheet V1.11 §10 typical schematic.
- **USB-C connector:** 16-pin power+CC receptacle (charge-only; D± not
  routed to anything since IP2326's D+/D- QC handshake is bypassed by the
  CH224K PD handshake on CC). Pick a JLCPCB Economic-assembly-friendly
  SMD part.
- **Output bulk caps:** account for MLCC DC-bias derating — a 22 µF/25 V
  X5R at 12 V keeps only ~40–60 %. Size *effective* C, likely mix of
  several 22 µF/25 V (or 35 V) X7R + a bulk electrolytic/polymer.
- **Input bulk caps:** across the pack at the converter input.
- **Sense resistors:** charger Rsns, converter Rsns, BMS pack-current
  sense (per each IC's ref design).
- **I²C breakout connector:** SDA/SCL/ALERT/GND/3V3 header (bus master =
  BQ76920 only; charger is autonomous. Optional future gauge IC could
  share the bus).

### Netclasses (ampacity per IPC-2152, REQUIREMENTS lesson):
- `POWER` (pack + 12 V out, 5–6.5 A): width sized for ≥6.5 A at chosen
  copper weight (verify before routing — v1's mistake was skipping this).
- `CHARGE` (charger ↔ pack): size for max charge current.
- `SIGNAL`/`I2C`: default.

## Open questions / unknowns

1. **TPS55288 vs LM5176 thermal** — need a loss/thermal estimate at 5 A
   continuous, 40 °C, no heatsink to confirm LM5176 is necessary (or that
   TPS55288 survives derated). Validate with a sim or a copper-area
   thermal calc before committing.
2. **PD source reality** — ✅ RESOLVED (2026-06-17): usually 20 V, but
   must also work on weaker chargers. → Validates buck-boost BQ25703A;
   BQ24610 ruled out. CYPD3177 uses a 20→15→9→5 V PDO fallback ladder.
3. **Charge current / cell spec** — ✅ RESOLVED: target plain **18650**
   cells (~2.5–3.5 Ah typical). Set nominal charge current ~**1.5 A**
   (≈0.5 C of a 3 Ah cell), but rely on the charger's input-power DPM so
   a weak PD source auto-reduces it. Size charger FETs/inductor for the
   1.5 A charge + the DPM headroom. Final number once a specific cell is
   chosen.
4. **Dedicated fuel gauge?** — ✅ RESOLVED: NOT needed. User wants only a
   rough "are we low" estimate, not accurate % SoC. BQ76920 per-cell
   voltage + coulomb counter (read over I²C) is sufficient. A BQ27z-class
   gauge stays an optional future add, off the BOM for now.
5. **Reverse-polarity approach** — ✅ RESOLVED 2026-07-02: rely on the
   BQ76920 sense-line filtering (1 kΩ + 100 nF per VC line) + small
   ~15 V ESD/TVS per line + BQ76920's built-in fault detection refusing
   to enable the FET stack on a reversed-cell event. No ideal-diode
   controller. Full rationale in §Protection.
6. **Single vs. dual 18650 holder orientation** — interacts with the
   opposite-side-cells mechanical decision and the converter hot-zone
   keep-away.
7. **Host-less vs. MCU (charger architecture)** — ✅ RESOLVED 2026-06-17:
   **(A) host-less / autonomous charger** (IP2326, no MCU). "Full" via LED,
   "low-batt" via BQ76920 I²C to an external host on the breakout. The
   BQ25703A + MCU path (B) was rejected to keep v1's firmware-free design.
8. **IP2326 datasheet validation** — ✅ RESOLVED 2026-07-02 against
   Injoinic IP2326 datasheet rev V1.11 (17 pp). Findings:
   - ✅ **3S CV = 12.6 V exact** (p.4 table, RVSET=NC). CON_SEL to GND
     via 1 kΩ selects 3S; recharge at 12 V; ICHG = 90 000/RISET.
   - ✅ **Integrated power MOSFETs** — very small external BOM.
   - ⚠️ **NOT USB-C PD** — IP2326 negotiates via D+/D- (legacy BC1.2/QC),
     not CC. Modern USB-C PD-only sources stay at 5 V default. → Added
     CH224K PD trigger on CC (see Decision above); satisfies the
     "USB-C PD sink" requirement and delivers 9 V to IP2326 VIN for the
     94 % efficiency point.
   - ✅ **No balancing conflict** with BQ76920 — IP2326's balancer is
     designed for 2S (VBATM = mid-pack of 2 cells); we float pins 23/24
     to disable it in 3S mode. BQ76920 does full 3-cell balancing.
   - ✅ **LED behaviour matches the spec** — LED pin: on = charging,
     off = full, blink = fault (p.13). BAT_STAT (low = trickle, high =
     CC) as a secondary hardware signal.
   - ✅ **Thermal fine** — QFN24-EP 4×4, θJA=60 °C/W, ~0.9 W max diss.
     at 15 W in → 94 °C junction at 40 °C ambient.
   - Notable: input Vmax operating = 9.5 V (OV trip ~11 V, abs max 25 V);
     trickle current 100 mA for VBAT 3.7–9 V (3S mode); charge timeout
     default 24 h; recommended L = 2.2 µH at 500 kHz Fsw.
