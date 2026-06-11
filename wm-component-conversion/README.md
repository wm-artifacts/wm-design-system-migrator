# WaveMaker Component Conversion & Layout Modernisation

> Automated skill for converting legacy WaveMaker layout components (`wm-layoutgrid`, `wm-linearlayout`) to modern Design System `wm-container` flex layout — deterministic, modular, and Studio-compatible.

---

## What This Does

Takes WaveMaker page layouts using legacy grid and linear layout components and automatically rewrites them to use modern `wm-container` flex-based layouts. Converts every page in a project, preserving all attributes, data bindings, and nested content. Output is modular and ready for Design System.

**No manual editing required.** Optional dry-run mode for preview before committing changes.

---

## Installation

Install the skill globally via npm for it to be available to AI Agents:

```bash
npx skills add wm-artifacts/wm-design-system-migrator
```

The `/wm-component-conversion` skill is included with the main migration package.

---

## Prerequisites

- AI agent which supports SKILL
- WaveMaker project folder or `.zip` file
- (Optional) Python 3.9+ for advanced transformation analysis

---

## Available Modes

| Mode | Purpose | Output |
|---|---|---|
| **Default** | Convert layouts in-place | Rewritten page HTML files |
| **Dry-run** | Preview changes without writing | Console report of all planned conversions |
| **Responsive** | Inject mobile-responsive CSS rules | Page CSS files with `@media (max-width: 767px)` stacking |

---

## What Gets Converted

### Feature Status Summary

| Feature | Web Projects | Mobile Projects |
|---|---|---|
| **Layout Conversion** | Available | Available |
| **Component Enhancements** | Available (opt-in) | Planned |
| **Responsive CSS Injection** | Available | Available |
| **Dry-Run Mode** | Available | Available |

---

### Optional: Component Attribute & Variant Enhancements

> **This is an opt-in feature available for WEB projects.** You will be prompted: *"Would you also like to apply component attribute and variant enhancements?"*
>
> When enabled, the skill converts WaveMaker component attributes and adds Design System variant mappings for consistency. **Web projects are fully supported. Mobile projects will receive component enhancements in a future release.**

#### Web Projects — Component Enhancements Available

**Forms & Data Input**
- `<wm-form>` — adds `class="app-form"`, `variant="default"`
- `<wm-form-field>` — adds `class="form-group"`, `variant="default"`
- `<wm-text>` — adds `class="form-control"`, `variant="filled:default"`
- `<wm-checkbox>` — adds `class="form-check-input"`, `variant="filled:default"`
- `<wm-radio>` — adds `class="form-check-input"`, `variant="filled:default"`
- `<wm-select>` — adds `class="form-control"`, `variant="filled:default"`
- `<wm-fileupload>` — adds `class="form-control-file"`, `variant="default"`

**Data Display**
- `<wm-list>` — adds `class="app-list"`, `variant="default"`
- `<wm-listtemplate>` — adds `class="list-item"`, `variant="default"`
- `<wm-table>` — adds `class="app-table table"`, `variant="default"`
- `<wm-table-column>` — preserves column attributes, adds `variant="default:column"`

**UI Elements (Buttons, Labels, Icons)**
- `<wm-button>` — maps Bootstrap classes to variants:
  - `btn-default` → `variant="filled:default"` + `class="btn-filled"`
  - `btn-primary` → `variant="filled:primary"` + `class="btn-filled"`
  - `btn-success` → `variant="filled:success"` + `class="btn-filled"`
  - `btn-danger` → `variant="filled:danger"` + `class="btn-filled"`

- `<wm-label>` — maps typography classes to variants:
  - `class="p"` → `variant="default:p"`
  - `class="h1"` → `variant="default:h1"`
  - `class="h2"` → `variant="default:h2"` (through `h6`)

- `<wm-icon>` — adds size and variant:
  - Adds `class="fa-xs"` + `variant="default:xs"`
  - Or maps existing size classes to variants

- `<wm-picture>` — maps shape attributes to classes:
  - `shape="rounded"` → `class="img-rounded"` + `variant="default:rounded"` + `resizemode="cover"`
  - `shape="circle"` → `class="img-circle"` + `variant="default:circle"` + `resizemode="cover"`
  - `shape="thumbnail"` → `class="img-thumbnail"` + `variant="default:standard"` + `resizemode="cover"`

