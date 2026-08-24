# Lab 5 — Conditional Constructs, Latch Inference, and Hardware Generation

## Goal

Look at how `if`/`case` coding choices shape inferred hardware (and how getting them wrong silently creates latches), then move on to `for` loops and `generate` blocks for building repeated structures like wide muxes and ripple-carry adders.

---

## Part A — `if` / `else if` / `else`: Priority Logic

An `if`-`else if` chain always implies **priority** — the first true condition wins, even if a later one would also be true.

```verilog
if (cond1)
    y = c1;
else if (cond2)
    y = c2;
else if (cond3)
    y = c3;
else
    y = 1'b0;
```

Structurally, this behaves like a chain of muxes, each one only reachable if every condition above it was false — `cond1` effectively overrides everything below it.

### The Danger — Incomplete `if`

```verilog
always @(*)
begin
    if (cond1)
        y = a;
    else if (cond2)
        y = b;
end
```

What happens when *both* `cond1` and `cond2` are false? There's no assignment to `y` at all — so simulation (and synthesis) has to assume `y` **holds its previous value**. For a combinational block, "holding a previous value" means only one thing: a **latch** got inferred, whether you wanted one or not.

### The Fix — Always Give Every Path an Assignment

**Option 1 — default before the conditionals:**
```verilog
always @(*)
begin
    y = 1'b0;
    if (cond1)
        y = a;
    else if (cond2)
        y = b;
end
```

**Option 2 — explicit final `else`:**
```verilog
always @(*)
begin
    if (cond1)
        y = a;
    else if (cond2)
        y = b;
    else
        y = 1'b0;
end
```

Either way, `y` is guaranteed a value under every possible condition — no latch.

### Note: This Doesn't Apply to Clocked Blocks

```verilog
always @(posedge clk or posedge reset)
begin
    if (reset)
        count <= 3'b000;
    else if (en)
        count <= count + 1;
end
```
Here, when `en = 0`, `count` intentionally holds its value — that's exactly what a register is supposed to do. "Incomplete" only signals a latch problem in a **combinational** (`always @(*)`) block, not a clocked one.

---

## Part B — `case`: Selection Logic

`case` is the natural fit when choosing among mutually exclusive options — muxes, decoders, state machines.

```verilog
always @(*)
begin
    case (sel)
        2'b00: y = c1;
        2'b01: y = c2;
        2'b10: y = c3;
        2'b11: y = c4;
    endcase
end
```
This is a 4:1 mux, selected by `sel`.

### Incomplete `case` = Same Latch Problem

```verilog
always @(*)
begin
    case (sel)
        2'b00: y = a;
        2'b01: y = b;
    endcase
end
```
`sel = 2'b10` and `2'b11` are left unhandled — `y` has no assignment for those values, so a latch gets inferred, exactly as with an incomplete `if`.

**Fix — add a `default`:**
```verilog
always @(*)
begin
    case (sel)
        2'b00:   y = a;
        2'b01:   y = b;
        default: y = 1'b0;
    endcase
end
```

### Partial Assignment Inside a Case Branch

A subtler version of the same bug: even a *complete* case (every value of `sel` handled) can still infer a latch if a branch forgets to assign one of several outputs.

```verilog
always @(*)
begin
    case (sel)
        2'b00: begin x = a; y = b; end
        2'b01: begin x = c; end          // y not assigned here!
        default: begin x = d; y = b; end
    endcase
end
```
In the `2'b01` branch, `x` gets a value but `y` doesn't — so `y` infers a latch specifically for that branch, even though the case as a whole looks "complete."

**Fix — pre-assign every output before the case:**
```verilog
always @(*)
begin
    x = 1'b0;
    y = 1'b0;
    case (sel)
        2'b00: begin x = a; y = b; end
        2'b01: begin x = c; end
        default: begin x = d; y = b; end
    endcase
end
```

### `if` vs. `case` — When to Use Which

| | `if` / `else if` | `case` |
|---|---|---|
| Semantics | Priority | Mutually exclusive selection |
| Typical hardware | Priority mux chain | Flat mux / decoder |
| Use when | Some conditions should override others | Options are cleanly distinct, no priority needed |

---

## Part C — `for` Loops vs. `generate for`

These look similar but solve completely different problems.

**Procedural `for`** — lives inside an `always` block, used for repeated *evaluation*:
```verilog
always @(*)
begin
    for (i = 0; i < 32; i = i + 1)
    begin
        if (i == sel)
            y = inp[i];
    end
end
```
This is how a wide (e.g. 32:1) mux can be written compactly instead of a 32-branch case statement.

**`generate for`** — lives outside procedural blocks, used for repeated *hardware instantiation*:
```verilog
genvar i;
generate
    for (i = 0; i < 8; i = i + 1)
    begin
        and u1 (.a(a[i]), .b(b[i]), .y(y[i]));
    end
endgenerate
```
This actually creates 8 separate AND gate instances — real, distinct hardware — not a loop that gets "executed."

| | Procedural `for` | `generate for` |
|---|---|---|
| Where it's used | Inside `always` | Outside procedural blocks |
| What it does | Repeats an evaluation | Replicates hardware/instances |

---

## Part D — Building a Ripple Carry Adder with `generate`

