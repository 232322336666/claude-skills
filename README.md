# vibe_circuit_deisgn.skill

A collection of Claude Code skills for IC design automation — VCS simulation, DC synthesis, Formality verification, and team-based EDA workflows on remote Linux servers.

## EDA Skills

### [remote-vcs-waveform-viewing](remote-vcs-waveform-viewing/)

Run VCS simulations on a remote server and view waveforms locally in VS Code with VaporView.

- Copy from verified example template
- Auto-generate FSDB (Verdi) + VCD (VaporView) dual waveforms
- Makefile-driven compile and run

### [remote-dc-synthesis](remote-dc-synthesis/)

Run Synopsys Design Compiler synthesis on a remote server with TSMC 65nm standard cell library.

- Copy from verified example template (`.synopsys_dc.setup`, `synth_top.tcl`, `Makefile`)
- Only modify `TOP_DESIGN_NAME` and `cons/top_cons.tcl` per design
- Embedded complete verified scripts from production template
- Report analysis: QoR, area, timing (setup/hold), power

### [remote-fm-formality](remote-fm-formality/)

Run Synopsys Formality formal verification (RTL vs gate-level netlist equivalence check).

- Copy from verified example template
- Correct `set_top r:/WORK/${TOP}` syntax pre-configured
- Auto-discover RTL files with `ls` — no manual file listing
- SVF guidance from DC synthesis

### [eda-team-flow](eda-team-flow/)

Serial team workflow: VCS -> DC -> FM with crash recovery via memory checkpoints.

- Serial execution ensures RTL consistency
- Memory checkpoint at each stage (VCS_DONE, DC_DONE, FM_DONE)
- Automatic crash recovery — resume from last completed stage
- Agent prompt templates for team-based execution

## Architecture

```
RTL Design (.v)
      |
      v
+-------------+     +-------------+     +------------------+
| VCS Agent   | --> | DC Agent    | --> | FM Agent         |
| Simulation  |     | Synthesis   |     | Formal Verify    |
+-------------+     +-------------+     +------------------+
      |                   |                     |
  checkpoint 1        checkpoint 2          checkpoint 3
  (wave.vcd)          (gate netlist)        (PASS/FAIL)
                      (SVF, reports)
```

## Prerequisites

- Remote Linux server with Synopsys tools (VCS, DC, Formality) licensed
- TSMC 65nm standard cell library (`tcbn65lp`)
- SSH access to the server
- VS Code with VaporView extension (for local waveform viewing)

## Quick Start

1. Ensure your RTL design is ready locally
2. Invoke the `eda-team-flow` skill to run the full flow
3. Or use individual skills (`remote-vcs-waveform-viewing`, `remote-dc-synthesis`, `remote-fm-formality`) step by step

## Template Reference

All EDA skills copy from the verified example template at:

```
/home/research/zhengnafu2/agent_digits/example/
  ├── dc/    (DC synthesis template)
  ├── fm/    (Formality template)
  └── vcs/   (VCS simulation template)
```

This ensures consistency and avoids syntax errors in tool configuration files.

## Other Skills

| Skill | Description |
|-------|-------------|
| [deep-research](deep-research/) | Multi-source deep research |
| [memory-audit](memory-audit/) | Memory quality review |
| [memory-evolution](memory-evolution/) | Memory optimization |
| [memory-intake](memory-intake/) | Structured memory creation |
| [planning-with-files](planning-with-files/) | File-based planning |
| [project-planner](project-planner/) | Project planning |
| [skill-creator](skill-creator/) | Create and edit skills |
| [ssh](ssh/) | SSH remote access utilities |
| [find-skills](find-skills/) | Discover and install skills |

## License

MIT
