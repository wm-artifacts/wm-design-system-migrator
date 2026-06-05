# WaveMaker DEFAULT → Design System Migration

> Automated migration skill for converting WaveMaker DEFAULT-template apps (WEB & NATIVE_MOBILE, 11.x) to Design System — deterministic, repeatable, and Studio-importable ZIP output.

---

## What This Does

Takes a WaveMaker DEFAULT project (folder or `.zip`) and automcatically rewrites every file that must change to produce a valid Design System project. No manual editing. Output is a ZIP ready to import into WaveMaker Studio.

---

## Installation

Install the skill globally via npm for it to be available to AI Agents:

```bash
npx skills add wm-artifacts/wm-design-system-migrator
```

Add specific skills or all when prompted. `/wm-design-system-migrator` is the main pipeline — the other skills must be present for it to work.

---

## Prerequisites

- AI agent which supports SKILL.
- Python 3.9+ on PATH (used internally; macOS ships with this)

---

## Version Targets (as of this writing)

| Field | WEB | MOBILE |
|---|---|---|
| Parent POM | `1.0.0-20260513150623` | `1.0.0-20260513150623` |
| Runtime UI | `1.0.0-next.27577` | `1.0.0-next.27601` |
| Studio upgrade | `1115.07` | `1115.08` |

The skill prompts for these at runtime and lets you override them — paste values from a known-good Design System reference project or accept the recommended defaults.

---

## Available Skills

| Command | Purpose |
|---|---|
| `/wm-design-system-migrator` | **Full Design System migration** — pom.xml, properties, index.html, variables, layouts, theme tokens, NPM scope, migration history. Produces a ZIP. |
| `/wm-component-conversion` | **Layout modernisation** — converts grid (`wm-layoutgrid / wm-gridrow / wm-gridcolumn`) and linear (`wm-linearlayout / wm-linearlayoutitem`) markup to `wm-container` flex layout. Dry-run mode available. |
| `/wm-projectconversion` | **Project conversion** — pom.xml, properties, index.html, variables, NPM scope, migration history. Produces a ZIP. |
| `/wm-theme-to-designsystem-conversion` | **Theme token migration** — extracts legacy theme CSS variables and CSS property values from `style.css` into Design System global tokens (color, spacing, typography) with `--wm-*` semantic naming. Writes to `design-tokens/app.override.css`. Handles deduplication, variable reference mapping, and font family customization. |

---

## What Gets Converted

### `/wm-design-system-migrator`

| File / Area | What changes |
|---|---|
| `pom.xml` | `com.wavemaker.*` → `ai.wavemaker.*`; parent version + runtime UI version bumped to Design System values |
| `.wmproject.properties` | `template=DEFAULT` → `template=PRISM`; `studioProjectUpgradeVersion` updated; `supportedLanguages` JSON injected (XML-escaped) when `languageBundleSources=STATIC` |
| `src/main/webapp/index.html` | Legacy `wm-style.css` / `wm-responsive.css` links removed; Design System `foundation.css` + `design-tokens` block injected (WEB); mobile index left as-is |
| `*.variables.json` (app + all pages) | `wm.NotificationVariable` → `wm.NotificationAction`; same for Navigation, Login, Logout — both `"category"` field and `"_id"` values |
| Page HTML layouts (WEB) | `<wm-left-panel>` moved outside `<wm-content>`; `<wm-header>` / `<wm-footer>` moved inside; `<wm-top-nav>` removed; `navtype="rail" navheight="full"` added |
| Page HTML layouts (MOBILE) | `wm-linearlayout` / `wm-linearlayoutitem` → `wm-container`; **`wm-layoutgrid` family kept as-is** (native Design System mobile widget) |
| `themes/` directory | Deleted after token extraction; replaced by `design-tokens/app.override.css` with actual extracted design tokens from legacy theme (typography, colors, spacing). Variable references updated to use new `--wm-*` naming. Deduplication applied (no duplicate tokens). |
| `ui-build.js` | `NPM_PACKAGE_SCOPE = '@wavemaker'` → `'@wavemaker-ai'` (direct string, not caught by bulk regex) |
| All source files (`.js .ts .tsx .jsx .json .html .css .md .yml .yaml`) | `@wavemaker/` → `@wavemaker-ai/` everywhere; email addresses (`@wavemaker.com`) preserved |
| `wm_rn_config.json` (MOBILE only) | `"enableDesignTokens": true` + `"enableHermes": true` added to `preferences` |
| `migration_info.json` | Design System migration history entries (1115.03–1115.07 for WEB, +1115.08 for MOBILE) appended; existing history preserved |


### `/wm-component-conversion`

