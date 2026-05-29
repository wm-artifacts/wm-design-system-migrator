---
name: wm-autolayout-conv
description: Use this skill to convert legacy WaveMaker grid layout widgets
  (wm-layoutgrid, wm-gridrow, wm-gridcolumn) and linear layout widgets (wm-linearlayout,
  wm-linearlayoutitem) to modern flex-based wm-container widgets in WaveMaker project page
  HTML files. It maps Bootstrap 12-column widths and flexgrow values to percentage widths,
  preserves element names and CSS classes, respects parent-direction context for
  linearlayoutitem, and automatically collapses redundant single-child converted wrappers.
  Optionally injects mobile-responsive CSS breakpoints via --responsive. Supports dry-run
  mode and page filtering via --pages. Use this skill when the user wants to modernize
  layout widgets in a WaveMaker project independently of a DesignSystem conversion — for example,
  on an already-DesignSystem project or as a standalone cleanup step. Do not use this skill for
  full DesignSystem template migration (use wm-designsystem-conv), or when the user wants the complete
  migration pipeline in one shot (use wm-studio-migrate instead).
metadata:
  version: 0.1.0
---

# /wm-autolayout-conv — WaveMaker Grid & LinearLayout → Flex Container Converter

Convert `wm-layoutgrid` / `wm-gridrow` / `wm-gridcolumn` and
`wm-linearlayout` / `wm-linearlayoutitem` markup to `wm-container`
flex-based layout in WaveMaker project page HTML files.

The generated containers use Bootstrap-mapped or flexgrow-mapped percentage widths and
`wrap="true"` so the layout is naturally responsive without extra configuration.

---

## Invocation

```
/wm-autolayout-conv <project_path>
/wm-autolayout-conv <project_path> --dry-run
/wm-autolayout-conv <project_path> --pages <Page1,Page2>
/wm-autolayout-conv <project_path> --responsive
```

| Argument | Required | Description |
|---|---|---|
| `<project_path>` | Yes | Absolute path to the WaveMaker project |
| `--dry-run` | No | Preview what would change — no files are written |
| `--pages <names>` | No | Comma-separated page names to convert (default: all pages) |
| `--responsive` | No | Also inject a mobile media-query into each modified page's `.css` file |

---

## Execution — follow every step in order

### STEP 0 · Parse arguments from $ARGUMENTS

Extract:
- `PROJECT_DIR` — first positional value
- `DRY_RUN` — `true` if `--dry-run` is present
- `PAGE_FILTER` — list of names after `--pages` (empty = all pages)
- `ADD_RESPONSIVE_CSS` — `true` if `--responsive` is present

If `PROJECT_DIR` is missing, ask: *"Please provide the absolute path to the WaveMaker project."*

---

### STEP 1 · Validate project and discover target files

Check that `<PROJECT_DIR>/src/main/webapp/pages/` exists.
If missing → abort: *"Not a valid WaveMaker project — pages directory not found."*

Find all HTML files matching `<PROJECT_DIR>/src/main/webapp/pages/**/*.html`.

Filter to files whose text contains `wm-layoutgrid` **or** `wm-linearlayout`.
If `PAGE_FILTER` is set, additionally restrict to pages whose folder name matches the list.

Count the qualifying files. If zero → abort: *"No layoutgrid or linearlayout markup found in the target pages. Nothing to do."*

---

### STEP 2 · Show discovery summary and confirm

Display a table:

```
Found layout markup in N page(s):

  Page              layoutgrids   gridrows   gridcolumns   linearlayouts   linearlayoutitems
  ──────────────    ───────────   ────────   ───────────   ─────────────   ─────────────────
  Main                   1            2           4               0                 0
  UserProfile            0            0           0               2                 5
  ...

Proceed with conversion? [Y/n]
```

If `--dry-run` is active, note that no files will be written.
Wait for user confirmation before continuing.

---

### STEP 3 · Write the conversion script and run it

Write the Python 3 script below to `<PROJECT_DIR>/wm_grid_conv_tmp.py`, run it
with `python3`, then delete it (`rm -f`). Parse its JSON output to build the summary.

