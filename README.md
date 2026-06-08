# Sentaurus TCAD 全流程 Agent Skill 【重大更新】正在实现新用户友好型skill，请稍后再下载新版本skill


[English README](README_EN.md) | 中文优先说明

这是一个面向 Claude Code、Claude.ai、OpenAI Codex、OpenCode、OpenClaw 等 Agent 环境的 **Sentaurus TCAD 仿真全流程 Skill**。它不是 Sentaurus 的替代品，也不包含 Synopsys 专有文件；它是一套让 AI Agent 更可靠地完成 TCAD 仿真的工作流说明和操作约束。

它的核心目标是让 Agent 不再“凭感觉写 deck 然后直接跑”，而是按科研工程闭环执行：

```text
问题定义 → 资料检索 → 官方文档/例子验证 → SWB 项目树 → SDE/SDevice/SVisual → gsub 提交 → 监控 → log/plt/tdr 诊断 → 可视化报告 → 经验沉淀 → 下一轮迭代
```

## 适用场景

当你希望 Agent 帮你做下面事情时，应该使用这个 skill：

- 创建或修复 Sentaurus Workbench 项目。
- 编写 SDE 结构、掺杂、接触、网格脚本。
- 编写 SDevice Physics、Math、Solve、Plot/Save 设置。
- 运行 Id-Vg、Id-Vd、BV、HeavyIon、SEB、TID、ESD 等仿真。
- 诊断收敛失败、`Step-size is too small`、异常漏电、击穿位置、SEB 判据。
- 做 GaN HEMT、p-GaN HEMT、辐照效应和可靠性仿真。
- 让仿真结果真正进入 SWB GUI，可视化 `.tdr/.plt`，并输出图表/报告。

## 最重要的规则

这个 skill 会强制 Agent 遵守以下规则：

1. **先查资料再仿真**  
   Physics 模型、材料参数、陷阱、极化、Avalanche、HeavyIon、热边界等必须有官方例子、文献或已验证经验依据。

2. **所有仿真必须在 SWB GUI 中可见**  
   新项目用 `swbpy2` 创建，已有项目用 `swbpy2` 添加参数和实验路径。不要在树外创建孤立 deck。

3. **默认使用 traditional SWB 项目**

   ```python
   deck = Deck(project_path, False)
   ```

4. **必须用 gsub 提交**

   ```bash
   gsub -q local:default -e <node-number> <project-path>
   ```

5. **禁止直接运行**

   ```bash
   sdevice pp*_des.cmd
   ```

   直接运行会绕过 SWB 状态管理，GUI 看不到结果，后续无法复现和追踪。

6. **提交后立即设置一次性后台监控**

   ```bash
   until grep -qE "Good Bye|FATAL|Step-size is too small" n<N>_des.log 2>/dev/null; do sleep 60; done
   tail -20 n<N>_des.log
   ```

7. **仿真结束必须看 `.plt` 和 `.tdr`**  
   `.plt` 告诉你曲线是否正确，`.tdr` 告诉你空间上哪里出问题。不能只看终端数字。

8. **必须输出持久化结果**  
   至少输出 `.png` 图、Markdown 表格或报告，并更新 `progress.md` / `findings.md`。

## 目录结构

```text
claude-sentaurus-skill/
├── SKILL.md                         # Skill 主入口：触发说明、流程、红线
├── references/
│   ├── swbpy2-gsub.md                # SWB、swbpy2、gsub、GUI 可见性
│   ├── sde-mesh-patterns.md          # SDE 几何、Boolean、接触、掺杂、网格
│   ├── sdevice-patterns.md           # SDevice Physics/Math/Solve/Plot/Save
│   ├── gan-hemt-and-seb.md           # GaN HEMT、BV、HeavyIon、SEB 方法论
│   └── results-reporting.md          # SVisual、plt/tdr 诊断、出图和报告
├── evals/
│   └── evals.json                    # 触发测试样例
├── dist/
│   └── sentaurus-tcad.skill          # 可导入的 skill 包
├── README.md                         # 中文说明
├── README_EN.md                      # English README
├── SECURITY.md                       # 安全说明
└── LICENSE                           # MIT License
```

## 安装方式

### 方式 A：Claude Code / 支持 `.skill` 包的环境