| Widget | Converts to |
|---|---|
| `wm-layoutgrid` | `wm-container direction="row" wrap="true" width="fill" gap="0" columngap="0"` |
| `wm-gridrow` | `wm-container direction="row" wrap="true" width="fill" gap="0" columngap="0"` |
| `wm-gridcolumn columnwidth="N"` | `wm-container direction="column" width="<N/12 as %>"`  (e.g. `columnwidth="6"` → `width="50%"`) |
| `wm-linearlayout` | `wm-container` inheriting `direction`, `spacing→gap`, `verticalalign+horizontalalign→alignment` |
| `wm-linearlayoutitem flexgrow="N"` | `wm-container` with perpendicular direction to parent + `flexgrow`-mapped `width` |

**Post-conversion:** redundant single-child wrapper containers are collapsed automatically (removes the outer; inner stays intact with all attributes and name).

**Optional `--responsive` flag:** injects a `@media (max-width: 767px)` rule into each page's `.css` that stacks converted columns to 100% width on mobile.

---

## Theme Token Migration (`/wm-theme-to-designsystem-conversion`)

Automatically extracts legacy theme CSS variables from `src/main/webapp/themes/<THEME_NAME>/style.css` and migrates them to Design System design tokens in `src/main/webapp/design-tokens/app.override.css`.

### What Gets Migrated

| Source | Category | Target | Example |
|---|---|---|---|
| `style.css` typography | font-family, font-size, font-weight, line-height | Design tokens | `--wm-font-family-brand`, `--wm-h1-font-size` |
| `style.css` colors | primary, secondary, accent, error, success, surface, text, border | Design tokens | `--wm-color-primary: #FF7250`, `--wm-color-error: #F44336` |
| `style.css` spacing | gap, margin, padding, space, size, radius, height, width | Design tokens | `--wm-gap-base: 8px`, `--wm-margin-base: 16px` |

### Token Extraction Features

| Feature | Behavior |
|---|---|
| **Smart dual-strategy extraction** | **Strategy 1:** Extract `:root { --var: value; }` CSS variables if present (explicit intent). **Strategy 2 (Fallback):** If no `:root` variables found, automatically scan selectors for actual CSS properties (`color:`, `font-family:`, `padding:`, etc.) and extract values. |
| **Semantic mapping to foundation naming** | Extracted values automatically mapped to Design System semantic tokens: primary colors → `--wm-color-primary`, brand font → `--wm-font-family-brand`, base gaps → `--wm-gap-base`, etc. Custom values get `--wm-` prefix. |
| **Property value extraction** | When CSS variables absent, extracts from CSS selectors: heading selectors → typography tokens (h1, h2, .heading), color definitions → color tokens, spacing properties → spacing tokens. Zero manual mapping required. |
| **Deduplication** | Duplicate token names automatically removed. First occurrence is kept; subsequent duplicates discarded with count reported. |
| **Variable reference mapping** | Token values referencing other variables updated to use mapped names: `var(--brand-primary)` → `var(--wm-color-primary)`. Ensures all references resolve correctly. |
| **Dependent variable mapping** | Color-mix and calc expressions using old token names updated to use new mapped names: `color-mix(in srgb, var(--brand-primary), ...)` → `color-mix(in srgb, var(--wm-color-primary), ...)` |
| **Font family customization** | User prompted whether to import custom font families. Google Fonts auto-detected and wrapped in `@import url()`. System fonts used directly. |
| **Dynamic foundation reference** | Installs `@wavemaker/foundation-css` npm package to access complete foundation token definitions, component styles, and global styles—ensures consistency with Design System standards and always uses latest definitions. |

### Token Categories

**Typography (25+ tokens typically):**
- Font families: `--wm-font-family-brand`, `--wm-font-family-plain`
- Font sizes: `--wm-font-size-sm`, `--wm-h1-font-size`, `--wm-h2-font-size`
- Line heights: `--wm-h1-line-height`, `--wm-h2-line-height`
- Other: font-weight, letter-spacing, text colors

**Colors (40+ tokens typically):**
- Semantic: `--wm-color-primary`, `--wm-color-secondary`, `--wm-color-success`, `--wm-color-error`, `--wm-color-warning`, `--wm-color-info`
- Surfaces & backgrounds: `--wm-color-surface`, `--wm-app-body-bg`, `--wm-header-bg`
- Borders & dividers: `--wm-border-color`, `--wm-border-radius-*`
- Component-specific: header, button, input, navigation colors

