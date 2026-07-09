# OpenLane 2 + Sky130 — RTL to GDS (Half Adder)

# OpenLane 2 Installation and ASIC Design Flow Guide

This document provides a step-by-step guide for setting up **OpenLane 2** on **Ubuntu 22.04**, installing the **Sky130 PDK**, and running a complete digital ASIC implementation flow—from a **Verilog design** through synthesis, floorplanning, placement, routing, and finally generating a **GDSII** layout for inspection in **KLayout**.

In addition to the standard workflow, the guide documents common installation and runtime issues, along with their solutions, making it a practical reference for both reproducing the flow and troubleshooting problems that may arise.

---

## System Configuration
 
| Component | Version |
|---|---|
| OS | Ubuntu 22.04.5 LTS |
| CPU | AMD Ryzen 5 3500U |
| RAM | 6 GB |
| Python | 3.10 |
| Docker | Installed |
| Git | Installed |
| OpenLane | OpenLane 2 (new CLI, `--design-dir` based) |

---

## 1. Install Nix

OpenLane 2 uses **Nix** to manage all tool dependencies (Yosys, OpenROAD, Magic, Netgen, etc.) so you don't have to install each one manually.

```bash
sudo apt update
sudo apt install -y curl
```

Install Nix (Determinate Systems installer, with the OpenLane binary cache pre-configured):

```bash
curl --proto '=https' --tlsv1.2 -sSf -L https://install.determinate.systems/nix | sh -s -- install --no-confirm --extra-conf "
extra-substituters = https://openlane.cachix.org
extra-trusted-public-keys = openlane.cachix.org-1:qqdwh+QMNGmZAuyeQJTH9ErW57OWSvdtuwfBKdS254E=
"
```

**Restart the terminal** after this (new shell, not just `source`).

Verify:

```bash
nix --version
nix doctor
```

---

## 2. Create Workspace and Clone OpenLane 2

```bash
mkdir -p ~/vlsi
cd ~/vlsi
git clone https://github.com/efabless/openlane2.git
cd openlane2
```

---

## 3. Enter the OpenLane Nix Shell

```bash
nix-shell
```

First run takes a while — it's pulling the whole toolchain. Subsequent runs are fast.

Verify tools are on PATH inside the shell:

```bash
type openlane
openlane --version
openroad -version
yosys -V
verilator --version
netgen -batch
```

### ⚠️ Mistake #1 — leftover OpenLane 1 alias

If you previously tried OpenLane 1, you may have an old alias sitting in `~/.bashrc`:

```bash
alias openlane="cd ~/OpenLane"
```

This silently breaks everything — `openlane` stops being the actual binary and just `cd`s you somewhere. Symptom looks like:

```
bash: cd: too many arguments
```

**Fix:**

```bash
unalias openlane
nano ~/.bashrc     # delete the alias line
source ~/.bashrc
```

---

## 4. Download the Sky130 PDK

No manual PDK download needed — running the smoke test triggers OpenLane to fetch everything via Volare:

```bash
openlane --smoke-test
```

This pulls down:
- Sky130 PDK
- Standard cell libraries (`sky130_fd_sc_hd`, etc.)
- Required tech/LEF/lib files

---

## 5. ⚠️ Mistake #2 — Using the old OpenLane 1 CLI syntax

My first instinct was to run the old-style command:

```bash
openlane --design ~/vlsi/projects/half_adder
```

Which failed with:

```
Error: No such option: --design (Possible options: --design-dir, --version)
```

**Why:** OpenLane 2's CLI is different from OpenLane 1. It doesn't take `--design`; it takes a **design directory** plus a config file, using `--design-dir`.

Always check the current syntax first with:

```bash
openlane --help
```

**Correct invocation for this version:**

```bash
cd ~/vlsi/projects/half_adder
openlane --design-dir . config.json
```

---

## 6. Create the Project

```bash
mkdir -p ~/vlsi/projects/half_adder/src
cd ~/vlsi/projects/half_adder
```

Target structure:

```
half_adder/
├── config.json
└── src/
    └── half_adder.v
```

---

## 7. Half Adder RTL

`src/half_adder.v`

```verilog
module half_adder (
    input  wire a,
    input  wire b,
    output wire sum,
    output wire carry
);

assign sum   = a ^ b;
assign carry = a & b;

endmodule
```

