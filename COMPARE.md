# Power module — v1 vs v2-alt head-to-head (compare phase)

Phase 3 of the anti-anchoring experiment. v2-alt's REQUIREMENTS.md +
TOPOLOGY.md were filled in **without** consulting v1. This doc unmasks v1
(`../power_module/`: BOM, INSTRUCTIONS.md, NOTES.md, schematics) and
compares the two independent passes.

**Method note:** v2-alt picks are from a first-principles pass against the
frozen requirements + a stale-but-usable JLCPCB DB (2026-05-06 snapshot).
v1 is a more mature design (schematics drawn, PCB placed/routed, DRC run).
The interesting signal is *where the fresh pass agreed vs. diverged*, not
maturity.

---

## 1. Convergences (independent agreement → validation)

The fresh pass landed on **the same parts** for the two hardest blocks —
without seeing v1. Same manufacturer, same LCSC:

| Block | v1 | v2-alt | LCSC | Verdict |
|---|---|---|---|---|
| **Buck-boost converter** | LM5176 + ext FETs | LM5176 + ext FETs | **C442493** (both) | identical |
| **BMS AFE** | BQ7692003 | BQ7692003 | **C601650** (both) | identical |
| Main inductor | 10 µH | ~6.8–10 µH | — | same ballpark |
| Protection FETs | dual-N SO-8 (C353066, 20 V/6.5 A) | dual-N, ~30 V/≥6.5 A (TBD) | — | same concept |
| Cell mounting | BH-18650-B5BA016 SMD holder | 18650 holder | — | same |
| USB-C connector | TYPE-C-31-M-12 | 16-pin Type-C | — | same family |
| Stackup | 4-layer | 4-layer (GND/PWR inner) | — | same |

**Reading:** the two genuinely hard, expensive-to-get-wrong choices — the
buck-boost topology/controller and the I²C BMS AFE — converged exactly.
That's the strongest possible signal that those choices are *correct for
the problem*, not artifacts of one designer's bias. The governing
constraint (12 V inside the 9.0–12.6 V window ⇒ 4-switch buck-boost) and
the telemetry requirement (⇒ I²C AFE) are strong enough that two
independent passes reach the same silicon.

---

## 2. Divergences (the experiment's payoff)

The charging front-end is where the passes split — and it's not a coin
toss, it tracks a **requirement v1 didn't have to satisfy.**

### 2a. PD sink
- **v1: CH224K** (C970725) — a fixed-function PD *trigger* ("decoy"),
  configured via CFG resistors to request a single target (up to 20 V).
  No host-visible PD status, no I²C.
- **v2-alt: CYPD3177** (C2959321) — EZ-PD BCR sink with a **20→15→9→5 V
  PDO fallback ladder** and I²C status.

### 2b. Charger
- **v1: BQ24650** (C53712) — a **buck-only** sync charge controller.
  Requires Vin above the pack (≳14 V for a 12.6 V 3S pack). Works great
  *given a 20 V source*. No I²C.
- **v2-alt: BQ25703A** (C188229) — a **buck-boost** I²C charger that
  charges the 12.6 V pack from **9 V / 15 V / 20 V alike**, with
  input-power DPM and an I²C ADC for charge V/I telemetry.

### Why they diverged — and which is "more right"
The split is driven by a v2-alt requirement that surfaced *during* the
fresh pass and was then confirmed by the user (2026-06-17):

> "The charger will usually offer 20 V PD, but I want to plug the module
> into less powerful chargers too."

- **Against the original v1 spec** (USB-C PD charging, no weak-source
  clause), **v1's CH224K + BQ24650 is a perfectly valid, cheaper,
  simpler choice.** It is not a bug — with a 20 V charger it charges
  correctly. (I initially suspected a 12 V-request undercharge bug; that
  was wrong — v1 requests up to 20 V.)
- **Against the weak-charger + telemetry requirement, v2-alt is
  strictly better:** the buck-only v1 path *cannot* charge from a source
  that maxes out below ~14 V (e.g. a 9 V-only or non-PD 5 V brick), and
  v1 exposes no charge telemetry. v2-alt's buck-boost + smart-PD handles
  those sources and feeds the I²C telemetry bus the spec calls for.

**Net:** the fresh pass produced a *meaningfully different and better*
charging architecture for the stated I/O + robustness goals — exactly the
outcome the experiment was designed to test for. Cost of the improvement:
BQ25703A buck-boost needs ~4 charger FETs vs the buck's 1–2, +1 inductor,
and CYPD3177 (~$1) over CH224K (~$0.3). Modest $ for real robustness.

---

## 3. Lessons-learned gaps: v1 reality vs v2-alt plan

