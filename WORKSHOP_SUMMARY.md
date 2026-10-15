# Digital VLSI Workshop — Detailed Summary

## 1. Workshop at a Glance

This workshop is a hands-on digital VLSI learning journal. It follows designs from
behavioral Verilog through functional verification, logic synthesis, standard-cell
mapping, physical design, timing analysis, and the concepts required to complete an
RTL-to-GDSII flow. The physical-design exercises use the open-source SKY130 PDK and
OpenLane/OpenROAD toolchain, with `picorv32a` as the larger example design and a
custom CMOS inverter as the standard-cell exercise.

The repository combines:

- Small RTL labs that demonstrate simulation, synthesis, optimization, gate-level
  simulation, and synthesizable coding style.
- A sequential set of OpenLane workshop modules that document synthesis,
  floorplanning, power delivery, placement, custom-cell design/integration, clock
  tree synthesis, routing theory, and timing verification.
- Source and generated design artifacts, including Verilog netlists, LEF/DEF,
  Magic layout, SPICE, and timing/synthesis reports.
- Theory diagrams embedded below from the workshop’s three `theory_images`
  directories.

This Git repository is the workshop root. All links and embedded diagrams in this
summary are relative to this root so they open correctly in GitHub as well as in a
local Markdown preview.

## 2. Learning Objectives

By working through the workshop, a learner can:

1. Describe combinational hardware in Verilog and validate its behavior with a
   self-checking or waveform-based testbench.
2. Understand the difference between RTL simulation and synthesis, and map RTL to
   cells in the SKY130 standard-cell library using Yosys and ABC.
3. Inspect Liberty timing libraries, mapped netlists, cell statistics, and synthesis
   reports.
4. Recognize logic simplification, constant propagation, sequential optimization,
   and unused-state pruning.
5. Avoid simulation/synthesis mismatches by using complete combinational
   sensitivities and non-blocking assignments in clocked logic.
6. Understand how incomplete conditional assignments infer latches, and distinguish
   procedural loops from elaboration-time `generate` loops.
7. Follow the physical-design progression from floorplanning through power
   distribution, placement, CTS, routing, and signoff checks.
8. Create, extract, characterize, and physically integrate a custom CMOS standard
   cell, while understanding the separate roles of LEF and Liberty.
9. Read setup/hold timing reports, interpret slack, identify critical paths, and
   understand common timing-closure techniques.

## 3. End-to-End Design Flow

The workshop’s central flow is:

```text
RTL design
  → RTL simulation and functional verification
  → logic synthesis and Boolean optimization
  → technology mapping to SKY130 standard cells
  → gate-level simulation
  → floorplanning and I/O placement
  → power distribution network and tap/decap insertion
  → standard-cell placement and optimization
  → clock tree synthesis
  → global and detailed routing
  → DRC, connectivity, and static timing checks
  → final layout / GDSII
```

The stages answer different questions:

| Stage | Main question | Representative tools/artifacts |
|---|---|---|
| RTL simulation | Does the RTL implement the intended behavior? | Icarus Verilog, testbench, VCD, GTKWave |
| Synthesis | What logic implements the RTL? | Yosys, ABC, mapped Verilog |
| Technology mapping | Which cells from the target library are used? | Liberty `.lib`, SKY130 cells |
| Gate-level simulation | Does the mapped netlist retain the RTL behavior? | Icarus Verilog, cell simulation models |
| Floorplanning and placement | Where do the die, core, rows, and cells go? | OpenLane/OpenROAD, DEF |
| CTS | How is the clock distributed with controlled skew? | Clock buffers, clock-tree reports |
| Routing | How are signal and power nets physically connected? | TritonRoute, routing guides, DEF |
| Signoff analysis | Are geometry, connectivity, and timing acceptable? | Magic/DRC, OpenSTA, reports |

## 4. Repository Guide

### Core documentation and labs

| Path | Subject |
|---|---|
| [`README.md`](README.md) | Workshop journal overview and links to standalone labs |
| [`lab1_mux_simulation.md`](lab1_mux_simulation.md) | 2:1 mux RTL simulation using Icarus Verilog and GTKWave |
| [`lab2_yosys_synthesis.md`](lab2_yosys_synthesis.md) | Yosys synthesis and mapping to SKY130 cells |
| [`lab3_optimization.md`](lab3_optimization.md) | Combinational and sequential optimization |
| [`lab4_gls_blocking_nonblocking.md`](lab4_gls_blocking_nonblocking.md) | Gate-level simulation and assignment semantics |
| [`lab5_conditionals_and_generate.md`](lab5_conditionals_and_generate.md) | Conditional logic, latches, loops, and generated hardware |

