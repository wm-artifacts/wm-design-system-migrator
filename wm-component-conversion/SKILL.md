---
name: wm-component-conversion
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
  full DesignSystem template migration (use wm-projectconversion), or when the user wants the complete
  migration pipeline in one shot (use wm-design-system-migrator instead).
metadata:
  version: 0.1.0
---

# /wm-component-conversion — WaveMaker Grid & LinearLayout → Flex Container Converter

Convert `wm-layoutgrid` / `wm-gridrow` / `wm-gridcolumn` and
`wm-linearlayout` / `wm-linearlayoutitem` markup to `wm-container`
flex-based layout in WaveMaker project page HTML files.

The generated containers use Bootstrap-mapped or flexgrow-mapped percentage widths and
`wrap="true"` so the layout is naturally responsive without extra configuration.

---

## Invocation

```
/wm-component-conversion <project_path>
/wm-component-conversion <project_path> --dry-run
/wm-component-conversion <project_path> --pages <Page1,Page2>
/wm-component-conversion <project_path> --responsive
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

### STEP 3 · Component Attribute & Variant Conversion

> **This step is opt-in — always prompt the user before executing any part of it.**

Ask the user:

*"Would you also like to apply component attribute and variant enhancements (forms, lists, buttons, labels, icons, tables)? [Y/n]"*

If the user declines, skip to STEP 4.

---

#### Determine project type

1. If `PROJECT_TYPE` was already resolved during this execution (e.g. read from `.wmproject.properties` in a prior step), reuse that value.
2. Otherwise read `<PROJECT_DIR>/.wmproject.properties` and extract the `type` line:
   - `type=WEB` → `PROJECT_TYPE=web`
   - `type=NATIVE_MOBILE` → `PROJECT_TYPE=mobile`
   - If absent or unrecognised → default to `web` and warn the user: *"Could not detect project type — defaulting to web rules."*

---

#### Load conversion rules

Resolve the rules directory based on `PROJECT_TYPE`:

- `PROJECT_TYPE=web`    → `wm-component-conversion/assets/rules/web/`
- `PROJECT_TYPE=mobile` → `wm-component-conversion/assets/rules/mobile/`

Rules are organised into category subdirectories. Each subdirectory contains exactly one `Rule.md`:

```
<RULES_DIR>/
  basic/Rule.md
  advanced/Rule.md
  charts/Rule.md
  containers/Rule.md
  data/Rule.md
  dialogs/Rule.md
  input/Rule.md
  layout/Rule.md
  navigation/Rule.md
```

**Scan every immediate subdirectory of `RULES_DIR` for a `Rule.md` and read each one in full.** Do not hardcode category names — discover them dynamically so any new category is picked up automatically.

The union of all loaded `Rule.md` files defines the complete set of transformations to apply.

If `RULES_DIR` does not exist or no `Rule.md` files are found, skip this step and warn: *"No component rules found for project type '`<PROJECT_TYPE>`'. Skipping component conversion."*

---

#### Execution

Each `Rule.md` file carries its Python implementation in a `## Script` section. The temp script that
runs the conversion is **assembled programmatically from the raw `Rule.md` files** — never by hand.

> ⚠️ **Do not hand-transcribe the rule code blocks.** Several rule regexes contain the attribute
> pattern `"[^"]*"|'[^']*'`. When that is re-typed by hand, the two `*` quantifiers can be read as a
> Markdown `*…*` emphasis span and silently dropped, turning it into the invalid `"[^"]"|'[^']'`. That
> produces `re.PatternError: missing ), unterminated subpattern` at runtime. The builder below sidesteps
> this entirely by reading each `Rule.md` verbatim from disk and slicing out the fenced blocks — the
> regexes are never re-typed.

**1 · Write the builder `<PROJECT_DIR>/wm_comp_conv_build.py`**

This is the *only* Python you author directly. It reads every `<category>/Rule.md` under `RULES_DIR`,
extracts each ```` ```python ```` block (with its `# Execution order: N`, default `99`), sorts by order,
discovers the `apply_*_rules` function names, and writes the runnable `wm_comp_conv_tmp.py`:

```python
#!/usr/bin/env python3
import re, sys
from pathlib import Path

RULES_DIR = Path(sys.argv[1])      # e.g. wm-component-conversion/assets/rules/web
OUT       = Path(sys.argv[2])      # <PROJECT_DIR>/wm_comp_conv_tmp.py

HEADER = r'''#!/usr/bin/env python3
import re, sys, json
from pathlib import Path

PROJECT_DIR = sys.argv[1]
DRY_RUN     = '--dry-run' in sys.argv
PAGE_FILTER = []
if '--pages' in sys.argv:
    idx = sys.argv.index('--pages')
    if idx + 1 < len(sys.argv):
        PAGE_FILTER = [p.strip() for p in sys.argv[idx + 1].split(',')]

def parse_attrs(s):
    return dict(re.findall(r'([\w-]+)="([^"]{0,})"', s))

def build_attrs(d):
    return ' '.join(f'{k}="{v}"' for k, v in d.items())

def merge_class(existing, *new_cls):
    parts = existing.split() if existing else []
    for c in new_cls:
        if c and c not in parts:
            parts.append(c)
    return ' '.join(parts)
'''

MAIN = r'''
RULE_FUNCS = [__RULE_FUNCS_LIST__]

pages_dir  = Path(PROJECT_DIR) / 'src/main/webapp/pages'
html_files = sorted(pages_dir.rglob('*.html'))
if PAGE_FILTER:
    html_files = [f for f in html_files if f.parent.name in PAGE_FILTER]

summary, all_counts = [], {}
for html_path in html_files:
    original = html_path.read_text(encoding='utf-8')
    text, counts = original, {}
    for rule_fn in RULE_FUNCS:
        text, c = rule_fn(text)
        for k, v in c.items():
            counts[k] = counts.get(k, 0) + v
    if sum(counts.values()) == 0:
        continue
    for k, v in counts.items():
        all_counts[k] = all_counts.get(k, 0) + v
    summary.append({'page': html_path.parent.name, 'file': str(html_path), 'changes': counts})
    if not DRY_RUN:
        html_path.write_text(text, encoding='utf-8')

print(json.dumps({'summary': summary, 'totals': all_counts, 'dry_run': DRY_RUN}))
'''

# Slice every ```python block out of each <category>/Rule.md (raw read — no re-typing).
fence = chr(96) * 3                                  # ``` without writing it literally
blocks = []                                          # (order, code)
for rule_md in sorted(RULES_DIR.glob('*/Rule.md')):
    raw = rule_md.read_text(encoding='utf-8')
    for seg in raw.split(fence + 'python')[1:]:
        code = seg.split(fence, 1)[0]
        if code.startswith('\n'):
            code = code[1:]
        m = re.search(r'# Execution order:\s*([0-9]+)', code)
        order = int(m.group(1)) if m else 99
        blocks.append((order, code))

blocks.sort(key=lambda b: b[0])
body = '\n\n'.join(code for _, code in blocks)

# apply_*_rules names in execution order, de-duplicated.
seen, funcs = set(), []
for name in re.findall(r'def (apply_\w+_rules)\(text\)', body):
    if name not in seen:
        seen.add(name)
        funcs.append(name)

script = HEADER + '\n' + body + '\n' + MAIN.replace('__RULE_FUNCS_LIST__', ', '.join(funcs))
OUT.write_text(script, encoding='utf-8')
print(f'Wrote {OUT} ({len(blocks)} blocks): {", ".join(funcs)}')
```

Notes:
- `RULES_DIR.glob('*/Rule.md')` matches exactly one `Rule.md` per immediate category subdirectory — new categories are picked up automatically.
- A single `Rule.md` with multiple `## Script` blocks contributes all of them; ordering is driven solely by `# Execution order: N`.

**2 · Build, run, and clean up**

