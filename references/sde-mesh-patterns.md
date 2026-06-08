# SDE 与网格模式

## SDE 修改后的强制验证

每次修改 SDE 后，先写出一张坐标表，再提交结构节点：

| 项 | 坐标/尺寸 | 验证点 |
|---|---|---|
| 垂直层叠 | substrate / nucleation / buffer / channel / barrier / cap / passivation | 无反向厚度、无未预期重叠 |
| 横向结构 | Lsg / Lg / Lgd / field plate / island / via | 不越界、不穿入不该覆盖区域 |
| 接触 | source/drain/gate/bulk/thermal | contact 名不等于 region 名，位置在边/面上 |
| 掺杂窗口 | p-GaN、p-island、S/D、trap 区域 | window 在目标材料内，sigma/Depth 不穿透关键界面 |
| 网格窗口 | channel、高场区、HI 轨迹、界面 | 关键区足够细，非关键区不过细 |

## Boolean 与区域规则

- `ABA`：新区域切割旧区域，新建体优先。嵌入 cap/island 常用。
- `BAB`：旧区域切割新区域，旧体优先。
- 同材料相邻体会自动合并；如果需要“同材料岛”独立行为，用 doping window 或改变材料/region 设计。
- 不要在嵌入层之后再创建同材料大块填充层，否则可能把嵌入层切掉。

## 接触创建模式

推荐“金属占位体 → 接触边界 → 删除占位体”：

```scheme
(define WGate (sdegeo:create-rectangle
  (position x1 y1 0) (position x2 y2 0) "Tungsten" "tmp_gate"))
(sdegeo:define-contact-set "gate" 4 (color:rgb 1 0 0) "##")
(sdegeo:set-current-contact-set "gate")
(sdegeo:set-contact-boundary-edges WGate)
(sdegeo:delete-region WGate)
```

简单结构也可 `find-edge-id`，但 position 必须在边上，通常取边中点。

## 掺杂

常用模式：

```scheme
; Constant in material
(sdedr:define-constant-profile "prof" "BoronActiveConcentration" 1e17)
(sdedr:define-constant-profile-material "place" "prof" "Silicon")

; Window placement
(sdedr:define-refeval-window "win" "Rectangle" (position x1 y1 0) (position x2 y2 0))
(sdedr:define-constant-profile-placement "place" "prof" "win")

; Gaussian profile: 关键字是 PeakPos / PeakVal
(sdedr:define-gaussian-profile "mg_gauss"
  "MagnesiumActiveConcentration" "PeakPos" 0 "PeakVal" 2e19
  "ValueAtDepth" 1e17 "Depth" 0.005 "Gauss" "Factor" 0.8)
```

p-GaN HEMT 中 Mg Gaussian sigma 是高风险参数：sigma 过大会穿透 barrier 补偿 2DEG。经验规则：`sMg ≤ tBarrier/5`。

## 分级网格策略

原则：关键区域细，非关键区域粗；先保证物理量解析，再控制节点数。

| 区域 | 细化建议 | 理由 |
|---|---|---|
| Barrier / channel interface | 垂直 0.001–0.003 µm | 2DEG/极化电荷 |
| Gate edge / drain-side high field | 横向 0.02–0.1 µm | BV/SEB 高场定位 |
| HeavyIon track | track 周围局部细化 | 瞬态电荷密度梯度 |
| p-GaN / AlN cap / 2DHG interface | 垂直 ≤ cap 厚度的 1/3 | 2DHG/极化阱 |
| Buffer/substrate | 0.1–0.5 µm | 非关键体区，避免过慢 |

避免全局 `MaxLenInt "GaN" "AlGaN"`，它会细化所有 GaN/AlGaN 界面，包括无用 buffer 界面，导致节点数爆炸。优先 region-based 或 window-based refinement。

## build mesh

```scheme
(sde:build-mesh "n@node@")
; 或需要自动界面节点：
(sde:build-mesh "-AI" "n@node@_msh")
```

## 常见失败诊断

| 现象 | 可能原因 | 检查 |
|---|---|---|
| SDE 报找不到 edge/vertex | 坐标不在边/点上，或 boolean 改变了拓扑 | 用中点，重算坐标 |
| 网格节点暴涨 | 全局 MaxLenInt、过细 global window | 改局部细化 |
| 2DEG 消失/Ion 为 0 | Mg tail 穿透 barrier、极化 Activation 错 | 看 eDensity/SpaceCharge |
| p 岛不独立 | 同材料合并 | 改 window doping 或材料/region 设计 |
