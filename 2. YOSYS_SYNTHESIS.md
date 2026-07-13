# Understanding Yosys Synthesis — From RTL to Standard Cells

Personal deep-dive log on how **Yosys** (the synthesis tool inside OpenLane 2) turns Verilog RTL into a gate-level netlist, using a Half Adder as the working example.

This picks up *after* the RTL → GDS flow was already working end-to-end (see the main `OpenLane2-Sky130` README). This document is about stopping to actually understand the **synthesis stage** instead of treating OpenLane as a black box — including the mistakes made along the way.

---

## Why This Document Exists

The first pass through OpenLane looked like this:

```
RTL
 │
 ▼
OpenLane
 │
 ▼
GDS
```

That works, but it teaches you nothing about *what happened in between*. This log is about opening up one single stage — **Yosys synthesis** — and inspecting it manually, command by command, the way a physical design engineer actually reads a flow instead of just trusting it.

---

## Prerequisites

Everything here assumes:
- The Half Adder project already exists at `~/vlsi/projects/half_adder`
- `config.json` and `src/half_adder.v` are in place (see main setup README)
- You're inside the OpenLane Nix shell:

```bash
cd ~/vlsi/openlane2
nix-shell
```

- The design was already run at least once, so a `runs/` directory exists with a `06-yosys-synthesis` stage inside it.

---

## Part 1 — Functional Verification Before Synthesis

Before touching synthesis at all, the RTL was simulated first. This matters because **synthesizing unverified RTL is a bad habit** — if the logic is wrong, synthesis will happily generate a wrong chip just as efficiently as a correct one.

### Testbench

`half_adder_tb.v` was written to drive every input combination and check `sum`/`carry` against expected values.

### Simulate with Icarus Verilog + GTKWave

```bash
iverilog -o half_adder_tb half_adder.v half_adder_tb.v
vvp half_adder_tb
gtkwave dump.vcd
```

Flow used:

```
RTL
 │
 ▼
iverilog   (compile)
 │
 ▼
vvp        (simulate)
 │
 ▼
GTKWave    (view waveform)
```

Confirmed `sum` and `carry` were correct for all 4 input combinations before moving on. This established the mindset for the rest of the project:

```
RTL → Simulation → Lint → Synthesis
```

not

```
RTL → Synthesis (and hope)
```

---

## Part 2 — Exploring Yosys Manually

Instead of letting OpenLane run Yosys silently in the background, Yosys was opened directly and driven interactively, one command at a time, to see what each step actually does to the design.

```bash
yosys
```

This drops you into the **Yosys interactive shell** — a completely separate environment from your normal Linux terminal. This distinction caused the biggest recurring mistake of this session (see Part 5).

### Step 1 — `read_verilog`

```
yosys> read_verilog half_adder.v
```

**What it does:** parses the Verilog source into Yosys's internal representation, called **RTLIL** (RTL Intermediate Language).

**Key takeaway:** this is *not* hardware yet. It's just a parsed, structured representation of the Verilog text.

```
Verilog
   │
   ▼
RTLIL
```

### Step 2 — `hierarchy`

```
yosys> hierarchy -check -top half_adder
```

**What it does:**
- Identifies the top-level module (`half_adder`)
- Checks the module hierarchy
- Flags/removes unused modules

For a single-module design like this half adder, the hierarchy is trivial — but this step is essential for larger SoC designs with many nested modules.

### Step 3 — `dump`

```
yosys> dump
```

**What it does:** prints the current internal RTLIL representation to the screen.

**This was the first major insight of the whole exercise.** The dump showed generic cells:

```
$xor
$and
```

instead of the Sky130-specific cells expected at the end:

```
sky130_fd_sc_hd__xor2_1
sky130_fd_sc_hd__and2_1
```

**Takeaway:** Yosys builds **generic, technology-independent logic first**. The mapping to real, physical Sky130 standard cells happens *later*, in a separate technology-mapping step. Synthesis is not one atomic operation — it's a pipeline.

### Step 4 — `proc`

```
yosys> proc
```

**What it does:** converts behavioral Verilog constructs — `always` blocks, `if`, `case` — into structural logic (muxes, latches, etc.).

**Observation for this design specifically:** the half adder was written using pure continuous assignments (`assign sum = a ^ b; assign carry = a & b;`), not `always` blocks. So `proc` had almost nothing to do here. This step would matter a lot more for a sequential or `case`-based design.

### Step 5 — `opt`

```
yosys> opt
```

**What it does:** runs optimization passes, including:
- Constant propagation
- Dead logic removal
- Unused wire removal
- Logic simplification

**Observation:** since the half adder RTL was already minimal (2 gates), `opt` found essentially nothing to simplify. This step earns its keep on larger, messier designs where redundant logic tends to creep in.

---

## Part 3 — Technology Mapping: Generic Logic → Real Silicon Cells

This is the conceptual center of the whole exercise — the point where an abstract logic equation becomes a specific, fabricable object in the Sky130 process.

