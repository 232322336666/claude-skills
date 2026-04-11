---
name: eda-team-flow
description: Use when you want to run VCS simulation, DC synthesis, and FM formal verification as a team workflow (serial: VCS → DC → FM). Includes memory checkpoint mechanism for crash recovery.
---

# EDA Team Flow: VCS → DC → FM Serial Workflow with Checkpoints

## Overview

Run a 3-agent team to execute the full EDA flow in serial order: VCS (verify RTL) → DC (synthesize) → FM (formal verify). Each agent uses its own skill and **writes a memory checkpoint** on completion, so the flow can resume from any point after a crash.

## Why Serial (Not Parallel)

1. **RTL consistency**: VCS must verify RTL first; if DC finds RTL issues and fixes them, VCS must re-run with the same fixed RTL
2. **No resource contention**: DC uses 16 cores + heavy memory; VCS also needs significant resources
3. **No license conflict**: Both use Synopsys licenses
4. **Clean dependency chain**: VCS passes → DC synthesizes verified RTL → FM verifies DC output

## Architecture

```
Step 1           Step 2              Step 3
┌─────────┐     ┌──────────┐       ┌──────────────┐
│  VCS    │────▶│   DC     │──────▶│     FM       │
│ Simulate│     │ Synthesize│      │ Formal Verify │
└────┬────┘     └────┬─────┘       └──────┬───────┘
     │               │                     │
     ▼               ▼                     ▼
  CHECKPOINT 1   CHECKPOINT 2          CHECKPOINT 3
  (memory)       (memory)              (memory)
```

Each checkpoint is written to the project memory system. On crash/restart, read the latest checkpoint to know exactly where to resume.

## Memory Checkpoint System

### How It Works

1. At each stage completion, write a memory file to `C:\Users\$USER\.claude\projects\D--claudecode-digits\memory\`
2. File named `eda-progress-{project}.md` with type `project`
3. Contains: stage completed, results, remote paths, next action, any issues
4. On crash restart: read MEMORY.md → find progress file → resume from last completed stage

### Checkpoint File Format

```markdown
---
name: EDA Progress {project}
description: EDA flow progress tracker for {project}. Current stage, results, and next action.
type: project
---

# EDA Flow Progress: {project}

**Last Updated:** 2026-04-09 15:30
**Project Dir:** ~/agent_digits/{project}/
**Top Module:** {top_module_name}
**Local RTL:** D:\claudecode\digits\{project}\

## Current Stage: VCS_DONE / DC_DONE / FM_DONE / ALL_COMPLETE

## Stage 1: VCS Simulation — [PENDING / DONE / FAILED]
- Remote: ~/agent_digits/{project}/vcs/
- Status: PASS/FAIL
- Tests: X/Y passed
- RTL fixes applied: (describe any changes, or "none")
- Waveform: wave.vcd downloaded to local (yes/no)
- Key log: (paste critical output)

## Stage 2: DC Synthesis — [PENDING / DONE / FAILED]
- Remote: ~/agent_digits/{project}/dc/
- Status: PASS/FAIL
- Clock: X MHz (period X ns)
- Setup slack: X.XX ns (MET/VIOLATED)
- Hold slack: X.XX ns (MET/VIOLATED)
- Area: XXXX um²
- Power: XXX nW dynamic + XXX nW leakage
- Gate netlist: gate/{top_module_name}_netlist.v (exists: yes/no)
- SVF: svf/{top_module_name}.svf (exists: yes/no)
- Key log: (paste critical output)

## Stage 3: FM Formal Verification — [PENDING / DONE / FAILED]
- Remote: ~/agent_digits/{project}/fm/
- Status: PASS/FAIL
- Compare points matched: XXX
- Compare points unmatched: XXX
- Unmatched details: (e.g., "27 clock-gate LAT = normal")
- Key log: (paste critical output)

## Next Action
(Describe what needs to happen next, e.g., "Run DC synthesis", "Run FM verification", "Fix RTL and re-run VCS")