```python
#!/usr/bin/env python3
"""wm-layoutgrid/gridrow/gridcolumn + wm-linearlayout/linearlayoutitem → wm-container converter."""
import re, os, sys, json
from pathlib import Path

PROJECT_DIR    = sys.argv[1]
DRY_RUN        = '--dry-run'    in sys.argv
ADD_RESPONSIVE = '--responsive' in sys.argv
PAGE_FILTER    = []
if '--pages' in sys.argv:
    idx = sys.argv.index('--pages')
    if idx + 1 < len(sys.argv):
        PAGE_FILTER = [p.strip() for p in sys.argv[idx + 1].split(',')]

# Bootstrap 12-col → flex width
COLUMN_WIDTH_MAP = {
    '1':  '8.33%',  '2':  '16.67%', '3':  '25%',   '4':  '33.33%',
    '5':  '41.67%', '6':  '50%',    '7':  '58.33%', '8':  '66.67%',
    '9':  '75%',    '10': '83.33%', '11': '91.67%', '12': 'fill',
}

# flexgrow (1–12) → flex width (same scale as bootstrap 12-col)
FLEXGROW_WIDTH_MAP = {
    '1':  '8.33%',  '2':  '16.67%', '3':  '25%',   '4':  '33.33%',
    '5':  '41.67%', '6':  '50%',    '7':  '58.33%', '8':  '66.67%',
    '9':  '75%',    '10': '83.33%', '11': '91.67%', '12': 'fill',
}

RESPONSIVE_CSS = """\n
/* Grid-to-container responsive: stack converted columns on mobile */
@media (max-width: 767px) {
    .app-container-default[direction="column"] {
        width: 100% !important;
        min-width: unset !important;
    }
}
"""

# ── attribute helpers ────────────────────────────────────────────────────────

def parse_attrs(s):
    """Return ordered dict from 'key="val" ...' string."""
    return {m.group(1): m.group(2) for m in re.finditer(r'([\w-]+)="([^"]*)"', s)}

def build_attrs(d):
    return ' '.join(f'{k}="{v}"' for k, v in d.items() if v is not None)

def merge_class(existing, new_cls):
    if not existing:
        return new_cls
    parts = existing.split()
    if new_cls not in parts:
        parts.append(new_cls)
    return ' '.join(parts)

def get_alignment(attrs):
    """Map verticalalign + horizontalalign to a single alignment value."""
    v_map = {'top': 'top', 'center': 'center', 'bottom': 'bottom'}
    h_map = {'left': 'left', 'center': 'center', 'right': 'right'}
    has_v = 'verticalalign' in attrs
    has_h = 'horizontalalign' in attrs
    if not has_v and not has_h:
        return None
    v = v_map.get(attrs.get('verticalalign', 'top'), 'top')
    h = h_map.get(attrs.get('horizontalalign', 'left'), 'left')
    return f'{v}-{h}'

# ── per-element converters ───────────────────────────────────────────────────

def conv_layoutgrid(attr_str):
    s = parse_attrs(attr_str)
    d = {}
    if s.get('name'):
        d['name'] = s['name']
    d['direction']  = 'row'
    d['wrap']       = 'true'
    d['width']      = 'fill'
    d['class']      = merge_class(s.get('class', ''), 'app-container-default')
    d['variant']    = 'default'
    d['gap']        = '0'
    d['columngap']  = '0'
    d['data-wm-conv'] = '1'
    return build_attrs(d)

def conv_gridrow(attr_str):
    s = parse_attrs(attr_str)
    d = {}
    if s.get('name'):
        d['name'] = s['name']
    d['direction']  = 'row'
    d['wrap']       = 'true'
    d['width']      = 'fill'
    d['class']      = merge_class(s.get('class', ''), 'app-container-default')
    d['variant']    = 'default'
    d['gap']        = '0'
    d['columngap']  = '0'
    d['data-wm-conv'] = '1'
    return build_attrs(d)

def conv_gridcolumn(attr_str):
    s = parse_attrs(attr_str)
    cw    = str(s.get('columnwidth', '12'))
    width = COLUMN_WIDTH_MAP.get(cw, 'fill')
    d = {}
    if s.get('name'):
        d['name'] = s['name']
    d['direction'] = 'column'
    d['width']     = width
    d['class']     = merge_class(s.get('class', ''), 'app-container-default')
    d['variant']   = 'default'
    d['data-wm-conv'] = '1'
    return build_attrs(d)

def conv_linearlayout(attr_str):
    s = parse_attrs(attr_str)
    d = {}
    if s.get('name'):
        d['name'] = s['name']
    d['direction'] = s.get('direction', 'column')
    d['wrap']      = 'true'
    d['width']     = 'fill'
    d['class']     = merge_class(s.get('class', ''), 'app-container-default')
    d['variant']   = 'default'
    if s.get('spacing'):
        d['gap'] = s['spacing']
    alignment = get_alignment(s)
    if alignment:
        d['alignment'] = alignment
    d['data-wm-conv'] = '1'
    return build_attrs(d)

def conv_linearlayoutitem(attr_str, parent_direction='row'):
    s = parse_attrs(attr_str)
    # Direction of item is perpendicular to its parent's layout axis:
    #   parent direction="row"    → items are side by side → item direction="column"
    #   parent direction="column" → items are stacked       → item direction="row"
    item_dir = 'column' if parent_direction == 'row' else 'row'
    flexgrow = str(s.get('flexgrow', '12'))
    width = FLEXGROW_WIDTH_MAP.get(flexgrow, 'fill')
    d = {}
    if s.get('name'):
        d['name'] = s['name']
    d['direction'] = item_dir
    d['width']     = width
    d['class']     = merge_class(s.get('class', ''), 'app-container-default')
    d['variant']   = 'default'
    if s.get('padding'):
        d['padding'] = s['padding']
    alignment = get_alignment(s)
    if alignment:
        d['alignment'] = alignment
    d['data-wm-conv'] = '1'
    return build_attrs(d)

# ── linearlayout context-aware converter (stack-based) ───────────────────────

LL_TAG = re.compile(
    r'(<wm-linearlayout\b[^>]*?>|</wm-linearlayout>'
    r'|<wm-linearlayoutitem\b[^>]*?/?>|</wm-linearlayoutitem>)',
    re.DOTALL
)

def convert_linearlayout_html(text):
    """Replace linearlayout/linearlayoutitem using a direction stack for context."""
    counts = {'linearlayout': 0, 'linearlayoutitem': 0}
    direction_stack = []   # tracks parent wm-linearlayout direction
    result = []
    pos = 0

    for m in LL_TAG.finditer(text):
        result.append(text[pos:m.start()])
        tag = m.group(0)

        if tag.startswith('<wm-linearlayout') and not tag.startswith('</'):
            am = re.match(r'<wm-linearlayout\b(.*?)>', tag, re.DOTALL)
            attr_str = am.group(1) if am else ''
            direction = parse_attrs(attr_str).get('direction', 'column')
            direction_stack.append(direction)
            counts['linearlayout'] += 1
            result.append(f'<wm-container {conv_linearlayout(attr_str)}>')

        elif tag == '</wm-linearlayout>':
            if direction_stack:
                direction_stack.pop()
            result.append('</wm-container>')

        elif tag.startswith('<wm-linearlayoutitem') and not tag.startswith('</'):
            am = re.match(r'<wm-linearlayoutitem\b(.*?)/?>', tag, re.DOTALL)
            attr_str = am.group(1) if am else ''
            parent_dir = direction_stack[-1] if direction_stack else 'row'
            counts['linearlayoutitem'] += 1
            result.append(f'<wm-container {conv_linearlayoutitem(attr_str, parent_dir)}>')

        elif tag == '</wm-linearlayoutitem>':
            result.append('</wm-container>')

        else:
            result.append(tag)

        pos = m.end()

    result.append(text[pos:])
    return ''.join(result), counts

# ── grid converter ───────────────────────────────────────────────────────────

def convert_grid_html(text):
    counts = {'layoutgrid': 0, 'gridrow': 0, 'gridcolumn': 0}

    def _sub_layoutgrid(m):
        counts['layoutgrid'] += 1
        return f'<wm-container {conv_layoutgrid(m.group(1))}>'

    def _sub_gridrow(m):
        counts['gridrow'] += 1
        return f'<wm-container {conv_gridrow(m.group(1))}>'

    def _sub_gridcolumn(m):
        counts['gridcolumn'] += 1
        return f'<wm-container {conv_gridcolumn(m.group(1))}>'

    text = re.sub(r'<wm-layoutgrid\b([^>]*)>',  _sub_layoutgrid, text)
    text = re.sub(r'<wm-gridrow\b([^>]*)>',      _sub_gridrow,    text)
    text = re.sub(r'<wm-gridcolumn\b([^>]*)>',   _sub_gridcolumn, text)

    text = text.replace('</wm-layoutgrid>', '</wm-container>')
    text = text.replace('</wm-gridrow>',    '</wm-container>')
    text = text.replace('</wm-gridcolumn>', '</wm-container>')

    return text, counts

# ── collapse redundant single-child containers ────────────────────────────────
# After grid/linearlayout conversion, adjacent same-direction containers or a
# row wrapping a single fill-column are structural no-ops. Remove the outer
# wrapper; keep the inner container exactly as-is (name + attrs preserved).
#
# GUARD: collapse only applies to containers produced by this conversion. Every
# converter function stamps data-wm-conv="1" on the element it creates. Only
# containers that carry this marker are eligible to collapse. Pre-existing
# wm-container elements (no marker) are never touched — they may carry JS
# references, show/hide bindings, or event handlers. The marker is stripped
# from all output after collapsing completes (see convert_html).
#
# Collapse when outer has exactly ONE wm-container child and no other content:
#   - outer.direction == inner.direction  (same axis, outer adds nothing), OR
#   - outer.direction == "row" AND inner.direction == "column" AND inner.width == "fill"
# Repeats until stable (handles stacked redundancies).

def _in_comment(text, pos):
    lo = text.rfind('<!--', 0, pos)
    lc = text.rfind('-->',  0, pos)
    return lo != -1 and (lc == -1 or lc < lo)

def _container_close(text, from_pos):
    """(start, end) of </wm-container> that matches the open whose body starts at from_pos."""
    depth, pos, CL = 0, from_pos, len('</wm-container>')
    while pos < len(text):
        o = text.find('<wm-container', pos)
        c = text.find('</wm-container>', pos)
        if c == -1: return -1, -1
        if o != -1 and o < c:
            tm = re.match(r'<wm-container\b[^>]*?(/?)>', text[o:], re.DOTALL)
            if tm and tm.group(1) != '/': depth += 1
            pos = o + (tm.end() if tm else 14)
        else:
            if depth == 0: return c, c + CL
            depth -= 1; pos = c + CL
    return -1, -1

def _should_collapse(p, c):
    if p.get('width', 'fill') != 'fill': return False  # outer has a width constraint; collapsing would lose it
    pd, cd, cw = p.get('direction','column'), c.get('direction','column'), c.get('width','fill')
    return pd == cd or (pd == 'row' and cd == 'column' and cw == 'fill')

def _collapse_pass(text):
    for m in re.finditer(r'<wm-container\b([^>]*)>', text, re.DOTALL):
        if _in_comment(text, m.start()): continue
        pa = parse_attrs(m.group(1))
        if 'data-wm-conv' not in pa: continue  # only collapse containers created by this conversion
        ps, pe = m.start(), m.end()
        pcs, pce = _container_close(text, pe)
        if pcs == -1: continue
        inner = text[pe:pcs]
        core  = re.sub(r'<!--.*?-->', '', inner, flags=re.DOTALL).strip()
        if not (core.startswith('<wm-container') and core.endswith('</wm-container>')): continue
        im = re.match(r'<wm-container\b([^>]*)>', core, re.DOTALL)
        if not im: continue
        ca = parse_attrs(im.group(1))
        if 'data-wm-conv' not in ca: continue  # only collapse if inner was also converted
        ics, ice = _container_close(core, im.end())
        if ics == -1 or ice != len(core) or core[ice:].strip(): continue
        if not _should_collapse(pa, ca): continue
        parent_name = pa.get('name')
        if parent_name:
            inner = re.sub(
                r'<wm-container\b[^>]*>',
                lambda m: (re.sub(r'\bname="[^"]*"', f'name="{parent_name}"', m.group(0), count=1)
                           if 'name="' in m.group(0)
                           else m.group(0).replace('<wm-container ', f'<wm-container name="{parent_name}" ', 1)),
                inner, count=1
            )
        return text[:ps] + inner.strip() + text[pce:], True
    return text, False

def collapse_containers(text):
    n = 0
    changed = True
    while changed:
        text, changed = _collapse_pass(text)
        if changed: n += 1
    return text, n

# ── per-file conversion ──────────────────────────────────────────────────────

def convert_html(text):
    # linearlayout first (stack-aware), then grid, then collapse redundant wrappers
    text, ll_counts   = convert_linearlayout_html(text)
    text, grid_counts = convert_grid_html(text)
    text, collapsed   = collapse_containers(text)
    text = re.sub(r' data-wm-conv="1"', '', text)  # strip conversion marker used by collapse guard
    return text, {**grid_counts, **ll_counts, 'collapsed': collapsed}

# ── main ─────────────────────────────────────────────────────────────────────

pages_dir  = Path(PROJECT_DIR) / 'src/main/webapp/pages'
html_files = sorted(pages_dir.glob('**/*.html'))

if PAGE_FILTER:
    html_files = [f for f in html_files if f.parent.name in PAGE_FILTER]

summary = []
totals  = {'layoutgrid': 0, 'gridrow': 0, 'gridcolumn': 0,
           'linearlayout': 0, 'linearlayoutitem': 0, 'collapsed': 0}

for html_path in html_files:
    original = html_path.read_text(encoding='utf-8')
    if 'wm-layoutgrid' not in original and 'wm-linearlayout' not in original:
        continue

    converted, counts = convert_html(original)
    for k in totals:
        totals[k] += counts.get(k, 0)

    page_name = html_path.parent.name
    summary.append({'page': page_name, 'file': str(html_path), 'changes': counts})

    if not DRY_RUN:
        html_path.write_text(converted, encoding='utf-8')

        if ADD_RESPONSIVE:
            css_path = html_path.parent / f'{page_name}.css'
            existing = css_path.read_text(encoding='utf-8') if css_path.exists() else ''
            if 'Grid-to-container responsive' not in existing:
                css_path.write_text(existing + RESPONSIVE_CSS, encoding='utf-8')

print(json.dumps({'summary': summary, 'totals': totals, 'dry_run': DRY_RUN}))
```

