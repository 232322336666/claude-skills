---
name: remote-vcs-waveform-viewing
description: Use when developing Verilog/SystemVerilog on a remote Linux server with VCS and viewing waveforms locally in VS Code with VaporView, without local EDA tools
---

# Remote VCS Simulation with Local Waveform Viewing

## Overview

Run VCS simulations on a remote Linux server and view waveforms locally in VS Code using VaporView plugin. Enables hardware development without local EDA tools.

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

## Quick Reference

| Step | Command |
|------|---------|
| Install VaporView | `code --install-extension lramseyer.vaporview` |
| Upload files | `scp local/*.v user@server:/path/` |
| Run simulation | `ssh user@server "cd /path && make"` |
| Download VCD | `scp user@server:/path/wave.vcd ./` |
| Open waveform | `code wave.vcd` |

## Implementation

### 1. Testbench with Dual Waveform Output

Add both FSDB (Verdi) and VCD (VaporView) outputs:

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

### 2. Makefile for VCS

```makefile
VCS = vcs
VCS_FLAGS = -full64 -sverilog -debug_access+all -kdb
VERDI_PLI = /opt/synopsys/verdi/V-2023.12/share/PLI/VCS/LINUX64

all: comp run

comp:
	$(VCS) $(VCS_FLAGS) -P $(VERDI_PLI)/novas.tab $(VERDI_PLI)/pli.a \
		-top $(TOP) $(RTL) $(TB) -o simv -l compile.log

run:
	./simv -l sim.log

clean:
	rm -rf simv* csrc* *.log *.fsdb *.vcd *.vdb
```

### 3. Workflow Commands

```bash
# Upload design files
scp *.v user@server:/project/path/

# Run simulation on server
ssh user@server "cd /project/path && make clean && make"

# Download VCD to local (VaporView doesn't support FSDB)
scp user@server:/project/path/wave.vcd ./

# Open in VS Code
code wave.vcd
```

## File Format Support

| Format | Tool | Use Case |
|--------|------|----------|
| FSDB | Verdi | Full debugging, complex analysis |
| VCD | VaporView | Quick preview, remote viewing |
| FST | VaporView | Compressed VCD (faster loading) |

## Common Mistakes

| Issue | Cause | Fix |
|-------|-------|-----|
| No VCD generated | Missing `$dumpfile`/`$dumpvars` | Add VCD dump code to testbench |
| VaporView won't open FSDB | FSDB is proprietary Synopsys format | Use VCD for VaporView, FSDB for Verdi |
| Waveform empty | Simulation didn't run properly | Check sim.log for errors |
| SCP connection timeout | Server firewall/VPN | Use VS Code Remote-SSH instead |

## Tips

1. **Dual output is safe** - FSDB and VCD can coexist without conflict
2. **VCD is portable** - Works with any VCD viewer, not just VaporView
3. **Compress for large waves** - Use FST format if VCD is too large
4. **Automate with script** - Create a shell script for upload-sim-download cycle

## Example Session

```bash
# Complete workflow in one command sequence
scp src/*.v user@mee1.ee.cityu.edu.hk:/home/user/project/ && \
ssh user@mee1.ee.cityu.edu.hk "cd /home/user/project && make" && \
scp user@mee1.ee.cityu.edu.hk:/home/user/project/wave.vcd ./ && \
code wave.vcd
```
