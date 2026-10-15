# Workshop Summary

This folder brings together the three requested workshop documents and the
shared workshop video link. The documents include the explanations, worked
examples, diagrams, recorded tool results, and caveats; the supporting lab and
module notes linked at the end of each document are additional detail, not a
substitute for the summaries here.

## Contents

| Document | Coverage |
|---|---|
| [RTL Online Course](./RTL_Online_Course.md) | Verilog, testbenches, simulation, Yosys/ABC synthesis, optimization, GLS, latches, and generated structures |
| [Physical Design Online Course](./PD_Online_Course.md) | Synthesis through floorplanning, PDN, placement, custom cells, CTS, routing, and timing checks |
| [VSD BabySoC Report](./VSDBabySoC.md) | Synthesis and post-route STA across 16 PVT corners, physical results, report assumptions, and open issues |
| [Workshop Video](./video/README.md) | Link to the workshop video hosted on Google Drive |

## Interpreting the recorded results

Measurements depend on the design revision, constraints, tool version, cell
libraries, operating corner, clock treatment, and parasitic source. Values from
different runs should not be combined as if they were one controlled experiment.
The BabySoC report separates synthesis-stage STA from the later post-route
database analysis and records remaining setup failures and met1 shorts; it does
not describe the design as sign-off complete.

The video is hosted externally on Google Drive rather than stored in this
repository. See the [video access notes](./video/README.md); viewers may need the
owner to set General access to **Anyone with the link** with **Viewer**
permission.
