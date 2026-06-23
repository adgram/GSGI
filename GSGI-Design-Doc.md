# GSGI — General Simple Geometry Information Design Document

**GSGI** (General Simple Geometry Information) is a lightweight geometry data format designed for fast AI read/write and bidirectional DXF conversion.

---

## 1. Design Philosophy

| Principle | Description |
|-----------|-------------|
| **AI First** | Plain text/JSON format, self-describing, no need to parse binary or complex nesting |
| **Minimal** | Contains only valid geometry information, no CAD system variables, history records, or other metadata noise |
| **Describable** | Each element can carry textual descriptions, supporting property-level granularity |

---

## 2. File Format

Uses **JSON** as the serialization format, file extension `.gsgi`.

```json
{
  "gsgi": "1.0",
  "tags": ["architecture", "parking", "standard"],
  "summary": "This file describes parking space spacing requirements in parking layout specifications",
  "author": "Zhang San",
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

### Top-level Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `gsgi` | string | Yes | Format version, currently `"1.0"` |
| `summary` | string | No | Document summary, for AI to quickly understand file purpose |
| `tags` | string[] | No | Document classification tags, recommend 2–6 items, sorted from broad to specific |
| `author` | string | No | Document author |
| `created` | string | No | Creation time, ISO 8601 format (e.g. `"2026-06-04T10:30:00Z"`) |
| `modified` | string | No | Last modified time, ISO 8601 format |
| `properties` | object | Yes | Document global properties |
| `layers` | array | No | Layer definition list |
| `text_styles` | array | No | Text style definition list |
| `linetypes` | array | No | Linetype definition list |
| `blocks` | array | No | Block definition list |
| `entities` | array | Yes | Top-level entity list |
| `groups` | array | No | Group list |
| `ext_derive` | object | No | Point derivation and combination extension rules, see [§9.3](#93-extended-rulesext_derive) |
| `descriptions` | array | No | Explicit description list |

---

## 3. Document Properties

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

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `unit` | string | No | `"mm"` | Drawing unit: mm / cm / m / km |
| `scale` | number | No | `1.0` | Document default scale factor, e.g. `100` means 1:100. Used as default for new entity `scale`. Rendering uses only entity's own `scale`, document-level `scale` does not participate in calculation |
| `coord_precision` | int | No | `3` | Coordinate decimal places (e.g. `3` means 0.001 precision). Suggested: `3` for mm (micron level), `6` for m |
| `naming` | object | No | — | Global naming convention (see below) |

**Naming Convention (`naming`)**:

```json
"naming": {
  "id_pattern": "^[a-zA-Z_\\u0080-\\uFFFF][a-zA-Z0-9_\\-\\u0080-\\uFFFF]{0,127}$",
  "reserved_prefix": "_gsgi_"
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `id_pattern` | string | No | `^[a-zA-Z_\\u0080-\\uFFFF][a-zA-Z0-9_\\-\\u0080-\\uFFFF]{0,127}$` | Regex constraint for IDs. `\\u0080-\\uFFFF` covers Chinese, Japanese, and all Unicode ranges |
| `reserved_prefix` | string | No | `"_gsgi_"` | System reserved prefix, user IDs must not start with this |

The global naming rules serve as the base constraint for all IDs (entities / layers / blocks / groups). Each type may reference this convention and add its own rules.

### Coordinate System Convention

GSGI adopts the **AutoCAD / DXF World Coordinate System (WCS)** convention:

| Item | Convention |
|------|------------|
| Coordinate system | Right-hand Cartesian |
| X axis | Horizontal right is positive |
| Y axis | Vertical up is positive (opposite to screen Y axis direction) |
| Z axis | Perpendicular to paper outward is positive (current GSGI 1.0 uses only 2D X/Y) |
| Origin | World coordinate origin `(0, 0)` defined by the document, typically corresponds to drawing reference point |
| Angles | In radians, measured from X axis positive direction, **counterclockwise** positive (exception: text rotation uses degrees) |
| Length | Numeric unit determined by `properties.unit` |

**Convention**: All `[x, y]` coordinates are considered WCS values. Entity transforms are implemented via the [`transform`](#8-transformtransform) matrix without changing the coordinate system attribution of coordinates themselves.

### Color System

Color values support the following formats:

| Format | Example | Description |
|--------|---------|-------------|
| **ACI Index** | `7` | AutoCAD color index (1–255), standard ACI→RGB mapping table in appendix |
| **Hex RGB** | `"#FF0000"` | 24-bit true color, `"#"` + 6 hex RRGGBB, case insensitive |

**Transparency**: True color can include 8-bit transparency in format `"#AARRGGBB"` (AA=alpha, 00=fully transparent, FF=opaque). ACI colors do not support transparency.

**ByLayer / ByBlock inheritance**:

| Value | Description |
|-------|-------------|
| `"ByLayer"` | Inherit the color of the containing layer |
| `"ByBlock"` | Inherit the color of the containing block |

**JSON Schema type definition**: `color` field type is `(integer | string)`, integer treated as ACI index, string treated as Hex RGB or inheritance keyword.

---

## 4. Layer and Style Definitions

### Layers

**Default layer**: Layer `"0"` is the built-in default layer, always present (no need to declare in `layers` array). Entities without a `layer` field automatically belong to this layer. Layer `"0"` cannot be renamed, deleted, or named.

```json
{
  "id": "0",
  "color": 7,
  "linetype": "CONTINUOUS",
  "visible": true,
  "frozen": false,
  "locked": false,
  "printable": true,
  "description": "默认图层，未指定图层的实体归入此层"
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `id` | string | Yes | — | Unique identifier, i.e. layer name, referenced by `entities[i].layer`. Format follows `properties.naming` |
| `color` | int or string | No | `7` | Color, see [Color System](#color-system) |
| `linetype` | string | No | `"CONTINUOUS"` | Linetype name |
| `visible` | bool | No | `true` | Visibility |
| `frozen` | bool | No | `false` | Freeze state (frozen layers do not participate in generation/calculation) |
| `locked` | bool | No | `false` | Lock state (locked layers cannot be edited) |
| `printable` | bool | No | `true` | Whether printable |
| `description` | string | No | — | Natural language description of the layer's purpose |

### Text Styles

Text styles define available text formats for `text`, `table`, etc.

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

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `id` | string | Yes | — | Style name, referenced by entity `style` field |
| `font` | string | Yes | — | Font file name (e.g. `"txt"`, `"Arial"`, `"simsun"`), without extension |
| `width_factor` | number | No | `1.0` | Width factor |
| `oblique_angle` | number | No | `0` | Oblique angle (degrees) |
| `height` | number | No | `0` | Fixed text height, `0` means variable (specified by entity) |

**Default style**: Style `"STANDARD"` is the built-in default style, always available (no need to declare in top-level `text_styles` array).

### Linetype Definitions

Define available linetype patterns referenced by layer and entity `linetype` fields.

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

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `id` | string | Yes | — | Linetype name, referenced by `linetype` field |
| `description` | string | No | — | Linetype description |
| `pattern` | array | Yes | — | Dash/gap sequence (drawing units), empty array `[]` means solid. Positive=pen-down segment length, negative=pen-up segment length, 0=dot |

**Default linetype**: Linetype `"CONTINUOUS"` is the built-in default linetype, always available (no need to declare in top-level `linetypes` array).

---

## 5. Basic Geometry Entities (entities)

Each entity contains common base fields and type-specific fields.

### ID Naming Convention

Global naming rules are defined by [`properties.naming`](#naming-conventionnaming) (charset, format, reserved prefix). Entity IDs extend upon this:

| Rule | Description |
|------|-------------|
| Namespace | Entity IDs are unique within the `entities` array (independent from layer/block/group IDs) |
| Recommended style | Semantic naming, e.g. `parking_01`, `wall_main`, `dim_A` |

### Common Base Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Unique identifier, used for description references and groups |
| `type` | string | Yes | Entity type (see below) |
| `layer` | string | No | Layer ID, defaults to default layer |
| `transform` | object | No | Transform (translate/rotate), see [§8. Transform] |
| `color` | int or string | No | Color, overrides layer color. See [Color System](#color-system) |
| `lineweight` | int | No | Lineweight, unit 1/100 mm. Special values: `-3`=ByLayer (inherit layer) / `-2`=ByBlock (inherit block) / `-1`=Default / `0–211`=specific (e.g. `25` means 0.25mm) |
| `linetype` | string | No | Linetype name, overrides layer linetype |
| `visible` | bool | No | Visibility |
| `space` | string | No | Space: `"model"` (model space, default) / `"paper"` (paper space) |
| `scale` | number | No | Entity-level render scale, defaults to `properties.scale` on creation. **Render size = actual size × entity `scale`** (document `properties.scale` does not participate in render calculation). See "Scale Factor Mechanism" below for affected properties. Does not affect coordinate position, different from `transform` matrix scaling semantics |
| `description` | string | No | Natural language description, for general semantic annotations. See [§10. Description System](#10-description-systemdescriptions) |
| `represent` | string \| object | No | Point derivation rule. String is built-in keyword (e.g. `"center"`); object uses `method` with parameters. See [§9](#9-point-derivation-and-combination-operations-represent--ref_op) |
| `ref_op` | string \| object | No | Combination operation. String is built-in keyword (e.g. `"offset"`); object uses `method` with parameters. Default `"offset"`. See [§9](#9-point-derivation-and-combination-operations-represent--ref_op) |

**Scale Factor Mechanism**:

GSGI's two-level `scale` works via **inheritance at creation, independence at render**:

```
entity.scale initial value = properties.scale (inherited at creation)
render size               = actual size × entity.scale (render uses only entity's own scale)
```

| Level | Field | Default | Role |
|-------|-------|---------|------|
| **Document-level** | `properties.scale` | `1.0` | Default template value for entity `scale`. E.g. `100` means 1:100 |
| **Entity-level** | `entities[i].scale` | Inherited from document | Each entity's independent render scale. Changing document-level value does not affect already created entities |

**Render rule**: Only entity's own `scale` is used at render time; document `properties.scale` does not participate.

**Example**: Document scale `100`, entity text height `2.5`:
| Scenario | Entity `scale` | Render text height |
|----------|---------------|-------------------|
| New entity, unmodified | `100` (inherited) | `2.5 × 100 = 250` |
| Document scale changed to `50` | `100` (already created, unaffected) | `2.5 × 100 = 250` |
| New entity (inherits new document value) | `50` | `2.5 × 50 = 125` |
| Single entity scale changed to `200` | `200` | `2.5 × 200 = 500` |

**Properties affected by scale**:

| Entity Type | Affected Properties | Description |
|-------------|-------------------|-------------|
| `text` | `height` | Text height |
| `linetype` (layer/entity) | `pattern` dash spacing | All linetype dash/gap lengths multiplied by scale |
| `dimension` | `dim_line_offset`, arrow size, text height | Dimension geometry scales proportionally |
| `table` | Row height/column width, `text_height` | Table grid spacing and internal text |

**Semantic convention**: `scale` is **render scale**, distinct from `transform` matrix (geometric coordinate transform). Render scale affects visual size properties without changing entity coordinate positions — e.g., `text`'s `position_ref` is unaffected by scale, only text size is controlled by scale.

### Entity Type Overview

| Type | Description |
|------|-------------|
| `point` | Point |
| `param_pt` | Curve parameter point (point on curve located by parameter t) |
| `line` | Straight line |
| `polyline` | Polyline (straight segments only, can be closed) |
| `polyarc` | Polyline with arc segments (with bulge parameters) |
| `polycurve` | Composite curve (composed of line/arc/subsegment_ref/curve_ref) |
| `circle` | Circle |
| `arc` | Arc |
| `rectangle` | Rectangle (axis-aligned) |
| `text` | Text (single/multi-line, use `\n` for multi-line) |
| `spline_fit` | Fit point curve (through a series of fit points) |
| `spline_cv` | Control point curve (through a series of control points) |
| `block_ref` | Block reference |
| `xref` | External reference (references external .gsgi files) |
| `table` | Table (simplified to markdown format) |
| `subsegment` | Sub-segment (path segment between two parameter points, following the original curve path rather than straight line) |
| `dimension` | Two-point dimension (definition points, length) |
| `region_anno` | Region annotation (defines region outline, area, also part of description system, see §10.5) |
| `position` | Position relationship (invisible, describes/constrains geometric position relationships) |
| `coord_sys` | Free coordinate system (visible/invisible, defines local coordinate system by position point + rotation angle) |

---

### 5.1 point

A point consists of coordinates, containing two main parameters: its own coordinate and a target reference.

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

```json
{
  "id": "P3",
  "type": "point",
  "point": [50.0, 0.0],
  "ref_pt": { "id": "CS1", "represent": "center", "ref_op": "link" },
  "represent": "self",
  "ref_op": "offset"
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `point` | `[number, number]` | Yes | — | Coordinate `[x, y]` |
| `ref_pt` | `string \| object` | No | — | References another entity. String is the referenced entity id (backward compatible). Object form `{ "id": "CS1", "represent": "center", "ref_op": "link" }` where `id` is required, `represent` and `ref_op` are optional overrides for the referenced entity's corresponding fields. When omitted, the referenced entity's own values are used |
| `represent` | Fixed | `"self"` | Own coordinate is the representative point, cannot be modified |
| `ref_op` | Fixed | `"offset"` | Offset stacking when referenced, cannot be modified |

**Coordinate Resolution Rules**:

1. No `ref_pt`: `point` is used directly as WCS absolute coordinate
2. Has `ref_pt` (string form): Resolve the referenced entity B's representative point P_rep (per B's `represent`), then combine per B's `ref_op`.
3. Has `ref_pt` (object form): Resolve the referenced entity B's representative point P_rep (use `ref_pt.represent` if present, otherwise B's own `represent`), then combine per `ref_pt.ref_op` (if present) or B's own `ref_op`.

**Referenced Type Restriction**: When the referenced entity B's type is `point` or `param_pt`, its `represent` is fixed to `"self"` and `ref_op` is fixed to `"offset"`. Therefore in the `ref_pt` object, `represent` only allows `"self"` and `ref_op` only allows `"offset"`. Other values are invalid (silently fall back to the referenced entity's own values).

> `point`'s `represent` / `ref_op` are fixed values (see table above), user cannot modify. Other entity types declare them via commonFields.

### 5.2 param_pt (Curve Parameter Point)

Defines a point on an existing curve via parameter `t`, not directly referenced by other entities; wrapped via `point.ref_pt` for use.

```json
{
  "id": "PP1",
  "type": "param_pt",
  "curve_ref": "E3",
  "t": 0.25,
  "point": [50, 0],
  "label": "Quarter point",
  "represent": "self",
  "ref_op": "offset"
}
```

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `curve_ref` | Yes | — | Parent curve entity id (line / polyline / polyarc / polycurve / arc / spline_* type) |
| `t` | Yes | — | Parameter value, semantics depend on curve type: `line/arc/spline_*` = normalized `[0, 1]`; `polyline/polyarc` = segment index + within-segment normalized (see 5.3 description) |
| `point` | No | — | Cached coordinate `[x, y]`, for render acceleration or degraded compatibility |
| `label` | No | — | Optional label |
| `represent` | No | `"self"` | Parameter point's own coordinate |
| `ref_op` | No | `"offset"` | Offset stacking when referenced |

**Parameter `t` Semantics**:

| Curve Type | t Range | Semantics |
|------------|---------|-----------|
| `line` | `[0, 1]` | Linear interpolation, 0=start, 1=end |
| `polyline` / `polyarc` | Open: `[0, N]`, Closed: `[0, N)` | Integer part is segment index (0-based), fractional part is within-segment normalized position `[0, 1)`. `N` = number of segments (open=vertex count−1, closed=vertex count, including closing segment). Example: `t=1.5` means midpoint of segment 1 |
| `arc` | `[0, 1]` | Arc-length proportional interpolation, `t=0` = start_ref, `t=1` = end_ref (for circular arcs, arc-length ratio equals central angle ratio) |
| `spline_fit` / `spline_cv` | `[0, 1]` | Normalized curve parameter u |

### 5.3 line

```json
{ "id": "E2", "type": "line", "start_ref": "P1", "end_ref": "P2", "represent": "center", "ref_op": "offset" }
```

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `start_ref` | Yes | — | Point entity id referenced for start point |
| `end_ref` | Yes | — | Point entity id referenced for end point |
| `represent` | No | `"center"` | Midpoint of segment |
| `ref_op` | No | `"offset"` | Offset stacking when referenced |

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

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `points` | Yes | — | Vertex array `[[x,y], ...]` |
| `closed` | No | `false` | Whether closed (connect first and last) |
| `represent` | No | `"center"` | Geometric center |
| `ref_op` | No | `"offset"` | Offset stacking when referenced |

**Design note**: `polyline` uses inline coordinates `points` (rather than point references) because polyline contains only straight segments with simple data structure; inline coordinates are more concise. For polyarc with bulge parameters and more complex curve types, point references are used to support coordinate reuse.

### 5.5 polyarc (Arc Polyline)

A type of polyline that supports arc segments via bulge parameters in addition to straight segments. Vertices are referenced via points.

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

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `point_refs` | Yes | — | Point entity id array for vertices, length = N |
| `bulges` | Yes | — | Bulge value array, length = N. `0`=straight segment, positive=CCW arc, negative=CW arc. `bulge = tan(θ/4)`, where θ is the arc central angle |
| `closed` | No | `false` | Whether closed (connect first and last) |
| `represent` | No | `"center"` | Geometric center |
| `ref_op` | No | `"offset"` | Offset stacking when referenced |

**Bulge to Arc Formula**:
- Central angle `θ = 4 · atan(|bulge|)`
- Direction: `bulge > 0` CCW, `bulge < 0` CW
- Radius `r = chord / (2 · sin(θ/2))`

**Bulge to Midpoint (polyarc to 3P arc conversion)**:
- Chord midpoint `C = ((Sx+Ex)/2, (Sy+Ey)/2)`
- Perpendicular distance `d = chord/2 · (1−b²) / (2·b)` where `b = bulge`; or equivalently `d = r − cos(θ/2)·r`
- Arc midpoint `M = C + d · normalize(perp(S→E))` where `perp((dx,dy)) = (−dy, dx)` for `b>0` (CCW), `perp((dx,dy)) = (dy, −dx)` for `b<0` (CW)

### 5.6 polycurve (Composite Curve)

A continuous path composed of multiple basic segment types. Each segment can be line, arc, subsegment reference, or curve reference.

```json
{
  "id": "E_pc1",
  "type": "polycurve",
  "segments": [
    { "type": "line", "start_ref": "P1", "end_ref": "P2" },
    { "type": "arc", "start_ref": "P2", "mid_ref": "P3", "end_ref": "P4" },
    { "type": "subsegment_ref", "ref": "ss1" },
    { "type": "curve_ref", "ref": "E3", "reverse": false }
  ],
  "closed": false,
  "represent": "center",
  "ref_op": "offset"
}
```

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `segments` | Yes | — | End-to-end connected segment sequence; segment endpoints auto-connect |
| `segments[].type` | Yes | — | `line` / `arc` / `subsegment_ref` / `curve_ref` |
| `closed` | No | `false` | Whether closed (last segment end → first segment start) |
| `represent` | No | `"center"` | Geometric center |
| `ref_op` | No | `"offset"` | Offset stacking when referenced |

**Segment type details**:

| type | Required Fields | Description |
|------|----------------|-------------|
| `line` | `start_ref`, `end_ref` | Straight segment |
| `arc` | `start_ref`, `mid_ref`, `end_ref` | Arc segment (3-point definition, three points uniquely determine a circular arc) |
| `subsegment_ref` | `ref` | Reference existing subsegment entity id |
| `curve_ref` | `ref`, `reverse` (optional) | Reference existing curve entity id (line/polyline/polyarc/spline_*/polycurve), `reverse=true` traverses path in reverse |

**Semantic convention**: Segments connect end-to-end in order. For line segments, `start_ref` must equal the previous segment's `end_ref`; for arc segments, `start_ref` must equal the previous segment's `end_ref`, and `end_ref` must equal the next segment's `start_ref` (validation rule: `segments[i].end_ref == segments[i+1].start_ref` for adjacent segments).

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

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `center_ref` | Yes | — | Point entity id referenced for center |
| `r` | Yes | — | Radius (>0) |
| `represent` | No | `"center"` | Center point |
| `ref_op` | No | `"offset"` | Offset stacking when referenced |

### 5.8 arc

Three-point defined circular arc. Three points uniquely determine a circle and the arc segment on it. Points must not be collinear.

```json
{
  "id": "E5",
  "type": "arc",
  "start_ref": "P1",
  "mid_ref": "P2",
  "end_ref": "P3",
  "represent": "center",
  "ref_op": "offset"
}
```

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `start_ref` | Yes | — | Point entity id referenced for start point |
| `mid_ref` | Yes | — | Point entity id referenced for mid point (determines arc direction) |
| `end_ref` | Yes | — | Point entity id referenced for end point |
| `represent` | No | `"center"` | Point at curve parameter `t=0.5` |
| `ref_op` | No | `"offset"` | Offset stacking when referenced |

**Mathematical basis**: Three non-collinear points uniquely determine a circumscribed circle. `param_pt.t` semantics remain `[0, 1]` normalized, `t=0` = start, `t=1` = end, linearly mapped along arc length (which equals central angle ratio for circular arcs).

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

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `min_ref` | Yes | — | Point entity id referenced for bottom-left corner |
| `max_ref` | Yes | — | Point entity id referenced for top-right corner |
| `represent` | No | `"center"` | Rectangle geometric center |
| `ref_op` | No | `"offset"` | Offset stacking when referenced |

**Semantic convention**: Vertices in order `(min[0], min[1]) → (max[0], min[1]) → (max[0], max[1]) → (min[0], max[1])` forming a closed rectangle.

### 5.10 text

```json
{
  "id": "E7",
  "type": "text",
  "position_ref": "P6",
  "text": "Parking A",
  "height": 2.5,
  "rotation": 0,
  "represent": "base",
  "ref_op": "offset"
}
```

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `position_ref` | Yes | — | Point entity id referenced for insertion point |
| `text` | Yes | — | Text content |
| `height` | No | `2.5` | Text height |
| `rotation` | No | `0` | Rotation angle (degrees) |
| `represent` | No | `"base"` | Base point (insertion point) |
| `ref_op` | No | `"offset"` | Offset stacking when referenced |

**Multi-line text**: Use `\n` in the `text` field for multi-line content. DXF export: when `text` contains `\n`, exports as `MTEXT`; otherwise exports as `TEXT`.

### 5.11 block_ref (Block Reference)

```json
{
  "id": "E8",
  "type": "block_ref",
  "block_id": "parking",
  "position_ref": "P7",
  "rotation": 0,
  "scale_x": 1,
  "scale_y": 1,
  "attrs": {
    "parking_number": "A1"
  },
  "represent": "base",
  "ref_op": "offset"
}
```

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `block_id` | Yes | — | Referenced block definition id |
| `position_ref` | Yes | — | Point entity id referenced for insertion point |
| `rotation` | No | `0` | Rotation angle (radians) |
| `scale_x` | No | `1` | X scale factor |
| `scale_y` | No | `1` | Y scale factor |
| `attrs` | No | — | Attribute override values for this instance |
| `represent` | No | `"base"` | Base point (insertion point) |
| `ref_op` | No | `"offset"` | Offset stacking when referenced |

**Multi-line text**: Use `\n` in the `text` field for multi-line content. DXF export: when `text` contains `\n`, exports as `MTEXT`; otherwise exports as `TEXT`.

### 5.12 spline_fit (Fit Point Curve)

A smooth curve defined through a series of fit points (passes through all points).

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

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `fit_point_refs` | Yes | — | Point entity id array for fit points |
| `degree` | No | `3` | Curve degree (2 or 3) |
| `closed` | No | `false` | Whether closed |
| `tolerance` | No | `0.0` | Fit tolerance |
| `represent` | No | `"center"` | Curve midpoint |
| `ref_op` | No | `"offset"` | Offset stacking when referenced |

### 5.13 spline_cv (Control Point Curve)

A Bezier/B-spline curve defined through control points (curve does not pass through control points, is "pulled" toward them).

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

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `control_point_refs` | Yes | — | Point entity id array for control points |
| `knots` | No | Auto-generated | Knot vector (uniform). Length must satisfy `len(knots) = len(control_point_refs) + degree + 1` |
| `weights` | No | All `1` | Control point weights (NURBS), length same as `control_point_refs` |
| `degree` | No | `3` | Curve degree |
| `closed` | No | `false` | Whether closed |
| `represent` | No | `"center"` | Curve midpoint |
| `ref_op` | No | `"offset"` | Offset stacking when referenced |

### 5.14 table (Table)

Represented using markdown table syntax. Cells support prefix markers for non-text content and cell references:

| Prefix | Meaning | Example |
|--------|---------|---------|
| None | Plain text | `Angle steel` |
| `@实体ID` | block_ref — displays the referenced block | `@BR1` |
| `%实体ID` | xref — displays the referenced external reference | `%XR1` |
| `^R行C列` | Cell reference — displays content from another cell (row, col 0-based) | `^R0C0` |

Escape with double prefix when literal content starts with `@`/`%`/`^`: `@@x` → `@x`.

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
  "markdown": "| No. | Width(mm) | Length(mm) |\n|-----|-----------|------------|\n| Parking Dimension Table | ^R0C0 | ^R0C0 |\n| A1  | 2500      | 5000       |\n| B1  | @BR1      | 6000       |",
  "represent": "base",
  "ref_op": "offset"
}
```

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `position_ref` | Yes | — | Point entity id referenced for table top-left insertion point |
| `width` | No | Auto | Table total width |
| `height` | No | Auto | Table total height |
| `col_widths` | No | Equal | Column width array |
| `row_heights` | No | Equal | Row height array |
| `style` | No | `"STANDARD"` | Text style |
| `text_height` | No | `2.5` | Text height |
| `markdown` | Yes | — | Markdown table syntax with prefix markers for cell types; `^R{r}C{c}` for cell references |
| `represent` | No | `"base"` | Base point (table top-left) |
| `ref_op` | No | `"offset"` | Offset stacking when referenced |

**Cell reference rules**:
- `^R{r}C{c}` references another cell in the same table (row `r`, col `c`, 0-based)
- Any shape of cross-cell reference is supported (not limited to rectangular merges)
- Reference chains resolve recursively; circular references must be detected and rejected by implementations
- The `merges` field is **removed** — all cell spanning is expressed via `^R…C…`

### 5.15 subsegment (Sub-segment)

A path segment between two parameter points, preserving the original curve path rather than a straight line connection.

```json
{
  "id": "E16",
  "type": "subsegment",
  "curve_ref": "E3",
  "from_t": 0.25,
  "to_t": 0.75,
  "from_point": [50, 0],
  "to_point": [100, 25],
  "label": "Middle segment",
  "represent": "center",
  "ref_op": "offset"
}
```

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `curve_ref` | Yes | — | Referenced parent curve entity id (line / polyline / polyarc / polycurve / arc / spline_* type) |
| `from_t` | Yes | — | Start parameter, semantics same as [5.2 param_pt t](#52-param_pt-curve-parameter-point) |
| `to_t` | Yes | — | End parameter (must be ≥ `from_t`) |
| `from_point` | No | — | Start cached coordinate `[x, y]` |
| `to_point` | No | — | End cached coordinate `[x, y]` |
| `label` | No | — | Optional label |
| `represent` | No | `"center"` | Point at subsegment curve parameter `t=(from_t+to_t)/2` |
| `ref_op` | No | `"offset"` | Offset stacking when referenced |

### 5.16 dimension (Two-point Dimension)

Dimensions the distance between two points. Dimension points are positioned by referencing `point` entities.

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

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `p1_ref` | Yes | — | Point entity id referenced for dimension start |
| `p2_ref` | Yes | — | Point entity id referenced for dimension end |
| `measurement` | Yes | — | Dimension length (value in unit) |
| `text` | No | `measurement` value | Dimension text |
| `dim_line_offset` | No | `10` | Dimension line offset distance |
| `category` | No | `"aligned"` | Dimension type: `horizontal` / `vertical` / `aligned` |
| `represent` | No | `"center"` | Midpoint of the two dimension points |
| `ref_op` | No | `"offset"` | Offset stacking when referenced |

**Semantic convention**:
- When dimension points are on geometry, first define `param_pt`, then define `point` via `ref_pt` referencing that `param_pt`, dimension references that `point`

### 5.17 region_anno (Region Annotation)

Region annotation defines a region's outline and area, and can associate entities contained within. It is both an entity type and part of the description system (see [§10.5](#105-region_anno-region-annotation)), together with `dimension`, `position`, and `description` forming four descriptive mechanisms.

```json
{
  "id": "RGA1",
  "type": "region_anno",
  "edges_refs": ["pc_region_outer", "pc_region_island_1"],
  "area": 25000000,
  "area_text": "25 m²",
  "contained_entities": ["E1", "E2", "E3"],
  "label": "Parking Zone A",
  "fill": {
    "pattern": "ANSI31",
    "color": 3,
    "angle": 0.785,
    "scale": 1.0
  },
  "represent": "center",
  "ref_op": "offset"
}
```

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `edges_refs` | Yes | — | Polycurve entity id array for closed boundary loops; first is outer boundary, subsequent are islands |
| `area` | No | Auto-calculated | Area value (square units) |
| `area_text` | No | Auto-formatted | Area text description |
| `contained_entities` | No | — | Entity id list contained within the region |
| `label` | No | — | Region label |
| `fill` | No | — | Fill pattern (see [fill Object Structure](#fill-object-structure)) — when present, this region_anno acts as a hatch area |
| `represent` | No | `"center"` | Region geometric center |
| `ref_op` | No | `"offset"` | Offset stacking when referenced |
| `operation` | No | — | Marks this region as derived from boolean operations (see below) |

**Semantic convention**:
- First item of `edges_refs` array is outer boundary, subsequent items are islands (internal holes)
- Referenced polycurves must form closed paths
- Each loop should be closed and non-intersecting
- Islands must be inside the outer boundary
- When region outline is entirely composed of existing geometry edges, region annotation itself produces no new lines, only declares the region concept
- When `fill` is present, the region_anno additionally defines a visual fill area — the boundary model (`edges_refs`) is shared for both region semantics and fill geometry
- DXF HATCH is mapped to/from `region_anno` + `fill`

#### operation Object Structure

| Sub-field | Type | Required | Description |
|-----------|------|----------|-------------|
| `op` | string | Yes | Operation type: `"union"`, `"intersect"`, `"difference"`, `"xor"` |
| `refs` | string[] | Yes | Operation operands, referencing other `region_anno` entity ids, must be ≥2 |
| `desc` | string | No | Human-readable description (e.g. "Intersection of shadow fan and building footprint") |

**Semantic conventions**:
- `operation` only declares the derivation source of the region, does not drive computation. Rendering/display uses `edges_refs` as the actual geometry
- `refs` must reference entities of type `region_anno`
- Chained references are allowed (A as operand of B, B as operand of C)
- Circular references are prohibited (implementations should detect and report errors)
- When `operation` contradicts `edges_refs` content, `edges_refs` is the actual geometry, `operation` is historical semantics only

#### fill Object Structure

When `fill` is present, the region_anno becomes a visual fill area. The boundary geometry is defined by `edges_refs` (shared with region semantics).

| Sub-field | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `pattern` | string | Yes | — | Fill pattern name; `"SOLID"` for solid fill, others like `"ANSI31"`, `"AR-CONC"` |
| `color` | int/string | No | Inherited | Fill color (ACI index or hex RGB), see [Color System](#color-system) |
| `angle` | number | No | `0` | Pattern rotation angle (radians) |
| `scale` | number | No | `1.0` | Pattern scale density |

**Semantic conventions**:
- `fill` absent or `null` → region_anno is pure semantic region (no visual fill)
- `fill` present → region_anno additionally defines a hatch area, mapped to DXF `HATCH` on export
- Multiple disconnected fill regions are expressed as multiple `region_anno` entities

### 5.18 position (Position Relationship)

**Invisible entity**, no graphical representation. Used to describe or constrain positional relationships between two geometry references, or to describe abstract adjacency semantics. **Distance values always refer to the shortest distance between two entities**, not specific points.

Supports three modes:

| Mode | kind value | Description | Example |
|------|-----------|-------------|---------|
| Constraint | `"constraint"` | Numeric (including equality/inequality) constraint on entity spacing | Shortest distance between E1 and E2 ≥ 30mm |
| Text | `"text"` | Plain text abstract description | "No other geometry within 50mm of the point" |
| Relation | `"relation"` | Precise structured description with entity references, based on datum type | "E1 is 50mm to the right of P1" |

#### constraint Example

```json
{
  "id": "R1",
  "type": "position",
  "kind": "constraint",
  "ref_a": "E1",
  "ref_b": "E2",
  "value": 5000,
  "description": "Shortest distance between parking 1 and parking 2 is 5000mm",
  "represent": "origin",
  "ref_op": "offset"
}
```

> `position`'s `represent`/`ref_op` default to `"origin"`/`"offset"` (omitted in all examples below, same effect).

```json
{
  "id": "R2",
  "type": "position",
  "kind": "constraint",
  "ref_a": "E1",
  "ref_b": "E2",
  "value": 30,
  "operator": ">=",
  "description": "Shortest distance between parking 1 long edge and parking 2 long edge ≥ 30mm"
}
```

#### text Example (Plain Text)

```json
{
  "id": "R3",
  "type": "position",
  "kind": "text",
  "description": "No other geometry within 50mm of the point"
}
```

```json
{
  "id": "R4",
  "type": "position",
  "kind": "text",
  "ref_a": "P1",
  "description": "No other geometry within 50mm of point P1"
}
```

`text` mode is only for abstract descriptions that don't need structuring. For precise expression, use `relation` mode:

#### relation Example (Enhanced Structured)

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
  "description": "E1 is 5000mm from P1"
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
  "description": "E1 is at 3000~5000mm along L1 direction under CS1"
}
```