### OpenLane workshop modules

| Path | Subject |
|---|---|
| [`Week3/README.md`](Week3/README.md) | Synthesis of `picorv32a` and post-synthesis timing |
| [`week3module2/Module2/README.md`](week3module2/Module2/README.md) | Floorplanning, PDN, tap/decap cells, placement, and timing changes |
| [`Week3-Module3/Module3/README.md`](Week3-Module3/Module3/README.md) | Custom inverter layout, extraction, SPICE simulation, characterization, and LEF |
| [`Week-3-Module4/README.md`](Week-3-Module4/README.md) | Custom-cell integration, timing investigation, upsizing, CTS, and related theory |
| [`Week3-Module5/README.md`](Week3-Module5/README.md) | Routing algorithms, TritonRoute concepts, DRC, and post-route STA theory |

`output_files/`, `synthesis_files/`, and `images/` directories within the modules
contain the recorded outputs and execution illustrations for the corresponding
exercises. Generated reports and physical-design files are useful snapshots of
individual runs; they are not substitutes for rerunning the flow with the required
PDK, tool versions, and configuration.

## 5. Standalone RTL and Synthesis Labs

### Lab 1 — 2:1 mux simulation

The mux implements `y = sel ? i1 : i0`. A testbench applies input changes, writes
a VCD waveform, and GTKWave is used to confirm that the output selects the expected
input. The testbench exercises the selected input and select signal over time; its
waveform is compared with the mux truth table. This establishes functional
simulation as an early check before synthesis and introduces the design-under-test,
stimulus, waveform dump, and waveform inspection roles of a testbench.

### Lab 2 — Yosys and SKY130 mapping

The mux RTL is read into Yosys, synthesized, mapped with ABC against a SKY130
Liberty library, and written as a gate-level Verilog netlist. The netlist can be
simulated with standard-cell Verilog models and the same testbench. Liberty describes
cell functions and their timing, power, area, and drive-strength variants.
The mapped mux may use a dedicated SKY130 mux cell or an equivalent gate network,
depending on the library and mapping result. A graphical view of the mapped
hierarchy can be inspected in Yosys, and the generated netlist is checked against
the original RTL behavior.

### Lab 3 — logic optimization

Examples show how constant inputs can reduce muxes to simpler Boolean functions,
how sequential constants can allow flip-flops and dependent logic to be removed,
and how unobserved state bits may be pruned. Examples include mux expressions
reduced to AND-style functions, flops proven constant, and a counter whose
unobservable upper bits may be eliminated. The lab emphasizes checking the
synthesized circuit rather than assuming an optimization happened, and using
cleanup passes such as `opt_clean -purge` to remove dangling logic.

### Lab 4 — gate-level simulation and assignment semantics

Gate-level simulation swaps the synthesized netlist under the existing testbench.
It can expose behavioral differences hidden by RTL simulation. In particular:

- Use complete combinational sensitivity (`always @(*)`) so every read input can
  trigger reevaluation.
- Use blocking assignments (`=`) for combinational procedural logic.
- Use non-blocking assignments (`<=`) for clocked sequential logic so registers
  update together from their pre-edge values.

The lab demonstrates how incomplete sensitivity lists and blocking assignments in
sequential logic can produce simulation/synthesis mismatches. It compares the
original RTL simulation against gate-level simulation of the mapped design using
the standard-cell Verilog models. The same testbench can be used for both, making
the design implementation the meaningful difference between the two runs.

### Lab 5 — conditionals, latches, and generated structures

An `if`/`else if` chain expresses priority; `case` expresses selection among
alternatives. Every output in a combinational process needs an assignment on every
possible path. Missing assignments imply state retention and may infer a latch.
The lab also distinguishes procedural `for` loops, which describe repeated
evaluation, from `generate for` loops, which replicate hardware at elaboration time,
and applies generated instances to structures such as a ripple-carry adder. It
compares priority behavior in `if`/`else if` chains with selection in `case`
statements, highlights incomplete branches (including unassigned individual
outputs), and demonstrates checking the inferred hardware in Yosys.