```bash
python3 "<PROJECT_DIR>/wm_comp_conv_build.py" "<RULES_DIR>" "<PROJECT_DIR>/wm_comp_conv_tmp.py"
python3 "<PROJECT_DIR>/wm_comp_conv_tmp.py" "<PROJECT_DIR>" [--dry-run] [--pages "Page1,Page2"]
```

`<RULES_DIR>` is the project-type rules root resolved in **Load conversion rules** (`.../assets/rules/web` or `.../assets/rules/mobile`).

Parse the JSON output and store results in `COMP_COUNTS` for STEP 4.

```bash
rm -f "<PROJECT_DIR>/wm_comp_conv_tmp.py" "<PROJECT_DIR>/wm_comp_conv_build.py"
```

---

### STEP 4 · Write the conversion script and run it

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
    """Map horizontalalign to alignment value with middle as default vertical."""
    h_map = {'left': 'left', 'center': 'center', 'right': 'right'}
    if 'horizontalalign' not in attrs:
        return None
    h = h_map.get(attrs.get('horizontalalign', 'left'), 'left')
    return f'middle-{h}'

# ── grid pre-scan: count direct-child gridrows per layoutgrid ────────────────

_LG_SCAN = re.compile(
    r'(<wm-layoutgrid\b[^>]*?>|</wm-layoutgrid>'
    r'|<wm-gridrow\b[^>]*?>)',
    re.DOTALL
)

def _count_gridrows_per_layoutgrid(text):
    """Return list of direct-child wm-gridrow counts, indexed by layoutgrid opening-tag order."""
    stack   = []   # each entry: {'idx': int, 'count': int}
    results = {}   # opening-tag index → gridrow count
    lg_idx  = 0

    for m in _LG_SCAN.finditer(text):
        tag = m.group(0)
        if tag.startswith('<wm-layoutgrid') and not tag.startswith('</'):
            stack.append({'idx': lg_idx, 'count': 0})
            lg_idx += 1
        elif tag == '</wm-layoutgrid>':
            if stack:
                entry = stack.pop()
                results[entry['idx']] = entry['count']
        elif tag.startswith('<wm-gridrow'):
            if stack:
                stack[-1]['count'] += 1   # only the innermost layoutgrid gets the credit

    return [results.get(i, 1) for i in range(lg_idx)]

# ── per-element converters ───────────────────────────────────────────────────

def conv_layoutgrid(attr_str, direction='row'):
    """direction is computed by the caller from the gridrow pre-scan."""
    s = parse_attrs(attr_str)
    d = {}
    if s.get('name'):
        d['name'] = s['name']
    d['direction']  = direction
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
    d['direction'] = 'row'
    d['wrap']      = 'true'
    d['width']     = width
    d['class']     = merge_class(s.get('class', ''), 'app-container-default')
    d['variant']   = 'default'
    alignment = get_alignment(s)
    if alignment:
        d['alignment'] = alignment
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

    # Pre-scan: determine direction for each layoutgrid before replacing tags.
    # direction = "column" when the layoutgrid has >1 direct-child gridrows
    # (rows must stack vertically); "row" when there is only 1.
    gridrow_counts = _count_gridrows_per_layoutgrid(text)
    lg_idx = [0]

    def _sub_layoutgrid(m):
        i = lg_idx[0]; lg_idx[0] += 1
        counts['layoutgrid'] += 1
        nr        = gridrow_counts[i] if i < len(gridrow_counts) else 1
        direction = 'column' if nr > 1 else 'row'
        return f'<wm-container {conv_layoutgrid(m.group(1), direction)}>'

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



### STEP 5 · Generate the importable ZIP (standalone only)

> **Skip this step when invoked from `wm-design-system-migrator`** — the parent orchestrator
> handles Packaging in its own PHASE 4. Only execute when running `wm-component-conversion`
> directly.

If `DRY_RUN` is `true`, skip this step entirely (dry-run never writes files or produces a ZIP).

