# Lab 3 — Timing Libraries, Hierarchical vs. Flat Synthesis, and Flip-Flop Coding Styles

## Goal

Go one level deeper than Lab 2: understand what the naming of a `.lib` file actually encodes (PVT corners), how Yosys handles multi-module designs (hierarchy vs. flattening), and how the way a flip-flop is written in Verilog affects what hardware actually gets inferred.

---

## Part A — Reading a Library's PVT Corner

A SKY130 liberty file is generated for a specific set of operating conditions, and the filename tells you exactly which ones. Take:

```
sky130_fd_sc_hd__tt_025C_1v80.lib
```

Breaking it down:

| Segment | Meaning |
|---------|---------|
| `sky130` | SkyWater 130nm process node |
| `fd_sc` | Fully-digital standard cell family |
| `hd` | High-density variant |
| `tt` | Process corner: **typical-typical** |
| `025C` | Characterized at 25°C |
| `1v80` | Characterized at 1.80V supply |

### Why PVT Matters

**P — Process:** No two fabricated chips are perfectly identical; manufacturing introduces natural variation from wafer to wafer. Corner libraries (slow-slow, fast-fast, typical-typical, etc.) bound this variation so a design can be checked against the worst and best cases, not just the nominal one.

**V — Voltage:** Supply voltage isn't perfectly constant in real operation, and circuit delay is voltage-dependent — lower voltage generally means slower switching.

**T — Temperature:** Delay and leakage both shift with temperature, and a chip in the field will see a range of operating temperatures, not a single fixed value.

A design that's only verified against one PVT corner isn't actually verified — it's just been checked at one point in a wide operating envelope. In practice, real timing closure requires checking multiple corners (worst-case for setup, best-case for hold, etc.).

---

## Part B — Hierarchical vs. Flat Synthesis

### Hierarchical

When a design is built from clearly separated submodules, Yosys can synthesize it while keeping those module boundaries intact.

```
top_module
 ├── submodule_A
 └── submodule_B
```

**Advantages:**
- Easier to navigate and debug
- Submodules can be reused elsewhere
- Natural fit for large, team-authored designs

### Flat

The `flatten` command in Yosys collapses all submodule boundaries into one unified logic network:

```tcl
flatten
```

Once flattened, optimization can happen *across* what used to be separate modules — potentially finding redundant logic or better cell packing that hierarchy boundaries would have hidden.

```
Hierarchical:  boundaries preserved → easier to read, less cross-module optimization
Flat:          boundaries removed  → harder to read, more optimization opportunity
```

Neither is strictly "better" — it depends on whether you need the design to stay human-readable/modular or whether you're optimizing for the smallest/fastest possible final implementation.

### Example: Instantiating a Submodule

```verilog
module and_gate (
    input  a,
    input  b,
    output y
);
    assign y = a & b;
endmodule

module top (
    input  a,
    input  b,
    output y
);
    and_gate u_and (
        .a(a),
        .b(b),
        .y(y)
    );
endmodule
```

`u_and` is the *instance name* — it's how this particular copy of `and_gate` is identified inside `top`, distinct from the module definition itself.



---

## Part C — Flip-Flop Coding Styles

A flip-flop is the basic sequential element used to hold state across clock cycles — it has a data input (`D`), a clock (`CLK`), typically a reset, and an output (`Q`). How it's described in Verilog directly shapes what hardware Yosys infers, so sloppy coding here can silently produce the wrong circuit (stray latches, broken reset behavior, etc.).

### Synchronous Reset

The reset is only evaluated on the active clock edge — it has no effect between clock edges, no matter what.

```verilog
always @(posedge clk) begin
    if (rst)
        q <= 1'b0;
    else
        q <= d;
end
```

### Asynchronous Reset

The reset is in the sensitivity list alongside the clock, so it takes effect the moment it's asserted — independent of any clock edge.

```verilog
always @(posedge clk or posedge rst) begin
    if (rst)
        q <= 1'b0;
    else
        q <= d;
end
```

### Comparing the Two

| | Synchronous | Asynchronous |
|---|---|---|
| When reset takes effect | Only at clock edge | Immediately, any time |
| Sensitivity list | `posedge clk` only | `posedge clk or posedge rst` |
| Common use case | Predictable, clock-aligned timing analysis | Fast/immediate initialization (e.g. power-on reset) |

### Why It Matters for Initialization

A flip-flop with no reset logic starts in an unknown state (`X`) in simulation, and its real hardware equivalent has no guaranteed power-up value either. Explicitly coding a reset path is how a design guarantees a known starting state.

---

## Part D — Synthesizing a Flip-Flop Design in Yosys

Sequential elements need one extra mapping step beyond what Lab 2 covered: mapping the inferred flip-flop onto an actual SKY130 flip-flop cell.

```tcl
read_verilog dff_async_reset.v
synth -top dff_async_reset
dfflibmap -liberty ./sky130_fd_sc_hd__tt_025C_1v80.lib
abc -liberty ./sky130_fd_sc_hd__tt_025C_1v80.lib
write_verilog -noattr dff_netlist.v
```

### `dfflibmap` vs. `abc` — Different Jobs

- **`dfflibmap`** — specifically maps the flip-flops Yosys inferred from your `always` blocks onto real sequential cells available in the library.
- **`abc`** — handles the surrounding *combinational* logic optimization and mapping.

Both need to run; skipping `dfflibmap` leaves flip-flops unmapped even if `abc` successfully maps everything else.



---

## Files in This Lab

| File | Purpose |
|------|---------|
| `dff_sync_reset.v` | Synchronous-reset flip-flop RTL |
| `dff_async_reset.v` | Asynchronous-reset flip-flop RTL |
| `sky130_fd_sc_hd__tt_025C_1v80.lib` | Standard-cell + timing library |
| `dff_netlist.v` | Synthesized, technology-mapped netlist |

## Takeaways

- A `.lib` filename encodes the exact PVT corner it was characterized at — this isn't cosmetic, it determines how trustworthy your timing numbers are for real silicon.
- Hierarchical synthesis keeps structure readable; flattening trades readability for cross-module optimization headroom.
- Sync vs. async reset isn't just a style choice — it changes when the reset actually takes effect in real hardware.
- `dfflibmap` and `abc` are complementary, not interchangeable — sequential and combinational mapping are handled separately.