## 6. OpenLane Physical-Design Modules

### Module 1 — RTL synthesis of `picorv32a`

The synthesis module runs OpenLane preparation and synthesis, producing a mapped
SKY130 netlist and OpenSTA reports. The recorded synthesis snapshot lists 14,876
cells, including 1,613 `dfxtp_2` flip-flops, and a reported area of 147,712.92
library units. The post-synthesis timing baseline is **WNS = -24.89 ns** and
**TNS = -759.46 ns**.

The module explains that negative slack at this point is a baseline rather than a
completed timing-closure result: placement, clock-tree construction, and routed
parasitics have not yet been incorporated.

### Module 2 — floorplanning, PDN, and placement

This module continues the `picorv32a` flow. `run_floorplan` establishes the die/core
and rows and, in the documented OpenLane version, also performs I/O placement,
tap/decap insertion, and PDN generation. `run_placement` places and legalizes cells,
then applies resizer-driven optimization.

Recorded design observations include:

- Core utilization target: 35%.
- Die dimensions: approximately 660.805 µm × 671.405 µm, or 0.44367 mm².
- Placement snapshot: 21,699 total instances and approximately 36% utilization
  (55% with padding).
- Timing improved from post-synthesis WNS/TNS of -24.89/-759.46 ns to the
  post-placement values of -16.57/-388.08 ns in the recorded run.

The floorplan DEF records die dimensions and physical structures, while the
placement DEF records cell coordinates. The documented run inserted 476 endcaps
and 5,878 tap cells; its resizer inserted buffers and changed cell drive strengths.
The module compares the floorplan and placement views with timing reports to show
that physical optimization can improve, but does not automatically close, timing.

### Module 3 — custom inverter design and characterization

The standard-cell exercise begins with a CMOS inverter layout inspected in Magic.
The layout contains the transistor geometry and routing details, while the placer
and router use an abstract LEF view with cell boundaries, pins, and obstructions.
These are two views of the same cell:

![Physical layout and abstract cell view](Week3-Module3/Module3/theory_images/theory_layout_vs_abstract.png)

The labeled layers connect that abstract representation back to the physical
structures in the layout:

![Labeled physical layout layers](Week3-Module3/Module3/theory_images/theory_layout_layers_labeled.png)

The CMOS fabrication diagrams explain what the well, gate, and device layers
represent in the finished inverter. The process forms the N-well and P-well,
patterns the gate, and then builds the complete transistor structure:

![N-well and P-well formation](Week3-Module3/Module3/theory_images/mask_nwell_pwell_formation.png)

![CMOS gate formation](Week3-Module3/Module3/theory_images/mask_gate_formation.png)

![Final fabricated transistor structure](Week3-Module3/Module3/theory_images/mask_final_fabricated_transistor.png)

After checking transistor and power/output connectivity, the exercise extracts
layout geometry and parasitics to an `.ext` file and converts it to a SPICE deck.
The threshold voltage depends in part on source-to-body voltage, which is why
correct well/body connections matter to the extracted transistor model:

![Threshold voltage and body effect](Week3-Module3/Module3/theory_images/theory_threshold_voltage.png)

The SPICE deck combines model declarations, circuit instances, sources, and
simulation commands. The workshop corrects model naming, pulse-source reference,
and extraction scale issues before running ngspice:

![SPICE deck structure](Week3-Module3/Module3/theory_images/theory_spice_deck.png)

The inverter is characterized using transient simulation and measured propagation
delay and rise/fall times. Its voltage-transfer characteristic (VTC) describes the
static relationship between input and output voltage; the switching threshold is
the point where the two voltages meet:

![Inverter voltage transfer characteristic and switching threshold](Week3-Module3/Module3/theory_images/theory_vtc_switching_threshold.png)

Finally, the module generates a LEF view and integrates the physical abstract into
OpenLane, confirming that the cell is present in the merged LEF.

The module reports approximately 36.6 ps rise propagation delay, 36.0 ps rise time,
and 22.7 ps fall time. The approximately 1.76 ps fall-delay value is explicitly
flagged in the source notes as potentially affected by startup ringing. A LEF
provides physical geometry and pin information; synthesis also needs a Liberty
timing model to use a custom cell for logic mapping.