Given a single Full Adder module already written, a 4-bit Ripple Carry Adder is just four Full Adders chained by carry:

```
       num1[0] num2[0]         num1[1] num2[1]
            │     │                │     │
            ▼     ▼                ▼     ▼
  cin --> [ FA0 ] --> sum[0]  c1 → [ FA1 ] --> sum[1] --> c2 → ...
```

Rather than instantiating each Full Adder by hand:
```verilog
genvar i;
generate
    for (i = 0; i < 4; i = i + 1)
    begin
        full_adder FA (
            .a(num1[i]),
            .b(num2[i]),
            .cin(carry[i]),
            .sum(sum[i]),
            .cout(carry[i+1])
        );
    end
endgenerate
```
`carry[0]` connects to the adder's external `cin`. This scales cleanly — widening the adder to 8 or 16 bits is a one-line change to the loop bound, not a rewrite.


---

## Lab Walkthroughs

### Lab — Incomplete `if` (Latch Demonstration)

```bash
iverilog incomp_if.v tb_incomp_if.v
vvp a.out
gtkwave tb_incomp_if.vcd
```
```tcl
yosys
read_liberty -lib ./sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog incomp_if.v
synth -top incomp_if
abc -liberty ./sky130_fd_sc_hd__tt_025C_1v80.lib
show
```
**Expected result:** a latch is visible in the synthesized output.



### Lab — Complete `if` (No Latch)

Same flow, run against `comp_if.v` (the fixed, fully-assigned version). Confirm no latch appears in `show`.

`[SCREENSHOT: comp_if_show_no_latch]`

### Lab — Incomplete `case`

```bash
iverilog incomp_case.v tb_incomp_case.v
vvp a.out
gtkwave tb_incomp_case.vcd
```
```tcl
read_verilog incomp_case.v
synth -top incomp_case
abc -liberty ./sky130_fd_sc_hd__tt_025C_1v80.lib
show
```
**Expected result:** a latch is inferred, same root cause as the incomplete `if` case.


### Lab — Complete `case` with `default`

Same flow, run against `comp_case.v`. With every branch (including `default`) assigning `y`, no latch should appear.


### Lab — Overlapping / Bad `case`

```bash
iverilog bad_case.v tb_bad_case.v
vvp a.out
gtkwave tb_bad_case.vcd
```
```tcl
read_verilog bad_case.v
synth -top bad_case
abc -liberty ./sky130_fd_sc_hd__tt_025C_1v80.lib
write_verilog -noattr bad_case_net.v
```
Then run GLS to check for a mismatch:
```bash
iverilog ../my_lib/verilog_model/primitives.v ../my_lib/verilog_model/sky130_fd_sc_hd.v bad_case_net.v tb_bad_case.v
vvp a.out
gtkwave tb_bad_case.vcd
```


### Lab — Mux/Demux via `case` vs. `generate`

Both a `case`-based demux and a `generate`-based demux describe the same function:

```verilog
// case-based
always @(*)
begin
    op_bus = 8'b0;
    case (sel)
        3'd0: op_bus[0] = input;
        3'd1: op_bus[1] = input;
        // ... through 3'd7
    endcase
end
```
```verilog
// generate-based
genvar i;
generate
    for (i = 0; i < 8; i = i + 1)
    begin
        assign op_bus[i] = (sel == i) ? input : 1'b0;
    end
endgenerate
```

Simulate each independently and confirm the resulting waveforms are identical:
```bash
iverilog demux_case.v tb_demux_case.v
vvp a.out
gtkwave tb_demux_case.vcd

iverilog demux_generate.v tb_demux_generate.v
vvp a.out
gtkwave tb_demux_generate.vcd
```



### Lab — Ripple Carry Adder

```bash
iverilog fa.v rca.v tb_rca.v
vvp a.out
gtkwave tb_rca.vcd
```
```tcl
read_verilog fa.v rca.v
synth -top rca
abc -liberty ./sky130_fd_sc_hd__tt_025C_1v80.lib
write_verilog -noattr rca_net.v
```
Then re-simulate the netlist to confirm the synthesized adder still adds correctly:
```bash
iverilog rca_net.v tb_rca.v
vvp a.out
gtkwave tb_rca.vcd
```


---

## Files in This Lab

| File | Purpose |
|------|---------|
| `incomp_if.v` / `comp_if.v` | Latch vs. no-latch `if` examples |
| `incomp_case.v` / `comp_case.v` | Latch vs. no-latch `case` examples |
| `bad_case.v` | Overlapping/ambiguous case conditions |
| `demux_case.v` / `demux_generate.v` | Equivalent demux implementations |
| `fa.v` / `rca.v` | Full adder + 4-bit ripple carry adder |
| `tb_*.v` | Corresponding testbenches |

## Takeaways

- Incomplete `if`/`case` in a **combinational** block is the single most common cause of accidental latch inference — always give every output a value on every path.
- `if` implies priority; `case` implies flat, mutually exclusive selection — picking the wrong one for the intent muddies both the RTL's readability and its synthesized structure.
- Procedural `for` repeats *evaluation inside a block*; `generate for` repeats *actual hardware instances* — they are not interchangeable.
- `generate` is what makes scalable structures (wide muxes, ripple-carry adders, gate arrays) practical to write and maintain.
