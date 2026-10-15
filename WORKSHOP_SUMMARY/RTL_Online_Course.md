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
write_verilog -noattr mux2to1_netlist.v
```

The mux may map to a dedicated `sky130_fd_sc_hd__mux2_1` or an equivalent network
of other library cells. Mapping can vary with the library, constraints, and tool
version; functional equivalence is the requirement. Re-simulate the mapped netlist
using the SKY130 Verilog models and the same testbench:

```bash
iverilog -o netlist_sim.out \
  ./sky130_lib_models/primitives.v \
  ./sky130_lib_models/sky130_fd_sc_hd.v \
  mux2to1_netlist.v tb_mux2to1.v
vvp netlist_sim.out
```

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
