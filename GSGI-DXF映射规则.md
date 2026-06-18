# GSGI — DXF 映射规则

本文档定义 GSGI 1.0 与 DXF 之间的双向映射规则。

> 版本：v1.0
> 对应 GSGI 实体类型：25 种 + 4 种描述手段

---

## 1. 全局映射约定

| 项 | 规则 |
|----|------|
| 坐标 | GSGI 中所有 `*_ref` 字段引用 point 实体，映射时需先解析 point 链得到 WCS 坐标再写入 DXF |
| 颜色 | ACI 索引直接映射；Hex RGB 转换为 DXF 真彩色组码（`420`）；ByLayer/ByBlock 映射为 `256`/`0` |
| 线型 | 字符串线型名直接写入 DXF `6` 组码 |
| 线重 | 整数映射为 DXF `370` 组码（-3=ByLayer, -2=ByBlock, -1=Default, 0-211=具体值） |
| 变换 | GSGI `transform` 矩阵应用于坐标后再写入 DXF；DXF 读取时提取坐标，transform 为单位矩阵 |
| 描述 | 统一存入 XDATA `GSGI_DESC`（见 §3） |
| point 解析 | 递归解析 point.ref_pt 链至最终坐标：param_pt → 曲线插值，coord_sys → 局部→WCS 变换 | 实体上的 `represent` 决定代表点位置，`ref_op` 决定偏移解释方式（详见 GSGI 设计文档 §9） |
| 角度单位 | 除 text 外，GSGI 角度字段使用弧度；DXF 使用度 | GSGI→DXF：弧度→度；DXF→GSGI：度→弧度 |

---

## 2. GSGI → DXF

### 2.1 实体映射表

| GSGI 类型 | DXF 实体 | 关键映射 | 备注 |
|-----------|----------|----------|------|
| point | POINT | `10,20`=坐标；`ref_pt` 链存入 XDATA `GSGI_PTREF` | 无 ref_pt 时仅写 POINT |
| param_pt | 不直接输出 | 被 point.ref_pt 引用时，由 point 展开 | 不独立输出 DXF |
| line | LINE | `10,20`=起点；`11,21`=终点 | start_ref/end_ref 解析为坐标 |
| polyline | LWPOLYLINE | `90`=顶点数；`70`=闭合标志；逐顶点 `10,20` | 仅直线段 |
| polyarc | LWPOLYLINE | `90`=顶点数；`42`=逐顶点 bulge | bulge 直接映射 |
| polycurve | LWPOLYLINE/SPLINE | 按 segment 类型拆分：`line`→LINE, `arc`→ARC（从3P计算圆心+半径+角度）, `subsegment_ref`→LWPOLYLINE, `curve_ref`→SPLINE/LINE | 或降级为采样 LWPOLYLINE |
| circle | CIRCLE | `10,20`=圆心；`40`=半径 | center_ref 解析 |
| arc | ARC | `10,20`=圆心（从3P外心公式计算）；`40`=半径；`50`=起角；`51`=终角 | 3P→圆心+半径+角度，通过外心公式计算；角度弧度→度 |
| rectangle | LWPOLYLINE | 4 顶点，`70`=1（闭合） | min_ref/max_ref 展开 |
| text | TEXT / MTEXT | `10,20`=插入点；`1`=内容；`40`=字高；`50`=旋转 | position_ref 解析；含 `\n` → MTEXT，不含 `\n` → TEXT |
| spline_fit | SPLINE | `210`=法向；`70`=flags；`74`=拟合点数；逐点 `11,21` | fit_point_refs 解析 |
| spline_cv | SPLINE | `210`=法向；`70`=flags；`72`=控制点数；`73`=阶数；`40`=节点；`10,20`=控制点 | control_point_refs 解析 |
| block_ref | INSERT | `2`=块名；`10,20`=插入点；`50`=旋转；`41,42`=缩放 | position_ref 解析；旋转弧度→度 |
| xref | XREF | `2`=文件名；`10,20`=插入点；`50`=旋转；`41,42`=缩放；文件路径存 XDATA `GSGI_XREF` | position_ref 解析；旋转弧度→度 |
| table | ACAD_TABLE | 提取 markdown 转为表格结构 | 降级：LINE+TEXT |
| region_anno + `fill` | HATCH | `91`=边界数；`52`=图案角度；边界由 `edges_refs` 引用的 polycurve 展开；`fill.pattern/color/angle/scale` 映射为 DXF 填充属性 | polycurve 须采样为闭合环；图案角度弧度→度；替代原 `hatch` 类型 |
| subsegment | LWPOLYLINE | 沿母曲线采样为多段线 | 采样密度由实现定义 |
| dimension | DIMENSION | `13,23`=定义点；`14,24`=文本点 | p1_ref/p2_ref 解析 |
| region_anno（无 `fill`） | （描述系统，无独立 DXF 实体） | 边界由 edges_refs 引用的 polycurve 映射为闭合 LWPOLYLINE；面积/标签/关联实体存入 XDATA `GSGI_REGION` | region_anno 在 GSGI 中既是实体也属描述系统；无填充时为纯语义区域 |
| region_anno（含 `fill`） | HATCH | 见上方的 hatch 行 | 存在 fill 时 region_anno 驱动 HATCH 实体 |
| position | 无实体 | 全部存入 XDATA `GSGI_POS` | ref_a/ref_b 存为 id |
| coord_sys | UCS + XDATA | 原点 `10,20`=origin_ref 坐标；旋转（弧度）存 XDATA `GSGI_CSYS` | origin_ref 解析 |
| custom_entity | 自定义 | 由 `entity_type` 决定降级策略 | 默认降级为 LWPOLYLINE+TEXT |

