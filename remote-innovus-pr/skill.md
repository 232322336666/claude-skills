---
name: remote-innovus-pr
description: Use when running Cadence Innovus Place & Route on a remote Linux server. Covers project setup, floorplan, CTS, routing, and signoff for TSMC 65nm designs. First copy from example template, then modify per design. Use this skill whenever the user mentions Innovus, P&R, place and route, backend, physical design, GDS generation, or wants to run post-synthesis layout.
---

# Remote Innovus Place & Route

## Overview

Run Cadence Innovus P&R flow on a remote Linux server via SSH/SCP. The server has Innovus v21.16 and TSMC 65nm libraries pre-installed. This skill covers: copying example template, modifying scripts for your design, running the full flow (init → floorplan → placement → CTS → routing → signoff), and downloading results.

## When to Use

- After DC synthesis and FM formal verification are complete
- When you need to generate GDS, SDF, SPEF from a gate-level netlist
- When the user says "跑后端", "P&R", "Innovus", "layout", "GDS"

## Prerequisites

### Remote (Linux server)
- Server: `mee1.ee.cityu.edu.hk`, User: `zhengnafu2`
- Innovus: `/opt/cadence/installs/INNOVUS211/bin/innovus` (**NOT** INNOVUSEXPORT21)
- TSMC 65nm libraries at `/home/research/yeke22/STDCELL/tcbn65lp_220a/`
- Example template at `/home/research/zhengnafu2/agent_digits/example/innovus_flow/`
- DC outputs must exist: `../dc/gate/{top}_netlist.v`, `../dc/sdc/{top}.sdc`

### Local (Windows/macOS)
- SSH/SCP access configured
- Disk space for GDS (~1-2MB per design)

## CRITICAL: Always Copy from Example Template

The example template has all library paths, MMMC config, and layer maps pre-configured. **Never write scripts from scratch.** Copy the template, then modify only the design-specific parts.

```bash
# Copy template
ssh zhengnafu2@mee1.ee.cityu.edu.hk \
  "cp -r /home/research/zhengnafu2/agent_digits/example/innovus_flow/ \
   /home/research/zhengnafu2/agent_digits/{project}/innovus/"

# Clean generated files from example run
ssh zhengnafu2@mee1.ee.cityu.edu.hk \
  "cd /home/research/zhengnafu2/agent_digits/{project}/innovus/ && \
   rm -rf save/* output/* rpt/* extLogDir cdBfile *.rpt *.rpt.old logFile power.rpt \
   rc_model.bin mp_data* HRAM_TOP timingReports .WeightWriting* .timing_file* *.swp .cadence/ \
   innovus.cmd* innovus.log* innovus.logv* innovus_run.log model.asrt*"
```

## Project Directory Structure

```
{project}/innovus/
├── lib/                    # LEF, GDS map, layer map (from template, DO NOT MODIFY)
│   ├── tcbn65lp_9lmT2.lef          # Standard cell LEF (9 metal layer)
│   ├── innovus_gds_out.map         # GDS stream out map
│   ├── quantus_extract.layermap    # RC extraction layer map
│   └── tQuantus_model.bin          # Generated at runtime
├── scripts/                # TCL scripts (modify per design)
│   ├── all.tcl              # Master script: sources 1-6 in order
│   ├── mmmc.tcl             # MMMC view definition (WC/TC/BC corners)
│   ├── 1-init.tcl           # Design import + MMMC setup
│   ├── 2-fp.tcl             # Floorplan + power ring + stripes + sroute
│   ├── 3-place.tcl          # Placement + optimization
│   ├── 4-cts.tcl            # Clock tree synthesis + decap
│   ├── 5-route.tcl          # Global/detail routing + post-route opt + filler
│   ├── 6-signoff.tcl        # DRC/connectivity/timing + GDS/SDF/SPEF output
│   └── fix_pwr_signoff.tcl  # (optional) Power fix + re-signoff
├── pre/                     # Input files (copy from DC outputs)
│   ├── {top}_netlist.v      # Gate-level netlist from DC
│   └── {top}.sdc            # Timing constraints from DC
├── save/                    # Checkpoint saves at each stage
├── output/                  # Final outputs (GDS, SDF, SPEF, netlist)
└── rpt/                     # Reports (timing, DRC, connectivity, power)
```

