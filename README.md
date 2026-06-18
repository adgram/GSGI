# GSGI — 通用简易几何信息模型

**General Simple Geometry Information**

GSGI 是一种面向 AI 的轻量级 CAD 数据交换格式，以几何实体 + 描述系统为核心，专为 AI 快速识图与 DXF 双向转换而设计。本仓库包含 GSGI 格式的完整定义和 JSON Schema。

GSGI is an AI-oriented lightweight CAD data exchange format centered on geometric entities + a description system, designed for fast AI diagram reading and bidirectional DXF conversion. This repository contains the complete GSGI format specification and JSON Schema.

## 核心特性 / Core Features

- **AI 优先**：纯文本 JSON 格式、自描述，AI 可先读 `summary` 了解全局，再按类型搜索描述信息，无需解析全部坐标
  **AI First**: Plain-text JSON, self-describing — AI reads `summary` for context, then searches descriptions by type without parsing all coordinates
- **描述系统**：四种描述手段（`dimension` / `position` / `region_anno` / `description`）支持实体级、属性级、文档级自然语言注释
  **Description System**: Four descriptive mechanisms (`dimension` / `position` / `region_anno` / `description`) for entity-level, property-level, and document-level natural language annotations
- **取点与组合运算**（`represent` / `ref_op`）：声明式的几何引用链，point 通过 `ref_pt` 链式解析坐标，支持偏移、投影、局部坐标等组合方式
  **Point Derivation & Combination** (`represent` / `ref_op`): Declarative geometry reference chain — points resolve coordinates via `ref_pt` chains, supporting offset, project, local transform, and more
- **21 种实体类型 / 21 Entity Types**: point, line, polyline, polyarc, polycurve, circle, arc, rectangle, text, spline_fit, spline_cv, block_ref, xref, table, subsegment, dimension, region_anno, position, coord_sys, param_pt, custom_entity
- **最小化**：仅含有效几何与描述信息，无 CAD 系统变量、历史记录等元数据噪音
  **Minimal**: Contains only valid geometry and descriptions — no CAD system variables, history, or other metadata noise

## 格式概览 / Format Overview

一个 GSGI 文件是一个 JSON 对象，顶层结构如下：

A GSGI file is a JSON object with the following top-level structure:

```json
{
  "gsgi": "1.0",
  "tags": ["建筑", "停车场", "规范"],
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

详细格式定义见 [GSGI设计文档.md](./GSGI设计文档.md)（中文）、[GSGI-Design-Doc.md](./GSGI-Design-Doc.md)（英文），DXF 映射规则见 [GSGI-DXF映射规则.md](./GSGI-DXF映射规则.md)（中文）、[GSGI-DXF-Mapping-Rules.md](./GSGI-DXF-Mapping-Rules.md)（英文），JSON Schema 定义见 [gsgi.schema.json](./gsgi.schema.json)。

For detailed format specification, see [GSGI设计文档.md](./GSGI设计文档.md) (Chinese), [GSGI-Design-Doc.md](./GSGI-Design-Doc.md) (English); DXF mapping rules: [GSGI-DXF映射规则.md](./GSGI-DXF映射规则.md) (Chinese), [GSGI-DXF-Mapping-Rules.md](./GSGI-DXF-Mapping-Rules.md) (English); JSON Schema: [gsgi.schema.json](./gsgi.schema.json).

## 许可 / License

本项目基于 [MIT 许可证](./LICENSE)。

This project is licensed under the [MIT License](./LICENSE).
