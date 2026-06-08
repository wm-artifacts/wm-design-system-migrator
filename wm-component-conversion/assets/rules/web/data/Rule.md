# Rule 01: Form (`<wm-form>`)

## Overview

The `<wm-form>` component replaces its internal `<wm-layoutgrid>` hierarchy with a flat flex container, and gains a responsive `itemsperrow` constraint on the outer tag. Form action buttons are also updated to use the filled variant pattern.

---

## Hierarchy

```
NDS:
<wm-form columns="2" ...>
  <wm-layoutgrid columns="2">
    <wm-gridrow>
      <wm-gridcolumn columnwidth="6">
        <wm-form-field ...></wm-form-field>
      </wm-gridcolumn>
    </wm-gridrow>
  </wm-layoutgrid>
  <wm-form-action class="btn-default ..."></wm-form-action>
  <wm-form-action class="btn-success ..."></wm-form-action>
</wm-form>

DS:
<wm-form itemsperrow="xs-2 sm-2 md-2 lg-2" ...>
  <wm-container direction="row" alignment="top-left" wrap="true" width="fill" columns="2" class="app-container-default" variant="default">
    <wm-form-field ...></wm-form-field>
    <wm-form-field ...></wm-form-field>
  </wm-container>
  <wm-form-action class="btn-default btn-filled ..."></wm-form-action>
  <wm-form-action class="btn-primary btn-filled ..."></wm-form-action>
</wm-form>
```

---

## Step 1: `<wm-form>` outer tag

- Determine column count from the inner `<wm-layoutgrid columns="N">`.
- Add `itemsperrow="xs-N sm-N md-N lg-N"` to the `<wm-form>` tag.

## Step 2: Remove layout grid wrappers

- Delete `<wm-layoutgrid>`, `<wm-gridrow>`, and `<wm-gridcolumn>` tags entirely.
- Preserve all `<wm-form-field>` children.

## Step 3: Wrap fields in `<wm-container>`

- Wrap the extracted `<wm-form-field>` elements in:

```html
<wm-container direction="row" alignment="top-left" wrap="true" width="fill"
    columns="{N}" name="container1" class="app-container-default" variant="default">
    <!-- wm-form-field elements -->
</wm-container>
```

## Step 4: Update `<wm-form-action>` buttons

- Add `btn-filled` to the existing class.
- Map button context class to variant:
  - `btn-default` → keep `btn-default`, add `btn-filled`
  - `btn-success` → replace with `btn-primary`, add `btn-filled`

---

## Script

Signature contract: `apply_form_rules(text) -> (text, counts_dict)`.

```python
# Execution order: 2
# Components: wm-form, wm-form-action (within wm-form context)

def apply_form_rules(text):
    counts = {'wm_form_itemsperrow': 0}

    def patch_form(m):
        attrs = parse_attrs(m.group(2))
        cols = attrs.get('columns', '')
        if cols and 'itemsperrow' not in attrs:
            attrs['itemsperrow'] = f'xs-{cols} sm-{cols} md-{cols} lg-{cols}'
            counts['wm_form_itemsperrow'] += 1
        return f'<wm-form {build_attrs(attrs)}>'

    text = re.sub(r'<(wm-form)\b([^>]*)>', patch_form, text)
    return text, counts
```

---

# Rule 03: Live Form (`<wm-liveform>`)

## Overview

The standalone `<wm-liveform>` follows the same structural pattern as `<wm-form>`: the internal `<wm-layoutgrid>` hierarchy is replaced by a `<wm-container>`, and `itemsperrow` is added to the outer tag.

---

## Hierarchy

```
NDS:
<wm-liveform ...>
  <wm-layoutgrid columns="3">
    <wm-gridrow>
      <wm-gridcolumn columnwidth="4">
        <wm-form-field ...></wm-form-field>
      </wm-gridcolumn>
    </wm-gridrow>
  </wm-layoutgrid>
  <wm-form-action class="btn-default ..."></wm-form-action>
  <wm-form-action class="btn-success ..."></wm-form-action>
</wm-liveform>

DS:
<wm-liveform itemsperrow="xs-3 sm-3 md-3 lg-3" ...>
  <wm-container direction="row" alignment="top-left" wrap="true" width="fill" columns="3" class="app-container-default" variant="default">
    <wm-form-field ...></wm-form-field>
    <wm-form-field ...></wm-form-field>
    <wm-form-field ...></wm-form-field>
  </wm-container>
  <wm-form-action class="btn-default ..."></wm-form-action>
  <wm-form-action class="btn-success ..."></wm-form-action>
</wm-liveform>
```

---

## Step 1: `<wm-liveform>` outer tag

