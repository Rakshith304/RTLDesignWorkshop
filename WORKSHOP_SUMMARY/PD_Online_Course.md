# Physical Design Online Course — RTL to GDSII

## Course scope

This course follows the physical-design stages recorded in the workshop: synthesis
of `picorv32a`, floorplanning and power delivery, placement, custom-cell
integration, clock-tree synthesis (CTS), routing concepts, and timing verification.
The implementation examples use SKY130, OpenLane/OpenROAD, and supporting open-source
EDA tools.

![OpenLane flow overview](../Week3/flow.png)

## RTL-to-GDSII sequence

```text
RTL → synthesis → floorplan and I/O placement → PDN and tap cells
    → placement and optimization → CTS → global and detailed routing
    → DRC/connectivity checks → static timing analysis → GDSII
```

Each stage produces inputs for the next one. DEF records physical design
information such as die/core geometry and placement; LEF describes technology and
cell abstracts; Liberty supplies logical and timing models; GDSII stores physical
layout geometry for downstream manufacturing data preparation.

### What physical design adds after synthesis

Synthesis determines which logical cells implement the RTL, but not where the
cells sit, how power reaches them, how the clock is distributed, or which metal
tracks connect them. Physical design assigns locations and routes, then
re-evaluates delay with physical effects included. Congestion, wire capacitance,
fanout, cell loading, clock insertion delay, and coupling can change the timing
picture from a synthesis-stage estimate.

| Stage | Typical question | Representative artifacts |
|---|---|---|
| Synthesis | What cells implement the design? | Mapped Verilog, cell statistics |
| Floorplanning | How large are die/core, and where do macros and pins go? | Floorplan DEF, macro placement |
| PDN and tap insertion | How are supply and substrate connections provided? | PDN shapes, tap/endcap instances |
| Placement | Where do standard cells sit, and is placement legal? | Placement DEF, utilization and timing reports |
| CTS | How is the clock delivered to sequential cells? | Clock buffers, propagated-clock reports |
| Routing | Which wires and vias connect each net? | Route guides, routed DEF/database |
| Sign-off checks | Are geometry, connectivity, and timing acceptable? | DRC, antenna, connectivity, STA reports |

## 1. Synthesis baseline

The Week 3 module synthesizes the `picorv32a` design with OpenLane and the
`sky130_fd_sc_hd` library. The recorded snapshot reports 14,876 standard cells,
including 1,613 `dfxtp_2` flip-flops, with reported library area 147,712.92 units.
The post-synthesis OpenSTA baseline is WNS = -24.89 ns and TNS = -759.46 ns.
Negative slack at this point is a baseline timing violation, not a completed
physical-design result.

The Week 3 baseline also reports 14,596 wires, 14,978 wire bits, 14,876 cells,
and 1,613 `dfxtp_2` flip-flops. The reported area is in library units; do not
treat it as square microns without checking the library unit and report
definition. WNS identifies the worst single path, while TNS aggregates setup
violations across paths. Neither metric alone identifies a root cause. A detailed
path report is needed to determine whether delay is dominated by logic depth,
fanout, cell load, or interconnect.

## 2. Floorplanning, PDN, and placement

Floorplanning establishes the die, core, rows, and utilization target. In the
documented OpenLane build, `run_floorplan` also performed I/O placement, tap/decap
insertion, and PDN generation. The power-distribution network uses rings, straps,
and standard-cell rails to deliver supply to the design.

The recorded `picorv32a` run used 35% target core utilization. Its DEF reports a
die of approximately 660.805 µm × 671.405 µm (0.44367 mm²). The run inserted 476
endcaps and 5,878 tap cells. Placement recorded 21,699 instances and about 36%
utilization (55% including padding).

| Stage | WNS | TNS |
|---|---:|---:|
| Post-synthesis | -24.89 ns | -759.46 ns |
| Post-placement | -16.57 ns | -388.08 ns |

