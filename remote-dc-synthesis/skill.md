---
name: remote-dc-synthesis
description: Use when running Synopsys Design Compiler (DC) synthesis on a remote Linux server. First copy from example template, then modify per design. Covers project setup, constraint writing, synthesis execution, and result analysis.
---

# Remote DC Synthesis Flow (Synopsys Design Compiler)

## Overview

Run RTL synthesis using Synopsys Design Compiler on a remote Linux server with TSMC 65nm standard cell library. **Always copy from the verified example template first**, then modify for your design.

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

## CRITICAL: Always Copy from Example Template

**The example template at `~/agent_digits/example/dc/` is a verified, working reference. Always copy from it to avoid syntax errors and configuration mismatches.**

### Step 1: Copy template structure

```bash
# On remote server — copy entire dc directory structure
ssh user@server "cd /project && cp -r ~/agent_digits/example/dc/ ./dc_new && cd dc_new && rm -rf gate/* rpt/* sdc/* sdf/* svf/* check/* log/* ddc/* work/* alib-* *.log default.svf command.log"
```

### Step 2: Upload your RTL files

```bash
scp your_design/*.v user@server:/project/dc_new/rtl/
# Remove example RTL
ssh user@server "rm /project/dc_new/rtl/shift_reg_16bit.v"
```

### Step 3: Modify only 2 things in `.synopsys_dc.setup`

```bash
ssh user@server "sed -i 's/shift_reg_16bit/YOUR_MODULE_NAME/g' /project/dc_new/.synopsys_dc.setup"
```

Only change `TOP_DESIGN_NAME` in `.synopsys_dc.setup`. Library paths, search_path, target_library, link_library are all pre-configured.

### Step 4: Write `cons/top_cons.tcl` for your design

Replace constraint file with your port-specific constraints (see template below).

### Step 5: Run

```bash
ssh user@server "cd /project/dc_new && make run"
```

## Project Directory Structure

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

## Verified Template Files (from example)

### `.synopsys_dc.setup` — Complete verified template

```tcl
#Initialize design, normally no need to change
remove_design -all
set_host_options -max_cores 16

#**************************************** Library Setup ***************************************************
set search_path [list "$TSMC_HOME/digital/Front_End/timing_power_noise/CCS/tcbn65lp_200a/"  \
		"/home/research/$USER/digital/study/dc/rtl/twovth_ctrl.v"  \
    ]

# Define worst case library
## TSMC 1.08V 125degC Slow(WC)
set LIB_WC_FILE	[list 	tcbn65lpwc_ccs.db  		\
                        tcbn65lpwc1d081d08_ccs.db   \
		]

set LIB_WC_NAME   "tcbn65lpwc_ccs.db:tcbn65lpwc_ccs"

# Define typical case library
## TSMC 1.2V 25degC Typ:
set LIB_TC_FILE	[list 	tcbn65lptc_ccs.db 		\
                        tcbn65lptc1d21d2_ccs.db     \
		]

set LIB_TC_NAME   "tcbn65lptc_ccs.db:tcbn65lptc_ccs"

# Define best case library
## TSMC 1.32V -40degC Fast(BC):
set LIB_BC_FILE	[list 	tcbn65lplt_ccs.db 		\
                        tcbn65lplt1d321d32_ccs.lib   \
		]

set LIB_BC_NAME   "tcbn65lplt_ccs.db:tcbn65lplt_ccs"

# Define operating conditions
set LIB_WC_OPCON  "WCCOM"
set LIB_TC_OPCON  "NCCOM"
set LIB_BC_OPCON  "LTCOM"

# Define nand2 gate name for area size calculation
set NAND2_NAME    "ND2D1"

# Set library
set target_library $LIB_TC_FILE
set synthetic_library [list dw_foundation.sldb]
set link_library [concat  [concat  "*" $target_library] $synthetic_library]
set_min_library    tcbn65lptc_ccs.db  -min_version tcbn65lplt_ccs.db
#**************************************** Library Setup ***************************************************

# No need to change
#**************************************** Define Path *****************************************************
sh rm -rf work
sh mkdir work
if { [file exists "./work/USER"] } {
  sh rm -rf ./work/USER/*
} else {
 file mkdir work/USER
}

define_design_lib work -path "./work"
define_design_lib user -path "./work/USER"
#**************************************** Define Path *****************************************************

# No need to change
#**************************************** Void Warning Info ***********************************************
suppress_message VER-130
suppress_message VER-129
suppress_message VER-318
suppress_message VER-936
suppress_message VER-329
suppress_message ELAB-311
#**************************************** Void Warning Info ***********************************************

# *** CHANGE THIS per design ***
set TOP_DESIGN_NAME "YOUR_MODULE_NAME"

set RTL_PATH "./rtl"

set EXCLUDE_FILE "Input_file_that_you_want_to_exclude"

set search_path [concat ${RTL_PATH} \
                        ${search_path}]

set UNMAPPED_CHECK_DESIGN "./check/check_design_${TOP_DESIGN_NAME}_unmapped"
set UNMAPPED_CHECK_TIMING "./check/check_timing_${TOP_DESIGN_NAME}_unmapped"
set UNMAPPED_DDC          "./ddc/${TOP_DESIGN_NAME}_unmapped.ddc"

set MAPPED_CHECK_DESIGN   "./check/check_design_${TOP_DESIGN_NAME}_mapped"
set MAPPED_CHECK_TIMING   "./check/check_timing_${TOP_DESIGN_NAME}_mapped"
set MAPPED_DDC            "./ddc/${TOP_DESIGN_NAME}_mapped.ddc"

set SVF                   "./svf/${TOP_DESIGN_NAME}.svf"

set CONS                  "./cons/top_cons.tcl"

set RPT_QOR               "./rpt/pre_${TOP_DESIGN_NAME}.qor"
set RPT_CONS              "./rpt/pre_${TOP_DESIGN_NAME}.constraint"
set RPT_AREA              "./rpt/pre_${TOP_DESIGN_NAME}.area"
set RPT_CLK               "./rpt/pre_${TOP_DESIGN_NAME}.clk"
set RPT_POWER             "./rpt/pre_${TOP_DESIGN_NAME}.power"
set RPT_SETUP_TIM         "./rpt/pre_setup_${TOP_DESIGN_NAME}.tim"
set RPT_HOLD_TIM          "./rpt/pre_hold_${TOP_DESIGN_NAME}.tim"

set GATE_NETLIST          "./gate/${TOP_DESIGN_NAME}_netlist.v"

set SDC                   "./sdc/pre_${TOP_DESIGN_NAME}.sdc"

set SDF                   "./sdf/pre_${TOP_DESIGN_NAME}.sdf"
```

