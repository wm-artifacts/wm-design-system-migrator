# Rule 01: Form (`<wm-form>`)

## Overview

The `<wm-form>` component replaces its internal `<wm-layoutgrid>` hierarchy with a flat flex container, and gains a responsive `itemsperrow` constraint on the outer tag. Form action buttons are also updated to use the filled variant pattern.

---

## Hierarchy

```
NDS:
<wm-form ...>
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
    counts = {'wm_form_itemsperrow': 0, 'wm_form_layoutgrid_converted': 0}

    def matching_close_span(s, name, body_start, exclude_dash=False):
        guard = r'(?!-)' if exclude_dash else ''
        open_pat = re.compile(r'<' + re.escape(name) + r'\b' + guard +
                              r'((?:[^>"\']|"[^"]*"|\'[^\']*\')*?)(/?)>', re.DOTALL)
        close_pat = re.compile(r'</' + re.escape(name) + r'\s*>')
        depth, pos = 1, body_start
        while pos < len(s):
            o = open_pat.search(s, pos)
            c = close_pat.search(s, pos)
            if not c:
                return -1, -1
            if o and o.start() < c.start():
                if o.group(2) != '/':          # not self-closing -> deeper nesting
                    depth += 1
                pos = o.end()
            else:
                depth -= 1
                if depth == 0:
                    return c.start(), c.end()
                pos = c.end()
        return -1, -1

    def strip_first_level_grid(inner):
        token_re = re.compile(
            r'<(/?)(wm-layoutgrid|wm-gridrow|wm-gridcolumn)\b'
            r'((?:[^>"\']|"[^"]*"|\'[^\']*\')*?)(/?)>', re.DOTALL)
        out, pos, nest = [], 0, 0
        for t in token_re.finditer(inner):
            out.append(inner[pos:t.start()])
            pos = t.end()
            is_close, name, self_close = t.group(1) == '/', t.group(2), t.group(4) == '/'
            if name == 'wm-layoutgrid':
                out.append(t.group(0))         # nested grid: keep verbatim, track depth
                if is_close:
                    nest -= 1
                elif not self_close:
                    nest += 1
            elif nest > 0:                     # gridrow/gridcolumn inside a nested grid: keep
                out.append(t.group(0))
            # else: first-level gridrow/gridcolumn -> drop the wrapper tag
        out.append(inner[pos:])
        return ''.join(out)

    lg_open_re = re.compile(r'<wm-layoutgrid\b((?:[^>"\']|"[^"]*"|\'[^\']*\')*?)(/?)>', re.DOTALL)
    form_open_re = re.compile(r'<wm-form(?!-)((?:[^>"\']|"[^"]*"|\'[^\']*\')*)>', re.DOTALL)

    out, cursor = [], 0
    for fm in form_open_re.finditer(text):
        if fm.start() < cursor:
            continue
        out.append(text[cursor:fm.start()])
        form_attrs = parse_attrs(fm.group(1))

        fclose_start, fclose_end = matching_close_span(text, 'wm-form', fm.end(), exclude_dash=True)
        body_end = fclose_start if fclose_start != -1 else len(text)
        body = text[fm.end():body_end]

        # Column count comes from the immediate-child <wm-layoutgrid>, not the <wm-form> tag.
        cols, new_body = '', body
        lg = lg_open_re.search(body)
        if lg and lg.group(2) != '/':
            lg_attrs = parse_attrs(lg.group(1))
            cols = lg_attrs.get('columns', '')
            lc_start, lc_end = matching_close_span(body, 'wm-layoutgrid', lg.end())
            if lc_start != -1:
                stripped = strip_first_level_grid(body[lg.end():lc_start])
                # Convert outermost wm-layoutgrid -> wm-container, keeping its attributes intact.
                cont = {'direction': 'row', 'alignment': 'top-left', 'wrap': 'true', 'width': 'fill'}
                for k, v in lg_attrs.items():
                    if k not in ('class', 'variant'):
                        cont[k] = v
                cont['class'] = merge_class(lg_attrs.get('class', ''), 'app-container-default')
                cont['variant'] = lg_attrs.get('variant', 'default')
                new_body = (body[:lg.start()] + f'<wm-container {build_attrs(cont)}>' +
                            stripped + '</wm-container>' + body[lc_end:])
                counts['wm_form_layoutgrid_converted'] += 1

        if 'itemsperrow' not in form_attrs:
            form_attrs['itemsperrow'] = (f'xs-{cols} sm-{cols} md-{cols} lg-{cols}'
                                         if cols else 'xs-1 sm-1 md-1 lg-1')
            counts['wm_form_itemsperrow'] += 1

        out.append(f'<wm-form {build_attrs(form_attrs)}>')
        out.append(new_body)
        if fclose_start != -1:
            out.append(text[fclose_start:fclose_end])
            cursor = fclose_end
        else:
            cursor = body_end

    out.append(text[cursor:])
    return ''.join(out), counts
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
    counts = {'wm_liveform_itemsperrow': 0, 'wm_liveform_layoutgrid_converted': 0}

    def matching_close_span(s, name, body_start):
        """(start, end) of the </name> that matches the open whose body starts at body_start.
        Depth-aware (skips self-closing opens)."""
        open_pat = re.compile(r'<' + re.escape(name) +
                              r'\b((?:[^>"\']|"[^"]*"|\'[^\']*\')*?)(/?)>', re.DOTALL)
        close_pat = re.compile(r'</' + re.escape(name) + r'\s*>')
        depth, pos = 1, body_start
        while pos < len(s):
            o = open_pat.search(s, pos)
            c = close_pat.search(s, pos)
            if not c:
                return -1, -1
            if o and o.start() < c.start():
                if o.group(2) != '/':          # not self-closing -> deeper nesting
                    depth += 1
                pos = o.end()
            else:
                depth -= 1
                if depth == 0:
                    return c.start(), c.end()
                pos = c.end()
        return -1, -1

    def strip_first_level_grid(inner):
        """Drop only the first-level <wm-gridrow>/<wm-gridcolumn> wrappers, preserving their
        children. Any nested <wm-layoutgrid> subtree is left untouched."""
        token_re = re.compile(
            r'<(/?)(wm-layoutgrid|wm-gridrow|wm-gridcolumn)\b'
            r'((?:[^>"\']|"[^"]*"|\'[^\']*\')*?)(/?)>', re.DOTALL)
        out, pos, nest = [], 0, 0
        for t in token_re.finditer(inner):
            out.append(inner[pos:t.start()])
            pos = t.end()
            is_close, name, self_close = t.group(1) == '/', t.group(2), t.group(4) == '/'
            if name == 'wm-layoutgrid':
                out.append(t.group(0))         # nested grid: keep verbatim, track depth
                if is_close:
                    nest -= 1
                elif not self_close:
                    nest += 1
            elif nest > 0:                     # gridrow/gridcolumn inside a nested grid: keep
                out.append(t.group(0))
            # else: first-level gridrow/gridcolumn -> drop the wrapper tag
        out.append(inner[pos:])
        return ''.join(out)

    lg_open_re = re.compile(r'<wm-layoutgrid\b((?:[^>"\']|"[^"]*"|\'[^\']*\')*?)(/?)>', re.DOTALL)
    lf_open_re = re.compile(r'<wm-liveform\b((?:[^>"\']|"[^"]*"|\'[^\']*\')*)>', re.DOTALL)

    out, cursor = [], 0
    for fm in lf_open_re.finditer(text):
        if fm.start() < cursor:
            continue
        out.append(text[cursor:fm.start()])
        lf_attrs = parse_attrs(fm.group(1))

        # Inline liveforms (inside wm-livetable) are handled by Rule 04 — leave them untouched.
        if lf_attrs.get('formlayout') == 'inline':
            out.append(fm.group(0))
            cursor = fm.end()
            continue

        lfc_start, lfc_end = matching_close_span(text, 'wm-liveform', fm.end())
        body_end = lfc_start if lfc_start != -1 else len(text)
        body = text[fm.end():body_end]

        # Column count comes from the immediate-child <wm-layoutgrid>, not the <wm-liveform> tag.
        cols, new_body = '', body
        lg = lg_open_re.search(body)
        if lg and lg.group(2) != '/':
            lg_attrs = parse_attrs(lg.group(1))
            cols = lg_attrs.get('columns', '')
            lc_start, lc_end = matching_close_span(body, 'wm-layoutgrid', lg.end())
            if lc_start != -1:
                stripped = strip_first_level_grid(body[lg.end():lc_start])
                # Convert outermost wm-layoutgrid -> wm-container, keeping its attributes intact.
                cont = {'direction': 'row', 'alignment': 'top-left', 'wrap': 'true', 'width': 'fill'}
                for k, v in lg_attrs.items():
                    if k not in ('class', 'variant'):
                        cont[k] = v
                cont['class'] = merge_class(lg_attrs.get('class', ''), 'app-container-default')
                cont['variant'] = lg_attrs.get('variant', 'default')
                new_body = (body[:lg.start()] + f'<wm-container {build_attrs(cont)}>' +
                            stripped + '</wm-container>' + body[lc_end:])
                counts['wm_liveform_layoutgrid_converted'] += 1

        if 'itemsperrow' not in lf_attrs:
            lf_attrs['itemsperrow'] = (f'xs-{cols} sm-{cols} md-{cols} lg-{cols}'
                                       if cols else 'xs-1 sm-1 md-1 lg-1')
            counts['wm_liveform_itemsperrow'] += 1

        out.append(f'<wm-liveform {build_attrs(lf_attrs)}>')
        out.append(new_body)
        if lfc_start != -1:
            out.append(text[lfc_start:lfc_end])
            cursor = lfc_end
        else:
            cursor = body_end

    out.append(text[cursor:])
    return ''.join(out), counts
```



