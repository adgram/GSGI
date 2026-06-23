# GSGI-Entities实例

### 1 point

- 坐标为[100, 70]的点P1

```json
{
  "id": "P1",
  "type": "point",
  "point": [100.0, 70.0]
}
```

- 直线L1的四等分点P2（基于ref_pt偏移）

```json
{
  "id": "P2",
  "type": "point",
  "point": [0.0, 25.0],
  "ref_pt": "PP1"
}
```

- 自定义坐标系CS1上坐标为[5, 3]的点P3

```json
{
  "id": "P3",
  "type": "point",
  "point": [5.0, 3.0],
  "ref_pt": "CS1"
}
```

- 直线L1和圆弧A1的交点P4（借助derive_pt，用point封装）

```json
{"id": "DP_INTERSECT_L1_A1", "type": "derive_pt", "op": "intersection", "refs": ["L1", "A1"]}
```
```json
{
  "id": "P4",
  "type": "point",
  "point": [0.0, 0.0],
  "ref_pt": "DP_INTERSECT_L1_A1"
}
```

- 构造点PC1（不可见辅助点）

```json
{
  "id": "PC1",
  "type": "point",
  "point": [50.0, 50.0],
  "point_role": "construction"
}
```

---

### 2 param_pt

- 圆弧A1的四等分点（t=0.25）

```json
{
  "id": "PP1",
  "type": "param_pt",
  "curve_ref": "A1",
  "t": 0.25,
  "label": "A1四分点"
}
```

- 多段线PL1的第二个角点（t=2.0）

```json
{
  "id": "PP2",
  "type": "param_pt",
  "curve_ref": "PL1",
  "t": 2.0,
  "label": "PL1第3顶点"
}
```

- 圆C1的90度位置（t=0.25，圆一周对应t∈[0,1]）

```json
{
  "id": "PP3",
  "type": "param_pt",
  "curve_ref": "C1",
  "t": 0.25,
  "point": [100.0, 150.0],
  "label": "C1顶部"
}
```

---

### 3 line

- 点P1和点P2的连线

```json
{
  "id": "L1",
  "type": "line",
  "start_ref": "P1",
  "end_ref": "P2"
}
```

- 圆弧A1的弦（起点到终点）

```json
{
  "id": "L2",
  "type": "line",
  "description": "圆弧A1的弦",
  "start_ref": "A1_start",
  "end_ref": "A1_end"
}
```

- 两个多段线PL1和PL2中心的连线（假设PL退化规则为取中心）

```json
{
  "id": "L3",
  "type": "line",
  "description": "PL1到PL2中心的连线",
  "start_ref": "PL1",
  "end_ref": "PL2"
}
```

---

### 4 polyline

- 开放多段线PL1

```json
{
  "id": "PL1",
  "type": "polyline",
  "points": [[0.0, 0.0], [50.0, 0.0], [50.0, 50.0], [100.0, 50.0]],
  "closed": false
}
```

- 封闭多段线PL2（三角形）

```json
{
  "id": "PL2",
  "type": "polyline",
  "points": [[0.0, 100.0], [50.0, 150.0], [100.0, 100.0]],
  "closed": true
}
```

---

### 5 polyarc

- 圆角矩形AR1（8个顶点：4条直边+4个90度弧角，bulge=0.414214对应90度弧）

```json
{
  "id": "AR1",
  "type": "polyarc",
  "point_refs": [
    "AR1_P1", "AR1_P2", "AR1_P3", "AR1_P4",
    "AR1_P5", "AR1_P6", "AR1_P7", "AR1_P8"
  ],
  "bulges": [0.0, 0.414214, 0.0, 0.414214, 0.0, 0.414214, 0.0, 0.414214],
  "closed": true
}
```

辅助点定义（width=100, height=60, corner_r=10）：

```json
{"id": "AR1_P1", "type": "point", "point": [10.0, 0.0]}
```
```json
{"id": "AR1_P2", "type": "point", "point": [90.0, 0.0]}
```
```json
{"id": "AR1_P3", "type": "point", "point": [100.0, 10.0]}
```
```json
{"id": "AR1_P4", "type": "point", "point": [100.0, 50.0]}
```
```json
{"id": "AR1_P5", "type": "point", "point": [90.0, 60.0]}
```
```json
{"id": "AR1_P6", "type": "point", "point": [10.0, 60.0]}
```
```json
{"id": "AR1_P7", "type": "point", "point": [0.0, 50.0]}
```
```json
{"id": "AR1_P8", "type": "point", "point": [0.0, 10.0]}
```

