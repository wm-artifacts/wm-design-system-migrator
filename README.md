# WaveMaker DEFAULT → PRISM Migration

> Claude Code skill that converts WaveMaker DEFAULT-template projects (from 11.x version WEB & NATIVE_MOBILE) to the PRISM design system — fully automated, producing a Studio-importable ZIP.

---

## What this skill does?

A set of Claude slash commands that take a WaveMaker DEFAULT project (folder or `.zip`) and mechanically rewrite every file that must change to make it a valid PRISM project. No manual editing. Output is a ZIP ready to drag into WaveMaker Studio.

---

## Prerequisites

- Claude Code CLI (the skill runs inside Claude)
- Python 3.9+ on PATH (used internally by the skill for JSON/XML processing; no install step needed beyond what macOS ships with)
- For MOBILE conversions: access to a reference PRISM mobile project to copy the `design-tokens/foundation/` skeleton from

---

## Version targets (as of this writing)

| Field | WEB | MOBILE |
|---|---|---|
| Parent POM | `1.0.0-20260513150623` | `1.0.0-20260513150623` |
| Runtime UI | `1.0.0-next.27577` | `1.0.0-next.27601` |
| Studio upgrade | `1115.07` | `1115.08` |

The skill prompts for these at runtime and lets you override them — paste values from a known-good PRISM reference project or accept the recommended defaults.

---

## Slash Commands

| Command | What it does |
|---|---|
| `/wm-studio-migrate` | **Core PRISM conversion** — pom.xml, properties, index.html, variables, page layouts, themes→design-tokens, NPM scope, migration history. Produces a ZIP. |
| `/wm-autolayout-conv` | **Layout modernisation** — converts grid (`wm-layoutgrid / wm-gridrow / wm-gridcolumn`) and linear (`wm-linearlayout / wm-linearlayoutitem`) markup to `wm-container` flex layout. Dry-run mode available. |
| `/wm-studio-migrate` | **Full pipeline** — runs both of the above in sequence, then ZIPs. Single command for the complete migration. |

---

## What gets converted automatically (`/wm-studio-migrate`)


| File / area | What changes |
|---|---|
| `pom.xml` | `com.wavemaker.*` → `ai.wavemaker.*`; parent version + runtime UI version bumped to PRISM values |
| `.wmproject.properties` | `template=DEFAULT` → `template=PRISM`; `studioProjectUpgradeVersion` updated; `supportedLanguages` JSON injected (XML-escaped) when `languageBundleSources=STATIC` |
| `src/main/webapp/index.html` | Legacy `wm-style.css` / `wm-responsive.css` links removed; PRISM `foundation.css` + `design-tokens` block injected (WEB); mobile index left as-is |
| `*.variables.json` (app + all pages) | `wm.NotificationVariable` → `wm.NotificationAction`; same for Navigation, Login, Logout — both `"category"` field and `"_id"` values |
| Page HTML layouts (WEB) | `<wm-left-panel>` moved outside `<wm-content>`; `<wm-header>` / `<wm-footer>` moved inside; `<wm-top-nav>` removed; `navtype="rail" navheight="full"` added |
| Page HTML layouts (MOBILE) | `wm-linearlayout` / `wm-linearlayoutitem` → `wm-container`; **`wm-layoutgrid` family kept as-is** (native PRISM mobile widget) |
| `themes/` directory | Deleted; replaced by `design-tokens/app.override.css` stub (WEB) or full foundation skeleton copied from reference project (MOBILE) |
| `ui-build.js` | `NPM_PACKAGE_SCOPE = '@wavemaker'` → `'@wavemaker-ai'` (direct string, not caught by bulk regex) |
| All source files (`.js .ts .tsx .jsx .json .html .css .md .yml .yaml`) | `@wavemaker/` → `@wavemaker-ai/` everywhere; email addresses (`@wavemaker.com`) preserved |
| `wm_rn_config.json` (MOBILE only) | `"enableDesignTokens": true` + `"enableHermes": true` added to `preferences` |
| `migration_info.json` | PRISM migration history entries (1115.03–1115.07 for WEB, +1115.08 for MOBILE) appended; existing history preserved |