### `scr/synth_top.tcl` — Complete verified template

```tcl
sh date
#********************************* Read rtl files **********************************************************
set hostFiles   [split [eval \ls -1 \
${RTL_PATH}/*.v | grep -v \
${EXCLUDE_FILE} \
]]

foreach filename ${hostFiles} {
analyze -f verilog -lib work ${filename}
}

#********************************* Compile top design **********************************************************
elaborate -library work ${TOP_DESIGN_NAME}

current_design ${TOP_DESIGN_NAME}

uniquify -force

link

check_design -unmapped > ${UNMAPPED_CHECK_DESIGN}

write -hierarchy -f ddc -output ${UNMAPPED_DDC}

set_svf ${SVF}

set_operating_conditions -analysis_type  on_chip_variation

source ${CONS}

check_timing > ${UNMAPPED_CHECK_TIMING}

set_clock_gating_style -sequential_cell latch \
                       -positive_edge_logic {integrated} \
                       -control_point before -control_signal scan_enable \
                       -observation_point false \
                       -minimum_bitwidth 4 \
                       -max_fanout 24 \
                       -num_stages 3

set_clock_gate_latency -overwrite -stage 0 -fanout_latency {1-inf 0}
set_clock_gate_latency -overwrite -stage 1 -fanout_latency {1-inf -0.05}
set_clock_gate_latency -overwrite -stage 2 -fanout_latency {1-inf -0.10}
set_clock_gate_latency -overwrite -stage 3 -fanout_latency {1-inf -0.15}
set_clock_gate_latency -overwrite -stage 4 -fanout_latency {1-inf -0.25}
set_clock_gate_latency -overwrite -stage 5 -fanout_latency {1-inf -0.45}

compile_ultra -no_boundary_optimization -no_autoungroup -no_seq_output_inversion -gate_clock
compile_ultra -no_boundary_optimization -no_autoungroup -no_seq_output_inversion -gate_clock -incremental

set_max_area 0
optimize_netlist -area

change_names -rules verilog -hierarchy

set_svf -off

report_qor > ${RPT_QOR}
report_constraint -significant_digits 4 -verbose > ${RPT_CONS}
report_area > ${RPT_AREA}
report_clock > ${RPT_CLK}
report_power > ${RPT_POWER}
report_timing -delay max -max_paths 50 > ${RPT_SETUP_TIM}
report_timing -delay min -max_paths 50 > ${RPT_HOLD_TIM}

check_design -unmapped > ${MAPPED_CHECK_DESIGN}
check_timing > ${MAPPED_CHECK_TIMING}

write -hierarchy -format verilog -output ${GATE_NETLIST}
write_sdc ${SDC}
write_sdf -version 2.1 ${SDF}

write -hierarchy -format ddc -output ${MAPPED_DDC}

sh date

exit
```

### `Makefile` — Verified template

```makefile
run:
	@dc_shell-t -f ./scr/synth_top.tcl | tee ./log/synth_top.log

read_ddc:
	@dc_shell-t -f ./scr/read_ddc.tcl

clean:
	@rm -rf *.log *.pvl *.syn *.mr default.svf alib-52/ work/
```