- S形曲线AR2（左→右走向，上半弧 bulge 正，下半弧 bulge 负）

```json
{
  "id": "AR2",
  "type": "polyarc",
  "point_refs": ["AR2_P1", "AR2_P2", "AR2_P3"],
  "bulges": [0.5, -0.5],
  "closed": false
}
```

```json
{"id": "AR2_P1", "type": "point", "point": [0.0, 50.0]}
```
```json
{"id": "AR2_P2", "type": "point", "point": [50.0, 50.0]}
```
```json
{"id": "AR2_P3", "type": "point", "point": [100.0, 50.0]}
```

---

### 6 polycurve

- 复合曲线PC1（直线+弧+子段引用）

```json
{
  "id": "PC1",
  "type": "polycurve",
  "segments": [
    { "type": "line", "start_ref": "PC_P1", "end_ref": "PC_P2" },
    { "type": "arc", "start_ref": "PC_P2", "mid_ref": "PC_MID", "end_ref": "PC_P3" },
    { "type": "curve_ref", "ref": "L1" }
  ],
  "closed": false
}
```

```json
{"id": "PC_P1", "type": "point", "point": [0.0, 0.0]}
```
```json
{"id": "PC_P2", "type": "point", "point": [50.0, 0.0]}
```
```json
{"id": "PC_P3", "type": "point", "point": [100.0, 0.0]}
```
```json
{"id": "PC_MID", "type": "point", "point": [80.0, 30.0]}
```

- 复合曲线PC2（全封闭，含subsegment_ref）

```json
{
  "id": "PC2",
  "type": "polycurve",
  "segments": [
    { "type": "line", "start_ref": "PC2_P1", "end_ref": "PC2_P2" },
    { "type": "subsegment_ref", "ref": "SS1" },
    { "type": "line", "start_ref": "PC2_P3", "end_ref": "PC2_P1" }
  ],
  "closed": false
}
```

```json
{"id": "PC2_P1", "type": "point", "point": [0.0, 0.0]}
```
```json
{"id": "PC2_P2", "type": "point", "point": [100.0, 0.0]}
```
```json
{"id": "PC2_P3", "type": "point", "point": [100.0, 50.0]}
```

---

### 7 circle

- 圆形C1（中心参考点+半径）

```json
{
  "id": "C1",
  "type": "circle",
  "center_ref": "C1_CENTER",
  "r": 50.0
}
```

```json
{"id": "C1_CENTER", "type": "point", "point": [100.0, 100.0]}
```

- 带图层和颜色的圆形C2

```json
{
  "id": "C2",
  "type": "circle",
  "center_ref": "C2_CENTER",
  "r": 30.0,
  "layer": "轮廓线",
  "color": 3
}
```

```json
{"id": "C2_CENTER", "type": "point", "point": [200.0, 100.0]}
```

---

### 8 arc

- 圆弧A1（三点定弧，半圆）

```json
{
  "id": "A1",
  "type": "arc",
  "start_ref": "A1_S",
  "mid_ref": "A1_M",
  "end_ref": "A1_E"
}
```

```json
{"id": "A1_S", "type": "point", "point": [0.0, 50.0]}
```
```json
{"id": "A1_M", "type": "point", "point": [50.0, 100.0]}
```
```json
{"id": "A1_E", "type": "point", "point": [100.0, 50.0]}
```

- 圆弧A2（90°到270° 弧段）

```json
{
  "id": "A2",
  "type": "arc",
  "start_ref": "A2_S",
  "mid_ref": "A2_M",
  "end_ref": "A2_E"
}
```

```json
{"id": "A2_S", "type": "point", "point": [0.0, 40.0]}
```
```json
{"id": "A2_M", "type": "point", "point": [-40.0, 0.0]}
```
```json
{"id": "A2_E", "type": "point", "point": [0.0, -40.0]}
```

- 圆弧A3（完整圆，用两个半弧拼合，此段为上半弧）

```json
{
  "id": "A3",
  "type": "arc",
  "start_ref": "A3_S",
  "mid_ref": "A3_M",
  "end_ref": "A3_E"
}
```

