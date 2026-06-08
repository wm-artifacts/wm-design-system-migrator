# Rule 09: Navigation Components — No Migration Required

## Overview

All navigation components retain identical markup between non–design system and design system projects. No transformations are required.

---

## Unchanged Navigation Components

| Component | Tag | Action |
|---|---|---|
| Breadcrumb | `<wm-breadcrumb>` | None |
| Menu | `<wm-menu>` | None |
| Nav | `<wm-nav>` / `<wm-nav-item>` | None |
| Popover | `<wm-popover>` | None |
| Navbar | `<wm-navbar>` | None |

---

## Script

Extracted by the skill assembler into `wm_comp_conv_tmp.py`.

Signature contract: `apply_navigation_rules(text) -> (text, counts_dict)`.
Use `parse_attrs`, `build_attrs`, and `merge_class` — they are injected by the assembler header.

```python
# Execution order: 9
# Components: wm-breadcrumb, wm-menu, wm-nav, wm-popover, wm-navbar
# No transformations required for navigation components.

def apply_navigation_rules(text):
    counts = {}
    return text, counts
```
