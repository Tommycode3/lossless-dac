# lossless-dac

A balanced headphone amplifier board: ±5 V rails from LT3045/LT3094 LDOs and one OPA1622 per channel, designed in [Zener](https://docs.pcb.new) to sit behind off-the-shelf Amanero (USB to I²S) and ES9038Q2M (DAC) modules. The custom board only does the two things that set what you hear at the jack, the analog rails and the output stage; the hard digital parts are bought as modules. Estimated amplifier noise is 1.4 µV RMS differential, which the DAC module will dominate.

Status: schematic captured as code (33 components, builds), layout not started. Nothing fabricated or measured; all performance figures are datasheet-derived.

| Parameter | Value |
|---|---|
| Input power | ±9 V dual rail, 3-pin header (+9 / GND / −9) |
| Analog rails | +4.99 V (LT3045EDD), −4.99 V (LT3094EDD), R_SET = 49.9 kΩ |
| LDO output noise | 0.8 µV RMS (10 Hz–100 kHz), each rail |
| LDO PSRR at 1 MHz | 76 dB (LT3045), 74 dB (LT3094) |
| LDO current limit | 500 mA per rail (R_ILIM_P = 300 Ω, R_ILIM_N = 7.5 kΩ) |
| Power-good (+ rail) | ≈ 4.92 V (93.1 k / 6.04 k on PGFB) |
| Amplifier | 2× OPA1622IDRCR (VSON-10), unity buffer (+) and unity inverter (−) per channel |
| Gain | +6.02 dB differential (×2), 0 dB single-ended (×1) |
| Input coupling | 2.2 µF / 100 kΩ, f₋₃dB ≈ 0.72 Hz |
| Outputs | balanced 5-pin (L+ R+ L− R− GND) for 4.4 mm; TRS (L, R, GND) for 6.35 mm |
| PCB | 4 copper layers |

There is no switching regulator on the board, so the ±9 V must come from a clean external supply. The modules are not on this board either, only the header for the DAC's single-ended L/R output. Connectors for the DAC, power input and both jacks are 2.54 mm pin headers standing in for the real parts.

## Building

Requires the Zener `pcb` toolchain (builds with `pcb` 0.4.6 and 0.4.95; `pcb.toml` pins `pcb-version = "0.4"`). Note that `pcb` may update itself when invoked.

```sh
pcb build lossless-dac.zen    # expect "✓ lossless-dac.zen (33 components)"
pcb layout lossless-dac.zen   # syncs the netlist into layout/layout.kicad_pcb and opens KiCad (--no-open to skip)
```

The Zener standard library and KiCad footprint libraries (~36 MB) are not committed. `pcb build` materialises them into `.pcb/stdlib/` on first run, and `layout/fp-lib-table` points there. Generated files (`layout/default.net`, `layout/snapshot.layout.json`, `layout/*.log`, `layout/.history/`) are ignored.

`pcb build` 0.4.95 currently warns about ten power pins tied to plain `Net`s instead of `Power()`/`Ground()` nets. 0.4.6 also reported no house-part match for the 2.2 µF 0805 capacitor and the 3- and 5-pin headers.

The circuit is in `lossless-dac.zen`, per-part modules, symbols and footprints in `components/`, the KiCad board in `layout/`.

## Rails

The LT3045/LT3094 SET pin sources/sinks a precision 100 µA (±1 %), so $V_{OUT} = \pm 100\ \mu\text{A} \times 49.9\ \text{k}\Omega = \pm 4.99\ \text{V}$. The 4.7 µF on SET filters the reference, which is what gets the regulators down to 0.8 µV RMS. The current limits and power-good threshold:

$$
I_{LIM,+} = \frac{150\ \text{mA·k}\Omega}{0.3\ \text{k}\Omega} = 500\ \text{mA},
\qquad
I_{LIM,-} = \frac{3.75\ \text{A·k}\Omega}{7.5\ \text{k}\Omega} = 500\ \text{mA},
\qquad
V_{PG} = 0.3\ \text{V} \times \left(1 + \frac{93.1\ \text{k}}{6.04\ \text{k}}\right) \approx 4.92\ \text{V}
$$

Headroom is about 4 V, far above the ~260 mV dropout, and it costs dissipation. At idle (OPA1622 $I_Q$ = 2.6 mA per channel, four channels in two duals) each LDO burns $4\ \text{V} \times 4 \times 2.6\ \text{mA} \approx 42\ \text{mW}$. At the 500 mA limit it would be 2 W per DFN package. If heavy loads are expected, the next step is a lower input voltage or using the LT3094's VIOC pin (unused for now) to steer an upstream pre-regulator.

The OPA1622's supply rejection is 0.1 µV/V typical (−140 dB), so rail noise referred to the amplifier input is $0.8\ \mu\text{V} \times 10^{-7} \approx 8 \times 10^{-14}\ \text{V}$. For the op-amps alone these LDOs are overkill. They would start to matter if the DAC module's analog stage were powered from the same rails, and that is the better argument for keeping them.

## Output stage

Unit A is a follower and unit B an inverter with $R_F = R_G = 1\ \text{k}\Omega$ fed from unit A, so $V_{L+} = V_{in}$, $V_{L-} = -V_{in}$ and $V_{\text{diff}} = 2V_{in}$ (+6.02 dB). The single-ended jack takes $V_{L+}$ directly. The input high-pass is $1/(2\pi \cdot 100\ \text{k}\Omega \cdot 2.2\ \mu\text{F}) \approx 0.72\ \text{Hz}$, ignoring the module's output impedance.

For noise, take $e_n = 2.8\ \text{nV}/\sqrt{\text{Hz}}$ at 1 kHz and 4.07 nV/√Hz for a 1 kΩ resistor at 300 K. Unit A's noise appears at both L+ and L−, so it doubles in the differential output; unit B runs at a noise gain of 2; each resistor reaches the output with gain 1:

$$
e_{o,\text{diff}} = \sqrt{(2 e_{n,A})^2 + (2 e_{n,B})^2 + e_{R_F}^2 + e_{R_G}^2}
= \sqrt{5.6^2 + 5.6^2 + 4.07^2 + 4.07^2} \approx 9.8\ \text{nV}/\sqrt{\text{Hz}}
$$

Current noise (0.8 pA/√Hz into 1 kΩ, 0.8 nV/√Hz) is negligible, and so is the 100 kΩ bias resistor, which the 2.2 µF and a low-impedance source shunt above a few hertz (about 0.03 nV/√Hz at 1 kHz). Over 20 Hz–20 kHz that gives $9.8\ \text{nV}/\sqrt{\text{Hz}} \times \sqrt{20\ \text{kHz}} \approx 1.4\ \mu\text{V RMS}$.

At ±5 V the OPA1622 datasheet gives 95 mW into 16 Ω and 150 mW into 32 Ω single-ended, both current-limited, and 11.3 mW into 600 Ω, voltage-limited. Bridging doubles the voltage across the load but not the current through it, so the balanced output gives about 4× the power when voltage-limited and about the same when current-limited:

| Load | Single-ended (datasheet) | Balanced (estimate) |
|---|---:|---:|
| 32 Ω | 150 mW | ≈ 150 mW (current-limited) |
| 600 Ω | 11.3 mW | ≈ 45 mW (voltage-limited) |

Balanced drive buys headroom for high-impedance headphones; for 16–32 Ω it gives symmetric drive and no extra power. Datasheet THD+N single-ended at those maximum powers is −118.7 dB (32 Ω) and −119.4 dB (600 Ω), and −132 dB at 1 kHz, G = 1, 3.5 V RMS into 2 kΩ. Taking 45 mW into 600 Ω as full scale ($V_{FS} \approx 5.2\ \text{V RMS}$), the amplifier-only unweighted dynamic range is $20 \log_{10}(5.2\ \text{V} / 1.4\ \mu\text{V}) \approx 131\ \text{dB}$, an upper bound the DAC module will set in practice.

## Not done yet

- Real connectors (4.4 mm, 6.35 mm, DC input, module headers) not chosen.
- Layout: footprints auto-placed, no outline, no routing, no pours.
- No design review or simulation.
- No Gerber or BOM/CPL; the board cannot be ordered.
- No measurements.

## License

Hardware design files (Zener sources, symbols, footprints, KiCad layout) and documentation: CERN-OHL-S-2.0 ([`LICENSE`](LICENSE)).

The LT3045/LT3094/OPA1622 footprints and symbols in `components/` are adapted from the KiCad libraries (CC-BY-SA 4.0 with the KiCad libraries exception) and remain under that licence. Datasheets are the property of Analog Devices and Texas Instruments and are not redistributed here.