## What to Modify Per Design

| File | What to change | Typical values |
|------|---------------|----------------|
| `scripts/1-init.tcl` | `init_top_cell`, `init_lef_file`, `init_verilog`, `init_pwr_net`, remove `win` | top module name, power net names |
| `scripts/mmmc.tcl` | SDC path in `create_constraint_mode` | `./pre/{top}.sdc` |
| `scripts/2-fp.tcl` | **Rewrite entirely** — floorplan size, power nets, ring/stripes | die size, VDD/VSS |
| `scripts/3-place.tcl` | Usually no changes needed | — |
| `scripts/4-cts.tcl` | Usually no changes needed | — |
| `scripts/5-route.tcl` | Fix path bug: `/rpt/` → `./rpt/` (template bug) | line ~72-73 |
| `scripts/6-signoff.tcl` | **Rewrite** — GDS merge list, filler cells, remove IO references | stdcell GDS only |

## Step-by-Step Workflow

### Step 1: Copy template and prepare inputs

```bash
# Copy template (see CRITICAL section above)

# Copy DC outputs to pre/ directory
ssh zhengnafu2@mee1.ee.cityu.edu.hk \
  "cd /home/research/zhengnafu2/agent_digits/{project}/innovus/pre && \
   rm -f *.v *.sdc *.io && \
   cp /home/research/zhengnafu2/agent_digits/{project}/dc/gate/{top}_netlist.v ./ && \
   cp /home/research/zhengnafu2/agent_digits/{project}/dc/sdc/{top}.sdc ./"
```

### Step 2: Modify scripts

#### 1-init.tcl — Key modifications

```tcl
# Change these lines:
set init_lef_file {
        ./lib/tcbn65lp_9lmT2.lef     # Only stdcell LEF, remove IO/analog LEFs
}
set init_top_cell {top}                # Your top module name
set init_verilog  ./pre/{top}_netlist.v
set init_gnd_net {VSS}
set init_pwr_net {VDD}                 # Single power domain, not AVDD/DVDD

# CRITICAL: Remove the "win" command — it crashes in -nowin mode
# Delete any line that is just "win"
```

#### 2-fp.tcl — Rewrite for pure core design

The example template has IO pads, analog macros, multiple power domains — all of which must be removed for a pure digital core design.

**Template for a simple core design:**

```tcl
##2-FLOORPLAN

# Die size: estimate from DC area. Target ~70% utilization.
# DC area / 0.70 = core area. Add margins (5um each side).
# Example: DC area 2200um² → core ~3140um² → side ~56 → die 66x66
floorPlan -site core -d 80 80 5.0 5.0 5.0 5.0

# Global nets — connect VDD/VSS to all std cells
clearGlobalNets
globalNetConnect VDD -type pgpin -pin VDD -inst * -all
globalNetConnect VSS -type pgpin -pin VSS -inst * -all
globalNetConnect VDD -type tiehi -inst * -all
globalNetConnect VSS -type tielo -inst * -all

# Power ring on M7 (horizontal) and M8 (vertical)
setAddRingMode -ring_target default -stacked_via_top_layer M8 -stacked_via_bottom_layer M1
addRing -nets {VDD VSS} -type core_rings -follow core \
  -layer {top M7 bottom M7 left M8 right M8} \
  -width {top 2 bottom 2 left 2 right 2} \
  -spacing {top 1 bottom 1 left 1 right 1} \
  -offset {top 0.5 bottom 0.5 left 0.5 right 0.5} -center 0

# Power stripes — MUST be added here (before routing!)
# These provide via stacks from M8 down to M1 followpins
setAddStripeMode -stacked_via_top_layer M8 -stacked_via_bottom_layer M1
addStripe -start_from left -start 20 -nets {VDD VSS} -layer M8 \
  -direction vertical -width 2 -spacing 1 -number_of_sets 2 -create_pins 1

# Special route: connect followpins to ring and stripes
setSrouteMode -viaConnectToShape {ring stripe followpin}
sroute -connect {corePin floatingStripe} -nets {VDD VSS} \
  -layerChangeRange {M1(1) M8(8)} \
  -floatingStripeTarget {ring followpin stripe} \
  -crossoverViaLayerRange {M1(1) M8(8)} \
  -allowLayerChange 1 -targetViaLayerRange {M1(1) M8(8)}

fixVia -minCut
fixVia -minStep
fixVia -short
fixOpenFill -net {VDD VSS}

saveDesign ./save/fp.enc
```

