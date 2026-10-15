# RTL Online Course — Verilog from Simulation to Synthesis

## Course scope

This course follows a small Verilog design from its behavioral description through
simulation, logic optimization, standard-cell mapping, and gate-level simulation.
The hands-on examples use a 2:1 multiplexer, with further examples covering
sequential logic, conditional coding, latch inference, and generated hardware.
SKY130 library and tool results are snapshots from the workshop environment; rerun
the flow with the local PDK and tool versions before treating them as reproducible
measurements.

![RTL-to-GDSII flow overview](../Week3/flow.png)

## Learning outcomes

After completing the labs, a learner should be able to:

- Write combinational and sequential RTL with clear, synthesizable semantics.
- Verify behavior with a testbench before relying on synthesis.
- Explain how Yosys and ABC optimize and map RTL into SKY130 standard cells.
- Compare RTL simulation with gate-level simulation.
- Identify accidental latches and simulation/synthesis mismatches.
- Distinguish procedural iteration from elaboration-time hardware generation.

## 1. Simulate a 2:1 multiplexer

The first example implements `y = sel ? i1 : i0`. When `sel` is low, `y` follows
`i0`; when it is high, `y` follows `i1`.

```verilog
module mux2to1 (
    input  i0,
    input  i1,
    input  sel,
    output y
);
    assign y = sel ? i1 : i0;
endmodule
```

The testbench is not hardware. It instantiates the design, applies inputs, and may
write a VCD trace for GTKWave. Exercise all four combinations of `sel` and the
selected input, and check that the output matches the truth table:

| `sel` | Selected input | Expected `y` |
|---:|---|---:|
| 0 | `i0` | `i0` |
| 1 | `i1` | `i1` |

Example simulation commands:

```bash
iverilog -o mux_sim.out mux2to1.v tb_mux2to1.v
vvp mux_sim.out
gtkwave tb_mux2to1.vcd
```

Simulation checks the behavior that was written, not the designer's intent. Fix
functional mistakes here before investigating the synthesized circuit.

### What each part of the testbench does

```verilog
`timescale 1ns / 1ps
module tb_mux2to1;
    reg i0, i1, sel;
    wire y;

    mux2to1 uut (.i0(i0), .i1(i1), .sel(sel), .y(y));

    initial begin
        $dumpfile("tb_mux2to1.vcd");
        $dumpvars(0, tb_mux2to1);
        i0 = 0; i1 = 0; sel = 0;
        #50 i0 = 1;
        #50 sel = 1;
        #50 i1 = 1;
        #50 i0 = 0;
        #100 $finish;
    end
endmodule
```

The `timescale` sets the delay unit and precision. Testbench `reg` variables drive
inputs, and `wire` observes the continuously driven output. The `uut` instance is
the design under test. `$dumpfile` and `$dumpvars` create the VCD trace. The
example uses delays to make input changes visible; a stronger regression test
would explicitly check every truth-table row and call `$fatal` when the result
differs from the expected value. A waveform is useful for learning and debugging,
but an automated assertion is easier to rerun and catches failures without visual
inspection.

For a two-state mux, the four combinations of select and selected input determine
the output. The unselected input is a don't-care for that row:

| `sel` | `i0` | `i1` | Expected `y` |
|---:|---:|---:|---:|
| 0 | 0 | 0 or 1 | 0 |
| 0 | 1 | 0 or 1 | 1 |
| 1 | 0 or 1 | 0 | 0 |
| 1 | 0 or 1 | 1 | 1 |

In four-state simulation, `X` and `Z` values can reveal uninitialized state,
floating inputs, or ambiguous control. Do not treat a result containing `X` as a
valid logic 0 or 1; investigate its source.

### GTKWave inspection

Load the generated VCD, add `i0`, `i1`, `sel`, and `y` to the waveform pane, then
use Zoom Fit to see the complete run. Check the select transitions as well as data
transitions while select is stable. The latter check is important: a mux output
must respond when its selected data input changes even if `sel` does not.

## 2. Synthesize and map to SKY130 cells

Synthesis transforms the RTL into a logic network. Technology mapping selects
concrete cells from a library. A Liberty (`.lib`) file describes cell functions and
their timing, power, and area; a physical LEF view is separate and describes cell
geometry and pins.

Typical Yosys sequence:

```tcl
read_liberty -lib ./sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog mux2to1.v
synth -top mux2to1
abc -liberty ./sky130_fd_sc_hd__tt_025C_1v80.lib
show mux2to1
write_verilog -noattr mux2to1_netlist.v
```

`read_liberty -lib` imports cell declarations for synthesis, `read_verilog` loads
the RTL, and `synth -top` elaborates the selected top module and performs generic
logic synthesis. ABC then maps the optimized combinational network to cells in the
specified Liberty library. `show` opens a logic graph; `write_verilog -noattr`
exports a structural netlist without internal synthesis attributes.

The mux may map to a dedicated `sky130_fd_sc_hd__mux2_1` or an equivalent network
of other library cells. Mapping depends on the library, constraints, and tool
version; functional equivalence is the requirement. More generally, cell choice
balances function, timing, area, and drive strength. A stronger cell may reduce
delay on a heavily loaded or critical net but costs more area and may increase
power. Cell statistics and the mapped netlist show what the tool actually chose.

When mapping sequential designs, the DFF mapping stage must also be considered.
Yosys flows commonly use `dfflibmap -liberty <library>` for sequential cells before
or as part of technology mapping. The exact order is flow-dependent; OpenLane
configures and sequences its own synthesis passes.

Re-simulate the mapped netlist using compatible SKY130 Verilog models and the same
testbench:

```bash
iverilog -o netlist_sim.out \
  ./sky130_lib_models/primitives.v \
  ./sky130_lib_models/sky130_fd_sc_hd.v \
  mux2to1_netlist.v tb_mux2to1.v