## Issues Encountered
(List any problems and how they were resolved)
```

### When to Write Checkpoints

| Checkpoint | When to write | What to include |
|-----------|---------------|-----------------|
| After VCS | VCS simulation completes (pass or fail) | Test results, any RTL fixes, waveform status |
| After DC | DC synthesis completes (pass or fail) | Slack, area, power, output files existence |
| After FM | FM verification completes | PASS/FAIL, matched/unmatched points |
| On error | Any stage fails | Error message, diagnosis, suggested fix |

## Step-by-Step Workflow

### Step 1: Create Team

```
Use TeamCreate to create a team called "eda-flow"
```

### Step 2: Create Tasks (Serial Order)

```
Task 1: VCS Simulation — no dependency
Task 2: DC Synthesis — blocked by Task 1
Task 3: FM Verification — blocked by Task 2
```

### Step 3: Write Initial Progress File

Before starting, write the checkpoint file with all stages as PENDING:

```markdown
Current Stage: VCS_PENDING
All stages: PENDING
Next Action: Run VCS simulation
```

### Step 4: Spawn Agents (one at a time, sequential)

**First: Spawn VCS agent**

```
Agent: vcs-runner
Task: Task 1
Prompt: (see VCS Agent Prompt below)
```

Wait for VCS agent to complete → **Write Checkpoint 1** → Shut down VCS agent

**Then: Spawn DC agent**

```
Agent: dc-runner
Task: Task 2
Prompt: (see DC Agent Prompt below)
```

Wait for DC agent to complete → **Write Checkpoint 2** → Shut down DC agent

**Finally: Spawn FM agent**

```
Agent: fm-runner
Task: Task 3
Prompt: (see FM Agent Prompt below)
```

Wait for FM agent to complete → **Write Checkpoint 3** → Shut down FM agent

### Step 5: Summarize Results

After all stages complete, output the final summary.

## Crash Recovery Procedure

### If the session crashes and restarts:

1. **Read memory**: The new session automatically loads MEMORY.md
2. **Find progress file**: Look for `eda-progress-{project}.md` entry
3. **Read checkpoint**: The file tells you exactly:
   - Which stages completed (DONE)
   - Which stages are pending
   - What the results were
   - Where the files are on the remote server
4. **Resume**: Spawn only the agents for remaining stages
5. **No duplicate work**: Completed stages are marked DONE, skip them

### Example Recovery Scenarios

| Crash point | What to do |
|-------------|-----------|
| During VCS | Read checkpoint → "VCS_PENDING" → restart VCS agent |
| After VCS, before DC | Read checkpoint → "VCS_DONE" → start DC agent |
| During DC | Read checkpoint → "VCS_DONE, DC_PENDING" → restart DC agent |
| After DC, before FM | Read checkpoint → "VCS_DONE, DC_DONE" → start FM agent |
| During FM | Read checkpoint → "VCS_DONE, DC_DONE, FM_PENDING" → restart FM agent |
| After FM | Read checkpoint → "ALL_COMPLETE" → just summarize |

### Important: Always Verify Remote State

After crash recovery, before resuming, verify the remote files match what the checkpoint says:

```bash
# Check if claimed-completed outputs actually exist
ssh user@server "ls /path/dc/gate/{top}_netlist.v /path/dc/svf/{top}.svf"
ssh user@server "ls /path/vcs/wave.vcd"
```

If files are missing, re-run that stage.

## Agent Prompt Templates

### VCS Agent Prompt

```
You are the VCS simulation agent on team "eda-flow".

Your task: Run VCS simulation for {project_name} on remote server.

Remote: $USER@$SERVER
Remote path: ~/agent_digits/{project}/vcs/
Example template: ~/agent_digits/example/vcs/
Local RTL: D:\claudecode\digits\{project}\

Steps (follow remote-vcs-waveform-viewing skill):
1. Copy example template to project vcs/ directory (clean generated files first)
2. Upload RTL and testbench files from local
3. Update Makefile: set RTL, TB, TOP variables
4. Update filelist.f with design files
5. Run: make clean && make
6. Check sim.log for pass/fail results
7. Download wave.vcd to local

When done, mark your task as completed and notify the team lead with:
- PASS/FAIL status
- Test count (X/Y passed)
- Any RTL issues found
- Waveform download status
```

### DC Agent Prompt

```
You are the DC synthesis agent on team "eda-flow".

Your task: Run DC synthesis for {project_name} on remote server.

Remote: $USER@$SERVER
Remote path: ~/agent_digits/{project}/dc/
Example template: ~/agent_digits/example/dc/
Top module: {top_module_name}
Local RTL: D:\claudecode\digits\{project}\

PREREQUISITE: VCS simulation has passed. RTL is verified functional.

