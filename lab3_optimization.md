# Lab 3 — Combinational and Sequential Logic Optimization

## Goal

Explore how Yosys simplifies logic during synthesis — both combinational (constant propagation, Boolean simplification) and sequential (removing flip-flops whose output can be proven constant, trimming unused counter bits).

## Why Bother Optimizing?

A synthesis tool doesn't just map RTL onto cells 1-to-1 — it actively looks for logic that can be simplified away entirely. The payoff is smaller area, lower power draw, and often better timing, all without changing what the circuit does.

---

## Part A — Combinational Optimization

### Constant Propagation

If a signal feeding into a piece of logic is a known constant, the tool can algebraically simplify the whole expression around that fact.

**Example:**
```
Y = A·B + C'
```
If `A` is tied to `0`:
```
Y = (0)·B + C'
Y = C'
```
The entire AND/OR structure collapses into a single inverter. What might have taken several transistors to implement now only needs one gate.

### Boolean Simplification

Beyond constants, straightforward algebraic identities (or a K-map / Quine–McCluskey pass) can collapse redundant mux chains or logic terms into a smaller equivalent expression. The point is always the same: fewer gates for the same truth table.

### Hands-On: Three Mux Variants

**`opt_check1.v`** — mux where input 0 is tied to 0:
```verilog
module opt_check1(input a, input b, output y);
    assign y = a ? b : 0;
endmodule
```
Expands to `y = a·b` — the mux reduces to a plain AND gate.

**`opt_check2.v`** — mux where input 1 is tied to 0:
```verilog
module opt_check2(input a, input b, output y);
    assign y = a ? 0 : b;
endmodule
```
Expands to `y = a'·b` — an AND gate with one inverted input.

**`opt_check3.v`** — nested mux:
```verilog
module opt_check3(input a, input b, input c, output y);
    assign y = a ? (c ? b : 0) : 0;
endmodule
```
Expands to `y = a·b·c` — a 3-input AND, despite starting out as two nested muxes.

**Synthesizing any of these:**
```tcl
yosys
read_liberty -lib ./sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog opt_check1.v
synth -top opt_check1
opt_clean -purge
abc -liberty ./sky130_fd_sc_hd__tt_025C_1v80.lib
show
```

`opt_clean -purge` is the key extra step here — it sweeps away now-dangling wires and cells left behind after the constant folding, rather than leaving them in the netlist unused.



---

## Part B — Sequential Optimization

### Sequential Constant Propagation

This is the sequential equivalent of the combinational case: if a flip-flop's output can be *proven* to always settle on the same constant value, the flop (and whatever logic only depends on it) can be removed entirely.

**Example — a flop held in reset:**
```
Q = 0   (proven constant, e.g. always reset)
Y = A · Q'
```
Substituting:
```
Y = A · 1
Y = A
```
The whole AND-with-a-flop structure disappears — `Y` just becomes a direct wire to `A`.

**Important caveat:** having a `set`/`reset` pin on a flip-flop does *not* automatically make it a constant. If the flop's `Q` can still change based on `D` and `clk` under some condition (e.g. `set=0`), it's not a sequential constant — the tool can only remove it once it's proven `Q` never varies.

**Simulating the constant-propagation examples (`dff_const1.v`, `dff_const2.v`, `dff_const3.v`):**
```bash
iverilog dff_const1.v tb_dff_const1.v
vvp a.out
gtkwave tb_dff_const1.vcd
```
Repeat similarly for `dff_const2.v` and `dff_const3.v` to compare behavior.



**Synthesizing to confirm the flop was actually removed:**
```tcl
yosys
read_liberty -lib ./sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog dff_const2.v
synth -top dff_const2
opt_clean -purge
abc -liberty ./sky130_fd_sc_hd__tt_025C_1v80.lib
show
```

Compare the resulting circuit against the un-optimized version — if the constant propagation worked, the flop should be gone entirely, replaced by a direct connection or a trivial gate.


### Removing Unused Sequential Bits

Optimization also applies at the bit level. Consider a 3-bit counter where only bit 0 is ever actually used externally:

```verilog
module counter_opt(input clk, input reset, output q);
    reg [2:0] count;
    assign q = count[0];

    always @(posedge clk, posedge reset) begin
        if (reset)
            count <= 3'b000;
        else
            count <= count + 1;
    end
endmodule
```

Since `count[1]` and `count[2]` never drive any observable output, the synthesis tool can determine they're unnecessary given this module's actual interface, and may optimize the associated flip-flops away — depending on how aggressively the flow is configured.

```tcl
yosys
read_liberty -lib ./sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog counter_opt.v
synth -top counter_opt
dfflibmap -liberty ./sky130_fd_sc_hd__tt_025C_1v80.lib
abc -liberty ./sky130_fd_sc_hd__tt_025C_1v80.lib
show
```


---

## Files in This Lab

| File | Purpose |
|------|---------|
| `opt_check1.v` / `opt_check2.v` / `opt_check3.v` | Combinational mux-constant examples |
| `dff_const1.v` / `dff_const2.v` / `dff_const3.v` | Sequential constant-propagation examples |
| `counter_opt.v` | Counter with an unused upper bit range |
| `sky130_fd_sc_hd__tt_025C_1v80.lib` | Standard-cell library |

## Takeaways

- Constant propagation works the same way whether it's combinational logic or a flip-flop's output — prove the value never changes, and the hardware behind it can go.
- `opt_clean -purge` is what actually removes the dangling logic left over after optimization passes — it doesn't happen automatically just by running `synth`.
- A reset/set pin alone doesn't make a flop "constant" — only a provably-always-fixed `Q` does.
- Bit-level pruning (unused counter bits) is a real optimization outcome, not just a theoretical one — always check the `show` output to confirm what actually got removed.
