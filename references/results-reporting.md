# 结果可视化、报告与经验沉淀

## 完成仿真后的必做项

1. 读取 `tail -20 n<N>_des.log` 判断终止状态。
2. 打开 `.plt` 看曲线，提取关键指标。
3. 打开 `.tdr` 看空间分布，定位电场、电流、载流子、温度、Avalanche/HeavyIon。
4. 导出至少一个持久化结果：`.png` 图、`.md` 表格或报告。
5. 更新 `progress.md` 与 `findings.md`。
6. 若是用户可读报告/论文/专利说明，最后做自然化润色。

## SVisual 打开

先确认显示环境：

```bash
printf 'DISPLAY=%s\nWAYLAND_DISPLAY=%s\n' "$DISPLAY" "$WAYLAND_DISPLAY"
which svisual
```

有 GUI/display 时：

```bash
svisual n<N>_des.tdr n<N>_des.plt &
```

批处理导图可用 Tcl，但仍通常需要显示环境。无显示环境时可用 Python 解析 `.plt` 后用 matplotlib 出图，但这不是 `.tdr` 空间诊断的完整替代：必须在记录中标明 `.tdr` 尚未查看，并请用户在有 GUI 的 Sentaurus Visual 中打开或提供截图后再下空间机理结论。

### 无头环境降级记录

```markdown
### 可视化状态
- `.plt` 曲线：已用 Python/表格分析，图表路径：...
- `.tdr` 空间分布：未完成 / 用户截图已检查 / SVisual 已打开
- 限制：当前环境无 DISPLAY，不能声称已完成空间诊断
```

### SVisual Tcl 导图占位模板

不同 Sentaurus 版本和用户布局的 Tcl 命令可能不同。若要批量导图，先在 GUI 中录制或验证 Tcl，再放入项目脚本；不要在未验证的情况下声称已导出 `.tdr` 空间图。

## `.plt` 曲线检查

| 仿真 | 曲线 | 指标 |
|---|---|---|
| IdVg | drain TotalCurrent vs gate OuterVoltage | Vth, Ion, Ioff, gm |
| IdVd | drain TotalCurrent vs drain Voltage | Ron, saturation, knee |
| BV | drain TotalCurrent vs drain InnerVoltage | BV, leakage, breakdown onset |
| HeavyIon | drain TotalCurrent / LatticeTemperature vs time | peak, recovery, SEB |
| TID | before/after IdVg | Vth shift, gm degradation |

## `.tdr` 空间诊断顺序

1. `TotalCurrent/Vector`：电流路径是否合理。
2. `ElectricField/Vector`：高场区在哪。
3. `eDensity/hDensity`：2DEG/2DHG、耗尽、注入。
4. `SpaceCharge/Potential`：极化与耗尽区。
5. `AvalancheGeneration/ImpactIonization`：击穿机制。
6. `HeavyIonChargeDensity`：离子径迹。
7. `LatticeTemperature`：热失控位置。

## 出图规范

科研图默认：
- DPI ≥ 300。
- 字号 ≥ 12 pt。
- 线宽 1.5–2 pt。
- 使用 colorblind-safe 配色，避免 matplotlib 默认循环色。
- 坐标轴单位清楚，例如 `I_D (mA/mm)`、`V_G (V)`、`Time (s)`。
- 专利/线性对比图如用户要求线性轴，不要用 log scale。

## 进度记录模板

```markdown
## YYYY-MM-DD Session: <主题>

### 已执行
- 项目: `<project_path>`
- 修改: SDE/SDevice/参数/网格/Physics
- 提交: `gsub -q local:default -e <node> <project>`
- 监控: Good Bye / FATAL / Step-size is too small

### 结果
| 节点 | 参数 | log | 关键指标 | 判定 | 图/表 |
|---|---|---|---|---|---|
| N | ... | Good Bye | Vth=?, BV=?, PeakId=? | pass/fail | path/to.png |

### 诊断
- `.plt`: ...
- `.tdr`: ...
- 根因判断: ...

### 下一步
- ...
```

## findings 记录模板

```markdown
## 发现：<一句话>
- 来源：文献/官方例子/仿真节点/knowledge-rag
- 证据：...
- 适用范围：...
- 不适用/风险：...
- 后续规则：...
```

## 经验沉淀

- 可复用规则写入 memory 或 Obsidian/项目文档。
- 外部文献 PDF 优先进入 Zotero；Sentaurus 用法、课程、例子库、网页资料可索引到 knowledge-rag 或写成 memory。
- 不要把一次失败隐去；失败记录能防止未来重复踩坑。