`direction_ref` references a line entity id to define direction. `distance` as single value means exact offset, as `[min, max]` array means closed interval. When object type does not match `datum`, automatically use bounding rectangle/box approximation.

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `kind` | Yes | — | `"constraint"` / `"text"` / `"relation"` |
| `ref_a` | No | — | **Subject** geometry reference (entity id); required for constraint and relation, optional for text |
| `ref_b` | No | — | **Object** geometry reference (entity id); required for constraint and relation, optional for text |
| `value` | No | — | Distance value (constraint mode only); shortest distance between two entities |
| `operator` | No | `"=="` | Constraint operator: `>=` / `<=` / `>` / `<` / `==` (constraint mode only) |
| `datum` | No | — | Datum type (relation mode only), determines available `relationship`: `"point"` / `"csys"` / `"line"` / `"region"` / `"rect"` |
| `relationship` | No | — | Determined by `datum`, see table below (relation mode only) |
| `params` | No | — | Relationship parameters (relation mode only) |
| `description` | No | — | Natural language description; relation mode recommended to write in subject→object direction |
| `represent` | No | `"origin"` | No geometric form, falls back to origin |
| `ref_op` | No | `"offset"` | Offset stacking when referenced |

**datum ↔ relationship reference table**:

| datum | relationship | params | Description |
|-------|-------------|--------|-------------|
| `"point"` | `"distance"` | `value`: number | Distance from object to point |
| | `"align_x"` | — | Object's x coordinate same as point |
| | `"align_y"` | — | Object's y coordinate same as point |
| `"csys"` | `"offset"` | `direction_ref`: string (line id), `distance`: number or [number, number] (single or closed interval) | Object offset along specified direction in coordinate system |
| `"line"` | `"parallel"` | — | Object parallel to line |
| | `"perpendicular"` | — | Object perpendicular to line |
| | `"collinear"` | — | Object collinear with line |
| | `"at_angle"` | `angle`: number | Object at specified angle to line |
| `"region"` | `"inside"` | — | Object inside closed region |
| | `"outside"` | — | Object outside closed region |
| | `"on_boundary"` | — | Object on region boundary |
| | `"overlaps"` | — | Object overlaps with region |
| `"rect"` | `"top_aligned"` | `gap`: number (optional) | Top aligned |
| | `"bottom_aligned"` | `gap`: number (optional) | Bottom aligned |
| | `"left_aligned"` | `gap`: number (optional) | Left aligned |
| | `"right_aligned"` | `gap`: number (optional) | Right aligned |
| | `"h_centered"` | `gap`: number (optional) | Horizontally centered |
| | `"v_centered"` | `gap`: number (optional) | Vertically centered |
| | `"centered"` | `gap`: number (optional) | Fully centered |