Run with:
```bash
python3 "<PROJECT_DIR>/wm_grid_conv_tmp.py" "<PROJECT_DIR>" [--dry-run] [--responsive] [--pages "Page1,Page2"]
```

After capturing the JSON output, delete the temp script:
```bash
rm -f "<PROJECT_DIR>/wm_grid_conv_tmp.py"
```

---

### STEP 4 · Print conversion summary

```
Grid & LinearLayout → Container Conversion — [DRY RUN: no files written | COMPLETE]

Project: <PROJECT_DIR>

Pages converted:
  ✓ Main          — 1 layoutgrid, 2 gridrow, 4 gridcolumn, 0 linearlayout, 0 linearlayoutitem, 3 collapsed
  ✓ Landing       — 0 layoutgrid, 0 gridrow, 0 gridcolumn, 2 linearlayout, 5 linearlayoutitem, 2 collapsed
  ...

Totals: N layoutgrid(s), N gridrow(s), N gridcolumn(s),
        N linearlayout(s), N linearlayoutitem(s), N collapsed across N page(s)

Collapse rule: only containers produced by this conversion are eligible (pre-existing wm-container
elements are never collapsed). A converted wm-container wrapping exactly ONE converted child is
removed when they share the same direction, or when a direction="row" wraps a single
direction="column" width="fill" child. The inner container stays in place with all attributes intact.

Width mappings applied:
  Source            attribute         width applied
  ──────────────────────────────────────────────────
  gridcolumn        columnwidth=6     50%
  linearlayoutitem  flexgrow=6        50%
  ...

Responsive layout:
  ✓ wrap="true" on all row containers — columns wrap naturally on narrow viewports
  [--responsive: mobile breakpoint CSS appended to each page's .css file]

Next steps:
  1. Open the project in WaveMaker Studio and preview each converted page
  2. Adjust gap / padding / alignment on wm-containers if needed
  3. Use Studio's flex properties panel to fine-tune individual containers
  4. For custom breakpoints, edit the page .css file or re-run with --responsive
```