Placement optimization improved these reported values, but did not close setup
timing. The values belong to the documented run and its constraints.

### Utilization, die area, and placement

Core utilization is standard-cell area divided by available core area. The target
is a planning constraint, not a guarantee that the final design will have exactly
that density. Whitespace is needed for routing, clock cells, buffers, power
structures, and optimization. Excessively high utilization can increase
congestion and detours; a low target generally requires a larger core. Placement
padding can make effective utilization differ from the bare cell-area percentage.

The recorded die area is:

```text
660.805 um × 671.405 um = 443,667.78 um² ≈ 0.44367 mm²
```

The 35% core-utilization target, approximately 36% placement utilization, and
55% including padding describe different density measures. Instance totals grow
between synthesis and placement because physical implementation adds taps,
endcaps, buffers, and related cells.

### PDN, tap cells, and decoupling

The power-distribution network distributes supply from upper-level rings and
straps through lower-metal rails to standard cells. The documented run uses met4
and met5 for straps and met1 for rails. Endcaps are inserted at row boundaries;
tap cells are placed periodically to connect well/substrate structures to the
appropriate supply rails and support latch-up prevention. Decoupling cells
provide local capacitance to reduce supply variation during switching. These
structures have different roles: taps provide substrate/well ties, decaps provide
capacitance, and PDN wires distribute current.

## 3. Custom cell and physical abstracts

The custom-inverter exercise connects a transistor-level layout to the
Place-and-Route representation:

![Custom physical layout and abstract cell view](../Week3-Module3/Module3/theory_images/theory_layout_vs_abstract.png)

![Labeled physical layout layers](../Week3-Module3/Module3/theory_images/theory_layout_layers_labeled.png)

Magic is used for layout, extraction, and custom-cell work. The exercise extracts
the layout to SPICE, simulates and characterizes an inverter, and generates a LEF
abstract. Correct pin names, pin class/use, routing-grid alignment, cell geometry,
and unique macro naming are needed for integration. A LEF abstract by itself does
not supply the timing model required for timing-aware synthesis and STA; that role
belongs to Liberty.

The recorded inverter was 1.380 µm × 2.720 µm. The module flags an unresolved
`Transistor width < 0.42um` DRC warning and an unintended extra `sky130_inv.lef`
in one merge. Those caveats should be resolved before treating the cell as clean
sign-off collateral.

### From extracted layout to a characterized cell

The custom-inverter exercise follows a practical sequence:

1. Inspect the Magic layout and verify transistor, supply, and output
   connectivity.
2. Run extraction to produce an `.ext` representation of geometry and
   parasitics.
3. Convert the extraction to SPICE and include the intended transistor models.
4. Simulate a transient response and measure propagation delay and transition
   times.
5. Produce a LEF abstract with legal dimensions and pin metadata.
6. Merge the LEF into the implementation flow and verify its macro name and pins.

The source notes describe three SPICE-deck corrections: use model names that
exist in the available libraries, reference the input pulse to ground, and set
the device scale to match Magic's extraction grid. Reported measurements are
about 36.6 ps rise propagation delay, 36.0 ps rise time, and 22.7 ps fall time.
The reported fall-delay value of about 1.76 ps is flagged as potentially affected
by startup ringing; it should not be treated as robust characterization without
repeating the measurement under a suitable stimulus and load.

A LEF present in a merged file does not establish a trustworthy timing model. A
valid Liberty characterization and consistent logical function are required
before a custom cell can safely participate in timing-driven synthesis and STA.
The module also records a DRC warning and an unintended extra LEF in one merge;
both are cleanup items, not sign-off passes.

## 4. Setup, hold, and critical-path optimization

Setup checks that data arrives before its capture edge. Hold checks that data does
not arrive too soon after that edge. Positive slack meets the relevant constraint;
negative slack is a violation. WNS is the worst path slack and TNS aggregates the
negative slack over violating paths.

