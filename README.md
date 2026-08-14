# Engineering Verification / Design Review
## TPS922050D1DGNR LED Driver — Design 1 (30V/800mA) & Design 2 (28.8V/280mA)
**Reference datasheet:** TI TPS922050/TPS922051, SLVSHL9 (November 2024)
**Reviewed document:** "TPS92205x LED Driver Designs" (Design 1 & Design 2)

All values below are **independently recalculated** from the datasheet equations, not copied from Document 2. Where my numbers match Document 2, that is stated explicitly as agreement — not assumed.

---

## 1. Design Requirements Verification

| Item | Design 1 | Design 2 | Category |
|---|---|---|---|
| VIN range | 48V ±10% → 43.2–52.8V | Same | Design input (matches datasheet recommended VIN 4.5–63V — OK, §6.3) |
| VIN,max used in calcs | 52.8V | 52.8V | Correctly computed as 48×1.10 |
| LED string voltage | 30V | 28.8V | Design target, not a datasheet spec |
| LED count/Vf assumed | 9 LEDs × ~3.33V | 9 LEDs × ~3.2V | **Design assumption**, unconfirmed against a real LED BOM part |
| ILED | 800mA | 280mA | Design requirement |
| Switching frequency | 1MHz | 1MHz | **Datasheet spec** — correct for D1DGNR/HVSSOP-8 (Device Comparison Table, p.3): TPS922050D1DGNR = 1MHz, PWM dimming, 1.5A class. Only the DRLR (SOT583) variant of TPS922050 runs at 400kHz. Document 2 got this right. |
| PWM dimming frequency | **Not specified anywhere in Document 2** | **Not specified** | **Gap.** TI's own reference design at 48V/1MHz (§8.2.2.1) specifies 20kHz PWM. Document 2 never states a PWM dimming frequency for either design, even though the task calls it out as something to verify. This must be defined before layout/firmware sign-off, since dimming frequency affects COMP loop settling and minimum on-time compliance (tPWM_IN_ON ≥ 100ns per §6.5). |
| LED current ripple target | ≤20mA | ≤20mA | **Design assumption** (matches TI's own reference-design target, not a hard datasheet limit) |
| Inductor ripple target (KIND) | 30% (0.3) | 30% (0.3) | Design assumption, consistent with TI examples |
| VIN ripple target | **Not stated** | **Not stated** | **Gap** — unlike TI's own reference designs (200mV/24V and 400mV/48V examples), Document 2 never states a target for VIN ripple; it only reports the calculated result. Recommend stating an explicit spec (e.g., ≤400mV, matching TI's 48V precedent) so the CIN choice can be checked against a requirement rather than just reported. |
| IC current-limit margin | Not explicitly required, but computed | Same | See §2 below — recomputed independently |
| Device selection | TPS922050D1DGNR | Same | **Appropriate.** ILIM (TPS922050, §6.5) = 1.4A min / 1.6A typ / 1.8A max — comfortably above both 800mA and 280mA continuous loads, with the peak inductor current (not just average ILED) being the correct comparison (see §2). |

---

## 2. Inductor — Independent Recalculation

Datasheet Eq. (2)/(11): `L = VOUT×(VIN(max)−VOUT) / [VIN(max)×KIND×IL(max)×fSW]` (§8.2.1.2.1 / §8.2.2.2.1, p.16/21)
Ripple Eq. (3)/(12), Peak Eq. (4)/(13), RMS Eq. (5)/(14).

### Design 1 (VOUT=30V, ILED=0.8A)

| Parameter | Independently calculated | Document 2 | Match? |
|---|---|---|---|
| L required | **53.98 µH** | 53.98 µH | ✔ |
| Selected standard L | 56 µH | 56 µH | ✔ (E12, rounded **up** — correct direction, reduces ripple) |
| Actual ripple @56µH | **231.3 mA (28.9%)** | 231.3 mA (28.9%) | ✔ |
| IL(peak) | **0.916 A** | 0.916 A | ✔ |
| IL(rms) | **0.803 A** | 0.803 A | ✔ |

**Current-limit margin, recomputed correctly:** Document 2 expresses margin as (1.4−0.916)/0.916 = 52.8%, i.e. relative to the *design's own peak current*. That is not how margin is conventionally reported to a component-limit spec — expressed against the **limit itself**, the peak inductor current (0.916A) is only **65.4% of the 1.4A minimum cycle-by-cycle current limit**, i.e. **34.6% margin below the limit**. Both statements are numerically consistent, but the datasheet-style convention is the latter. This is a **presentation/terminology note, not a numerical error** — the underlying peak current and limit are both correct, and 34.6% margin is still healthy.

Saturation/RMS ratings selected (Isat ≥2A, Irms ≥1A) satisfy the datasheet's guidance (§8.2.1.2.1): "select a saturation current rating equal to or greater than the converter current limit" (1.8A max) — 2A gives adequate headroom. **Correct.**

### Design 2 (VOUT=28.8V, ILED=0.28A)

