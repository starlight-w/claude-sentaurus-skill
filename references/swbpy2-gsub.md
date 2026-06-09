# SWB / swbpy2 / gsub 工作流

## 目标
保证所有仿真都在 Sentaurus Workbench GUI 中可见、可刷新、可复现。

## 路径与 Python

- swbpy2 需要 TCAD 自带 Python 或二进制兼容环境。
- 常见 TCAD Python：`$STROOT/tcad/$STRELEASE/linux64/bin/python3.11`
- 项目通常放在 `STDB` 下，例如 `$STDB/<project>`。

## swbpy2 smoke test

新设备首次运行前，先确认 `swbpy2` 能在 TCAD Python 中导入：

```bash
"$STROOT/tcad/$STRELEASE/linux64/bin/python3.11" - <<'PY'
from swbpy2 import *
print("swbpy2 import ok")
PY
```

如果失败，先修复 Sentaurus 环境或 Python 路径；不要开始创建 SWB 项目。

## 创建 traditional 项目

```python
from swbpy2 import *
from pathlib import Path

project = "$STDB/my_project"
deck = Deck(project, False)  # False = traditional organization
tree = deck.getGtree()

tree.AddTool("sde", "sde", 0)
tree.AddTool("IdVg", "sdevice", 1)
tree.AddTool("BV", "sdevice", 2)
tree.AddTool("vis", "svisual", 3)

step = tree.ToolSteps("sde")[-1] + 1
tree.AddParam("Lgd", "10", step)
step = tree.ToolSteps("IdVg")[-1] + 1
tree.AddParam("VgMax", "6", step)

# pvalues 顺序必须等于 tree.AllPnames()
tree.AddPath(pvalues=["10", "6"])
deck.save()
```

## 参数/实验规则

| 操作 | 正确做法 | 常见错误 |
|---|---|---|
| 添加参数 | `step = tree.ToolSteps(tool)[-1] + 1` | 硬编码 step，导致参数挂错工具 |
| 添加实验 | `tree.AddPath(pvalues=[...])`，长度等于 `len(AllPnames())` | 只给变更参数，导致路径错误 |
| 改已有实验值 | `ChangeParamValues(name, new_values)` | `SetPdefaultValue` 只改默认值，不改已有节点 |
| 运行节点 | `deck.run(expr="55 58")` 或 gsub | 运行 virtual node |
| 共享上游 | 只有后级参数不同会共享 SDE/IdVg 节点 | 重复创建不必要结构节点 |

## 节点发现

不要猜节点号。创建或修改实验树后，用 `swbpy2` 列出参数和节点关系，再选择 leaf/executable node 运行。不同 Sentaurus 版本的 API 显示函数可能略有差异；原则是读取 `gtree` 中的节点、工具标签和参数值，并避开 virtual nodes。

记录提交前至少写清：

| 项 | 内容 |
|---|---|
| 项目路径 | `$STDB/<project>` |
| 节点号 | 纯数字，例如 `63` |
| 上游结构节点 | SDE/mesh 节点号 |
| 工具 | IdVg / BV / HeavyIon 等 |
| 参数值 | `tree.AllPnames()` 顺序对应的值 |

## gsub 队列检查

默认使用 `local:default`：

```bash
gsub -q local:default -e <node> <project>
```

新设备上队列名可能不同。首次提交前用该环境支持的队列查询命令或本地文档确认 `local:default` 是否存在；若不存在，必须让用户确认队列名，不要自动换队列。

## gsub 提交

标准命令：

```bash
gsub -q local:default -e 63 $STDB/project_name
```

规则：
- `-e` 后是纯数字节点号，不写 `n63` 或 `node63`。
- 项目路径是最后一个位置参数，不能省略。
- gsub 自动预处理，不必先跑 `spp`。
- 多节点：`gsub -q local:default -e "55 58 60" .`
- 需要详细调试时：`gsub -verbose -q local:default -e 63 .`

## 队列追踪辅助工具

在当前工作区可用轻量队列记录脚本：

```bash
python3 scripts/sentaurus/sim_queue.py add <node> <project> "description"
python3 scripts/sentaurus/sim_queue.py done <node> "result"
python3 scripts/sentaurus/sim_queue.py fail <node> "reason"
python3 scripts/sentaurus/sim_queue.py status
```

规则：
- 该脚本只记录提交/完成/失败，状态默认写入 `claude_tmp/sentaurus/sim_queue.json`。
- 它不替代 SWB `.sta/.job` 状态，也不替代 `gsub`。
- 提交 `gsub` 后可立即 `add`；后台等待看到 `Good Bye` 后 `done`，看到 `FATAL` 或 `Step-size is too small` 后按实际情况 `fail` 或记录为需诊断。

## 为什么禁止直接 sdevice

| 项目 | gsub | 直接 sdevice |
|---|---|---|
| 自动预处理 | 是 | 否 |
| 更新 `.sta` | 是 | 否 |
| 创建 `.job` | 是 | 否 |
| SWB GUI 可见 | 是 | 否 |
| 依赖检查 | 是 | 否 |
| 结果可追踪 | 是 | 弱 |

直接跑 `sdevice pp*_des.cmd` 即使产生数值结果，也绕过 SWB 状态管理，用户在 GUI 中看不到，视为无效交付。

## 监控模板

提交后立即运行后台等待。必须在项目目录中运行，或使用绝对 log 路径，避免监控到错误文件：

```bash
until grep -qE "Good Bye|FATAL|Step-size is too small" /path/to/n<N>_des.log 2>/dev/null; do sleep 60; done
tail -20 /path/to/n<N>_des.log
```

不要 grep `Error`，不要 pgrep，不要持续轮询。若 log 一直没有生成，先回到 preflight 检查 PATH、license、STDB、队列和项目路径。

## GUI 刷新

- GUI 已打开：按 F5 或 `deck.reload()`。
- GUI 未打开：打开 SWB 后载入项目，读取 `.sta` 和输出文件。
- 如果外部修改了工具 cmd 或 gtree，保存后 reload。