### 2.2 映射示例

#### line（point 引用）

```dxf
# GSGI:
# { "id": "L1", "type": "line", "start_ref": "P_A", "end_ref": "P_B" }
# { "id": "P_A", "type": "point", "point": [0, 0] }
# { "id": "P_B", "type": "point", "point": [100, 50] }
#
# DXF output:
  0
LINE
  8
0
 10
0.0
 20
0.0
 11
100.0
 21
50.0
```

#### polyarc（bulge）

```dxf
# GSGI:
# { "id": "PA1", "type": "polyarc",
#   "point_refs": ["P1","P2","P3"],
#   "bulges": [0, 0.414, 0],
#   "closed": true }
#
# DXF output:
  0
LWPOLYLINE
 90
3
 70
1
 10
0.0
 20
0.0
 42
0.0
 10
100.0
 20
0.0
 42
0.414
 10
100.0
 20
50.0
 42
0.0
```

#### region_anno + fill（polycurve 引用）

```dxf
# GSGI:
# { "id": "RA_H1", "type": "region_anno",
#   "edges_refs": ["pc_out", "pc_isl"],
#   "fill": { "pattern": "ANSI31", "color": 3, "angle": 0.0, "scale": 1.0 } }
# { "id": "pc_out", "type": "polycurve", "segments": [...] }
#
# DXF output（简化）:
  0
HATCH
 91
2
 92
7
 72
1
 93
4
 10
0.0
 20
0.0
 10
100.0
 20
0.0
 10
100.0
 20
50.0
 10
0.0
 20
50.0
 92
7
 72
1
 93
4
 10
20.0
 20
20.0
 10
80.0
 20
20.0
 10
80.0
 20
30.0
 10
20.0
 20
30.0
```

---

## 3. DXF → GSGI

### 3.1 实体映射表

| DXF 实体 | GSGI 类型 | 映射逻辑 |
|----------|-----------|----------|
| POINT | point 或 param_pt | 有 XDATA `GSGI_PARAM`（curve_ref/t）→ param_pt；有 XDATA `GSGI_PTREF`（ref_pt）→ point；其余→ point |
| LINE | line | 生成两个 point（终点/起点），line 引用之 |
| LWPOLYLINE（无 bulge） | polyline | 4 顶点闭合且轴对齐 → rectangle |
| LWPOLYLINE（有 bulge） | polyarc | 生成 point 数组 + bulge 数组 |
| POLYLINE | polyline | 同 LWPOLYLINE |
| MLINE | polyline（多条） | 降级为多条 polyline，每个 element 对应一条偏移 polyline |
| CIRCLE | circle | 生成 center point |
| ARC | arc | 从圆心+半径+角度生成3个 point（起点、中点、终点），外心公式见 §5.8 |
| TEXT | text | 生成 position point |
| MTEXT | text | 生成 position point；多行内容保留 `\n` 在 `text` 字段 |
| SPLINE（拟合点） | spline_fit | 生成 fit point 数组 |
| SPLINE（控制点） | spline_cv | 生成 control point 数组 |
| INSERT | block_ref | 生成 position point；块定义存入 blocks |
| XREF | xref | 生成 position point；文件路径从 XDATA `GSGI_XREF` 还原 |
| ACAD_TABLE | table | 生成 position point；提取网格转 markdown |
| HATCH（有填充） | region_anno + `fill` | 边界环生成 polycurve → `edges_refs`；pattern/color/angle/scale → `fill` 对象 |
| HATCH（无填充）+ XDATA `GSGI_REGION` | region_anno | 边界环生成 polycurve → `edges_refs`；面积/标签从 XDATA 还原 |
| DIMENSION | dimension | 生成两个 point（p1_ref/p2_ref） |
| LEADER | polyline + text | 降级为 polyline 和 text |
| UCS + XDATA | coord_sys | 生成 origin point |
| —（XDATA） | position | 从 XDATA `GSGI_POS` 还原 |