- Determine column count from inner `<wm-layoutgrid columns="N">`.
- Add `itemsperrow="xs-N sm-N md-N lg-N"` to the `<wm-liveform>` tag.

## Step 2: Remove layout grid wrappers

- Delete `<wm-layoutgrid>`, `<wm-gridrow>`, and `<wm-gridcolumn>` tags entirely.

## Step 3: Wrap fields in `<wm-container>`

```html
<wm-container direction="row" alignment="top-left" wrap="true" width="fill"
    columns="{N}" name="container2" class="app-container-default" variant="default">
    <!-- wm-form-field elements -->
</wm-container>
```

> **Note:** `<wm-form-action>` buttons inside `<wm-liveform>` are handled by the Basic rules (execution order 2) via `apply_basic_rules`.

---

## Script

Signature contract: `apply_liveform_rules(text) -> (text, counts_dict)`.

```python
# Execution order: 3
# Components: wm-liveform (standalone, outside wm-livetable)

def apply_liveform_rules(text):
    counts = {'wm_liveform_itemsperrow': 0}

    def patch_liveform(m):
        attrs = parse_attrs(m.group(1))
        cols = attrs.get('columns', '')
        if cols and 'itemsperrow' not in attrs:
            attrs['itemsperrow'] = f'xs-{cols} sm-{cols} md-{cols} lg-{cols}'
            counts['wm_liveform_itemsperrow'] += 1
        return f'<wm-liveform {build_attrs(attrs)}>'

    text = re.sub(r'<wm-liveform\b([^>]*)>', patch_liveform, text)
    return text, counts
```

---

# Rule 04: Live Table (`<wm-livetable>`)

## Overview

`<wm-livetable>` is a composite container that wraps a `<wm-table>` and an inline `<wm-liveform>`. The outer `<wm-livetable>` tag itself is unchanged. The inner `<wm-table>` gets `variant="default"` (handled by Rule 05). The inner `<wm-liveform>` gets `itemsperrow="xs-2 sm-2 md-2 lg-2"` — a fixed two-column constraint for the inline form layout.

---

## Hierarchy

```
NDS:
<wm-livetable name="...">
  <wm-table ... navigation="Classic"></wm-table>
  <wm-liveform formlayout="inline" ...></wm-liveform>
</wm-livetable>

DS:
<wm-livetable name="...">
  <wm-table ... navigation="Classic"></wm-table>          ← variant="default" added (Rule 05)
  <wm-liveform formlayout="inline" itemsperrow="xs-2 sm-2 md-2 lg-2" ...></wm-liveform>
</wm-livetable>
```

---

## `<wm-livetable>` outer tag

- **Action:** None — the `<wm-livetable>` tag is unchanged.

## `<wm-table>` inside `<wm-livetable>`

- **Action:** Add `variant="default"` — handled by Rule 05 (`apply_datatable_rules`).

## `<wm-liveform>` inside `<wm-livetable>`

- **NDS Pattern:** `<wm-liveform formlayout="inline" ...>` — no `itemsperrow`.
- **DS Pattern:** `<wm-liveform formlayout="inline" itemsperrow="xs-2 sm-2 md-2 lg-2" ...>`

### Action

Add `itemsperrow="xs-2 sm-2 md-2 lg-2"` to any `<wm-liveform>` that has `formlayout="inline"` and does not already have `itemsperrow`.

---

## Script

Signature contract: `apply_livetable_rules(text) -> (text, counts_dict)`.

```python
# Execution order: 4
# Components: wm-liveform (inline, inside wm-livetable — formlayout="inline")

def apply_livetable_rules(text):
    counts = {'wm_liveform_inline_itemsperrow': 0}

    def patch_inline_liveform(m):
        attrs = parse_attrs(m.group(1))
        if attrs.get('formlayout') == 'inline' and 'itemsperrow' not in attrs:
            attrs['itemsperrow'] = 'xs-2 sm-2 md-2 lg-2'
            counts['wm_liveform_inline_itemsperrow'] += 1
        return f'<wm-liveform {build_attrs(attrs)}>'

    text = re.sub(r'<wm-liveform\b([^>]*)>', patch_inline_liveform, text)
    return text, counts
```

---

# Rule 05: Data Table (`<wm-table>`)

## Overview

The `<wm-table>` component requires only one change: a `variant="default"` attribute is added to the opening tag. All other attributes, columns, and child elements are preserved as-is.

---

## Hierarchy

```
NDS:
<wm-table statehandler="URL" name="..." dataset="..." navigation="Basic">
  <wm-table-column binding="name" caption="Name" ...></wm-table-column>
</wm-table>

DS:
<wm-table statehandler="URL" name="..." dataset="..." navigation="Basic" variant="default">
  <wm-table-column binding="name" caption="Name" ...></wm-table-column>
</wm-table>
```