#### mmmc.tcl — Only change SDC path

```tcl
# Find this line and change the SDC path:
create_constraint_mode -name synth -sdc_files { ./pre/{top}.sdc}
```

Everything else (RC corners, library sets, delay corners, analysis views) stays the same — it's pre-configured for TSMC 65nm WC/TC/BC.

#### 5-route.tcl — Fix template bug

The example template has absolute paths on lines ~72-73:
```tcl
# WRONG (template bug):
verifyConnectivity -type all -report /rpt/postECO_v4.conn.rpt
reportGateCount -level 5 -stdCellOnly -outfile /rpt/postECO_v4.gateCount.rpt

# FIX: add "./" prefix
verifyConnectivity -type all -report ./rpt/postECO_v4.conn.rpt
reportGateCount -level 5 -stdCellOnly -outfile ./rpt/postECO_v4.gateCount.rpt
```

#### 6-signoff.tcl — Rewrite for core design

```tcl
##SIGNOFF — simplified for pure core design

# Reports
verify_drc -report ./rpt/signOff.drc.rpt -limit 10000
verifyConnectivity -type all -report ./rpt/signOff.conn.rpt
reportGateCount -level 5 -stdCellOnly -outfile ./rpt/signOff.gatecount.rpt
report_power -rail_analysis_format VS -outfile ./rpt/signOff.pwr.rpt
timeDesign -postRoute -outDir ./rpt/signOffTimingReports
timeDesign -postRoute -hold -outDir ./rpt/signOffTimingReports

# SPEF output (WC and BC corners)
set timing_enable_simultaneous_setup_hold_mode false
setAnalysisMode -checkType setup
set_analysis_view -setup setup -hold setup
rcOut -spef ./output/{top}_WC.spef -rc_corner rcWC

setAnalysisMode -checkType hold
set_analysis_view -setup hold -hold hold
rcOut -spef ./output/{top}_BC.spef -rc_corner rcBC

# SDF output
set_analysis_view -setup {setup typical} -hold {hold typical}
write_sdf -version 3.0 ./output/${init_top_cell}.sdf \
  -process best:typical:worst -filter -nonegchecks -celltiming all \
  -target_application verilog \
  -temperature -40.0:25.0:125 -voltage 1.32:1.20:1.08 \
  -typ_view hold -max_view setup -min_view hold

# Netlists
saveNetlist ./output/Calibre_${init_top_cell}_wo_filler.v \
  -includePowerGround -phys \
  -excludeCellInst {FILL1 FILL16 FILL2 FILL32 FILL4 FILL64 FILL8}
saveNetlist ./output/Calibre_${init_top_cell}_w_filler.v -includePowerGround -phys
saveNetlist ./output/EDI_to_VCS_${init_top_cell}.v \
  -excludeCellInst {FILL1 FILL16 FILL2 FILL32 FILL4 FILL64 FILL8}

# GDS — merge with stdcell GDS only (no IO, no analog)
setStreamOutMode -SEvianames false -specifyViaName default \
  -supportPathType4 true -virtualConnection false -textSize 1
streamOut ./output/${init_top_cell}.gds -dieAreaAsBoundary \
  -libName DesignLib -mapFile ./lib/innovus_gds_out.map \
  -mode ALL -stripes 1 -units 1000 -merge {\
  /home/research/yeke22/STDCELL/tcbn65lp_220a/A100001_20180209/TSMCHOME/digital/Back_End/gds/tcbn65lp_200a/tcbn65lp.gds}

saveDesign ./save/signoff.enc
```