Let:
- `PARENT_DIR` = directory containing `PROJECT_DIR`
- `FOLDER_BASENAME` = basename of `PROJECT_DIR`
- `ZIP_NAME` = `<FOLDER_BASENAME>_conv_al`
- `ZIP_PATH` = `<PARENT_DIR>/<ZIP_NAME>.zip`

Zip from **inside** `PROJECT_DIR` so project files land at the ZIP root with no
enclosing folder. Studio's `createNewProject` import path expects `.wmproject.properties`
at the ZIP root; a nested folder triggers a stricter validation code path that causes
import failures.

```bash
cd "<PROJECT_DIR>" \
  && rm -f "../<ZIP_NAME>.zip" \
  && zip -rq "../<ZIP_NAME>.zip" . -x "*.DS_Store"
```

Capture `ZIP_SIZE` via `ls -lh "../<ZIP_NAME>.zip"`.

If `zip` is not available on the user's system → fall back to:
```bash
python3 -c "
import shutil, os
os.chdir('<PROJECT_DIR>')
shutil.make_archive('../<ZIP_NAME>', 'zip', '.', '.')
"
```

Pass `ZIP_PATH` and `ZIP_SIZE` into the STEP 4 summary.

---



### STEP 6 · Print conversion summary

```
Grid & LinearLayout → Container Conversion — [DRY RUN: no files written | COMPLETE]

Project: <PROJECT_DIR>
ZIP:     <ZIP_PATH>  (<ZIP_SIZE>)    ← omit this line when run from wm-design-system-migrator or when DRY_RUN

Pages converted (layout):
  ✓ Main          — 1 layoutgrid, 2 gridrow, 4 gridcolumn, 0 linearlayout, 0 linearlayoutitem, 3 collapsed
  ✓ Landing       — 0 layoutgrid, 0 gridrow, 0 gridcolumn, 2 linearlayout, 5 linearlayoutitem, 2 collapsed
  ...

Totals (layout): N layoutgrid(s), N gridrow(s), N gridcolumn(s),
                 N linearlayout(s), N linearlayoutitem(s), N collapsed across N page(s)

[Include the following block only if STEP 3c ran:]

Component Attribute & Variant Conversion — [DRY RUN: no files written | COMPLETE]
Rules applied: wm-component-conversion/assets/rules/<PROJECT_TYPE>/

Pages converted (components):
  ✓ Main    — N form_itemsperrow, N wm_list, N wm_listtemplate, N wm_table, N wm_container, N button, N label, N icon, N picture
  ...

Totals (components): [data rules] N form_itemsperrow, N wm_list, N wm_listtemplate, N wm_table, N wm_container
                     [basic rules] N button, N label, N icon, N picture  — across N page(s)

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
  1. Import <ZIP_PATH> into WaveMaker Studio    ← standalone only; omit when run from wm-design-system-migrator
     (or: Open the project in WaveMaker Studio and preview each converted page)
  2. Adjust gap / padding / alignment on wm-containers if needed
  3. Use Studio's flex properties panel to fine-tune individual containers
  4. For custom breakpoints, edit the page .css file or re-run with --responsive
```

If any page had zero changes after filtering, list it under *"Pages skipped (no layout markup found)."*

---

## Conversion rules reference

### `wm-layoutgrid` → flex container (direction determined by gridrow count)

**Direction rule (pre-scanned before any tag is replaced):**

| Direct-child `wm-gridrow` count | `direction` |
|---|---|
| 1 | `"row"` — single row, columns sit side by side |
| > 1 | `"column"` — multiple rows must stack vertically |

```html
<!-- BEFORE: single gridrow -->
<wm-layoutgrid name="layoutgrid1" class="custom">
  <wm-gridrow>...</wm-gridrow>
</wm-layoutgrid>

<!-- AFTER: direction="row" (1 gridrow) -->
<wm-container name="layoutgrid1" direction="row" wrap="true" width="fill"
    class="app-container-default custom" variant="default" gap="0" columngap="0">
  ...
</wm-container>
```