如果你的环境支持导入 `.skill` 文件，可直接使用：

```text
dist/sentaurus-tcad.skill
```

具体导入方式取决于你的 Agent 平台。

### 方式 B：手动安装到 Claude Code

```bash
mkdir -p ~/.claude/skills/sentaurus-tcad
cp SKILL.md ~/.claude/skills/sentaurus-tcad/
cp -r references ~/.claude/skills/sentaurus-tcad/
```

重启 Claude Code 或重新加载 skill 后即可使用。

### 方式 C：其他 Agent 环境

把 `SKILL.md` 和 `references/` 放到你的 Agent 支持的 skill / instruction / knowledge 目录中。关键是让 Agent 在执行 Sentaurus 相关任务时先读取 `SKILL.md`，再按需读取 `references/`。

## 使用前准备

你需要自己安装并授权使用 Synopsys Sentaurus TCAD。本仓库不包含任何 Sentaurus 软件、许可证、官方 PDF、官方示例或商业文件。

建议准备：

- 可用的 `swb`、`gsub`、`sdevice`、`svisual` 命令。
- Sentaurus Workbench 的 `STDB` 工作目录。
- 可用的 Sentaurus Python / `swbpy2` 环境。
- 官方 Applications Library 和 PDF 文档路径。
- 文献检索工具，例如 Zotero、机构订阅、机构网络或公开数据库。

在新机器上，先让 Agent 确认：

```bash
which swb gsub sdevice svisual
printf '%s\n' "$STROOT" "$STRELEASE" "$STDB"
```

## 典型提示词

### 从零建立 p-GaN HEMT 项目

```text
请用 sentaurus-tcad skill 帮我从零建立一个 p-GaN HEMT 的 SWB 项目。目标是先跑 IdVg 验证 Vth>1.2V，再跑 BV 到 900V。请先查官方例子和资料，再设计 SWB 树、SDE/SDevice 文件、gsub 提交和结果记录流程。
```

### 诊断 BV 收敛失败

```text
我的 GaN HEMT BV 节点在 720V 附近 Step-size is too small，Newton 最大误差坐标在 gate edge 附近。请不要盲改参数，按 log→plt→tdr→查资料→修复 的闭环流程诊断。
```

### HeavyIon / SEB 阈值扫描

```text
我要做 HeavyIon SEB 阈值扫描，固定 LET=0.8 pC/um，扫描多个 LoadVoltage 节点。请用 SWB/swbpy2 添加实验，gsub 提交，并输出曲线、表格和判据说明。
```

## 这个 skill 解决了哪些常见问题？

| 常见问题 | skill 的做法 |
|---|---|
| Agent 直接跑 `sdevice pp*_des.cmd` | 明确禁止，强制 `gsub`，保证 SWB GUI 可见 |
| 只看 log 不看结构 | 强制按 log → plt → tdr 分层诊断 |
| 参数和模型拍脑袋 | 强制先查官方例子、knowledge-rag、文献和资料 |
| 结果只在终端里说一句 | 要求输出 `.png`、Markdown 表格或报告 |
| 每轮仿真没有记录 | 要求维护 `task_plan.md`、`progress.md`、`findings.md` |
| SKILL.md 太长 | 主入口短小，细节放 references，按需读取 |

## 安全与合规

- 本仓库不包含 Synopsys Sentaurus 软件、许可证、官方 PDF 或官方示例文件。
- 使用者必须自行确认拥有合法 Sentaurus 许可。
- 仓库中的命令是工作流模板，不应在不了解项目路径和节点含义时盲目执行。
- 对任何会覆盖、删除或破坏项目的操作，应先备份并得到用户确认。
- 详见 [SECURITY.md](SECURITY.md)。

## 许可证

MIT License。详见 [LICENSE](LICENSE)。

## 贡献建议

欢迎提交 issue 或 PR，尤其是：

- 新器件体系 reference，例如 SiC、CMOS、photonic device。
- 更稳健的 SVisual 批量导图模板。
- 更多真实 eval prompt。
- 其他平台的安装说明，例如 Codex、OpenCode、OpenClaw。

维护原则：**主 `SKILL.md` 保持短小；新细节优先进入 `references/`。**