**Messages & Progress**
- `<wm-message>` — type-based styling:
  - Defaults `type="success"` if missing
  - Adds `class="app-message alert-{type}"` + `variant="filled:{type}"`
  
- `<wm-progress-bar>` — adds `class="app-progress progress-bar-default"` + `variant="filled:default"`

- `<wm-progress-circle>` — adds `class="app-progress circle progress-circle-default"` + `variant="filled:default"`

**Dialogs & Navigation**
- `<wm-dialog>` — adds `class="app-dialog"`, `variant="default"`
- `<wm-nav>` — adds `class="app-nav navbar"`, `variant="default"`
- `<wm-nav-item>` — adds `class="nav-item"`, `variant="default"`
- `<wm-accordion>` — adds `class="app-accordion"`, `variant="default"`
- `<wm-accordion-pane>` — adds `class="accordion-item"`, `variant="default"`

**Containers (Design System)**
- `<wm-container>` — adds/merges `class="app-container-{variant}"` based on `variant` attribute
  - `variant="default"` → `class="app-container-default"`
  - `variant="filled"` → `class="app-container-filled"`
  - `variant="outlined"` → `class="app-container-outlined"`

#### Mobile Project Component Changes

> **Status:** Mobile component attribute enhancements are currently **under development** and will be available in a future release.
>
> Currently, layout conversions (`wm-layoutgrid`, `wm-linearlayout` → `wm-container`) are fully supported for mobile projects. Component attribute enhancements (class/variant mappings) for mobile are planned.

**Planned Mobile Component Transformations:**

When enabled, the following component enhancements are intended for mobile projects (Bootstrap-derived patterns replaced with native mobile UX):

**Forms & Input (Mobile — Planned)**
- `<wm-text>` → Will add `class="mobile-input"`, `variant="mobile:default"`, optimized mobile keyboard
- `<wm-select>` → Will add `class="mobile-select"`, `variant="mobile:default"`, uses native OS picker
- `<wm-checkbox>` → Will add `class="mobile-checkbox"`, `variant="mobile:default"`
- `<wm-radio>` → Will add `class="mobile-radio"`, `variant="mobile:default"`

**Data Display (Mobile — Planned)**
- `<wm-list>` → Will add `class="mobile-list"`, `variant="mobile:default"`, optimized spacing for touch
- `<wm-table>` → Will add `class="mobile-table"`, `variant="mobile:responsive"`, stacks on narrow screens
- `<wm-listtemplate>` → Will add `class="mobile-list-item"`, `variant="mobile:default"`

**UI Elements (Mobile — Planned)**
- `<wm-button>` → Mobile-optimized touch targets:
  - Will add `class="mobile-button"`, `variant="mobile:{context}"`
  - Enforced minimum touch area: 44px × 44px
  
- `<wm-icon>` → Mobile sizing:
  - Will add `class="mobile-icon"`, `variant="mobile:sm|md|lg"`
  - Defaults to `mobile:md` (24px) for touch usability

- `<wm-label>` → Mobile typography:
  - Will add `class="mobile-text"`, `variant="mobile:body|caption|headline"`
  - Sizes optimized for mobile readability (≥16px base)

- `<wm-picture>` → Mobile image handling:
  - Will add `class="mobile-image"`, `variant="mobile:default"`
  - Enforces `resizemode="cover"` for consistent aspect ratio
  - Responsive width: `width="100%"`

**Dialogs & Navigation (Mobile — Planned)**
- `<wm-dialog>` → Mobile modal behavior:
  - Will add `class="mobile-dialog"`, `variant="mobile:sheet|modal"`
  - Sheet style (slide-up from bottom) by default
  
- `<wm-nav>` → Mobile navigation:
  - Will add `class="mobile-nav"`, `variant="mobile:bottom|side"`
  - Bottom navigation pattern default for mobile

**Containers (Mobile — Planned)**
- `<wm-container>` → Mobile-optimized flex layout:
  - Will add `class="mobile-container"`, `variant="mobile:default"`
  - Safe area padding for notch/status bar
  - Responsive direction with `wrap="true"` enforced

---

### 1. **Grid Layout Components** → `wm-container`