If any page had zero changes after filtering, list it under *"Pages skipped (no layout markup found)."*

---

## Conversion rules reference

### `wm-layoutgrid` → outer flex row container

```html
<!-- BEFORE -->
<wm-layoutgrid name="layoutgrid1" class="custom">
  ...
</wm-layoutgrid>

<!-- AFTER -->
<wm-container name="layoutgrid1" direction="row" wrap="true" width="fill"
    class="app-container-default custom" variant="default" gap="0" columngap="0">
  ...
</wm-container>
```

Attribute rules:
- `name` → kept as-is (preserves JS/CSS references)
- `class` → merged; `app-container-default` appended if not already present
- All layoutgrid-specific attributes → discarded
- Fixed additions: `direction="row"` `wrap="true"` `width="fill"` `variant="default"` `gap="0"` `columngap="0"`

---

### `wm-gridrow` → flex row container

Same rules as `wm-layoutgrid` above.

---

### `wm-gridcolumn` → flex column container

```html
<!-- BEFORE -->
<wm-gridcolumn columnwidth="6" name="gridcolumn1">...</wm-gridcolumn>

<!-- AFTER -->
<wm-container name="gridcolumn1" direction="column" width="50%"
    class="app-container-default" variant="default">...</wm-container>
```

- `columnwidth` → `width` via the Bootstrap 12-col table below; attribute is removed
- Fixed: `direction="column"` `variant="default"`

