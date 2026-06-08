# Rule 03: Attribute & Class Enhancements — UI Elements

## Overview

Individual visual components now strictly bind to a variant identifier and specific class name combinations.

---

## Buttons (`<wm-button>`, `<wm-form-action>`)

- **NDS Pattern:** Utilized standard Bootstrap contextual classes (e.g., `class="btn-default"` or `class="btn-primary"`).
- **DS Pattern:** Requires explicit mapping to `"filled"` or `"outline"` variants.

### Action

1. Add `btn-filled` to the class attribute.
2. Create a `variant` attribute mapped to the class context:
   - For `btn-default` → add `variant="filled:default"`
   - For `btn-primary` → add `variant="filled:primary"`

---

## Typography (`<wm-label>`)

- **NDS Pattern:** `class="p"` or `class="h5"`, along with `type="p"`.
- **DS Pattern:** Requires a `variant` attribute bound to the typographical class.

### Action

Read the class size (`p`, `h5`, etc.) and inject the matching variant:
- `class="p"` → `variant="default:p"`
- `class="h5"` → `variant="default:h5"`

---

## Icons (`<wm-icon>`)

- **NDS Pattern:** Basic icons `<wm-icon name="icon1"></wm-icon>`.
- **DS Pattern:** Requires size definition via both classes and variants.

### Action

Add specific Font Awesome size classes and map the variant to match:
- Add `class="fa-xs"`
- Add `variant="default:xs"`

---

## Images/Pictures (`<wm-picture>`)

- **NDS Pattern:** Handled shapes natively via `shape="circle"`.
- **DS Pattern:** Replaces the `shape` property with CSS classes and sets a crop mode.

### Action

1. Remove `shape="circle"`.
2. Inject `resizemode="cover"`.
3. Add `class="img-circle img-rounded"`.
4. Add `variant="default:rounded"`.

---

## Script

Extracted by the skill assembler into `wm_comp_conv_tmp.py`. Edit this block to add or change
basic-component conversion behaviour — the skill picks it up automatically on the next run.

Signature contract: `apply_basic_rules(text) -> (text, counts_dict)`.
Use `parse_attrs`, `build_attrs`, and `merge_class` — they are injected by the assembler header.
Module-level constants defined here (e.g. `BTN_VARIANT_MAP`) are included verbatim before the function.

```python
# Execution order: 2
# Components: wm-button, wm-form-action, wm-label, wm-icon, wm-picture

BTN_VARIANT_MAP = {
    'btn-default': 'filled:default',
    'btn-primary': 'filled:primary',
    'btn-success': 'filled:success',
    'btn-danger':  'filled:danger',
    'btn-warning': 'filled:warning',
    'btn-info':    'filled:info',
}
LABEL_SIZES = ['h1', 'h2', 'h3', 'h4', 'h5', 'h6', 'p', 'lead']

def apply_basic_rules(text):
    counts = {'button': 0, 'label': 0, 'icon': 0, 'picture': 0}

    def patch_button(m):
        tag_name, attr_str = m.group(1), m.group(2)
        attrs = parse_attrs(attr_str)
        cls = attrs.get('class', '')
        variant = next((v for k, v in BTN_VARIANT_MAP.items() if k in cls.split()), 'filled:default')
        attrs['class'] = merge_class(cls, 'btn-filled')
        attrs['variant'] = variant
        counts['button'] += 1
        return f'<{tag_name} {build_attrs(attrs)}>'

    def patch_label(m):
        attrs = parse_attrs(m.group(1))
        cls = attrs.get('class', '')
        size = next((s for s in LABEL_SIZES if s in cls.split()), None)
        if size:
            attrs['variant'] = f'default:{size}'
            counts['label'] += 1
        return f'<wm-label {build_attrs(attrs)}>'

    def patch_icon(m):
        attrs = parse_attrs(m.group(1))
        attrs['class'] = merge_class(attrs.get('class', ''), 'fa-xs')
        attrs['variant'] = 'default:xs'
        counts['icon'] += 1
        return f'<wm-icon {build_attrs(attrs)}>'

    def patch_picture(m):
        attrs = parse_attrs(m.group(1))
        attrs.pop('shape', None)
        attrs['resizemode'] = 'cover'
        attrs['class'] = merge_class(attrs.get('class', ''), 'img-rounded')
        attrs['variant'] = 'default:rounded'
        counts['picture'] += 1
        return f'<wm-picture {build_attrs(attrs)}>'

    text = re.sub(r'<(wm-button|wm-form-action)\b([^>]*)>', patch_button, text)
    text = re.sub(r'<wm-label\b([^>]*)>', patch_label, text)
    text = re.sub(r'<wm-icon\b([^>]*)>', patch_icon, text)
    text = re.sub(r'<wm-picture\b([^>]*?)>', patch_picture, text)
    return text, counts
```
