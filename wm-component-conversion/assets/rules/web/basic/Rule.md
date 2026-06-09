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

- **NDS Pattern:** Handled shapes natively via `shape="rounded|circle|thumbnail"`.
- **DS Pattern:** Replaces `shape` with a CSS class and variant, and sets crop mode.

### Action

1. Read `shape` and map to class + variant (remove `shape` after):
   - `rounded` → `class="img-rounded"`, `variant="default:rounded"`
   - `circle` → `class="img-circle"`, `variant="default:circle"`
   - `thumbnail` → `class="img-thumbnail"`, `variant="default:standard"`
   - _(absent)_ → default to `img-rounded` / `default:rounded`
2. Inject `resizemode="cover"` (if not already set).

---

## Message (`<wm-message>`)

- **NDS Pattern:** `<wm-message name="message1"></wm-message>`
- **DS Pattern:** `<wm-message name="message1" type="success" class="app-message alert-success" variant="filled:success"></wm-message>`

### Action

1. If `type` is not set, default to `"success"`.
2. Add `class="app-message alert-{type}"` based on the resolved type value.
3. Add `variant="filled:{type}"`.

---

## Progress Bar (`<wm-progress-bar>`)

- **NDS Pattern:** `<wm-progress-bar datavalue="30" name="progress_bar1"></wm-progress-bar>`
- **DS Pattern:** `<wm-progress-bar datavalue="30" name="progress_bar1" class="app-progress progress-bar-default" variant="filled:default"></wm-progress-bar>`

### Action

1. Add `class="app-progress progress-bar-default"` (append to any existing classes).
2. Add `variant="filled:default"`.

---

## Progress Circle (`<wm-progress-circle>`)

- **NDS Pattern:** `<wm-progress-circle width="150px" height="150px" name="progress_circle1"></wm-progress-circle>`
- **DS Pattern:** `<wm-progress-circle width="150px" height="150px" name="progress_circle1" class="app-progress circle progress-circle-default" variant="filled:default"></wm-progress-circle>`

### Action

1. Add `class="app-progress circle progress-circle-default"` (append to any existing classes).
2. Add `variant="filled:default"`.

---

## Button Group (`<wm-buttongroup>`)

- **NDS Pattern:** `<wm-buttongroup name="buttongroup1"></wm-buttongroup>` with inner `<wm-button class="btn-default" ...></wm-button>` children.
- **DS Pattern:** The `<wm-buttongroup>` wrapper is unchanged. Each inner `<wm-button>` gains `btn-filled` and `variant` via the Buttons rule above.
- **Action:** None required on the `<wm-buttongroup>` tag itself — inner buttons are patched automatically.

---

## Anchor (`<wm-anchor>`)

- **NDS Pattern:** `<wm-anchor margin="unset 0.5em" name="anchor1"></wm-anchor>`
- **DS Pattern:** Identical — no structural change required.
- **Action:** None.

---

## Audio (`<wm-audio>`)

- **NDS Pattern:** `<wm-audio controls="controls" audiopreload="none" name="audio1"></wm-audio>`
- **DS Pattern:** Identical — no structural change required.
- **Action:** None.

---

## HTML (`<wm-html>`)

- **NDS Pattern:** `<wm-html name="html1"></wm-html>`
- **DS Pattern:** Identical — no structural change required.
- **Action:** None.

---

## Iframe (`<wm-iframe>`)

- **NDS Pattern:** `<wm-iframe name="iframe1"></wm-iframe>`
- **DS Pattern:** Identical — no structural change required.
- **Action:** None.

---

## Rich Text Editor (`<wm-richtexteditor>`)

- **NDS Pattern:** `<wm-richtexteditor name="richtexteditor1"></wm-richtexteditor>`
- **DS Pattern:** Identical — no structural change required.
- **Action:** None.

---

## Search (`<wm-search>`)

- **NDS Pattern:** `<wm-search name="search1"></wm-search>`
- **DS Pattern:** Identical — no structural change required.
- **Action:** None.

---

## Spinner (`<wm-spinner>`)

- **NDS Pattern:** `<wm-spinner show="true" name="spinner1"></wm-spinner>`
- **DS Pattern:** Identical — no structural change required.
- **Action:** None.

---

