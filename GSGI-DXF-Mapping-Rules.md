# GSGI — DXF Mapping Rules

This document defines the bidirectional mapping rules between GSGI 1.0 and DXF.

> Version: v1.0
> Corresponding GSGI entity types: 25 + 4 descriptive mechanisms

---

## 1. Global Mapping Conventions

| Item | Rule |
|----|------|
| Coordinates | All `*_ref` fields in GSGI reference point entities; when mapping, the point chain must first be resolved to WCS coordinates before writing to DXF |
| Color | ACI index maps directly; Hex RGB converts to DXF true color group code (`420`); ByLayer/ByBlock map to `256`/`0` |
| Linetype | String linetype name writes directly to DXF `6` group code |
| Lineweight | Integer maps to DXF `370` group code (-3=ByLayer, -2=ByBlock, -1=Default, 0-211=Specific) |
| Transforms | GSGI `transform` matrix is applied to coordinates before writing to DXF; on DXF read, coordinates are extracted and transform is identity |
| Descriptions | Stored uniformly in XDATA `GSGI_DESC` (see §3) |
| point resolution | Recursively resolve point.ref_pt chain to final coordinate: param_pt → curve interpolation, coord_sys → local-to-WCS transform | Entity `represent` determines representation point location, `ref_op` determines offset interpretation (see GSGI Design Doc §9) |
| Angle units | Except text/mtext, GSGI angle fields use radians; DXF uses degrees | GSGI→DXF: rad→deg; DXF→GSGI: deg→rad |

---

## 2. GSGI → DXF

### 2.1 Entity Mapping Table

| GSGI Type | DXF Entity | Key Mapping | Notes |
|-----------|----------|-------------|-------|
| point | POINT | `10,20`=coordinate; `ref_pt` chain stored in XDATA `GSGI_PTREF` | No ref_pt writes only POINT |
| param_pt | Not directly output | When referenced by point.ref_pt, expanded by point | Not independently output to DXF |
| line | LINE | `10,20`=start; `11,21`=end | start_ref/end_ref resolved to coordinates |
| polyline | LWPOLYLINE | `90`=vertex count; `70`=closed flag; per-vertex `10,20` | Straight segments only |
| polyarc | LWPOLYLINE | `90`=vertex count; `42`=per-vertex bulge | bulge maps directly |
| polycurve | LWPOLYLINE/SPLINE | Split by segment type: `line`→LINE, `arc`→ARC, `subsegment_ref`→LWPOLYLINE, `curve_ref`→SPLINE/LINE | Or degrade to sampled LWPOLYLINE |
| mline | MLINE | Multi-line vertices expanded to point coordinates; element attributes mapped | TODO |
| circle | CIRCLE | `10,20`=center; `40`=radius | center_ref resolved |
| arc | ARC | `10,20`=center; `40`=radius; `50`=start angle; `51`=end angle | Angles rad→deg |
| rectangle | LWPOLYLINE | 4 vertices, `70`=1 (closed) | min_ref/max_ref expanded |
| text | TEXT | `10,20`=insertion point; `1`=content; `40`=text height; `50`=rotation | position_ref resolved |
| mtext | MTEXT | `10,20`=insertion point; `1`=content; `40`=text height; `41`=width; `50`=rotation; `71`=attachment | position_ref resolved |
| spline_fit | SPLINE | `210`=normal; `70`=flags; `74`=fit point count; per-point `11,21` | fit_point_refs resolved |
| spline_cv | SPLINE | `210`=normal; `70`=flags; `72`=control point count; `73`=degree; `40`=knots; `10,20`=control points | control_point_refs resolved |
| block_ref | INSERT | `2`=block name; `10,20`=insertion point; `50`=rotation; `41,42`=scale | position_ref resolved; rotation rad→deg |
| xref | XREF | `2`=file name; `10,20`=insertion point; `50`=rotation; `41,42`=scale; file path in XDATA `GSGI_XREF` | position_ref resolved; rotation rad→deg |
| table | ACAD_TABLE | Extract markdown to table structure | Degrade: LINE+TEXT |
| hatch | HATCH | `91`=boundary count; `52`=pattern angle; boundaries expanded from `outer_ref`/`island_refs` polycurves | polycurve must be sampled to closed loop; pattern angle rad→deg |
| subsegment | LWPOLYLINE | Sampled along parent curve as polyline | Sampling density implementation-defined |
| dimension | DIMENSION | `13,23`=definition point; `14,24`=text point | p1_ref/p2_ref resolved |
| region_anno | (descriptive system, no independent DXF entity) | Boundary from edges_refs polycurves maps to closed LWPOLYLINE/HATCH; area/label/contained entities in XDATA `GSGI_REGION` | region_anno is both entity and descriptive system in GSGI |
| position | No entity | All stored in XDATA `GSGI_POS` | ref_a/ref_b stored as id |
| measure | No entity | All stored in XDATA (same as position) | Invisible entity |
| coord_sys | UCS + XDATA | Origin `10,20`=origin_ref coordinates; rotation (rad) in XDATA `GSGI_CSYS` | origin_ref resolved |
| custom_entity | Custom | Degradation determined by `entity_type` | Default degrade to LWPOLYLINE+TEXT |

### 2.2 Mapping Examples

#### line (point reference)

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

#### polyarc (bulge)

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

#### hatch (polycurve reference)

