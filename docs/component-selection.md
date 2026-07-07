# Component Selection & Defense
A good designers always compares and justifies his component , So-
For each major part: *why it was chosen, what else was on the table, the trade-offs, and where it
falls short.* This is the "why this over that" 
---

## AD620 — instrumentation amplifier (the front-end)

**The job.** The ECG is a ~1 mV *difference* between two electrodes, sitting on top of hundreds
of millivolts of 50 Hz mains hum that appears almost equally on both electrodes (common-mode).
The front-end's real job is therefore **rejection** — pulling out the tiny differential signal
while throwing away the large common-mode — not just gain. That points straight at an
instrumentation amplifier.

**Why the AD620 specifically.**
- **High CMRR:** ~100 dB at G = 100 — this is what rejects the 50 Hz that rides equally on both
  leads.
- **One-resistor gain:** G = 1 + 49.4 kΩ/R_g, so gain is set with a single R_g and no matched
  resistor network to trim.
- **Low input-referred noise:** ~9 nV/√Hz at 1 kHz, and low input bias current (≤1 nA), suited
  to the high source impedance of skin electrodes.
- **Availability:** stocked in essentially every Indian student lab, which mattered for a
  breadboard project.

**Alternatives considered.**

| Option | Why not chosen |
|---|---|
| Three discrete op-amps in a classic 3-op-amp IA | Cheapest in parts, but CMRR is capped by resistor matching — even 0.1 % resistors give only ~60 dB, which is not enough to reject mains on a 1 mV signal. Also more board area and more to go wrong. |
| AD8220 (JFET-input IA, used in the reference paper) | Excellent, but pricier and far less available locally; its JFET-input advantage (ultra-low bias current) is not the bottleneck here. |
| INA128 / INA333 | Genuinely comparable — either is a valid drop-in. AD620 won on lab availability and familiarity, not on spec. |
| Op-amp difference amplifier (single op-amp, 4 resistors) | Low input impedance and CMRR entirely dependent on resistor matching — unsuitable for electrode-level source impedance. |

**Trade-offs / limitations.**
- The AD620 gives **no DC-offset rejection** — the ±300 mV electrode offset is amplified along
  with the signal. This is *by design here*: the high-pass filter downstream removes it. It is
  the reason the design keeps the AD620 gain moderate (×98) and adds any further gain *after* the
  high-pass filter, rather than taking the whole system gain at the input where it would saturate
  on the amplified offset. (In the measured build the gain was the AD620's ×98 only — see
  [design-calculations.md §4](design-calculations.md).)
- Single-supply operation would need a mid-rail reference; this build uses ±5 V to keep the
  input range symmetric around ground.

**Verdict.** Best CMRR-per-cost-per-availability for a single-lead teaching build. Any of AD620 /
INA128 / INA333 would work; the AD620 was the pragmatic pick.



---

## The filter topology: 1st-order vs. 5th-order Bessel + Twin-T notch

The reference design (Du & Jose, 2017) uses a **5th-order Bessel LPF and a 60 Hz Twin-T notch**.
This project deliberately replaced that with **two first-order sections**.

**Why simplify.**
- The reference chain needs 9–12 op-amps and several precision capacitors — overkill for a
  demonstrator and for heart-rate counting.
- Its notch is at **60 Hz** (North-American mains). In India (**50 Hz**) it would have to be
  redesigned anyway.
- For *heart-rate counting*, the only requirement is that the QRS R-peak stays above the
  comparator reference. Amplitude fidelity of the full PQRST morphology — which the Bessel filter
  protects — is not needed.

**The trade-off, stated honestly.** First-order filtering leaves more residual 50 Hz hum than a
notch would. That is acceptable *only because* the comparator (with hysteresis) ignores sub-
threshold ripple as long as the R-peak clears V_ref. For a diagnostic-quality trace, the notch
and higher-order LPF would be reinstated. This is the central engineering judgement of the
project: **match the filter to the task, not to the textbook.**

---

## LM393 — comparator (R-peak → pulse)

**The job.** Convert each R-peak into one clean, full-swing logic pulse to clock the counter.

**Why the LM393.**
- Fast enough (<1.3 µs propagation) and runs off the same +5 V as the digital section.
- Dual channel, cheap, everywhere.
- **Open-collector output** — with a pull-up to +5 V it produces a clean TTL-level swing suited
  to a CMOS clock input.

**Trade-offs / limitations.**
- Open-collector needs an external pull-up (10 kΩ here) — its rise time is set by that resistor
  and stray capacitance, so the pull-up is sized for a fast enough edge.
- A bare comparator on a slowly varying input will **chatter** near threshold. Fixed with ~100 mV
  of hysteresis (1 MΩ output-to-noninverting feedback) — turning it into a Schmitt trigger. This
  is the classic comparator gotcha and worth calling out.

**Alternative:** using an op-amp as a comparator — rejected; op-amps aren't specified for
comparator use, saturate slowly, and can behave badly with rail-to-rail overdrive.

---

## NE555 — the 60-second timing window

**The job.** Open a precise one-minute gate so the raw R-peak count *is* the BPM.

**Why the 555 (monostable).** One IC, one R, one C set a clean single-shot pulse of
T = 1.1·R·C. Push-button triggered, which exactly matches the intended user action:
press → wait 60 s → read.

**Trade-offs / limitations.**
- Timing accuracy is only as good as the RC — and a 100 µF electrolytic has ±20 % tolerance plus
  leakage that drifts with temperature. Mitigated by trimming R and characterising against a
  stopwatch (achieved 60.0 ± 0.5 s).
- Push-button bounce can false-retrigger; fixed with a 10 nF debounce cap on the trigger pin.

**Alternatives.** A crystal + counter chain would give far better absolute timing accuracy, but
adds parts and complexity for a benefit (sub-1 % window accuracy) that heart-rate counting does
not need — a ±0.5 s window is ±0.8 % of a minute, i.e. <1 BPM.

---

## 7404 — the gate

**The job.** The CD4026's clock-inhibit (INH) pin disables counting when HIGH. The 555 output is
HIGH during the active window, which is the wrong polarity. A single 7404 inverter flips it so INH
is LOW (counting enabled) exactly during the window.

**Why an inverter and not AND-gate logic.** It's the minimum part count: one gate replaces a
gating network. Clean and cheap.

---

## CD4026 ×2 — decade counter + 7-segment driver

**The job.** Count pulses and drive the display, with no decoder logic in between.

**Why the CD4026.** It integrates a decade counter *and* a seven-segment decoder/driver in one
16-pin package — so it drives a common-cathode display directly. Cascading is trivial: the
CARRY-OUT of the units digit clocks the tens digit, giving a 00–99 range that covers normal
(60–100 BPM) and most tachycardic rates.

**Trade-offs / limitations.**
- **Two-digit range only.** Above 99 the cascaded pair **rolls over 99 → 00** (it does not freeze).
  A third CD4026 would extend the range; for resting heart rate two digits are enough.
- CMOS, so all unused control pins (DEI high, MR defined) must be tied — floating inputs cause
  erratic counting. Managed with pull-downs and a reset button on MR.

**Alternative:** a 74xx counter + 7447 decoder + display — more chips for the same result. The
CD4026's counter-plus-decoder integration is exactly why it was chosen.

---

## One-line summary

Every part was chosen for the **minimum complexity that satisfies the heart-rate task on parts Ican actually buy from pocket.** — AD620 for CMRR, first-order filters instead of a Bessel+notch,
LM393+555+7404+CD4026 to replace a microcontroller and a PC entirely.
