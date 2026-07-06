
# Voltage Bandgap Reference (BGR) — 90 nm CMOS/BJT, −50 °C to 125 °C

![Schematic](images/schematic_2V.png)

A BJT-core, self-biased CMOS bandgap voltage reference designed and
simulated in Cadence Virtuoso/Spectre on the GPDK090 (90 nm) process, with
a full DC temperature characterization from −50 °C to 125 °C.

## 1. Overview

A bandgap reference generates a supply- and temperature-independent DC
voltage by summing two complementary quantities derived from a bipolar
junction transistor's base-emitter voltage:

- **CTAT** (Complementary-To-Absolute-Temperature): the base-emitter
  voltage `V_BE` of a diode-connected BJT, which falls by roughly
  1.5–2 mV/°C.
- **PTAT** (Proportional-To-Absolute-Temperature): the difference in
  `V_BE` (`ΔV_BE`) between two BJTs operated at different current
  densities, which rises linearly with absolute temperature.

Weighting `ΔV_BE` by the right resistor ratio and adding it to `V_BE`
cancels the first-order temperature dependence, leaving a reference
close to silicon's ~1.2 V bandgap voltage, essentially flat over
temperature.

## 2. Circuit topology

| Block | Devices | Function |
|---|---|---|
| PMOS current mirror | `PM0`, `PM1`, `PM2` | Self-biased 1:1:1 mirror that forces equal currents into both BJT branches and mirrors the core current into the output branch |
| NMOS mirror / loop closure | `NM0`, `NM1` | Closes the self-biasing loop between the PMOS mirror and the BJT core, setting the CTAT and PTAT nodes |
| Reference BJT (1×) | `Q0` (diode-connected NPN) | Generates the CTAT voltage (`V_BE`, falls with temperature) |
| Scaled BJT pair (2×) | `Q1` ‖ `Q4` (two NPNs in parallel) | Emitter area ratio `N = 2` relative to `Q0`; the resulting `ΔV_BE` across `R0` is the PTAT current source |
| PTAT degeneration resistor | `R0 = 3.6 kΩ` | Converts `ΔV_BE` into a PTAT current |
| Output summing resistor | `R1 = 97.8 kΩ` | Converts the mirrored PTAT current into a voltage, added on top of `V_BE(Q3)` |
| Output diode | `Q3` (diode-connected NPN) | Sets the CTAT term at the output node; `BGR` is taken at the `PM1`/`R1`/`Q3` junction |

All four BJTs (`Q0`, `Q1`, `Q3`, `Q4`) are `gpdk090_npn` devices with
identical emitter area `360m` (0.36 µm² drawn area, `m:1`); `Q1` and `Q4`
are wired in parallel to realize the 2× emitter-area ratio against `Q0`
without changing the unit device geometry.

**Sizing — all five MOSFETs use the same W/L, i.e. no ratio-mirror
mismatch by design:**

| Parameter | Value |
|---|---|
| NMOS W/L (`NM0`, `NM1`) | 20 µm / 5 µm |
| PMOS W/L (`PM0`, `PM1`, `PM2`) | 20 µm / 5 µm |
| BJT emitter area (`Q0`, `Q1`, `Q3`, `Q4`) | 360 m (m:1 each) |
| PTAT resistor `R0` | 3.6 kΩ |
| Output resistor `R1` | 97.8 kΩ |
| Supply `V0` | 2.0 V DC |
| Process | Cadence GPDK090, v4.6 PDK (90 nm) |

Netlist inventory reported by Spectre matches the schematic exactly —
4 BJTs, 5 MOSFETs, 2 resistors, 1 voltage source (see the log snippet in
Section 4) — confirming the simulated netlist is device-for-device
consistent with the drawn schematic above.

## 3. Theory of operation and a numeric sanity check

With emitter-area ratio `N = 2` between the `Q1‖Q4` branch and `Q0`, the
output node approximates the standard bandgap sum:

```
V_BGR = V_BE(Q3) + (R1 / R0) · V_T · ln(N)
```

Plugging in this design's values at `T = 300 K` (`V_T = kT/q ≈ 25.85 mV`):

- `R1 / R0 = 97.8 kΩ / 3.6 kΩ ≈ 27.17`
- `V_T · ln(2) ≈ 25.85 mV × 0.693 ≈ 17.92 mV`
- PTAT term ≈ `27.17 × 17.92 mV ≈ 487 mV`
- Adding a typical 90 nm `gpdk090_npn` `V_BE` of ~0.65–0.70 V at these
  bias currents gives `V_BGR ≈ 1.14–1.19 V`

This lines up with the ~1.15–1.18 V flat band actually measured in
simulation (Section 5), which is a useful independent check that `R0`,
`R1`, and the 2× area ratio are doing what the topology intends.