```html
<!-- BEFORE: multiple gridrows -->
<wm-layoutgrid name="layoutgrid1">
  <wm-gridrow>...</wm-gridrow>
  <wm-gridrow>...</wm-gridrow>
</wm-layoutgrid>

<!-- AFTER: direction="column" (> 1 gridrow) -->
<wm-container name="layoutgrid1" direction="column" wrap="true" width="fill"
    class="app-container-default" variant="default" gap="0" columngap="0">
  ...
</wm-container>
```

Attribute rules:
- `name` → kept as-is (preserves JS/CSS references)
- `class` → merged; `app-container-default` appended if not already present
- All layoutgrid-specific attributes → discarded
- `direction` → `"column"` if direct-child gridrow count > 1, else `"row"` (pre-scanned)
- Fixed additions: `wrap="true"` `width="fill"` `variant="default"` `gap="0"` `columngap="0"`

---

### `wm-gridrow` → flex row container

Always `direction="row"` `wrap="true"` regardless of contents.

---

### `wm-gridcolumn` → flex row container

```html
<!-- BEFORE -->
<wm-gridcolumn columnwidth="6" name="gridcolumn1" horizontalalign="center">...</wm-gridcolumn>

<!-- AFTER -->
<wm-container name="gridcolumn1" direction="row" wrap="true" width="50%"
    class="app-container-default" variant="default" alignment="middle-center">...</wm-container>
```

- `columnwidth` → `width` via the Bootstrap 12-col table below; attribute is removed
- `horizontalalign` → `alignment` as `"middle-{horizontal}"` (left/center/right); omitted if absent
- Fixed: `direction="row"` `wrap="true"` `variant="default"`

---

### `wm-linearlayout` → flex row/column container

Direction is taken directly from the `direction` attribute of the source element.

```html
<!-- BEFORE -->
<wm-linearlayout direction="row" spacing="12" name="linearlayout1" horizontalalign="center">
  ...
</wm-linearlayout>

<!-- AFTER -->
<wm-container name="linearlayout1" direction="row" wrap="true" width="fill"
    class="app-container-default" variant="default" gap="12" alignment="middle-center">
  ...
</wm-container>
```

Attribute mapping:

| Source attribute | Target attribute | Notes |
|---|---|---|
| `name` | `name` | kept as-is |
| `direction` | `direction` | copied; default `"column"` if absent |
| `spacing` | `gap` | copied; omitted if absent |
| `horizontalalign` | `alignment` | `"middle-{horizontal}"` where horizontal is left/center/right; omitted if absent |
| `class` | `class` | merged with `app-container-default` |
| — | `wrap="true"` | always added |
| — | `width="fill"` | always added |
| — | `variant="default"` | always added |

**Alignment mapping** (`horizontalalign` → horizontal part, vertical part is always `middle`):

| `horizontalalign` | alignment value |
|:---:|:---:|
| `left` | `middle-left` |
| `center` | `middle-center` |
| `right` | `middle-right` |
| _(absent)_ | _(omitted)_ |

---

### `wm-linearlayoutitem` → flex item container

**Direction rule:** the item's `direction` is perpendicular to its parent linearlayout's `direction`:
- parent `direction="row"` → item `direction="column"` (items stack their children vertically inside a horizontal row)
- parent `direction="column"` → item `direction="row"` (items stack their children horizontally inside a vertical column)

```html
<!-- BEFORE — inside a direction="row" linearlayout -->
<wm-linearlayoutitem name="linearlayoutitem1" flexgrow="6" padding="unset unset 160px unset" horizontalalign="center">
  <wm-container name="container7">...</wm-container>
</wm-linearlayoutitem>

<!-- AFTER -->
<wm-container name="linearlayoutitem1" direction="column" width="50%"
    class="app-container-default" variant="default" padding="unset unset 160px unset" alignment="middle-center">
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
| `horizontalalign` | `alignment` | `"middle-{horizontal}"` where horizontal is left/center/right; omitted if absent |
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
<wm-linearlayout direction="row" spacing="12" name="linearlayout1" horizontalalign="left">
    <wm-linearlayoutitem name="linearlayoutitem1" flexgrow="6" padding="unset unset 160px unset">
        <wm-container name="container7">
            <wm-label name="label1" class="h1" caption="One-Stop Shop for All"></wm-label>
            <wm-label class="h1" caption="Your Mobile Needs" name="label5"></wm-label>
        </wm-container>
    </wm-linearlayoutitem>
    <wm-linearlayoutitem flexgrow="6" name="linearlayoutitem2" horizontalalign="center">
        <wm-container name="container8">
            <wm-picture name="picture1" picturesource="resources/images/landing.png"></wm-picture>
        </wm-container>
    </wm-linearlayoutitem>
</wm-linearlayout>
```