![Setup analysis with ideal clocks](../Week-3-Module4/theory_images/theory_02_setup_analysis_ideal_clock.png)

![Setup analysis with real clocks](../Week-3-Module4/theory_images/theory_08_setup_analysis_real_clock.png)

![Hold analysis with real clocks](../Week-3-Module4/theory_images/theory_09_hold_analysis_real_clock.png)

The Module 4 investigation traced a `picorv32a` critical path through roughly 65
serial OR gates. Cell upsizing improved the reported WNS from -22.77 ns to
-12.626 ns, but did not close timing. The changing critical endpoint illustrates
why timing work is iterative and why reducing logic depth can be more effective
than indefinitely increasing cell drive strength.

## 5. Clock-tree synthesis

CTS inserts buffers to distribute the clock to sequential cells while controlling
latency, transition, and skew. Skew is the difference in clock arrival time at
different endpoints; buffer and wire delays both contribute.

![Clock-tree buffering and RC model](../Week-3-Module4/theory_images/theory_03_cts_buffering_rc_model.png)

![Power-aware CTS delay tables](../Week-3-Module4/theory_images/theory_01_power_aware_cts_delay_tables.jpeg)

![Clock skew and buffer tree](../Week-3-Module4/theory_images/theory_10_skew_formula_buffer_tree.png)

In the recorded `picorv32a` Module 4 run, CTS completed and reported 3.39 ns skew
between the shown clock pins. Post-CTS `run_sta` reported WNS = -13.64 ns and
TNS = -269.12 ns. Clock shielding, crosstalk, and glitching are important theory
topics, but explicit clock-net shielding was not recorded as an implemented step:

![Crosstalk-induced clock skew](../Week-3-Module4/theory_images/theory_06_crosstalk_delta_delay_skew.png)

![Clock-net shielding](../Week-3-Module4/theory_images/theory_04_clock_net_shielding.png)

### Why clock quality matters

Clock sinks should receive transitions with controlled arrival time and slew.
CTS inserts and sizes buffers to manage fanout and wire capacitance while
balancing branches. Insertion delay is the time from the clock source to a sink;
skew is the difference in arrival time between sinks:

```text
skew(A, B) = clock_arrival(B) - clock_arrival(A)
```

The sign and impact depend on which endpoint is the launch or capture register.
For setup, a later capture-clock arrival can provide useful time; for hold, that
same relationship can be harmful. Clock analysis must consider both max-delay
setup and min-delay hold, not only absolute skew.

Delay tables indexed by input slew and output capacitance let CTS estimate
buffer delay under different loads. Shielding may reduce coupling from adjacent
switching nets, but consumes routing resources and must follow the technology
and flow rules. A clock glitch can create an unintended state transition; it is
not solved merely by improving data-path slack.

## 6. Routing and post-route checks

Global routing estimates connectivity and generates guides; detailed routing
connects pins on legal tracks with wires and vias while obeying layer and geometric
rules. Lee's maze-routing algorithm illustrates grid-based path search. Modern
routers such as TritonRoute perform detailed routing using routing guides and
panel-based techniques.

### Routing concepts in more detail

Lee's algorithm treats the routing space as a grid. It expands a wavefront from
the source, assigning distances to reachable locations, then backtraces from the
target through decreasing labels. Breadth-first search finds a shortest grid path
if one exists, but a full-chip grid search can use substantial memory and runtime;
it is a conceptual/local search model, not the whole modern detailed-router
implementation.

Global routing works at coarse granularity. It estimates demand between regions
and produces routing guides. Before detailed routing, guides can be split, merged,
or bridged into usable connected regions. Detailed routing selects exact tracks
and vias while obeying blockages, pin access, and geometric rules. TritonRoute's
panel-based work divides routing regions into panels; work can be parallelized
within a layer while dependencies between layers still require ordering.