### Module 4 — custom-cell integration, timing, and CTS

The OpenLane flow diagram places synthesis, floorplanning, clock distribution, and
routing in the larger implementation sequence:

![OpenLane implementation flow](Week-3-Module4/theory_images/theory_07_openlane_flow_diagram.png)

This module focuses on physical-design integration requirements: routing-grid
alignment, valid pin `class`/`use`, cell dimensions, unique naming, LEF merge, and
OpenLane configuration. It then analyzes the synthesized netlist’s critical setup
path. Setup timing requires data to arrive before the capture edge. An ideal-clock
analysis introduces the basic timing relationship before accounting for real clock
insertion delays:

![Setup timing analysis with ideal clock](Week-3-Module4/theory_images/theory_02_setup_analysis_ideal_clock.png)

In an implemented design, the launch and capture clocks arrive at different times.
Real-clock setup analysis accounts for those delays and setup uncertainty:

![Setup timing analysis with real clock](Week-3-Module4/theory_images/theory_08_setup_analysis_real_clock.png)

Hold analysis checks the opposite side of the capture edge: data must not change
too soon after that edge. Setup and hold are separate requirements, and fixing one
does not guarantee the other passes:

![Hold timing analysis with real clock](Week-3-Module4/theory_images/theory_09_hold_analysis_real_clock.png)

The recorded critical path contains a long serial OR-gate chain. Manual upsizing
improves WNS from -22.77 ns to -12.626 ns, but does not fully close timing. The
notes identify logic-depth restructuring as a more fundamental remedy than
indefinite per-cell upsizing.

Clock tree synthesis (CTS) distributes the clock to sequential cells. Buffering
balances the load and wire RC delay across branches; power-aware delay tables help
estimate the behavior of clock cells under their characterized conditions:

![Clock-tree buffering and RC model](Week-3-Module4/theory_images/theory_03_cts_buffering_rc_model.png)

![Power-aware clock-tree synthesis delay tables](Week-3-Module4/theory_images/theory_01_power_aware_cts_delay_tables.jpeg)

Clock arrival differences create skew. The skew equation and buffer-tree diagram
show how branch delays contribute to the difference between launch and capture
clock arrival:

![Clock skew equation and buffer tree](Week-3-Module4/theory_images/theory_10_skew_formula_buffer_tree.png)

Clock quality also depends on signal integrity and waveform correctness. Coupling
from neighboring nets can create crosstalk-induced delta delay and alter skew;
shielding reduces coupling to sensitive clock wires. A malformed or glitching
clock can cause unintended state capture:

![Crosstalk, delta delay, and clock skew](Week-3-Module4/theory_images/theory_06_crosstalk_delta_delay_skew.png)

![Clock-net shielding](Week-3-Module4/theory_images/theory_04_clock_net_shielding.png)

![Clock glitch problem](Week-3-Module4/theory_images/theory_05_clock_glitch_problem.png)

Known caveats recorded by the module include a custom-cell DRC warning
(`Transistor width < 0.42um`) and an unintended extra `sky130_inv.lef` included in
the merge alongside the intended custom cell. They did not prevent the documented
flow from proceeding, but should be reviewed before treating the cell integration
as clean signoff.

### Module 5 — routing and post-route analysis

Routing connects placed pins while navigating obstacles and obeying layer and
geometric rules. The basic routing problem is to find a legal path from source to
target; Lee’s algorithm explores a grid breadth-first and backtraces a shortest
path if one exists:

![Routing source-to-target concept](Week3-Module5/theory_images/theory_01_routing_concept.png)

![Lee maze-routing algorithm](Week3-Module5/theory_images/theory_02_lee_algorithm.png)

Physical routes must obey minimum wire width and spacing as well as the routing
grid pitch. Vias connect conductors on different metal layers and have their own
geometry and spacing rules. Violating these constraints can cause shorts:

![Wire width, spacing, and pitch rules](Week3-Module5/theory_images/theory_03_wire_design_rules.png)

![Via connecting metal layers](Week3-Module5/theory_images/theory_04_via_metal_layers.png)