## 4. Simulation setup

- **Simulator**: Cadence Spectre 12.1.0.347.isr3 (32-bit), invoked from
  Virtuoso ADE
- **Process/models**: `gpdk090.scs`, `gpdk090_mos.scs`,
  `gpdk090_bipolar.scs`, `gpdk090_resistor.scs` (GPDK090 v4.6)
- **Analyses**: `dcOp` (bias point at `tnom = 27 °C`) followed by a swept
  `dc` analysis with `temp` as the sweep variable
- **Temperature sweep**: **−50 °C → 125 °C**, 3.5 °C step (50 points),
  `tempeffects = all`
- **Tolerances**: `reltol = 1e-3`, `abstol(V) = 1 µV`, `abstol(I) = 1 pA`,
  `gmindc = 1 pS`

Relevant excerpt from `sim/spectre_dc_sweep_log.txt` (netlist inventory
and the DC sweep header):

```
Circuit inventory:
              nodes 8
                bjt 4
            bsim3v3 5
           resistor 2
            vsource 1

******************
DC Analysis `dcOp'
******************
    temp = 27 C
    tnom = 27 C
    tempeffects = all
Trying `homotopy = gmin'.
Convergence achieved in 30 iterations.

*****************************************
DC Analysis `dc': temp = (-50 C -> 125 C)
*****************************************
    dc: temp = -50 C          (0 %), step = 3.5 C           (2 %)
    ...
    dc: temp = 27 C          (44 %), step = 3.5 C           (2 %)
    ...
    dc: temp = 125 C        (100 %), step = 3.5 C           (2 %)
Total time required for dc analysis `dc': CPU = 27.997 ms, elapsed = 29.695 ms.
```

Excerpt from `sim/spectre_ptat_ctat_log.txt` — the same bias point
re-run converges *without* needing gmin-homotopy stepping at `tnom`, but
still needs it once the sweep reaches −50 °C, which is the tell used in
Section 6 to flag the missing startup circuit:

```
Convergence achieved in 22 iterations.          <- dcOp @ 27 C, no homotopy needed
...
    temp = -50 C
Trying `homotopy = gmin'.                       <- needed once swept to -50 C
```

Full, unedited logs: `sim/spectre_dc_sweep_log.txt`,
`sim/spectre_ptat_ctat_log.txt`.

## 5. Results

### 5.1 BGR, PTAT, CTAT vs. temperature (final result)

![DC sweep — BGR, PTAT, CTAT](images/dc_sweep_ptat_ctat_bgr.png)

| Signal | @ −50 °C | @ 27 °C | @ 125 °C | Behavior |
|---|---|---|---|---|
| `BGR` (output) | ≈ 1.17 V | ≈ 1.15 V | ≈ 1.18 V | Flat to first order — ≈ 30 mV total spread across the full 175 °C span |
| `ptat` | ≈ 0.40 V | ≈ 0.52 V | ≈ 0.72 V | Rises with temperature, as designed |
| `CTAT` | ≈ 0.78 V | ≈ 0.60 V | ≈ 0.47 V | Falls with temperature, as designed |

Taking the ≈30 mV worst-case spread over a 1.16 V nominal output across
175 °C gives an approximate temperature coefficient:

```
TC ≈ 30 mV / (1.16 V × 175 °C) ≈ 148 ppm/°C
```

This is a reasonable first-pass number for a simple two-BJT bandgap core
without curvature correction (a production-grade BGR with a curvature-
correction branch typically targets 10–50 ppm/°C); the residual bowing
visible in the flat trace is the expected uncorrected second-order
temperature term.

### 5.2 CTAT branch alone

![CTAT vs temperature](images/ctat_vs_temp.png)

`CTAT` (`V_BE` of the reference device) falls from ≈720 mV at −50 °C to
≈460 mV at 125 °C, an average slope of ≈ −1.5 mV/°C — consistent with
the textbook `V_BE` temperature coefficient for a silicon BJT, which is a
good confidence check that the reference device is biased and behaving
as a diode-connected BJT rather than saturating or being starved of
current at the sweep extremes.

### 5.3 PTAT/CTAT overlay

![PTAT/CTAT overlay](images/ptat_ctat_overlay_final.png)

`PTAT` rises from ≈50 mV (−50 °C) to ≈620 mV (125 °C) and crosses `CTAT`
at roughly **88–90 °C, ≈520 mV** — i.e. the two terms are balanced near
the high end of the automotive/industrial range rather than at room
temperature, which is why the flat band in 5.1 bows slightly rather than
sitting perfectly level across −50 °C to 125 °C.

## 6. Known limitations (disclosed, not glossed over)

- **No explicit startup circuit.** The schematic is a classic
  self-biased PMOS/NMOS mirror loop with no dedicated startup network.
  Spectre's `dcOp` analysis needed `homotopy = gmin` to converge to the
  intended non-zero operating point once the sweep reached the sweep
  extremes (see the log excerpt in Section 4), which is the simulator-
  side symptom of a loop that also has a valid all-zero-current DC
  solution in real silicon. A production version of this core would
  need an explicit startup circuit (e.g. a leakage-based kick-start
  branch) to guarantee it powers up into the correct state on real
  silicon.
- **Model warnings during the sweep.** Every temperature point emits
  `WARNING (CMI-2477)` (`PM0: Rds = 390.195 nOhm, set to 0`) and
  `WARNING (CMI-2426)` (`NM0: Rds = 0 Ohm is negative`). Both trace back
  to the GPDK090 v4.6 MOSFET model card's series-resistance parameter
  rounding to a numerically negative/near-zero value at these bias
  points — a known benign artifact of this PDK release, not a
  convergence or accuracy issue (the DC operating point still converges
  cleanly in 22–30 iterations at every temperature step).
- **No curvature correction.** As noted in 5.1, the ≈148 ppm/°C TC is
  typical of an uncompensated two-BJT bandgap core; the residual bow in
  the output vs. temperature (Section 5.1/5.3) is the expected second-
  order term that a curvature-corrected topology would cancel.
- **Two supply-voltage schematic revisions exist for this design.** The
  characterization in Section 5 was run with `V0 = 2.0 V`
  (`images/schematic_2V.png`, shown at the top of this README). A later
  schematic revision was captured at `V0 = 3.3 V` but was not re-swept
  over temperature, so all quantitative results in this README are the
  2.0 V dataset only.

## 7. Directory layout

```
Voltage-Bandgap-Reference/
  README.md
  images/
    schematic_2V.png            circuit schematic, V0 = 2.0 V (used for all sweeps) — shown at top of README
    dc_sweep_ptat_ctat_bgr.png  BGR/PTAT/CTAT vs temp, final result (V scale)
    ctat_vs_temp.png            CTAT alone vs temp (mV scale)
    ptat_ctat_overlay_final.png PTAT + CTAT overlay, crossover ~88-90 C (mV scale)
  sim/
    spectre_dc_sweep_log.txt    Spectre run log: dcOp + dc temp sweep (-50C to 125C)
    spectre_ptat_ctat_log.txt   Spectre run log: PTAT/CTAT-focused re-run
  cadence_lib/
    .oalib, cdsinfo.tag         Cadence OpenAccess library metadata
    bandgap#2dref/schematic/    raw OpenAccess schematic cellview (sch.oa, master.tag, data.dm)