### 3.2 映射示例

#### LINE → GSGI

```
# DXF input:
#   0
# LINE
#   8
# 0
#  10
# 0.0
#  20
# 0.0
#  11
# 100.0
#  21
# 50.0
#
# GSGI output:
{
  "entities": [
    { "id": "_p1", "type": "point", "point": [0.0, 0.0] },
    { "id": "_p2", "type": "point", "point": [100.0, 50.0] },
    { "id": "_line1", "type": "line", "start_ref": "_p1", "end_ref": "_p2" }
  ]
}
```

---

## 4. 描述信息的映射

GSGI 的 `descriptions` 不直接对应 DXF 标准字段，统一使用 XDATA 存储。

### 推荐方案：XDATA

| XDATA 应用名 | 存储内容 | 适用实体 |
|-------------|----------|----------|
| `GSGI_ID` | GSGI 实体 id（字符串） | 所有 DXF 实体 |
| `GSGI_DESC` | 属性级描述 JSON `{"target":"","property":"","text":""}` | descriptions 数组 |
| `GSGI_PTREF` | point.ref_pt 引用链 | POINT |
| `GSGI_PARAM` | param_pt 参数 `{"curve_ref":"","t":0}` | POINT |
| `GSGI_POS` | position 实体全量 JSON | 无图形实体 |
| `GSGI_XREF` | xref 文件路径 `{"file_path":"..."}` | XREF |
| `GSGI_REGION` | region_anno 全量 JSON `{"area":0,"label":"","contained_entities":[],"operation":{},"fill":{}}` | 边界由 polycurve 实体映射；有填充时 fill 存在 |
| `GSGI_CSYS` | coord_sys 属性 `{"rotation":0}` | UCS |

### XDATA 格式示例

```dxf
# 实体 ID 标记
 1001
GSGI_ID
 1000
E1

# 属性级描述
 1001
GSGI_DESC
 1000
{"target":"E1","property":"max","text":"车位宽度为5000mm"}

# param_pt 参数
 1001
GSGI_PARAM
 1000
{"curve_ref":"E3","t":0.25}

# region_anno 布尔运算
 1001
GSGI_REGION
 1000
{"area":250.5,"label":"阴影与建筑重叠区","contained_entities":["building_A"],"operation":{"op":"intersect","refs":["RGA_shadow_fan","RGA_building"],"desc":"阴影扇形区与建筑轮廓的交集"}}
```

### 旁路方案

对于不支持 XDATA 的处理环境，可使用同名 `.gsgi.desc.json` 旁路文件存储描述数据。

```json
// 旁路文件：drawing.dxf.gsgi.desc.json
{
  "descriptions": [
    { "target": "E1", "property": "max", "text": "车位宽度为5000mm" }
  ]
}
```

---

## 5. 注意事项

1. **point 引用解析**：GSGI→DXF 方向需递归解析 point.ref_pt 链获取最终 WCS 坐标；DXF→GSGI 方向为每个关键坐标创建 point 实体
2. **polycurve 采样**：HATCH（来自 region_anno + fill）/REGION_ANNO 边界引用 polycurve，DXF 输出时须将 polycurve 采样为闭合 LWPOLYLINE 顶点序列
3. **不可见实体**：position 在 DXF 中无图形对应，全部序列化为 XDATA；DXF 加载后自动重建
4. **块定义**：block 内的 entities 采用与顶层一致的映射规则；嵌套实体（block_ref/xref）禁止出现在块定义内
5. **自定义实体**：custom_entity 默认降级为 LWPOLYLINE + TEXT（properties 转 XDATA）；可通过扩展插件自定义映射

---

## 6. 格式对比

| 特性 | DXF | GSGI |
|------|-----|------|
| 体积 | 极大（含元数据） | 极小（仅几何+描述） |
| 可读性 | 差（组码堆叠） | 好（JSON 结构化） |
| AI 友好度 | 低（需解析噪音） | 高（description 引导+按需搜索） |
| 描述系统 | 无原生支持 | 一等公民 |
| 几何类型 | 30+ 种 | 25 种实体 + 4 种描述手段（按需扩展） |
| 工具链 | 成熟 | 需自行构建转换器 |