Output:
```html
<wm-container name="linearlayout1" direction="row" wrap="true" width="fill"
    class="app-container-default" variant="default" gap="12" alignment="middle-left">
    <wm-container name="linearlayoutitem1" direction="column" width="50%"
        class="app-container-default" variant="default" padding="unset unset 160px unset">
        <wm-container name="container7">
            <wm-label name="label1" class="h1" caption="One-Stop Shop for All"></wm-label>
            <wm-label class="h1" caption="Your Mobile Needs" name="label5"></wm-label>
        </wm-container>
    </wm-container>
    <wm-container name="linearlayoutitem2" direction="column" width="50%"
        class="app-container-default" variant="default" alignment="middle-center">
        <wm-container name="container8">
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

**Example walkthrough — single gridrow (direction="row"):**

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

Output (1 gridrow → `direction="row"` on outer; collapse removes the redundant row wrapper):
```html
<wm-container name="gridrow1" direction="row" wrap="true" width="fill"
    class="app-container-default" variant="default" gap="0" columngap="0">
    <wm-container name="gridcolumn1" direction="row" wrap="true" width="50%"
        class="app-container-default" variant="default">
        <wm-button caption="Button" name="button1"></wm-button>
    </wm-container>
    <wm-container name="gridcolumn2" direction="row" wrap="true" width="50%"
        class="app-container-default" variant="default"></wm-container>
</wm-container>
```

> The outer layoutgrid and the single gridrow both had `direction="row"` and `width="fill"`, so the collapse pass merged them into one container (gridrow1 name is preserved).

---

**Example walkthrough — multiple gridrows (direction="column"):**

Input:
```html
<wm-layoutgrid name="layoutgrid1">
    <wm-gridrow name="gridrow1">
        <wm-gridcolumn columnwidth="6" name="gridcolumn1"></wm-gridcolumn>
        <wm-gridcolumn columnwidth="6" name="gridcolumn2"></wm-gridcolumn>
    </wm-gridrow>
    <wm-gridrow name="gridrow2">
        <wm-gridcolumn columnwidth="12" name="gridcolumn3"></wm-gridcolumn>
    </wm-gridrow>
</wm-layoutgrid>
```

Output (2 gridrows → `direction="column"` on outer; rows stack vertically):
```html
<wm-container name="layoutgrid1" direction="column" wrap="true" width="fill"
    class="app-container-default" variant="default" gap="0" columngap="0">
    <wm-container name="gridrow1" direction="row" wrap="true" width="fill"
        class="app-container-default" variant="default" gap="0" columngap="0">
        <wm-container name="gridcolumn1" direction="row" wrap="true" width="50%"
            class="app-container-default" variant="default"></wm-container>
        <wm-container name="gridcolumn2" direction="row" wrap="true" width="50%"
            class="app-container-default" variant="default"></wm-container>
    </wm-container>
    <wm-container name="gridrow2" direction="row" wrap="true" width="fill"
        class="app-container-default" variant="default" gap="0" columngap="0">
        <wm-container name="gridcolumn3" direction="row" wrap="true" width="fill"
            class="app-container-default" variant="default"></wm-container>
    </wm-container>
</wm-container>
```

> The outer container uses `direction="column"` so the two rows stack. Each gridrow uses `direction="row"` so its columns sit side by side.

On mobile (with `--responsive`): each gridcolumn stacks to 100% width vertically.
