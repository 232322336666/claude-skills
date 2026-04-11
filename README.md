# vibe_circuit_deisgn.skill

**[:us: English](#english)** | **[:cn: 中文](#中文)**

---

<a id="english"></a>

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

### [remote-innovus-pr](remote-innovus-pr/) :new:

Run Cadence Innovus Place & Route on a remote server with TSMC 65nm libraries.

- Full flow: init -> floorplan -> placement -> CTS -> routing -> signoff
- Copy from verified example template (`innovus_flow/`)
- Auto-generate GDS, SDF, SPEF (WC/BC), post-P&R netlist
- Supports pure digital core and mixed-signal (analog + IO pad) designs
- Includes floorplan sizing guide, power stripe strategy, and common issue fixes

### [remote-fm-formality](remote-fm-formality/)

Run Synopsys Formality formal verification (RTL vs gate-level netlist equivalence check).

- Copy from verified example template
- Correct `set_top r:/WORK/${TOP}` syntax pre-configured
- Auto-discover RTL files with `ls` -- no manual file listing
- SVF guidance from DC synthesis

### [eda-team-flow](eda-team-flow/)

Serial team workflow: VCS -> DC -> FM with crash recovery via memory checkpoints.

- Serial execution ensures RTL consistency
- Memory checkpoint at each stage (VCS_DONE, DC_DONE, FM_DONE)
- Automatic crash recovery -- resume from last completed stage
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
                          |
                          v
                  +------------------+
                  | Innovus Agent    |
                  | Place & Route    |
                  | (GDS, SDF, SPEF) |
                  +------------------+
                       checkpoint 4
                       (physical layout)
```

## Prerequisites

- Remote Linux server with EDA tools licensed (VCS, DC, Formality, Innovus)
- TSMC 65nm standard cell library (`tcbn65lp`)
- SSH access to the server
- VS Code with VaporView extension (for local waveform viewing)

## Quick Start

1. Ensure your RTL design is ready locally
2. Invoke the `eda-team-flow` skill to run the full flow (VCS -> DC -> FM)
3. After FM passes, invoke `remote-innovus-pr` to run Place & Route
4. Or use individual skills step by step

## Template Reference

All EDA skills copy from verified example templates at:

```
~/agent_digits/example/
  ├── dc/            (DC synthesis template)
  ├── fm/            (Formality template)
  ├── vcs/           (VCS simulation template)
  └── innovus_flow/  (Innovus P&R template)
```

This ensures consistency and avoids syntax errors in tool configuration files.

## Other Skills

| Skill | Description |
|-------|-------------|
| [conf-papers](conf-papers/) | Top conference paper search (CVPR/ICLR/NeurIPS etc.) |
| [deep-research](deep-research/) | Multi-source deep research |
| [extract-paper-images](extract-paper-images/) | Extract figures from papers |
| [memory-audit](memory-audit/) | Memory quality review |
| [memory-evolution](memory-evolution/) | Memory optimization |
| [memory-intake](memory-intake/) | Structured memory creation |
| [paper-analyze](paper-analyze/) | Deep single-paper analysis with notes |
| [paper-search](paper-search/) | Search paper notes |
| [planning-with-files](planning-with-files/) | File-based planning |
| [project-planner](project-planner/) | Project planning |
| [skill-creator](skill-creator/) | Create and edit skills |
| [ssh](ssh/) | SSH remote access utilities |
| [start-my-day](start-my-day/) | Daily paper reading workflow |
| [find-skills](find-skills/) | Discover and install skills |

## License

MIT

---

**[Back to Top](#vibe_circuit_deisgnskill)** | **[Switch to 中文](#中文)**

---
---

<a id="中文"></a>

## EDA 核心技能

### [remote-vcs-waveform-viewing](remote-vcs-waveform-viewing/)

在远程服务器运行 VCS 仿真，本地 VS Code 通过 VaporView 查看波形。

- 从验证过的 example 模板复制，避免手写错误
- 自动生成 FSDB (Verdi) + VCD (VaporView) 双格式波形
- Makefile 驱动的编译和运行流程

### [remote-dc-synthesis](remote-dc-synthesis/)

在远程服务器运行 Synopsys Design Compiler 综合，使用 TSMC 65nm 标准单元库。

- 从验证过的 example 模板复制（`.synopsys_dc.setup`、`synth_top.tcl`、`Makefile`）
- 每个设计只需修改 `TOP_DESIGN_NAME` 和 `cons/top_cons.tcl`
- 内嵌完整的生产级验证脚本
- 报告分析：QoR、面积、时序（setup/hold）、功耗

### [remote-innovus-pr](remote-innovus-pr/) :new:

在远程服务器运行 Cadence Innovus 布局布线，使用 TSMC 65nm 库。

- 完整流程：初始化 -> 布局规划 -> 布局 -> 时钟树综合 -> 布线 -> 签收
- 从验证过的 example 模板复制（`innovus_flow/`）
- 自动生成 GDS、SDF、SPEF（WC/BC）、布局后网表
- 支持纯数字核心和混合信号（模拟 + IO pad）设计
- 包含布局规划尺寸指南、电源条带策略和常见问题修复

### [remote-fm-formality](remote-fm-formality/)

运行 Synopsys Formality 形式验证（RTL 与门级网表等价性检查）。

- 从验证过的 example 模板复制
- `set_top r:/WORK/${TOP}` 语法已预配置，避免常见错误
- RTL 文件自动发现，无需手动列文件
- 自动加载 DC 综合生成的 SVF 指导文件

### [eda-team-flow](eda-team-flow/)

串行团队工作流：VCS -> DC -> FM，带崩溃恢复记忆检查点。

- 串行执行确保 RTL 版本一致性
- 每阶段写入记忆检查点（VCS_DONE、DC_DONE、FM_DONE）
- 自动崩溃恢复 -- 从上次完成阶段继续
- 包含 Agent prompt 模板，直接启动团队

## 流程架构

```
RTL 设计 (.v)
      |
      v
+-------------+     +-------------+     +------------------+
| VCS Agent   | --> | DC Agent    | --> | FM Agent         |
| 功能仿真     |     | 逻辑综合     |     | 形式验证          |
+-------------+     +-------------+     +------------------+
      |                   |                     |
  检查点 1             检查点 2              检查点 3
  (wave.vcd)          (门级网表)            (PASS/FAIL)
                      (SVF, 报告)
                          |
                          v
                  +------------------+
                  | Innovus Agent    |
                  | 布局布线          |
                  | (GDS, SDF, SPEF) |
                  +------------------+
                       检查点 4
                       (物理版图)
```

## 前置条件

- 远程 Linux 服务器，已安装并授权 EDA 工具（VCS、DC、Formality、Innovus）
- TSMC 65nm 标准单元库（`tcbn65lp`）
- SSH 访问权限
- VS Code 安装 VaporView 扩展（用于本地查看波形）

## 快速开始

1. 准备好本地 RTL 设计文件
2. 调用 `eda-team-flow` 技能一键运行完整流程（VCS -> DC -> FM）
3. FM 通过后，调用 `remote-innovus-pr` 运行布局布线
4. 或使用单独技能逐步执行

## 模板参考

所有 EDA 技能均从远程服务器上验证过的 example 模板复制：

```
~/agent_digits/example/
  ├── dc/            (DC 综合模板)
  ├── fm/            (Formality 模板)
  ├── vcs/           (VCS 仿真模板)
  └── innovus_flow/  (Innovus 布局布线模板)
```

这确保了工具配置文件的一致性，避免语法错误。

## 崩溃恢复机制

EDA 流程在每个阶段完成后自动写入记忆检查点：

| 阶段 | 检查点内容 | 恢复动作 |
|------|-----------|---------|
| VCS 完成后 | 测试结果、RTL 修改记录、波形状态 | 直接启动 DC |
| DC 完成后 | Slack、面积、功耗、输出文件状态 | 直接启动 FM |
| FM 完成后 | PASS/FAIL、匹配点数 | 启动 Innovus P&R |
| Innovus 完成后 | GDS、SDF、SPEF、DRC 结果 | 输出最终汇总 |

如果会话崩溃，新会话自动读取检查点，从上次完成阶段继续，无需重复工作。

## 其他技能

| 技能 | 描述 |
|------|------|
| [conf-papers](conf-papers/) | 顶会论文搜索（CVPR/ICLR/NeurIPS 等） |
| [deep-research](deep-research/) | 多源深度研究 |
| [extract-paper-images](extract-paper-images/) | 从论文中提取图片 |
| [memory-audit](memory-audit/) | 记忆质量审查 |
| [memory-evolution](memory-evolution/) | 记忆优化 |
| [memory-intake](memory-intake/) | 结构化记忆创建 |
| [paper-analyze](paper-analyze/) | 深度单篇论文分析 |
| [paper-search](paper-search/) | 论文笔记搜索 |
| [planning-with-files](planning-with-files/) | 基于文件的规划 |
| [project-planner](project-planner/) | 项目规划 |
| [skill-creator](skill-creator/) | 创建和编辑技能 |
| [ssh](ssh/) | SSH 远程访问工具 |
| [start-my-day](start-my-day/) | 每日论文阅读工作流 |
| [find-skills](find-skills/) | 发现和安装技能 |

## License

MIT

---

**[Back to Top](#vibe_circuit_deisgnskill)** | **[Switch to English](#english)**