**Semantic convention**:
- `position` entity produces no graphical output, exists only as a semantic layer
- `constraint` mode: shortest distance from `ref_a` to `ref_b` must satisfy `operator value`. `operator` defaults to `"=="` (exact distance)
- `text` mode: plain text abstract description, may have `ref_a` as context aid, no structured parsing capability
- `relation` mode: subject datum type defined by `datum`, precisely describes object's relationship to subject per `relationship` + `params`
- When object type does not match `datum`, automatically use object's bounding rectangle/box as approximation to match datum type
- AI reading only needs to scan `kind` + `description` to understand position relationship, no need to parse all geometry

---

### 5.19 xref (External Reference)

References geometric content from external `.gsgi` files, similar to `block_ref` but loaded from external files.

```json
{
  "id": "E20",
  "type": "xref",
  "file_path": "other_floor.gsgi",
  "block_id": "parking",
  "position_ref": "P21",
  "rotation": 0,
  "scale_x": 1,
  "scale_y": 1,
  "represent": "base",
  "ref_op": "offset"
}
```

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `file_path` | Yes | — | External `.gsgi` file path (relative or absolute) |
| `block_id` | No | — | Block definition id to reference in external file; if absent, load all top-level entities |
| `position_ref` | Yes | — | Point entity id referenced for insertion point |
| `rotation` | No | `0` | Rotation angle (radians) |
| `scale_x` | No | `1` | X scale factor |
| `scale_y` | No | `1` | Y scale factor |
| `represent` | No | `"base"` | Base point (insertion point) |
| `ref_op` | No | `"offset"` | Offset stacking when referenced |

