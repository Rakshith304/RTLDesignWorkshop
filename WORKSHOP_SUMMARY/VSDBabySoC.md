# VSD BabySoC — Static Timing and Physical-Design Report

## Executive summary

This document combines the supplied synthesis-stage and post-route static timing
analysis (STA) reports for VSDBabySoC. The design combines the `rvmyth` RISC-V
core with the AVSD PLL and AVSD DAC macros and targets SKY130 HD standard cells.
Both reports use an 11.00 ns clock target (about 90.91 MHz) and analyze 16
process-voltage-temperature (PVT) corners.

The synthesis-stage report finds setup timing met in 8/16 corners and hold timing
met in all 16. The later routed-database report finds setup timing met in 10/16
corners and hold timing met in all 16. The routed report still records 28 met1
short violations, so the design is **not DRC-clean or ready for sign-off**.

![RTL-to-GDSII flow overview](../Week3/flow.png)

## Source reports and scope

| Source report | Date | What it contains |
|---|---|---|
| `VSDBabySoC_STA_Report.pdf` | 19 September 2026 | OpenSTA results for the synthesized gate-level netlist across 16 SKY130 HD corners |
| `Post_Route_STA_Report_vsdbabysoc_15.pdf` | 10 October 2026 | OpenROAD post-route-database results across 16 corners, plus flow and route-quality measurements |