---

## Action

Append `variant="default"` to the `<wm-table>` opening tag if not already present.

> **Note:** The `(?!-)` negative lookahead in the regex prevents matching `<wm-table-column>` and `<wm-table-action>` child tags.

---

## Script

Signature contract: `apply_datatable_rules(text) -> (text, counts_dict)`.

```python
# Execution order: 5
# Components: wm-table

def apply_datatable_rules(text):
    counts = {'wm_table': 0}

    def patch_table(m):
        attrs = parse_attrs(m.group(1))
        if 'variant' not in attrs:
            attrs['variant'] = 'default'
            counts['wm_table'] += 1
        return f'<wm-table {build_attrs(attrs)}>'

    text = re.sub(r'<wm-table(?!-)([^>]*)>', patch_table, text)
    return text, counts
```

---

# Rule 06: List (`<wm-list>`)

## Overview

The `<wm-list>` component drops the legacy `listclass` attribute and adopts explicit flex layout properties. The inner `<wm-listtemplate>` also gains flex attributes to define its row/column direction and sizing.

---

## Hierarchy

```
NDS:
<wm-list listclass="list-group" itemclass="list-group-item" template="true"
    itemsperrow="xs-1 sm-1 md-1 lg-1" class="media-list" statehandler="URL" ...>
  <wm-listtemplate layout="inline" name="listtemplate1">
    <!-- list item content -->
  </wm-listtemplate>
</wm-list>

DS:
<wm-list itemclass="list-group-item" template="true" direction="column"
    alignment="top-left" gap="4" wrap="false" itemsperrow="xs-1 sm-1 md-1 lg-1"
    class="media-list" statehandler="URL" ...>
  <wm-listtemplate layout="inline" direction="row" alignment="top-left" gap="4"
      width="fill" padding="12px" height="56" name="listtemplate1">
  </wm-listtemplate>
</wm-list>
```

---

## `<wm-list>` changes

1. Remove `listclass="list-group"`.
2. Add flex attributes (if not already present):
   - `direction="column"`
   - `alignment="top-left"`
   - `gap="4"`
   - `wrap="false"`

## `<wm-listtemplate>` changes

Add flex attributes (if not already present):
- `direction="row"`
- `alignment="top-left"`
- `gap="4"`
- `width="fill"`

> **Note:** `class="media-left"` and `class="media-body"` on any child elements inside the listtemplate are stripped.

---

## Script

Signature contract: `apply_list_rules(text) -> (text, counts_dict)`.

```python
# Execution order: 6
# Components: wm-list, wm-listtemplate

def apply_list_rules(text):
    counts = {'wm_list': 0, 'wm_listtemplate': 0}

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

    text = re.sub(r'<wm-list\b([^>]*)>', patch_list, text)
    text = re.sub(r'<wm-listtemplate\b([^>]*)>', patch_listtemplate, text)
    text = re.sub(r'\s+class="(?:media-left|media-body)"', '', text)
    return text, counts
```

---

# Rule 07: Card (`<wm-list>` Card Variant)

## Overview

The card-style list uses a different `<wm-list>` configuration with row-direction flex layout and auto column sizing. The legacy `<wm-card>`, `<wm-card-content>`, and `<wm-card-footer>` wrapper elements **do not exist** in design system projects and must be removed entirely while preserving their child content.

---

## Hierarchy

```
NDS:
<wm-list listclass="list-group" template="true" template-name="Media Post"
    itemsperrow="xs-1 sm-2 md-2 lg-3" class="media list-card" statehandler="URL" ...>
  <wm-listtemplate layout="media" name="listtemplate1">
    <wm-card name="Picture">
      <wm-card-content name="card_content1" padding="1.25em">
        <!-- content -->
      </wm-card-content>
      <wm-card-footer name="card_footer1">
        <!-- footer -->
      </wm-card-footer>
    </wm-card>
  </wm-listtemplate>
</wm-list>

DS:
<wm-list template="true" template-name="Blank Card" direction="row"
    alignment="top-left" gap="4" columngap="4" wrap="true" itemsperrow="auto"
    statehandler="URL" ...>
  <wm-listtemplate layout="media" direction="row" alignment="top-left" gap="4"
      width="280px" height="260px" padding="12px" name="listtemplate1">
    <!-- content directly (no wm-card wrappers) -->
  </wm-listtemplate>
</wm-list>
```

---

## `<wm-list>` outer tag changes

1. Remove `listclass="list-group"`.
2. Change `itemsperrow` to `"auto"`.
3. Set flex attributes:
   - `direction="row"`
   - `alignment="top-left"`
   - `gap="4"`
   - `columngap="4"`
   - `wrap="true"`

