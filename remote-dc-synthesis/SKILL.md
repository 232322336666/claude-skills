---
name: remote-dc-synthesis
description: Use when running Synopsys Design Compiler (DC) synthesis on a remote Linux server from a local machine. Covers project setup, constraint writing, synthesis execution, and result analysis.
---

# Remote DC Synthesis Flow (Synopsys Design Compiler)

## Overview

Run RTL synthesis using Synopsys Design Compiler on a remote Linux server (e.g., university HPC) with TSMC 65nm standard cell library. Includes project template setup, constraint configuration, synthesis execution, and report analysis.

## When to Use

- Need to synthesize Verilog RTL into gate-level netlist
- Remote server has DC license and standard cell libraries
- Working from local machine (Windows/macOS) without EDA tools
- Starting a new DC project or adapting the template for a new design

## Prerequisites

**Remote (Linux server):**
- Synopsys Design Compiler installed and licensed (`dc_shell-t`)
- Standard cell library (e.g., TSMC 65nm `tcbn65lp`)
- SSH access configured

**Local (Windows/macOS):**
- SSH client
- (Optional) VS Code for viewing reports

## Project Directory Template

```
dc/
├── .synopsys_dc.setup    # DC environment config (library, paths, variables)
├── Makefile              # Run/clean commands
├── rtl/                  # RTL source files (.v)
├── scr/
│   └── synth_top.tcl     # Main synthesis script
├── cons/
│   └── top_cons.tcl      # Design constraints (clock, I/O, timing)
├── ddc/                  # DDC database (mapped/unmapped)
├── gate/                 # Gate-level netlist output
├── rpt/                  # Reports (area, timing, power, QoR)
├── sdc/                  # SDC output (for P&R)
├── sdf/                  # SDF output (for back-annotation)
├── svf/                  # SVF file (for formal verification)
├── check/                # check_design / check_timing logs
├── log/                  # Synthesis log
└── work/                 # Design library workspace
```

## Key Files to Modify Per Design

When starting a new synthesis project, you need to modify **3 files**:

### 1. `.synopsys_dc.setup` — Environment & Variables

Key variables to change:

```tcl
# Top module name — MUST match your RTL module name
set TOP_DESIGN_NAME "your_module_name"

# RTL source path (relative to dc/ directory)
set RTL_PATH "./rtl"

# Libraries — only change if using a different process node
# Default: TSMC 65nm (tcbn65lp) with TC corner
set target_library $LIB_TC_FILE
```

**Library configuration (usually no change needed):**

| Corner | Voltage | Temp | File | Usage |
|--------|---------|------|------|-------|
| WC (worst) | 1.08V | 125C | `tcbn65lpwc_ccs.db` | Setup analysis |
| TC (typical) | 1.2V | 25C | `tcbn65lptc_ccs.db` | Synthesis target |
| BC (best) | 1.32V | -40C | `tcbn65lplt_ccs.db` | Hold analysis |

The setup file also sets `min_library` mapping: TC (target) paired with BC (min) for OCV analysis.

### 2. `cons/top_cons.tcl` — Design Constraints

Write constraints matching your actual RTL ports:

```tcl
# Clock definition
set clk_period 1000    # 1MHz = 1000ns; adjust for your target frequency
create_clock -name Clk -period $clk_period [get_ports clk]
set_dont_touch_network {clk}

# Virtual clock for I/O delay calculation
create_clock -name v_Clk -period $clk_period

# Input delays (signals arriving from outside)
set_input_delay  [expr $clk_period * 0.1] -clock v_Clk [get_ports data_in]
set_output_delay [expr $clk_period * 0.1] -clock v_Clk [get_ports data_out[*]]

# Asynchronous reset — no timing check
set_false_path -from [get_ports rst_n]
set_dont_touch_network [get_ports rst_n]
set_drive 0 [get_ports rst_n]
set_ideal_network -no_propagate rst_n

# Output load (in pF)
set_load [expr {100/1000}] [get_ports data_out[*]]

# Design-level constraints
set_max_fanout 24 [current_design]
set_max_transition 20 [current_design]
```

**Constraint writing checklist:**

| Item | What to check |
|------|--------------|
| Clock port name | Match `clk` in RTL |
| Clock period | Target frequency (ns) |
| All input ports | `set_input_delay` on each |
| All output ports | `set_output_delay` on each |
| Async signals | `set_false_path` (reset, scan_enable, etc.) |
| Output load | Reasonable capacitance for I/O |

### 3. `scr/synth_top.tcl` — Synthesis Script (rarely modified)

The script follows this flow:

```
1. analyze    → Read RTL files
2. elaborate  → Expand design hierarchy
3. link       → Link to library cells
4. check_design → Verify unmapped design
5. Apply constraints (source cons/top_cons.tcl)
6. compile_ultra → Optimize (2 passes with -incremental)
7. optimize_netlist -area → Area optimization
8. Generate reports (QoR, area, timing, power)
9. Write outputs (netlist, SDC, SDF, DDC)
```

**Note:** The `set_clock_gating_style` in the script enables integrated clock gating (ICG). DC automatically inserts clock gate cells when it finds register banks with shared enable signals.

## Workflow Commands

```bash
# Upload design files to server
scp rtl/*.v user@server:/project/dc/rtl/

# Run synthesis
ssh user@server "cd /project/dc && dc_shell-t -f ./scr/synth_top.tcl | tee ./log/synth_top.log"

# Or use Makefile
ssh user@server "cd /project/dc && make run"

# Download results
scp user@server:/project/dc/gate/*_netlist.v ./
scp -r user@server:/project/dc/rpt/ ./

# Clean up (removes all generated files)
ssh user@server "cd /project/dc && make clean"
```

## Reading Synthesis Reports

### QoR Report (`rpt/pre_*.qor`)

Key metrics to check:

```
Timing Path Group 'Clk'
  Critical Path Slack:         899.87    ← Must be >= 0 (MET)
  Total Negative Slack:          0.00    ← Must be 0
  No. of Violating Paths:        0.00    ← Must be 0
  Worst Hold Violation:          0.00    ← Must be 0
```

### Area Report (`rpt/pre_*.area`)

```
Combinational area:         1.08     ← Logic gates
Noncombinational area:    133.20     ← Flip-flops / registers
Total cell area:          134.28     ← Total standard cell area (um²)
```

### Timing Report (`rpt/pre_setup_*.tim`)

Check the critical path:
- `slack (MET)` = positive → setup timing OK
- `slack (VIOLATED)` = negative → need to fix (lower frequency or optimize)

### Power Report (`rpt/pre_*.power`)

```
Total Dynamic Power    = 108.65 nW   ← Switching + Internal
Cell Leakage Power     =   4.60 nW   ← Static power
```

## Common Issues & Fixes

| Issue | Cause | Fix |
|-------|-------|-----|
| `Cannot find design 'xxx' in library 'work'` | `TOP_DESIGN_NAME` doesn't match module name | Set correct name in `.synopsys_dc.setup` |
| `ls: cannot access ../rtl/*.v` | `RTL_PATH` points to wrong directory | Change `../rtl` to `./rtl` in `.synopsys_dc.setup` |
| `Can't find port 'xxx'` | Constraint port name doesn't match RTL | Update `cons/top_cons.tcl` with correct port names |
| Setup violation (negative slack) | Clock period too tight | Increase `clk_period` or optimize design |
| Hold violation | Fast path with no logic | Usually fixed in post-layout; add delay buffers if needed |
| `set_load` on wrong ports | Port names mismatch | Verify port names in RTL module declaration |
| High area | Excessive logic | Check `compile_ultra` flags; enable `-boundary_optimization` |
| No clock gating inserted | No shared enable in register bank | Group registers with common enable signal |

## Synthesis Flow Diagram

```
RTL (.v)
  │
  ▼
┌──────────────┐
│   analyze     │  Parse Verilog
└──────┬───────┘
       ▼
┌──────────────┐
│  elaborate    │  Build design hierarchy
└──────┬───────┘
       ▼
┌──────────────┐
│    link       │  Link to std cell library
└──────┬───────┘
       ▼
┌──────────────┐
│  constraints  │  Clock, I/O timing, false paths
└──────┬───────┘
       ▼
┌──────────────┐
│ compile_ultra │  Logic optimization + mapping (×2)
└──────┬───────┘
       ▼
┌──────────────┐
│optimize_netlist│ Area optimization
└──────┬───────┘
       ▼
┌─────────────────────────────────┐
│ Outputs:                        │
│  gate/*.v    - Gate-level netlist│
│  sdc/*.sdc   - Constraints      │
│  sdf/*.sdf   - Delay annotation │
│  ddc/*.ddc   - DC database      │
│  rpt/*       - Reports          │
└─────────────────────────────────┘
```

## Tips

1. **Always check `check_design` output** before and after synthesis — it catches unmapped cells and connectivity issues
2. **Run `compile_ultra` twice** (with `-incremental`) for better optimization results
3. **Clock gating** is automatic — DC inserts ICG cells when multiple FFs share an enable signal (minimum 4 bits by default)
4. **`set_ideal_network` on reset** means DC won't buffer the reset tree — this is intentional, APR tools handle it later
5. **For first run, use relaxed clock** (e.g., 1MHz) to verify flow correctness, then tighten to target frequency
6. **Gate-level netlist** can be used for post-synthesis simulation with the SDF file