| Parameter | Independently calculated | Document 2 | Match? |
|---|---|---|---|
| L required | **155.8 µH** | 155.8 µH | ✔ |
| Selected standard L | 150 µH | 150 µH | — see finding below |
| Actual ripple @150µH | **87.3 mA (31.2%)** | 87.3 mA (31.2%) | ✔ |
| IL(peak) | **0.324 A** | 0.324 A | ✔ |
| IL(rms) | **0.281 A** | 0.281 A | ✔ |

**Finding — Design 2 inductor rounds the wrong direction.** 150µH is *below* the calculated 155.8µH requirement. Rounding down increases ripple rather than decreasing it: the result is 31.2%, which **marginally exceeds the stated 30% KIND target** (a 4% relative overshoot). This is the opposite of what Design 1 did (56µH, rounded *up* from 53.98µH, which reduced ripple below target). I recomputed with the next-larger common stock value, **180µH**: ripple drops to **72.7mA (26.0%)**, comfortably under the 30% target.

- Severity: **Minor** — 31.2% vs. a 30% target is a small overshoot with no functional consequence (LED ripple spec of ≤20mA is still met, see §4), but it is not "acceptable rounding" in the strict sense, because it moves the design in the *unsafe* direction rather than the safe one, unlike Design 1's rounding.
- Recommendation: switch to **180 µH** for strict compliance with the stated 30% ripple ceiling, or explicitly accept 150 µH and update the requirement table to show "≤31.2%" as the as-built ripple.

