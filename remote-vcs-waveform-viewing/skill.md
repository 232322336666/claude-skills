---
name: remote-vcs-waveform-viewing
description: Use when developing Verilog/SystemVerilog on a remote Linux server with VCS and viewing waveforms locally in VS Code with VaporView. First copy from example template, then modify per design.
---

# Remote VCS Simulation with Local Waveform Viewing

## Overview

Run VCS simulations on a remote Linux server and view waveforms locally in VS Code using VaporView plugin. **Always copy from the verified example template first**, then modify for your design.

## When to Use

- Remote server has VCS/Verdi license
- Local machine is Windows/macOS without EDA tools
- Need quick waveform preview without launching Verdi
- Working from home/remote with limited VPN bandwidth

## Prerequisites

**Remote (Linux server):**
- VCS + Verdi installed and licensed
- SSH access configured

**Local (Windows/macOS):**
- VS Code with VaporView extension

## CRITICAL: Always Copy from Example Template

**The example template at `/home/research/zhengnafu2/agent_digits/example/vcs/` is a verified, working reference. Always copy from it to avoid build errors.**

### Step 1: Copy template structure

```bash
# On remote server — copy entire vcs directory
ssh user@server "cd /project && cp -r /home/research/zhengnafu2/agent_digits/example/vcs/ ./vcs_new && cd vcs_new && rm -rf simv* csrc* DVEfiles AN.DB *.fsdb *.log *.vpd *.key *.vdb ucli.key verdi_config_file wave.fsdb wave.vcd"
```

### Step 2: Upload your design files

```bash
scp your_design.v your_tb.v user@server:/project/vcs_new/
```

### Step 3: Modify Makefile variables

Change `RTL`, `TB`, and `TOP` to match your design:

```bash
ssh user@server "cd /project/vcs_new && sed -i 's/shift_reg_16bit.v/your_design.v/g; s/tb_shift_reg.v/your_tb.v/g; s/tb_shift_reg/your_tb_module/g' Makefile"
```

### Step 4: Update filelist.f

```bash
ssh user@server "echo 'your_design.v
your_tb.v' > /project/vcs_new/filelist.f"
```

### Step 5: Run

```bash
ssh user@server "cd /project/vcs_new && make clean && make"
```

## Project Directory Structure

```
vcs/
├── Makefile              # Compile + run commands
├── filelist.f            # File list for Verdi
├── design.v              # RTL design files
├── tb_design.v           # Testbench files
├── simv                  # Generated simulation binary
├── simv.daidir/          # Generated simulation data
├── csrc/                 # Generated C source
├── wave.fsdb             # FSDB waveform (Verdi)
└── wave.vcd              # VCD waveform (VaporView)
```

## Verified Template Files (from example)

### `Makefile` — Verified template

```makefile
# Makefile for VCS Simulation
# VCS + Verdi flow

# *** CHANGE THESE per design ***
RTL = your_design.v
TB  = your_tb.v
TOP = your_tb_module

# VCS options
VCS = vcs
VCS_FLAGS = -full64 -sverilog -debug_access+all -kdb

# Verdi PLI (use absolute path)
VERDI_PLI = /opt/synopsys/verdi/V-2023.12/share/PLI/VCS/LINUX64

# Simulation executable
SIMV = simv

# Waveform file
WAVE = wave.fsdb

# Default target: compile and run
all: comp run

# Compile only
comp:
	$(VCS) $(VCS_FLAGS) -P $(VERDI_PLI)/novas.tab $(VERDI_PLI)/pli.a \
		-top $(TOP) $(RTL) $(TB) -o $(SIMV) -l compile.log

# Run simulation
run:
	./$(SIMV) -l sim.log

# Open Verdi
verdi:
	verdi -f filelist.f -ssf $(WAVE) &

# Clean generated files
clean:
	rm -rf simv* csrc* DVEfiles AN.DB *.fsdb *.log *.vpd *.key *.vdb ucli.key

# Deep clean
distclean: clean
	rm -rf *.fsdb

.PHONY: all comp run verdi clean distclean
```

### `filelist.f` — File list for Verdi

```
your_design.v
your_tb.v
```

### Testbench waveform dump template

Add to your testbench for dual output (FSDB + VCD):

```verilog
// FSDB for Verdi (full-featured debugging)
initial begin
    $fsdbDumpfile("wave.fsdb");
    $fsdbDumpvars(0, tb_module);
end

// VCD for VS Code VaporView (quick preview)
initial begin
    $dumpfile("wave.vcd");
    $dumpvars(0, tb_module);
end
```

## What to Modify Per Design

| File | What to change |
|------|---------------|
| `Makefile` | `RTL`, `TB`, `TOP` variables |
| `filelist.f` | List all design + testbench files |
| Testbench | Add waveform dump code with your module name |
| Design files | Upload your `.v` files |

## Workflow Commands

```bash
# Upload design files
scp *.v user@server:/project/vcs/

# Run simulation on server
ssh user@server "cd /project/vcs && make clean && make"

# Download VCD to local (VaporView doesn't support FSDB)
scp user@server:/project/vcs/wave.vcd ./

# Open in VS Code
code wave.vcd
```

## File Format Support

| Format | Tool | Use Case |
|--------|------|----------|
| FSDB | Verdi | Full debugging, complex analysis |
| VCD | VaporView | Quick preview, remote viewing |

## Common Issues & Fixes

| Issue | Cause | Fix |
|-------|-------|-----|
| No VCD generated | Missing `$dumpfile`/`$dumpvars` | Add VCD dump code to testbench |
| VaporView won't open FSDB | FSDB is proprietary Synopsys format | Use VCD for VaporView, FSDB for Verdi |
| Waveform empty | Simulation didn't run properly | Check sim.log for errors |
| Compile error: module not found | `TOP` doesn't match testbench module | Fix `TOP` in Makefile |
| `-top` flag error | Wrong module hierarchy | Ensure `TOP` matches the testbench module name |
| `novas.tab` not found | Wrong Verdi PLI path | Check VERDI_PLI path matches installed version |

## Tips

1. **Always copy from example template** — avoids Makefile syntax errors
2. **Dual output is safe** - FSDB and VCD can coexist without conflict
3. **VCD is portable** - Works with any VCD viewer, not just VaporView
4. **`-top` flag is important** — tells VCS which module is the top-level testbench
5. **Clean before recompile** — `make clean && make` avoids stale build artifacts

## Example Session

```bash
# Complete workflow
scp src/*.v user@mee1.ee.cityu.edu.hk:/home/user/project/vcs/ && \
ssh user@mee1.ee.cityu.edu.hk "cd /home/user/project/vcs && make clean && make" && \
scp user@mee1.ee.cityu.edu.hk:/home/user/project/vcs/wave.vcd ./ && \
code wave.vcd
```
