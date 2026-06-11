# Rule 08: Charts — No Migration Required

## Overview

All chart types in WaveMaker (Bar, Column, Bubble, Cumulative Line, Line, Area, Pie, Donut) have identical markup between non–design system and design system projects. No transformations are required for chart components.

---

## Unchanged Chart Components

| Chart Type | Component Attribute | Action |
|---|---|---|
| Bar | `type="Bar"` | None |
| Column | `type="Column"` | None |
| Bubble | `type="Bubble"` | None |
| Cumulative Line | `type="Cumulative Line"` | None |
| Line | `type="Line"` | None |
| Area | `type="Area"` | None |
| Pie | `type="Pie"` | None |
| Donut | `type="Donut"` | None |

All `<wm-chart>` components preserve their existing `type`, `title`, `height`, `iconclass`, and `name` attributes unchanged.

---

## Script

Extracted by the skill assembler into `wm_comp_conv_tmp.py`.

Signature contract: `apply_charts_rules(text) -> (text, counts_dict)`.
Use `parse_attrs`, `build_attrs`, and `merge_class` — they are injected by the assembler header.

```python
# Execution order: 8
# Components: wm-chart
# No transformations required for chart components.

def apply_charts_rules(text):
    counts = {}
    return text, counts
```