Saturation/RMS ratings (Isat ≥2A, Irms ≥0.4A) are generously oversized relative to the 0.324A/0.281A actual currents — not wrong, just conservative (shared BOM with Design 1's inductor family). No issue.

---

## 3. Input Capacitors — Independent Recalculation

Datasheet Eq. (6) (§8.2.1.2.2, p.16): `VIN(ripple) = IL(max)×[VOUT/(KDR×CIN×fSW×VIN(max)) + ESR_CIN]`

**Note on equation selection:** the datasheet actually gives *two different* input-ripple equations — Eq. (6) for the 24V/analog-dimming example and Eq. (15) for the 48V/PWM-dimming example (§8.2.2.2.2, p.21): `VIN(ripple) = IL(max) / (2π×fPWM×COUT)`. Eq. (15) is dimensionally and physically inconsistent for input-ripple purposes — it references **COUT** and **fPWM** in a filter equation for VIN ripple, which does not match the physics (input ripple is governed by CIN and the switching frequency fSW, not the output cap and the PWM dimming rate). This appears to be a **copy/paste error in the TI datasheet itself** (the 48V design section reused a mislabeled equation). Document 2 correctly used the physically consistent Eq. (6) equation for both designs instead of literally following the datasheet's Eq. (15) label for the 48V case. **This is the right engineering call, but it should be documented as a deliberate deviation from the datasheet's stated Eq. (15) for the 48V/PWM section, with the reasoning above**, per the "explicitly identify conflicts" instruction.

### Design 1

| Parameter | Recalculated | Document 2 |
|---|---|---|
| VIN ripple (CIN=10µF ceramic, KDR=0.7 assumed, ESR=5mΩ assumed) | **68.9 mV** | 69 mV ✔ |

- **KDR = 0.7 is a design assumption, not a datasheet value** — the datasheet defines KDR only in words ("derating coefficient of ceramic capacitance at applied DC voltage") and gives no numeric table. 0.7 is plausible for a 100V-rated 10µF X7R at 52.8V bias but must be confirmed against the actual manufacturer's DC-bias curve for the selected part number (case size dependent — a 10µF/100V X7R in 1210 could derate to 0.3–0.5, not 0.7, at 52.8V bias). **Flag for confirmation before fabrication.**
- ESR = 5mΩ is likewise an assumption, not from the datasheet. Reasonable for ceramic, but unverified.
- Selected: 10µF electrolytic (bulk) + 10µF X7R ceramic + 0.1µF X7R ceramic, all 100V. Voltage margin: 52.8V/100V = 53% of rating — reasonable and consistent with the datasheet's own examples (100V parts on a 48V rail), though 63V-rated parts would also technically clear "rating > VIN,max" with far less design margin and worse DC-bias derating (52.8/63 = 84%). **100V selection is correct and appropriately conservative.**

### Design 2

| Parameter | Recalculated | Document 2 |
|---|---|---|
| VIN ripple (CIN=10µF ceramic) | **23.2 mV** | 23 mV ✔ |

Selected: 10µF + 0.1µF X7R ceramic, 100V, no mandatory bulk electrolytic (280mA load draws far less surge current) — reasonable; the optional 4.7µF electrolytic for long input leads is a sensible caveat, not a requirement. **No issues.**

---

## 4. Output Capacitor — Independent Recalculation

Datasheet Eqs. (7)–(10): `R_LED = ΔVF/ΔIF × #LEDs`, `Z_COUT = R_LED×ILED(ripple) / (IL(ripple)−ILED(ripple))`, `C_OUT = 1/(2π×fSW×Z_COUT)`, `ILED(ripple) = Z_COUT×IL(ripple)/(Z_COUT+R_LED)`.

**R_LED = 0.5Ω/LED is a design assumption**, not sourced from an actual LED manufacturer datasheet as the TI procedure explicitly requires (§8.2.1.2.3 step 1: "Calculate the total dynamic resistance of the LED string using the LED manufacturer's datasheet"). TI's own reference design used a specific part (Osram WLED, 0.67Ω at 1–2A). Document 2 does not cite an LED part number for either design. **This must be confirmed against the actual selected LED once chosen** — a lower R_LED (stiffer LED string) would require *more* output capacitance than calculated here, and a higher R_LED would need less.

### Design 1 (RLED=4.5Ω assumed, IL(ripple)=231.3mA)

| Parameter | Recalculated | Document 2 |
|---|---|---|
| Z_COUT required | **0.426 Ω** | 0.426 Ω ✔ |
| C_OUT minimum | **0.374 µF** | 0.374 µF ✔ |
| Selected | 0.47µF + 0.1µF = 0.57µF | same |
| ILED ripple @0.57µF (no derating) | **13.5 mA** | 13.5 mA ✔ (<20mA target) |

**Finding — missing DC-bias derating step (Significant).** The datasheet's own 4-step procedure explicitly requires: *"4. Increase the output capacitance appropriately due to the derating effect of applied DC voltage."* Document 2 never applies this step for either design — it verifies ripple using the **nominal, undegraded** 0.57µF value. I recomputed the effect of realistic ceramic DC-bias derating (COUT sees ~30V DC bias on 100V-rated X7R parts):

| Effective capacitance after derating | Resulting LED ripple |
|---|---|
| 100% (0.570µF, as-calculated) | 13.5 mA |
| 80% (0.456µF) | 16.7 mA |
| 65% (0.371µF) | **20.2 mA — exceeds the 20mA target** |
| 50% (0.285µF) | 25.5 mA — clearly exceeds target |

Typical 0.47µF/100V X7R ceramic parts in small case sizes (0603/0805) can lose 40–60%+ of nominal capacitance at 30V DC bias, depending on dielectric formulation and case size — this is not a remote corner case. The nominal 53% margin between the calculated minimum (0.374µF) and the selected value (0.57µF) is **not enough headroom to guarantee the derating margin is absorbed**. **Recommendation:** either (a) pull the actual DC-bias derating curve for the specific 0.47µF/100V part number and confirm effective capacitance stays ≥0.374µF at 30V bias, or (b) increase to 1µF + 0.1µF nominal to build in real margin against derating.

### Design 2 (RLED=4.5Ω assumed, IL(ripple)=87.3mA)

| Parameter | Recalculated | Document 2 |
|---|---|---|
| Z_COUT required | **1.338 Ω** | 1.338 Ω ✔ |
| C_OUT minimum | **0.119 µF** | 0.119 µF ✔ |
| Selected | 0.15µF + 0.1µF = 0.25µF | same |
| ILED ripple @0.25µF (no derating) | **10.8 mA** | 10.8 mA ✔ (<20mA target) |

Design 2 has **2.1×** margin between calculated minimum (0.119µF) and selected (0.25µF) — even at 65% derating, effective capacitance (~0.163µF) still clears the 0.119µF minimum with room to spare. **No correction needed for Design 2**, but the same derating step should still be explicitly documented for completeness/consistency.

**Voltage rating:** COUT sees ~VOUT (30V / 28.8V) across it, not VIN,max. A 100V rating gives 30%/29% of rated voltage — conservative, but a 50V-rated part would also clear the requirement with a smaller/cheaper package and less severe derating penalty at the same absolute DC bias fraction. This is a **possible cost/size optimization**, not a correctness issue — flagging as a recommendation only.

---

## 5. Current-Sense Resistor — Independent Recalculation

Datasheet Eq. (1) (§7.3.2, p.11): `R_SENSE = VREF / ILED_FS`, with **VREF = 200mV typical (193mV min / 207mV max, §6.5 Electrical Characteristics)** at full-scale (D1: 100% PWM; D2: 2V analog input).

### Design 1

| | Recalculated | Document 2 |
|---|---|---|
| R_SENSE | 0.2/0.8 = **0.25 Ω** | 0.25 Ω ✔ (exact — zero standardization error, 0.25Ω is a standard value) |
| Power dissipation | 0.8²×0.25 = **0.16 W** | 0.16 W ✔ |
| Selected | 0.25Ω, 1%, 1W (2512) | 6.25× power margin — appropriate, if slightly conservative |

### Design 2

| | Recalculated | Document 2 |
|---|---|---|
| R_SENSE | 0.2/0.28 = **0.7143 Ω** | 0.714 Ω ✔ |
| Nearest std E96 1% | **0.715 Ω** (+0.09% error) | 0.715 Ω ✔ — negligible/acceptable rounding |
| Power dissipation | 0.28²×0.715 = **0.0561 W** | 0.056 W ✔ |
| Selected | 0.715Ω, 1%, 0.25W (1206) | 4.5× power margin — appropriate |

### Current-error budget (requested in task, absent from Document 2)

Combining the datasheet's VREF tolerance (193–207mV, i.e. **±3.5%** around the 200mV nominal, §6.5) with a 1% sense-resistor tolerance gives a **worst-case (additive) LED-current error of approximately ±4.5%** for both designs (RSS combination would give ≈±3.6%, but worst-case additive is the conservative bound to design to). This was requested explicitly in the review scope and is **not present anywhere in Document 2** — it should be added to the design record, since it directly bounds the LED-current accuracy customers/end-application will see.

**Additional datasheet caveat not addressed in Document 2:** §7.3.2 states *"An offset on VREF need[s] to be considered due to voltage drop on RFLT with common-mode leakage current of CSP and CSN pins."* With RFLT=100Ω and worst-case CSP+CSN leakage of 48µA (§6.5, at VIN=60V, DIM=2V — our VIN,max of 52.8V will be somewhat lower but of the same order), this produces a ≈**4.8mV offset**, or **≈2.4% of the 200mV VREF** — an additional systematic current-sense error not folded into the ±4.5% tolerance budget above. **This should be added to the total error budget** (roughly ±4.5% tolerance + up to ~2.4% RFLT-leakage offset, depending on sign/direction), and is a datasheet-flagged effect that Document 2's sense-resistor section does not mention at all.

---

## 6. Freewheeling (Schottky) Diode — Independent Recalculation

The device is a non-synchronous buck with the FET low-side and a catch diode carrying the inductor's off-time current (§7 functional block diagram, Fig 8-14 reference schematic). Reverse voltage is approximated as VIN(max), consistent with standard non-synchronous buck freewheeling-diode practice for this common-anode floating-LED topology.

### Design 1 (D=VOUT/VIN=30/48=0.625)

| | Recalculated | Document 2 |
|---|---|---|
| Duty cycle D | 0.625 | 0.625 ✔ |
| Iavg (diode) = (1−D)×ILED | 0.375×0.8 = **0.30 A** | 0.30 A ✔ |
| Ipeak (diode) = IL(peak) | **0.916 A** | 0.916 A ✔ |
| VR (reverse voltage) | **52.8 V** (=VIN,max) | 52.8 V ✔ |
| Selected | 100V / 2A Schottky | 100V margin: 1.89× VR; current margin: 2.18× peak, 6.7× avg |

**Assessment:** correct and comfortably rated. The 100V/2A part is generously oversized on current (could be downsized to a 1A-class part for cost/size, since Ipeak is only 0.916A) but this is a conservative choice, not an error — flagged as an optional cost optimization, not a correction.

### Design 2 (D=VOUT/VIN=28.8/48=0.6)

| | Recalculated | Document 2 |
|---|---|---|
| Duty cycle D | 0.6 | 0.6 ✔ |
| Iavg (diode) | 0.4×0.28 = **0.112 A** | 0.112 A ✔ |
| Ipeak (diode) | **0.324 A** | 0.324 A ✔ |
| VR | **52.8 V** | 52.8 V ✔ |
| Selected | 100V / 1A Schottky | 3.1× peak margin, 8.9× avg margin — appropriate |

No corrections needed for either diode's ratings; both are correctly calculated and adequately margined.

---

## 7. COMP / Feedback / Filter Components

| Component | Purpose | Datasheet requirement | Proposed (both designs) | Verification | Recommendation |
|---|---|---|---|---|---|
| **CVCC** | LDO output decoupling | **Mandatory**, exactly "16V, 1µF capacitor" (Table 5-1 pin description) | 1µF, 16V ceramic | **Exact match — this is a strict requirement, not a design choice, and Document 2 complies exactly.** | None — correct as specified. |
| **RFLT** | CSN noise immunity | Datasheet's own example: "a 100Ω resistor is recommended for RFLT" (§8.2.1.2.5, §8.2.2.2.5) — advisory, not a hard limit | 100Ω | Matches datasheet's recommended value exactly | None |
| **CFLT** | CSP–CSN HF noise filter | Datasheet example: "optional 1nF, 50V X7R" | 1nF, 50V X7R | Exact match to datasheet's own optional example | None |
| **CCOMP** | Loop compensation / soft-start | No closed-form sizing equation given anywhere in the datasheet — only qualitative ("different capacitor values determine different softstart times and bandwidths," pin table) | 1nF, 10V X7R | **Cannot be independently verified by calculation — the datasheet provides no CCOMP design equation.** Value matches TI's own worked examples (§8.2.1.2.5, §8.2.2.2.5) exactly, so it is a reasonable, precedented choice, but loop bandwidth/phase margin cannot be confirmed without a full small-signal loop model (which the datasheet does not provide) or bench measurement. | **Missing information — flag for bench verification of loop stability/transient response before production release; cannot be computed from the datasheet alone.** |
| **RCOMP** | Compensation series R | Optional, per datasheet example | 100Ω | Matches datasheet's own optional example exactly | None |
| **RDAMP** | Suppress inductor-current overshoot at PWM turn-on | Datasheet's own 48V/1MHz PWM reference design explicitly states "**A 20MΩ resistor** is chosen for RDAMP" (§8.2.2.2.5, p.22) | 20MΩ (both designs) | Document 2 copies the datasheet's own stated value verbatim. **However, 20MΩ is physically unusual for a damping resistor in a compensation network** (typical RC-damping resistors are Ω–kΩ, not MΩ — at 20MΩ the resistor is nearly an open circuit and would contribute negligible active damping current). This reads as a plausible **typo in the TI datasheet itself** (e.g., possibly intended as 20kΩ or 200Ω), but since Document 2 is following the datasheet's explicit text rather than inventing the value, this is **not an error by Document 2** — it is a datasheet inconsistency worth flagging per the review's "identify conflicts" instruction. | **Confirm 20MΩ against TI (E2E forum / TI support) or bench-test RDAMP's effect on turn-on overshoot before committing to this value in production; do not treat the datasheet text as self-verifying.** |
| **RLK_COMP** | Compensate CSP+CSN common-mode leakage current | Optional, datasheet: "An optional resistor is chosen for RLK_COMP to compensate the CSP+CSN common-mode leakage current" — no value given | 0Ω / DNP, populate if needed | Reasonable default (datasheet gives no sizing guidance) — deferred to bring-up testing | Acceptable as-is; confirm during bring-up if a current offset is observed (consistent with the ~2.4% RFLT-leakage offset noted in §5) |

---

## 8. Circuit Topology Check

**Finding — cannot be fully verified: no schematic was provided.** Document 2 is a text-only design narrative with a component-by-component BOM; it does not include an actual schematic or netlist. I can confirm that:
- The component list matches, one-for-one, every component shown in the datasheet's Figure 8-14 reference schematic (48V/1A/12-LED/PWM design): D, L, RSNS, COUT, CVCC, CIN, RFLT, CFLT, RCOMP, CCOMP, RDAMP, RLK_COMP.
- Pin functions and mandatory connections (VCC→1µF/16V cap, COMP→external RC network, CSP/CSN→sense resistor, SW→inductor+diode) are all present in the BOM as expected for this topology.

What **cannot** be verified without an actual schematic/layout: SW-node polarity of the diode, correct orientation of the common-anode LED string relative to VIN, CSP-vs-CSN pin assignment across RSNS, and placement/routing (short SW trace, GND plane, CSN/CSP trace routing away from the switching node per §8.4.1 layout guidelines). **This must be checked against the actual PCB schematic/layout before fabrication — it is explicitly out of scope of this review because no schematic was supplied.**

---

## 9. Voltage Ratings — Worst-Case Check

VIN,max = 48V × 1.10 = **52.8V** (per task's stated formula, confirmed).

| Component | Sees (worst case) | Selected rating | Margin | Verdict |
|---|---|---|---|---|
| CIN (both designs) | 52.8V | 100V | 1.89× | OK — appropriately conservative for ceramic DC-bias derating |
| COUT (both designs) | ~30V / ~28.8V (VOUT node) | 100V | 3.3×/3.5× | OK, generous — 50V-rated parts would also clear this with smaller size (optimization opportunity, not a defect) |
| CVCC | 5.15V typ (VVCC max) | 16V | 3.1× | OK — matches datasheet's mandatory spec exactly |
| Schottky diode D1 | 52.8V | 100V (both designs) | 1.89× | OK |
| SW node | Switches between ~0V and VIN — absolute max rating on VIN/SW/CSP/CSN pins is 65V (§6.1) | N/A (IC pin, not a BOM part) | 52.8V vs 65V abs-max = 1.23× | **OK but worth flagging: only 23% margin to the IC's own absolute maximum pin rating.** Combined with SW-node ringing/overshoot from parasitic inductance at 1MHz switching, this margin could be eroded by leading-edge spikes. Recommend scoping the SW node on the bench for overshoot above 52.8V once boards are built — this is a normal buck-converter check, not a Document 2 defect, but the 65V abs-max ceiling leaves less headroom than the capacitor/diode voltage selections do. |
| CFLT | Sees CSP–CSN common-mode voltage, order of the VOUT-node level | 50V | Adequate for LED-string voltages in this range (28.8–30V) | OK |
| CSP/CSN pins | Recommended common-mode range 0–63V (§6.3) | N/A | 52.8V max well within 0–63V range | OK |

No voltage-rating violations found. The one item worth extra attention before fabrication is the SW-node margin to the IC's 65V absolute maximum, given typical switch-node ringing at 1MHz.

---

## 10. Thermal / Power Dissipation

The datasheet does not publish gate charge, switching-transition-time, or inductor DCR/core-loss data, so a rigorous switching-loss and full thermal budget **cannot be computed from the datasheet alone** — this is explicitly noted rather than invented, per the review's instructions. What can be estimated from published data:

**Design 1 (800mA, 30V):**
- IC quiescent/operating current: IOP(1MHz) = 3.5mA typ (§6.5) → ≈48V×3.5mA ≈ **0.17W** rough order-of-magnitude bias dissipation (approximation only — actual value depends on internal LDO efficiency, not separately specified).
- FET conduction loss: I_FET,rms ≈ IL,rms×√D ≈ 0.803×√0.625 ≈ 0.635A; at RDSON = 300mΩ typ (up to ~500mΩ at 125°C per Fig 6-8): **≈0.12–0.20W**.
- Switching loss: **cannot be calculated — no Qg/switching-time data published.**
- Diode conduction loss: Vf(Schottky, typ ~0.4–0.5V) × 0.30A(avg) ≈ **0.12–0.15W**.
- RSENSE loss: **0.16W** (calculated above, §5).
- Inductor copper/core loss: **cannot be calculated — inductor DCR and core-loss curve not specified for the selected 56µH part.**
- Rough total *known* IC-package dissipation ≈ 0.17+0.15+0.2 ≈ **0.5W** (excluding unmeasurable switching loss), giving ΔT ≈ 0.5W × 47.8°C/W (RθJA, HVSSOP-8, §6.4) ≈ **24°C** rise. At the 85°C max ambient (§6.3), junction ≈109°C — within the 125°C max Tj (§6.1) but with **only ~16°C headroom**, and this excludes switching loss, which at 1MHz is typically not negligible. **PCB copper area, layer count/thickness, and ambient airflow are not specified in Document 2 and materially affect actual RθJA achieved (the 47.8°C/W datasheet figure assumes a specific JEDEC test board) — this must be confirmed with a thermal image or junction-temp measurement on the actual board before release.**

**Design 2 (280mA, 28.8V):** all loss terms scale down substantially (RSENSE loss alone drops to 0.056W, diode/FET conduction losses drop roughly proportionally to current), so thermal margin is materially better than Design 1's; a full re-check is lower priority but the same missing-data caveats (Qg, inductor core loss, real RθJA) apply.

**Efficiency:** cannot be stated with confidence — the datasheet's own efficiency curves (Fig 6-9/6-10, Fig 8-1) are for different operating points (24V/48V input, 2A output, 20kHz PWM, different LED counts) and are not directly transferable to 800mA/280mA operation at unspecified PWM dimming duty. **Missing information — no efficiency data applicable to these exact operating points exists in the datasheet or Document 2; bench measurement is required.**

---

## DISCREPANCIES FOUND

| # | Component/Parameter | Document 2 value | Independently calculated / correct value | Datasheet basis | Difference | Severity | Recommended correction |
|---|---|---|---|---|---|---|---|
| 1 | Design 2 inductor (L1) | 150 µH | 155.8 µH required; rounds to 31.2% ripple vs. 30% target (150µH), or 180µH giving 26.0% ripple | Eq. (11)/(12), §8.2.2.2.1 | Ripple overshoots target by ~4% relative | **Minor** | Use 180µH for strict 30% compliance, or explicitly accept/document 31.2% as-built ripple |
| 2 | Output capacitor DC-bias derating (Design 1, COUT) | Not applied — ripple verified using nominal 0.57µF | At realistic 65% derating, effective 0.371µF is below the 0.374µF calculated minimum → ripple rises to ~20.2mA, exceeding the 20mA target | Datasheet procedure step 4, §8.2.1.2.3 | Margin (53% nominal) may not survive real ceramic DC-bias derating | **Significant** | Pull the actual DC-bias curve for the chosen part, or increase to 1µF+0.1µF nominal |
| 3 | Output capacitor DC-bias derating (Design 2, COUT) | Not applied | 2.1× margin absorbs realistic derating without issue | Same procedure step 4 | No numerical problem, but step is undocumented | **Minor / documentation gap** | Document the derating check explicitly even though no change is needed |
| 4 | Current-error budget from VREF tolerance + RSENSE tolerance | Not calculated in Document 2 | ≈±4.5% worst-case combined (VREF ±3.5% per §6.5 + 1% resistor tolerance) | §6.5 Electrical Characteristics, VREF min/max | Missing analysis, not a wrong value | **Minor / missing analysis** | Add to design record for both designs |
| 5 | RFLT-induced VREF offset from CSP/CSN leakage current | Not addressed | ≈4.8mV offset (~2.4% of VREF) at 48µA worst-case leakage, RFLT=100Ω | §7.3.2 explicit datasheet caveat; §6.5 ILEAK_CSP/N | Missing analysis | **Minor / missing analysis** | Add to current-accuracy budget; datasheet explicitly calls this out |
| 6 | PWM dimming frequency | Not specified for either design | N/A — must be defined by the design team | Requested in review scope; TI's own reference example uses 20kHz | Requirement gap | **Significant (process gap)** | Specify a PWM dimming frequency and confirm ≥100ns min pulse width (tPWM_IN_ON, §6.5) compatibility with the chosen value |
| 7 | VIN ripple target | Not specified for either design (only reported, not checked against a spec) | N/A | TI's own reference designs specify 200mV/400mV targets | Requirement gap | **Minor (process gap)** | State an explicit VIN ripple spec so 69mV/23mV can be verified as compliant, not just reported |
| 8 | RDAMP = 20MΩ | Copied verbatim from datasheet §8.2.2.2.5 | Physically atypical for a compensation-network damping resistor; likely datasheet typo | Datasheet §8.2.2.2.5 states "20MΩ" | Datasheet-level inconsistency, not a Document 2 error | **Flag for confirmation**, not a correction to Document 2 | Confirm with TI (E2E) or bench-verify PWM turn-on overshoot behavior before committing to production |
| 9 | Current-limit margin presentation | Expressed as % relative to the design's own peak current (52.8%/332%) | Conventionally expressed relative to the limit itself: 34.6% margin (Design 1), much larger for Design 2 | §6.5 ILIM | Same underlying numbers, different presentation convention | **Acceptable / presentation note only** | No numerical correction needed; recommend standardizing to % of limit in future docs |
| 10 | Schematic/topology verification | No schematic supplied in Document 2 | Cannot be checked pin-by-pin against Fig. 8-14 | §8.4.1, Fig. 8-14/8-25 | Missing information | **Missing info — not a defect** | Verify actual schematic/layout against Fig. 8-14/8-25 before fabrication |
| 11 | Thermal/efficiency figures | Not computed in Document 2 | Rough estimate only possible; Qg, inductor DCR/core-loss, and real RθJA all missing | §6.4, Fig 6-8, Fig 6-9/6-10 | Missing information | **Missing info — not a defect** | Bench-measure junction temperature and efficiency on built boards |
| 12 | Inductor rounding direction (Design 1) vs. (Design 2) | 56µH rounds up (safe); 150µH rounds down (unsafe direction) | — | Eq (2)/(11) | Inconsistent standardization philosophy between the two designs | **Minor** | Apply the same "round up to reduce ripple" convention consistently — see item 1 |

**Everything else checked** — sense resistor values (both designs), diode current/voltage ratings (both designs), CVCC/RFLT/CFLT/RCOMP/CCOMP values, and all core inductor/capacitor arithmetic — **matched my independent recalculation exactly**, with no numerical errors found beyond the items listed above.

---

## 11. Corrected BOM

Only line items with an actual change are marked; all other rows are confirmed as-proposed.

### Design 1 — 30V / 800mA

| Ref | Component | Calculated Value | Final Standard Value | Voltage Rating | Current/Power Rating | Package | Reason |
|---|---|---|---|---|---|---|---|
| U1 | TPS922050D1DGNR | — | **No change** | — | ILIM 1.4–1.8A | HVSSOP-8 | Correct device: 1MHz, PWM dimming, adequate current-limit margin for 800mA |
| L1 | Inductor | 53.98 µH | **No change — 56 µH** | — | Isat ≥2A, Irms ≥1A | Shielded, low-DCR | Correct standardization (rounds up, reduces ripple) |
| D1 | Schottky diode | Ipeak 0.916A, VR 52.8V | **No change — 100V/2A** | 100V | 2A (6.7× avg margin) | e.g. MBRS2100T3G | Adequate; could downsize to 1A for cost (optional) |
| RSNS | Sense resistor | 0.25 Ω | **No change — 0.25Ω, 1%** | — | 1W (2512), 6.25× margin | 2512 | Exact calculated value, zero standardization error |
| CIN1 | Bulk input cap | — | **No change — 10µF electrolytic** | 100V | — | — | Surge margin |
| CIN2 | Input filter ceramic | 0.374µF min (per Eq 6 target) | **No change — 10µF X7R** | 100V | — | — | Confirm KDR=0.7 assumption against actual part's DC-bias curve |
| CIN3 | HF input cap | — | **No change — 0.1µF X7R** | 100V | — | — | Standard HF decoupling |
| COUT1 | Output cap | 0.374µF min | **Recommend 1µF X7R** (was 0.47µF) | 100V | — | — | **Correction: builds in margin against DC-bias derating (§4, finding 2)** |
| COUT2 | HF output cap | — | **No change — 0.1µF X7R** | 100V | — | — | — |
| CVCC | VCC decoupling | Mandatory 1µF/16V | **No change** | 16V | — | ceramic | Exact datasheet requirement |
| RFLT | CSN filter | 100Ω per datasheet example | **No change** | — | — | — | Matches datasheet exactly |
| CFLT | CSP–CSN filter | 1nF per datasheet example | **No change** | 50V | — | X7R | Matches datasheet exactly |
| CCOMP | Loop comp | Not independently calculable | **No change — 1nF, 10V X7R** | 10V | — | X7R | Matches TI's own worked example; **bench-verify loop stability** |
| RCOMP | Comp series R | Optional | **No change — 100Ω** | — | — | — | Matches datasheet example |
| RDAMP | PWM overshoot damping | Per datasheet's own value | **No change — 20MΩ, but flag for TI confirmation** | — | — | — | Possible datasheet typo — confirm before production (§7, item) |
| RLK_COMP | Leakage compensation | Optional, 0Ω DNP | **No change** | — | — | — | Populate only if bring-up shows offset |

### Design 2 — 28.8V / 280mA

| Ref | Component | Calculated Value | Final Standard Value | Voltage Rating | Current/Power Rating | Package | Reason |
|---|---|---|---|---|---|---|---|
| U1 | TPS922050D1DGNR | — | **No change** | — | ILIM 1.4–1.8A | HVSSOP-8 | Large current-limit margin (332% relative to peak, or ~77% relative to the 1.4A limit) |
| L1 | Inductor | 155.8 µH | **Recommend 180 µH** (was 150 µH) | — | Isat ≥2A, Irms ≥0.4A | Shielded, low-DCR | **Correction: 150µH rounds below calculated value and pushes ripple to 31.2%, above the 30% target (§2, finding 1). 180µH restores compliance (25.97%).** |
| D1 | Schottky diode | Ipeak 0.324A, VR 52.8V | **No change — 100V/1A** | 100V | 1A (3.1× peak margin) | e.g. MBRS1100T3G | Appropriate |
| RSNS | Sense resistor | 0.7143 Ω | **No change — 0.715Ω, 1%** | — | 0.25W (1206), 4.5× margin | 1206 | Correct nearest-E96 standardization |
| CIN1 | Input filter ceramic | 0.119µF min applies to COUT, not CIN — no CIN minimum was specified beyond ripple target | **No change — 10µF X7R** | 100V | — | — | Adequate; VIN ripple only 23mV |
| CIN2 | HF input cap | — | **No change — 0.1µF X7R** | 100V | — | — | Standard |
| CIN3 (opt.) | Bulk electrolytic | — | **No change — 4.7µF, optional** | 100V | — | — | Only needed for long input leads |
| COUT1 | Output cap | 0.119µF min | **No change — 0.15µF X7R** | 100V | — | — | 2.1× margin — adequately absorbs realistic DC-bias derating |
| COUT2 | HF output cap | — | **No change — 0.1µF X7R** | 100V | — | — | — |
| CVCC | VCC decoupling | Mandatory 1µF/16V | **No change** | 16V | — | ceramic | Exact datasheet requirement |
| RFLT | CSN filter | 100Ω | **No change** | — | — | — | Matches datasheet |
| CFLT | CSP–CSN filter | 1nF | **No change** | 50V | — | X7R | Matches datasheet |
| CCOMP | Loop comp | Not independently calculable | **No change — 1nF, 10V X7R** | 10V | — | X7R | Matches TI's own example; bench-verify |
| RCOMP | Comp series R | Optional | **No change — 100Ω** | — | — | — | Matches datasheet |
| RDAMP | PWM overshoot damping | Per datasheet's own value | **No change — 20MΩ, flag for TI confirmation** | — | — | — | Same datasheet-typo concern as Design 1 |
| RLK_COMP | Leakage compensation | Optional, 0Ω DNP | **No change** | — | — | — | Populate only if needed |

---

## 12. Final Engineering Verdict

### Design 1 (48V → 30V → 800mA): **PASS WITH CHANGES**

The core power-stage math (inductor, input cap, sense resistor, diode) is correct and matches independent recalculation exactly. The one substantive technical risk is the **output capacitor's missing DC-bias derating check** — the as-specified 0.47µF+0.1µF combination has only ~53% nominal margin over the calculated minimum, and realistic ceramic derating at 30V bias could erode that margin enough to exceed the 20mA LED-ripple target. This is fixable by either confirming the actual part's derated capacitance or bumping to 1µF+0.1µF. Several process/documentation gaps (PWM dimming frequency undefined, VIN ripple target undefined, current-error budget not calculated) should also be closed before sign-off, though none of them represent a functional defect in the circuit as specified.

### Design 2 (48V → 28.8V → 280mA): **PASS WITH CHANGES**

Every calculation matches independent recalculation exactly, and every current/voltage margin (current-limit, output-cap derating tolerance, diode, sense resistor) is comfortable. The only technical finding is the **inductor rounding down instead of up** (150µH selected vs. 155.8µH calculated), producing a small (31.2% vs. 30%) ripple-target overshoot — a minor, easily-corrected issue (switch to 180µH), not a functional risk. The same process/documentation gaps as Design 1 (PWM frequency, VIN ripple target, current-error budget) apply here too.

---

### Before PCB fabrication, these parameters/components MUST be corrected or confirmed:

1. **Design 1 COUT1** — confirm actual DC-bias-derated capacitance of the selected 0.47µF/100V part at 30V bias stays ≥0.374µF, or increase to 1µF nominal.
2. **Design 2 L1** — switch to 180µH (or explicitly accept the 31.2% ripple overshoot as a documented deviation from the 30% target).
3. **RDAMP = 20MΩ (both designs)** — confirm this value with TI (it is copied correctly from the datasheet's own example, but the value itself looks anomalous for its stated function); verify PWM turn-on overshoot behavior on the bench before committing to production.
4. **PWM dimming frequency** — not specified anywhere in either design; must be defined and checked against the 100ns minimum-pulse-width spec (tPWM_IN_ON).
5. **LED string assumptions** (9 LEDs × ~0.5Ω dynamic resistance, both designs) — confirm against the actual LED part number once selected; this directly affects the COUT calculation in §4.
6. **CCOMP/loop stability** — cannot be verified analytically from the published datasheet (no loop-compensation equation is given); confirm transient response and stability on the bench.
7. **Schematic/layout verification** — no schematic was supplied for this review; confirm actual net connections and PCB layout against datasheet Fig. 8-14 (schematic) and Fig. 8-25 (layout) before release, particularly SW-node trace length, CSN/CSP routing, and diode orientation.
8. **SW-node absolute-maximum margin** — only ~23% headroom between VIN,max (52.8V) and the IC's 65V absolute maximum pin rating; scope the SW node for turn-off ringing/overshoot on the built board.
9. **Current-error budget and RFLT-leakage offset** — add the ~±4.5% VREF/RSENSE tolerance stack and the ~2.4% RFLT-leakage-current VREF offset (both designs) to the design record; the datasheet explicitly calls out the RFLT-leakage effect and it has not been addressed.
10. **Thermal/efficiency confirmation** — the datasheet does not publish switching-loss or inductor-loss parameters needed for a complete thermal budget; measure junction temperature (or case temperature + RθJC) and efficiency on built prototypes, especially for Design 1 at 800mA where the estimated thermal margin (~16°C to Tj,max at 85°C ambient, excluding switching loss) is the tightest of the two designs.
