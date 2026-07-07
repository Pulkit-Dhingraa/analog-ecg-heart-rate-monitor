# Design Calculations

Every value in the schematic traces back to a requirement. This document shows the derivations,
the assumptions behind them, and the gap between the ideal value and the standard part actually
used. Original handwritten working: [`handwritten-analog-calcs.jpg`](handwritten-analog-calcs.jpg)
and [`handwritten-digital-calcs.jpg`](handwritten-digital-calcs.jpg).

## Design assumptions

These fix the whole design; if any changes, the numbers below change with it.

- ECG at the electrodes: **0.5–5 mV** peak-to-peak, taken as ~1 mV for sizing.
- Heart-rate band of interest: **~0.5–40 Hz** (the QRS energy needed to detect an R-peak — not
  the full 0.05–100 Hz diagnostic band).
- Electrode half-cell DC offset: up to **±300 mV** (Ag/AgCl).
- Dominant interferer: **50 Hz** mains (India), coupled as common-mode.
- Supply: **±5 V** for the analog section, **+5 V** for the digital section.
- Comparator must see the R-peak cross a fixed reference of **~1 V**, so the front-end must lift a
  1 mV ECG to **≥1 V** with margin → target ≈2 V.

---

## 1. AD620 gain (differential amplifier)

The AD620 sets its gain with a single external resistor R_g (datasheet):

```
G = 1 + 49.4 kΩ / R_g
```

Chosen R_g = 510 Ω (standard E24 value):

```
G = 1 + 49400 / 510 = 1 + 96.86 = 97.86
```

**Why ≈98 and not more?** It is a headroom compromise. On a ±5 V rail the AD620 output must not
clip even with the residual DC offset and common-mode ride-through present. A 1 mV ECG × 98 ≈
98 mV keeps the AD620 far from the rail. The catch: 98 mV is also **below the 1 V comparator
reference**, so ×98 alone cannot trigger on a real ECG. The design's answer is to add gain
*after* the DC offset is removed (§4), so the offset itself is never re-amplified. In the measured
hardware that extra gain was not effective (the chain reads unity after the AD620), which is why
validation used a larger function-generator input — see [results.md](results.md).

| R_g | Resulting G | Note |
|---|---|---|
| 100 Ω | 495 | Too high — amplified offset would saturate the AD620 |
| 510 Ω | **97.86** | Chosen |
| 1 kΩ | 50.4 | Usable but needs more post-gain later |

---

## 2. High-pass filter — DC offset blocking

First-order RC high-pass (AC-coupling cap in series, resistor to ground), buffered by a 741:

```
f_L = 1 / (2π R C)
```

**Requirement:** cut *below* the lowest useful ECG component (~0.05 Hz) but block DC. Fixing
C = 10 µF (a common electrolytic) and solving for the exact 0.05 Hz corner:

```
R = 1 / (2π · 0.05 · 10 µF) = 318 kΩ
```

Nearest E24 value is **330 kΩ**, giving the realised corner:

```
f_L = 1 / (2π · 330 kΩ · 10 µF) = 0.0482 Hz
```

Slightly *lower* than 0.05 Hz, which is the safe direction — it preserves more low-frequency ECG
content while still removing the DC offset.

---

## 3. Low-pass filter — noise / EMI / mains attenuation

First-order active LPF (741), same corner equation:

```
f_H = 1 / (2π R C)
```

**Requirement:** cut at 50 Hz to attenuate EMG and mains. Fixing C = 10 nF:

```
R = 1 / (2π · 50 · 10 nF) = 318.3 kΩ
```

318 kΩ is not an E24 value. Practical realisations:

| Realisation | Actual R | Actual f_H |
|---|---|---|
| 330 kΩ (single E24) | 330 kΩ | 48.2 Hz |
| **270 kΩ + 47 kΩ (series)** | 317 kΩ | 50.2 Hz |

The series pair is the accurate choice.