Steps (follow remote-dc-synthesis skill):
1. Copy example template to project dc/ directory (clean generated files first)
2. Upload RTL files ONLY (no testbench) from local
   Files: {list of .v files}
3. sed -i 's/shift_reg_16bit/{top_module_name}/g' .synopsys_dc.setup
4. Write cons/top_cons.tcl with constraints matching RTL ports:
   Clock: {clk_period}ns ({frequency}MHz)
   Ports: {list of input/output ports}
5. Run: make run
6. Check ALL reports:
   - rpt/pre_*.qor → slack >= 0?
   - rpt/pre_*.area → total cell area
   - rpt/pre_setup_*.tim → setup MET?
   - rpt/pre_hold_*.tim → hold MET?
   - rpt/pre_*.power → dynamic + leakage
7. Verify outputs exist:
   - gate/{top_module_name}_netlist.v
   - svf/{top_module_name}.svf
8. If DC crashes with VER-37: check RTL for for-loop with early exit in
   combinational blocks, rewrite as priority encoder outside always block

When done, mark your task as completed and notify the team lead with:
- Setup/Hold slack and MET/VIOLATED
- Area (um²)
- Gate netlist and SVF existence confirmed
- Any RTL modifications made for DC compatibility
```

### FM Agent Prompt

```
You are the FM formal verification agent on team "eda-flow".

Your task: Run Formality verification for {project_name} on remote server.

Remote: $USER@$SERVER
Remote path: ~/agent_digits/{project}/fm/
Example template: ~/agent_digits/example/fm/
Top module: {top_module_name}

PREREQUISITE: DC synthesis completed successfully.
Gate netlist: ../dc/gate/{top_module_name}_netlist.v
SVF: ../dc/svf/{top_module_name}.svf

Steps (follow remote-fm-formality skill):
1. Copy example template to project fm/ directory (clean generated files first)
2. sed -i 's/shift_reg_16bit/{top_module_name}/g' .synopsys_fm.setup
3. Verify DC outputs exist on remote (ls the files)
4. Run: make run
5. Check log for "Verification SUCCEEDED" or "Verification FAILED"
6. If FAILED: check rpt/*_unmatched_points.rpt and rpt/*_failing

When done, mark your task as completed and notify the team lead with:
- PASS/FAIL status
- Compare points matched/unmatched
- Any unmatched details (clock-gate LAT = normal)
```

## Remote Server Details

| Item | Value |
|------|-------|
| Server | `$SERVER` |
| User | `$USER` |
| Project base | `~/agent_digits/{project}/` |
| Example templates | `~/agent_digits/example/` |
| Library | TSMC 65nm `tcbn65lp` |

## Project Directory Layout on Remote Server

```
~/agent_digits/{project}/
├── vcs/          ← VCS workspace
│   ├── Makefile
│   ├── filelist.f
│   ├── *.v       (RTL + testbench)
│   └── wave.vcd
├── dc/           ← DC workspace
│   ├── .synopsys_dc.setup
│   ├── Makefile
│   ├── rtl/      (RTL only, no testbench)
│   ├── scr/synth_top.tcl
│   ├── cons/top_cons.tcl
│   ├── gate/     (output)
│   └── svf/      (output for FM)
└── fm/           ← FM workspace
    ├── .synopsys_fm.setup
    ├── Makefile
    └── scr/fm.tcl
```

## Result Summary Template

After all stages complete:

```
## EDA Flow Results: {project_name}

### VCS Simulation: PASS/FAIL
- Tests: X/Y passed
- Waveform: wave.vcd

### DC Synthesis: PASS/FAIL
- Clock: X MHz, Setup slack: X.XX ns (MET), Hold slack: X.XX ns (MET)
- Area: XXXX um² | Power: XXX nW

### FM Formal Verification: PASS/FAIL
- Compare points: XXX matched, X unmatched (clock-gate LAT = normal)

### Overall: ALL PASS / NEEDS FIX
```

## Troubleshooting

| Scenario | What to do |
|----------|-----------|
| VCS fails | Fix RTL/testbench locally, re-run VCS agent |
| DC crashes (VER-37) | Rewrite for-loops as priority encoders in RTL, re-run VCS then DC |
| DC timing violation | Relax clock period in cons/top_cons.tcl, re-run DC |
| FM verification fails | Check unmatched points, may need DC re-run |
| Session crashes | Read memory checkpoint, resume from last DONE stage |
| Checkpoint says DONE but files missing | Verify remote state, re-run that stage |