# Rule 04: Data Table (`<wm-table>`)

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

    text = re.sub(r'<wm-table(?!-)((?:[^>"\']|"[^"]*"|\'[^\']*\')*)>', patch_table, text)
    return text, counts
```

---

# Rule 05: List (`<wm-list>`)

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
- `height="56px"`
- `padding="12px"`

> **Note:** `class="media-left"` and `class="media-body"` on any child elements inside the listtemplate are stripped.

---

## Script

Signature contract: `apply_list_rules(text) -> (text, counts_dict)`.

```python
# Execution order: 6
# Components: wm-list, wm-listtemplate

def apply_list_rules(text):
    counts = {'wm_list': 0, 'wm_listtemplate': 0}

    # Card-variant lists/templates (a <wm-listtemplate> whose content contains a <wm-card>)
    # are handled by apply_card_rules (order 7). They must be left untouched here, so the
    # negative lookaheads below skip any list/template that has a wm-card descendant.
    # The tempered token stops the scan at the next list/listtemplate boundary so it never
    # leaks into a sibling list.
    CARD_CHILD = r'(?:(?!<wm-(?:list|listtemplate)\b).)*?<wm-card\b'

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
        attrs.setdefault('height', 'fill')
        attrs.setdefault('padding', '12px')
        counts['wm_listtemplate'] += 1
        return f'<wm-listtemplate {build_attrs(attrs)}>'

    # Skip card-variant lists: <wm-list> directly wrapping a <wm-listtemplate> that holds a <wm-card>.
    text = re.sub(
        r'<wm-list\b((?:[^>"\']|"[^"]*"|\'[^\']*\')*)>'
        r'(?!\s*<wm-listtemplate\b(?:[^>"\']|"[^"]*"|\'[^\']*\')*>' + CARD_CHILD + r')',
        patch_list, text, flags=re.DOTALL)
    # Skip card-variant templates: <wm-listtemplate> whose content holds a <wm-card>.
    text = re.sub(
        r'<wm-listtemplate\b((?:[^>"\']|"[^"]*"|\'[^\']*\')*)>'
        r'(?!' + CARD_CHILD + r')',
        patch_listtemplate, text, flags=re.DOTALL)
    text = re.sub(r'\s+class="(?:media-left|media-body)"', '', text)
    return text, counts
