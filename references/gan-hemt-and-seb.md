# GaN / p-GaN HEMT / BV / HeavyIon / SEB 参考

## 官方例子优先级

在 `Applications_Library/` 中优先查：

| 目标 | 例子 |
|---|---|
| p-GaN HEMT 基础 IV | `Power/GaN/HFET_pGate_GaN` |
| p-GaN HEMT 高级/TAT | `Power/GaN/pGate_HEMT` |
| 电流崩塌 | `Power/GaN/pGate_HEMT_CC` |
| Vth instability | `Power/GaN/pGate_HEMT_VtInstability` |
| 栅漏电 | `Power/GaN/pGate_Leakage`, `pGate_Barrier_TAT_1D` |
| 极化验证 | `Power/GaN/GaN_AlGaN_Polarization` |
| 自热 | `Power/GaN/GaN_HFET_SH` |
| 耗尽型 HFET | `Hetero/HFET_GaN_DC` |
| PiN 击穿 | `GettingStarted/GaN_PiN_Diode` |

不要凭记忆写模型；先对照同类例子。

## p-GaN HEMT 基本结构参数

常见层结构：p-GaN gate / AlGaN barrier / GaN channel / C-doped buffer / nucleation / substrate。设计参数必须来自文献、官方例子或已验证项目。

关键经验：
- 官方 p-GaN gate 常用 gate Workfunction 接近 5.8 eV。
- Mg 掺杂、p-GaN 厚度、barrier 厚度共同决定 Vth。
- 评估功率 p-GaN HEMT Ion 时，Vd=1 V 常处于线性区；需要根据目标选择 Vd=10 V 等饱和区条件。
- Mg Gaussian tail 不能穿透 barrier 杀死 2DEG；经验规则 `sMg ≤ tBarrier/5`。

## 极化

全局 `Piezoelectric_Polarization(strain)` 不等于真正激活界面极化。必须在 `RegionInterface` 或 `MaterialInterface` 设 Activation。

```tcl
Physics(RegionInterface="Barrier/Channel") {
  Piezoelectric_Polarization(Activation=0.6)
}
Physics(RegionInterface="Barrier.l/Channel") {
  Piezoelectric_Polarization(Activation=1.0)
}
```

根据结构区分栅下、access、drift 区。全部用同一 Activation 可能导致 Vth/Ion 失真。

## Buffer trap / BV

GaN HEMT BV 中，buffer trap 的 `Add2TotalDoping` 经常是稳定性的关键：

```tcl
Physics(Region="GaNBuffer") {
  Traps((Acceptor Level Conc=1e18 EnergyMid=0.9 FromValenceBand Add2TotalDoping))
}
```

对于 BV：
- 优先用 `eAvalanche`，避免完整 Avalanche 在高场低载流子区数值不稳。
- BV 判据要明确，例如 drain current 达到 1e-3 A/mm 或用户指定阈值。
- 保存多个电压点 `.tdr`，看 ElectricField 和 AvalancheGeneration 位置。

## HeavyIon / SEB 方法论

### 流程

1. 先完成 IdVg/IdVd 校准，确认器件基线合理。
2. 完成 BV，知道工作电压相对于 BV 的比例。
3. 选择 HeavyIon 入射位置：通常在高场区或最坏保护薄弱位置。
4. 固定 LET，扫描 LoadVoltage；或固定电压，扫描 LET。
5. 用瞬态漏极电流是否恢复、温升、步长坍缩和空间电流路径判断。

### LoadVoltage

经验上 SEB 测试电压常取 BV 的 50–80% 起步；若目标是阈值，逐步扫描直到从“恢复”变成“持续上升”。

### Drain ramp

常规方案用 Transient ramp 到 LoadVoltage，然后 HeavyIon 时间略晚于 ramp 结束：

```tcl
Transient(
  InitialTime=0 FinalTime=1
  InitialStep=0.05 MinStep=1e-8 MaxStep=0.5
  Goal {Name=drain Voltage=@LoadVoltage@}
) { Coupled {Poisson Electron Hole} }
Save(FilePrefix="drain_n@node@_@LoadVoltage@V")
```

HeavyIon time 可设为 `1.001`，避免和 ramp 边界重合。

复杂多极化界面结构中，Transient drain ramp 可能因时间导数项带来大 RHS；可用 Quasistationary ramp + HI 阶段 Transient。此时要谨慎使用 Extrapolate，并考虑 `Notdamped=5` 或 Pardiso。

### SEB 判据

| 证据 | 解释 |
|---|---|
| drain current 持续上升不恢复 | SEB/正反馈 |
| peak current 比 baseline 高 >1e3（按项目定义） | 强瞬态导通 |
| LatticeTemperature 上升 >200 K（若用户要求热判据） | 热失控证据 |
| step size 坍缩到极小值 | 数值上进入强非线性/失控区 |
| `.tdr` 中 TotalCurrent/Vector 形成贯通路径 | 空间证据 |
| Avalanche/ImpactIonization 集中在高场区 | 机理证据 |

不要只用单个数字判定；至少结合 `.plt` 曲线和 `.tdr` 空间分布。

## 2DHG / p-island / AlN cap 物理约束

- 2DHG 的优势是极化场和局域空穴束缚，不是高横向迁移率。
- 室温 2DHG mobility 不要随意写成 100–300 cm²/Vs；常见更低，需要文献依据。
- AlGaN 组分梯度、深埋层源极连接、MOCVD 再生长氢钝化等工艺假设必须查文献后再写。

## SEB 闭环记录模板

```markdown
### Node N: HI @ LET=?, LoadVoltage=?, location=?
- 前置 BV: ? V；测试电压比例: ?% BV
- log: Good Bye / Step-size / FATAL；关键行: ...
- plt: baseline Id=?, peak Id=?, 是否恢复=?
- tdr: 高场/电流路径/温度峰值位置=...
- 判定: Survive / SEB / 不确定
- 下一步: 降/升 LoadVoltage、换位置、查资料、改模型...
```