#### `wm-layoutgrid` (parent grid)
```html
<!-- NDS (BEFORE) -->
<wm-layoutgrid columns="3" gap="16px" name="grid1">
  <wm-gridrow>
    <wm-gridcolumn columnwidth="4">
      <wm-button></wm-button>
    </wm-gridcolumn>
  </wm-gridrow>
</wm-layoutgrid>

<!-- DS (AFTER) -->
<wm-container direction="row" wrap="true" width="fill" gap="16px" columngap="16px" name="grid1">
  <wm-container direction="row" wrap="true" width="fill" gap="16px" columngap="16px">
    <wm-container direction="column" width="33.33%" name="gridcolumn1">
      <wm-button></wm-button>
    </wm-container>
  </wm-container>
</wm-container>
```

**Transformations:**
- `<wm-layoutgrid columns="N">` → `<wm-container direction="row" wrap="true" columngap="<gap>">` 
- Flex attributes: `direction="row"`, `wrap="true"`, `width="fill"`
- Gap mapping: `gap` attribute → `gap` + `columngap` (same value)

#### `wm-gridrow` (row wrapper)
```html
<!-- NDS -->
<wm-gridrow>
  <wm-gridcolumn></wm-gridcolumn>
</wm-gridrow>

<!-- DS -->
<wm-container direction="row" wrap="true" width="fill" gap="0" columngap="0">
  <wm-container direction="column" width="...">
    <!-- content -->
  </wm-container>
</wm-container>
```

**Transformations:**
- `<wm-gridrow>` → `<wm-container direction="row" wrap="true" width="fill">`
- Inherits gap from parent grid (or defaults to `0`)

#### `wm-gridcolumn` (column cell)
```html
<!-- NDS -->
<wm-gridcolumn columnwidth="6">
  <wm-text></wm-text>
</wm-gridcolumn>

<!-- DS -->
<wm-container direction="column" width="50%">
  <wm-text></wm-text>
</wm-container>
```

**Transformations:**
- `<wm-gridcolumn columnwidth="N">` → `<wm-container direction="column" width="<(N/12)*100>%">`
- Width calculation: 12-column grid → percentage
  - `columnwidth="6"` → `width="50%"`
  - `columnwidth="4"` → `width="33.33%"`
  - `columnwidth="12"` → `width="100%"`
- Preserves all child elements and content

---

### 2. **Linear Layout Components** → `wm-container`

#### `wm-linearlayout` (parent)
```html
<!-- NDS (BEFORE) -->
<wm-linearlayout direction="row" spacing="8px" verticalalign="top" horizontalalign="start" name="layout1">
  <wm-linearlayoutitem flexgrow="1"></wm-linearlayoutitem>
</wm-linearlayout>

<!-- DS (AFTER) -->
<wm-container direction="row" gap="8px" alignment="top-start" name="layout1">
  <wm-container direction="column" flexgrow="1"></wm-container>
</wm-container>
```

**Transformations:**
- `<wm-linearlayout>` → `<wm-container>`
- Attribute mappings:
  | Old | New | Example |
  |---|---|---|
  | `direction="row\|column"` | `direction="row\|column"` | Same |
  | `spacing="Npx"` | `gap="Npx"` | `spacing="8px"` → `gap="8px"` |
  | `verticalalign` + `horizontalalign` | `alignment="<vertical>-<horizontal>"` | `verticalalign="top" horizontalalign="start"` → `alignment="top-start"` |

**Alignment mapping:**
- Vertical: `top`, `middle`, `bottom`
- Horizontal: `start`, `center`, `end`
- Combined: `"top-start"`, `"middle-center"`, `"bottom-end"`, etc.

#### `wm-linearlayoutitem` (child)
```html
<!-- NDS -->
<wm-linearlayoutitem flexgrow="1" padding="8px">
  <wm-container></wm-container>
</wm-linearlayoutitem>

<!-- DS -->
<wm-container direction="column" flexgrow="1" padding="8px">
  <wm-container></wm-container>
</wm-container>
```

**Transformations:**
- `<wm-linearlayoutitem>` → `<wm-container>`
- Direction is perpendicular to parent:
  - Parent `direction="row"` → item `direction="column"`
  - Parent `direction="column"` → item `direction="row"`
- Preserves `flexgrow`, `padding`, `margin`, `width`, `height`
- All child content preserved

---

### 3. **Post-Conversion Optimization**

#### Redundant Wrapper Collapse
Single-child wrapper containers are automatically removed to simplify markup:

```html
<!-- Before optimization -->
<wm-container width="100%">
  <wm-container direction="column" name="inner">
    <wm-button></wm-button>
  </wm-container>
</wm-container>

<!-- After optimization -->
<wm-container direction="column" name="inner">
  <wm-button></wm-button>
</wm-container>
```

**Rules:**
- Outer container must have only one child (a container)
- Outer container attributes are discarded (inner attributes are preserved)
- Only applied to automatically-generated wrappers (no user-named containers removed)

---

## Advanced Modes

### Dry-Run Mode
Preview all planned changes without modifying files:

```bash
/wm-component-conversion /path/to/MyApp --dry-run
```

**Output:**
- List of all pages to be converted
- Count of components found per page
- Layout structure before/after (formatted trees)
- Estimated attribute changes per file

**No files written.** Safe to run on production projects.

---

### Responsive CSS Injection
Auto-generate mobile-responsive CSS rules for converted layouts:

```bash
/wm-component-conversion /path/to/MyApp --responsive
```

**Behavior:**
- For each converted grid/layout in a page, injects a `@media (max-width: 767px)` rule
- Stacks columns to 100% width on mobile:
  ```css
  @media (max-width: 767px) {
    .wm-app .container-name {
      flex-direction: column !important;
      width: 100% !important;
    }
  }
  ```
- Preserves original desktop layout (no changes to non-media CSS)
- One rule per page CSS file (shared across all containers in that page)

---

## Component Mapping Reference

### Component Attribute Transformations

#### Web Projects

| Component | Transformation | Attributes Added |
|---|---|---|
| `wm-button` | Bootstrap class → variant | `class="btn-filled"`, `variant="filled:{context}"` |
| `wm-label` | Typography class → variant | `variant="default:{size}"` (p, h1–h6) |
| `wm-icon` | Size class → variant | `class="fa-xs"`, `variant="default:xs"` |
| `wm-picture` | Shape → class + variant | `class="img-{shape}"`, `variant="default:{shape}"`, `resizemode="cover"` |
| `wm-message` | Type-based styling | `class="app-message alert-{type}"`, `variant="filled:{type}"` |
| `wm-progress-bar` | Default styling | `class="app-progress progress-bar-default"`, `variant="filled:default"` |
| `wm-progress-circle` | Default styling | `class="app-progress circle progress-circle-default"`, `variant="filled:default"` |
| `wm-form` | Form wrapper | `class="app-form"`, `variant="default"` |
| `wm-text` | Input field | `class="form-control"`, `variant="filled:default"` |
| `wm-select` | Select dropdown | `class="form-control"`, `variant="filled:default"` |
| `wm-checkbox` | Checkbox input | `class="form-check-input"`, `variant="filled:default"` |
| `wm-radio` | Radio button | `class="form-check-input"`, `variant="filled:default"` |
| `wm-list` | Data list | `class="app-list"`, `variant="default"` |
| `wm-table` | Data table | `class="app-table table"`, `variant="default"` |
| `wm-dialog` | Modal dialog | `class="app-dialog"`, `variant="default"` |
| `wm-nav` | Navigation bar | `class="app-nav navbar"`, `variant="default"` |
| `wm-container` | DS container | `class="app-container-{variant}"` merged |

#### Mobile Projects

> **Status: Under Development** — Mobile component rules are currently being developed and will be available in a future release.

| Component | Planned Transformation | Attributes (Planned) |
|---|---|---|
| `wm-button` | Touch-optimized sizing | `class="mobile-button"`, `variant="mobile:{context}"`, enforced min 44×44px |
| `wm-label` | Mobile typography | `variant="mobile:body|caption|headline"`, ≥16px base |
| `wm-icon` | Mobile sizing | `class="mobile-icon"`, `variant="mobile:sm|md|lg"`, defaults to `md` (24px) |
| `wm-picture` | Responsive images | `class="mobile-image"`, `variant="mobile:default"`, `resizemode="cover"`, `width="100%"` |
| `wm-text` | Mobile input | `class="mobile-input"`, `variant="mobile:default"`, optimized keyboard |
| `wm-select` | Native picker | `class="mobile-select"`, `variant="mobile:default"`, uses OS picker |
| `wm-checkbox` | Mobile checkbox | `class="mobile-checkbox"`, `variant="mobile:default"` |
| `wm-radio` | Mobile radio | `class="mobile-radio"`, `variant="mobile:default"` |
| `wm-list` | Mobile list | `class="mobile-list"`, `variant="mobile:default"`, optimized spacing |
| `wm-table` | Mobile table | `class="mobile-table"`, `variant="mobile:responsive"`, responsive stacking |
| `wm-dialog` | Mobile sheet/modal | `class="mobile-dialog"`, `variant="mobile:sheet|modal"` |
| `wm-nav` | Mobile navigation | `class="mobile-nav"`, `variant="mobile:bottom|side"`, defaults to bottom nav |
| `wm-container` | Mobile container | `class="mobile-container"`, `variant="mobile:default"`, safe area padding |