```
assign a ^ b
    │
    ▼
$xor                          ← generic, technology-independent
    │
    ▼
sky130_fd_sc_hd__xor2_1        ← real physical standard cell
```

The full picture:

```
Verilog RTL
     │
     ▼
Generic Logic ($and, $xor, $mux, ...)
     │
     ▼
Technology Mapping   (using the Liberty file as the cell library)
     │
     ▼
Physical Standard Cells (sky130_fd_sc_hd__*)
```

### The Liberty file

The Sky130 standard cell library was located and opened directly:

```bash
find ~ -name "sky130_fd_sc_hd__tt_025C_1v80.lib"
```

```bash
less ~/.volare/volare/sky130/versions/<hash>/sky130A/libs.ref/sky130_fd_sc_hd/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```

**What a `.lib` file contains, per cell:**
- Area
- Leakage power
- Timing arcs
- Pin definitions
- Boolean function (the cell's logical behavior)
- Power pin definitions

Example entry structure:

```liberty
cell (sky130_fd_sc_hd__xor2_1) {
    area : ...
    pin (A)  { ... }
    pin (B)  { ... }
    pin (X)  { function : "(A^B)"; ... }
}
```

This was the first time the design's logic (`a ^ b`) was seen matched, verbatim, against a real cell's Boolean function in the library — making the abstract "technology mapping" concept concrete.

### PVT Corners

While reading the library filename (`tt_025C_1v80`), the naming convention was decoded:

| Component | Meaning |
|---|---|
| `tt` | Process corner — **T**ypical-**T**ypical |
| `025C` | Temperature — 25°C |
| `1v80` | Voltage — 1.80V |

Other common corners: `ff` (Fast-Fast), `ss` (Slow-Slow) — used for timing signoff across best-case and worst-case silicon/voltage/temperature conditions. Only the `tt` corner was used at this stage; full multi-corner signoff comes later.

---

## Part 4 — Inspecting OpenLane's Own Synthesis Output

Rather than treating OpenLane's Yosys run as a black box, its actual output directory was opened directly:

```bash
cd ~/vlsi/projects/half_adder/runs/RUN_2026-07-10_08-46-18/06-yosys-synthesis
ls
```

Output:

```
AREA_0.abc            hierarchy.dot          synthesis.abc.sdc
COMMANDS              primitive_techmap.dot  synthesized.dot
config.json            reports                synthesized.svg
extra.json             runtime.txt            yosys-synthesis.log
half_adder.nl.v        state_in.json          yosys-synthesis.process_stats.json
half_adder.nl.v.json   state_out.json
```

Files of interest:

| File | What it is |
|---|---|
| `half_adder.nl.v` | The synthesized gate-level netlist (real Verilog, but built from standard cells instead of behavioral logic) |
| `hierarchy.dot` | Graphviz source for the module hierarchy — most useful on large multi-module SoCs |
| `primitive_techmap.dot` | Graphviz source for the generic logic graph, *before* technology mapping |
| `synthesized.dot` / `synthesized.svg` | Graphviz/SVG of the final mapped netlist — the actual gate-level schematic |
| `reports/` | Synthesis statistics — cell counts, area, etc. |
| `yosys-synthesis.log` | Full log of every Yosys command OpenLane ran internally |

Three different views of the same design, in order of how the flow actually reasons about it:

```
hierarchy.dot          → module hierarchy (structural, high level)
primitive_techmap.dot  → generic logic (post read/proc/opt, pre-mapping)
synthesized.svg         → final mapped netlist (real standard cells)
```

---

## Part 5 — ⚠️ Mistake: Running Linux Commands Inside the Yosys Shell

This was the recurring error of the session, and worth documenting clearly because it's an easy trap for a beginner.

### What happened

After generating the schematic:

```
yosys> show -format svg
```

the instinct was to immediately view it with:

```
xdg-open synthesized.svg
```

**But this was typed while still inside the interactive Yosys shell**, not the normal Linux terminal. `xdg-open` is a **Linux/desktop command**, not a Yosys command — Yosys has no idea what it means and it does nothing useful (or errors, depending on context).

### The fix

**Exit Yosys first:**

```
yosys> exit
```

or

```
Ctrl + D
```

This returns you to the actual Linux shell prompt, recognizable by the normal terminal prompt instead of `yosys>`:

```
[nix-shell:~/vlsi/projects/half_adder/runs/RUN_2026-07-10_08-46-18/06-yosys-synthesis]$
```

**Then** confirm the file actually exists:

```bash
ls
```

Expected to see:

```
synthesized.dot
synthesized.svg
```

**Then** open it from the real shell:

```bash
xdg-open synthesized.svg
```

If `xdg-open` isn't available or doesn't do anything, fall back to a browser directly:

```bash
firefox synthesized.svg
```

or

```bash
google-chrome synthesized.svg
```

To check whether `xdg-open` is even installed:

```bash
which xdg-open
```

Expected output if present:

```
/usr/bin/xdg-open
```

On a GUI desktop, double-clicking `synthesized.svg` in the file manager works too — no terminal needed at that point.

### The lesson, generalized

**Two different shells were involved, and it's easy to lose track of which one you're in:**

```
Linux terminal (bash, inside nix-shell)
   │
   │  yosys
   ▼
Yosys interactive shell (yosys>)
   │
   │  exit  /  Ctrl+D
   ▼
Linux terminal (bash, inside nix-shell)   ← back here to run xdg-open, firefox, ls, etc.
```

Any command that isn't a Yosys command (`read_verilog`, `hierarchy`, `proc`, `opt`, `dump`, `show`, `write_verilog`, `exit`, etc.) needs to be run **outside** the `yosys>` prompt.

---

## Command Reference — What Was Actually Run

For quick copy-paste next time:

```bash
# --- Linux shell, inside nix-shell ---
cd ~/vlsi/projects/half_adder/runs/<RUN_ID>/06-yosys-synthesis
yosys
```

```
# --- Yosys interactive shell ---
read_verilog half_adder.v
hierarchy -check -top half_adder
dump
proc
opt
show -format svg
exit
```

```bash
# --- back in Linux shell ---
ls
xdg-open synthesized.svg
```

---

## Common Errors — Quick Reference

### Typing `xdg-open` (or any Linux command) and nothing happens / it errors
**Cause:** still inside the `yosys>` interactive shell.
**Fix:** `exit` or `Ctrl+D` to return to the real terminal first, then run the command.

### `dump` shows `$xor` / `$and` instead of `sky130_fd_sc_hd__xor2_1`
**Cause:** this is expected, not a bug. Yosys builds generic, technology-independent logic first (`$xor`, `$and`, `$mux`, ...) and only maps it to real Sky130 standard cells in a later technology-mapping pass.
**Verdict:** confirms synthesis is working correctly at that stage.

### Can't find the `.lib` file
**Fix:** search under the Volare-managed PDK install:

```bash
find ~/.volare -name "*tt_025C_1v80.lib"
```

### Confusing which `.dot`/`.svg` file to look at
**Fix:** remember the order —
`hierarchy.dot` (structure) → `primitive_techmap.dot` (generic logic) → `synthesized.svg` (final real netlist).

---

## Key Concepts Learned

| Term | Meaning |
|---|---|
| **RTL** | Behavioral description of hardware (the Verilog you write) |
| **RTLIL** | Yosys's internal intermediate representation of the design |
| **Generic cells** (`$and`, `$xor`, `$mux`) | Technology-independent logic, before mapping |
| **Standard cells** (`sky130_fd_sc_hd__*`) | Technology-dependent, real physical cells from the Sky130 library |
| **Liberty (`.lib`)** | File defining timing, power, area, and logical function for every standard cell |
| **PVT corner** | Process / Voltage / Temperature condition a `.lib` file is characterized for (e.g. `tt`, `ff`, `ss`) |
| **Technology mapping** | The step where generic logic is matched against real standard cells using the Liberty file |
| **OpenLane** | Not an EDA tool itself — an automation framework that orchestrates Yosys → OpenROAD → Magic → Netgen → KLayout |

---

## Files Produced by This Stage

```
half_adder.v            # original RTL
half_adder_tb.v          # testbench used for pre-synthesis simulation
config.json               # OpenLane config
half_adder.nl.v          # synthesized gate-level netlist
synthesized.svg           # visual schematic of the mapped netlist
hierarchy.dot             # module hierarchy graph
primitive_techmap.dot     # generic logic graph (pre-mapping)
```

---

## The Mindset Shift

**Before this exercise**, the flow looked like a single opaque step:

```
RTL
 │
 ▼
OpenLane
 │
 ▼
GDS
```

**After manually walking through Yosys**, the real flow is understood as:

```
RTL
 │
 ▼
Simulation
 │
 ▼
Lint
 │
 ▼
Synthesis (read_verilog → hierarchy → proc → opt → techmap)
 │
 ▼
Floorplan
 │
 ▼
PDN
 │
 ▼
Placement
 │
 ▼
CTS
 │
 ▼
Routing
 │
 ▼
DRC
 │
 ▼
LVS
 │
 ▼
GDS
```

This is the actual ASIC implementation flow — and from here on, each stage gets the same treatment: open the intermediate files OpenLane generates and understand *why* they look the way they do, instead of trusting the automation blindly.

---

## What Comes Next

With synthesis understood at the command level, the next stages to dig into the same way:

- [ ] **Floorplanning** — die/core area, utilization, standard cell rows, tap cells, endcaps
- [ ] **PDN** — power rings, straps, rails, VDD/VSS distribution
- [ ] **Placement** — cell placement, legalization, congestion
- [ ] **CTS** — clock tree synthesis (needs a sequential design to be meaningful)
- [ ] **Routing** — global and detailed routing, vias, metal layer usage
- [ ] **Signoff** — timing (STA), DRC, LVS, and final GDS export

---
