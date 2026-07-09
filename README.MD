# OpenLane 2 + Sky130 RTL to GDS Flow (Ubuntu 22.04)

This repository demonstrates how to install **OpenLane 2** on **Ubuntu 22.04**, download the **Sky130 PDK**, and successfully generate the **GDSII layout** of a simple **Half Adder**.

This guide is intended for complete beginners who have just installed Ubuntu and want to start learning ASIC Physical Design using open-source tools.

---

# Install Nix

OpenLane 2 officially recommends using **Nix** to manage all tool dependencies.

Install curl

```bash
sudo apt update
sudo apt install -y curl
```

Install Nix

```bash
curl --proto '=https' --tlsv1.2 -sSf -L https://install.determinate.systems/nix | sh -s -- install --no-confirm --extra-conf "
extra-substituters = https://openlane.cachix.org
extra-trusted-public-keys = openlane.cachix.org-1:qqdwh+QMNGmZAuyeQJTH9ErW57OWSvdtuwfBKdS254E=
"
```

Restart the terminal.

Verify installation

```bash
nix --version
```

Example

```
nix (Determinate Nix 3.21.1)
```

Check installation

```bash
nix doctor
```

---

# Create Workspace

```bash
mkdir -p ~/vlsi
cd ~/vlsi
```

---

# Clone OpenLane 2

```bash
git clone https://github.com/efabless/openlane2.git
```

Enter repository

```bash
cd openlane2
```

---

# Enter OpenLane Environment

```bash
nix-shell
```

The first time may take several minutes.

---

# Verify Installation

```bash
type openlane
```

Expected

```
openlane is hashed (/nix/store/.../bin/openlane)
```

Verify important tools

```bash
openroad -version

yosys -V

verilator --version

netgen -batch
```

---

# Remove Old OpenLane Alias (If Needed)

If you previously installed OpenLane 1, you may have an alias like

```bash
alias openlane="cd ~/OpenLane"
```

Check

```bash
type openlane
```

If it says

```
openlane is aliased to ...
```

Remove it temporarily

```bash
unalias openlane
```

Remove permanently

```bash
nano ~/.bashrc
```

Delete

```bash
alias openlane="cd ~/OpenLane"
```

Reload

```bash
source ~/.bashrc
```

---

# Download Sky130 PDK

Simply run

```bash
openlane --smoke-test
```

OpenLane automatically downloads

- Sky130 PDK
- Standard Cell Libraries
- Technology files
- Required databases

---

# Create Project

```bash
mkdir -p ~/vlsi/projects

cd ~/vlsi/projects

mkdir half_adder

cd half_adder

mkdir src
```

Project structure

```
half_adder
│
├── config.json
└── src
    └── half_adder.v
```

---

# Half Adder Verilog

Create

```
src/half_adder.v
```

```verilog
module half_adder (
    input wire a,
    input wire b,
    output wire sum,
    output wire carry
);

assign sum = a ^ b;
assign carry = a & b;

endmodule
```

---

# OpenLane Configuration

Create

```
config.json
```

```json
{
    "DESIGN_NAME": "half_adder",

    "VERILOG_FILES": [
        "dir::src/half_adder.v"
    ],

    "CLOCK_PORT": null,

    "FP_SIZING": "absolute",

    "DIE_AREA": [0,0,300,300],

    "CORE_AREA": [20,20,280,280]
}
```

---

# Run RTL → GDS

```bash
cd ~/vlsi/projects/half_adder

openlane \
    --design-dir . \
    config.json
```

The flow automatically performs

- RTL Parsing
- Linting
- Synthesis
- Floorplanning
- Placement
- Clock Tree Synthesis
- Routing
- DRC
- LVS
- GDS Generation

---

# Output Directory

Everything is generated inside

```
runs/
```

Example

```
runs
└── RUN_2026-07-09_20-55-31
    ├── final
    │   ├── def
    │   ├── gds
    │   ├── lef
    │   └── verilog
    │
    ├── reports
    ├── logs
    └── ...
```

---

# Locate Generated GDS

```bash
find ~/vlsi/projects/half_adder -name "*.gds"
```

Example

```
~/vlsi/projects/half_adder/runs/RUN_xxxxx/final/gds/half_adder.gds
```

---

# View Layout in KLayout

```bash
klayout ~/vlsi/projects/half_adder/runs/RUN_xxxxx/final/gds/half_adder.gds
```

Useful shortcuts

| Key | Function |
|------|----------|
| F | Fit layout |
| Mouse Wheel | Zoom |
| Middle Mouse | Pan |

---

# View Layout in Magic

```bash
magic ~/vlsi/projects/half_adder/runs/RUN_xxxxx/final/gds/half_adder.gds
```

---

# Common Errors

## 1. openlane is aliased

Problem

```
openlane is aliased to ...
```

Solution

```bash
unalias openlane
```

Delete alias from

```
~/.bashrc
```

---

## 2. PDN-0185 Insufficient Width

Problem

```
PDN-0185
Insufficient width to add straps
```

Solution

Increase die size

```json
"FP_SIZING": "absolute",

"DIE_AREA": [0,0,300,300],

"CORE_AREA": [20,20,280,280]
```

---

## 3. No GDS Generated

Search

```bash
find . -name "*.gds"
```

If no file exists, check

```bash
cat runs/*/error.log
```

---

# Complete OpenLane Flow

```
RTL (Verilog)
        │
        ▼
Verilator
        │
        ▼
Yosys
(Synthesis)
        │
        ▼
OpenROAD
(Floorplanning)
        │
        ▼
Placement
        │
        ▼
Clock Tree Synthesis
        │
        ▼
Routing
        │
        ▼
Magic
(DRC)
        │
        ▼
Netgen
(LVS)
        │
        ▼
GDSII
        │
        ▼
KLayout
(View Layout)
```

---

# What You Learn

After completing this project you will know how to

- Install OpenLane 2
- Install Sky130 PDK
- Create a digital design project
- Write RTL
- Configure OpenLane
- Run complete RTL → GDS flow
- Locate generated files
- View GDS in KLayout
- View GDS in Magic

---

# Next Projects

- Inverter
- NAND Gate
- XOR Gate
- Half Adder ✅
- Full Adder
- Ripple Carry Adder
- 4-bit ALU
- RISC-V Core

---

# References

- https://github.com/efabless/openlane2
- https://github.com/The-OpenROAD-Project/OpenROAD
- https://github.com/google/skywater-pdk
- https://github.com/RTimothyEdwards/magic
- https://www.klayout.de

---

## Author

**Ebinesh K**

Learning ASIC Physical Design using OpenLane 2, OpenROAD and Sky130.