**Semantic convention**:
- If `file_path` resolution fails, implementation should error rather than silently ignore
- Circular external references not supported (A references B, B references A); implementation should detect and error
- On DXF conversion, xref maps to `XREF` / `INSERT` + XDATA recording external path

### 5.20 coord_sys (Free Coordinate System)

Defines a local coordinate system, composed of a position point and a rotation angle. Other entities (e.g. point) can reference this coordinate system via `ref_pt`, interpreting their coordinates as local.

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

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `origin_ref` | Yes | — | Point entity id referenced for origin |
| `rotation` | No | `0` | Angle between local X axis and world X axis (radians), CCW positive |
| `visible` | No | `false` | Whether to show coordinate system icon in graphics |
| `represent` | No | `"center"` | Use point referenced by `origin_ref` |
| `ref_op` | No | `"local"` | When referenced, transform as local coordinates |

**Coordinate system transform**: When a point references this coordinate system, point coordinates convert to WCS: `P_wcs = P_origin + R · P_local`, where `R = [[cosθ, -sinθ], [sinθ, cosθ]]`. Example above θ=0.7854 (45°).

## 6. Block Definitions (blocks)

```json
{
  "id": "parking",
  "name": "Parking Space",
  "base_point": [0, 0],
  "attributes": {
    "parking_number": { "type": "string", "default": "P" }
  },
  "entities": [
    {
      "id": "parking_rect",
      "type": "rectangle",
      "min": [-1250, -2500],
      "max": [1250, 2500]
    },
    {
      "id": "parking_text",
      "type": "text",
      "position": [0, 0],
      "text": "{parking_number}",
      "height": 500
    }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Unique identifier, referenced by block_ref. Format follows `properties.naming` |
| `name` | string | No | Block name (defaults to id) |
| `base_point` | [number, number] | No | Base point coordinates, default `[0, 0]` |
| `attributes` | object | No | Declared parameters for parametric block instantiation |
| `entities` | array | Yes | Entity list within block (auto-generate id if absent) |

Entities within blocks share the same entity type definitions as top-level entities.

**No nesting**: Block definitions must not contain `block_ref` or `xref` type entities. All block references must exist flattened in top-level entities, no recursive nesting.

**Simplified coordinates in blocks**: Entities in block definitions support inline coordinates (e.g. `rectangle`'s `min`/`max`, `text`'s `position`) rather than mandatory point references. This is for block self-containment — blocks are typically instantiated as block_ref, and their entity coordinates should be directly expressed in block local coordinate system without needing extra point entities.

---

## 7. Groups (groups)

```json
{
  "id": "G1",
  "name": "Parking Group",
  "members": ["E1", "E2", "E3"]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Unique identifier, format follows `properties.naming` |
| `name` | string | No | Group name |
| `members` | string[] | Yes | Entity ID / Block ID list |

Groups support nesting: members can reference other group ids.

**Circular reference prohibited**: Group nesting must not form circular references (e.g. G1→G2→G1). Implementation must detect and reject any membership addition that would cause a cycle.

---

## 8. Transform (transform)

Each entity can carry a transform, applying an affine transform to entity coordinates without modifying the original coordinates.

### 8.1 Definition

```json
"transform": [[1, 0, 100], [0, 1, 50]]
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `matrix` | `[[number, number, number], [number, number, number]]` | No | `[[1,0,0],[0,1,0]]` | 2×3 affine transform matrix (see below); defaults to identity |

**Resolution rules**:
1. When `transform` is array `[[a,b,tx],[c,d,ty]]`, equivalent to `{ matrix: [[a,b,tx],[c,d,ty]] }` (shorthand)
2. When `transform` is absent, identity matrix (no transform)

### 8.2 Matrix Definition

Uses **2×3 affine transform matrix** `[[a, b, tx], [c, d, ty]]`:

```
| x' |   | a  b  tx | | x |
| y' | = | c  d  ty | | y |
| 1  |   | 0  0  1  | | 1 |
```

i.e.:
```
x' = a·x + b·y + tx
y' = c·x + d·y + ty
```

```json
"transform": [[1, 0, 100], [0, 1, 50]]
```

Default is identity matrix `[[1, 0, 0], [0, 1, 0]]` (no transform).

### 8.3 Basic Transform Matrices

| Transform | Matrix |
|-----------|--------|
| Translate `(dx, dy)` | `[[1, 0, dx], [0, 1, dy]]` |
| Scale `(sx, sy)` | `[[sx, 0, 0], [0, sy, 0]]` |
| Rotate `θ°` (CCW) | `[[cosθ, -sinθ, 0], [sinθ, cosθ, 0]]` |
| Shear X direction `tanφ` | `[[1, tanφ, 0], [0, 1, 0]]` |

### 8.4 Combined Transforms

Multiple transforms combine into one matrix via **matrix multiplication**:

```
M = Mₙ · ... · M₂ · M₁
```

Transform order is **right to left** (right-multiply): apply M₁ first, then M₂, finally Mₙ.

Example — scale 2×, then rotate 45°, then translate (100, 50):
```
M = T(100,50) · R(45°) · S(2)
  = [[1,0,100],[0,1,50]] · [[0.707,-0.707,0],[0.707,0.707,0]] · [[2,0,0],[0,2,0]]
  = [[0.707,-0.707,100],[0.707,0.707,50]] · [[2,0,0],[0,2,0]]
  = [[1.414,-1.414,100],[1.414,1.414,50]]
```

**Convention**: Transform matrix uses entity local origin `(0, 0)` as transform center. To transform about geometric center, first translate to origin, apply transform, then translate back (combined into same matrix).

---

## 9. Point Derivation and Combination Operations (represent / ref_op)

Entities declare geometric operation rules via `commonFields.represent` and `commonFields.ref_op`. Rule definitions are centralized in this system; entities store only keywords or method+params, not embedded rule logic.

`ext_derive` top-level dictionary provides user-defined extensions, referenced via `"rule:xxx"` or `{ "method": "rule", "name": "xxx" }`.

### 9.1 Point Derivation Rules (represent)

Declares how an entity reduces to a representative point. `represent` type: `string | object`. Each keyword corresponds to an entity attribute field as the point source.

#### string form (no additional parameters)

| Value | Point Source | Formula |
|-------|-------------|---------|
| *absent* | Per entity type default rule | See default table below |
| `"self"` | Entity's own `point` attribute coordinate | `P = entity.point` |
| `"center"` | Geometric center/midpoint, calculated per entity type (see table below) | See per-type table |
| `"base"` | Coordinate from entity's `position_ref` attribute point | `P = resolve(entity.position_ref)` |
| `"origin"` | Coordinate system origin, i.e. `(0, 0)` | `P = (0, 0)` |

**`"center"` calculation per entity type:**

| Entity Type | Calculation Method |
|-------------|-------------------|
| line, spline_fit, spline_cv, dimension | Midpoint between two endpoints / first and last parameter points |
| arc | Point at curve parameter `t=0.5` (same as param_pt method) |
| subsegment | Point at curve parameter `t = (from_t + to_t) / 2` (same as param_pt method) |
| circle | Directly use point referenced by `center_ref` |
| coord_sys | Directly use point referenced by `origin_ref` |
| rectangle | Midpoint between `min_ref` and `max_ref` |
| polyline | Arithmetic mean of all vertices |
| polyarc | Arithmetic mean of all `point_refs` referenced points |
| polycurve | Arithmetic mean of all segment endpoint coordinates |
| region_anno | Arithmetic mean of all `edges_refs` polycurve endpoint coordinates |
| Other types | Arithmetic mean of all definition coordinates (same as default fallback) |

#### object form (method + parameters)

```json
{ "method": "corner",   "param": 0 }
{ "method": "bbox",     "which": "center" }
{ "method": "intersect", "curve_ref": "E2" }
{ "method": "param",    "t": 0.25 }
```

| method | Parameters | Meaning | Formula |
|--------|-----------|---------|---------|
| `"corner"` | `param: int` (0‑based) | Take the param-th feature point in entity's definition sequence: line/arc 0=start 1=end; polyline/polyarc 0..(n-1)=vertices; spline_fit/spline_cv 0..(n-1)=fit/control points; polycurve 0..(n-1)=segment endpoints; rectangle 0=min 1=min→max 2=max 3=max→min | `P = entity.pointAt(param)` |
| `"bbox"` | `which: "min" \| "max" \| "center" \| "top_left" \| "top_right" \| "bottom_left" \| "bottom_right"` | Bounding box feature point or corner | `P = bbox(geometry).which` |
| `"intersect"` | `curve_ref: string` | Reference curve entity id | Intersection of this curve with reference curve; if multiple, take first along this curve's param; no intersection falls back to `"origin"` | `P = intersect(this, curve_ref)` |
| `"param"` | `t: number` | Evaluate point on own curve at parameter `t`. `t` semantics identical to [§5.2 param_pt](#52-param_pt-curve-parameter-point): line/arc/spline_* normalized `[0, 1]`; polyline/polyarc open `[0, N]` closed `[0, N)`. Entities without `getCurve()` (text/block_ref etc.) fall back to `"origin"` | `P = curve.eval(t)` |

### 9.2 Combination Operation (ref_op)

When a point references this entity via `ref_pt`, this entity's `ref_op` determines how the point's `[dx, dy]` is interpreted. `ref_op` type: `string | object`.

#### string form (no additional parameters)

| Value | Meaning | Formula |
|-------|---------|---------|
| `"offset"` (default) | Offset stacking | `P = P_rep + [dx, dy]` |
| `"local"` | Local coordinates | `P = P_origin + [[cosθ, -sinθ], [sinθ, cosθ]] · [dx, dy]` |
| `"link"` | Direct forwarding, ignores [dx, dy] | `P = P_rep` |

#### object form (method + parameters)

```json
{ "method": "project", "direction": [0, -1] }
{ "method": "closest" }
{ "method": "rule", "name": "point_at_length", "params": { "distance": 1500 } }
```

| method | Parameters | Meaning | Formula |
|--------|-----------|---------|---------|
| `"offset"` | — | Offset stacking (same as string form) | `P = P_rep + [dx, dy]` |
| `"local"` | — | Local coordinates (same as string form) | `P = P_origin + [[cosθ, -sinθ], [sinθ, cosθ]] · [dx, dy]` |
| `"link"` | — | Direct forwarding (same as string form) | `P = P_rep` |
| `"project"` | `direction` | Project point `[dx, dy]` along specified direction onto this entity's curve | `P = ray_intersect([dx, dy], direction, curve)` |
| `"closest"` | — | Nearest point from `[dx, dy]` to this entity's curve | `P = closest([dx, dy], curve)` |
| `"rule"` | `name: string`, `params?: object` | Reference ext_derive rule | Defined by ext_derive |

### 9.3 Extended Rules (ext_derive)

`ext_derive` is a top-level dictionary field providing user-defined rules for `represent` and `ref_op`. Referenced via `"rule:xxx"` (string shorthand) or `{ "method": "rule", "name": "xxx" }` (object form).

```json
{
  "ext_derive": {
    "point_at_length": {
      "for": ["line", "arc", "polyline", "polycurve"],
      "params": {
        "distance": { "type": "number", "minimum": 0, "description": "Path length from start (model units)" }
      },
      "desc": "Walk specified distance along curve from start and take point"
    },
    "mirror_across": {
      "for": ["point", "line", "arc", "polyline", "polycurve"],
      "params": {
        "line_ref": { "type": "string", "description": "Mirror reference line entity id" }
      },
      "desc": "Mirror [dx, dy] across the specified line"
    }
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ext_derive` | object | No | Key is rule name, value is definition object |
| `*.for` | string[] | Yes | Applicable entity type list |
| `*.params` | object | No | Parameter description (JSON Schema subset), defines acceptable parameter fields for the rule |
| `*.desc` | string | No | Natural language description |

Reference methods:

```json
// object form (with parameters)
"represent": { "method": "rule", "name": "point_at_length", "params": { "distance": 1500 } }
"ref_op": { "method": "rule", "name": "mirror_across", "params": { "line_ref": "L1" } }
```

When an entity type not listed in `for` uses the rule, it falls back to the entity type's default representative point.

---

## 10. Description System (descriptions)

This is the key design that distinguishes GSGI from ordinary DXF — supporting human/AI communication in natural language.

### 10.1 Four Descriptive Mechanisms

| Mechanism | Description |
|-----------|-------------|
| `dimension` entity | Point-to-point exact distance annotation, visible on drawing |
| `position` entity | Inter-entity position description: numeric constraint / datum-structured / plain text |
| `region_anno` entity | Region outline, area, label, and associated entities |
| `description` | Entity-level / property-level / document-level natural language annotation |

**Selection rules (check in order, use first match)**:

1. Are there **two specific point entities** needing **graphical dimension annotation** (dimension line + number on drawing)?
   - Yes → **`dimension`** (point-to-point, exact value + drawing-visible)
2. Describing **positional relationship between two entities**?
   - 2a. Has definite numeric value (equality/inequality)? → **`position` (`constraint`)**
   - 2b. Needs precise structured description based on datum type? → **`position` (`relation`)**
   - 2c. Plain text only? → **`position` (`text`)**
3. Need to define a **region concept** (outline boundary, area value, containment)?
   - Yes → **`region_anno`**
4. Neither dimension-annotatable, nor entity spacing, nor region concept?
   - Yes → **`description`** field or top-level `descriptions`

**Conflict scenario examples**:

| Expression | Choice | Reason |
|-----------|--------|--------|
| "Parking width 2500mm" (two corner points) | `dimension` | Point-to-point + drawing annotation line |
| "Parking spacing must be ≥30mm" (nearest points of two parkings) | `position` (`constraint`) | Has numeric constraint, no specific annotation points |
| "E1 is 50mm to the right of P1" (point-based datum) | `position` (`relation`, `datum: point`) | Entity references and datum type |
| "E1 is at 3000~5000mm along L1 direction under CS1" | `position` (`relation`, `datum: csys`) | Coordinate system + direction reference + distance interval |
| "Parking spacing 30mm and must be ≥30mm" | **Both**: `dimension` marks point distance 30, `position` (`constraint`) records nearest point constraint | Point distance ≠ nearest point distance, two different things |
| "No obstacle within 50mm of parking" | `position` (`text`) | Plain text abstract description, no structuring needed |
| "Parking Zone A, area 25m², contains parking E1/E2/E3" | `region_anno` | Region outline + area + containment |
| "Floor material is concrete" | `description` | Non-geometric information |

### 10.2 Entity-level Description

`description` is a common base field for all entities (see [§5 Common Base Fields](#common-base-fields)), directly attached to the entity, for general semantic annotations unrelated to dimensions/positions:

```json
{
  "id": "E1",
  "type": "rectangle",
  "min": [0, 0], "max": [5000, 5000],
  "description": "This is the outer outline of a standard parking space"
}
```

`dimension` and `position` entity `description` fields hold annotation text/position notes:

```json
{
  "id": "D1",
  "type": "dimension",
  "p1": [0, 0], "p2": [5000, 0],
  "measurement": 5000,
  "description": "Parking width is 5000mm"
}
```

```json
{
  "id": "R1",
  "type": "position",
  "kind": "constraint",
  "ref_a": "E1", "ref_b": "E2",
  "value": 30, "operator": ">=",
  "description": "Shortest distance between parking 1 long edge and parking 2 long edge ≥ 30"
}
```

### 10.3 Property-level Description

Via top-level `descriptions` array, targeting a specific property of a specific entity. Used for semantic annotations **that cannot be expressed** via dimension/position:

```json
{
  "descriptions": [
    {
      "target": "E1",
      "property": "max",
      "text": "Parking width is 5000mm"
    },
    {
      "target": "E1",
      "property": null,
      "text": "This is a standard parking space"
    },
    {
      "target": "E1",
      "property": "material",
      "text": "Floor material is concrete"
    }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `target` | string | Yes | Entity/block/group id |
| `property` | string | No | Property name (e.g. `max`, `material`, `height`), null means entity-level description |
| `text` | string | Yes | Description text, natural language |

### 10.4 Block-level Description

Entities within blocks can also carry descriptions:

```json
{
  "id": "parking",
  "name": "Parking Space",
  "entities": [ ... ],
  "descriptions": [
    { "target": "parking_rect", "property": "min", "text": "Parking bottom-left corner" }
  ]
}
```

### 10.5 region_anno (Region Annotation)

**Region annotation** is both an entity type (see [§5.17](#517-region_anno-region-annotation)) and part of the description system. At the description level, it declares the semantic concept of a region (outline, area, label, and associated entities). It produces no new geometric lines, only references existing geometry boundaries to define the region.

```json
{
  "id": "RGA1",
  "type": "region_anno",
  "edges_refs": ["pc_region_outer", "pc_region_island_1"],
  "area": 25000000,
  "area_text": "25 m²",
  "contained_entities": ["E1", "E2", "E3"],
  "label": "Parking Zone A",
  "description": "Parking Zone A, area 25m², contains parking E1/E2/E3"
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `type` | string | Yes | — | Fixed as `"region_anno"` |
| `edges_refs` | string[] | Yes | — | Polycurve entity id array for closed boundary loops; first is outer boundary, subsequent are islands |
| `area` | number | No | Auto-calculated | Area value (square units) |
| `area_text` | string | No | Auto-formatted | Area text description |
| `contained_entities` | string[] | No | — | Entity id list contained within the region |
| `label` | string | No | — | Region label |
| `fill` | object | No | — | Fill pattern definition (see §5.17); when present, this region_anno also serves as a DXF HATCH equivalent |
| `description` | string | No | — | Natural language description |
| `operation` | object | No | — | Marks this region as derived from boolean operations; see §5.17 `operation` Object Structure |

**Semantic convention**:
- First item of `edges_refs` array is outer boundary, subsequent items are islands (internal holes)
- Referenced polycurves must form closed paths
- Each loop should be closed and non-intersecting
- Islands must be inside the outer boundary
- When region outline is entirely composed of existing geometry edges, region annotation itself produces no new lines, only declares the region concept

---

## 11. AI Reading Workflow Example

Requirement: "Open file → Read descriptions → Find main components → Read relevant properties"

```
1. Open file: Read top-level "summary" field
   → "This file describes parking space spacing requirements in parking layout specifications"

2. Iterate entities, find type="block_ref" with
   block_id corresponding block name containing "parking"
   → Locate block_ref entity

3. Read dimensions: Search type="dimension" with description
   containing "spacing"/"edge"
   → dimension.description: "Parking 1 long edge to parking 2 long edge distance ≥ 30"

4. Read position constraints: Search type="position" with
   ref_a / ref_b pointing to parking-related point entities
   → position.description: "Long edge spacing must be ≥ 30"

5. AI can further search type="region_anno" for region semantics (area, label, associated entities)

6. AI only needs to parse dimension / position / region_anno + description,
   no need to load all coordinates
```

**Core advantage**: AI first reads document `summary` for global intent, then searches by type for `dimension` / `position` / `region_anno` to obtain semantic information, without needing to parse all coordinates and property-level descriptions line by line.

---

## 12. Extension Mechanism

```json
{
  "type": "custom_entity",
  "entity_type": "dimension",
  "properties": {
    "def_point": [0, 0],
    "text_point": [50, -10],
    "measurement": 30.0
  },
  "description": "Parking spacing annotation"
}
```

| Field | Description |
|-------|-------------|
| `type` | Fixed as `"custom_entity"` |
| `entity_type` | Custom type name, degrades to polyline + text on DXF conversion |
| `properties` | Custom property key-value pairs |

---

## 13. Complete Example

> This example demonstrates GSGI core features: **geometry reference chain** (`represent`/`ref_op` + `param_pt`) and **semantic description layer** (`summary` + `description` + `position`).

```json
{
  "gsgi": "1.0",
  "summary": "Standard parking spaces in parking lot, with outline, label, and spacing constraint",
  "properties": {
    "unit": "mm",
    "coord_precision": 1
  },
  "blocks": [
    {
      "id": "standard_parking",
      "name": "Standard Parking Space",
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
      "description": "Left standard parking space (2500×5000mm)"
    },
    {
      "id": "E2",
      "type": "rectangle",
      "min": [2530, 0],
      "max": [5030, 5000],
      "description": "Right standard parking space (2500×5000mm), spacing 30mm"
    },
    {
      "id": "PP1",
      "type": "param_pt",
      "curve_ref": "E1",
      "t": 0.5,
      "point": [1250, 2500],
      "represent": "self",
      "label": "E1 geometric center"
    },
    {
      "id": "PP2",
      "type": "param_pt",
      "curve_ref": "E2",
      "t": 0.5,
      "point": [3780, 2500],
      "represent": "self",
      "label": "E2 geometric center"
    },
    {
      "id": "P1",
      "type": "point",
      "point": [0, 0],
      "ref_pt": "PP1",
      "represent": "self",
      "description": "Points to parking 1 center, for annotation positioning"
    },
    {
      "id": "P2",
      "type": "point",
      "point": [30, 0],
      "ref_pt": "PP2",
      "represent": "self",
      "description": "Points to parking 2 center, offset dx=30 from P1"
    },
    {
      "id": "D1",
      "type": "dimension",
      "p1_ref": "P1",
      "p2_ref": "P2",
      "measurement": 30,
      "description": "Horizontal center-to-center spacing between two parkings is 30mm"
    },
    {
      "id": "R1",
      "type": "position",
      "kind": "constraint",
      "ref_a": "E1",
      "ref_b": "E2",
      "value": 30,
      "operator": ">=",
      "description": "Nearest point distance between parkings must be ≥ 30mm"
    }
  ],
  "descriptions": [
    { "target": "E1", "property": null, "text": "Left standard parking space" },
    { "target": "E2", "property": null, "text": "Right standard parking space, 30mm spacing from E1" }
  ]
}
```