```

---

# Rule 06: Card (`<wm-list>` Card Variant)

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
    <!-- content directly (wm-container wrappers) -->
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
## Convert card wrapper elements into <wm-container>

- Replace opening and closing tags of `<wm-card>`, `<wm-card-content>`, `<wm-card-footer>` with `<wm-container>`.
- Prserve all child elemnts inside them. Preserve all attributes of the wrapper elements.
- Replace the 'picturesource' attribute with 'backgroundimage' attribute.

> **Note:** The `<wm-listtemplate>` `width`, `height`, and `padding` values above are defaults from the DS drag-and-drop template. Adjust per actual design requirements.

---

## Script

Signature contract: `apply_card_rules(text) -> (text, counts_dict)`.

```python
# Execution order: 7
# Components: wm-card, wm-card-content, wm-card-footer (converted to wm-container)
# Also patches wm-list and wm-listtemplate that have wm-card as a child (card variant).
# Runs AFTER apply_list_rules (order 6). apply_list_rules deliberately SKIPS card-variant
# wm-list / wm-listtemplate (detected by a wm-card descendant), so the card-specific layout
# values set here are the only ones applied to card lists — no conflict, no double-patching.
# Non-card wm-list / wm-listtemplate are untouched here; apply_list_rules handles them.
# Assumption: wm-list does not nest inside wm-list in practice.

