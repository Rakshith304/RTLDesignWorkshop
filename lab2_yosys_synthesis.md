# Lab 2 — Logic Synthesis with Yosys and the SKY130 Library

## Goal

Take the mux RTL verified in Lab 1 and convert it into a **gate-level netlist** built from real standard cells in the SKY130 open-source PDK, using Yosys.

## What Synthesis Actually Does

Simulation confirms *behavior*. Synthesis answers a different question: *which physical logic gates, from a specific manufacturing library, implement that behavior?* The output isn't abstract Verilog anymore — it's a netlist of concrete cells (`sky130_fd_sc_hd__...`) wired together, ready for downstream physical design steps.

```
RTL (behavioral)  →  Yosys  →  Gate-level netlist (structural, library-specific)
```

## What's Inside a `.lib` File

The `.lib` (Liberty format) file is what tells Yosys/ABC what cells are even available to use. For each cell it lists things like:

- the logic function it implements
- input/output pin names
- delay and timing arcs
- power draw
- physical area
- multiple drive-strength variants of the same function (e.g. a mux cell available in several sizes)

Yosys can't map anything to real silicon without this file — it's the bridge between abstract logic and the actual SKY130 cell set.

## Why Multiple Drive Strengths Exist

The same logical cell (say, a 2:1 mux) is usually offered in several sizes:

- **Larger/faster variants** — lower delay, but bigger area and higher power draw
- **Smaller/slower variants** — less area and power, but slower switching

There's no universally "best" choice — the synthesis tool picks based on the timing, area, and power targets of the design. Blindly using the fastest cell everywhere wastes area and power for no benefit on non-critical paths.

## Step-by-Step Synthesis Flow

**1. Launch Yosys from the directory containing your source files:**
```bash
yosys
```

**2. Load the standard-cell library:**
```tcl
read_liberty -lib ./sky130_fd_sc_hd__tt_025C_1v80.lib
```

**3. Load the RTL:**
```tcl
read_verilog mux2to1.v
```

**4. Run synthesis, specifying the top module:**
```tcl
synth -top mux2to1
```

**5. Perform technology mapping using ABC:**
```tcl
abc -liberty ./sky130_fd_sc_hd__tt_025C_1v80.lib
```
This is the step where Yosys hands off the optimized logic network to ABC, which maps it onto actual SKY130 cells.

**6. View the mapped circuit graphically:**
```tcl
show mux2to1
```
> Note: once multiple modules are loaded (which happens automatically once library cells come in during `abc`), `show` needs an explicit module name — running it with no argument will throw an ambiguity error.


**7. Write out the synthesized netlist:**
```tcl
write_verilog -noattr mux2to1_netlist.v
```
The `-noattr` flag strips Yosys's internal synthesis attributes so the output is clean, readable Verilog.

**8. (Optional) Open the netlist in an editor directly from the Yosys prompt:**
```tcl
!gvim mux2to1_netlist.v
```

## What the Mapped Netlist Looks Like

For a simple mux, ABC typically maps the whole function onto a single dedicated SKY130 mux cell, e.g.:

```
sky130_fd_sc_hd__mux2_1
```

with `i0`, `i1`, `sel` as inputs and `y` as the output — matching the original RTL's ports and function exactly. (Depending on the exact library/version, a mux can instead be broken into a combination of NAND/inverter/AOI-OAI cells — the specific decomposition can vary, but the logical behavior must not.)


## Verifying the Netlist Still Works

Synthesis must never change what the circuit *does* — only how it's implemented. The netlist can be checked against the same testbench used for RTL simulation:

```bash
iverilog -o netlist_sim.out mux2to1_netlist.v tb_mux2to1.v -I ./sky130_lib_models
vvp netlist_sim.out
gtkwave tb_mux2to1.vcd
```

If the resulting waveform matches Lab 1's RTL simulation exactly, the synthesis preserved functional correctness.


## Files in This Lab

| File | Purpose |
|------|---------|
| `mux2to1.v` | Original RTL (from Lab 1) |
| `sky130_fd_sc_hd__tt_025C_1v80.lib` | Standard-cell timing library |
| `mux2to1_netlist.v` | Synthesized, technology-mapped netlist |

## Takeaways

- `.lib` files carry the real electrical/timing/area data that lets a synthesis tool make informed cell choices.
- ABC does the heavy lifting of mapping optimized logic onto actual library cells.
- `show` needs a module name once more than one module is loaded — this trips people up constantly.
- A synthesized netlist should be re-simulated against the original testbench as a sanity check, not just trusted blindly.
