# SDevice 模式：Physics / Math / Solve / Plot

## 文件块

Traditional 项目通常可直接引用上游节点输出；Hierarchical 项目必须使用绝对跨节点路径。

```tcl
File {
  Grid    = "@pwdout@/@nodedir|sde@/n@node|sde@_msh.tdr"
  Plot    = "n@node@_des.tdr"
  Current = "n@node@_des.plt"
  Output  = "n@node@_des.log"
}
```

## Physics 选择原则

先用最小物理模型跑通基线，再按目标加入模型。

| 目标 | 常见模型 | 注意 |
|---|---|---|
| MOS/Si 基线 | Fermi, Mobility(DopingDep Enormal HFS), SRH | eQuantumPotential 需要细界面网格 |
| GaN HEMT IV | Fermi, Thermionic, Piezoelectric_Polarization, IncompleteIonization, SRH/Radiative | 极化必须在界面设置 Activation |
| BV | eAvalanche, ElementVolumeAvalanche, AvalDensGradQF | GaN HEMT 不默认完整 Avalanche |
| 栅漏电/TAT | NonLocal + BarrierTunneling + traps | 非常慢，非必要不用 |
| 自热/SEB | Thermodynamic + Thermode + AnalyticTEP | 热边界要物理合理 |
| TID/SEE | Radiation/HeavyIon + traps | 先校准未辐照基线 |

## Math：GaN HEMT 默认强设置

复杂 GaN 结构不要用弱 Math 省时间。弱设置导致失败会更慢。

```tcl
Math {
  Extrapolate
  ExtendedPrecision(80)
  Iterations=100
  Digits=7
  ErrRef(electron)=1e9
  ErrRef(hole)=1e9
  RHSMax=1e18
  RHSMin=1e-12
  RHSFactor=1e30
  CDensityMin=1e-20
  Transient=BE
  DirectCurrentComputation
  ComputeDopingConcentration
  TensorGridAniso(Aniso)
  eMobilityAveraging=ElementEdge
  hMobilityAveraging=ElementEdge
  RefDens_eGradQuasiFermi_EparallelToInterface=1e8
  RefDens_hGradQuasiFermi_ElectricField_HFS=1e8
  RefDens_eGradQuasiFermi_ElectricField=1e8
  RefDens_hGradQuasiFermi_ElectricField=1e8
  Method=ILS(set=11)
}
```

BV 可额外加入：

```tcl
Math {
  ElementVolumeAvalanche
  AvalDensGradQF
}
```

Thermodynamic / 高残差瞬态若反复发散，可比较 ILS 与 Pardiso；Pardiso 更稳但慢且占内存。

## Extrapolate 陷阱

`-Extrapolate` 在命令文件解析阶段可能全局禁用外推，不能用它“只禁用某一段”。如果需要不同 Math 策略，优先分成 Save/Load 多个节点或多个 cmd。

## Solve 基本模式

```tcl
Solve {
  Coupled(Iterations=100 LineSearchDamping=1e-4) { Poisson }
  Coupled { Poisson Electron }
  Coupled { Poisson Electron Hole }

  Quasistationary(
    InitialStep=0.01 MinStep=1e-5 MaxStep=0.1
    Goal { Name="drain" Voltage=@Vd@ }
  ) { Coupled { Poisson Electron Hole } }

  NewCurrentPrefix="IdVg_"
  Quasistationary(
    DoZero InitialStep=1e-3 Increment=1.3
    MaxStep=0.02 MinStep=1e-6
    Goal { Name="gate" Voltage=@VgMax@ }
  ) { Coupled { Poisson Electron Hole }
      CurrentPlot(Time=(Range=(0 1) Intervals=50))
  }
}
```

## Plot / Save / CurrentPlot

必须保存可诊断快照。

```tcl
Plot {
  eDensity hDensity
  TotalCurrent/Vector eCurrent/Vector hCurrent/Vector
  ElectricField/Vector Potential SpaceCharge
  Doping DonorConcentration AcceptorConcentration
  BandGap ConductionBand ValenceBand
  eQuasiFermi hQuasiFermi
  SRH Auger ImpactIonization
  AvalancheGeneration HeavyIonChargeDensity
  LatticeTemperature
}
```

在 Solve 中保存关键点：

```tcl
Plot(FilePrefix="n@node@_Zero")
Transient(... Goal {Name=drain Voltage=@Vd@}) {
  Coupled { Poisson Electron Hole }
  Plot(FilePrefix="n@node@_Vd" Time=(Range=(0 1) Intervals=10) NoOverwrite)
}
Save(FilePrefix="drain_n@node@_@Vd@V")
```

| 命令 | 输出 | 用途 |
|---|---|---|
| `Plot` | `.tdr` | 空间分布诊断 |
| `Save` | `.sav` | Load 续跑/分叉 |
| `CurrentPlot` | `.plt` | 曲线 |
| `NewCurrentFile/Prefix` | 分段 `.plt` | 分离不同扫描 |

## Log 诊断

Newton 行中的 coordinate 是定位关键：

```text
C-norm_equation max_error vertex coordinate value
```

把最大误差坐标对应到 `.tdr` 中的高场/电流路径/界面位置。

## 常见收敛策略

| 症状 | 优先修正 |
|---|---|
| 初始 Poisson 不收敛 | LineSearchDamping=1e-4，检查接触/区域/掺杂 |
| drain ramp 高压卡住 | 减小 MaxStep，Save/Load 分段，看最大误差坐标 |
| BV avalanche 卡在固定电压 | 改 eAvalanche，检查 trap Add2TotalDoping |
| Thermodynamic HI 过慢 | 先用等温/粗网格定位，再开热模型；必要时 Pardiso |
| NonLocal 发散 | 确认确实需要；缩短范围、加细局部网格 |