def apply_card_rules(text):
    counts = {'wm_card_converted': 0, 'wm_list_card': 0, 'wm_listtemplate_card': 0}

    # 1. Patch opening <wm-list> tags that are the direct wrapper of a card-variant template
    def patch_list_open(m):
        attrs = parse_attrs(m.group(1))
        attrs.pop('listclass', None)
        attrs.setdefault('direction', 'row')
        attrs.setdefault('alignment', 'top-left')
        attrs.setdefault('gap', '4')
        attrs.setdefault('columngap', '4')
        attrs.setdefault('wrap', 'true')
        attrs.setdefault('itemsperrow', 'auto')
        counts['wm_list_card'] += 1
        return f'<wm-list {build_attrs(attrs)}>'

    list_pattern = r'<wm-list\b((?:[^>"\']|"[^"]*"|\'[^\']*\')*)>(?=\s*<wm-listtemplate\b(?:[^>"\']|"[^"]*"|\'[^\']*\')*>(?:(?!<wm-(?:list|listtemplate)\b).)*?<wm-card\b)'
    text = re.sub(list_pattern, patch_list_open, text, flags=re.DOTALL)

    # 2. Patch opening <wm-listtemplate> tags that are the direct parent of a card
    def patch_listtemplate_open(m):
        attrs = parse_attrs(m.group(1))
        attrs.setdefault('direction', 'row')
        attrs.setdefault('alignment', 'top-left')
        attrs.setdefault('gap', '4')
        attrs.setdefault('width', '280px')
        attrs.setdefault('height', '260px')
        attrs.setdefault('padding', '12px')
        counts['wm_listtemplate_card'] += 1
        return f'<wm-listtemplate {build_attrs(attrs)}>'

    template_pattern = r'<wm-listtemplate\b((?:[^>"\']|"[^"]*"|\'[^\']*\')*)>(?=(?:(?!<wm-(?:list|listtemplate)\b).)*?<wm-card\b)'
    text = re.sub(template_pattern, patch_listtemplate_open, text, flags=re.DOTALL)

    # 3. Convert all cards to wm-container 
    def patch_card_elements(m):
        tag_name = m.group(1)
        attrs = parse_attrs(m.group(2))
        class_map = {
            'wm-card': 'app-card card app-panel',
            'wm-card-content': 'app-card-content card-body card-block',
            'wm-card-footer': 'app-card-footer card-footer'
        }
        default_cls = class_map.get(tag_name, '')
        attrs['class'] = merge_class(attrs.get('class', ''), default_cls)
        if 'picturesource' in attrs:
            attrs['backgroundimage'] = attrs.pop('picturesource')
        for attr in list(attrs.keys()):
            if attr == 'actions' or attr.startswith('item'):
                attrs.pop(attr, None)
        
        counts['wm_card_converted'] += 1
        return f'<wm-container {build_attrs(attrs)}>'

    # Global replacement for opening tags
    text = re.sub(r'<(wm-card-content|wm-card-footer|wm-card)\b((?:[^>"\']|"[^"]*"|\'[^\']*\')*)>', patch_card_elements, text)
    # Global replacement for closing tags (relaxed regex)
    text = re.sub(r'</(wm-card|wm-card-content|wm-card-footer)\s*>', '</wm-container>', text)

    return text, counts
