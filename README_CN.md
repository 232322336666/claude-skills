# vibe_circuit_deisgn.skill

Claude Code IC 设计自动化技能集合 — VCS 仿真、DC 综合、Formality 形式验证，以及基于 Agent Team 的 EDA 团队工作流。

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
- 自动崩溃恢复 — 从上次完成阶段继续
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
```

## 前置条件

- 远程 Linux 服务器，已安装并授权 Synopsys 工具（VCS、DC、Formality）
- TSMC 65nm 标准单元库（`tcbn65lp`）
- SSH 访问权限
- VS Code 安装 VaporView 扩展（用于本地查看波形）

## 快速开始

1. 准备好本地 RTL 设计文件
2. 调用 `eda-team-flow` 技能一键运行完整流程
3. 或使用单独技能（`remote-vcs-waveform-viewing`、`remote-dc-synthesis`、`remote-fm-formality`）逐步执行

## 模板参考

所有 EDA 技能均从远程服务器上验证过的 example 模板复制：

```
/home/research/zhengnafu2/agent_digits/example/
  ├── dc/    (DC 综合模板)
  ├── fm/    (Formality 模板)
  └── vcs/   (VCS 仿真模板)
```

这确保了工具配置文件的一致性，避免语法错误。

## 崩溃恢复机制

EDA 流程在每个阶段完成后自动写入记忆检查点：

| 阶段 | 检查点内容 | 恢复动作 |
|------|-----------|---------|
| VCS 完成后 | 测试结果、RTL 修改记录、波形状态 | 直接启动 DC |
| DC 完成后 | Slack、面积、功耗、输出文件状态 | 直接启动 FM |
| FM 完成后 | PASS/FAIL、匹配点数 | 输出最终汇总 |

如果会话崩溃，新会话自动读取检查点，从上次完成阶段继续，无需重复工作。

## 其他技能

| 技能 | 描述 |
|------|------|
| [deep-research](deep-research/) | 多源深度研究 |
| [memory-audit](memory-audit/) | 记忆质量审查 |
| [memory-evolution](memory-evolution/) | 记忆优化 |
| [memory-intake](memory-intake/) | 结构化记忆创建 |
| [planning-with-files](planning-with-files/) | 基于文件的规划 |
| [project-planner](project-planner/) | 项目规划 |
| [skill-creator](skill-creator/) | 创建和编辑技能 |
| [ssh](ssh/) | SSH 远程访问工具 |
| [find-skills](find-skills/) | 发现和安装技能 |

## License

MIT