---

### `wm-linearlayout` → flex row/column container

Direction is taken directly from the `direction` attribute of the source element.

```html
<!-- BEFORE -->
<wm-linearlayout direction="row" spacing="12" name="linearlayout1" verticalalign="center">
  ...
</wm-linearlayout>

<!-- AFTER -->
<wm-container name="linearlayout1" direction="row" wrap="true" width="fill"
    class="app-container-default" variant="default" gap="12" alignment="center-left">
  ...
</wm-container>
```

Attribute mapping:

| Source attribute | Target attribute | Notes |
|---|---|---|
| `name` | `name` | kept as-is |
| `direction` | `direction` | copied; default `"column"` if absent |
| `spacing` | `gap` | copied; omitted if absent |
| `verticalalign` + `horizontalalign` | `alignment` | `"{v}-{h}"` — see table below; omitted if neither present |
| `class` | `class` | merged with `app-container-default` |
| — | `wrap="true"` | always added |
| — | `width="fill"` | always added |
| — | `variant="default"` | always added |

**Alignment mapping** (`verticalalign` → vertical part, `horizontalalign` → horizontal part):

| `verticalalign` | vertical | `horizontalalign` | horizontal |
|:---:|:---:|:---:|:---:|
| `top` | `top` | `left` | `left` |
| `center` | `center` | `center` | `center` |
| `bottom` | `bottom` | `right` | `right` |
| _(absent)_ | `top` | _(absent)_ | `left` |