```dxf
# GSGI:
# { "id": "H1", "type": "hatch",
#   "pattern": "ANSI31",
#   "boundaries": [{ "outer_ref": "pc_out", "island_refs": ["pc_isl"] }] }
# { "id": "pc_out", "type": "polycurve", "segments": [...] }
#
# DXF output (simplified):
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

### 3.1 Entity Mapping Table

| DXF Entity | GSGI Type | Mapping Logic |
|-----------|-----------|---------------|
| POINT | point or param_pt | Has XDATA `GSGI_PARAM` (curve_ref/t) → param_pt; has XDATA `GSGI_PTREF` (ref_pt) → point; otherwise → point |
| LINE | line | Generate two points (end/start), line references them |
| LWPOLYLINE (no bulge) | polyline | 4 vertices closed and axis-aligned → rectangle |
| LWPOLYLINE (with bulge) | polyarc | Generate point array + bulge array |
| POLYLINE | polyline | Same as LWPOLYLINE |
| MLINE | mline | Generate point array + elements |
| CIRCLE | circle | Generate center point |
| ARC | arc | Generate center point |
| TEXT | text | Generate position point |
| MTEXT | mtext | Generate position point |
| SPLINE (fit points) | spline_fit | Generate fit point array |
| SPLINE (control points) | spline_cv | Generate control point array |
| INSERT | block_ref | Generate position point; block definition stored in blocks |
| XREF | xref | Generate position point; file path restored from XDATA `GSGI_XREF` |
| ACAD_TABLE | table | Generate position point; extract grid to markdown |
| HATCH (hatched) | hatch | Boundary loops generate polycurve; outer_ref + island_refs |
| HATCH (unhatched) + XDATA `GSGI_REGION` | region_anno | Boundary loops generate polycurve; area/label restored from XDATA |
| DIMENSION | dimension | Generate two points (p1_ref/p2_ref) |
| LEADER | polyline + text | Degrade to polyline and text |
| UCS + XDATA | coord_sys | Generate origin point |
| —（XDATA） | position | Restore from XDATA `GSGI_POS` |
| —（XDATA） | measure | Restore from XDATA (same as position) |

### 3.2 Mapping Example

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

## 4. Descriptive Information Mapping

GSGI `descriptions` do not directly correspond to DXF standard fields; they use XDATA uniformly.

### Recommended: XDATA

| XDATA App Name | Stored Content | Applicable Entities |
|---------------|----------------|---------------------|
| `GSGI_ID` | GSGI entity id (string) | All DXF entities |
| `GSGI_DESC` | Property-level description JSON `{"target":"","property":"","text":""}` | descriptions array |
| `GSGI_PTREF` | point.ref_pt reference chain | POINT |
| `GSGI_PARAM` | param_pt parameters `{"curve_ref":"","t":0}` | POINT |
| `GSGI_POS` | position entity full JSON | Non-graphical entities |
| `GSGI_XREF` | xref file path `{"file_path":"..."}` | XREF |
| `GSGI_REGION` | region_anno full JSON `{"area":0,"label":"","contained_entities":[],"operation":{}}` | Boundary mapped from polycurve entities |
| `GSGI_CSYS` | coord_sys attributes `{"rotation":0}` | UCS |

### XDATA Format Example

```dxf
# Entity ID marker
 1001
GSGI_ID
 1000
E1

# Property-level description
 1001
GSGI_DESC
 1000
{"target":"E1","property":"max","text":"Parking space width is 5000mm"}

# param_pt parameters
 1001
GSGI_PARAM
 1000
{"curve_ref":"E3","t":0.25}

# region_anno boolean operation
 1001
GSGI_REGION
 1000
{"area":250.5,"label":"Shadow-building overlap","contained_entities":["building_A"],"operation":{"op":"intersect","refs":["RGA_shadow_fan","RGA_building"],"desc":"Intersection of shadow fan region and building footprint"}}
```

### Bypass Method

For environments that do not support XDATA, a companion `.gsgi.desc.json` file can store descriptive data.

```json
// Companion file: drawing.dxf.gsgi.desc.json
{
  "descriptions": [
    { "target": "E1", "property": "max", "text": "Parking space width is 5000mm" }
  ]
}
```

---

## 5. Notes

1. **point reference resolution**: GSGI→DXF direction requires recursive resolution of point.ref_pt chain to obtain final WCS coordinates; DXF→GSGI direction creates a point entity for each key coordinate
2. **polycurve sampling**: HATCH/REGION_ANNO boundaries reference polycurve; DXF output requires sampling polycurve to closed LWPOLYLINE vertex sequence
3. **Invisible entities**: position has no graphical counterpart in DXF; all serialized to XDATA; auto-rebuilt on DXF load
4. **Block definitions**: entities within block follow the same mapping rules as top level; nested entities (block_ref/xref) must not appear within block definitions
5. **Custom entities**: custom_entity defaults to LWPOLYLINE + TEXT (properties to XDATA); custom mapping via extension plugins

---

## 6. Format Comparison

| Feature | DXF | GSGI |
|---------|-----|------|
| Size | Very large (includes metadata) | Minimal (geometry + descriptions only) |
| Readability | Poor (stacked group codes) | Good (structured JSON) |
| AI-friendliness | Low (noise parsing required) | High (description guidance + on-demand search) |
| Description system | No native support | First-class citizen |
| Geometry types | 30+ | 25 entities + 4 descriptive mechanisms (extensible on-demand) |
| Toolchain | Mature | Requires custom converter |