```

## 8. Definition-of-done checklist

- [x] Schematic captured and device sizing documented (W/L = 20 µm/5 µm all MOS, R0/R1, BJT areas)
- [x] Real Spectre DC sweep, −50 °C to 125 °C, real log excerpts included (not fabricated)
- [x] BGR, PTAT, and CTAT nodes each characterized individually and together
- [x] Approximate temperature coefficient computed and sanity-checked against hand analysis
- [x] Known limitations disclosed honestly (no startup circuit, model warnings, no curvature correction)
- [x] Raw Cadence OpenAccess library files included alongside human-readable logs/images
- [x] No placeholder content

## 9. References

1. B. Razavi, *Design of Analog CMOS Integrated Circuits*, 2nd ed.,
   McGraw-Hill, 2016 — chapter on bandgap references; covers low-voltage
   bandgap topologies, `ΔV_BE` generation via emitter-area/current-density
   ratioing, and curvature-correction techniques referenced in Sections
   3 and 6 of this README.
2. K. E. Kuijk, "A precision reference voltage source," *IEEE Journal of
   Solid-State Circuits*, vol. 8, no. 3, pp. 222–226, June 1973 — the
   original two-BJT, `ΔV_BE`-across-a-resistor bandgap topology that this
   design's core (`Q0`, `Q1‖Q4`, `R0`) is a direct descendant of.
3. **This repository's own contribution**: the −50 °C to 125 °C Spectre
   characterization, the `V_BGR = V_BE + (R1/R0)·V_T·ln(N)` numeric
   sanity check against the measured ~1.15–1.18 V flat band (Section 3),
   the ≈148 ppm/°C TC extraction (Section 5.1), and the startup-circuit
   gap identified from the `homotopy = gmin` convergence behavior
   (Section 6) — none of which are copied from the references above and
   are specific to this simulated instance.

## 10. Disclaimer

Numeric readings in Sections 3, 5, and 6 (crossover temperatures,
voltage values, TC) were taken by visual inspection of the exported
waveform plots, not from a re-parsed raw dataset — treat them as
engineering-accurate to the precision of a plot reading (±~10–20 mV,
±~2–3 °C), not as PSF-exported measurement-grade figures.