Result format: `alignment="{vertical}-{horizontal}"`. If neither `verticalalign` nor `horizontalalign` is present, the `alignment` attribute is omitted entirely.

---

### `wm-linearlayoutitem` → flex item container

**Direction rule:** the item's `direction` is perpendicular to its parent linearlayout's `direction`:
- parent `direction="row"` → item `direction="column"` (items stack their children vertically inside a horizontal row)
- parent `direction="column"` → item `direction="row"` (items stack their children horizontally inside a vertical column)

```html
<!-- BEFORE — inside a direction="row" linearlayout -->
<wm-linearlayoutitem name="linearlayoutitem1" flexgrow="6" padding="unset unset 160px unset">
  <wm-container name="container7">...</wm-container>
</wm-linearlayoutitem>

<!-- AFTER -->
<wm-container name="linearlayoutitem1" direction="column" width="50%"
    class="app-container-default" variant="default" padding="unset unset 160px unset">
  <wm-container name="container7">...</wm-container>
</wm-container>
```

Attribute mapping:

| Source attribute | Target attribute | Notes |
|---|---|---|
| `name` | `name` | kept as-is |
| — | `direction` | perpendicular to parent (see rule above) |
| `flexgrow` | `width` | converted via flexgrow table below |
| `padding` | `padding` | kept as-is if present |
| `verticalalign` + `horizontalalign` | `alignment` | same rule as linearlayout |
| `class` | `class` | merged with `app-container-default` |
| — | `variant="default"` | always added |

---

## Column / flexgrow width mapping