vvp netlist_sim.out
```

The model files must define every mapped cell type referenced by the netlist.
Compilation errors about unknown modules generally mean the correct cell-model
source was not included or the library version does not match the netlist. When
the mapped netlist contains timing primitives or specify blocks, the simulator
options and model variant can affect whether the run is functional-only or
timing-aware.

### Liberty, LEF, and Verilog models are different views

| View | Main role | Information it provides |
|---|---|---|
| Liberty (`.lib`) | Synthesis and STA | Logical function, timing arcs, power, area, drive variants |
| LEF | Floorplanning, placement, routing | Cell outline, pin geometry, routing obstructions |
| Verilog cell model | RTL/GLS simulator | Behavioral definition of a standard-cell instance |

Using a LEF file does not by itself provide delay models. Likewise, a Liberty
description is not a physical layout. A design flow needs mutually consistent
logical, timing, simulation, and physical views.

## 3. Logic optimization

Yosys can simplify logic before mapping. Examples from the workshop include:

| RTL expression | Simplified behavior |
|---|---|
| `a ? b : 0` | `a & b` |
| `a ? 0 : b` | `~a & b` |
| `a ? (c ? b : 0) : 0` | `a & b & c` |

Constant propagation can remove gates, and sequential optimization may remove
registers or bits whose values cannot affect any observable output. Always inspect
the resulting netlist or cell statistics to verify what the tool actually removed.
For example:

```tcl
read_verilog opt_check1.v
synth -top opt_check1
opt_clean -purge
abc -liberty ./sky130_fd_sc_hd__tt_025C_1v80.lib
show opt_check1
```

Optimization must preserve the behavior at the module's outputs. A register is
not constant merely because it has a reset pin; its value must be provably fixed.

### Sequential optimization and observability

The workshop's sequential examples demonstrate two useful questions:

1. Is a state bit provably fixed for all reachable operation? If so, synthesis
   may replace it with a constant and simplify logic that uses it.
2. Can a state bit affect an output or any other observable state? If not, it may
   be removed as unobservable logic.

For example, if only bit 0 of a counter reaches a module output, upper bits may be
unnecessary in the synthesized implementation. Whether a specific pass removes
them depends on the elaborated design and optimization settings. Inspect both the
cell count and synthesized netlist rather than assuming pruning occurred.

Constant folding and cleanup are related but different: constant folding proves
values and simplifies functions, while cleanup removes now-unreferenced cells
and wires. `opt_clean -purge` is useful after optimization when checking that
dangling logic is removed. It is not a substitute for choosing the correct top
module, library, or constraints.

### A reliable synthesis review

After mapping, check:

- The expected top module and input/output ports are present.
- No unintended latches, black boxes, or unmapped generic logic remain.
- The cell histogram is plausible for the design and target library.
- The cell functions and connections implement the intended behavior.
- Area and timing reports are interpreted with their units and constraints.
- RTL and gate-level simulations agree for representative inputs and state
  transitions.

## 4. Gate-level simulation and assignment semantics

Gate-level simulation (GLS) runs the synthesized netlist under a testbench. It
provides a useful cross-check that synthesis retained the intended behavior.

Use these coding rules to avoid common simulation/synthesis mismatches:

| Logic | Recommended assignment | Reason |
|---|---|---|
| Combinational process | Blocking `=` | Recomputes outputs from current inputs |
| Clocked sequential process | Non-blocking `<=` | Models simultaneous register updates |

For combinational processes, use a complete sensitivity list such as
`always @(*)`. An incomplete list can leave a simulation output stale when an
omitted input changes. In a clocked process, blocking assignments can make a
later statement observe a just-updated register in simulation, unlike the
simultaneous state update of physical flip-flops.

For a two-stage pipeline:

```verilog
always @(posedge clk) begin
    q0 <= d;
    q  <= q0;