**Roll-off:** first order = 20 dB/decade. So relative to 50 Hz, the response is ≈−6 dB at 100 Hz
and ≈−12 dB at 200 Hz, and only −0.18 dB at the 10 Hz test frequency — i.e. the filter is
essentially flat across the heart-rate band and mild on mains. This is a deliberate trade
(see [component-selection.md](component-selection.md#the-filter-topology-1st-order-vs-5th-order-bessel--twin-t-notch)).

---

## 4. Post-filter gain stage (×21) — design intent vs. hardware

The Proteus schematic shows the second 741 (U22) configured for non-inverting gain:

```
G_2 = 1 + R_22 / R_23 = 1 + 20 kΩ / 1 kΩ = 21
```

**Why the design calls for it.** The AD620 alone (×98) turns a 1 mV ECG into ~98 mV — below the
1 V comparator reference, so the comparator would never fire on a real signal. Adding gain
*after* the high-pass filter (so the DC offset has already been stripped and is not re-amplified)
would raise the total to:

```
G_total = G · G_2 = 97.86 × 21 = 2055 V/V ≈ 66.3 dB
1 mV ECG × 2055 ≈ 2.06 V  →  clears the 1 V reference with ~2× margin.
```

**What was actually measured.** The bench captures read unity through the filter chain
(AD620 1.899 V → HPF 1.859 V → LPF 1.798 V), so the ×21 was **not effective in the measured
hardware** — total measured gain was ≈98. The system was therefore demonstrated with a
function-generator input (~20 mV) large enough that ×98 alone cleared the 1 V reference. The ×21
is the design-stage fix that a real 1 mV ECG would require; it was not hardware-validated. This
gap (98 mV real-ECG output vs. 1 V reference) is the single most important thing to fix before a
live-ECG version — either populate the ×21 stage or drop the reference to ~50–100 mV.

---

## 5. Comparator reference

Resistor divider on the +5 V rail:

```
V_ref = V_CC · R_5 / (R_4 + R_5) = 5 · 10 kΩ / (40 kΩ + 10 kΩ) = 1.00 V
```

Hysteresis was added in hardware: a 1 MΩ resistor from output to the non-inverting input gives
~100 mV of hysteresis, which stops the comparator from multi-counting on the slow flanks of a
near-threshold R-peak.

---

## 6. NE555 monostable — the 60-second window

Counting R-peaks for exactly 60 s makes the count equal BPM directly (no scaling). Monostable
pulse width:

```
T = 1.1 · R · C
```

Fixing C = 100 µF (standard electrolytic) and solving for T = 60 s:

```
R = 60 / (1.1 · 100 µF) = 545,454 Ω ≈ 545 kΩ
```

---

## 7. Seven-segment current limiting

Each CD4026 segment output drives a common-cathode LED through a series resistor. With a ~1.8 V
red-LED forward drop on a 5 V rail and 330 Ω:

```
I_seg = (5 − 1.8) / 330 = 9.7 mA per segment
```

Well within LED and CD4026 output ratings, and bright in normal room light.

---

## Design vs. realised — summary

| Quantity | Target | Standard part | Realised value |
|---|---|---|---|
| AD620 gain | ~98 | R_g = 510 Ω | 97.86 |
| HPF corner | 0.05 Hz | 10 µF + 330 kΩ | 0.048 Hz |
| LPF corner | 50 Hz | 10 nF + 318 kΩ | 50.0 Hz (270k+47k) |
| Post-gain (design only) | ×21 | 20 kΩ / 1 kΩ | 21 — not effective in measured HW |
| Total gain — design | ~2000 | — | 2055 (66 dB) |
| Total gain — measured | — | — | ≈98 (AD620 only) |
| V_ref | 1 V | 40 kΩ / 10 kΩ | 1.00 V |
| Timing window | 60 s | 545 kΩ + 100 µF | 60.0 ± 0.5 s |

---

## Future work

To make the system live-ECG-capable and quieter:

- **PCB with an analog ground plane and guard rings** around the AD620 inputs — the single
  biggest reduction in 50 Hz pickup.
- **Right-leg drive** (one inverter + buffer feeding the RL electrode) to actively cancel
  common-mode and approach clinical CMRR.
- **50 Hz Twin-T notch** ahead of a 100 Hz LPF for explicit mains rejection *and* wider
  diagnostic bandwidth.
- **Rolling R-R average** instead of a single 60 s window for continuous, low-latency BPM.
- **Patient isolation barrier** — mandatory before any human connection.