**Spacing (35+ tokens typically):**
- Gaps & margins: `--wm-gap-base`, `--wm-margin-base`, `--wm-padding-vertical-base`
- Radii: `--wm-border-radius-sm`, `--wm-border-radius-md`, `--wm-border-radius-lg`
- Dimensions: `--wm-header-height`, `--wm-left-panel-width`, `--wm-input-height`
- Layout spacing: `--wm-page-content-space`, `--wm-header-padding`

### Output Format

Writes to `src/main/webapp/design-tokens/app.override.css`:

```css
/**
 * Design Token Overrides — Migrated from <THEME_NAME> theme
 * Source: src/main/webapp/themes/<THEME_NAME>/style.css
 *
 * These tokens override foundation.css values.
 */

:root {
  /* Typography Tokens */
  --wm-font-family-brand: "Arial", sans-serif;
  --wm-font-size-base: 12px;
  
  /* Color Tokens */
  --wm-color-primary: #2294ef;
  --wm-color-error: #ff6464;
  
  /* Spacing & Layout Tokens */
  --wm-gap-base: 8px;
  --wm-border-radius-md: 8px;
}
```

### Edge Cases & Handling

| Scenario | Behavior | Outcome |
|---|---|---|
| **No `style.css` file** | STEP 1 validation fails | ✗ Abort: *"No style.css found in theme folder."* |
| **No `:root { }` variables** | **Strategy 2 (Fallback)** — Extracts actual CSS property values from selectors | ✓ Auto-creates tokens from `color:`, `font-*:`, `padding:`, `margin:`, `border-radius:` properties with `--wm-*` naming |
| **Mixed:** CSS variables + properties | Extracts both; prioritizes explicit `:root` variables | ✓ Combines all found values into unified token set with foundation naming |
| **Empty `style.css`** | No selectors or properties to parse | ✓ Creates blank `app.override.css`; user can customize via Studio Theme panel |
| **Variables outside `:root`** | Strategy 2 extracts from actual properties | ✓ Component-scoped variables ignored (only extract global properties) |
| **Duplicate token names** | First occurrence kept | ✓ Subsequent duplicates discarded; count reported in `--verbose` mode |
| **Unresolved `var()` reference** | Reference preserved in mapped token | ⚠ May fail at runtime; manually fix in `app.override.css` if needed |
| **Complex values** (`calc()`, `color-mix()`, etc.) | Extracted and preserved | ✓ Variable references within values are remapped to new names |

### Invocation

```bash
# Extract theme tokens from a project
/wm-theme-to-designsystem-conversion /path/to/MyApp Wavemaker-Ai

# Dry run (preview without writing files)
/wm-theme-to-designsystem-conversion /path/to/MyApp Wavemaker-Ai --dry-run

# Verbose output showing full token extraction and deduplication details
/wm-theme-to-designsystem-conversion /path/to/MyApp Wavemaker-Ai --verbose
```

### Foundation Package Installation

When extracting theme tokens, the skill automatically installs the `@wavemaker/foundation-css` npm package into the project's `design-tokens/` folder. This package provides:

**Foundation CSS file:**
- `foundation/foundation.css` — all `--wm-*` semantic token definitions

**Global token definitions** (JSON reference):
- `src/tokens/web/global/border.json` — border-radius, border-width, border-color tokens
- `src/tokens/web/global/color.json` — semantic color tokens (primary, secondary, error, success, warning, info)
- `src/tokens/web/global/spacing.json` — gap, margin, padding, size tokens
- `src/tokens/web/global/typography.json` — font-family, font-size, font-weight, line-height tokens

**Component styles** (optional):
- `src/tokens/web/components/` — component-specific token overrides for button, input, navigation, etc.

The installation happens transparently during token extraction (STEP 2) and ensures your design tokens are always aligned with the current Design System standards. The skill uses the global token definitions to intelligently map extracted theme values to semantic `--wm-*` token names.

### Execution in Full Pipeline

When invoked via `/wm-design-system-migrator`, theme token migration happens **AFTER** design system conversion but **BEFORE** theme folder deletion:

1. **PHASE 1** — DesignSystem conversion
2. **PHASE 3** — Theme token extraction (reads `themes/` folder)
3. **PHASE 3.5** — Delete legacy `themes/` folder (safe: tokens already extracted)
4. **PHASE 5** — Packaging with tokens in `design-tokens/app.override.css`

This execution order ensures no data loss and proper token capture before cleanup.

---

## Project Conversion Coverage