### Step 3: Run Innovus

```bash
# IMPORTANT: Use full path to INNOVUS211, not the INNOVUSEXPORT21 that `which` resolves to
# IMPORTANT: Use -nowin flag for non-GUI mode
ssh zhengnafu2@mee1.ee.cityu.edu.hk \
  "cd /home/research/zhengnafu2/agent_digits/{project}/innovus && \
   nohup /opt/cadence/installs/INNOVUS211/bin/innovus -nowin \
   -files scripts/all.tcl > innovus_run.log 2>&1 &"
```

Typical runtime: 15-20 minutes for a small design (~2000 gates).

### Step 4: Monitor progress

```bash
# Check log line count (grows as flow progresses)
ssh zhengnafu2@mee1.ee.cityu.edu.hk \
  "wc -l /home/research/zhengnafu2/agent_digits/{project}/innovus/innovus.log"

# Check latest activity
ssh zhengnafu2@mee1.ee.cityu.edu.hk \
  "tail -30 /home/research/zhengnafu2/agent_digits/{project}/innovus/innovus.log"

# Check if still running
ssh zhengnafu2@mee1.ee.cityu.edu.hk \
  "ps aux | grep 'zhengnafu2.*innovus' | grep -v grep | wc -l"
```

Log milestones by line count (approximate):
- ~160: Init complete, libraries loaded
- ~500-1000: Placement
- ~5000-10000: CTS + initial routing
- ~10000-15000: Post-route optimization + RC extraction
- ~16000+: Signoff complete

### Step 5: Analyze results

```bash
# Check errors
ssh zhengnafu2@mee1.ee.cityu.edu.hk \
  "grep 'ERROR' innovus.log | grep -v 'Error Limit'"

# Check DRC
ssh zhengnafu2@mee1.ee.cityu.edu.hk \
  "cat rpt/signOff.drc.rpt"

# Check connectivity
ssh zhengnafu2@mee1.ee.cityu.edu.hk \
  "tail -10 rpt/signOff.conn.rpt"

# Check timing
ssh zhengnafu2@mee1.ee.cityu.edu.hk \
  "zcat rpt/signOffTimingReports/{top}_postRoute.summary.gz | tail -20"
ssh zhengnafu2@mee1.ee.cityu.edu.hk \
  "zcat rpt/signOffTimingReports/{top}_postRoute_hold.summary.gz | tail -20"

# Check gate count
ssh zhengnafu2@mee1.ee.cityu.edu.hk \
  "cat rpt/signOff.gatecount.rpt"
```

### Step 6: Download outputs

```bash
mkdir -p {local_project}/innovus_output
scp zhengnafu2@mee1.ee.cityu.edu.hk:\
  /home/research/zhengnafu2/agent_digits/{project}/innovus/output/{top}.gds \
  {local_project}/innovus_output/
scp zhengnafu2@mee1.ee.cityu.edu.hk:\
  /home/research/zhengnafu2/agent_digits/{project}/innovus/output/{top}.sdf \
  {local_project}/innovus_output/
scp zhengnafu2@mee1.ee.cityu.edu.hk:\
  /home/research/zhengnafu2/agent_digits/{project}/innovus/output/EDI_to_VCS_{top}.v \
  {local_project}/innovus_output/
scp zhengnafu2@mee1.ee.cityu.edu.hk:\
  /home/research/zhengnafu2/agent_digits/{project}/innovus/output/{top}_WC.spef \
  {local_project}/innovus_output/
scp zhengnafu2@mee1.ee.cityu.edu.hk:\
  /home/research/zhengnafu2/agent_digits/{project}/innovus/output/{top}_BC.spef \
  {local_project}/innovus_output/
```

## Reading Innovus Reports

### Timing Summary

```
+--------------------+---------+
|     Setup mode     |   all   |
+--------------------+---------+
|           WNS (ns):|  0.000  |  ← Worst Negative Slack. 0 = MET, negative = VIOLATED
|           TNS (ns):|  0.000  |  ← Total Negative Slack. 0 = all paths meet timing
|    Violating Paths:|    0    |  ← Must be 0
+--------------------+---------+
```