## `<wm-listtemplate>` changes

Set explicit card dimensions and flex attributes:
- `direction="row"`
- `alignment="top-left"`
- `gap="4"`
- `width="280px"`
- `height="260px"`
- `padding="12px"`

## Remove card wrapper elements

- Remove opening and closing tags of `<wm-card>`, `<wm-card-content>`, `<wm-card-footer>`.
- Preserve all child content inside them.

> **Note:** The `<wm-listtemplate>` `width`, `height`, and `padding` values above are defaults from the DS drag-and-drop template. Adjust per actual design requirements.

---

## Script

Signature contract: `apply_card_rules(text) -> (text, counts_dict)`.

```python
# Execution order: 7
# Components: wm-card, wm-card-content, wm-card-footer (removal only)
# Note: wm-list and wm-listtemplate for card layout are handled by apply_list_rules (Rule 06).
# Card-specific wm-list overrides (itemsperrow="auto", direction="row", wrap="true", etc.)
# must be applied manually or detected via template-name="Media Post" heuristic.

def apply_card_rules(text):
    counts = {'wm_card_removed': 0}

    def remove_card_open(m):
        counts['wm_card_removed'] += 1
        return ''

    # Remove opening tags for card wrapper elements
    text = re.sub(r'<(?:wm-card|wm-card-content|wm-card-footer)\b[^>]*>', remove_card_open, text)
    # Remove closing tags for card wrapper elements
    text = re.sub(r'</(?:wm-card|wm-card-content|wm-card-footer)>', '', text)
    return text, counts
```

---

# Rule 08: Live Filter (`<wm-livefilter>`)

## Overview

`<wm-livefilter>` follows the same structural migration as `<wm-form>`: the internal `<wm-layoutgrid>` hierarchy is removed and replaced with a `<wm-container>`, and `itemsperrow` is added to the outer tag based on the original column count.

---

## Hierarchy

```
NDS:
<wm-livefilter name="..." dataset="..." ...>
  <wm-layoutgrid columns="1">
    <wm-gridrow>
      <wm-gridcolumn columnwidth="12">
        <wm-filter-field ...></wm-filter-field>
      </wm-gridcolumn>
    </wm-gridrow>
  </wm-layoutgrid>
  <wm-filter-action key="filter" class="btn-primary" ...></wm-filter-action>
  <wm-filter-action key="clear" class="btn-default" ...></wm-filter-action>
</wm-livefilter>

DS:
<wm-livefilter itemsperrow="xs-1 sm-1 md-1 lg-1" name="..." dataset="..." ...>
  <wm-container direction="row" alignment="top-left" wrap="true" width="fill" columns="1" name="container1">
    <wm-filter-field ...></wm-filter-field>
    <wm-filter-field ...></wm-filter-field>
  </wm-container>
  <wm-filter-action key="filter" class="btn-primary" ...></wm-filter-action>
  <wm-filter-action key="clear" class="btn-default" ...></wm-filter-action>
</wm-livefilter>
```

---

## Step 1: `<wm-livefilter>` outer tag

- Determine column count from inner `<wm-layoutgrid columns="N">`.
- Add `itemsperrow="xs-N sm-N md-N lg-N"` to the `<wm-livefilter>` tag.

## Step 2: Remove layout grid wrappers

- Delete `<wm-layoutgrid>`, `<wm-gridrow>`, and `<wm-gridcolumn>` tags entirely.
- Preserve all `<wm-filter-field>` children.

## Step 3: Wrap fields in `<wm-container>`

```html
<wm-container direction="row" alignment="top-left" wrap="true" width="fill"
    columns="{N}" name="container1">
    <!-- wm-filter-field elements -->
</wm-container>
```

> **Note:** `<wm-filter-action>` buttons inside `<wm-livefilter>` are unchanged — `btn-primary` and `btn-default` classes are preserved as-is in the DS pattern for filter actions.

---

## Script

Signature contract: `apply_livefilter_rules(text) -> (text, counts_dict)`.

```python
# Execution order: 8
# Components: wm-livefilter

def apply_livefilter_rules(text):
    counts = {'wm_livefilter_itemsperrow': 0}

    def patch_livefilter(m):
        attrs = parse_attrs(m.group(1))
        cols = attrs.get('columns', '')
        if cols and 'itemsperrow' not in attrs:
            attrs['itemsperrow'] = f'xs-{cols} sm-{cols} md-{cols} lg-{cols}'
            counts['wm_livefilter_itemsperrow'] += 1
        return f'<wm-livefilter {build_attrs(attrs)}>'

    text = re.sub(r'<wm-livefilter\b([^>]*)>', patch_livefilter, text)
    return text, counts
```