The PDFs were supplied from `C:\vlsid\tcl19-09-26\` and
`C:\vlsid\10-10-26\`, respectively. Corner-specific paths and some raw flow
outputs are described in the source reports; this document summarizes their
reported results. The synthesis and post-route results are not perfectly
like-for-like because the analyzed netlists and clock/parasitic assumptions differ.

The PDFs are report artifacts, not the design source itself. They do not include
enough RTL, constraints, macro models, or build scripts to reproduce the whole
SoC from this repository alone. Values below are attributed to the dated reports
and should be read as snapshots from those runs.

## Design and STA concepts

STA propagates arrival and required times through timing paths without needing
simulation vectors. For setup, data must arrive before the capture edge:

```text
Setup slack = required time - arrival time
```

For hold, data must not arrive too soon after the capture edge:

```text
Hold slack = arrival time - required time
```

Positive slack meets that check; negative slack violates it. Setup is commonly
worst in slow-cell, low-voltage conditions. Hold is commonly stressed by fast-cell,
high-voltage conditions. A single typical-corner pass is not enough to establish
timing across the supported operating range.

### Interpreting PVT corner names

A corner selects a characterized delay model for process variation, temperature,
and supply voltage:

- `ff` means fast-fast devices; shorter delays generally make this family
  relevant to hold analysis.
- `tt` means typical-typical devices, including the nominal
  `tt_025C_1v80` point.
- `ss` means slow-slow devices; longer delays generally make this family
  important for setup analysis.
- `n40C`, `025C`, and `100C` describe -40 °C, 25 °C, and 100 °C.
- `1v28`, `1v80`, and similar suffixes encode the supply voltage.

The synthesis-stage report observes temperature inversion at low voltage: its
cold slow-corner result can be worse than its hot slow-corner result. Do not
assume one temperature is always worst; use the supplied Liberty timing corners
and the operating range required by the design.

### WNS, TNS, and independent setup/hold checks

WNS is the worst slack among the paths checked. TNS is the sum of negative slack
over violating paths for that check. A very negative WNS with a very negative
TNS can indicate a broad timing problem rather than one isolated path. Positive
hold slack cannot cancel negative setup slack: setup and hold are independent
requirements and each must pass at the appropriate corner.

![Setup timing with an ideal clock](../Week-3-Module4/theory_images/theory_02_setup_analysis_ideal_clock.png)

![Setup timing with a propagated clock](../Week-3-Module4/theory_images/theory_08_setup_analysis_real_clock.png)

![Hold timing with a propagated clock](../Week-3-Module4/theory_images/theory_09_hold_analysis_real_clock.png)

## Synthesis-stage STA: 19 September 2026

The first report used OpenSTA 3.1.0 on the synthesized gate-level netlist with an
11.00 ns SDC clock period. The same netlist and constraints were used while
changing the standard-cell Liberty timing corner. The 16 corner runs reported
hold timing met at every corner; setup timing was met at all FF and TT corners
and violated at all eight SS corners.

| Corner | Setup slack (ns) | Setup TNS (ns) | Hold slack (ns) | Setup |
|---|---:|---:|---:|---|
| `ff_n40C_1v95` | +3.9382 | 0.00 | +0.1875 | Met |
| `ff_100C_1v95` | +3.7659 | 0.00 | +0.1960 | Met |
| `ff_n40C_1v76` | +2.8565 | 0.00 | +0.2243 | Met |
| `ff_100C_1v65` | +2.3464 | 0.00 | +0.2491 | Met |
| `ff_n40C_1v65` | +1.9603 | 0.00 | +0.2551 | Met |
| `ff_n40C_1v56` | +0.9378 | 0.00 | +0.2915 | Met |
| `tt_100C_1v80` | +1.0230 | 0.00 | +0.3145 | Met |
| `tt_025C_1v80` | +0.9581 | 0.00 | +0.3096 | Met |
| `ss_n40C_1v76` | -4.7641 | -2,231.95 | +0.5038 | Violated |
| `ss_100C_1v60` | -6.5384 | -3,224.37 | +0.6420 | Violated |
| `ss_n40C_1v60` | -11.1771 | -5,759.85 | +0.6628 | Violated |
| `ss_100C_1v40` | -13.4015 | -8,275.72 | +0.9053 | Violated |
| `ss_n40C_1v44` | -27.3124 | -14,732.52 | +0.9909 | Violated |
| `ss_n40C_1v40` | -33.2707 | -18,474.39 | +1.1249 | Violated |
| `ss_n40C_1v35` | -45.5748 | -24,946.91 | +1.3475 | Violated |
| `ss_n40C_1v28` | -76.8905 | -39,451.23 | +1.8296 | Violated |

The most severe setup corner is `ss_n40C_1v28`: WNS is -76.8905 ns and TNS is
-39,451.23 ns. The large negative TNS indicates widespread violations, rather
than one isolated path. All corners meet hold, and setup is met in the fast and
typical families.

The best setup margin is +3.9382 ns at `ff_n40C_1v95`; the worst hold slack is
+0.1875 ns at that same corner. Setup violations range from -4.7641 ns at
`ss_n40C_1v76` to -76.8905 ns at `ss_n40C_1v28`. That corner's TNS indicates
many failing endpoints and substantial slow-corner timing risk at the 11 ns
target, despite the passing hold checks in this synthesis-stage snapshot.

## Post-route STA: 10 October 2026

The second report used OpenROAD v2.0-25794, the routed database `5_route.odb`,
propagated clocks, and an 11.0 ns period. It analyzed the same 16 named
standard-cell corners. This report states that wire RC parasitics were estimated
from global routing with the platform `setRC.tcl`; it did not use SPEF extracted
from detailed routing.

| Corner | Setup slack (ns) | Setup TNS (ns) | Hold slack (ns) | Min period (ns) | Fmax (MHz) | Setup |
|---|---:|---:|---:|---:|---:|---|
| `ff_100c_1v65` | +7.2474 | 0.00 | +0.2615 | 3.75 | 266.48 | Met |
| `ff_100c_1v95` | +8.1128 | 0.00 | +0.2067 | 2.89 | 346.36 | Met |
| `ff_n40c_1v56` | +5.9962 | 0.00 | +0.3056 | 5.00 | 199.85 | Met |
| `ff_n40c_1v65` | +6.7210 | 0.00 | +0.2693 | 4.28 | 233.70 | Met |
| `ff_n40c_1v76` | +7.3479 | 0.00 | +0.2376 | 3.65 | 273.82 | Met |
| `ff_n40c_1v95` | +8.0564 | 0.00 | +0.1998 | 2.94 | 339.72 | Met |
| `ss_100c_1v40` | -3.4532 | -583.34 | +0.9219 | 14.45 | 69.19 | Violated |
| `ss_100c_1v60` | +1.2683 | 0.00 | +0.6574 | 9.73 | 102.76 | Met |
| `ss_n40c_1v28` | -28.2831 | -14,827.03 | +1.7165 | 39.28 | 25.46 | Violated |
| `ss_n40c_1v35` | -16.0734 | -7,001.72 | +1.3140 | 27.07 | 36.94 | Violated |
| `ss_n40c_1v40` | -10.7110 | -3,705.87 | +1.1194 | 21.71 | 46.06 | Violated |
| `ss_n40c_1v44` | -7.6047 | -2,125.70 | +0.9896 | 18.60 | 53.75 | Violated |
| `ss_n40c_1v60` | -0.5273 | -13.45 | +0.6702 | 11.53 | 86.75 | Violated |
| `ss_n40c_1v76` | +2.7816 | 0.00 | +0.5008 | 8.22 | 121.68 | Met |
| `tt_025c_1v80` | +6.1344 | 0.00 | +0.3245 | 4.87 | 205.52 | Met |
| `tt_100c_1v80` | +6.2103 | 0.00 | +0.3269 | 4.79 | 208.78 | Met |

Ten corners meet setup and all 16 meet hold. The worst routed setup result is
`ss_n40c_1v28` at -28.2831 ns, with TNS -14,827.03 ns and an estimated minimum
period of 39.28 ns. Its reported Fmax is 25.46 MHz. These numbers are corner- and
constraint-specific, not a general guarantee of operating frequency.

At the nominal `tt_025c_1v80` corner, the report gives +6.1344 ns worst setup
slack and +0.3245 ns worst hold slack. The example worst setup path ends at
`CPU_Xreg_value_a4[9][18]`, in the register-file endpoint family.

All six setup-failing routed corners are SS corners: `ss_100c_1v40` and
`ss_n40c_1v28`, `1v35`, `1v40`, `1v44`, and `1v60`. The slow hot corner
`ss_100c_1v60` passes at +1.2683 ns, while `ss_n40c_1v60` narrowly fails at
-0.5273 ns. `ss_n40c_1v76` passes at +2.7816 ns. The minimum hold slack across
the routed set is +0.1998 ns at `ff_n40c_1v95`; the checks pass, but the margin
must still be judged against final constraints and sign-off methodology.

### Critical path and clock assumptions

At `tt_025c_1v80`, the post-route report identifies a worst setup path ending at
`CPU_Xreg_value_a4[9][18]`, within the register-file endpoint family also seen
in placement/CTS observations. The report gives propagated clock delay of about
0.88 ns, close to an earlier CTS-stage value near 0.90 ns. Propagated-clock
analysis includes inserted clock-tree delay at the sinks rather than assuming
ideal, simultaneous clock arrival.

These results remain conditional on the STA models. The report uses an 11 ns
clock and standard-cell corners, while the AVSD PLL and DAC macro libraries were
available at a single characterization condition and shared across corner runs.
That limits conclusions about full-chip PVT sign-off.

## Physical-design results recorded by the post-route report

The flow summary describes an OpenROAD-flow-scripts run for `sky130hd/vsdbabysoc`
with a 2000 µm × 2000 µm die. The report records:

| Stage / metric | Reported result |
|---|---|
| Synthesis standard-cell area | 53,019.6 µm²; macro area is unknown in Yosys |
| Sequential cell area | 22,901.96 µm² (43.20%) |
| Floorplan | Die 2000 × 2000 µm; core from (20,20) to (1990,1990) µm |
| Fixed macros | DAC at (100.225, 100.105) µm; PLL at (1700.01, 100) µm |
| Final placement | 48,747 instances; 724,388 µm² design area; 19% utilization |
| Clock tree | 1,144 sinks; 112 `clkbuf_16` buffers; reported setup skew 0.21 ns |
| Detailed route | 64 iterations; 237,296 µm wire length; 50,061 vias |
| Antenna checks | 0 net and 0 pin violations |
| Detailed-route shorts | 28 unresolved met1 short violations |

The report's first-run typical-corner stage table gives +6.08 ns setup slack at
placement and +6.04 ns after CTS; the final routed run gives +6.1344 ns. Those
stage data are not all from the same final run, as the report explicitly notes.
The final route remained at 28 met1 shorts after the 64-iteration limit. The
design is therefore not DRC-clean.

The flow report also records 48,747 placed instances, about 19% utilization,
1,144 clock sinks, 112 clock buffers, 237,296 µm total detailed-route wire
length, and 50,061 vias. It reports zero antenna net/pin violations. Those
results do not override the shorts: producing a routed database and reporting no
antenna violations do not mean the layout passed DRC or connectivity sign-off.

The report notes about 8% global-placement routing overflow and identifies
congestion as a likely contributor to the remaining shorts. It does not prove
the root cause of each violation. Inspect DRC markers before deciding whether to
adjust pin access, blockages, routing constraints, or macro placement.

![Clock-tree buffering and RC model](../Week-3-Module4/theory_images/theory_03_cts_buffering_rc_model.png)

![Routing process overview](../Week3-Module5/theory_images/theory_05_tritonroute_flow.png)

![Illustrative full routed layout](../Week-3-Module4/theory_images/theory_13_route_full_layout.png)

The routed-layout illustration above is a theory image from the workshop, not
proof that this BabySoC run passed routing sign-off.

## Interpreting the synthesis-to-route comparison

Across the reports, setup-violating corners decrease from 8 to 6, and the worst
reported slack improves from -76.8905 ns to -28.2831 ns. The worst TNS improves
from -39,451.23 ns to -14,827.03 ns. This should not be attributed to routing
alone: the later report notes that the compared netlists differ, clock treatment
changes from ideal to propagated, and the parasitics are estimated. Treat this as
a comparison of the two recorded snapshots, not a controlled experiment isolating
the effect of routing.

The report's corner-by-corner comparison illustrates the trend:

| Corner | Synthesis setup slack | Post-route setup slack |
|---|---:|---:|
| `ss_100c_1v40` | -13.4015 ns | -3.4532 ns |
| `ss_100c_1v60` | -6.5384 ns | +1.2683 ns |
| `ss_n40c_1v28` | -76.8905 ns | -28.2831 ns |
| `ss_n40c_1v35` | -45.5748 ns | -16.0734 ns |
| `ss_n40c_1v40` | -33.2707 ns | -10.7110 ns |
| `ss_n40c_1v44` | -27.3124 ns | -7.6047 ns |
| `ss_n40c_1v60` | -11.1771 ns | -0.5273 ns |
| `ss_n40c_1v76` | -4.7641 ns | +2.7816 ns |
| `tt_025c_1v80` | +0.9581 ns | +6.1344 ns |

The synthesis report describes ideal-clock analysis; the later report includes
the propagated CTS clock and a different flow netlist with buffering and
fanout limiting. Later parasitics come from global-routing estimates rather
than detailed-route SPEF extraction. The improvement is a comparison between
snapshots, not an isolated measurement of routing benefit or sign-off on
extracted parasitics.

### How the corner reports were produced

For the synthesis-stage analysis, the report describes running `report_checks`
for min/max paths and collecting `report_worst_slack` and `report_tns` for each
corner. It used the same synthesized netlist and 11 ns SDC while swapping the
standard-cell Liberty corner. The PDF includes path-by-path detail.

For the later report, the described OpenROAD run loaded the routed database,
constraints, multiple standard-cell corner libraries, and the PLL/DAC macro
libraries, then reported setup and hold paths per corner. It says the same
routed database and parasitic estimate were used as the standard-cell corner
changed. Parasitics were estimated with:

```text
estimate_parasitics -global_routing
```

and the platform RC file. The run did not use SPEF from detailed-route
extraction. These values are useful implementation evidence, but are not
extracted-parasitic final sign-off results. Reproduction also requires the exact
netlists, SDC, macro timing models, Liberty files, scripts, and tool builds.

### Frequency and minimum-period interpretation

The post-route report calculates minimum period as `11 ns - worst setup slack`
and estimates `Fmax` as `1000 / minimum_period` when period is in nanoseconds
and frequency is in megahertz. At `ss_n40c_1v28`, the resulting 39.28 ns minimum
period corresponds to about 25.46 MHz. This derived value assumes the timing
model and constraints in the report; it is not a blanket chip-frequency
specification. Product-level frequency claims require timing to pass across the
specified PVT range with appropriate clock and IO constraints, macro models,
extracted parasitics, and a DRC-clean implementation.

The PVT trend is also important. Setup slack falls rapidly as voltage decreases
in the SS family, particularly below about 1.6 V. At `ss_n40c_1v60`, slack is
-0.5273 ns; at `ss_n40c_1v76`, it is +2.7816 ns. The low-voltage corner is a
material timing risk if operation in that range is required. Conversely, the
fastest-corner hold slack remains positive in the report, with the minimum
reported value +0.1998 ns.

## Recommended next steps

1. Resolve the 28 met1 short violations and rerun detailed-route DRC.
2. Confirm the intended operating voltage and temperature range, then target all
   required SS setup corners rather than only the typical corner.
3. Repair or restructure critical setup paths at the failing SS corners; rerun
   timing after each netlist/constraint change.
4. Use extracted detailed-route parasitics for final STA rather than the
   global-routing estimate used in this report.
5. Re-run the full multi-corner setup and hold checks on the final DRC-clean
   database and document tool versions, constraints, libraries, and outputs.
6. Archive the exact RTL/netlist, SDC, macro models, Liberty files, OpenROAD
   scripts, DRC output, and STA outputs used for the final run so the results
   can be reproduced.

## Related documents and theory

- [Physical-design online course](./PD_Online_Course.md)
- [RTL online course](./RTL_Online_Course.md)
- [Setup timing theory](../Week-3-Module4/theory_images/theory_02_setup_analysis_ideal_clock.png)
- [Real-clock setup theory](../Week-3-Module4/theory_images/theory_08_setup_analysis_real_clock.png)
- [Real-clock hold theory](../Week-3-Module4/theory_images/theory_09_hold_analysis_real_clock.png)
- [Workshop video and access notes](./video/README.md)