For a multi-terminal net, the router needs a topology connecting all terminals.
An access point is a legal point where a pin can connect to a routing layer. An
approximate minimum spanning tree (MST) over access points minimizes the sum of
selected connection costs while keeping the terminals connected without loops.
This is a topology choice, not the final detailed wire geometry.

| Check | What it protects |
|---|---|
| Minimum wire width | Conductor reliability and manufacturability |
| Minimum spacing | Isolation between neighboring nets |
| Routing pitch/alignment | Legal use of routing tracks |
| Via width and spacing | Legal vertical connections between metal layers |
| Pin access and guide connectivity | Ability to connect each terminal |
| Open-net/short check | Correct electrical connectivity |

![Maze routing concept](../Week3-Module5/theory_images/theory_01_routing_concept.png)

![Lee maze-routing algorithm](../Week3-Module5/theory_images/theory_02_lee_algorithm.png)

![Wire width, spacing, and pitch](../Week3-Module5/theory_images/theory_03_wire_design_rules.png)

![Vias between metal layers](../Week3-Module5/theory_images/theory_04_via_metal_layers.png)

![TritonRoute global-to-detailed flow](../Week3-Module5/theory_images/theory_05_tritonroute_flow.png)

![Power distribution network](../Week3-Module5/theory_images/theory_11_pdn_floorplan.png)

After routing, inspect connectivity, shorts, open nets, antenna checks, DRC, and
post-route timing. Routing completion does not imply sign-off: the detailed route
must pass physical checks, and the timing model must match the extraction and
constraints used for sign-off.

### Interpreting timing during physical design

For setup, arrival time includes data-cell and data-wire delay; required time
depends on the clock period, capture-clock arrival, setup time, and uncertainty.
A reported slack is meaningful only alongside its clock constraint, clock
propagation state, parasitic source, Liberty corner, and design revision. Check
these conditions before comparing checkpoints.

The `picorv32a` Module 4 baseline path contained approximately 65 serial OR
cells. Upsizing improved that run's WNS from -22.77 ns to -12.626 ns, but did not
make timing pass. A long chain points to logic depth as a root cause; repeatedly
increasing drive strength may consume area and power without removing the
structural limit. A subsequent CTS/STA run used a different execution checkpoint,
so values from those sequences must not be combined into a misleading monotonic
progression.

Hold is a separate minimum-delay constraint. Upsizing or shortening a data path
may reduce minimum delay and expose a hold issue. Hold buffers increase minimum
delay but can hurt setup. Recheck both setup and hold after optimization.

### Recorded OpenLane sequence

The documented interactive sequence is:

```tcl
package require openlane 0.9
prep -design picorv32a
run_synthesis
run_floorplan
run_placement
run_cts
run_sta
```

Commands and exact stage behavior depend on the OpenLane version and
configuration. The Week 3 Module 5 document is a routing theory walkthrough and
does not claim that the `picorv32a` route/sign-off run was completed. A successful
synthesis, placement, or CTS step is not evidence of a final GDS or DRC pass.

The `picorv32a` Module 5 documentation is a theory walkthrough and does not claim
that the route/sign-off run was completed. The VSD BabySoC report is a separate,
measured routed run; it still has 28 met1 short violations and therefore is not
DRC-clean. See the [BabySoC report](./VSDBabySoC.md) for its conditions and
multi-corner timing results.

## Further reading in this workspace

- [Week 3 synthesis module](../Week3/README.md)
- [Floorplanning, PDN, and placement module](../week3module2/Module2/README.md)
- [Custom inverter design module](../Week3-Module3/Module3/README.md)
- [Custom-cell integration and CTS module](../Week-3-Module4/README.md)
- [Routing theory module](../Week3-Module5/README.md)
- [RTL online course](./RTL_Online_Course.md)
- [VSD BabySoC STA report](./VSDBabySoC.md)
- [Workshop video](./video/README.md)