- **Setup** (max delay): checks if data arrives before clock edge. WNS ≥ 0 = PASS
- **Hold** (min delay): checks if data stable after clock edge. WNS ≥ 0 = PASS

### DRC Report

```
No DRC violations were found  ← Must see this line
```

### Connectivity Report

```
Begin Summary
    0 Problem(s)  ← Must be 0
End Summary
```

Common issue: "special routes with opens" on VDD/VSS means M1 followpins not fully connected to power ring. Add power stripes in floorplan to fix.

### Gate Count

```
Level 0 Module {top}  Gates=2032 Cells=599 Area=2194.9 um^2
```

## Common Issues & Fixes

| Issue | Cause | Fix |
|-------|-------|-----|
| `innovus` fails with "build signature file" error | `which innovus` resolves to INNOVUSEXPORT21 | Use full path: `/opt/cadence/installs/INNOVUS211/bin/innovus` |
| Script stops at `win` command in -nowin mode | `win` opens GUI, incompatible with -nowin | Remove `win` line from 1-init.tcl |
| `IMPCCOPT-2215: route graph for net 'clk' not fully connected` | Clock gating cells split traversal graph | Harmless — timing still passes |
| `IMPOPT-310: Design density exceeds 95%` | Floorplan too small | Increase die size in 2-fp.tcl |
| DRC violations after adding stripes post-route | Via stacks from stripes short signal wires | Add stripes in floorplan stage, BEFORE routing |
| `IMPSYT-7338: session directory not found` | restoreDesign uses `.enc` file instead of `.enc.dat` directory | Use `restoreDesign ./save/{name}.enc.dat {top}` |
| VDD/VSS "special routes with opens" | M1 followpins not connected to power ring | Add power stripes + use `setSrouteMode -viaConnectToShape {ring stripe followpin}` |
| `/rpt/` path errors in signoff | Template bug: absolute path `/rpt/` instead of `./rpt/` | Fix in 5-route.tcl lines ~72-73 |
| IMPCCOPT-2215 clock graph error | Clock gating cells in traversal path | Informational only, does not affect functionality |

## Advanced: Partial Re-run with restoreDesign

If you need to fix something and re-run only part of the flow (e.g., redo routing after fixing floorplan):

```tcl
# Correct syntax: use .enc.dat directory, not .enc file
restoreDesign ./save/{checkpoint}.enc.dat {top}

# Example: load from placement, add stripes, then re-run CTS+route+signoff
restoreDesign ./save/place.enc.dat aer_top
# ... modify floorplan, add stripes ...
saveDesign ./save/place_modified.enc
```

**Important checkpoints:**
- `place.enc` — after placement, before CTS/routing (best for adding stripes)
- `cts.enc` — after CTS, before routing
- `route4.enc` — after all routing + optimization (for signoff tweaks only)

## Tips

1. **Always use the full Innovus path** — `nohup /opt/cadence/installs/INNOVUS211/bin/innovus` — because `nohup` doesn't load shell aliases from .bashrc, and the default `innovus` binary is broken.

2. **Floorplan sizing rule of thumb**: DC area × 1.5 / 0.70 = core area. Then sqrt for side length. Add 10um for margins. It's better to be too large than too small — density under 85% is comfortable.

3. **Power stripes must go in the floorplan** — they create via stacks from M8 down to M1. If added after routing, these vias will short signal wires. Use `addStripe` in 2-fp.tcl before any routing happens.

4. **Monitor memory usage** — Innovus can use 10-12GB for small designs. The server has enough RAM but watch for OOM on larger designs.

5. **The IMPCCOPT-2215 error is harmless** — it appears when clock gating cells are present (DC inserts them automatically when `set_clock_gating_style` is used). The clock tree still works correctly.

6. **MMMC config is pre-configured** — the mmmc.tcl from the example template has WC/TC/BC corners, QRC tech files, SI-aware CDB files all pointing to the right paths. Only the SDC path needs changing.