end
```

At each rising edge, both right-hand sides are evaluated from the pre-edge state:
`q0` captures the old `d`, and `q` captures the old `q0`. Replacing `<=` with
blocking assignments makes simulation order-dependent; the second statement can
see the newly assigned `q0`. Use non-blocking assignments for clocked state even
when the simulator appears to produce a plausible result.

Likewise, `always @(sel)` is not a correct sensitivity list for a mux that also
reads `i0` and `i1`. If either data input changes while `sel` is stable, the
simulator does not reevaluate the process. Use `always @(*)` for combinational
logic, or use a continuous assignment as in the mux example. This sensitivity
problem is a simulation issue; synthesis infers logic from the expressions and
does not interpret the omitted event trigger as a physical circuit element.

### GLS comparison checklist

1. Simulate the original RTL and retain its VCD or assertion results.
2. Synthesize and map the same top module with the intended library.
3. Compile the gate netlist together with the matching cell simulation models.
4. Reuse the same testbench and compare outputs at matching input/clock events.
5. If a mismatch appears, inspect reset/initialization assumptions, unknown
   values, edge ordering, incomplete sensitivity lists, and assignment style.

Functional GLS checks behavior without necessarily modeling cell delay. Timing
GLS uses delay-aware models or timing annotation and can reveal timing-dependent
effects; it is not a replacement for static timing analysis.

## 5. Conditions, latches, and generated hardware

An `if` / `else if` chain gives priority to earlier conditions. A `case` statement
expresses selection among alternatives. In a combinational process, every output
must be assigned on every path. Otherwise the process must retain its old value,
which infers a latch.

```verilog
always @(*) begin
    y = 1'b0;
    case (sel)
        2'b00: y = a;
        2'b01: y = b;
        default: y = 1'b0;
    endcase
end
```

A `default` branch alone is not enough if a branch assigns only some of several
outputs; provide defaults for every output at the start of the process. In a
clocked process, intentionally retaining a register value when enable is low is
normal sequential behavior, not a latch.

A procedural `for` loop inside an `always` block describes repeated evaluation.
A `generate for` loop elaborates repeated hardware instances, such as one full
adder per bit in a ripple-carry adder. Loop bounds and carry connectivity should
be checked for the intended hardware width.

### Conditional coding: priority, selection, and storage

In an `if` / `else if` chain, an earlier true condition takes precedence. This is
appropriate for priority logic such as interrupt selection. A `case` statement is
often a clearer expression of mutually exclusive selection, such as a decoder or
multiplexer. These descriptions may map to different logic structures; compare
the mapped circuit if area or timing matters.

For combinational logic, assign every output for every path. A default assignment
at the start of the process makes the no-latch intent explicit:

```verilog
always @(*) begin
    x = 1'b0;
    y = 1'b0;
    case (sel)
        2'b00: begin x = a; y = b; end
        2'b01: begin x = c; end
        default: begin x = d; y = b; end
    endcase
end
```

Here `y` has a defined default when `sel == 2'b01`; without it, `y` retains its
previous value on that branch and a latch is inferred even though the `case`
includes a `default`. Conversely, a clocked register may intentionally hold its
state when enable is low:

```verilog
always @(posedge clk or posedge reset) begin
    if (reset)
        count <= '0;
    else if (enable)
        count <= count + 1'b1;
end
```

That hold behavior is a flip-flop enable/state behavior, not an accidental
combinational latch.

### Procedural loops and generated structures

A procedural loop expresses a repeated computation in a procedural block and is
typically unrolled by synthesis when bounds are static. A `generate for` loop
replicates module instances or continuous assignments at elaboration time. For an
N-bit ripple-carry adder, each full-adder instance receives one bit from each
operand and the previous stage's carry; the final carry is `carry[N]`. The
generate index and vector ranges must be constant and correctly sized.

For a demultiplexer, a `case` version and a generated set of decode equations can
express the same selection behavior. Simulation can verify equivalence, while
synthesis reveals whether the resulting gate structures differ in the chosen
library.

## Common learning checks

| Symptom | Likely cause to inspect |
|---|---|
| Output does not react to a data change in RTL simulation | Incomplete combinational sensitivity list |
| A combinational output remembers an earlier value | Missing assignment on an `if` or `case` path; inferred latch |
| A pipeline appears to skip a register in RTL simulation | Blocking assignment used for clocked state |
| `show` or mapping reports are ambiguous | Multiple modules loaded; select the intended top explicitly |
| GLS cannot compile | Missing/mismatched standard-cell Verilog models |
| Synthesis did not remove expected state | State may still be observable or not provably constant |
| RTL and GLS disagree | Check initialization, X propagation, blocking/non-blocking style, and model consistency |

## Complete workflow

```text
Write RTL
   → simulate with a testbench
   → synthesize and optimize
   → map with ABC and the target Liberty library
   → inspect reports/netlist
   → run gate-level simulation
```

## Related workshop material

- [Mux simulation lab](../lab1_mux_simulation.md)
- [Yosys and SKY130 synthesis lab](../lab2_yosys_synthesis.md)
- [Logic optimization lab](../lab3_optimization.md)
- [Gate-level simulation and assignment semantics lab](../lab4_gls_blocking_nonblocking.md)
- [Conditionals, latches, and generate lab](../lab5_conditionals_and_generate.md)
- [Physical-design course](./PD_Online_Course.md)
- [VSD BabySoC report](./VSDBabySoC.md)
- [Workshop video and sharing notes](./video/README.md)
