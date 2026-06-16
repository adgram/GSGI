# GSGI — 通用简易几何信息模型设计文档

**GSGI**（General Simple Geometry Information）是一种面向 AI 快速读写、可与 DXF 双向转换的轻量几何数据格式。

---

## 1. 设计理念

| 原则 | 说明 |
|------|------|
| **AI 优先** | 纯文本/JSON 格式、自描述、无需解析二进制或复杂嵌套 |
| **最小化** | 仅包含有效几何信息，无 CAD 系统变量、历史记录等元数据噪音 |
| **可描述** | 每个元素可附带文字说明，支持按属性粒度描述 |

---

## 2. 文件格式

采用 **JSON** 作为序列化格式，文件扩展名 `.gsgi`。

```json
{
  "gsgi": "1.0",
  "summary": "本文件描述停车位布置规范中的车位间距要求",
  "author": "张三",
  "created": "2026-06-04T10:30:00Z",
  "modified": "2026-06-04T15:20:00Z",
  "properties": {
    "unit": "mm",
    "scale": 1.0
  },
  "layers": [ ... ],
  "blocks": [ ... ],
  "entities": [ ... ],
  "groups": [ ... ],
  "ext_derive": { ... },
  "descriptions": [ ... ]
}
```

### 顶层字段

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `gsgi` | string | 是 | 格式版本号，当前 `"1.0"` |
| `summary` | string | 否 | 文档摘要，供 AI 快速理解文件用途 |
| `author` | string | 否 | 文档作者 |
| `created` | string | 否 | 创建时间，ISO 8601 格式（如 `"2026-06-04T10:30:00Z"`） |
| `modified` | string | 否 | 最后修改时间，ISO 8601 格式 |
| `properties` | object | 是 | 文档全局属性 |
| `layers` | array | 否 | 图层定义列表 |
| `text_styles` | array | 否 | 文本样式定义列表 |
| `linetypes` | array | 否 | 线型定义列表 |
| `blocks` | array | 否 | 图块定义列表 |
| `entities` | array | 是 | 顶层实体列表 |
| `groups` | array | 否 | 群组列表 |
| `ext_derive` | object | 否 | 取点与组合运算扩展规则，见 [§9.3](#93-扩展规则ext_derive) |
| `descriptions` | array | 否 | 显式描述列表 |

---

## 3. 文档属性

```json
"properties": {
  "unit": "mm",
  "scale": 1.0,
  "coord_precision": 3,
  "naming": {
    "id_pattern": "^[a-zA-Z_][a-zA-Z0-9_-]{0,127}$",
    "reserved_prefix": "_gsgi_"
  }
}
```

| 字段 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `unit` | string | 否 | `"mm"` | 绘图单位：mm / cm / m / km |
| `scale` | number | 否 | `1.0` | 文档默认比例因子，如 `100` 表示 1:100。作为新建实体时实体级 `scale` 的默认值。渲染时仅以实体自身 `scale` 为准，文档级 `scale` 不参与计算 |
| `coord_precision` | int | 否 | `3` | 坐标保留小数位数（如 `3` 表示 0.001 精度）。建议值：mm 单位时取 `3`（微米级），m 单位时取 `6` |
| `naming` | object | 否 | — | 全局命名规范（见下） |

**命名规范（`naming`）**：

```json
"naming": {
  "id_pattern": "^[a-zA-Z_\\u0080-\\uFFFF][a-zA-Z0-9_\\-\\u0080-\\uFFFF]{0,127}$",
  "reserved_prefix": "_gsgi_"
}
```

| 字段 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `id_pattern` | string | 否 | `^[a-zA-Z_\\u0080-\\uFFFF][a-zA-Z0-9_\\-\\u0080-\\uFFFF]{0,127}$` | ID 的正则表达式约束。`\\u0080-\\uFFFF` 覆盖中文、日文等全 Unicode 区段 |
| `reserved_prefix` | string | 否 | `"_gsgi_"` | 系统保留前缀，用户 ID 不得以此开头 |

全局命名规则是所有 ID（实体 / 图层 / 块 / 群组）的基础约束。各类型可在各自章节引用此规范并补充专属规则。

### 坐标系约定

GSGI 沿用 **AutoCAD / DXF 的世界坐标系（WCS）** 约定：

| 项 | 约定 |
|----|------|
| 坐标系 | 右手笛卡尔系 |
| X 轴 | 水平向右为正 |
| Y 轴 | 垂直向上为正（与屏幕坐标系 Y 轴方向相反） |
| Z 轴 | 垂直纸面向外为正（当前 GSGI 1.0 仅使用 X/Y 二维） |
| 原点 | 世界坐标系原点 `(0, 0)` 由文档定义，通常对应图纸基准点 |
| 角度 | 以弧度（radian）为单位，从 X 轴正方向起，**逆时针**为正（text / mtext 的 rotation 例外，使用度） |
| 长度 | 数值单位由 `properties.unit` 决定 |

**约定**：所有 `[x, y]` 坐标均视为世界坐标系（WCS）下的值。实体变换通过 [`transform`](#8-变换transform) 矩阵实现，不改变坐标本身的坐标系归属。

### 颜色系统

颜色值支持以下两种格式：

| 格式 | 示例 | 说明 |
|------|------|------|
| **ACI 索引** | `7` | AutoCAD 颜色索引（1–255），标准 ACI→RGB 映射表见附录 |
| **Hex RGB** | `"#FF0000"` | 24 位真彩色，`"#"` + 6 位十六进制 RRGGBB，不分区大小写 |

**透明度**：真彩色可附加 8 位透明度，格式 `"#AARRGGBB"`（AA=透明度，00=全透明，FF=不透明）。ACI 色不支持透明度。

**ByLayer / ByBlock 继承**：

| 值 | 说明 |
|----|------|
| `"ByLayer"` | 继承所在图层的颜色 |
| `"ByBlock"` | 继承所在块的颜色 |

**JSON Schema 类型定义**：`color` 字段类型为 `(integer \| string)`，整数视为 ACI 索引，字符串视为 Hex RGB 或继承关键字。

---

## 4. 图层与样式定义

### 图层

**默认图层**：图层 `"0"` 为内置默认图层，始终存在（无需在 `layers` 数组中声明）。未指定 `layer` 字段的实体自动归入该层。图层 `"0"` 不支持改名、删除、被命名。

```json
{
  "id": "0",
  "color": 7,
  "linetype": "CONTINUOUS",
  "visible": true,
  "frozen": false,
  "locked": false,
  "printable": true
}
```

| 字段 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `id` | string | 是 | — | 唯一标识符，即图层名称，被 entities[i].layer 引用。格式遵循 `properties.naming` |
| `color` | int 或 string | 否 | `7` | 颜色，详见 [颜色系统](#颜色系统) |
| `linetype` | string | 否 | `"CONTINUOUS"` | 线型名 |
| `visible` | bool | 否 | `true` | 可见性 |
| `frozen` | bool | 否 | `false` | 冻结状态（冻结层不参与生成/计算） |
| `locked` | bool | 否 | `false` | 锁定状态（锁定层不可编辑） |
| `printable` | bool | 否 | `true` | 是否可打印 |

### 文本样式

文本样式定义文档中可用的文字样式，供 `text`、`mtext`、`table` 等实体引用。

```json
"text_styles": [
  {
    "id": "STANDARD",
    "font": "txt",
    "width_factor": 1.0,
    "oblique_angle": 0,
    "height": 0
  }
]
```

| 字段 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `id` | string | 是 | — | 样式名称，被实体的 `style` 字段引用 |
| `font` | string | 是 | — | 字体文件名（如 `"txt"`、`"Arial"`、`"simsun"`），不含扩展名 |
| `width_factor` | number | 否 | `1.0` | 宽度因子 |
| `oblique_angle` | number | 否 | `0` | 倾斜角度（度） |
| `height` | number | 否 | `0` | 固定字高，`0` 表示可变（由实体指定） |

**默认样式**：样式 `"STANDARD"` 为内置默认样式，始终可用（无需在顶层 `text_styles` 数组中声明）。

### 线型定义

定义文档中可用的线型模式，供图层和实体的 `linetype` 字段引用。

```json
"linetypes": [
  {
    "id": "CONTINUOUS",
    "description": "Continuous",
    "pattern": []
  },
  {
    "id": "DASHED",
    "description": "Dashed",
    "pattern": [10.0, 5.0]
  },
  {
    "id": "CENTER",
    "description": "Center",
    "pattern": [20.0, 5.0, 5.0, 5.0]
  }
]
```

| 字段 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `id` | string | 是 | — | 线型名称，被 `linetype` 字段引用 |
| `description` | string | 否 | — | 线型描述 |
| `pattern` | array | 是 | — | 划线/间隔序列（图纸单位），空数组 `[]` 表示实线。正数=落笔段长度，负数=提笔段长度，0=点 |

**默认线型**：线型 `"CONTINUOUS"` 为内置默认线型，始终可用（无需在顶层 `linetypes` 数组中声明）。

---

## 5. 基本几何实体（entities）

每个实体包含公共基础字段和类型专属字段。

### ID 命名规范

全局命名规则由 [`properties.naming`](#命名规范naming) 定义（字符集、格式、保留前缀）。实体 ID 在此基础上拓展：

| 规则 | 说明 |
|------|------|
| 命名空间 | 实体 ID 在 `entities` 数组内唯一（与图层 / 块 / 群组的 ID 独立） |
| 推荐风格 | 语义化命名，如 `parking_01`、`wall_main`、`标注_A` |

### 公共基础字段

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | 唯一标识符，用于描述引用和群组 |
| `type` | string | 是 | 实体类型（见下） |
| `layer` | string | 否 | 图层 ID，缺省为默认图层 |
| `transform` | object | 否 | 变换（平移/旋转），详见 [8. 变换] |
| `color` | int 或 string | 否 | 颜色，覆盖图层颜色。详见 [颜色系统](#颜色系统) |
| `lineweight` | int | 否 | 线重，单位 1/100 mm。特殊值：`-3`=ByLayer（继承图层）/ `-2`=ByBlock（继承块）/ `-1`=Default（默认）/ `0–211`=具体值（如 `25` 表示 0.25mm） |
| `linetype` | string | 否 | 线型名，覆盖图层线型 |
| `visible` | bool | 否 | 可见性 |
| `space` | string | 否 | 所属空间：`"model"`（模型空间，默认）/ `"paper"`（图纸空间） |
| `scale` | number | 否 | 实体级渲染比例尺，新建时默认继承 `properties.scale` 的值。**渲染尺寸 = 实际尺寸 × 实体 `scale`**（文档 `properties.scale` 不参与渲染计算）。影响范围见下方"比例因子机制"中的受作用属性清单。不影响坐标位置，与 `transform` 矩阵中的缩放语义不同 |
| `description` | string | 否 | 自然语言描述，用于非尺寸/位置的一般语义注释。详见 [10. 描述系统](#10-描述系统descriptions) |
| `represent` | string \| object | 否 | 取点规则。string 为内置 keyword（如 `"center"`）；object 以 `method` 指定并携带参数。详见 [§9](#9-取点与组合运算represent--ref_op) |
| `ref_op` | string \| object | 否 | 组合运算。string 为内置 keyword（如 `"offset"`）；object 以 `method` 指定并携带参数。缺省 `"offset"`。详见 [§9](#9-取点与组合运算represent--ref_op) |

**比例因子机制**：

GSGI 中的两级 `scale` 通过**创建时继承、渲染时独立**的方式协作：

```
实体.scale 初始值 = properties.scale（创建时继承）
渲染尺寸           = 实际尺寸 × 实体.scale（渲染时仅看实体自身）
```

| 层级 | 字段 | 默认值 | 角色 |
|------|------|--------|------|
| **文档级** | `properties.scale` | `1.0` | 实体 `scale` 的默认模板值。如 `100` 表示 1:100 |
| **实体级** | `entities[i].scale` | 继承文档值 | 每个实体独立的渲染比例。修改文档值不影响已创建的实体 |

**渲染规则**：渲染时仅以实体自身 `scale` 为准，文档 `properties.scale` 不参与计算。

**示例**：文档比例 `100`，实体字高 `2.5`：
| 场景 | 实体 `scale` | 渲染字高 |
|------|-------------|----------|
| 新建实体未修改 | `100`（继承文档） | `2.5 × 100 = 250` |
| 文档比例改为 `50` | `100`（已创建，不受影响） | `2.5 × 100 = 250` |
| 新建实体（继承新文档值） | `50` | `2.5 × 50 = 125` |
| 单独修改某实体比例为 `200` | `200` | `2.5 × 200 = 500` |

**受 scale 影响的属性清单**：

| 实体类型 | 受 scale 作用的属性 | 说明 |
|----------|-------------------|------|
| `text` | `height` | 字高 |
| `mtext` | `height`、行距 | 字高及与字高成比例的行距 |
| `linetype`（图层/实体） | `pattern` 虚线间距 | 所有线型虚线段的划线/间隔长度乘以 scale |
| `hatch` | `scale`（图案密度） | 图案缩放密度 |
| `dimension` | `dim_line_offset`、箭头尺寸、文字高度 | 标注几何整体按比例缩放 |
| `table` | 网格行高/列宽、`text_height` | 表格网格间距及内部文字 |

**语义约定**：scale 为**渲染比例**，区别于 `transform` 矩阵（几何坐标变换）。渲染比例影响实体的视觉尺寸属性，而不改变实体的坐标位置——例如 `text` 的 `position_ref` 不受 scale 影响，仅文字大小受 scale 控制。

### 实体类型一览

| 类型 | 说明 |
|------|------|
| `point` | 点 |
| `param_pt` | 曲线参数点（在曲线上通过参数 t 定位的点） |
| `line` | 直线 |
| `polyline` | 多段线（仅直线段，可闭合） |
| `polyarc` | 含圆弧段的多段线（带 bulge 参数） |
| `polycurve` | 复合曲线（由 line/arc/subsegment_ref/curve_ref 拼接） |
| `mline` | 多线（多条平行线） |
| `circle` | 圆 |
| `arc` | 圆弧 |
| `rectangle` | 矩形（轴对齐） |
| `text` | 单行文字 |
| `mtext` | 多行文字 |
| `spline_fit` | 拟合点曲线（通过一系列拟合点） |
| `spline_cv` | 控制点曲线（通过一系列控制点） |
| `block_ref` | 块参照 |
| `xref` | 外部参照（引用外部 .gsgi 文件） |
| `table` | 表格（简化为 markdown 格式） |
| `hatch` | 填充区域 |
| `subsegment` | 子线段（两个参数点之间的路径段，沿原始曲线路径而非直线） |
| `dimension` | 两点标注（定义标注点、长度） |
| `region_anno` | 区域标注（定义区域轮廓、面积，亦属于描述系统，见 §10.5） |
| `position` | 位置关系（不可见，描述/约束几何体间的位置关系） |
| `coord_sys` | 自由坐标系（可见/不可见，由位置点+转角定义局部坐标系） |
| `measure` | 测量值（不可见，计算几何量供参数化引用，见 §5.24） |

---

### 5.1 point

点由坐标组成，包含自身坐标和目标引用两个主要参数。

```json
{
  "id": "P1",
  "type": "point",
  "point": [100.0, 200.0],
  "represent": "self",
  "ref_op": "offset"
}
```

```json
{
  "id": "P2",
  "type": "point",
  "point": [50.0, 0.0],
  "ref_pt": "CS1",
  "represent": "self",
  "ref_op": "offset"
}
```

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `point` | 是 | — | 坐标 `[x, y]` |
| `ref_pt` | 否 | — | 引用另一实体的 id。实际坐标由被引用方的 `ref_op` 决定如何与本 point 组合 |
| `represent` | 固定 | `"self"` | 自身坐标即是代表点，不可修改 |
| `ref_op` | 固定 | `"offset"` | 被引用时偏移叠加，不可修改 |

**坐标解析规则**：

1. 无 `ref_pt`：`point` 直接作为 WCS 绝对坐标
2. 有 `ref_pt`：解析被引用实体 B 的代表点 P_rep（按 B 的 `represent`），再按 B 的 `ref_op` 组合。

> `point` 的 `represent` / `ref_op` 为固定值（见上表），用户不可修改。其他实体类型通过 commonFields 显式声明。

### 5.2 param_pt（曲线参数点）

定义在已有曲线上通过参数 `t` 定位的点，不直接被其他实体引用，通过 `point.ref_pt` 包装后使用。

```json
{
  "id": "PP1",
  "type": "param_pt",
  "curve_ref": "E3",
  "t": 0.25,
  "point": [50, 0],
  "label": "四分点",
  "represent": "self",
  "ref_op": "offset"
}
```

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `curve_ref` | 是 | — | 母曲线实体 id（line / polyline / polyarc / polycurve / arc / spline_* 类型） |
| `t` | 是 | — | 参数值，语义依赖曲线类型：`line/arc/spline_*` = 归一化 `[0, 1]`；`polyline/polyarc` = 段索引+段内归一化（见 5.3 说明） |
| `point` | 否 | — | 缓存坐标 `[x, y]`，用于渲染加速或降级兼容 |
| `label` | 否 | — | 可选标签 |
| `represent` | 否 | `"self"` | 参数点自身坐标 |
| `ref_op` | 否 | `"offset"` | 被引用时偏移叠加 |

**参数 `t` 的语义**：

| 曲线类型 | t 范围 | 语义 |
|----------|--------|------|
| `line` | `[0, 1]` | 线性插值，0=起点，1=终点 |
| `polyline` / `polyarc` | 开放：`[0, N]`，闭合：`[0, N)` | 整数部分为段索引（0-based），小数部分为段内归一化位置 `[0, 1)`。`N` 为段数（开放=顶点数−1，闭合=顶点数，含闭合段）。例：`t=1.5` 表示第 1 段中点 |
| `arc` | `[0, 1]` | `t = 圆心角偏移 / (end_angle − start_angle)`，即圆心角占比；起点 `t=0`，终点 `t=1` |
| `spline_fit` / `spline_cv` | `[0, 1]` | 归一化的曲线参数 u |

### 5.3 line

```json
{ "id": "E2", "type": "line", "start_ref": "P1", "end_ref": "P2", "represent": "center", "ref_op": "offset" }
```

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `start_ref` | 是 | — | 起点引用的 point 实体 id |
| `end_ref` | 是 | — | 终点引用的 point 实体 id |
| `represent` | 否 | `"center"` | 线段中点 |
| `ref_op` | 否 | `"offset"` | 被引用时偏移叠加 |

### 5.4 polyline

```json
{
  "id": "E3",
  "type": "polyline",
  "points": [ [0,0], [100,0], [100,50], [0,50] ],
  "closed": true,
  "represent": "center",
  "ref_op": "offset"
}
```

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `points` | 是 | — | 顶点数组 `[[x,y], ...]` |
| `closed` | 否 | `false` | 是否闭合（连接首尾） |
| `represent` | 否 | `"center"` | 几何中心 |
| `ref_op` | 否 | `"offset"` | 被引用时偏移叠加 |

**设计说明**：`polyline` 使用内联坐标 `points`（而非 point 引用），因为 polyline 仅包含直线段，数据结构简单，直接内联坐标更简洁。对于需要 bulge 参数的 polyarc 及更复杂曲线类型，则采用 point 引用以支持坐标复用。

### 5.5 polyarc（圆弧多段线）

多段线的一种，在直线段基础上通过 bulge 参数支持圆弧段。顶点通过 point 引用。

```json
{
  "id": "E_parc",
  "type": "polyarc",
  "point_refs": ["P1", "P2", "P3", "P4"],
  "bulges": [0, 0.414, 0, -0.414],
  "closed": true,
  "represent": "center",
  "ref_op": "offset"
}
```

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `point_refs` | 是 | — | 顶点引用的 point 实体 id 数组，长度 = N |
| `bulges` | 是 | — | bulge 值数组，长度 = N。`0`=直线段，正=逆时针弧，负=顺时针弧。`bulge = tan(θ/4)`，其中 θ 为圆弧圆心角 |
| `closed` | 否 | `false` | 是否闭合（连接首尾） |
| `represent` | 否 | `"center"` | 几何中心 |
| `ref_op` | 否 | `"offset"` | 被引用时偏移叠加 |

**bulge 转圆弧公式**：
- 圆心角 `θ = 4 · atan(|bulge|)`
- 方向：`bulge > 0` 逆时针，`bulge < 0` 顺时针
- 半径 `r = chord / (2 · sin(θ/2))`

### 5.6 polycurve（复合曲线）

由多种基本线段拼接而成的一段连续路径。每段可为直线、圆弧、子线段引用或曲线引用。

```json
{
  "id": "E_pc1",
  "type": "polycurve",
  "segments": [
    { "type": "line", "start_ref": "P1", "end_ref": "P2" },
    { "type": "arc", "center_ref": "P3", "r": 30, "start_angle": 0, "end_angle": 180 },
    { "type": "subsegment_ref", "ref": "ss1" },
    { "type": "curve_ref", "ref": "E3", "reverse": false }
  ],
  "closed": false,
  "represent": "center",
  "ref_op": "offset"
}
```

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `segments` | 是 | — | 首尾相连的线段序列；各段首尾自动衔接 |
| `segments[].type` | 是 | — | `line` / `arc` / `subsegment_ref` / `curve_ref` |
| `closed` | 否 | `false` | 是否闭合（末段终点→首段起点） |
| `represent` | 否 | `"center"` | 几何中心 |
| `ref_op` | 否 | `"offset"` | 被引用时偏移叠加 |

**段类型详解**：

| type | 必填字段 | 说明 |
|------|----------|------|
| `line` | `start_ref`, `end_ref` | 直线段 |
| `arc` | `center_ref`, `r`, `start_angle`, `end_angle` | 圆弧段 |
| `subsegment_ref` | `ref` | 引用已有 subsegment 实体 id |
| `curve_ref` | `ref`, `reverse`(可选) | 引用已有曲线实体 id（line/polyline/polyarc/spline_*/polycurve），`reverse=true` 反向取路径 |

**语义约定**：各段按顺序首尾相接，后一段的起点自动等于前一段的终点（无需坐标字段显式指定）。

### 5.7 circle

```json
{
  "id": "E4",
  "type": "circle",
  "center_ref": "P3",
  "r": 30,
  "represent": "center",
  "ref_op": "offset"
}
```

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `center_ref` | 是 | — | 圆心引用的 point 实体 id |
| `r` | 是 | — | 半径（>0） |
| `represent` | 否 | `"center"` | 圆心 |
| `ref_op` | 否 | `"offset"` | 被引用时偏移叠加 |

### 5.8 arc

```json
{
  "id": "E5",
  "type": "arc",
  "center_ref": "P3", "r": 30,
  "start_angle": 0,
  "end_angle": 6.2832,
  "represent": "center",
  "ref_op": "offset"
}
```

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `center_ref` | 是 | — | 圆心引用的 point 实体 id |
| `r` | 是 | — | 半径 |
| `start_angle` | 否 | `0` | 起始角度（弧度） |
| `end_angle` | 否 | `6.2832` | 终止角度（弧度） |
| `represent` | 否 | `"center"` | 弧曲线参数 `t=0.5` 处的点 |


### 5.9 rectangle

```json
{
  "id": "E6",
  "type": "rectangle",
  "min_ref": "P4",
  "max_ref": "P5",
  "represent": "center",
  "ref_op": "offset"
}
```

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `min_ref` | 是 | — | 左下角引用的 point 实体 id |
| `max_ref` | 是 | — | 右上角引用的 point 实体 id |
| `represent` | 否 | `"center"` | 矩形几何中心 |
| `ref_op` | 否 | `"offset"` | 被引用时偏移叠加 |

**语义约定**：按 `(min[0], min[1]) → (max[0], min[1]) → (max[0], max[1]) → (min[0], max[1])` 顺序构成闭合矩形。

### 5.10 text

```json
{
  "id": "E7",
  "type": "text",
  "position_ref": "P6",
  "text": "车位A",
  "height": 2.5,
  "rotation": 0,
  "represent": "base",
  "ref_op": "offset"
}
```

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `position_ref` | 是 | — | 插入点引用的 point 实体 id |
| `text` | 是 | — | 文字内容 |
| `height` | 否 | `2.5` | 文字高度 |
| `rotation` | 否 | `0` | 旋转角度（度） |
| `represent` | 否 | `"base"` | 基点位（插入点） |
| `ref_op` | 否 | `"offset"` | 被引用时偏移叠加 |

### 5.11 block_ref（块参照）

```json
{
  "id": "E8",
  "type": "block_ref",
  "block_id": "停车位",
  "position_ref": "P7",
  "rotation": 0,
  "scale_x": 1,
  "scale_y": 1,
  "represent": "base",
  "ref_op": "offset"
}
```

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `block_id` | 是 | — | 引用的块定义的 id |
| `position_ref` | 是 | — | 插入点引用的 point 实体 id |
| `rotation` | 否 | `0` | 旋转角度（弧度） |
| `scale_x` | 否 | `1` | X 缩放比例 |
| `scale_y` | 否 | `1` | Y 缩放比例 |
| `represent` | 否 | `"base"` | 基点位（插入点） |
| `ref_op` | 否 | `"offset"` | 被引用时偏移叠加 |

### 5.12 mline（多线）

表示两条或多条平行线段，每段可独立控制颜色和线型。

```json
{
  "id": "E9",
  "type": "mline",
  "point_refs": ["P8", "P9", "P10"],
  "elements": [
    { "offset": -0.5, "color": 1, "linetype": "CONTINUOUS" },
    { "offset": 0.5, "color": 2, "linetype": "CONTINUOUS" }
  ],
  "closed": false,
  "scale": 1.0,
  "represent": "center",
  "ref_op": "offset"
}
```

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `point_refs` | 是 | — | 基线顶点引用的 point 实体 id 数组 |
| `elements` | 是 | — | 平行元素列表 |
| `elements[].offset` | 是 | — | 距基线的偏移量，单位为图纸单位（同 `properties.unit`）；正值=沿基线方向的右侧，负值=左侧。实际偏移 = `offset × scale` |
| `elements[].color` | 否 | 继承 | 该元素颜色（int 或 string），继承优先级：元素 color → mline 实体 color → 图层 color，详见 [颜色系统](#颜色系统) |
| `elements[].linetype` | 否 | 继承 | 该元素线型，继承优先级同 color |
| `closed` | 否 | `false` | 是否闭合 |
| `scale` | 否 | `1.0` | 整体缩放（所有 offset 乘以该值） |
| `represent` | 否 | `"center"` | 基线中点 |
| `ref_op` | 否 | `"offset"` | 被引用时偏移叠加 |

### 5.13 mtext（多行文字）

支持多行和段落级属性。

```json
{
  "id": "E10",
  "type": "mtext",
  "position_ref": "P11",
  "width": 100,
  "text": "车位说明：\n标准车位\n宽度2500mm",
  "height": 2.5,
  "style": "STANDARD",
  "rotation": 0,
  "attachment": "top_left",
  "represent": "base",
  "ref_op": "offset"
}
```

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `position_ref` | 是 | — | 插入点引用的 point 实体 id |
| `width` | 是 | — | 段落宽度（文字换行边界） |
| `text` | 是 | — | 文字内容，`\n` 换行；支持内联格式化代码（见下） |
| `height` | 否 | `2.5` | 文字高度 |
| `style` | 否 | `"STANDARD"` | 文字样式名 |
| `rotation` | 否 | `0` | 旋转角度（度） |
| `attachment` | 否 | `"top_left"` | 附着点：`top_left` / `top_center` / `top_right` / `middle_left` / `middle_center` / `middle_right` / `bottom_left` / `bottom_center` / `bottom_right` |
| `represent` | 否 | `"base"` | 基点位（插入点） |
| `ref_op` | 否 | `"offset"` | 被引用时偏移叠加 |

**内联格式化代码**（与 DXF MTEXT 兼容，可选支持）：

| 代码 | 说明 | 示例 |
|------|------|------|
| `\P` | 段落换行（与 `\n` 等价） | `第一行\P第二行` |
| `\C<n>` | 颜色索引（1–255） | `\C1红字` |
| `\H<v>x` | 字高倍数 | `\H1.5x放大` |
| `\H<v>` | 绝对字高 | `\H5;固定` |
| `\W<v>` | 宽度因子 | `\W0.8紧缩` |
| `\f<font>;` | 字体名 | `\fArial;text` |
| `\L ... \l` | 下划线开/关 | `\L下划线\l` |
| `\O ... \o` | 上划线开/关 | `\O上划线\o` |
| `\~` | 不间断空格 | `A\~B` |
| `\\` | 字面反斜杠 | `路径\\` |
| `{...}` | 作用域局部格式（出 `{}` 后恢复） | `普通{\C1红}普通` |

**约定**：未启用格式化代码时，文本内容按字面渲染；AI 写入纯文本时应转义 `\` 为 `\\`。

### 5.14 spline_fit（拟合点曲线）

通过一系列拟合点定义的平滑曲线（通过所有点）。

```json
{
  "id": "E11",
  "type": "spline_fit",
  "fit_point_refs": ["P12", "P13", "P14", "P15"],
  "degree": 3,
  "closed": false,
  "tolerance": 0.0,
  "represent": "center",
  "ref_op": "offset"
}
```

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `fit_point_refs` | 是 | — | 拟合点引用的 point 实体 id 数组 |
| `degree` | 否 | `3` | 曲线阶数（2 或 3） |
| `closed` | 否 | `false` | 是否闭合 |
| `tolerance` | 否 | `0.0` | 拟合公差 |
| `represent` | 否 | `"center"` | 曲线中点 |
| `ref_op` | 否 | `"offset"` | 被引用时偏移叠加 |

### 5.15 spline_cv（控制点曲线）

通过控制点定义的贝塞尔/B样条曲线（不通过控制点，曲线被"拉"向控制点）。

```json
{
  "id": "E12",
  "type": "spline_cv",
  "control_point_refs": ["P16", "P17", "P18", "P19"],
  "knots": [0, 0, 0, 1, 2, 3, 3, 3],
  "weights": [1, 1, 1, 1],
  "degree": 3,
  "closed": false,
  "represent": "center",
  "ref_op": "offset"
}
```

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `control_point_refs` | 是 | — | 控制点引用的 point 实体 id 数组 |
| `knots` | 否 | 自动生成 | 节点向量（均匀）。长度必须满足 `len(knots) = len(control_point_refs) + degree + 1` |
| `weights` | 否 | 全 `1` | 各控制点权重（NURBS），长度同 `control_point_refs` |
| `degree` | 否 | `3` | 曲线阶数 |
| `closed` | 否 | `false` | 是否闭合 |
| `represent` | 否 | `"center"` | 曲线中点 |
| `ref_op` | 否 | `"offset"` | 被引用时偏移叠加 |

### 5.16 table（表格）

用 markdown 表格语法简化表示。每个单元格为文本。

```json
{
  "id": "E13",
  "type": "table",
  "position_ref": "P20",
  "width": 200,
  "height": 100,
  "col_widths": [60, 70, 70],
  "row_heights": [25, 25, 25, 25],
  "style": "STANDARD",
  "text_height": 2.5,
  "markdown": "| 编号 | 宽度(mm) | 长度(mm) |\n|------|----------|----------|\n| A1   | 2500     | 5000     |\n| A2   | 2500     | 5000     |\n| B1   | 3000     | 6000     |",
  "merges": [
    { "row": 0, "col": 0, "rowspan": 1, "colspan": 3, "text": "停车位尺寸表" }
  ],
  "represent": "base",
  "ref_op": "offset"
}
```

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `position_ref` | 是 | — | 表格左上角插入点引用的 point 实体 id |
| `width` | 否 | 自动 | 表格总宽 |
| `height` | 否 | 自动 | 表格总高 |
| `col_widths` | 否 | 等分 | 各列宽度数组 |
| `row_heights` | 否 | 等分 | 各行高度数组 |
| `style` | 否 | `"STANDARD"` | 文字样式 |
| `text_height` | 否 | `2.5` | 文字高度 |
| `markdown` | 是 | — | markdown 表格语法，首行为表头，第二行为分隔线 |
| `merges` | 否 | — | 合并单元格规则 |
| `represent` | 否 | `"base"` | 基点位（表格左上角） |
| `ref_op` | 否 | `"offset"` | 被引用时偏移叠加 |

**合并单元格（merges）**：

```json
"merges": [
  { "row": 0, "col": 0, "rowspan": 2, "colspan": 1 },
  { "row": 0, "col": 1, "rowspan": 1, "colspan": 2, "text": "尺寸" }
]
```

| 字段 | 必填 | 说明 |
|------|------|------|
| `row` | 是 | 起始行索引（0-based） |
| `col` | 是 | 起始列索引（0-based） |
| `rowspan` | 否 | `1` | 合并的行数 |
| `colspan` | 否 | `1` | 合并的列数 |
| `text` | 否 | — | 合并后单元格文字（覆盖 markdown 中原位置内容） |

**语义约定**：
- 第一行作为表头，第二行为分隔线，第三行起为数据行
- 列数与 `col_widths` 一致
- 合并区域在 `markdown` 中的被覆盖单元格应留空
- 渲染时先按 markdown 填充，再按 merges 覆盖合并区域

### 5.17 hatch（填充区域）

```json
{
  "id": "E14",
  "type": "hatch",
  "pattern": "SOLID",
  "color": 3,
  "angle": 0,
  "scale": 1.0,
  "boundaries": [
    {
      "outer_ref": "pc_outer_1",
      "island_refs": ["pc_island_1"]
    }
  ],
  "associative": true,
  "represent": "center",
  "ref_op": "offset"
}
```

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `pattern` | 是 | — | 填充图案名；`"SOLID"` 为实心填充，其他如 `"ANSI31"`、`"AR-CONC"` |
| `color` | 否 | 继承 | 填充颜色（int 或 string），详见 [颜色系统](#颜色系统) |
| `angle` | 否 | `0` | 图案旋转角度（弧度） |
| `scale` | 否 | `1.0` | 图案缩放 |
| `boundaries` | 是 | — | 环组列表；每个环组通过 polycurve 引用定义一片填充区域 |
| `boundaries[].outer_ref` | 是 | — | 外边界引用的 polycurve 实体 id |
| `boundaries[].island_refs` | 否 | `[]` | 岛（内部挖空）引用的 polycurve 实体 id 数组 |
| `associative` | 否 | `true` | 是否关联边界 |
| `represent` | 否 | `"center"` | 填充区域几何中心 |
| `ref_op` | 否 | `"offset"` | 被引用时偏移叠加 |

**语义约定**：
- 一个 `boundaries` 数组元素 = 一个连通区域 = 一个外环 + 零或多个内岛
- 多个非连通区域用多个环组表达
- `outer_ref` 引用的 polycurve 必须构成闭合路径；`island_refs` 各项独立闭合且应位于外边界内
- 圆形/圆弧边界用 `type: "arc"` segment 的 polycurve 表达（`start_angle=0, end_angle=360` 为圆）

### 5.18 subsegment（子线段）

两个参数点之间的路径段，保留沿原始曲线的路径而非直线连接。

```json
{
  "id": "E16",
  "type": "subsegment",
  "curve_ref": "E3",
  "from_t": 0.25,
  "to_t": 0.75,
  "from_point": [50, 0],
  "to_point": [100, 25],
  "label": "中间段",
  "represent": "center",
  "ref_op": "offset"
}
```

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `curve_ref` | 是 | — | 引用的母曲线实体 id（line / polyline / polyarc / polycurve / arc / spline_* 类型） |
| `from_t` | 是 | — | 起始参数，语义同 [5.2 param_pt t](#52-param_pt曲线参数点) |
| `to_t` | 是 | — | 终止参数（必须 ≥ `from_t`） |
| `from_point` | 否 | — | 起点缓存坐标 `[x, y]` |
| `to_point` | 否 | — | 终点缓存坐标 `[x, y]` |
| `label` | 否 | — | 可选标签 |
| `represent` | 否 | `"center"` | 子线段曲线参数 `t=(from_t+to_t)/2` 处的点 |
| `ref_op` | 否 | `"offset"` | 被引用时偏移叠加 |

### 5.19 dimension（两点标注）

标注两点间的距离。标注点通过引用 `point` 实体来定位。

```json
{
  "id": "D1",
  "type": "dimension",
  "p1_ref": "P1",
  "p2_ref": "P2",
  "measurement": 50.0,
  "text": "50",
  "dim_line_offset": 10.0,
  "category": "horizontal",
  "represent": "center",
  "ref_op": "offset"
}
```

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `p1_ref` | 是 | — | 标注起点引用的 point 实体 id |
| `p2_ref` | 是 | — | 标注终点引用的 point 实体 id |
| `measurement` | 否 | 自动计算 | 标注长度（单位内的数值） |
| `text` | 否 | `measurement` 值 | 标注文字 |
| `dim_line_offset` | 否 | `10` | 尺寸线偏移距离 |
| `category` | 否 | `"aligned"` | 标注类型：`horizontal` / `vertical` / `aligned` |
| `represent` | 否 | `"center"` | 两标注点的中点 |
| `ref_op` | 否 | `"offset"` | 被引用时偏移叠加 |

**语义约定**：
- 标注点在几何体上时，先定义 `param_pt`，再定义 `point` 通过 `ref_pt` 引用该 `param_pt`，dimension 引用该 `point`

### 5.20 region_anno（区域标注）

区域标注定义一块区域的轮廓和面积，并可关联区域内包含的实体。它既是实体类型，也属于描述系统（见 [§10.5](#105-region_anno区域标注)），与 `dimension`、`position`、`description` 共同构成四种描述手段。

```json
{
  "id": "RGA1",
  "type": "region_anno",
  "edges_refs": ["pc_region_outer", "pc_region_island_1"],
  "area": 25000000,
  "area_text": "25 m²",
  "contained_entities": ["E1", "E2", "E3"],
  "label": "停车区A",
  "represent": "center",
  "ref_op": "offset"
}
```

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `edges_refs` | 是 | — | 闭合边界环引用的 polycurve 实体 id 数组；第一个为外边界，后续为岛 |
| `area` | 否 | 自动计算 | 面积值（平方单位） |
| `area_text` | 否 | 自动格式化 | 面积文本描述 |
| `contained_entities` | 否 | — | 区域内包含的实体 id 列表 |
| `label` | 否 | — | 区域标签 |
| `represent` | 否 | `"center"` | 区域几何中心 |
| `ref_op` | 否 | `"offset"` | 被引用时偏移叠加 |
| `operation` | 否 | — | 标记该区域是由布尔运算派生得来（见下文） |

**语义约定**：
- `edges_refs` 数组第一项为外边界，后续项为岛（内部挖空区域）
- 引用的 polycurve 必须构成闭合路径
- 各环应闭合且互不相交
- 岛必须位于外边界内
- 当区域轮廓完全由现有几何边构成时，区域标注本身不产生新线条，仅声明区域概念

#### operation 对象结构

| 子字段 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| `op` | string | 是 | 运算类型：`"union"`（并集）、`"intersect"`（交集）、`"difference"`（差集）、`"xor"`（异或） |
| `refs` | string[] | 是 | 运算操作数，引用其他 `region_anno` 实体 id，需 ≥2 个 |
| `desc` | string | 否 | 自然语言说明（如"阴影扇形区与建筑轮廓的交集"） |

**语义约定**：
- `operation` 仅声明区域的派生来源，不驱动计算。渲染/显示时以 `edges_refs` 为准
- `refs` 中引用的实体必须为 `region_anno` 类型
- 允许链式引用（A 作为 B 的操作数，B 作为 C 的操作数）
- 不允许循环引用（实现应检测并报错）
- 当 `operation` 与 `edges_refs` 内容矛盾时，以 `edges_refs` 为实际几何，`operation` 仅为历史语义

### 5.21 position（位置关系）

**不可见实体**，无图形表示。用于描述或约束两个几何参照之间的位置关系，或者描述抽象的邻接语义。**距离值均指两实体最近点之间的距离**，而非某两个特定点。

支持三种模式：

| 模式 | kind 值 | 说明 | 示例 |
|------|---------|------|------|
| 约束 | `"constraint"` | 用数值（含等式/不等式）约束实体间距 | E1 到 E2 最近点距离 ≥ 30mm |
| 文本 | `"text"` | 纯文本抽象描述 | "点的50mm范围内无其他几何体" |
| 关系 | `"relation"` | 有实体参照的精确结构化描述，基于基准类型 | "E1 在 P1 右侧 50mm" |

#### constraint 示例

```json
{
  "id": "R1",
  "type": "position",
  "kind": "constraint",
  "ref_a": "E1",
  "ref_b": "E2",
  "value": 5000,
  "description": "车位1到车位2的最近点距离为5000mm",
  "represent": "origin",
  "ref_op": "offset"
}
```

> `position` 的 `represent`/`ref_op` 默认值为 `"origin"`/`"offset"`（所有示例中略写，效果同上）。

```json
{
  "id": "R2",
  "type": "position",
  "kind": "constraint",
  "ref_a": "E1",
  "ref_b": "E2",
  "value": 30,
  "operator": ">=",
  "description": "车位1长边到车位2长边最近点距离≥30mm"
}
```

#### text 示例（纯文本）

```json
{
  "id": "R3",
  "type": "position",
  "kind": "text",
  "description": "点的50mm范围内无其他几何体"
}
```

```json
{
  "id": "R4",
  "type": "position",
  "kind": "text",
  "ref_a": "P1",
  "description": "P1点的50mm范围内无其他几何体"
}
```

`text` 模式仅用于无须结构化的抽象描述。需要精确表达时使用 `relation` 模式：

#### relation 示例（增强结构化）

```json
{
  "id": "R5",
  "type": "position",
  "kind": "relation",
  "ref_a": "P1",
  "ref_b": "E1",
  "datum": "point",
  "relationship": "distance",
  "params": { "value": 5000 },
  "description": "E1 距 P1 5000mm"
}
```

```json
{
  "id": "R6",
  "type": "position",
  "kind": "relation",
  "ref_a": "CS1",
  "ref_b": "E1",
  "datum": "csys",
  "relationship": "offset",
  "params": {
    "direction_ref": "L1",
    "distance": [3000, 5000]
  },
  "description": "E1 在 CS1 下沿 L1 方向 3000~5000mm 处"
}
```

`direction_ref` 引用直线实体 id，由直线定义方向。`distance` 为单值时表示精确偏移距离，为 `[min, max]` 数组时表示闭区间。客体类型与 `datum` 不匹配时自动取外接矩形/包围盒。

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `kind` | 是 | — | `"constraint"` / `"text"` / `"relation"` |
| `ref_a` | 否 | — | **主体**几何参照（实体 id）；constraint 和 relation 必填，text 可选 |
| `ref_b` | 否 | — | **客体**几何参照（实体 id）；constraint 和 relation 必填，text 可选 |
| `value` | 否 | — | 距离值（仅 constraint 模式）；两实体最近点间距离 |
| `operator` | 否 | `"=="` | 约束运算符：`>=` / `<=` / `>` / `<` / `==`（仅 constraint 模式） |
| `datum` | 否 | — | 基准类型（仅 relation 模式），决定可用的 `relationship`：`"point"` / `"csys"` / `"line"` / `"region"` / `"rect"` |
| `relationship` | 否 | — | 由 `datum` 决定可选值，见下表（仅 relation 模式） |
| `params` | 否 | — | 关系参数（仅 relation 模式） |
| `description` | 否 | — | 自然语言描述；relation 模式建议以主体→客体的方向书写 |
| `represent` | 否 | `"origin"` | 无几何形态，回退到原点 |
| `ref_op` | 否 | `"offset"` | 被引用时偏移叠加 |

**datum ↔ relationship 对照表**：

| datum | relationship | params | 说明 |
|-------|-------------|--------|------|
| `"point"` | `"distance"` | `value`: number | 客体到点的距离 |
| | `"align_x"` | — | 客体的 x 坐标与点相同 |
| | `"align_y"` | — | 客体的 y 坐标与点相同 |
| `"csys"` | `"offset"` | `direction_ref`: string（直线 id），`distance`: number 或 [number, number]（单值或闭区间） | 客体在坐标系下沿指定方向偏移 |
| `"line"` | `"parallel"` | — | 客体与直线平行 |
| | `"perpendicular"` | — | 客体与直线垂直 |
| | `"collinear"` | — | 客体与直线共线 |
| | `"at_angle"` | `angle`: number | 客体与直线成指定角度 |
| `"region"` | `"inside"` | — | 客体在封闭区域内 |
| | `"outside"` | — | 客体在封闭区域外 |
| | `"on_boundary"` | — | 客体在区域边界上 |
| | `"overlaps"` | — | 客体与区域重叠 |
| `"rect"` | `"top_aligned"` | `gap`: number（可选） | 顶部对齐 |
| | `"bottom_aligned"` | `gap`: number（可选） | 底部对齐 |
| | `"left_aligned"` | `gap`: number（可选） | 左对齐 |
| | `"right_aligned"` | `gap`: number（可选） | 右对齐 |
| | `"h_centered"` | `gap`: number（可选） | 水平居中 |
| | `"v_centered"` | `gap`: number（可选） | 垂直居中 |
| | `"centered"` | `gap`: number（可选） | 完全居中 |

**语义约定**：
- `position` 实体不产生任何图形输出，仅作为语义层存在
- `constraint` 模式：`ref_a` 到 `ref_b` 的最近点距离需满足 `operator value`。`operator` 缺省为 `"=="`（精确距离）
- `text` 模式：纯文本抽象描述，可有 `ref_a` 作为上下文辅助，不具备结构化解析能力
- `relation` 模式：基于 `datum` 定义主体基准类型，按 `relationship` + `params` 精确描述客体相对主体的关系
- 客体类型与 `datum` 不匹配时，自动取客体的外接矩形/包围盒等近似形状来匹配基准类型
- AI 读取时只需扫描 `kind` + `description` 即可理解位置关系，无需解析全部几何

---

### 5.22 xref（外部参照）

引用外部 `.gsgi` 文件中的几何内容，类似 `block_ref` 但从外部文件加载。

```json
{
  "id": "E20",
  "type": "xref",
  "file_path": "其他楼层.gsgi",
  "block_id": "parking",
  "position_ref": "P21",
  "rotation": 0,
  "scale_x": 1,
  "scale_y": 1,
  "represent": "base",
  "ref_op": "offset"
}
```

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `file_path` | 是 | — | 外部 `.gsgi` 文件路径（相对或绝对路径） |
| `block_id` | 否 | — | 外部文件中要引用的块定义 id；缺省则加载全部顶层实体 |
| `position_ref` | 是 | — | 插入点引用的 point 实体 id |
| `rotation` | 否 | `0` | 旋转角度（弧度） |
| `scale_x` | 否 | `1` | X 缩放比例 |
| `scale_y` | 否 | `1` | Y 缩放比例 |
| `represent` | 否 | `"base"` | 基点位（插入点） |
| `ref_op` | 否 | `"offset"` | 被引用时偏移叠加 |

**语义约定**：
- `file_path` 解析失败时，实现应报错而非静默忽略
- 不支持循环外部引用（A 引用 B，B 又引用 A）；实现应检测并报错
- DXF 转换时，xref 映射为 `XREF` / `INSERT` + XDATA 记录外部路径

### 5.23 coord_sys（自由坐标系）

定义局部坐标系，由位置点和转角组成。其他实体（如 point）可通过 `ref_pt` 引用此坐标系，将其坐标解释为局部坐标。

```json
{
  "id": "CS1",
  "type": "coord_sys",
  "origin_ref": "P22",
  "rotation": 0.7854,
  "visible": false,
  "represent": "center",
  "ref_op": "local"
}
```

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `origin_ref` | 是 | — | 原点引用的 point 实体 id |
| `rotation` | 否 | `0` | 局部坐标系 X 轴与世界 X 轴的夹角（弧度），逆时针为正 |
| `visible` | 否 | `false` | 是否在图形中显示坐标系图标 |
| `represent` | 否 | `"center"` | 取 `origin_ref` 引用的 point |
| `ref_op` | 否 | `"local"` | 被引用时按局部坐标变换 |

**坐标系变换**：point 引用此坐标系时，point 坐标转换为 WCS：`P_wcs = P_origin + R · P_local`，其中 `R = [[cosθ, -sinθ], [sinθ, cosθ]]`。以上例 θ=0.7854（45°）。

---

### 5.24 measure（测量值）

**不可见实体**，无图形表示。用于计算几何量（距离、长度、面积、角度），产出数值可被其他实体的参数引用。

```json
{ "id": "M1", "type": "measure", "kind": "distance", "ref_a": "E1", "ref_b": "E2", "represent": "origin", "ref_op": "offset" }
```

| kind | ref_a | ref_b | 输出说明 |
|------|-------|-------|---------|
| `distance` | 实体 id | 实体 id | 两实体最近点距离 |
| `length` | 曲线实体 id | — | 曲线长度 |
| `area` | 闭合区域实体 id | — | 区域面积 |
| `angle` | 直线实体 id | 直线实体 id | 两直线夹角（度） |

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `type` | string | 是 | `"measure"` |
| `kind` | string | 是 | 测量类型 |
| `ref_a` | string | 是 | 主参照实体 id |
| `ref_b` | string | 否 | 次参照实体 id |
| `represent` | string | 否 | `"origin"` | 无几何形态，回退到原点 |
| `ref_op` | string | 否 | `"offset"` | 被引用时偏移叠加 |

**语义约定**：
- 不产生任何图形输出，仅作为参数化数值层存在
- 任意 ref 无法解析时返回 null
- AI 解析时可通过 `type="measure"` 识别参数化依赖关系

---

## 6. 图块定义（blocks）

```json
{
  "id": "停车位",
  "name": "停车位",
  "base_point": [0, 0],
  "entities": [
    {
      "id": "停车位_rect",
      "type": "rectangle",
      "min": [-1250, -2500],
      "max": [1250, 2500]
    },
    {
      "id": "停车位_text",
      "type": "text",
      "position": [0, 0],
      "text": "P",
      "height": 500
    }
  ]
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | 唯一标识符，被 block_ref 引用。格式遵循 `properties.naming` |
| `name` | string | 否 | 块名称（缺省同 id） |
| `base_point` | [number, number] | 否 | 基点坐标，默认 `[0, 0]` |
| `entities` | array | 是 | 块内实体列表（无 id 则自动生成） |

块内的 entities 结构与顶层 entities 共享相同的实体类型定义。

**禁止嵌套**：块定义内不得包含 `block_ref` 或 `xref` 类型的实体。所有块引用必须扁平化存在于顶层 entities 中，不可递归嵌套。

**块内坐标简化**：块定义中的实体支持使用内联坐标（如 `rectangle` 的 `min`/`max`、`text` 的 `position`），而非强制通过 point 引用。这是为了块定义的自包含性——块通常被实例化为 block_ref，其实体坐标应在块局部坐标系下直接表达，无需额外定义 point 实体。

---

## 7. 群组（groups）

```json
{
  "id": "G1",
  "name": "车位组",
  "members": ["E1", "E2", "E3"]
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | 唯一标识符，格式遵循 `properties.naming` |
| `name` | string | 否 | 群组名称 |
| `members` | string[] | 是 | 实体 ID / 块 ID 列表 |

群组支持嵌套：members 可引用其他群组的 id。

**循环引用禁止**：群组嵌套不得形成循环引用（如 G1→G2→G1）。实现必须检测并拒绝任何导致循环的 membership 添加。

---

## 8. 变换（transform）

每个实体可附带 transform，对实体坐标施加仿射变换而非修改原始坐标。

### 8.1 定义

```json
"transform": [[1, 0, 100], [0, 1, 50]]
```

| 字段 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `matrix` | `[[number, number, number], [number, number, number]]` | 否 | `[[1,0,0],[0,1,0]]` | 2×3 仿射变换矩阵（见下）；缺省为单位矩阵 |

**解析规则**：
1. 当 `transform` 为数组 `[[a,b,tx],[c,d,ty]]` 时，等价于 `{ matrix: [[a,b,tx],[c,d,ty]] }`（简写形式）
2. 当 `transform` 缺省时，为单位矩阵（不产生变换）

### 8.2 矩阵定义

使用 **2×3 仿射变换矩阵** `[[a, b, tx], [c, d, ty]]`：

```
| x' |   | a  b  tx | | x |
| y' | = | c  d  ty | | y |
| 1  |   | 0  0  1  | | 1 |
```

即：
```
x' = a·x + b·y + tx
y' = c·x + d·y + ty
```

```json
"transform": [[1, 0, 100], [0, 1, 50]]
```

缺省为单位矩阵 `[[1, 0, 0], [0, 1, 0]]`（不产生变换）。

### 8.3 基本变换对应矩阵

| 变换 | 矩阵 |
|------|------|
| 平移 `(dx, dy)` | `[[1, 0, dx], [0, 1, dy]]` |
| 缩放 `(sx, sy)` | `[[sx, 0, 0], [0, sy, 0]]` |
| 旋转 `θ°`（逆时针） | `[[cosθ, -sinθ, 0], [sinθ, cosθ, 0]]` |
| 错切 X 方向 `tanφ` | `[[1, tanφ, 0], [0, 1, 0]]` |

### 8.4 组合变换

多个变换通过**矩阵乘法**组合为一个矩阵：

```
M = Mₙ · ... · M₂ · M₁
```

变换作用顺序为**从右向左**（右乘）：先应用 M₁，再 M₂，最后 Mₙ。

示例 — 先缩放 2 倍，再旋转 45°，再平移 (100, 50)：
```
M = T(100,50) · R(45°) · S(2)
  = [[1,0,100],[0,1,50]] · [[0.707,-0.707,0],[0.707,0.707,0]] · [[2,0,0],[0,2,0]]
  = [[0.707,-0.707,100],[0.707,0.707,50]] · [[2,0,0],[0,2,0]]
  = [[1.414,-1.414,100],[1.414,1.414,50]]
```

**约定**：变换矩阵以实体局部原点 `(0, 0)` 为变换中心。如需绕几何中心变换，需先平移到原点，再应用变换，最后平移回原位置（组合入同一矩阵）。

---

## 9. 取点与组合运算（represent / ref_op）

实体通过 `commonFields.represent` 和 `commonFields.ref_op` 两个字段声明几何运算规则。规则定义集中在本系统，实体仅存 keyword 或 method+params，不内嵌规则逻辑。

`ext_derive` 顶层字典提供用户自定义扩展，通过 `"rule:xxx"` 或 `{ "method": "rule", "name": "xxx" }` 引用。

### 9.1 取点规则（represent）

声明实体如何退化为一个代表点。`represent` 类型：`string | object`。每个 keyword 对应实体的某个属性字段作为点来源。

#### string 形式（无额外参数）

| 值 | 取点来源 | 公式 |
|----|---------|------|
| *缺省* | 按实体类型执行默认规则 | 见下方默认表 |
| `"self"` | 实体自身的 `point` 属性坐标 | `P = entity.point` |
| `"center"` | 几何中心／中点，按实体类型计算（见下表） | 见下方分表 |
| `"base"` | 实体 `position_ref` 属性引用的 point 坐标 | `P = resolve(entity.position_ref)` |
| `"origin"` | 坐标系原点，即 `(0, 0)` | `P = (0, 0)` |

**`"center"` 各实体类型计算方式：**

| 实体类型 | 计算方式 |
|---------|---------|
| line, mline, spline_fit, spline_cv, dimension | 两端点／首尾参数点的中点 |
| arc | 曲线参数 `t=0.5` 处的点（同 param_pt 方法） |
| subsegment | 曲线参数 `t = (from_t + to_t) / 2` 处的点（同 param_pt 方法） |
| circle | 直接取 `center_ref` 引用的 point |
| coord_sys | 直接取 `origin_ref` 引用的 point |
| rectangle | `min_ref` 与 `max_ref` 的中点 |
| polyline | 所有顶点的算术平均 |
| polyarc | 所有 `point_refs` 引用的 point 的算术平均 |
| polycurve | 所有 segment 端点坐标的算术平均 |
| hatch | 所有边界 polycurve 的端点坐标的算术平均 |
| region_anno | 所有 `edges_refs` 引用的 polycurve 端点坐标的算术平均 |
| 其余类型 | 所有定义坐标的算术平均（同默认回退规则） |

#### object 形式（method + 参数）

```json
{ "method": "corner",   "param": 0 }
{ "method": "bbox",     "which": "center" }
{ "method": "intersect", "curve_ref": "E2" }
```

| method | 参数 | 含义 | 公式 |
|--------|------|------|------|
| `"corner"` | `param: int` (0‑based) | 取实体定义序列中的第 param 个特征点：line/arc 0=起点 1=终点；polyline/polyarc/mline 0..(n-1)=顶点；spline_fit/spline_cv 0..(n-1)=拟合/控制点；polycurve 0..(n-1)=段端点；rectangle 0=min 1=min→max 2=max 3=max→min | `P = entity.pointAt(param)` |
| `"bbox"` | `which: "min" \| "max" \| "center" \| "top_left" \| "top_right" \| "bottom_left" \| "bottom_right"` | 包围盒特征点或四角 | `P = bbox(geometry).which` |
| `"intersect"` | `curve_ref: string` | 参考曲线实体 id | 本曲线与参考曲线的交点，存在多条时按本曲线 param 取第一个，无交点则回退 `"origin"` | `P = intersect(this, curve_ref)` |

### 9.2 组合运算（ref_op）

当一个 point 通过 `ref_pt` 引用本实体时，本实体的 `ref_op` 决定如何解释 point 的 `[dx, dy]`。`ref_op` 类型：`string | object`。

#### string 形式（无额外参数）

| 值 | 含义 | 公式 |
|----|------|------|
| `"offset"`（缺省） | 偏移叠加 | `P = P_rep + [dx, dy]` |
| `"local"` | 局部坐标 | `P = P_origin + [[cosθ, -sinθ], [sinθ, cosθ]] · [dx, dy]` |
| `"link"` | 直接转发，忽略 [dx, dy] | `P = P_rep` |

#### object 形式（method + 参数）

```json
{ "method": "project", "direction": [0, -1] }
{ "method": "closest" }
{ "method": "rule", "name": "point_at_length", "params": { "distance": 1500 } }
```

| method | 参数 | 含义 | 公式 |
|--------|------|------|------|
| `"offset"` | — | 偏移叠加（同 string 形式） | `P = P_rep + [dx, dy]` |
| `"local"` | — | 局部坐标（同 string 形式） | `P = P_origin + [[cosθ, -sinθ], [sinθ, cosθ]] · [dx, dy]` |
| `"link"` | — | 直接转发（同 string 形式） | `P = P_rep` |
| `"project"` | `direction` | 点 `[dx, dy]` 沿指定方向到本实体曲线的投影点 | `P = ray_intersect([dx, dy], direction, curve)` |
| `"closest"` | — | 点 `[dx, dy]` 到本实体曲线的最近点 | `P = closest([dx, dy], curve)` |
| `"rule"` | `name: string`, `params?: object` | 引用 ext_derive 规则 | 由 ext_derive 定义 |

### 9.3 扩展规则（ext_derive）

`ext_derive` 是顶层字典字段，为 `represent` 和 `ref_op` 提供用户自定义规则。通过 `"rule:xxx"`（string 简写）或 `{ "method": "rule", "name": "xxx" }`（object 形式）引用。

```json
{
  "ext_derive": {
    "point_at_length": {
      "for": ["line", "arc", "polyline", "polycurve"],
      "params": {
        "distance": { "type": "number", "minimum": 0, "description": "距起点的路径长度（模型单位）" }
      },
      "desc": "从起点沿曲线行走指定距离取点"
    },
    "mirror_across": {
      "for": ["point", "line", "arc", "polyline", "polycurve"],
      "params": {
        "line_ref": { "type": "string", "description": "镜像参考线实体 id" }
      },
      "desc": "将 [dx, dy] 沿指定直线镜像"
    }
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `ext_derive` | object | 否 | 键为规则名称，值为定义对象 |
| `*.for` | string[] | 是 | 适用的实体类型列表 |
| `*.params` | object | 否 | 参数描述（JSON Schema 子集），定义规则可接受的参数字段 |
| `*.desc` | string | 否 | 自然语言说明 |

引用方式：

```json
// object 形式（传入参数）
"represent": { "method": "rule", "name": "point_at_length", "params": { "distance": 1500 } }
"ref_op": { "method": "rule", "name": "mirror_across", "params": { "line_ref": "L1" } }
```

未在 `for` 中列出的实体类型使用该规则时，回退到各实体类型默认代表点。

---

## 10. 描述系统（descriptions）

这是 GSGI 区别于普通 DXF 的关键设计——支持对人类/AI 说人话。

### 10.1 四种描述手段

| 手段 | 说明 |
|------|------|
| `dimension` 实体 | 点对点精确距离标注，图纸可见 |
| `position` 实体 | 实体间位置描述：数值约束 / 基准结构化 / 纯文本 |
| `region_anno` 实体 | 区域轮廓、面积、标签及关联实体 |
| `description` | 实体级 / 属性级 / 文档级自然语言注释 |

> `measure` 实体（§5.22）计算几何量供参数引用，属于参数化数值层而非语义描述层，不出现在本体系中。

**选用规则（按判定顺序，第一条命中即采用）**

1. 是否存在**两个具体的 point 实体**，需要**图形化尺寸标注**（图纸上的尺寸线 + 数字）？
   - 是 → **`dimension`**（点对点，精确数值 + 图纸可见）
2. 是否描述**两个实体之间的位置关系**？
   - 2a. 有确定数值（等式/不等式）？→ **`position` (`constraint`)**
   - 2b. 需基于基准类型精确结构化描述？→ **`position` (`relation`)**
   - 2c. 仅抽象文字说明？→ **`position` (`text`)**
3. 是否需要定义**区域概念**（轮廓边界、面积数值、包含关系）？
   - 是 → **`region_anno`**
4. 既非可标注的尺寸，也非实体间距，也非区域概念？
   - 是 → **`description`** 字段或顶层 `descriptions`

**冲突场景示例**：

| 表达 | 选择 | 理由 |
|------|------|------|
| "车位宽 2500mm"（两个角的 point 标注） | `dimension` | 点对点 + 图纸标注线 |
| "车位间距必须 ≥30mm"（两车位的最近点） | `position` (`constraint`) | 有数值约束，无特定标注点 |
| "E1 在 P1 右侧 50mm"（基于点的基准） | `position` (`relation`, `datum: point`) | 有实体参照和基准类型 |
| "E1 在 CS1 下沿 L1 方向 3000~5000mm" | `position` (`relation`, `datum: csys`) | 坐标系 + 方向引用 + 距离区间 |
| "车位间距 30mm 且必须 ≥30mm" | **两者并存**：`dimension` 标点间距 30，`position` (`constraint`) 记最近点约束 | 点距离 ≠ 最近点距离，是两件事 |
| "车位附近 50mm 内无障碍" | `position` (`text`) | 纯文本抽象描述，无须结构化 |
| "停车区A 面积 25m²，含车位 E1/E2/E3" | `region_anno` | 区域轮廓 + 面积 + 包含关系 |
| "矩形R1中心到直线L1的距离" | `measure` (`distance`) | 参数化数值，被其他实体的参数引用 |
| "地面材质为混凝土" | `description` | 非几何信息 |

### 10.2 实体级描述

`description` 是所有实体的公共基础字段（见 [§5 公共基础字段](#公共基础字段)），直接在实体上附加，用于非尺寸/位置的一般语义注释：

```json
{
  "id": "E1",
  "type": "rectangle",
  "min": [0, 0], "max": [5000, 5000],
  "description": "这是一个标准停车位的外轮廓"
}
```

`dimension` 和 `position` 实体的 `description` 填写标注文字/位置说明：

```json
{
  "id": "D1",
  "type": "dimension",
  "p1": [0, 0], "p2": [5000, 0],
  "measurement": 5000,
  "description": "车位宽度为5000mm"
}
```

```json
{
  "id": "R1",
  "type": "position",
  "kind": "constraint",
  "ref_a": "E1", "ref_b": "E2",
  "value": 30, "operator": ">=",
  "description": "车位1长边到车位2长边最近点距离≥30"
}
```

### 10.3 属性级描述

通过顶层 `descriptions` 数组，精确到某个实体的某个属性。用于**无法用 dimension/position 表达**的语义注释：

```json
{
  "descriptions": [
    {
      "target": "E1",
      "property": "max",
      "text": "车位宽度为5000mm"
    },
    {
      "target": "E1",
      "property": null,
      "text": "这是一个标准停车位"
    },
    {
      "target": "E1",
      "property": "material",
      "text": "地面材质为混凝土"
    }
  ]
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `target` | string | 是 | 实体/块/组的 id |
| `property` | string | 否 | 属性名（如 `max`、`material`、`height`），null 表示整体描述 |
| `text` | string | 是 | 描述文字，自然语言 |

### 10.4 块级描述

块内的 entities 也可叠加描述：

```json
{
  "id": "停车位",
  "name": "停车位",
  "entities": [ ... ],
  "descriptions": [
    { "target": "停车位_rect", "property": "min", "text": "车位左下角" }
  ]
}
```

### 10.5 region_anno（区域标注）

**区域标注**既是实体类型（见 [§5.20](#520-region_anno区域标注)），也是描述系统的一部分。在描述层面，它用于声明一块区域的语义概念（轮廓、面积、标签及关联实体）。它不产生新几何线条，仅引用已有几何边界来定义区域。

```json
{
  "id": "RGA1",
  "type": "region_anno",
  "edges_refs": ["pc_region_outer", "pc_region_island_1"],
  "area": 25000000,
  "area_text": "25 m²",
  "contained_entities": ["E1", "E2", "E3"],
  "label": "停车区A",
  "description": "停车区A，面积25m²，含车位E1/E2/E3"
}
```

| 字段 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `type` | string | 是 | — | 固定为 `"region_anno"` |
| `edges_refs` | string[] | 是 | — | 闭合边界环引用的 polycurve 实体 id 数组；第一个为外边界，后续为岛 |
| `area` | number | 否 | 自动计算 | 面积值（平方单位） |
| `area_text` | string | 否 | 自动格式化 | 面积文本描述 |
| `contained_entities` | string[] | 否 | — | 区域内包含的实体 id 列表 |
| `label` | string | 否 | — | 区域标签 |
| `description` | string | 否 | — | 自然语言描述 |
| `operation` | object | 否 | — | 标记该区域是由布尔运算派生得来；详见 §5.20 `operation` 对象结构 |

**语义约定**：
- `edges_refs` 数组第一项为外边界，后续项为岛（内部挖空区域）
- 引用的 polycurve 必须构成闭合路径
- 各环应闭合且互不相交
- 岛必须位于外边界内
- 当区域轮廓完全由现有几何边构成时，区域标注本身不产生新线条，仅声明区域概念

---

## 11. AI 读取流程示例

需求："打开文件 → 读取描述 → 找到主要构件 → 读取相关属性"

```
1. 打开文件：读取最外层 "summary" 字段
   → "本文件描述停车位布置规范中的车位间距要求"

2. 遍历 entities，找到 type="block_ref" 且
   block_id 对应的 block name 包含 "停车位"
   → 定位到 block_ref 实体

3. 读取尺寸：搜索 type="dimension" 且 description
   包含 "间距"/"长边" 的实体
   → dimension.description: "车位1长边到车位2长边距离≥30"

4. 读取位置约束：搜索 type="position" 且
   ref_a / ref_b 指向该车位相关 point 的实体
   → position.description: "长边间距必须≥30"

5. AI 可进一步搜索 type="region_anno" 获取区域语义（面积、标签、关联实体）

6. AI 仅需解析 dimension / position / region_anno + description，
   无需加载全部坐标
```

**核心优势**：AI 先读文档 `summary` 了解全局意图，再按类型搜索 `dimension` / `position` / `region_anno` 获取语义信息，无需逐行解析全部坐标和属性级描述。

---

## 12. 扩展机制

```json
{
  "type": "custom_entity",
  "entity_type": "dimension",
  "properties": {
    "def_point": [0, 0],
    "text_point": [50, -10],
    "measurement": 30.0
  },
  "description": "车位间距标注"
}
```

| 字段 | 说明 |
|------|------|
| `type` | 固定为 `"custom_entity"` |
| `entity_type` | 自定义类型名，DXF 转换时降级为多段线+文字 |
| `properties` | 自定义属性键值对 |

---

## 13. 完整示例

> 本示例展示 GSGI 核心特性：**几何引用链**（`represent`/`ref_op` + `param_pt`）和**语义描述层**（`summary` + `description` + `position`）。

```json
{
  "gsgi": "1.0",
  "summary": "停车场标准车位，含轮廓、标签及间距约束",
  "properties": {
    "unit": "mm",
    "coord_precision": 1
  },
  "blocks": [
    {
      "id": "standard_parking",
      "name": "标准车位",
      "base_point": [0, 0],
      "entities": [
        { "id": "pk_rect", "type": "rectangle", "min": [-1250, -2500], "max": [1250, 2500] },
        { "id": "pk_label", "type": "text", "position_ref": "pk_rect", "text": "P", "height": 500, "represent": "center" }
      ]
    }
  ],
  "entities": [
    {
      "id": "E1",
      "type": "rectangle",
      "min": [0, 0],
      "max": [2500, 5000],
      "description": "左侧标准停车位（2500×5000mm）"
    },
    {
      "id": "E2",
      "type": "rectangle",
      "min": [2530, 0],
      "max": [5030, 5000],
      "description": "右侧标准停车位（2500×5000mm），间距30mm"
    },
    {
      "id": "PP1",
      "type": "param_pt",
      "curve_ref": "E1",
      "t": 0.5,
      "point": [1250, 2500],
      "represent": "self",
      "label": "E1 几何中心"
    },
    {
      "id": "PP2",
      "type": "param_pt",
      "curve_ref": "E2",
      "t": 0.5,
      "point": [3780, 2500],
      "represent": "self",
      "label": "E2 几何中心"
    },
    {
      "id": "P1",
      "type": "point",
      "point": [0, 0],
      "ref_pt": "PP1",
      "represent": "self",
      "description": "指向车位1中心，用于标注定位"
    },
    {
      "id": "P2",
      "type": "point",
      "point": [30, 0],
      "ref_pt": "PP2",
      "represent": "self",
      "description": "指向车位2中心，相对于 P1 的 dx=30"
    },
    {
      "id": "D1",
      "type": "dimension",
      "p1_ref": "P1",
      "p2_ref": "P2",
      "measurement": 30,
      "description": "两车位中心水平间距30mm"
    },
    {
      "id": "R1",
      "type": "position",
      "kind": "constraint",
      "ref_a": "E1",
      "ref_b": "E2",
      "value": 30,
      "operator": ">=",
      "description": "车位最近点距离必须≥30mm"
    }
  ],
  "descriptions": [
    { "target": "E1", "property": null, "text": "左侧标准车位" },
    { "target": "E2", "property": null, "text": "右侧标准车位，与E1间距30mm" }
  ]
}
```
