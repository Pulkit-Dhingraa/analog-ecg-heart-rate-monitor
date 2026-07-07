# Bill of Materials

## Active devices

| Item | Qty | Part | Role | Datasheet |
|---|---|---|---|---|
| Instrumentation amplifier | 1 | AD620AN | Differential front-end, G ≈ 97.86 | Analog Devices AD620 |
| Op-amp | 2 | LM741CN | HPF buffer + LPF/gain stage | TI LM741 |
| Dual comparator | 1 | LM393N | R-peak → TTL pulse | TI LM393 |
| Timer | 1 | NE555P | 60 s monostable window | TI NE555 |
| Hex inverter | 1 | SN7404N | Inverts 555 output for CD4026 INH | TI SN7404 |
| Decade counter / 7-seg driver | 2 | CD4026BE | Units + tens, cascaded | TI CD4026B |
| 7-segment display (common cathode) | 2 | Kingbright SA52 | BPM digits | Kingbright SA52 |

## Passives

| Item | Value(s) | Notes |
|---|---|---|
| R_g (AD620 gain) | 510 Ω | Sets G = 97.86; mount directly across pins 1–8 |
| HPF resistor | 330 kΩ | With 10 µF → f_L ≈ 0.048 Hz |
| HPF cap | 10 µF electrolytic | AC coupling |
| LPF resistor | 318 kΩ (270 kΩ + 47 kΩ) | With 10 nF → f_H ≈ 50 Hz |
| LPF cap | 10 nF ceramic | |
| Post-gain | 20 kΩ / 1 kΩ | Sets ×21 on U22 (Proteus design; unity in measured build) |
| V_ref divider | 40 kΩ / 10 kΩ | → V_ref = 1.00 V |
| Comparator pull-up | 10 kΩ | LM393 open-collector |
| Comparator hysteresis | 1 MΩ | Output → non-inverting input, ~100 mV |
| 555 timing R | 545 kΩ (510 kΩ + 33 kΩ) | With 100 µF → T ≈ 60 s |
| 555 timing C | 100 µF electrolytic | Low-leakage preferred |
| 555 trigger pull-up | 10 kΩ | |
| 555 debounce / CV bypass | 10 nF ×2 | Trigger debounce + pin-5 bypass |
| Segment current-limit | 330 Ω ×14 | ~9.7 mA/segment |
| Decoupling | 0.1 µF ceramic ×8, 10 µF electrolytic | One 0.1 µF at every IC |

## Infrastructure

| Item | Qty | Notes |
|---|---|---|
| Push-buttons | 2 | TRIGGER, RESET |
| Breadboards | 2 | MB-102, analog + digital separated |
| Power supply | 1 | Dual ±5 V (analog) + single +5 V (digital) |

## Notes

- **Analog and digital grounds** meet at a single star point near the supply entry to avoid
  ground loops coupling digital switching into the mV-level ECG.
- All CD4026 unused control inputs must be tied (DEI high; MR defined via pull-down + reset
  button) — floating CMOS inputs cause erratic counting.
