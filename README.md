# Digital VLSI Lab Journal

This repository documents my hands-on work in digital VLSI design — moving from RTL Verilog code through simulation, logic synthesis, and technology mapping onto the SKY130 open-source PDK. All experiments were carried out on a Linux (Ubuntu) virtual machine using open-source EDA tools.

The goal of this journal is to record what I did, the commands I ran, and what I learned at each stage of the RTL-to-netlist flow.

---

## 📁 Contents

| # | Topic | Link |
|---|-------|------|
| 1 | Simulating a 2:1 Mux with Icarus Verilog & GTKWave | [lab1_mux_simulation.md](./lab1_mux_simulation.md) |
| 2 | Logic Synthesis with Yosys + SKY130 | [lab2_yosys_synthesis.md](./lab2_yosys_synthesis.md) |
| 3 | Timing Libraries, Hierarchy, and Flip-Flop Coding | [lab3_libraries_and_flops.md](./lab3_libraries_and_flops.md) |

---

## 🧰 Tools Used

- **Verilog** — RTL description language
- **Icarus Verilog (`iverilog`)** — compiles and simulates RTL/testbenches
- **GTKWave** — waveform viewer for `.vcd` dumps
- **Yosys** — open-source RTL synthesis tool
- **ABC** — logic optimization / technology mapping engine (invoked from within Yosys)
- **SKY130 PDK** — SkyWater 130nm open-source standard-cell library
- **Ubuntu VM** — working environment for the whole flow

---

## 🔄 The Flow, End to End

```
Verilog RTL
   │
   ▼
Testbench simulation (iverilog + GTKWave)  →  confirm functional correctness
   │
   ▼
Yosys: read_verilog → synth → technology mapping (abc + dfflibmap)
   │
   ▼
Gate-level netlist (SKY130 standard cells)
   │
   ▼
Re-simulate netlist → compare against original RTL waveforms
```

Each lab below expands on one stage of this pipeline.

---

## 📝 Notes on this repo

This is a personal lab record — command sequences, explanations, and takeaways are written in my own words based on my own runs of the tools. Screenshots referenced in each file are marked with placeholders (e.g. `[SCREENSHOT: waveform_output]`) — I'll drop my own images into those spots.