---

## 8. ⚠️ Mistake #3 — PDN generation failing on a tiny design

First `config.json` attempt (minimal, default floorplan sizing):

```json
{
    "DESIGN_NAME": "half_adder",
    "VERILOG_FILES": ["dir::src/half_adder.v"],
    "CLOCK_PORT": null,
    "CLOCK_PERIOD": 10,
    "FP_PDN_MULTILAYER": true
}
```

This got through synthesis, floorplanning, and placement, but died at **Power Distribution Network (PDN) generation**:

```
[PDN-0185] Insufficient width (6.44 um) to add straps on layer met4
in grid "stdcell_grid" with total strap width 4.9 um and offset 16.32 um.
```

**What was actually going on:**
A half adder synthesizes down to almost nothing — in my run it was just:

```
Total cells: 6
Total area: 38.79 µm²
```

The default Sky130 PDN template assumes a much bigger core and tries to lay down power straps that are simply wider than the die. Setting `FP_PDN_MULTILAYER: false` did **not** fix it — that only changes *how* the PDN is built, not whether it fits.

**Fix — force an oversized, absolute floorplan** so there's physically enough room for the power straps:

```json
{
    "DESIGN_NAME": "half_adder",
    "VERILOG_FILES": [
        "dir::src/half_adder.v"
    ],
    "CLOCK_PORT": null,
    "FP_SIZING": "absolute",
    "DIE_AREA": [0, 0, 300, 300],
    "CORE_AREA": [20, 20, 280, 280]
}
```

Notes on the fix:
- `CLOCK_PERIOD` removed — a purely combinational design doesn't need one, and OpenROAD just complains about a virtual clock (`STA-0366`, `STA-0419`, `STA-0450` — harmless warnings, not errors, for this design).
- `FP_SIZING: "absolute"` + explicit `DIE_AREA` / `CORE_AREA` gives the PDN generator enough room to route full-width straps on `met4`.

This config completed the full flow successfully.

---

## 9. Run the Flow

```bash
cd ~/vlsi/projects/half_adder
openlane --design-dir . config.json
```

Stages executed automatically:

```
RTL (Verilog)
   │
   ▼
Yosys            → synthesis
   │
   ▼
OpenROAD         → floorplan → placement → CTS → routing
   │
   ▼
Magic            → DRC
   │
   ▼
Netgen           → LVS
   │
   ▼
GDSII
```

---

## 10. Output Layout

```
runs/
└── RUN_2026-07-09_20-55-31/
    ├── final/
    │   ├── gds/
    │   ├── lef/
    │   ├── def/
    │   └── verilog/
    ├── reports/
    └── logs/
```

---

## 11. ⚠️ Mistake #4 — Getting lost searching for the GDS

Running a blind `find ~ -name "*.gds"` turns up **thousands** of results, because the Sky130 PDK ships one GDS per standard cell, I/O cell, macro, etc. (`sky130_fd_sc_hd__and2_0.gds`, `sky130_fd_sc_hd__xor2_1.gds`, `sky130_fd_io.gds`, ...). None of those are the chip.

**Fix — filter specifically for the flow's final output directory:**

```bash
find ~/vlsi/projects/half_adder -name "*.gds"
```

or more generally, target the `runs/.../final/gds/` pattern:

```bash
find ~ -path "*/runs/*/final/gds/*.gds"
```

The one that matters looks like:

```
~/vlsi/projects/half_adder/runs/RUN_2026-07-09_20-55-31/final/gds/half_adder.gds
```

---

## 12. View the Layout in KLayout

```bash
klayout ~/vlsi/projects/half_adder/runs/RUN_2026-07-09_20-55-31/final/gds/half_adder.gds
```

| Key | Function |
|---|---|
| `F` | Fit entire layout to view |
| Mouse wheel | Zoom |
| Middle-click drag | Pan |
| Ctrl + wheel | Smooth zoom |

If the window looks empty, open the **Cell** panel on the left and double-click `half_adder` (the top cell).

### What the layout actually contains

Cells panel confirmed synthesis mapped the RTL exactly as expected:

```
half_adder
├── sky130_fd_sc_hd__xor2_2      ← from  sum = a ^ b
├── sky130_fd_sc_hd__and2_1      ← from  carry = a & b
├── sky130_fd_sc_hd__buf_2       ← inserted buffer
├── sky130_fd_sc_hd__decap_...   ← decoupling caps
├── sky130_fd_sc_hd__fill_...    ← filler cells
└── sky130_fd_sc_hd__tapvpwrvgnd_1  ← tap cells
```

**Note on visual appearance:** because `DIE_AREA` was deliberately oversized (300×300 µm) to fix the PDN issue, the actual logic (2 real gates) occupies a tiny fraction of the die. At first glance in KLayout it just looks like a large green/patterned square — that's the top metal + fill layers dominating the view, not a bug. Zooming into the center and/or hiding fill/decap layers reveals the actual XOR2/AND2 cells.

---

## 13. View the Layout in Magic

```bash
magic ~/vlsi/projects/half_adder/runs/RUN_2026-07-09_20-55-31/final/gds/half_adder.gds
```

or with a lighter rendering mode:

```bash
magic -d XR ~/vlsi/projects/half_adder/runs/RUN_2026-07-09_20-55-31/final/gds/half_adder.gds
```

---

## Common Errors — Quick Reference

### `bash: cd: too many arguments`
**Cause:** leftover OpenLane 1 alias (`alias openlane="cd ~/OpenLane"`).
**Fix:** `unalias openlane`, then remove the line from `~/.bashrc`.

### `Error: No such option: --design`
**Cause:** using OpenLane 1 CLI syntax on OpenLane 2.
**Fix:** use `openlane --design-dir . config.json`. Always cross-check with `openlane --help` first when unsure of the installed version's syntax.

### `[PDN-0185] Insufficient width to add straps`
**Cause:** design/die too small for the default PDN strap template.
**Fix:** set `FP_SIZING: "absolute"` with an oversized `DIE_AREA` / `CORE_AREA` (e.g. `[0,0,300,300]` / `[20,20,280,280]` for a tiny design like this).

### Can't find the design's GDS among thousands of files
**Cause:** blind `find ~ -name "*.gds"` also matches every PDK standard-cell GDS.
**Fix:** search specifically inside the project's own `runs/*/final/gds/` directory.

### `STA-0366 port '__VIRTUAL_CLK__' not found` / virtual clock warnings
**Cause:** no `CLOCK_PORT` defined for a purely combinational design — OpenROAD falls back to a virtual clock for timing analysis.
**Verdict:** harmless warning for a combinational-only design like this half adder; not something to chase.

---

## Final Working `config.json`

```json
{
    "DESIGN_NAME": "half_adder",
    "VERILOG_FILES": [
        "dir::src/half_adder.v"
    ],
    "CLOCK_PORT": null,
    "FP_SIZING": "absolute",
    "DIE_AREA": [0, 0, 300, 300],
    "CORE_AREA": [20, 20, 280, 280]
}
```

Run with:

```bash
openlane --design-dir . config.json
```

---

## What I Learned From This Run

- How to install OpenLane 2 via Nix, with the Sky130 PDK auto-pulled through Volare
- OpenLane 2's CLI (`--design-dir`) differs from OpenLane 1 (`--design`) — always verify with `--help`
- Tiny combinational designs need an explicit, oversized `DIE_AREA`/`CORE_AREA` or PDN generation fails
- How to isolate the actual chip GDS from thousands of PDK library GDS files
- How to read the Cells panel in KLayout to confirm synthesis mapped RTL correctly (`^` → XOR2, `&` → AND2)
- Full RTL → GDS flow: Yosys (synthesis) → OpenROAD (floorplan/placement/CTS/routing) → Magic (DRC) → Netgen (LVS) → GDSII → KLayout (view)

---

## Next Projects (Same Repo, Same Setup)

- [ ] Inverter
- [ ] NAND gate
- [ ] XOR gate (standalone)
- [x] Half Adder
- [ ] Full Adder
- [ ] Ripple Carry Adder
- [ ] 4-bit ALU
- [ ] RISC-V core

---

## References

- https://github.com/efabless/openlane2
- https://github.com/The-OpenROAD-Project/OpenROAD
- https://github.com/google/skywater-pdk
- https://github.com/RTimothyEdwards/magic
- https://www.klayout.de

---

## Author

**Ebinesh K**
