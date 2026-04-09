---
name: remote-fm-formality
description: Use when running Synopsys Formality (FM) formal verification on a remote Linux server. First copy from example template, then modify per design. Covers project setup, SVF integration, execution, and result analysis.
---

# Remote Formality Flow (Synopsys Formality)

## Overview

Run formal equivalence verification (FEV) using Synopsys Formality on a remote Linux server. Verifies that DC synthesis output (gate-level netlist) is functionally equivalent to the original RTL, using SVF guidance from Design Compiler. **Always copy from the verified example template first**.

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

## CRITICAL: Always Copy from Example Template

**The example template at `/home/research/zhengnafu2/agent_digits/example/fm/` is a verified, working reference. Always copy from it to avoid syntax errors (especially `set_top` format).**

### Step 1: Copy template structure

```bash
# On remote server — copy entire fm directory structure
ssh user@server "cd /project && cp -r /home/research/zhengnafu2/agent_digits/example/fm/ ./fm_new && cd fm_new && rm -rf rpt/* log/* fss/* FM_INFO* formality_svf/* formality*.log fm_shell_command.log"
```

### Step 2: Modify only 1 thing in `.synopsys_fm.setup`

```bash
ssh user@server "sed -i 's/shift_reg_16bit/YOUR_MODULE_NAME/g' /project/fm_new/.synopsys_fm.setup"
```

Only change `TOP_DESIGN_NAME`. Library paths and `set_top` syntax are pre-configured.

### Step 3: Run

```bash
ssh user@server "cd /project/fm_new && make run"
```

## Project Directory Structure

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

## Verified Template Files (from example)

### `.synopsys_fm.setup` — Complete verified template

```tcl
set synopsys_auto_setup true

set verification_clock_gate_hold_mode COLLAPSE_ALL_CG_CELLS

set verification_clock_gate_edge_analysis true

set verification_set_undriven_signals 0

set verification_failing_point_limit 0

set hdlin_dwroot /opt/synopsys/syn/V-2023.12

#**************************************** Library setup ***************************************************
set search_path [list "/home/research/yeke22/STDCELL/tcbn65lp_220a/A100001_20180209/TSMCHOME/digital/Front_End/timing_power_noise/CCS/tcbn65lp_200a/"  \
		]

set lib_files [list 	tcbn65lptc_ccs.db 		\
                        tcbn65lptc1d21d2_ccs.db     \
		]
#**************************************** Library setup ***************************************************

#**************************************** Set Alias *******************************************************

# *** CHANGE THIS per design ***
set TOP_DESIGN_NAME "YOUR_MODULE_NAME"

set SVF "../dc/svf/${TOP_DESIGN_NAME}.svf"
set RTL_PATH "../dc/rtl"

set GATE_NETLIST "../dc/gate/${TOP_DESIGN_NAME}_netlist.v"

set search_path [concat ${RTL_PATH}\
                        ${search_path}]

set RPT_UNMATCHED "./rpt/${TOP_DESIGN_NAME}_unmatched_points.rpt"
set RPT_MATCHED "./rpt/${TOP_DESIGN_NAME}_matched_points.rpt"
set RPT_FAILING "./rpt/${TOP_DESIGN_NAME}_verify_failing.rpt"
set RPT_PASSING "./rpt/${TOP_DESIGN_NAME}_verify_passing.rpt"

set SESSION "./fss/verify_failed.fss"
#**************************************** Set Alias *******************************************************
```

### `scr/fm.tcl` — Complete verified template

**IMPORTANT: This script uses the exact `set_top` syntax that works with Formality V-2023.12. Do NOT modify `set_top` format.**

```tcl
sh date

set_svf ${SVF}

foreach lib_file ${lib_files} {
  set fm_shell_status [read_db -technology_library ${lib_file}]
  if { $fm_shell_status == 0 } {
    exit
  }
}

set hostFiles   [split [eval \ls -1 \
${RTL_PATH}/*.v \
]]

foreach filename ${hostFiles} {
  set fm_shell_status [read_verilog -container r -libname WORK ${filename}]
  if { $fm_shell_status == 0 } {
    exit
  }
}
set_top r:/WORK/${TOP_DESIGN_NAME}

read_verilog -container i -libname WORK ${GATE_NETLIST}
set_top i:/WORK/${TOP_DESIGN_NAME}

match
report_unmatched_points > $RPT_UNMATCHED
report_matched_points > $RPT_MATCHED

if {[verify]!=1} {
  analyze_points -all > "./log/analyze_points.log"
  report_failing > $RPT_FAILING
  report_passing > $RPT_PASSING
  save_session -replace $SESSION
}

sh date
exit
```

### `Makefile` — Verified template