## Tree (`<wm-tree>`)

- **NDS Pattern:** `<wm-tree name="tree1"></wm-tree>`
- **DS Pattern:** Identical — no structural change required.
- **Action:** None.

---

## Video (`<wm-video>`)

- **NDS Pattern:** `<wm-video controls="controls" videopreload="none" name="video1"></wm-video>`
- **DS Pattern:** Identical — no structural change required.
- **Action:** None.

---

## Script

Extracted by the skill assembler into `wm_comp_conv_tmp.py`. Edit this block to add or change
basic-component conversion behaviour — the skill picks it up automatically on the next run.

Signature contract: `apply_basic_rules(text) -> (text, counts_dict)`.
Use `parse_attrs`, `build_attrs`, and `merge_class` — they are injected by the assembler header.
Module-level constants defined here (e.g. `BTN_VARIANT_MAP`) are included verbatim before the function.

```python
# Execution order: 2
# Components: wm-button, wm-form-action, wm-label, wm-icon, wm-picture,
#             wm-message, wm-progress-bar, wm-progress-circle

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
    counts = {
        'button': 0,
        'label': 0,
        'icon': 0,
        'picture': 0,
        'wm_message': 0,
        'wm_progress_bar': 0,
        'wm_progress_circle': 0,
    }

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
        else:
            attrs['variant'] = 'default:default'
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
        shape = attrs.pop('shape', None)
        shape_map = {
            'rounded':   ('img-rounded',    'default:rounded'),
            'circle':    ('img-circle',     'default:circle'),
            'thumbnail': ('img-standard',  'default:standard'),
        }
        cls_suffix, variant = shape_map.get(shape, ('img-rounded', 'default:rounded'))
        attrs.setdefault('resizemode', 'cover')
        attrs['class'] = merge_class(attrs.get('class', ''), cls_suffix)
        attrs['variant'] = variant
        counts['picture'] += 1
        return f'<wm-picture {build_attrs(attrs)}>'

    def patch_message(m):
        attrs = parse_attrs(m.group(1))
        if 'variant' not in attrs:
            msg_type = attrs.get('type', 'success')
            attrs.setdefault('type', msg_type)
            attrs['class'] = merge_class(attrs.get('class', ''), f'app-message alert-{msg_type}')
            attrs['variant'] = f'filled:{msg_type}'
            counts['wm_message'] += 1
        return f'<wm-message {build_attrs(attrs)}>'

    def patch_progress_bar(m):
        attrs = parse_attrs(m.group(1))
        if 'variant' not in attrs:
            attrs['class'] = merge_class(attrs.get('class', ''), 'app-progress progress-bar-default')
            attrs['variant'] = 'filled:default'
            counts['wm_progress_bar'] += 1
        return f'<wm-progress-bar {build_attrs(attrs)}>'

    def patch_progress_circle(m):
        attrs = parse_attrs(m.group(1))
        if 'variant' not in attrs:
            attrs['class'] = merge_class(attrs.get('class', ''), 'app-progress circle progress-circle-default')
            attrs['variant'] = 'filled:default'
            counts['wm_progress_circle'] += 1
        return f'<wm-progress-circle {build_attrs(attrs)}>'

    text = re.sub(r'<(wm-button|wm-form-action)\b((?:[^>"\']|"[^"]*"|\'[^\']*\')*)>', patch_button, text)
    text = re.sub(r'<wm-label\b((?:[^>"\']|"[^"]*"|\'[^\']*\')*)>', patch_label, text)
    text = re.sub(r'<wm-icon\b((?:[^>"\']|"[^"]*"|\'[^\']*\')*)>', patch_icon, text)
    text = re.sub(r'<wm-picture\b((?:[^>"\']|"[^"]*"|\'[^\']*\')*)>', patch_picture, text)
    text = re.sub(r'<wm-message\b((?:[^>"\']|"[^"]*"|\'[^\']*\')*)>', patch_message, text)
    text = re.sub(r'<wm-progress-bar\b((?:[^>"\']|"[^"]*"|\'[^\']*\')*)>', patch_progress_bar, text)
    text = re.sub(r'<wm-progress-circle\b((?:[^>"\']|"[^"]*"|\'[^\']*\')*)>', patch_progress_circle, text)
    return text, counts
```
