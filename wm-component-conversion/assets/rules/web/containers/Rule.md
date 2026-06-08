# Rule 03: Container Component Migrations

## Overview

Several container-type components require class additions and variant assignments for design system compliance. This rule covers: Container, Accordion, Panel, Tile, Wizard, and Tabs.

---

## Container (`<wm-container>`)

- **NDS Pattern:** `<wm-container name="container1"></wm-container>`
- **DS Pattern:** `<wm-container direction="row" alignment="top-left" gap="4" width="fill" name="container1" class="app-container-default" variant="default"></wm-container>`

### Action

1. Add flex attributes (if not already present):
   - `direction="row"`
   - `alignment="top-left"`
   - `gap="4"`
   - `width="fill"`
2. Add `class="app-container-default"` (append to any existing classes).
3. Add `variant="default"`.

---

## Accordion (`<wm-accordion>`)

- **NDS Pattern:** `<wm-accordion type="static" statehandler="URL" name="accordion1"></wm-accordion>`
- **DS Pattern:** `<wm-accordion type="static" statehandler="URL" name="accordion1" class="app-accordion panel panel-default" variant="default:default"></wm-accordion>`

### Action

1. Add `class="app-accordion panel panel-default"` (append to any existing classes).
2. Add `variant="default:default"`.

> **Note:** Child `<wm-accordionpane>` elements are not affected.

---

## Panel (`<wm-panel>`)

- **NDS Pattern:** `<wm-panel subheading="..." iconclass="..." title="Title" name="panel1"></wm-panel>`
- **DS Pattern:** `<wm-panel ... name="panel1" class="panel panel-default" variant="default:default"></wm-panel>`

### Action

1. Add `class="panel panel-default"` (append to any existing classes).
2. Add `variant="default:default"`.

> **Note:** Inner `<wm-label>` elements within panel footers receive their variant via the Basic rules (execution order 2). `<wm-panel-footer>` is not patched by this rule.

---

## Tile (`<wm-tile>`)

- **NDS Pattern:** `<wm-tile name="tile1"></wm-tile>`
- **DS Pattern:** `<wm-tile class="bg-default" name="tile1" variant="default:default"></wm-tile>`

### Action

1. Add `class="bg-default"` (append to any existing classes).
2. Add `variant="default:default"`.

---

## Wizard (`<wm-wizard>`)

- **NDS Pattern:** `<wm-wizard type="static" stepstyle="justified" class="number" name="wizard1"></wm-wizard>`
- **DS Pattern:** `<wm-wizard type="static" stepstyle="justified" class="number wizard-number" name="wizard1" variant="number"></wm-wizard>`

### Action

1. If the class attribute contains `"number"`, append `"wizard-number"` to the class.
2. Add `variant="number"`.

> **Note:** `<wm-button>` elements inside wizard actions are upgraded by the Basic rules. `<wm-container>` elements inside wizard actions are upgraded by this rule.

---

## Tabs (`<wm-tabs>`)

- **NDS Pattern:** `<wm-tabs type="static" statehandler="URL" name="tabs1"></wm-tabs>`
- **DS Pattern:** Identical outer `<wm-tabs>` tag — no structural change.
- **Action:** None required for the outer `<wm-tabs>` wrapper.

---

## Script

Extracted by the skill assembler into `wm_comp_conv_tmp.py`. Edit this block to add or change
container-component conversion behaviour — the skill picks it up automatically on the next run.

Signature contract: `apply_containers_rules(text) -> (text, counts_dict)`.
Use `parse_attrs`, `build_attrs`, and `merge_class` — they are injected by the assembler header.

```python
# Execution order: 3
# Components: wm-container, wm-accordion, wm-panel, wm-tile, wm-wizard

def apply_containers_rules(text):
    counts = {
        'wm_container': 0,
        'wm_accordion': 0,
        'wm_panel': 0,
        'wm_tile': 0,
        'wm_wizard': 0,
    }

    def patch_container(m):
        attrs = parse_attrs(m.group(1))
        if 'variant' not in attrs:
            attrs.setdefault('direction', 'row')
            attrs.setdefault('alignment', 'top-left')
            attrs.setdefault('gap', '4')
            attrs.setdefault('width', 'fill')
            attrs['class'] = merge_class(attrs.get('class', ''), 'app-container-default')
            attrs['variant'] = 'default'
            counts['wm_container'] += 1
        return f'<wm-container {build_attrs(attrs)}>'

    def patch_accordion(m):
        attrs = parse_attrs(m.group(1))
        if 'variant' not in attrs:
            attrs['class'] = merge_class(attrs.get('class', ''), 'app-accordion panel panel-default')
            attrs['variant'] = 'default:default'
            counts['wm_accordion'] += 1
        return f'<wm-accordion {build_attrs(attrs)}>'

    def patch_panel(m):
        attrs = parse_attrs(m.group(1))
        if 'variant' not in attrs:
            attrs['class'] = merge_class(attrs.get('class', ''), 'panel panel-default')
            attrs['variant'] = 'default:default'
            counts['wm_panel'] += 1
        return f'<wm-panel {build_attrs(attrs)}>'

    def patch_tile(m):
        attrs = parse_attrs(m.group(1))
        if 'variant' not in attrs:
            attrs['class'] = merge_class(attrs.get('class', ''), 'bg-default')
            attrs['variant'] = 'default:default'
            counts['wm_tile'] += 1
        return f'<wm-tile {build_attrs(attrs)}>'

    def patch_wizard(m):
        attrs = parse_attrs(m.group(1))
        if 'variant' not in attrs:
            cls = attrs.get('class', '')
            if 'number' in cls.split():
                attrs['class'] = merge_class(cls, 'wizard-number')
                attrs['variant'] = 'number'
                counts['wm_wizard'] += 1
        return f'<wm-wizard {build_attrs(attrs)}>'

    text = re.sub(r'<wm-container\b([^>]*)>', patch_container, text)
    text = re.sub(r'<wm-accordion\b([^>]*)>', patch_accordion, text)
    # Use (?!-) to avoid matching wm-panel-footer
    text = re.sub(r'<wm-panel\b(?!-)([^>]*)>', patch_panel, text)
    text = re.sub(r'<wm-tile\b([^>]*)>', patch_tile, text)
    text = re.sub(r'<wm-wizard\b([^>]*)>', patch_wizard, text)
    return text, counts
```
