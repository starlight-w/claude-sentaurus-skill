# 新设备 Preflight：首次运行前环境体检

## 目标

在一台新设备、新服务器或新用户环境上，不要直接写 deck 或提交 `gsub`。先确认 Sentaurus 环境、许可证、SWB 工作目录、`swbpy2`、队列和可视化路径都可用。

Preflight 未通过时，停止仿真计划，向用户报告阻塞项。不要把环境问题误判成 SDE/SDevice/Physics 问题。

## 不做的事

- 不安装或破解 Synopsys Sentaurus。
- 不读取、记录或要求用户提供 license server、账号、密码、token、cookie。
- 不把 proprietary manual、Applications Library 项目、license 文件复制进 skill 仓库。
- 不在未知项目目录中删除、覆盖或移动文件。

## 1. 命令与环境变量

先检查命令是否在当前 shell 中可见：

```bash
which swb gsub sdevice svisual
printf 'STROOT=%s\nSTRELEASE=%s\nSTDB=%s\n' "$STROOT" "$STRELEASE" "$STDB"
```

判定：

| 结果 | 处理 |
|---|---|
| `which` 找不到 Sentaurus 命令 | 询问用户如何加载 Sentaurus 环境，例如 module/load 脚本；不要猜安装路径 |
| `STROOT/STRELEASE/STDB` 为空 | 让用户提供或加载环境；不要硬编码个人路径 |
| 命令和变量都存在 | 继续下一步 |

如果用户需要自己加载环境，建议他们在 Claude Code 中输入 `! <command>`，这样输出会回到会话中，例如：

```text
! module load sentaurus
```

## 2. STDB 与项目目录

检查 `STDB` 是否存在且可写：

```bash
test -n "$STDB" && test -d "$STDB" && test -w "$STDB"
```

规则：

- 新 SWB 项目默认放在 `$STDB/<project_name>`。
- 如果用户指定别的项目路径，先确认它是 SWB 可管理目录，并且用户确实希望这样做。
- 不要在临时目录、下载目录或 skill 仓库里创建 Sentaurus 项目。
- 如果目录不可写，停止并让用户修复权限或指定可写 STDB。

## 3. License 可用性

命令在 PATH 中不代表 license 可用。首次运行前必须确认 Sentaurus 工具能正常启动到非破坏性查询状态，例如版本/help 查询或用户环境已有的 license 检查方式。

原则：

- 若出现 license checkout failed、license server unreachable、feature not available 等信息，停止仿真计划。
- 不要修改 deck、Physics、Math 来“修” license 问题。
- 不要让用户把 license server、账号密码或 token 粘贴给 Agent。
- 只记录“license 不可用/功能不可用”这一事实，不记录敏感配置。

## 4. TCAD Python 与 swbpy2 smoke test

`swbpy2` 通常需要 TCAD 自带 Python 或兼容环境。优先用 `$STROOT/tcad/$STRELEASE/linux64/bin/python3.11`：

```bash
"$STROOT/tcad/$STRELEASE/linux64/bin/python3.11" - <<'PY'
from swbpy2 import *
print("swbpy2 import ok")
PY
```

如果失败：

| 现象 | 处理 |
|---|---|
| Python 路径不存在 | 检查 Sentaurus 版本目录；让用户提供实际 TCAD Python 路径 |
| `ModuleNotFoundError: swbpy2` | 说明没有在正确 TCAD Python 环境中运行 |
| import 成功 | 可以使用 `Deck(project, False)` 管理 SWB 树 |

## 5. gsub 队列

默认命令是：

```bash
gsub -q local:default -e <node> <project>
```

但新设备上 `local:default` 可能不存在。首次提交前检查本机/服务器队列配置。可用 `gqueues`、`gsub -help` 或该环境文档确认队列名。

规则：

- 如果 `local:default` 不存在，不要自动换成未知队列。
- 让用户确认队列名和并发策略。
- 在本项目规则下，全系统最多同时 3 个 `sdevice` 节点；提交前仍要检查：

```bash
ps aux | grep sdevice | grep -v grep
```

## 6. SVisual 与显示环境

完整交付必须看 `.tdr` 空间分布。首次运行前检查是否有 GUI/display：

```bash
printf 'DISPLAY=%s\nWAYLAND_DISPLAY=%s\n' "$DISPLAY" "$WAYLAND_DISPLAY"
which svisual
```

判定：

| 环境 | 处理 |
|---|---|
| 本地图形桌面或 X11 转发可用 | 可用 `svisual n<N>_des.tdr n<N>_des.plt &` 打开 |
| 无 DISPLAY 的服务器 | `.plt` 可用 Python 出曲线；`.tdr` 空间诊断必须标记为待用户在 GUI/SVisual 中查看 |
| `svisual` 不存在 | 不能声称完成 `.tdr` 诊断；请求用户补充可视化环境 |

无头环境降级不是完整替代：曲线图可以先做，空间机理结论必须等 `.tdr` 被查看后再下最终判断。

## 7. 官方资料与例子

检查是否能访问官方资料路径：

```bash
test -d "$STROOT/tcad/$STRELEASE/Applications_Library"
test -d "$STROOT/tcad/$STRELEASE/manuals/olh_sentaurus/pdf"
```

如果路径不存在：

- 不要声称已经对照官方例子。
- 可使用用户提供的本地例子、公开文献、已有项目经验替代，但要明确依据来源。
- 需要专有 manual 或 Applications Library 时，让用户在有权限的环境中查看或提供允许分享的摘要。

## 8. 环境错误 vs deck 错误

| 症状 | 类型 | 下一步 |
|---|---|---|
| `command not found: swb/gsub/sdevice` | 环境错误 | 加载 Sentaurus 环境或让用户提供路径 |
| license checkout failed | 环境/许可错误 | 停止，用户修复 license |
| `$STDB` 为空或不可写 | 环境/权限错误 | 用户设置 STDB 或修复权限 |
| `from swbpy2 import *` 失败 | 环境错误 | 使用 TCAD Python 或修复安装 |
| `local:default` 队列不存在 | 队列配置错误 | 用户确认队列名 |
| `svisual` 无法显示 | 可视化阻塞 | 先做 `.plt`，`.tdr` 等 GUI 补查 |
| `Step-size is too small` 且 log/plt/tdr 可读 | 数值/模型问题 | 进入正常闭环诊断 |
| Newton 最大误差在器件内部特定坐标 | 数值/物理问题 | 对照 `.tdr` 空间分布与模型设置 |

## 9. Preflight 结果记录模板

```markdown
## New-device preflight

| 项 | 状态 | 证据/输出 | 处理 |
|---|---|---|---|
| Sentaurus commands | pass/fail | `which ...` | ... |
| STROOT/STRELEASE/STDB | pass/fail | values present? | ... |
| STDB writable | pass/fail | test result | ... |
| license availability | pass/fail/unknown | non-sensitive summary | ... |
| TCAD Python + swbpy2 | pass/fail | import ok? | ... |
| gsub queue | pass/fail/unknown | queue name | ... |
| SVisual/display | pass/degraded/fail | DISPLAY/svisual | ... |
| manuals/examples | pass/degraded | paths exist? | ... |

Decision: proceed / blocked / degraded proceed.
```

只有 `Decision: proceed` 或用户明确接受 `degraded proceed` 时，才进入 SWB 项目创建和 `gsub` 提交流程。
