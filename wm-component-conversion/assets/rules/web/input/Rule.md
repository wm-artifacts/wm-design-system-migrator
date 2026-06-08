# Rule 05: Input Component Migrations

## Overview

Most input components retain identical markup between non–design system and design system projects. Only the `<wm-chips>` component requires additional class and variant attributes.

---

## Chips (`<wm-chips>`)

- **NDS Pattern:** `<wm-chips name="chips1"></wm-chips>`
- **DS Pattern:** `<wm-chips name="chips1" class="app-chips filled default" variant="filled:default"></wm-chips>`

### Action

1. Add `class="app-chips filled default"` (append to any existing classes).
2. Add `variant="filled:default"`.

---

## Unchanged Input Components

The following input components have identical markup in both NDS and DS projects — no transformation required:

| Component | Tag |
|---|---|
| Calendar | `<wm-calendar>` |
| Checkbox | `<wm-checkbox>` |
| Checkboxset | `<wm-checkboxset>` |
| Color Picker | `<wm-colorpicker>` |
| Composite | `<wm-composite>` |
| Currency | `<wm-currency>` |
| Date | `<wm-date>` |
| Date Time | `<wm-datetime>` |
| File Upload | `<wm-fileupload>` |
| Number | `<wm-number>` |
| RadioSet | `<wm-radioset>` |
| Rating | `<wm-rating>` |
| Select | `<wm-select>` |
| Slider | `<wm-slider>` |
| Switch | `<wm-switch>` |
| Text | `<wm-text>` |
| TextArea | `<wm-textarea>` |
| Time | `<wm-time>` |

---

## Script

Extracted by the skill assembler into `wm_comp_conv_tmp.py`. Edit this block to add or change
input-component conversion behaviour — the skill picks it up automatically on the next run.

Signature contract: `apply_input_rules(text) -> (text, counts_dict)`.
Use `parse_attrs`, `build_attrs`, and `merge_class` — they are injected by the assembler header.

```python
# Execution order: 5
# Components: wm-chips

def apply_input_rules(text):
    counts = {'wm_chips': 0}

    def patch_chips(m):
        attrs = parse_attrs(m.group(1))
        if 'variant' not in attrs:
            attrs['class'] = merge_class(attrs.get('class', ''), 'app-chips filled default')
            attrs['variant'] = 'filled:default'
            counts['wm_chips'] += 1
        return f'<wm-chips {build_attrs(attrs)}>'

    text = re.sub(r'<wm-chips\b([^>]*)>', patch_chips, text)
    return text, counts
```