```json
{"id": "A3_S", "type": "point", "point": [90.0, 150.0]}
```
```json
{"id": "A3_M", "type": "point", "point": [150.0, 210.0]}
```
```json
{"id": "A3_E", "type": "point", "point": [210.0, 150.0]}
```

---

### 9 rectangle

- 矩形R1（轴对齐，通过min/max对角点）

```json
{
  "id": "R1",
  "type": "rectangle",
  "min_ref": "R1_MIN",
  "max_ref": "R1_MAX"
}
```

```json
{"id": "R1_MIN", "type": "point", "point": [0.0, 0.0]}
```
```json
{"id": "R1_MAX", "type": "point", "point": [200.0, 100.0]}
```

---

### 10 text

- 文字T1（单行文字，默认旋转）

```json
{
  "id": "T1",
  "type": "text",
  "position_ref": "T1_POS",
  "text": "Hello GSGI",
  "height": 10.0,
  "rotation": 0.0
}
```

```json
{"id": "T1_POS", "type": "point", "point": [50.0, 200.0]}
```

- 旋转文字T2（旋转45度，带描述）

```json
{
  "id": "T2",
  "type": "text",
  "position_ref": "T2_POS",
  "text": "旋转标注",
  "height": 8.0,
  "rotation": 45.0
}
```

```json
{"id": "T2_POS", "type": "point", "point": [50.0, 180.0]}
```

---

### 11 text（多行示例）

- 多行文本T1（使用 `\n` 换行）

```json
{
  "id": "T1",
  "type": "text",
  "position_ref": "T1_POS",
  "text": "第一行文字\n第二行文字\n第三行文字",
  "height": 10.0,
  "rotation": 0.0
}
```

```json
{"id": "T1_POS", "type": "point", "point": [300.0, 200.0]}
```

---

### 12 block_ref

- 参照块B1（标准插入，旋转45度）

```json
{
  "id": "BR1",
  "type": "block_ref",
  "block_id": "BLK_DOOR",
  "position_ref": "BR1_POS",
  "rotation": 0.7853981633974483,
  "scale_x": 1.0,
  "scale_y": 1.0,
  "attrs": {
    "door_number": "D001"
  }
}
```

```json
{"id": "BR1_POS", "type": "point", "point": [100.0, 300.0]}
```

- 不等比缩放参照块B2

```json
{
  "id": "BR2",
  "type": "block_ref",
  "block_id": "BLK_WINDOW",
  "position_ref": "BR2_POS",
  "rotation": 0.0,
  "scale_x": 2.0,
  "scale_y": 1.5
}
```

```json
{"id": "BR2_POS", "type": "point", "point": [300.0, 300.0]}
```

---

### 13 spline_fit

- 拟合样条SF1（经过所有拟合点，三次Catmull-Rom）

```json
{
  "id": "SF1",
  "type": "spline_fit",
  "fit_point_refs": ["SF1_P1", "SF1_P2", "SF1_P3", "SF1_P4", "SF1_P5"],
  "degree": 3,
  "closed": false
}
```

```json
{"id": "SF1_P1", "type": "point", "point": [0.0, 600.0]}
```
```json
{"id": "SF1_P2", "type": "point", "point": [50.0, 650.0]}
```
```json
{"id": "SF1_P3", "type": "point", "point": [100.0, 550.0]}
```
```json
{"id": "SF1_P4", "type": "point", "point": [150.0, 700.0]}
```
```json
{"id": "SF1_P5", "type": "point", "point": [200.0, 600.0]}
```

- 封闭拟合样条SF2

```json
{
  "id": "SF2",
  "type": "spline_fit",
  "fit_point_refs": ["SF2_P1", "SF2_P2", "SF2_P3", "SF2_P4"],
  "degree": 3,
  "closed": true
}
```

```json
{"id": "SF2_P1", "type": "point", "point": [300.0, 600.0]}
```
```json
{"id": "SF2_P2", "type": "point", "point": [350.0, 650.0]}
```
```json
{"id": "SF2_P3", "type": "point", "point": [400.0, 600.0]}
```
```json
{"id": "SF2_P4", "type": "point", "point": [350.0, 550.0]}
```

---

### 14 spline_cv

- 控制点样条SC1（4个控制点，三次均匀B样条）

```json
{
  "id": "SC1",
  "type": "spline_cv",
  "control_point_refs": ["SC1_C1", "SC1_C2", "SC1_C3", "SC1_C4"],
  "knots": [0.0, 0.0, 0.0, 1.0, 1.0, 1.0],
  "degree": 2,
  "closed": false
}
```