### Layout modernisation (`/wm-autolayout-conv`)

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


## Project Conversion Coverage

* Maven groupId rewrites — simple string replacement, no edge cases
* Version bumps — single regex per field
* `template=PRISM` + `studioProjectUpgradeVersion` — simple property substitution
* `wm.*Variable` → `wm.*Action` renames — exact string match across all JSON files
* `@wavemaker/` → `@wavemaker-ai/` bulk scope migration — trailing-slash pattern avoids false positives
* `ui-build.js` `NPM_PACKAGE_SCOPE` — direct literal replacement, handled separately from bulk regex
* `migration_info.json` — idempotent append of PRISM migration history
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
| **Brand CSS / theme variables** | LESS-specific constructs (`darken()`, `lighten()`, `~"..."` wrappers) in the legacy theme cannot be mechanically translated to plain CSS variables. The skill creates a blank `:root {}` stub; brand customisation must be done manually via Studio's Theme panel or by hand-editing `design-tokens/app.override.css`. |
| **MOBILE `design-tokens/` foundation skeleton** | The 40+ files in `design-tokens/foundation/` (Android + iOS style/token/variable files, assets) must be copied from a reference PRISM mobile project. The skill tells you to do this; it does not generate the files itself. |
| **Prefab migration** | Prefab-internal files are not touched. Prefabs are platform-agnostic and run as-is under PRISM; if a prefab breaks at runtime it is a prefab-build issue, not a host-project conversion issue. |
| **Java backend / services** | `services/`, JPA mappings, security config, `build.xml`, `mvnw` — template-agnostic, not touched and not needed. |
| **Highly custom page HTML** | Non-standard widget nesting, inline `<script>` blocks referencing layout elements, or pages that already have a partial PRISM structure may need a visual review after conversion. |

---


## Invocation examples

```bash
# Basic PRISM conversion (accepts folder or .zip)
/wm-studio-migrate /path/to/MyApp

# Convert + output to a different folder
/wm-studio-migrate /path/to/MyApp.zip -o /path/to/MyApp_prism

# Rename project during conversion
/wm-studio-migrate /path/to/MyApp --project-name FinancePortal

# Layout modernisation only (dry run to preview)
/wm-autolayout-conv /path/to/MyApp --dry-run

# Full pipeline: PRISM + layout modernisation + ZIP
/wm-studio-migrate /path/to/MyApp

# Full pipeline, skip layout conversion
/wm-studio-migrate /path/to/MyApp --skip-autolayout

# Full pipeline with responsive CSS injected into page stylesheets
/wm-studio-migrate /path/to/MyApp --responsive
```

---

## Upcoming Enhancements

- Automatic generation of design-tokens/foundation/ skeleton files
- Enhanced support for highly customised page layouts PRISM structures
- Smarter responsive layout optimisation during auto-layout conversion

---

## Summary

This Claude Code skill automates the migration of WaveMaker 11.x DEFAULT-template WEB and NATIVE_MOBILE projects to the PRISM design system. It performs mechanical rewrites across project configuration, layouts, variables, themes, design tokens, and package scopes, producing a Studio-importable ZIP with minimal manual effort.

The migration includes:

* PRISM-compatible dependency and project configuration updates
* Automatic variable and NPM scope migration
* WEB and MOBILE layout transformations
* Theme-to-design-token migration support
* Optional layout modernisation using `wm-container`
* Migration history updates and compatibility fixes

The skill is designed to handle the majority of standard WaveMaker projects deterministically, while also guarding against common migration failures discovered during testing.
Certain advanced customisations — such as brand theme logic, custom page structures, and MOBILE design-token foundations — still require manual review or reference-project assets.





