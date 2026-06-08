# Rule 01: Structural Replacements — Forms & Live Filters

## Overview

The most significant DOM structural change involves removing legacy grid systems in favor of responsive flex containers.

---

## Form & LiveFilter Wrappers

- **Pattern Change:** The parent `<wm-form>` and `<wm-livefilter>` tags must now explicitly define responsive row constraints based on their original grid column count.
- **Action:** If the original layout had `columns="3"`, inject `itemsperrow="xs-3 sm-3 md-3 lg-3"` directly into the `<wm-form>` or `<wm-livefilter>` tag.

---

## Removing `<wm-layoutgrid>`

- **NDS Pattern:** Used `<wm-layoutgrid>`, `<wm-gridrow>`, and `<wm-gridcolumn>` to structure fields.
- **DS Pattern:** Flattens this hierarchy into a single Flexbox `<wm-container>`.

### Action

1. Delete `<wm-layoutgrid>`, `<wm-gridrow>`, and `<wm-gridcolumn>` wrappers.
2. Wrap the child input fields (`wm-form-field`, `wm-filter-field`) in a newly structured container:

```html
<!-- Replace Grid Hierarchy With: -->
<wm-container direction="row" alignment="top-left" wrap="true" width="fill" columns="{original_column_count}" name="containerX" class="app-container-default" variant="default">
    <!-- Fields go here -->
</wm-container>
```

---

## `<wm-list>` and `<wm-listtemplate>`

- **NDS Pattern:** Relied on `listclass="list-group"` and internal components having `class="media-left"`, `class="media-body"`.
- **DS Pattern:** Uses explicit flex properties.

### Action on `<wm-list>`

1. Remove `listclass="list-group"`.
2. Inject Flex attributes:
   - `direction="column"`
   - `alignment="top-left"`
   - `gap="4"`
   - `wrap="false"`

### Action on `<wm-listtemplate>`

1. Inject Flex attributes:
   - `direction="row"` (or `column` depending on list type)
   - `alignment="top-left"`
   - `gap="4"`
   - `width="fill"`

---

## Tables (`<wm-table>`)

- **NDS Pattern:** Basic instantiation `<wm-table editmode="inline"...>`
- **DS Pattern:** Adds base UI variant.

### Action

Append `variant="default"` to the `<wm-table>` tag.

---

## Script

Extracted by the skill assembler into `wm_comp_conv_tmp.py`. Edit this block to add or change
data-component conversion behaviour — the skill picks it up automatically on the next run.

Signature contract: `apply_data_rules(text) -> (text, counts_dict)`.
Use `parse_attrs`, `build_attrs`, and `merge_class` — they are injected by the assembler header.

```python
# Execution order: 1
# Components: wm-form, wm-livefilter, wm-list, wm-listtemplate, wm-table, wm-container

def apply_data_rules(text):
    counts = {
        'form_itemsperrow': 0,
        'wm_list': 0,
        'wm_listtemplate': 0,
        'wm_table': 0,
        'wm_container': 0,
    }

    def patch_form_tag(m):
        tag, attr_str = m.group(1), m.group(2)
        attrs = parse_attrs(attr_str)
        cols = attrs.get('columns', '')
        if cols and 'itemsperrow' not in attrs:
            counts['form_itemsperrow'] += 1
            return f'<{tag} {attr_str} itemsperrow="xs-{cols} sm-{cols} md-{cols} lg-{cols}">'
        return m.group(0)

    def patch_list(m):
        attrs = parse_attrs(m.group(1))
        attrs.pop('listclass', None)
        attrs.setdefault('direction', 'column')
        attrs.setdefault('alignment', 'top-left')
        attrs.setdefault('gap', '4')
        attrs.setdefault('wrap', 'false')
        counts['wm_list'] += 1
        return f'<wm-list {build_attrs(attrs)}>'

    def patch_listtemplate(m):
        attrs = parse_attrs(m.group(1))
        attrs.setdefault('direction', 'row')
        attrs.setdefault('alignment', 'top-left')
        attrs.setdefault('gap', '4')
        attrs.setdefault('width', 'fill')
        counts['wm_listtemplate'] += 1
        return f'<wm-listtemplate {build_attrs(attrs)}>'

    def patch_table(m):
        attrs = parse_attrs(m.group(1))
        if 'variant' not in attrs:
            attrs['variant'] = 'default'
            counts['wm_table'] += 1
        return f'<wm-table {build_attrs(attrs)}>'

    def patch_container(m):
        attrs = parse_attrs(m.group(1))
        attrs.setdefault('direction', 'row')
        attrs.setdefault('alignment', 'top-left')
        attrs.setdefault('gap', '4')
        attrs.setdefault('width', 'fill')
        attrs['class'] = merge_class(attrs.get('class', ''), 'app-container-default')
        attrs.setdefault('variant', 'default')
        counts['wm_container'] += 1
        return f'<wm-container {build_attrs(attrs)}>'

    text = re.sub(r'<(wm-form|wm-livefilter)\b([^>]*)>', patch_form_tag, text)
    text = re.sub(r'<wm-list\b([^>]*)>', patch_list, text)
    text = re.sub(r'<wm-listtemplate\b([^>]*)>', patch_listtemplate, text)
    text = re.sub(r'<wm-table(?!-)([^>]*)>', patch_table, text)
    text = re.sub(r'<wm-container\b([^>]*)>', patch_container, text)
    text = re.sub(r'\s+class="(?:media-left|media-body)"', '', text)
    return text, counts
```