---

## Execution Examples

```bash
# Convert all layouts in project folder
/wm-component-conversion /path/to/MyApp

# Convert from ZIP
/wm-component-conversion /path/to/MyApp.zip

# Preview changes first (dry run)
/wm-component-conversion /path/to/MyApp --dry-run

# Convert + add mobile responsive CSS
/wm-component-conversion /path/to/MyApp --responsive

# Convert + output to different folder
/wm-component-conversion /path/to/MyApp -o /path/to/MyApp_converted

# Verbose mode: show all parsed layout trees
/wm-component-conversion /path/to/MyApp --verbose
```

---

## Integration with Design System Migration

This skill is a **sub-component** of `/wm-design-system-migrator`. When run as part of the full pipeline:

1. **PHASE 1:** Design System project conversion (pom.xml, properties, etc.)
2. **PHASE 2:** Component layout conversion (this skill)
3. **PHASE 3:** Theme token extraction
4. **PHASE 4:** Package into ZIP

Can be run **independently** for layout modernisation on any WaveMaker project.

---

## Output Structure

### File Changes
- **Modified:** All page HTML files with converted layouts
- **Modified:** Page CSS files (if `--responsive` flag used)
- **Created:** Migration metadata (`conversion_log.json` in root)

### Metadata
Each conversion creates a `conversion_log.json`:

```json
{
  "skill": "wm-component-conversion",
  "version": "2.0.0",
  "date": "2026-06-11T10:30:00Z",
  "project_path": "/path/to/MyApp",
  "pages_processed": 8,
  "summary": {
    "layoutgrids_converted": 12,
    "gridrows_converted": 24,
    "gridcolumns_converted": 48,
    "linearlayouts_converted": 5,
    "linearlayoutitems_converted": 18,
    "containers_collapsed": 8
  },
  "pages": [
    {
      "page": "MainPage.page.xml",
      "status": "success",
      "components_converted": 5,
      "details": {
        "layoutgrids": 1,
        "gridrows": 2,
        "gridcolumns": 3,
        "linearlayouts": 0
      }
    }
  ]
}
```

---

## Current Limitations

| Gap | Why | Workaround |
|---|---|---|
| **Nested responsive values** | `columnwidth` with responsive breakpoints (e.g. `xs-12 md-6`) are converted linearly without breakpoint awareness | Manually adjust `width` with CSS media queries after conversion |
| **LESS/SCSS layout variables** | Layout-related SCSS variables in `.css` are not resolved before conversion | Ensure CSS is compiled or manually convert variable references |
| **Inline style merging** | `style="..."` attributes on grid components are preserved but may need review for flex compatibility | Check output for conflicting flex rules in inline styles |
| **Custom directives** | Non-standard WaveMaker directives in legacy layouts (custom repeat, conditionals) are preserved as-is | Test thoroughly; directive behavior may differ under flex layout |
| **Prefab internal layouts** | Prefab page layouts are not converted (prefabs are isolated components) | Convert prefabs independently or via Studio's prefab editor |

---

## Known Issues & Fixes

| Issue | Detection | Fix |
|---|---|---|
| **Redundant wrapper cascades** | Multiple single-child wrappers after grid conversion | Collapse removes outer; inner preserved with attributes |
| **Gap inheritance edge case** | `wm-gridrow` without explicit gap (inherits from parent) | Now correctly inherits or defaults to `0` |
| **Alignment enum conflicts** | Old `verticalalign="top"` vs new `alignment="top-start"` | Mapper handles all valid enum combinations |
| **Width rounding errors** | Column widths with odd 12-column divisions | Precision to 2 decimals (e.g., `33.33%` for `columnwidth="4"`) |

