# Digital VLSI Lab Journal

This repository documents my hands-on work in digital VLSI design — moving from RTL Verilog code through simulation, logic synthesis, and technology mapping onto the SKY130 open-source PDK. All experiments were carried out on a Linux (Ubuntu) virtual machine using open-source EDA tools.

The goal of this journal is to record what I did, the commands I ran, and what I learned at each stage of the RTL-to-netlist flow.

---

## 📁 Contents

| # | Topic | Link |
|---|-------|------|
| 1 | RTL Online Course | [RTL_Online_Course.md](./WORKSHOP_SUMMARY/RTL_Online_Course.md) |
| 2 | Physical Design Online Course | [PD_Online_Course.md](./WORKSHOP_SUMMARY/PD_Online_Course.md) |
| 3 | VSD BabySoC STA and Physical-Design Report | [VSDBabySoC.md](./WORKSHOP_SUMMARY/VSDBabySoC.md) |

The [workshop summary index](./WORKSHOP_SUMMARY/README.md) links to the standalone RTL
course, physical-design course, and BabySoC report. The documents explain their
context and results; they also link to the detailed lab/module notes for reference.

## Workshop Video

The shared workshop video is available in the [summary video folder](./WORKSHOP_SUMMARY/video/README.md).

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