```json
{"id": "SC1_C1", "type": "point", "point": [0.0, 750.0]}
```
```json
{"id": "SC1_C2", "type": "point", "point": [50.0, 800.0]}
```
```json
{"id": "SC1_C3", "type": "point", "point": [100.0, 700.0]}
```
```json
{"id": "SC1_C4", "type": "point", "point": [150.0, 780.0]}
```

- 带权重的NURBS样条SC2

```json
{
  "id": "SC2",
  "type": "spline_cv",
  "control_point_refs": ["SC2_C1", "SC2_C2", "SC2_C3"],
  "knots": [0.0, 0.0, 0.0, 1.0, 1.0, 1.0],
  "weights": [1.0, 2.0, 1.0],
  "degree": 2,
  "closed": false
}
```

```json
{"id": "SC2_C1", "type": "point", "point": [200.0, 750.0]}
```
```json
{"id": "SC2_C2", "type": "point", "point": [250.0, 800.0]}
```
```json
{"id": "SC2_C3", "type": "point", "point": [300.0, 750.0]}
```

---

### 15 table

- 表格TB1（基于Markdown源数据）

```json
{
  "id": "TB1",
  "type": "table",
  "position_ref": "TB1_POS",
  "width": 300.0,
  "height": 120.0,
  "col_widths": [100.0, 100.0, 100.0],
  "row_heights": [30.0, 25.0, 25.0, 25.0],
  "style": "STANDARD",
  "text_height": 5.0,
  "markdown": "|名称|长度|数量|\n|---|---|---|\n|角钢|6000|10|\n|槽钢|9000|5|\n|钢板|2000|20|"
}
```

```json
{"id": "TB1_POS", "type": "point", "point": [400.0, 600.0]}
```

- 带跨格引用的表格TB2

```json
{
  "id": "TB2",
  "type": "table",
  "position_ref": "TB2_POS",
  "col_widths": [80.0, 120.0],
  "markdown": "|项目|值|\n|---|---|\n|汇总表头|^R0C0|\n|总长|10000|\n|总宽|5000|"
}
```

```json
{"id": "TB2_POS", "type": "point", "point": [400.0, 750.0]}
```

- 表格TB3（含图块引用和外部参照，展示 @/% 前缀标记）

```json
{
  "id": "TB3",
  "type": "table",
  "position_ref": "TB3_POS",
  "col_widths": [80.0, 100.0, 120.0],
  "row_heights": [25.0, 30.0, 30.0],
  "markdown": "| 编号 | 图例 | 备注 |\n|------|------|------|\n| A1   | @BR1 | 标准件 |\n| A2   | %XR1 | 外购件 |"
}
```

```json
{"id": "TB3_POS", "type": "point", "point": [400.0, 800.0]}
```

---

### 16 subsegment

- 子段SS1（曲线L1上t∈[0.2, 0.8]的片段）

```json
{
  "id": "SS1",
  "type": "subsegment",
  "curve_ref": "L1",
  "from_t": 0.2,
  "to_t": 0.8
}
```

- 子段SS2（圆C1上t∈[0.0, 0.25]即90度弧）

```json
{
  "id": "SS2",
  "type": "subsegment",
  "curve_ref": "C1",
  "from_t": 0.0,
  "to_t": 0.25,
  "label": "C1第一象限弧"
}
```

---

### 17 dimension

- 水平尺寸标注DIM1（点P1到P2的水平距离）

```json
{
  "id": "DIM1",
  "type": "dimension",
  "p1_ref": "DIM1_P1",
  "p2_ref": "DIM1_P2",
  "measurement": 100.0,
  "text": "水平间距",
  "dim_line_offset": 15.0,
  "category": "horizontal"
}
```

```json
{"id": "DIM1_P1", "type": "point", "point": [0.0, 0.0]}
```
```json
{"id": "DIM1_P2", "type": "point", "point": [100.0, 0.0]}
```

- 对齐尺寸标注DIM2（两点间的实际距离）

```json
{
  "id": "DIM2",
  "type": "dimension",
  "p1_ref": "DIM2_P1",
  "p2_ref": "DIM2_P2",
  "measurement": 141.421,
  "category": "aligned"
}
```