* Maven groupId rewrites — simple string replacement, no edge cases
* Version bumps — single regex per field
* `template=PRISM` + `studioProjectUpgradeVersion` — simple property substitution
* `wm.*Variable` → `wm.*Action` renames — exact string match across all JSON files
* `@wavemaker/` → `@wavemaker-ai/` bulk scope migration — trailing-slash pattern avoids false positives
* `ui-build.js` `NPM_PACKAGE_SCOPE` — direct literal replacement, handled separately from bulk regex
* `migration_info.json` — idempotent append of Design System migration history
* MOBILE: `wm_rn_config.json` preferences update
* MOBILE: `wm-layoutgrid` family is correctly left untouched (a previous known mistake has been fixed)
* `supportedLanguages` injection — reads actual `i18n/*.json` files and XML-escapes them. Handles raw `<br/>` tags that previously caused Studio import failures.
* Web page layout restructuring — regex-based conversion supporting the standard WaveMaker shell structure.
* `wm-linearlayout` conversion — direction-stack-aware parser handles nested layouts and flags multi-child wraps for review.
* `index.html` CSS block replacement — supports both `preload`/`onload` and direct `<link>` stylesheet variants.

---

### Current Limitations

| Gap | Why |
|---|---|
| **LESS-specific theme constructs** | Functions like `darken()`, `lighten()`, and `~"..."` wrappers in LESS files cannot be mechanically translated. Plain CSS variable extraction handles static values; dynamic LESS logic requires manual conversion via Studio's Theme panel. |
| **Prefab migration** | Prefab-internal files are not touched. Prefabs are platform-agnostic and run as-is under Design System; if a prefab breaks at runtime it is a prefab-build issue, not a host-project conversion issue. |
| **Java backend / services** | `services/`, JPA mappings, security config, `build.xml`, `mvnw` — template-agnostic, not touched and not needed. |
| **Highly custom page HTML** | Non-standard widget nesting, inline `<script>` blocks referencing layout elements, or pages that already have a partial Design System structure may need a visual review after conversion. |

---


## Invocation examples

```bash
# Basic project conversion (accepts folder or .zip)
/wm-design-system-migrator /path/to/MyApp

# Convert + output to a different folder
/wm-design-system-migrator /path/to/MyApp.zip -o /path/to/MyApp_DesignSystem

# Rename project during conversion
/wm-design-system-migrator /path/to/MyApp --project-name FinancePortal

# Layout modernisation only (dry run to preview)
/wm-component-conversion /path/to/MyApp --dry-run

# Full pipeline: Project conversion + Layout conversion + Packaging
/wm-design-system-migrator /path/to/MyApp

# Full pipeline, skip layout conversion
/wm-design-system-migrator /path/to/MyApp --skip-autolayout

# Full pipeline with responsive CSS injected into page stylesheets
/wm-design-system-migrator /path/to/MyApp --responsive
```

---

## Upcoming Enhancements

- Automatic generation of design-tokens/foundation/ skeleton files
- Enhanced support for highly customised page layouts Design System structures
- Smarter responsive layout optimisation during auto-layout conversion
- Additional migration safety checks

---


## Summary

This skill automates the migration of WaveMaker 11.x DEFAULT-template WEB and NATIVE_MOBILE projects to the Design System design system. It performs mechanical rewrites across project configuration, layouts, variables, themes, design tokens, and package scopes, producing a Studio-importable ZIP with minimal manual effort.

The migration includes:

* Design System-compatible dependency and project configuration updates
* Automatic variable and NPM scope migration
* WEB and MOBILE layout transformations
* **Automated theme-to-design-token migration** — extracts legacy CSS variables with semantic mapping, deduplication, and variable reference updates to produce `design-tokens/app.override.css` with production-ready tokens
* Optional layout modernisation using `wm-container`
* Migration history updates and compatibility fixes

The skill is designed to handle the majority of standard WaveMaker projects deterministically, while also guarding against common migration failures discovered during testing.

### Theme Token Migration Details

The `/wm-theme-to-designsystem-conversion` skill provides:

* **Automatic extraction** — reads `src/main/webapp/themes/<THEME_NAME>/style.css` and extracts typography, color, and spacing tokens
* **Semantic mapping** — maps legacy token names to Design System semantic tokens (`--my-primary` → `--wm-color-primary`)
* **Deduplication** — removes duplicate token definitions, keeping first occurrence
* **Variable reference resolution** — updates dependent tokens to reference mapped names (`var(--brand-primary)` → `var(--wm-color-primary)`)
* **Font family customization** — prompts user to import custom fonts; auto-detects Google Fonts and system fonts
* **Foundation reference** — uses Design System foundation.css to ensure semantic correctness

Typical output: **140+ unique design tokens** (25+ typography, 40+ colors, 35+ spacing) in a single `app.override.css` file ready for Studio import.

Certain advanced customisations — such as brand theme logic and heavily custom page structures — still require manual review.