Both `columnwidth` (gridcolumn) and `flexgrow` (linearlayoutitem) use the same 12-unit scale:

| value | `width` | | value | `width` |
|:---:|:---:|---|:---:|:---:|
| 1  | 8.33%  | | 7  | 58.33% |
| 2  | 16.67% | | 8  | 66.67% |
| 3  | 25%    | | 9  | 75%    |
| 4  | 33.33% | | 10 | 83.33% |
| 5  | 41.67% | | 11 | 91.67% |
| 6  | 50%    | | 12 | fill   |

When `columnwidth` or `flexgrow` is absent → defaults to `fill`.

---

## Full example — linearlayout conversion

Input:
```html
<wm-linearlayout direction="row" spacing="12" name="linearlayout1" verticalalign="center">
    <wm-linearlayoutitem name="linearlayoutitem1" flexgrow="6" padding="unset unset 160px unset">
        <wm-container name="container7" horizontalalign="left">
            <wm-label name="label1" class="h1" caption="One-Stop Shop for All"></wm-label>
            <wm-label class="h1" caption="Your Mobile Needs" name="label5"></wm-label>
        </wm-container>
    </wm-linearlayoutitem>
    <wm-linearlayoutitem flexgrow="6" name="linearlayoutitem2">
        <wm-container name="container8" horizontalalign="center">
            <wm-picture name="picture1" picturesource="resources/images/landing.png"></wm-picture>
        </wm-container>
    </wm-linearlayoutitem>
</wm-linearlayout>
```

Output:
```html
<wm-container name="linearlayout1" direction="row" wrap="true" width="fill"
    class="app-container-default" variant="default" gap="12" alignment="center-left">
    <wm-container name="linearlayoutitem1" direction="column" width="50%"
        class="app-container-default" variant="default" padding="unset unset 160px unset">
        <wm-container name="container7" horizontalalign="left">
            <wm-label name="label1" class="h1" caption="One-Stop Shop for All"></wm-label>
            <wm-label class="h1" caption="Your Mobile Needs" name="label5"></wm-label>
        </wm-container>
    </wm-container>
    <wm-container name="linearlayoutitem2" direction="column" width="50%"
        class="app-container-default" variant="default">
        <wm-container name="container8" horizontalalign="center">
            <wm-picture name="picture1" picturesource="resources/images/landing.png"></wm-picture>
        </wm-container>
    </wm-container>
</wm-container>
```

---

## Responsive behavior explained

**Built-in (always active):**
- Row containers have `wrap="true"` — when the viewport is too narrow to fit all
  columns at their percentage widths, they automatically wrap to the next line.
- Percentage widths keep column proportions relative to the parent.

**`--responsive` flag (opt-in):**
Appends this media query to each modified page's `.css` file, stacking all
converted columns to full width on screens ≤ 767 px (mobile portrait):

```css
/* Grid-to-container responsive: stack converted columns on mobile */
@media (max-width: 767px) {
    .app-container-default[direction="column"] {
        width: 100% !important;
        min-width: unset !important;
    }
}
```

**Example walkthrough:**

Input:
```html
<wm-layoutgrid name="layoutgrid1">
    <wm-gridrow name="gridrow1">
        <wm-gridcolumn columnwidth="6" name="gridcolumn1">
            <wm-button caption="Button" name="button1"></wm-button>
        </wm-gridcolumn>
        <wm-gridcolumn columnwidth="6" name="gridcolumn2"></wm-gridcolumn>
    </wm-gridrow>
</wm-layoutgrid>
```

Output:
```html
<wm-container name="layoutgrid1" direction="row" wrap="true" width="fill"
    class="app-container-default" variant="default" gap="0" columngap="0">
    <wm-container name="gridrow1" direction="row" wrap="true" width="fill"
        class="app-container-default" variant="default" gap="0" columngap="0">
        <wm-container name="gridcolumn1" direction="column" width="50%"
            class="app-container-default" variant="default">
            <wm-button caption="Button" name="button1"></wm-button>
        </wm-container>
        <wm-container name="gridcolumn2" direction="column" width="50%"
            class="app-container-default" variant="default"></wm-container>
    </wm-container>
</wm-container>
```

On desktop: two equal-width flex columns side by side.
On mobile (with `--responsive`): each column stacks to 100% width vertically.
