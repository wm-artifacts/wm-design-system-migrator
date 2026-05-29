---
name: wm-studio-migrate
description: Use this skill to run the complete WaveMaker project migration pipeline in
  one shot. It orchestrates wm-designsystem-conv (DEFAULT → DesignSystem format conversion) followed by
  wm-autolayout-conv (wm-layoutgrid / wm-gridrow / wm-gridcolumn + wm-linearlayout /
  wm-linearlayoutitem → wm-container), then produces a Studio-importable ZIP. Shows a
  unified plan before writing any files and prints a combined summary of both phases on
  completion. Either phase can be skipped individually via --skip-designsystem or
  --skip-autolayout. Supports project renaming, output path override, page filtering, and
  responsive CSS injection. Use this skill when the user wants to fully migrate a
  WaveMaker project from DEFAULT template to DesignSystem with modern flex layouts in a single
  command. Do not use this skill if the user only wants DesignSystem conversion without touching
  layouts (use wm-designsystem-conv), or only wants to convert layout widgets on an already-DesignSystem
  project (use wm-autolayout-conv).
metadata:
  version: 0.1.0
---

# /wm-studio-migrate — WaveMaker Full Migration Orchestrator

Full pipeline to convert a WaveMaker DEFAULT-template project to DesignSystem and
(optionally) convert grid layouts to flex containers, then produce a Studio-
importable ZIP.

Sub-skills this orchestrates:
- **wm-designsystem-conv** — DesignSystem conversion (pom.xml, .wmproject.properties,
  index.html, variables, page layouts, themes → design-tokens, npm scope,
  migration_info)
- **wm-autolayout-conv** — Grid & LinearLayout → flex container conversion
  (wm-layoutgrid / wm-gridrow / wm-gridcolumn + wm-linearlayout / wm-linearlayoutitem → wm-container)

Both sub-skills remain independently usable. Use this skill when you want the
full pipeline in one shot.

---

## Invocation

```
/wm-studio-migrate <project_path>
/wm-studio-migrate <project_path> -o <output_path>
/wm-studio-migrate <project_path> --project-name <name>
/wm-studio-migrate <project_path> --skip-autolayout
/wm-studio-migrate <project_path> --skip-designsystem
/wm-studio-migrate <project_path> --responsive
/wm-studio-migrate <project_path> --pages <Page1,Page2>
```

| Argument | Required | Description |
|---|---|---|
| `<project_path>` | Yes | Absolute path to the WaveMaker project |
| `-o <output_path>` | No | Write converted project here; source stays untouched |
| `--project-name <name>` | No | Rename project (updates artifactId, displayName, etc.) |
| `--skip-designsystem` | No | Skip DesignSystem conversion; only run autolayout + ZIP |
| `--skip-autolayout` | No | Skip autolayout conversion; only run DesignSystem + ZIP |
| `--responsive` | No | Inject mobile media-query CSS when converting autolayout |
| `--pages <names>` | No | Comma-separated pages to target for autolayout (default: all) |

