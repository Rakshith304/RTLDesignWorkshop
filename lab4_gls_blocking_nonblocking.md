# Lab 4 — Gate-Level Simulation & the Blocking-Assignment Trap

## Goal

Understand what Gate-Level Simulation (GLS) actually checks, and why sloppy use of blocking (`=`) vs. non-blocking (`<=`) assignments — or an incomplete sensitivity list — can make RTL simulation lie to you about what the synthesized hardware will really do.

## What GLS Is

Up through Lab 4, every simulation ran the *original RTL* against a testbench. GLS swaps that out: same testbench, but now pointed at the **synthesized gate-level netlist** instead.

```
RTL           + testbench → RTL simulation
Gate netlist  + testbench → Gate-Level Simulation (GLS)
```

The testbench doesn't change at all — only the design underneath it does. If GLS and RTL simulation disagree, something about synthesis introduced a behavioral difference that plain RTL simulation never would have caught.

## Why Run GLS At All?

- Confirms synthesis didn't silently change the design's behavior
- Catches **synthesis-simulation mismatches** before they reach hardware
- When the gate models include delay annotations, it can also give a rough timing sanity-check (this is *timing GLS*, distinct from plain *functional GLS*)

```
Gate netlist + delay-annotated cell models → Timing-aware GLS
```

---

## Where Mismatches Actually Come From

Two RTL habits are responsible for the vast majority of synthesis-simulation mismatches:

1. An incomplete sensitivity list on a combinational `always` block
2. Using blocking assignments (`=`) where sequential logic needs non-blocking (`<=`)

### Problem 1 — Incomplete Sensitivity List

```verilog
always @(sel)
begin
    if (sel)
        y = i1;
    else
        y = i0;
end
```

This block only wakes up when `sel` changes. If `i0` or `i1` change while `sel` stays put, the block never re-runs — so in simulation, `y` can freeze on a stale value even though the real combinational hardware would have updated instantly.

**Fix — let Verilog build the sensitivity list for you:**
```verilog
always @(*)
begin
    if (sel)
        y = i1;
    else
        y = i0;
end
```

`@(*)` automatically includes every signal the block actually reads, so there's no way to accidentally leave one out.

| | `always @(sel)` | `always @(*)` |
|---|---|---|
| Reacts to `sel` changing | ✅ | ✅ |
| Reacts to `i0`/`i1` changing | ❌ | ✅ |
| Mismatch risk | High | Low |

### Problem 2 — Blocking Assignments in Sequential Logic

Consider two flip-flops meant to shift data through on successive clock edges:

```
d → [FF1: q0] → [FF2: q]
```

**Written with blocking assignments:**
```verilog
always @(posedge clk)
begin
    q0 = d;
    q  = q0;
end
```

Because `=` executes immediately and in program order, the second line sees the *already-updated* `q0` — so in simulation, `q` can appear to jump straight to the new value of `d` in the same clock edge, as if there were only one flip-flop instead of two.

**Written correctly with non-blocking assignments:**
```verilog
always @(posedge clk)
begin
    q0 <= d;
    q  <= q0;
end
```

With `<=`, every right-hand side is evaluated first, using the values as they stood *before* this clock edge, and only then are all the updates applied together. `q` correctly receives the *old* `q0`, matching what two real flip-flops would do.

```
Blocking (wrong for this case):   q0 = d;  q = new q0   → looks like 1 flop
Non-blocking (correct):           q0 <= d; q <= old q0  → correctly models 2 flops
```

### Why This Becomes a Synthesis Mismatch

Synthesis tools infer hardware from the *structure* of a clocked always block, not from the simulator's step-by-step execution order. So even when blocking assignments make the **simulator** show one merged behavior, Yosys can still generate two separate physical flip-flops from the same code — meaning the RTL simulation and the real synthesized hardware now disagree.

```
RTL simulation (blocking, order-dependent)   ≠   Synthesized hardware (structural)
```

Reordering the two blocking statements can even change the simulation result while the synthesized circuit stays identical — a clear sign that the simulation behavior was never a reliable stand-in for the hardware in the first place.

### The Governing Rule