---

## Performance

| Metric | Typical |
|---|---|
| Pages per second | ~10–20 pages/sec (depends on layout complexity) |
| Dry-run overhead | ~5% (preview mode slightly slower due to reporting) |
| Output size | ~same as input (HTML-structure-preserving) |

---

## Testing Checklist

After conversion, verify:

- [ ] All page layouts render correctly (no visual regressions)
- [ ] Responsive breakpoints work (if `--responsive` used)
- [ ] Data bindings in form fields still resolve correctly
- [ ] Nested containers maintain proper alignment
- [ ] Custom padding/margin still applies
- [ ] Dialog and modal layouts render without overflow
- [ ] Mobile preview shows proper mobile stacking

---

## Web vs. Mobile Project Rules

### Automatic Detection

The skill detects your project type from `.wmproject.properties`:
- `type=WEB` → applies web component rules (Bootstrap, responsive breakpoints) + layout conversion
- `type=NATIVE_MOBILE` → applies layout conversion only; mobile component rules coming soon

If the project type cannot be determined, it defaults to **web** rules.

### Web Rules — Available Now

**Focus:** Bootstrap-compatible, responsive grid, flex-based layouts

**Layout Conversion (All Projects):**
- `wm-layoutgrid`, `wm-gridrow`, `wm-gridcolumn` → `wm-container` with flex properties
- `wm-linearlayout`, `wm-linearlayoutitem` → `wm-container` with proper direction/alignment

**Component Enhancements (Web Projects):**
- Button variants: `filled:default`, `filled:primary`, `filled:success`, `filled:danger`
- Table layouts with responsive column sizing
- Form controls with standard Bootstrap classes
- Form wrapper with responsive `itemsperrow` constraint
- Progress bars and circles with default styling
- Message components with type-based class and variant mapping
- Progressive enhancement: desktop-first, responsive wrapping with `wrap="true"`
- Dialogs: modal-based (centered, overlay)
- Navigation: navbar pattern

### Mobile Rules — Under Development

**Focus:** Native mobile UX, touch-friendly, safe areas

**Layout Conversion (Available Now):**
- `wm-layoutgrid`, `wm-gridrow`, `wm-gridcolumn` → `wm-container` with flex properties
- `wm-linearlayout`, `wm-linearlayoutitem` → `wm-container` with proper direction/alignment

**Component Enhancements (Planned for Future Release):**
- Touch-optimized button sizing: 44px × 44px minimum
- Lists with mobile-specific item spacing and swipe affordances
- Bottom navigation pattern (default) or side drawer
- Dialogs: sheet-style (slide-up from bottom) or modal (centered)
- Image handling: responsive widths, notch/status bar safe area padding
- Keyboard handling: optimized for mobile keyboards
- Native picker for select components
- Typography sizes optimized for mobile readability (≥16px base)

---

## Summary

The `/wm-component-conversion` skill automates the migration of WaveMaker legacy layout components to Design System `wm-container` flex layouts. It:

### Core Features (All Projects)
- **Converts grids** — `wm-layoutgrid` + `wm-gridrow` + `wm-gridcolumn` → `wm-container` with flex properties
- **Converts linear layouts** — `wm-linearlayout` + `wm-linearlayoutitem` → `wm-container` with proper direction/alignment
- **Optimises output** — removes redundant single-child wrapper containers automatically
- **Preserves all content** — data bindings, child components, attributes remain intact
- **Supports dry-run** — preview mode to verify changes before writing
- **Optional responsive CSS** — injects mobile stacking rules on demand

### Optional Component Enhancements (Web Projects Only)
- **Available:** Web project component attribute and variant enhancements (opt-in during conversion)
- **Coming Soon:** Mobile project component enhancements (planned for future release)

The conversion is deterministic, modular, and integrates seamlessly with the full Design System migration pipeline via `/wm-design-system-migrator`. **Mobile projects currently support layout conversion only; component attribute enhancements for mobile are under development.**

---

## Related Skills

- [`/wm-design-system-migrator`](README.md) — Full Design System migration pipeline (includes this skill)
- [`/wm-projectconversion`](README.md) — Project metadata conversion (pom.xml, properties)
- [`/wm-theme-to-designsystem-conversion`](README.md) — Theme token extraction and mapping