**Natural-language equivalents** (Claude resolves these from the user's prompt):

| User says | Resolves to |
|---|---|
| "don't convert layout" / "skip autolayout" / "designsystem only" | `--skip-autolayout` |
| "don't convert to designsystem" / "layout only" / "skip designsystem" | `--skip-designsystem` |
| "with responsive CSS" / "add mobile breakpoints" | `--responsive` |
| "only convert Home and Login" | `--pages Home,Login` |

---

## Execution — follow every step in order

### STEP 0 · Parse arguments and detect intent

Extract from the invocation string (positional args + flags + natural language):

- `SOURCE_DIR` — first positional value. If missing, ask: *"Please provide the path to the WaveMaker project."*
  - **If `SOURCE_DIR` ends with `.zip`** (a ZIP archive was provided instead of a directory):
    - Set `SOURCE_ZIP_BASENAME` = filename without `.zip` extension (e.g. `DataWiz.zip` → `DataWiz`)
    - Extract: `unzip -q "<SOURCE_DIR>" -d "<parent_dir>/<SOURCE_ZIP_BASENAME>_extracted"`
    - Update `SOURCE_DIR` = `<parent_dir>/<SOURCE_ZIP_BASENAME>_extracted`
    - If `-o` was not specified, set `TARGET_DIR` = `SOURCE_DIR`
  - **Otherwise** (directory path given): `SOURCE_ZIP_BASENAME` = basename of `SOURCE_DIR`
- `TARGET_DIR` — value after `-o` (default = `SOURCE_DIR`)
- `PROJECT_NAME` — value after `--project-name` (optional)
- `RUN_DESIGNSYSTEM` — `true` unless `--skip-designsystem` or equivalent intent detected
- `RUN_AUTOLAYOUT` — `true` unless `--skip-autolayout` or equivalent intent detected
- `ADD_RESPONSIVE` — `true` if `--responsive` or equivalent
- `PAGE_FILTER` — list after `--pages` (empty = all)

Show the detected plan before doing anything:

```
Migration plan for: <SOURCE_DIR>

  Phase 1 — DesignSystem conversion:    [ENABLED | SKIPPED (--skip-designsystem)]
  Phase 2 — AutoLayout conversion: [ENABLED | SKIPPED (--skip-autolayout)]
  Phase 3 — ZIP creation:         ALWAYS

Proceed? [Y/n]
```

Wait for confirmation. If the user says no, stop.

---

### STEP 1 · Validate project + detect platform

Read `<SOURCE_DIR>/.wmproject.properties`.

- File missing → abort: *"Not a WaveMaker project — .wmproject.properties not found."*
- `RUN_DESIGNSYSTEM = true` AND contains `<entry key="template">PRISM</entry>` → warn:
  *"Project is already DesignSystem. Skipping Phase 1 and proceeding to Phase 2."*
  Set `RUN_DESIGNSYSTEM = false` and continue.

Detect `PLATFORM`:
- `<entry key="platformType">WEB</entry>` → `PLATFORM = WEB`
- `<entry key="platformType">NATIVE_MOBILE</entry>` → `PLATFORM = MOBILE`
- Any other value → `PLATFORM = WEB`

Also read `pom.xml` and extract:
- `CURRENT_PARENT_VERSION` — inside `<parent><version>…</version></parent>`
- `CURRENT_RUNTIME_VERSION` — value of `<wavemaker.app.runtime.ui.version>`
- `CURRENT_UPGRADE_VERSION` — value of `.wmproject.properties` key `studioProjectUpgradeVersion`

---

### STEP 2 · Confirm DesignSystem target versions (only when RUN_DESIGNSYSTEM = true)

**This sub-step is mandatory whenever Phase 1 runs — do not skip or use defaults silently.**

Display:

```
Detected platform: [WEB or MOBILE]
Current versions:
  Parent POM:     <CURRENT_PARENT_VERSION>
  Runtime UI:     <CURRENT_RUNTIME_VERSION>
  Studio upgrade: <CURRENT_UPGRADE_VERSION>

Target DesignSystem versions — press Enter to accept recommended:
  Parent POM     [recommended: <REC_PARENT>]:  ___
  Runtime UI     [recommended: <REC_RUNTIME>]: ___
  Studio upgrade [recommended: <REC_UPGRADE>]: ___
```

Recommended defaults:

| Field | WEB | MOBILE |
|---|---|---|
| Parent POM | `1.0.0-20260513150623` | `1.0.0-20260513150623` |
| Runtime UI | `1.0.0-next.27577` | `1.0.0-next.27601` |
| Studio upgrade | `1115.07` | `1115.08` |

Set `PARENT_VERSION`, `RUNTIME_UI_VERSION`, `UPGRADE_VERSION` from user input or defaults.

If `RUN_AUTOLAYOUT = true`, also show the autolayout scope at this point so the
user sees the full plan before any files are written:

Scan `<SOURCE_DIR>/src/main/webapp/pages/**/*.html` for `wm-layoutgrid` **and** `wm-linearlayout`.
If `PAGE_FILTER` is set, restrict to matching folder names.

```
AutoLayout scope (Phase 2):
  Page              layoutgrids   gridrows   gridcolumns   linearlayouts   linearlayoutitems
  ──────────────    ───────────   ────────   ───────────   ─────────────   ─────────────────
  <page>               N             N           N               N                 N
  ...
  Total: N pages
```

If neither `wm-layoutgrid` nor `wm-linearlayout` is found and `RUN_AUTOLAYOUT = true`, warn:
*"No wm-layoutgrid or wm-linearlayout found in target pages — Phase 2 will be a no-op."*
Continue (don't abort).

---

### STEP 3 · Copy project (only when TARGET_DIR ≠ SOURCE_DIR)

```bash
cp -r "<SOURCE_DIR>" "<TARGET_DIR>"
```

All remaining steps operate on `TARGET_DIR`.

---

## PHASE 1 — DesignSystem Conversion (skip entirely if RUN_DESIGNSYSTEM = false)

Use the **Read** tool to load `../wm-designsystem-conv/SKILL.md` (sibling skill folder).
**Do NOT use the Skill tool** — read the file directly and execute its steps inline.

Execute **STEP 4 through STEP 13** from that file inline, using the
variables already resolved in STEP 0–2 above:

| Variable | Source |
|---|---|
| Project directory | `TARGET_DIR` |
| `PARENT_VERSION` | Resolved in STEP 2 |
| `RUNTIME_UI_VERSION` | Resolved in STEP 2 |
| `UPGRADE_VERSION` | Resolved in STEP 2 |
| `PROJECT_NAME` | `--project-name` flag (pass `NONE` if not supplied) |
| `PLATFORM` | Detected in STEP 1 (`WEB` or `MOBILE`) |

**Skip** these wm-designsystem-conv steps — already handled by this orchestrator:

| Skip | Reason |
|---|---|
| STEP 0 — Parse arguments | Done in STEP 0 above |
| STEP 1 — Validate + detect platform | Done in STEP 1 above |
| STEP 2 — Confirm target versions | Done in STEP 2 above |
| STEP 3 — Copy project | Done in STEP 3 above |
| STEP 13b — Generate ZIP | Handled by PHASE 3 of this orchestrator |
| STEP 14 — Print summary | Handled by STEP 4 of this orchestrator |

Execute **STEP 4 through STEP 13** in order, including all per-step mandatory
verifications defined in each step (XML parse check after STEP 5b, grep check
after STEP 7, scope checks after STEP 10). Fix any verification failure before
proceeding to the next step.

---

## PHASE 2 — AutoLayout Conversion (skip entirely if RUN_AUTOLAYOUT = false)

Use the **Read** tool to load `../wm-autolayout/SKILL.md` (sibling skill folder).
**Do NOT use the Skill tool** — read the file directly and execute its steps inline.

Execute **STEP 1 and STEP 3** from that file inline, using the
variables already resolved in STEP 0 above:

| Variable | Source |
|---|---|
| Project directory | `TARGET_DIR` |
| `PAGE_FILTER` | `--pages` flag (empty list = all pages) |
| `ADD_RESPONSIVE` | `true` if `--responsive` was specified |
| `DRY_RUN` | always `false` when called from this orchestrator |

**Skip** these wm-autolayout-conv steps — already handled by this orchestrator:

| Skip | Reason |
|---|---|
| STEP 0 — Parse arguments | Done in STEP 0 above |
| STEP 2 — Show summary + confirm | Autolayout scope shown in STEP 2 above; user confirmed in STEP 0 |
| STEP 4 — Print summary | Handled by STEP 4 of this orchestrator |

Execute **STEP 1** (validate project and discover target HTML files) and
**STEP 3** (write the conversion script, run it, delete it). Parse the JSON
output to build the per-page counts for the unified summary.

---

## PHASE 3 — ZIP Creation (always runs)

Output ZIP is always named `<SOURCE_ZIP_BASENAME>_conv_ds.zip` and placed in the same
directory as the source ZIP (or `TARGET_DIR`'s parent). Files are zipped from inside
`TARGET_DIR` so they sit at the ZIP root (Studio-importable without a nested folder).

```bash
cd "<TARGET_DIR>" \
  && zip -rq "../<SOURCE_ZIP_BASENAME>_conv_ds.zip" . -x "*.DS_Store" \
  && cd .. \
  && rm -rf "<TARGET_DIR>"
```

Capture `ZIP_SIZE` via `ls -lh "../<SOURCE_ZIP_BASENAME>_conv_ds.zip"`.

---

## STEP 4 · Unified summary

```
Migration complete!

Project:   <TARGET_DIR>
Platform:  <WEB or MOBILE>
ZIP:       <ZIP_PATH>  (<ZIP_SIZE>)

════════════════════════════════════════════════════════
PHASE 1 — DesignSystem Conversion        [COMPLETE | SKIPPED]
════════════════════════════════════════════════════════
Versions applied:
  Parent POM:       <PARENT_VERSION>
  Runtime UI:       <RUNTIME_UI_VERSION>
  Studio upgrade:   <UPGRADE_VERSION>

  ✓ pom.xml                      groupIds → ai.wavemaker.* + versions
  ✓ .wmproject.properties        template=PRISM, upgradeVersion, supportedLanguages (XML valid ✓)
  ✓ index.html                   [WEB: foundation.css + design-tokens | MOBILE: title only]
  ✓ app.variables.json + N files Variable→Action renamed
  ✓ Page layout restructured     <list of pages> (left-panel outside, header/footer inside)
  ✓ themes/ removed              design-tokens/app.override.css stub created
  ✓ ui-build.js                  NPM_PACKAGE_SCOPE = '@wavemaker-ai'
  ✓ @wavemaker/ scope            N source files updated
  ✓ migration_info.json          [created | appended] — history preserved
  [MOBILE] ✓ wm_rn_config.json  enableDesignTokens=true + enableHermes=true

Prefabs (passthrough — no conversion required):
  • <prefab>   used in: <pages>   ← omit section if no prefabs

════════════════════════════════════════════════════════
PHASE 2 — AutoLayout Conversion   [COMPLETE | SKIPPED | DRY RUN]
════════════════════════════════════════════════════════
  Page              layoutgrids   gridrows   gridcolumns   linearlayouts   linearlayoutitems
  ──────────────    ───────────   ────────   ───────────   ─────────────   ─────────────────
  <page>                 N            N           N               N                 N
  ...
  Total: N layoutgrids, N gridrows, N gridcolumns, N linearlayouts, N linearlayoutitems across N pages
  Collapsed: N redundant single-child converted wrappers removed (pre-existing wm-container elements are never collapsed — only containers produced by this conversion are eligible)

  wrap="true" on all row containers — responsive wrapping built-in
  [--responsive: mobile breakpoint CSS injected into each page's .css]

════════════════════════════════════════════════════════
Next steps
════════════════════════════════════════════════════════
  1. Import <ZIP_PATH> into WaveMaker Studio
  2. Studio auto-applies remaining DesignSystem migrations on first open
  3. Preview each page; adjust container gap/padding/alignment as needed
  4. Customise branding via Theme panel (design-tokens)
  5. Build and test the application

Known harmless Studio log lines:
  • "ResourceDoesNotExistException … wm_rn_config.json" — WEB projects don't
    have this file; Studio handles the 404 silently.
  • "ResourceDoesNotExistException … design-tokens/app.override.css" on first
    open — Studio creates it itself during initial generation.
```

---

## Quick reference — what each phase does

### Phase 1 (wm-designsystem-conv)

| File | Change |
|---|---|
| `pom.xml` | `com.wavemaker.*` → `ai.wavemaker.*`; versions updated |
| `.wmproject.properties` | `template=PRISM`; `studioProjectUpgradeVersion`; `supportedLanguages` JSON (XML-escaped) when `languageBundleSources=STATIC` |
| `index.html` | WEB: removes `wm-style.css`/`wm-responsive.css`; injects `foundation.css` + `design-tokens` block |
| `*.variables.json` | `wm.NotificationVariable` → `wm.NotificationAction`; Navigation/Login/Logout same |
| Page `*.html` (WEB) | left-panel moved outside `wm-content` + `navtype="rail" navheight="full"`; header/footer moved inside |
| `themes/` | Removed; `design-tokens/app.override.css` stub created |
| `ui-build.js` | `NPM_PACKAGE_SCOPE = '@wavemaker-ai'`; bulk `@wavemaker/` → `@wavemaker-ai/` |
| `wm_rn_config.json` | MOBILE only: `enableDesignTokens=true`, `enableHermes=true` |
| `migration_info.json` | DesignSystem entries 1115.03–1115.07 appended; full history preserved |

### Phase 2 (wm-autolayout-conv)

| Widget | Converts to |
|---|---|
| `wm-layoutgrid` | `wm-container direction="row" wrap="true" width="fill" gap="0" columngap="0"` |
| `wm-gridrow` | `wm-container direction="row" wrap="true" width="fill" gap="0" columngap="0"` |
| `wm-gridcolumn columnwidth="N"` | `wm-container direction="column" width="<bootstrap%>"` |
| `wm-linearlayout direction="row/column"` | `wm-container direction="<same>" wrap="true" width="fill" gap="<spacing>" alignment="<v>-<h>"` |
| `wm-linearlayoutitem flexgrow="N"` | `wm-container direction="<perpendicular to parent>" width="<flexgrow%>" padding="<if set>"` |

Width scale (both `columnwidth` and `flexgrow`): 1→8.33% 2→16.67% 3→25% 4→33.33% 5→41.67% 6→50% 7→58.33% 8→66.67% 9→75% 10→83.33% 11→91.67% 12→fill

Item direction rule: parent `direction="row"` → item `direction="column"`; parent `direction="column"` → item `direction="row"`

**Post-conversion collapse** — redundant single-child converted wrappers are removed automatically.
**Only containers created by this conversion** are eligible. Every converter stamps a temporary
`data-wm-conv="1"` attribute on produced elements; collapse checks for this marker on both
outer and inner before acting, then strips all markers from the final output. Pre-existing
`wm-container` elements (no marker) are never touched, even if they are the sole child of a
converted wrapper — they may carry JS references, `show`/`hide` bindings, or event handlers
that would break if the element were removed.

| Collapse condition | Action |
|---|---|
| Both outer AND inner carry `data-wm-conv` AND same `direction` | Remove outer; inner moves up |
| Both carry `data-wm-conv` AND outer `direction="row"` + inner `direction="column" width="fill"` | Remove outer; inner moves up |

Inner container's name and all attributes are preserved exactly. Only the outer wrapper's open/close tags are removed.