```

---

# Rule 07: Live Filter (`<wm-livefilter>`)

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

    def matching_close_span(s, name, body_start):
        """(start, end) of the </name> that matches the open whose body starts at body_start.
        Depth-aware (skips self-closing opens)."""
        open_pat = re.compile(r'<' + re.escape(name) +
                              r'\b((?:[^>"\']|"[^"]*"|\'[^\']*\')*?)(/?)>', re.DOTALL)
        close_pat = re.compile(r'</' + re.escape(name) + r'\s*>')
        depth, pos = 1, body_start
        while pos < len(s):
            o = open_pat.search(s, pos)
            c = close_pat.search(s, pos)
            if not c:
                return -1, -1
            if o and o.start() < c.start():
                if o.group(2) != '/':          # not self-closing -> deeper nesting
                    depth += 1
                pos = o.end()
            else:
                depth -= 1
                if depth == 0:
                    return c.start(), c.end()
                pos = c.end()
        return -1, -1

    lg_open_re = re.compile(r'<wm-layoutgrid\b((?:[^>"\']|"[^"]*"|\'[^\']*\')*?)(/?)>', re.DOTALL)
    lf_open_re = re.compile(r'<wm-livefilter\b((?:[^>"\']|"[^"]*"|\'[^\']*\')*)>', re.DOTALL)

    out, cursor = [], 0
    for fm in lf_open_re.finditer(text):
        if fm.start() < cursor:
            continue
        out.append(text[cursor:fm.start()])
        lf_attrs = parse_attrs(fm.group(1))

        lfc_start, lfc_end = matching_close_span(text, 'wm-livefilter', fm.end())
        body_end = lfc_start if lfc_start != -1 else len(text)
        body = text[fm.end():body_end]

        # Column count comes from the immediate-child <wm-layoutgrid>, not the <wm-livefilter> tag.
        cols = ''
        lg = lg_open_re.search(body)
        if lg and lg.group(2) != '/':
            lg_attrs = parse_attrs(lg.group(1))
            cols = lg_attrs.get('columns', '')

        if 'itemsperrow' not in lf_attrs:
            lf_attrs['itemsperrow'] = (f'xs-1 sm-{cols} md-{cols} lg-{cols}'
                                       if cols else 'xs-1 sm-1 md-1 lg-1')
            counts['wm_livefilter_itemsperrow'] += 1

        out.append(f'<wm-livefilter {build_attrs(lf_attrs)}>')
        out.append(body)
        if lfc_start != -1:
            out.append(text[lfc_start:lfc_end])
            cursor = lfc_end
        else:
            cursor = body_end

    out.append(text[cursor:])
    return ''.join(out), counts
```