![Via width and spacing rules related to signal shorts](Week3-Module5/theory_images/theory_07_via_width_spacing_short.png)

Routing is performed around fixed physical structures, including the power
distribution network (PDN), which supplies power through rings, straps, and cell
rails:

![Power-distribution network in the floorplan](Week3-Module5/theory_images/theory_11_pdn_floorplan.png)

The documented flow separates coarse global routing from detailed routing.
Global routing produces guides; TritonRoute uses processed guides to create
track-level wires and vias:

![TritonRoute global-to-detailed routing flow](Week3-Module5/theory_images/theory_05_tritonroute_flow.png)

Before detailed routing, guides are split, merged, or bridged so they form usable
continuous routing regions. TritonRoute then routes within panels, handling
same-layer work in parallel where possible and resolving dependencies between
layers in sequence:

![Route-guide splitting, merging, and bridging](Week3-Module5/theory_images/theory_08_preprocessed_route_guides.png)

![Panel routing across metal layers](Week3-Module5/theory_images/theory_09_panel_routing_flow.png)

For each net, the router identifies legal access points on pins and neighboring
layers. It then builds a topology connecting those points; a minimum spanning tree
is one way to minimize the total connection distance without leaving terminals
disconnected:

![Routing access points for pins and adjacent layers](Week3-Module5/theory_images/theory_10_access_points.png)

![Minimum-spanning-tree routing topology](Week3-Module5/theory_images/theory_06_mst_topology.png)

After routing, connectivity, design-rule violations, shorts, and timing must be
checked. The full-layout illustration shows the physical result these routing and
verification steps aim to produce:

![Full routed layout](Week-3-Module4/theory_images/theory_13_route_full_layout.png)

The module README currently describes this as theory prepared ahead of execution:
the route, DRC, and post-route timing outputs have not yet been recorded. The
diagrams above are part of that theory material, not evidence of a completed
route/signoff run. The workshop also includes two Lee-algorithm diagrams in Module 4
which provide additional maze-routing illustrations:

![Lee maze-routing algorithm example 1](Week-3-Module4/theory_images/theory_11_maze_routing_lee_algorithm_1.png)

![Lee maze-routing algorithm example 2](Week-3-Module4/theory_images/theory_12_maze_routing_lee_algorithm_2.png)

## 7. Main Tools and File Formats

| Tool / format | Role in the workshop |
|---|---|
| Verilog | RTL and testbench descriptions |
| Icarus Verilog (`iverilog`, `vvp`) | RTL and gate-level simulation |
| GTKWave | Viewing VCD simulation traces |
| Yosys | RTL elaboration, synthesis, and optimization |
| ABC | Boolean optimization and standard-cell technology mapping |
| SKY130 PDK | Open-source process and standard-cell data |
| Liberty (`.lib`) | Cell logic, timing, power, and area models |
| OpenLane / OpenROAD | Flow orchestration and physical implementation |
| OpenSTA | Static setup/hold timing analysis |
| Magic | Layout viewing, extraction, and custom-cell work |
| ngspice | SPICE simulation and electrical characterization |
| TritonRoute | Detailed routing in the documented flow |
| LEF | Abstract cell geometry, pins, and routing obstructions |
| DEF | Design floorplan, placement, and physical connectivity data |
| SPICE / EXT | Transistor-level circuit description and extracted parasitics |
| GDSII | Final mask-layout interchange format targeted by the flow |

## 8. Important Interpretation Notes

- Synthesis preserves the behavior described by RTL; it does not determine whether
  that behavior matches the designer’s intent. Verify the RTL first.
- A mapped netlist should be checked, preferably with gate-level simulation and
  appropriate cell models.
- LEF and Liberty serve different purposes: LEF provides physical abstracts;
  Liberty provides logical and timing characterization used by synthesis and STA.
- Setup and hold are distinct constraints. Setup checks late data arrival; hold
  checks data arriving too soon after a capture edge.
- WNS is the worst single path slack; TNS aggregates negative slack across violating
  paths. Negative results identify work still required, not a passing timing result.
- Physical snapshots and timing values are specific to their run, tool configuration,
  PDK, and constraints.
- The repository documents partial progress: synthesis, floorplanning, placement,
  and custom-cell experiments are recorded, while the final routing/signoff stage
  remains theoretical in Module 5.