| Logic type | Assignment | Reason |
|---|---|---|
| Combinational | Blocking `=` | Executes immediately; matches how combinational outputs are recalculated from current inputs |
| Sequential | Non-blocking `<=` | Models simultaneous register updates correctly |

---

## Hands-On: Ternary-Operator Mux, RTL → Netlist → GLS

**1. RTL:**
```verilog
module ternary_operator_mux (
    input  i0,
    input  i1,
    input  sel,
    output y
);
    assign y = sel ? i1 : i0;
endmodule
```

**2. RTL simulation:**
```bash
iverilog ternary_operator_mux.v tb_ternary_operator_mux.v
vvp a.out
gtkwave tb_ternary_operator_mux.vcd
```

**3. Synthesize to a gate-level netlist:**
```tcl
read_verilog ternary_operator_mux.v
synth -top ternary_operator_mux
abc -liberty ./sky130_fd_sc_hd__tt_025C_1v80.lib
write_verilog -noattr ternary_operator_mux_net.v
show
```

**4. Run GLS on the netlist, using the same testbench — but now also including the SKY130 cell-model files so the simulator knows how each standard cell behaves:**
```bash
iverilog \
  my_lib/verilog_model/primitives.v \
  my_lib/verilog_model/sky130_fd_sc_hd.v \
  ternary_operator_mux_net.v \
  tb_ternary_operator_mux.v
vvp a.out
gtkwave tb_ternary_operator_mux.vcd
```

**5. Compare** the RTL-sim waveform against the GLS waveform. They should match exactly for a clean design.



---

## Hands-On: Reproducing a Real Mismatch

Repeat the same RTL → netlist → GLS flow using a design that intentionally uses blocking assignments for its sequential logic (`blocking_caveat.v`):

```bash
# RTL simulation
iverilog blocking_caveat.v tb_blocking_caveat.v
vvp a.out
gtkwave tb_blocking_caveat.vcd
```

```tcl
# Synthesis
read_verilog blocking_caveat.v
synth -top blocking_caveat
abc -liberty ./sky130_fd_sc_hd__tt_025C_1v80.lib
write_verilog -noattr blocking_caveat_net.v
show
```

```bash
# GLS
iverilog \
  my_lib/verilog_model/primitives.v \
  my_lib/verilog_model/sky130_fd_sc_hd.v \
  blocking_caveat_net.v \
  tb_blocking_caveat.v
vvp a.out
gtkwave tb_blocking_caveat.vcd
```

**Expected outcome:** the RTL-sim waveform and the GLS waveform disagree — a live demonstration of a synthesis-simulation mismatch caused purely by coding style, not by any error in the tools.



---

## RTL Simulation vs. GLS — Summary

| | RTL Simulation | Gate-Level Simulation |
|---|---|---|
| Design under test | Behavioral RTL | Synthesized netlist |
| Abstraction level | High-level | Gate/cell-level |
| Speed | Faster | Slower |
| Checks | Intended functionality | Actual synthesized behavior |
| Timing detail | Usually none | Can include gate delays if models are annotated |
| When it's run | Before synthesis | After synthesis |

## Files in This Lab

| File | Purpose |
|------|---------|
| `ternary_operator_mux.v` | Clean combinational mux example |
| `blocking_caveat.v` | Sequential design misusing blocking assignments |
| `tb_*.v` | Testbenches (shared between RTL sim and GLS) |
| `*_net.v` | Synthesized gate-level netlists |
| `my_lib/verilog_model/` | SKY130 cell behavioral models required for GLS |

## Takeaways

- GLS exists specifically to catch the gap between what RTL simulation *shows* and what synthesized hardware *does*.
- `always @(*)` isn't just a style preference — an incomplete manual sensitivity list is a genuine correctness bug waiting to happen.
- Blocking assignments in sequential (`posedge clk`) blocks are a classic, well-documented source of synthesis-simulation mismatch — non-blocking is the rule for a reason.
- Running GLS requires the actual cell behavioral models (`primitives.v`, `sky130_fd_sc_hd.v`), not just the netlist — without them the simulator has no idea what a `sky130_fd_sc_hd__...` cell instance actually does.