```json
{"id": "DIM2_P1", "type": "point", "point": [0.0, 50.0]}
```
```json
{"id": "DIM2_P2", "type": "point", "point": [100.0, 150.0]}
```

---

### 18 region_anno

- 区域标注RA1（由多条边围成的区域）

```json
{
  "id": "RA1",
  "type": "region_anno",
  "edges_refs": ["RA1_E1", "RA1_E2", "RA1_E3", "RA1_E4"],
  "area": 5000.0,
  "area_text": "A=5000",
  "contained_entities": ["C1"],
  "label": "办公区域"
}
```

```json
{"id": "RA1_E1", "type": "line", "start_ref": "RA1_P1", "end_ref": "RA1_P2"}
```
```json
{"id": "RA1_E2", "type": "line", "start_ref": "RA1_P2", "end_ref": "RA1_P3"}
```
```json
{"id": "RA1_E3", "type": "line", "start_ref": "RA1_P3", "end_ref": "RA1_P4"}
```
```json
{"id": "RA1_E4", "type": "line", "start_ref": "RA1_P4", "end_ref": "RA1_P1"}
```
```json
{"id": "RA1_P1", "type": "point", "point": [0.0, 0.0]}
```
```json
{"id": "RA1_P2", "type": "point", "point": [100.0, 0.0]}
```
```json
{"id": "RA1_P3", "type": "point", "point": [100.0, 80.0]}
```
```json
{"id": "RA1_P4", "type": "point", "point": [0.0, 80.0]}
```

- 区域标注RA2（带布尔运算派生来源标记）

```json
{
  "id": "RA2",
  "type": "region_anno",
  "edges_refs": ["RA2_E1", "RA2_E2", "RA2_E3", "RA2_E4"],
  "area": 1250.0,
  "label": "阴影与建筑重叠区",
  "contained_entities": ["building_A"],
  "operation": {
    "op": "intersect",
    "refs": ["RGA_shadow_fan", "RGA_building"],
    "desc": "阴影扇形区与建筑轮廓的交集"
  }
}
```

```json
{"id": "RA2_E1", "type": "line", "start_ref": "RA2_P1", "end_ref": "RA2_P2"}
```
```json
{"id": "RA2_E2", "type": "line", "start_ref": "RA2_P2", "end_ref": "RA2_P3"}
```
```json
{"id": "RA2_E3", "type": "line", "start_ref": "RA2_P3", "end_ref": "RA2_P4"}
```
```json
{"id": "RA2_E4", "type": "line", "start_ref": "RA2_P4", "end_ref": "RA2_P1"}
```
```json
{"id": "RA2_P1", "type": "point", "point": [120.0, 0.0]}
```
```json
{"id": "RA2_P2", "type": "point", "point": [180.0, 0.0]}
```
```json
{"id": "RA2_P3", "type": "point", "point": [180.0, 50.0]}
```
```json
{"id": "RA2_P4", "type": "point", "point": [120.0, 50.0]}
```

- 填充区域RA3（带填充图案的 region_anno，替代原 hatch）

```json
{
  "id": "RA3",
  "type": "region_anno",
  "edges_refs": ["RA3_OUTER", "RA3_ISLAND"],
  "area": 6400.0,
  "fill": {
    "pattern": "ANSI31",
    "color": 3,
    "angle": 0.7853981633974483,
    "scale": 1.0
  }
}
```

```json
{"id": "RA3_OUTER", "type": "polyline", "points": [[0.0, 0.0], [100.0, 0.0], [100.0, 80.0], [0.0, 80.0]], "closed": true}
```
```json
{"id": "RA3_ISLAND", "type": "polyline", "points": [[20.0, 20.0], [80.0, 20.0], [80.0, 60.0], [20.0, 60.0]], "closed": true}
```

- 实心填充RA4（SOLID模式）

```json
{
  "id": "RA4",
  "type": "region_anno",
  "edges_refs": ["RA4_OUTER"],
  "fill": {
    "pattern": "SOLID",
    "color": "#FF0000"
  }
}
```

```json
{"id": "RA4_OUTER", "type": "circle", "center_ref": "RA4_CENTER", "r": 40.0}
```
```json
{"id": "RA4_CENTER", "type": "point", "point": [200.0, 100.0]}
```

---

### 19 position

- 约束标注POS1（P1到P2距离必须≥30）