### `cons/top_cons.tcl` — Template (modify per design ports)

```tcl
# Constraints for YOUR_MODULE_NAME
# Adjust clock period and port names to match your RTL

set clk_period 1000    # 1MHz = 1000ns

# Create main clock on clk port
create_clock -name Clk -period $clk_period [get_ports clk]
set_dont_touch_network {clk}

# Virtual clock for I/O delay calculation
create_clock -name v_Clk -period $clk_period

# Input delays — list ALL input ports (except clk and rst_n)
set_input_delay  [expr $clk_period * 0.1] -clock v_Clk [get_ports data_in]
set_input_delay  [expr $clk_period * 0.1] -clock v_Clk [get_ports en]

# Output delays — list ALL output ports
set_output_delay [expr $clk_period * 0.1] -clock v_Clk [get_ports data_out[*]]

# Clock groups
set_clock_groups -asynchronous -group {Clk v_Clk}

# Reset is asynchronous - false path
set_false_path -from [get_ports rst_n]
set_dont_touch_network [get_ports rst_n]
set_drive 0 [get_ports rst_n]
set_ideal_network -no_propagate rst_n

# Output load
set_load [expr {100/1000}] [get_ports data_out[*]]

# Design-level constraints
set_max_fanout 24 [current_design]
set_max_transition 20 [current_design]
```

## What to Modify Per Design

When starting a new synthesis project, modify **only these items**:

| File | What to change | How |
|------|---------------|-----|
| `.synopsys_dc.setup` | `TOP_DESIGN_NAME` | Replace with your RTL top module name |
| `cons/top_cons.tcl` | Clock period, all port names | Match `clk_period` and ports to your RTL |
| `rtl/` | RTL source files | Upload your `.v` files, remove example RTL |

**Do NOT modify:** `scr/synth_top.tcl`, library paths, `Makefile` — these are pre-verified.

## Constraint Writing Checklist

| Item | What to check |
|------|--------------|
| Clock port name | Match `clk` in RTL |
| Clock period | Target frequency (ns) |
| All input ports | `set_input_delay` on each (except clk, rst_n) |
| All output ports | `set_output_delay` on each |
| Async signals | `set_false_path` (reset, scan_enable, etc.) |
| Output load | Reasonable capacitance for I/O |
| Clock groups | `set_clock_groups -asynchronous` |

## Workflow Commands

```bash
# Upload design files to server
scp rtl/*.v user@server:/project/dc/rtl/

# Run synthesis
ssh user@server "cd /project/dc && make run"

# Download results
scp user@server:/project/dc/gate/*_netlist.v ./
scp -r user@server:/project/dc/rpt/ ./

# Clean up (removes all generated files)
ssh user@server "cd /project/dc && make clean"
```

## Reading Synthesis Reports

### QoR Report (`rpt/pre_*.qor`)

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

- `slack (MET)` = positive → setup timing OK
- `slack (VIOLATED)` = negative → need to fix

### Hold Timing (`rpt/pre_hold_*.tim`)

- Check hold violations; usually fixed in post-layout

### Power Report (`rpt/pre_*.power`)

```
Total Dynamic Power    = 108.65 nW   ← Switching + Internal
Cell Leakage Power     =   4.60 nW   ← Static power
```

## Common Issues & Fixes

| Issue | Cause | Fix |
|-------|-------|-----|
| `Cannot find design 'xxx' in library 'work'` | `TOP_DESIGN_NAME` doesn't match module name | Set correct name in `.synopsys_dc.setup` |
| `VER-37 Internal assertion failure` | For-loop with early exit in combinational logic | Rewrite using combinational priority encoder outside always block |
| `VER-318 signed to unsigned part selection` | Integer loop variable used as bit select | Use combinational block with `always @(*)` for arbitration |
| `Can't find port 'xxx'` | Constraint port name doesn't match RTL | Update `cons/top_cons.tcl` with correct port names |
| Setup violation (negative slack) | Clock period too tight | Increase `clk_period` or optimize design |
| Hold violation | Fast path with no logic | Usually fixed in post-layout |
| `ls: cannot access rtl/*.v` | No RTL files in rtl/ directory | Upload RTL files first |

## Tips

1. **Always copy from example template** — avoids syntax errors in `.synopsys_dc.setup` and `synth_top.tcl`
2. **Only change `TOP_DESIGN_NAME` and `cons/top_cons.tcl`** — rest is pre-verified
3. **Always check `check_design` output** in `check/` directory before and after synthesis
4. **For first run, use relaxed clock** (e.g., 1MHz / 1000ns) to verify flow correctness, then tighten
5. **Gate-level netlist** can be used for post-synthesis simulation with the SDF file
6. **SVF file is critical** — needed for Formality verification after synthesis
