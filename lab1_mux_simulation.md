# Lab 1 — Simulating a 2:1 Multiplexer with Icarus Verilog & GTKWave

## Goal

Before synthesizing any circuit, it's important to confirm the RTL actually behaves the way it's supposed to. This lab walks through writing a simple 2:1 mux in Verilog, simulating it, and inspecting the output waveform.

## Why Simulate First?

Synthesis tools will happily convert incorrect RTL into an incorrect circuit — they don't know what you *meant* to build, only what you *wrote*. Simulating against a testbench is how you catch logic mistakes early, before they get baked into a gate-level netlist.

## The Design (Unit Under Test)

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

When `sel = 0`, the output follows `i0`. When `sel = 1`, it follows `i1`.

## The Testbench

A testbench doesn't get synthesized — it exists purely to drive inputs into the design and record what comes out. A typical testbench:

1. Instantiates the design under test (DUT)
2. Declares registers to drive the inputs and wires to observe the outputs
3. Applies a sequence of input combinations with delays between them
4. Dumps signal activity to a `.vcd` file for later viewing
5. Ends the simulation with `$finish`

```verilog
`timescale 1ns / 1ps
module tb_mux2to1;

    reg i0, i1, sel;
    wire y;

    mux2to1 uut (
        .i0(i0),
        .i1(i1),
        .sel(sel),
        .y(y)
    );

    initial begin
        $dumpfile("tb_mux2to1.vcd");
        $dumpvars(0, tb_mux2to1);

        i0 = 0; i1 = 0; sel = 0;
        #50 i0 = 1;
        #50 sel = 1;
        #50 i1 = 1;
        #50 i0 = 0;
        #100;
        $finish;
    end

endmodule
```

## Running the Simulation

**1. Compile the design and testbench together:**
```bash
iverilog -o mux_sim.out mux2to1.v tb_mux2to1.v
```

**2. Execute the compiled simulation:**
```bash
vvp mux_sim.out
```
This produces `tb_mux2to1.vcd`.

**3. View the waveform in GTKWave:**
```bash
gtkwave tb_mux2to1.vcd
```


## Reading the Waveform

In GTKWave:
- Add `i0`, `i1`, `sel`, and `y` to the signal viewer (drag from the signal list into the waveform pane)
- Use **Zoom → Zoom Fit** to see the full simulation window at once
- Step through time and confirm: whenever `sel` toggles, `y` should switch between tracking `i0` and `i1`


## What This Confirms

If the waveform matches the expected truth table for a 2:1 mux across every input change, the RTL is functionally correct and ready to move on to synthesis (covered in Lab 2).

| sel | i0 | i1 | y (expected) |
|-----|----|----|----------------|
| 0   | 0  | X  | 0              |
| 0   | 1  | X  | 1              |
| 1   | X  | 0  | 0              |
| 1   | X  | 1  | 1              |

## Files in This Lab

| File | Purpose |
|------|---------|
| `mux2to1.v` | RTL design (unit under test) |
| `tb_mux2to1.v` | Testbench |
| `tb_mux2to1.vcd` | Generated waveform dump |
| `mux_sim.out` | Compiled simulation executable |

## Takeaways

- A testbench is not synthesizable hardware — it's a verification harness.
- `$dumpfile` / `$dumpvars` are what make a `.vcd` file possible; skip them and GTKWave has nothing to show.
- Functional simulation is a mandatory checkpoint before synthesis, not an optional nice-to-have.