REQUIREMENTS.md lists constraints "learned from v1 + EMC review." Checking
them against the actual v1 BOM:

| Lesson | v1 actual | v2-alt plan |
|---|---|---|
| USB-C ESD/EMC within 25 mm | **none in BOM** (only a 5.1 k CC pull-down) | SRV05-4 on CC + ~24 V VBUS TVS + CM choke |
| MLCC DC-bias derating on Vout | **single 22 µF/25 V 1206** (~10–13 µF effective at 12 V) | size *effective* C; multiple X7R + bulk |
| Trace width = ampacity | v1 had 33 DRC track-width violations | POWER netclass sized per IPC-2152, verified pre-route |
| Switching return-path GND | (PCB-level; not assessed here) | GND-both-sides + GND↔PWR caps ≤5 mm of vias |

The first two are concrete, visible-in-the-BOM v1 shortfalls that v2-alt's
requirements explicitly close. The ESD gap matches the note that "lack of
USB-C filtering was v1's single biggest EMC finding."

---

## 4. Mechanical

- **v1:** ~80 × 50 mm, cells *beside* electronics (single-side layout
  implied by the SMD holder + components on one board face).
- **v2-alt:** ~75 × 65 mm, **cells on top / SMT on bottom** (opposite
  sides) — shorter pack/tap runs, converter heat kept off the cells, and
  SMT stays single-sided (no double-side-assembly cost step).

Different optimization: v1 minimized board *complexity* (everything one
side); v2-alt minimized *trace length + thermal coupling* by using both
faces. Neither is wrong; v2-alt's is better for the high-current + thermal
constraints, at the cost of a two-sided mechanical assembly.

---

## 5. Scorecard

| Axis | Winner | Note |
|---|---|---|
| Buck-boost converter | tie | identical (LM5176) — validates both |
| BMS | tie | identical (BQ7692003) — validates both |
| Charging robustness | **v2-alt** | buck-boost charges from weak sources |
| Telemetry | **v2-alt** | I²C charger + PD status vs none |
| USB-C ESD/EMC | **v2-alt** | v1 BOM has none |
| Output cap sizing | **v2-alt** | v1 single 22 µF ignores DC-bias derating |
| Simplicity / cost | **v1** | fewer FETs, cheaper PD trigger, 1 inductor fewer |
| Maturity | **v1** | actually drawn, placed, routed, DRC'd |

**Bottom line.** The experiment worked. On the two hardest decisions the
fresh pass *independently reproduced v1* (strong correctness signal). On
charging it *diverged toward a better answer* for the telemetry +
weak-charger goals — a divergence a copy-revise pass would likely have
missed, because it would have inherited v1's "20 V PD + buck charger"
framing. v2-alt also bakes in the EMC/derating/ampacity lessons that v1
learned the hard way. The price is more parts and complexity on the
charge path. If the weak-charger + telemetry requirements are real
(they are, per the user), v2-alt is the better architecture; if the spec
were truly "20 V charger only, no telemetry," v1's simpler path would win
on cost.

### Honest caveats
- v2-alt is a topology/BOM pass only; v1 is a routed board. No v2-alt PCB
  exists yet, so manufacturability/DRC parity is unproven.
- v2-alt's JLCPCB picks used a 6-week-stale DB snapshot; re-verify stock
  before committing a BOM.
- The BQ25703A + CYPD3177 combo is more firmware/config work than CH224K
  + BQ24650; the simplicity cost is real, not just part count.

---

## Post-compare refinement (2026-06-17)

After the compare, the user clarified that battery telemetry comes from
the **BMS** (BQ76920, I²C) — not the charger — and "fully charged" only
needs an **LED**. That removed the I²C-charger justification and exposed
that BQ25703A would have forced an **MCU** (its charge registers boot at 0).
The charger was therefore changed to an **autonomous host-less boost
charger (IP2326)**, no MCU.

**Effect on the scorecard:** the "Simplicity/cost → v1" gap **narrows
sharply**. v2-alt is now also host-less, with a cheaper, higher-stock,
integrated charger that even folds in cell balancing — while *keeping* the
weak-charger win (boost-from-5 V charges from any USB) and the
ESD/derating/ampacity improvements. The remaining v1 edge on simplicity is
mostly maturity (v1 is routed) rather than architecture.

**Meta-observation for the experiment:** the fresh pass first *over-reached*
(BQ25703A buck-boost + smart PD, implicitly an MCU), then a requirements
clarification pulled it back to an architecture **closer to v1's host-less
philosophy but strictly better on the charge path**. The copy-revise route
would likely have kept v1's buck charger and never explored boost-from-5 V;
the from-scratch route over-shot then converged. Both the over-shoot and
the convergence are useful data about how the fresh-pass method behaves.