```json
{
  "id": "POS1",
  "type": "position",
  "kind": "constraint",
  "ref_a": "DIM1_P1",
  "ref_b": "DIM1_P2",
  "value": 30.0,
  "operator": ">=",
  "datum": "point"
}
```

- 关系标注POS2（描述两组实体间的位置关系）

```json
{
  "id": "POS2",
  "type": "position",
  "kind": "relation",
  "ref_a": "C1",
  "ref_b": "C2",
  "relationship": "相切外接"
}
```

- 文字描述POS3（实体位置备注）

```json
{
  "id": "POS3",
  "type": "position",
  "kind": "text",
  "ref_a": "R1",
  "params": {
    "text": "矩形R1应位于图纸中心区域"
  }
}
```

---

### 20 xref

- 外部参照XREF1（引用外部DWG文件）

```json
{
  "id": "XREF1",
  "type": "xref",
  "file_path": "基础建筑图.dwg",
  "position_ref": "XREF1_POS",
  "rotation": 0.0,
  "scale_x": 1.0,
  "scale_y": 1.0
}
```

```json
{"id": "XREF1_POS", "type": "point", "point": [0.0, 0.0]}
```

- 带块ID的外部参照XREF2（指定内部块名）

```json
{
  "id": "XREF2",
  "type": "xref",
  "file_path": "标准图框.dwg",
  "block_id": "A3_TITLE_BLOCK",
  "position_ref": "XREF2_POS",
  "scale_x": 1.0,
  "scale_y": 1.0
}
```

```json
{"id": "XREF2_POS", "type": "point", "point": [500.0, 500.0]}
```

---

### 21 coord_sys

- 局部坐标系CS1（用户自定义坐标系）

```json
{
  "id": "CS1",
  "type": "coord_sys",
  "origin_ref": "CS1_ORIGIN",
  "rotation": 0.7853981633974483,
  "visible": true
}
```

```json
{"id": "CS1_ORIGIN", "type": "point", "point": [50.0, 50.0]}
```

- 不可见坐标系CS2（仅用于语义参考）

```json
{
  "id": "CS2",
  "type": "coord_sys",
  "origin_ref": "CS2_ORIGIN",
  "rotation": 0.0,
  "visible": false
}
```

```json
{"id": "CS2_ORIGIN", "type": "point", "point": [200.0, 200.0]}
```

---

### 22 custom_entity

- 自定义实体CE1（扩展类型，任意属性）

```json
{
  "id": "CE1",
  "type": "custom_entity",
  "entity_type": "MyCustomComponent",
  "properties": {
    "param_a": 100.0,
    "param_b": "string_value",
    "nested": {
      "x": 1.0,
      "y": 2.0
    }
  },
  "description": "自定义组件实例，由外部插件解析"
}
```

- 自定义传感器实体CE2

```json
{
  "id": "CE2",
  "type": "custom_entity",
  "entity_type": "SensorNode",
  "properties": {
    "sensor_type": "temperature",
    "range_min": -20.0,
    "range_max": 60.0,
    "unit": "celsius"
  }
}
```

---

### 派生点工具（derive_pt）

`derive_pt` 为工具侧扩展实体类型，不在 GSGI schema 1.0 标准实体中，由实现层（如 gsgi-tool）提供支持。

- 加权平均中心点DP1

```json
{
  "id": "DP_CENTER_PL1",
  "type": "derive_pt",
  "op": "average",
  "refs": ["PL1_P1", "PL1_P2", "PL1_P3", "PL1_P4"],
  "params": { "weights": [1, 2, 2, 1] }
}
```

- 交点DP2

```json
{
  "id": "DP_CENTER_PL2",
  "type": "derive_pt",
  "op": "intersection",
  "refs": ["L1", "L2"]
}
```

- 边界框中心DP3

```json
{
  "id": "DP3",
  "type": "derive_pt",
  "op": "bbox",
  "refs": ["R1"],
  "params": { "which": "center" }
}
```

- 投影点DP4（点P1到直线L1的正交投影）

```json
{
  "id": "DP4",
  "type": "derive_pt",
  "op": "project",
  "refs": ["P1", "L1"]
}
```

- 偏移点DP5（从P1沿P1→P2方向偏移80单位）

```json
{
  "id": "DP5",
  "type": "derive_pt",
  "op": "offset",
  "refs": ["P1", "P2"],
  "params": { "distance": 80.0 }
}
```