```makefile
run:
	@fm_shell -f ./scr/fm.tcl | tee ./log/fm.log

clean:
	@rm -rf *.log *.lck formality*_svf FM_WORK*
```

## Key Syntax Notes (learned from errors)

### `set_top` format — CRITICAL

Formality requires this exact format:

```tcl
# CORRECT — uses /WORK/ prefix
set_top r:/WORK/${TOP_DESIGN_NAME}
set_top i:/WORK/${TOP_DESIGN_NAME}

# WRONG — will cause CMD-012 error
set_top r:/${TOP_DESIGN_NAME}
set_top r:/ -design ${TOP_DESIGN_NAME}
```

### RTL reading — use auto-discovery

The verified script uses `ls` to auto-discover all `.v` files in `RTL_PATH`, so you don't need to list individual files:

```tcl
# CORRECT — auto-discover all .v files
set hostFiles [split [eval \ls -1 ${RTL_PATH}/*.v]]
foreach filename ${hostFiles} {
  read_verilog -container r -libname WORK ${filename}
}

# AVOID — fragile, easy to miss files
read_verilog -r [list ${RTL_PATH}/a.v ${RTL_PATH}/b.v ...]
```

### Error handling — check return status

The verified script checks `fm_shell_status` after each `read_db` and `read_verilog` call and exits on failure. This prevents cascading errors.

## What to Modify Per Design

| File | What to change | How |
|------|---------------|-----|
| `.synopsys_fm.setup` | `TOP_DESIGN_NAME` | Replace with your DC top module name |
| That's it! | | |

**Do NOT modify:** `scr/fm.tcl`, library paths, `Makefile` — these are pre-verified.

The fm.tcl script auto-discovers all RTL files in `../dc/rtl/` and reads the gate netlist from `../dc/gate/`, so no file list modifications needed.

## Workflow Commands

```bash
# Run FM verification
ssh user@server "cd /project/fm && make run"

# Check results — look for "Verification SUCCEEDED" or "Verification FAILED"
ssh user@server "grep -A5 'Verification' /project/fm/log/fm.log"

# Check unmatched points (if any)
ssh user@server "cat /project/fm/rpt/*_unmatched_points.rpt"

# Clean up
ssh user@server "cd /project/fm && make clean"
```

## Reading Verification Results

### PASS

```
********************************* Verification Results *********************************
Verification SUCCEEDED
 159 Passing compare points
----------------------------------------------------------------------------------------
Passing (equivalent)    0    0    0    0    18    141       0     159
Failing (not equivalent) 0    0    0    0      0      0       0       0
```

### Matching Results

```
159 Compare points matched by name        ← Good: all points matched
0(27) Unmatched ref(impl) compare points  ← 27 clock-gate LAT = normal
```

### Understanding Compare Points

| Type | Meaning |
|------|---------|
| Port | I/O ports |
| DFF | D flip-flops |
| LAT | Latches (from clock gating — unmatched is normal) |
| BBPin | Black-box pins |

### FAIL

```
Verification FAILED
```

Check `rpt/*_unmatched_points.rpt` and `rpt/*_verify_failing.rpt` for details.

## Common Issues & Fixes

| Issue | Cause | Fix |
|-------|-------|-----|
| `CMD-012 extra positional option` | Wrong `set_top` syntax | Use `set_top r:/WORK/${TOP_DESIGN_NAME}` (with /WORK/) |
| `CMD-010 unknown option '-design'` | Wrong `set_top` syntax | Use `/WORK/` path, not `-design` flag |
| `Verification FAILED` | Netlist not equivalent to RTL | Check `rpt/*_failing.rpt`, analyze failing points |
| Many unmatched points | SVF not loaded or wrong SVF | Ensure SVF path is correct and DC generated it |
| `Cannot find design` | `TOP_DESIGN_NAME` mismatch | Match module name in RTL and netlist |
| Clock-gate LAT unmatched | DC inserted ICG cell not in RTL | Normal — SVF `environment` commands handle this |
| `read_db` fails | Wrong library path | Ensure same std cell lib as DC synthesis |
| Stack size warning | ulimit too low | Usually harmless, but can add `ulimit -s 65536` before fm_shell |

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

1. **Always copy from example template** — the `set_top` syntax is tricky and template avoids errors
2. **Only change `TOP_DESIGN_NAME`** — one `sed` command and you're done
3. **SVF is critical** — Without it, clock gating and name changes cause unmatched points and verification failures
4. **`synopsys_auto_setup true`** — Always enabled in template, auto-configures optimal settings
5. **Clock-gate LAT unmatched is normal** — DC inserts ICG cells that don't exist in RTL
6. **Run FM after every synthesis change** — Catch equivalence issues early
7. **Library must match DC** — Template uses same library paths as DC template
