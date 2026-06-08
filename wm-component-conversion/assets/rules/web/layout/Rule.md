# Rule 06: Layout Component Migrations

## Overview

Left and right navigation panels gain two new attributes in the design system that define the navigation style and height. The remaining layout components (Header, Footer, TopNav) retain identical markup.

---

## LeftNav (`<wm-left-panel>`)

- **NDS Pattern:** `<wm-left-panel columnwidth="2" name="leftpanel" content="leftnav"></wm-left-panel>`
- **DS Pattern:** `<wm-left-panel columnwidth="2" name="leftpanel" content="leftnav" navtype="rail" navheight="full"></wm-left-panel>`

### Action

1. Add `navtype="rail"`.
2. Add `navheight="full"`.

---

## RightNav (`<wm-right-panel>`)

- **NDS Pattern:** `<wm-right-panel columnwidth="2" content="rightnav" name="right_panel"></wm-right-panel>`
- **DS Pattern:** `<wm-right-panel columnwidth="2" content="rightnav" navtype="rail" navheight="full" name="right_panel"></wm-right-panel>`

### Action

1. Add `navtype="rail"`.
2. Add `navheight="full"`.

---

## Unchanged Layout Components

| Component | Tag | Action |
|---|---|---|
| Header | `<wm-header>` | None |
| Footer | `<wm-footer>` | None |
| TopNav | `<wm-top-nav>` | None |

---

## Script

Extracted by the skill assembler into `wm_comp_conv_tmp.py`. Edit this block to add or change
layout-component conversion behaviour — the skill picks it up automatically on the next run.

Signature contract: `apply_layout_rules(text) -> (text, counts_dict)`.
Use `parse_attrs`, `build_attrs`, and `merge_class` — they are injected by the assembler header.

```python
# Execution order: 6
# Components: wm-left-panel, wm-right-panel

def apply_layout_rules(text):
    counts = {'wm_left_panel': 0, 'wm_right_panel': 0}

    def patch_left_panel(m):
        attrs = parse_attrs(m.group(1))
        if 'navtype' not in attrs:
            attrs['navtype'] = 'rail'
            attrs['navheight'] = 'full'
            counts['wm_left_panel'] += 1
        return f'<wm-left-panel {build_attrs(attrs)}>'

    def patch_right_panel(m):
        attrs = parse_attrs(m.group(1))
        if 'navtype' not in attrs:
            attrs['navtype'] = 'rail'
            attrs['navheight'] = 'full'
            counts['wm_right_panel'] += 1
        return f'<wm-right-panel {build_attrs(attrs)}>'

    text = re.sub(r'<wm-left-panel\b([^>]*)>', patch_left_panel, text)
    text = re.sub(r'<wm-right-panel\b([^>]*)>', patch_right_panel, text)
    return text, counts
```
