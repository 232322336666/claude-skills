---
name: remote-fm-formality
description: Use when running Synopsys Formality (FM) formal verification to check RTL vs gate-level netlist equivalence on a remote Linux server. Covers project setup, SVF integration, execution, and result analysis.
---

# Remote Formality Flow (Synopsys Formality)

## Overview

Run formal equivalence verification (FEV) using Synopsys Formality on a remote Linux server. Verifies that DC synthesis output (gate-level netlist) is functionally equivalent to the original RTL, using SVF guidance from Design Compiler.

## When to Use

- After DC synthesis completes — verify RTL-to-gate equivalence
- After any netlist transformation (DFT insertion, ECO, etc.)
- Before tape-out signoff
- Debugging synthesis mismatches

## Prerequisites

- DC synthesis completed with SVF file generated (`dc/svf/*.svf`)
- Gate-level netlist available (`dc/gate/*_netlist.v`)
- RTL source files available
- Remote server has Formality license (`fm_shell`)

## Project Directory Template

```
fm/
├── .synopsys_fm.setup    # FM environment config (library, paths, variables)
├── Makefile              # Run/clean commands
├── scr/
│   └── fm.tcl            # Main FM verification script
├── rpt/                  # Reports output
├── log/                  # Log files
├── fss/                  # Failed session (for debugging)
└── formality_svf/        # SVF produced by FM
```

## Key Files to Modify Per Design

### `.synopsys_fm.setup` — Environment & Variables

```tcl
set synopsys_auto_setup true    # Auto-apply SVF guidance, critical for matching

# Clock gate handling
set verification_clock_gate_hold_mode COLLAPSE_ALL_CG_CELLS
set verification_clock_gate_edge_analysis true

# Library setup (same as DC)
set search_path [list "<std_cell_lib_path>"]
set lib_files [list tcbn65lptc_ccs.db tcbn65lptc1d21d2_ccs.db]

# === MUST CHANGE per design ===
set TOP_DESIGN_NAME "your_module_name"

set SVF "../dc/svf/${TOP_DESIGN_NAME}.svf"           # From DC synthesis
set RTL_PATH "../dc/rtl"                               # RTL source path
set GATE_NETLIST "../dc/gate/${TOP_DESIGN_NAME}_netlist.v"  # DC output

set RPT_UNMATCHED "./rpt/${TOP_DESIGN_NAME}_unmatched_points.rpt"
set RPT_MATCHED "./rpt/${TOP_DESIGN_NAME}_matched_points.rpt"
set RPT_FAILING "./rpt/${TOP_DESIGN_NAME}_verify_failing.rpt"
set RPT_PASSING "./rpt/${TOP_DESIGN_NAME}_verify_passing.rpt"
set SESSION "./fss/verify_failed.fss"
```

### `scr/fm.tcl` — Verification Script (rarely modified)

```
1. set_svf         → Load SVF guidance from DC
2. read_db         → Load standard cell libraries
3. read_verilog r  → Read RTL as reference design
4. set_top r       → Set reference top module
5. read_verilog i  → Read gate netlist as implementation
6. set_top i       → Set implementation top module
7. match           → Match compare points between ref and impl
8. verify          → Run formal equivalence check
9. report          → Generate pass/fail reports
```

## Workflow Commands

```bash
# Run FM verification
ssh user@server "cd /project/fm && fm_shell -f ./scr/fm.tcl | tee ./log/fm.log"

# Or use Makefile
ssh user@server "cd /project/fm && make run"

# Check results
ssh user@server "cat /project/fm/rpt/*_unmatched_points.rpt"

# Clean up
ssh user@server "cd /project/fm && make clean"
```

## Reading Verification Results

### Key line in log — PASS

```
********************************* Verification Results *********************************
Verification SUCCEEDED
 33 Passing compare points
----------------------------------------------------------------------------------------
Passing (equivalent)    0    0    0    0    17    16    0    33
Failing (not equivalent) 0    0    0    0     0     0    0     0
```

### Key line in log — FAIL

```
Verification FAILED
```

### Matching Results

```
33 Compare points matched by name        ← Good: all points matched
0 Unmatched reference compare points     ← Good: no orphan ref points
1 Unmatched implementation compare points ← May be OK (e.g., clock gate)
```

### Understanding Compare Points

| Type | Meaning |
|------|---------|
| Port | I/O ports |
| DFF | D flip-flops |
| LAT | Latches (often from clock gating) |
| BBPin | Black-box pins |

## Common Issues & Fixes

| Issue | Cause | Fix |
|-------|-------|-----|
| `Verification FAILED` | Netlist not equivalent to RTL | Check `rpt/*_unmatched_points.rpt`, analyze failing points |
| Many unmatched points | SVF not loaded or wrong SVF | Ensure SVF path is correct and DC generated it |
| `Cannot find design` | `TOP_DESIGN_NAME` mismatch | Match module name in RTL and netlist |
| Clock-gate LAT unmatched | DC inserted ICG cell not in RTL | Normal — SVF `environment` commands handle this |
| `synopsys_auto_setup` not set | Missing auto setup | Keep `set synopsys_auto_setup true` in setup file |
| `read_db` fails | Wrong library path | Ensure same std cell lib as DC synthesis |
| Stack size warning | ulimit too low | Increase stack limit: `ulimit -s 65536` |

## Verification Flow Diagram

```
DC Synthesis Output
  │
  ├── RTL (.v)          ──→  Reference Design (r:)
  │                           │
  ├── Gate Netlist (.v) ──→  Implementation Design (i:)
  │                           │
  └── SVF (.svf)        ──→  Guidance (name changes, optimizations)
                              │
                              ▼
                         ┌──────────┐
                         │  match   │  Match compare points
                         └────┬─────┘
                              ▼
                         ┌──────────┐
                         │ verify   │  Formal equivalence check
                         └────┬─────┘
                              ▼
                     ┌────────┴────────┐
                     │                 │
                 SUCCEEDED          FAILED
                     │                 │
                 Done ✓          Debug with:
                                - rpt/*_unmatched
                                - rpt/*_failing
                                - fss/ session
```

## Relationship with DC Flow

```
RTL (.v)
  │
  ▼
┌──────────────┐
│  DC Synthesis │  → SVF + Gate Netlist + DDC
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Formality    │  RTL vs Gate Netlist equivalence check
└──────┬───────┘
       │
       ▼
   PASS → proceed to P&R (ICC/Innovus)
   FAIL → debug and fix synthesis
```

## Tips

1. **SVF is critical** — Without it, clock gating and name changes cause unmatched points and verification failures
2. **`synopsys_auto_setup true`** — Always enable this, it automatically configures optimal settings
3. **Clock-gate LAT unmatched is normal** — DC inserts ICG cells that don't exist in RTL, SVF handles the mapping
4. **Failed session (fss/)** — Load in Formality GUI to debug failing points interactively
5. **Run FM after every synthesis change** — Catch equivalence issues early
6. **Library must match DC** — Use the same std cell library and search path as DC synthesis
