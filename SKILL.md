---
name: wm-design-system-migrator
description: Use this skill to run the complete WaveMaker project migration pipeline in
  one shot. It orchestrates wm-projectconversion (DEFAULT → DesignSystem format conversion), 
  wm-component-conversion (wm-layoutgrid / wm-gridrow / wm-gridcolumn + wm-linearlayout /
  wm-linearlayoutitem → wm-container), wm-theme-to-designsystem-conversion (legacy theme tokens → design-tokens), 
  then produces a Studio-importable ZIP. Shows a unified plan before writing any files and 
  prints a combined summary of all phases on completion. Individual phases can be skipped via 
  --skip-designsystem, --skip-autolayout, or --skip-theme. Supports project renaming, output 
  path override, page filtering, responsive CSS injection, and custom font selection. Use this 
  skill when the user wants to fully migrate a WaveMaker project from DEFAULT template to 
  DesignSystem with modern flex layouts and theme tokens in a single command.
metadata:
  version: 0.1.0
---

# /wm-design-system-migrator — WaveMaker Full Migration Orchestrator

Full pipeline to convert a WaveMaker DEFAULT-template project to DesignSystem,
(optionally) convert grid layouts to flex containers, migrate legacy theme tokens,
and produce a Studio-importable ZIP.

Sub-skills this orchestrates:
- **wm-projectconversion** — Project conversion (pom.xml, .wmproject.properties,
  index.html, variables, page layouts, themes → design-tokens, npm scope,
  migration_info)
- **wm-component-conversion** — Grid & LinearLayout → flex container conversion
  (wm-layoutgrid / wm-gridrow / wm-gridcolumn + wm-linearlayout / wm-linearlayoutitem → wm-container)
- **wm-theme-to-designsystem-conversion** — Legacy theme token extraction and migration
  (style.css typography/colors/spacing → design-tokens/app.override.css)

Both sub-skills remain independently usable. Use this skill when you want the
full pipeline in one shot.

---

## Invocation

```
/wm-design-system-migrator <project_path>
/wm-design-system-migrator <project_path> -o <output_path>
/wm-design-system-migrator <project_path> --project-name <name>
/wm-design-system-migrator <project_path> --skip-autolayout
/wm-design-system-migrator <project_path> --skip-designsystem
/wm-design-system-migrator <project_path> --skip-theme
/wm-design-system-migrator <project_path> --responsive
/wm-design-system-migrator <project_path> --pages <Page1,Page2>
/wm-design-system-migrator <project_path> --theme <theme_name>
```

| Argument | Required | Description |
|---|---|---|
| `<project_path>` | Yes | Absolute path to the WaveMaker project |
| `-o <output_path>` | No | Write converted project here; source stays untouched |
| `--project-name <name>` | No | Rename project (updates artifactId, displayName, etc.) |
| `--skip-designsystem` | No | Skip Project conversion; only run autolayout + theme + ZIP |
| `--skip-autolayout` | No | Skip autolayout conversion; only run DesignSystem + theme + ZIP |
| `--skip-theme` | No | Skip theme conversion; only run DesignSystem + autolayout + ZIP |
| `--responsive` | No | Inject mobile media-query CSS when converting autolayout |
| `--pages <names>` | No | Comma-separated pages to target for autolayout (default: all) |
| `--theme <name>` | No | Theme folder name to migrate (detected from .wmproject.properties if absent) |

**Natural-language equivalents** (Claude resolves these from the user's prompt):

| User says | Resolves to |
|---|---|
| "don't convert layout" / "skip autolayout" / "designsystem only" | `--skip-autolayout` |
| "don't convert to designsystem" / "layout only" / "skip designsystem" | `--skip-designsystem` |
| "don't migrate theme" / "skip theme" | `--skip-theme` |
| "with responsive CSS" / "add mobile breakpoints" | `--responsive` |
| "only convert Home and Login" | `--pages Home,Login` |
| "migrate the 'custom' theme" / "use theme 'blue'" | `--theme custom` or `--theme blue` |

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
- `THEME_NAME` — value after `--theme` (optional; will detect from project in STEP 1 if absent)
- `RUN_DESIGNSYSTEM` — `true` unless `--skip-designsystem` or equivalent intent detected
- `RUN_AUTOLAYOUT` — `true` unless `--skip-autolayout` or equivalent intent detected
- `RUN_THEME` — `true` unless `--skip-theme` or equivalent intent detected
- `ADD_RESPONSIVE` — `true` if `--responsive` or equivalent
- `PAGE_FILTER` — list after `--pages` (empty = all)

Show the detected plan before doing anything:

```
Migration plan for: <SOURCE_DIR>

  Phase 1 — Project conversion:    [ENABLED | SKIPPED (--skip-designsystem)]
  Phase 2 — Layout conversion:      [ENABLED | SKIPPED (--skip-autolayout)]
  Phase 3 — Theme to DesignSystem conversion:      [ENABLED | SKIPPED (--skip-theme)]
  Phase 4 — Packaging:               ALWAYS

Proceed? [Y/n]
```

Wait for confirmation. If the user says no, stop.

---

### STEP 1 · Validate project + detect platform + detect theme

Read `<SOURCE_DIR>/.wmproject.properties`.

- File missing → abort: *"Not a WaveMaker project — .wmproject.properties not found."*
- `RUN_DESIGNSYSTEM = true` AND contains `<entry key="template">PRISM</entry>` → warn:
  *"Project is already DesignSystem. Skipping Phase 1 and proceeding to Phase 2."*
  Set `RUN_DESIGNSYSTEM = false` and continue.

Detect `PLATFORM`:
- `<entry key="platformType">WEB</entry>` → `PLATFORM = WEB`
- `<entry key="platformType">NATIVE_MOBILE</entry>` → `PLATFORM = MOBILE`
- Any other value → `PLATFORM = WEB`

**Detect theme** (only if `RUN_THEME = true` and `THEME_NAME` was not provided via `--theme`):
- Read `.wmproject.properties` and extract `<entry key="currentThemeName">…</entry>`
- If found, set `THEME_NAME` = extracted value
- If not found, scan `<SOURCE_DIR>/src/main/webapp/themes/` for subdirectories
  - If exactly one theme folder exists, use it
  - If multiple theme folders exist, ask user: *"Which theme to migrate? [list]\nEnter theme name:"*
  - If no theme folders exist, warn: *"No themes found; skipping Phase 3 (theme conversion)."* Set `RUN_THEME = false`

Also read `pom.xml` and extract:
- `CURRENT_PARENT_VERSION` — inside `<parent><version>…</version></parent>`
- `CURRENT_RUNTIME_VERSION` — value of `<wavemaker.app.runtime.ui.version>`
- `CURRENT_UPGRADE_VERSION` — value of `.wmproject.properties` key `studioProjectUpgradeVersion`

---

### STEP 2 · Confirm DesignSystem target versions (only when RUN_DESIGNSYSTEM = true)

**This sub-step is mandatory whenever Phase 1 runs — stop and wait for the user to choose; do not fill in versions or use defaults without explicit user input.**

Display the detected platform, current versions, and recommended DesignSystem versions, then prompt the user for how to proceed:

```
Detected platform: [WEB or MOBILE]

Current project versions:
  Parent POM:     <CURRENT_PARENT_VERSION>
  Runtime UI:     <CURRENT_RUNTIME_VERSION>
  Studio upgrade: <CURRENT_UPGRADE_VERSION>

Recommended DesignSystem versions:
  Parent POM:     <REC_PARENT>      ← from defaults table below, matched to PLATFORM
  Runtime UI:     <REC_RUNTIME>
  Studio upgrade: <REC_UPGRADE>

How would you like to set the target DesignSystem versions?
  1) Use recommended defaults
  2) Copy versions from a reference DesignSystem project
  3) Enter versions manually
```

**Stop here — wait for the user's choice before continuing. Do not proceed or infer a choice.**

---

**Option 1 — Use recommended defaults**

Set versions immediately from the table below and confirm to the user:

```
Using recommended defaults:
  Parent POM version:     <RECOMMENDED_PARENT>
  Runtime UI version:     <RECOMMENDED_RUNTIME>
  Studio upgrade version: <RECOMMENDED_UPGRADE>
```

---

**Option 2 — Copy from a reference DesignSystem project**

Ask: *"Please provide the path to the reference DesignSystem project folder or zip file."*

Set `REFERENCE_INPUT` = the path the user supplies.

**Resolve `REFERENCE_DIR`:**

If `REFERENCE_INPUT` ends with `.zip` (case-insensitive):
1. Verify the file exists — if not, tell the user and re-ask.
2. Set `REF_BASENAME` = zip filename without extension.
3. Set `REF_EXTRACT_DIR` = `<dirname(REFERENCE_INPUT)>/<REF_BASENAME>/`
4. If `REF_EXTRACT_DIR` already exists, remove it first:
   ```bash
   rm -rf "<REF_EXTRACT_DIR>"
   ```
5. Extract:
   ```bash
   unzip -q "<REFERENCE_INPUT>" -d "<REF_EXTRACT_DIR>"
   ```
6. Detect the project root (same as STEP 0 zip handling):
   - If `<REF_EXTRACT_DIR>/.wmproject.properties` exists → `REFERENCE_DIR = REF_EXTRACT_DIR`
   - Otherwise → find the single subdirectory that contains `.wmproject.properties` and set `REFERENCE_DIR` to it.
   - If none found → tell the user: *"Could not locate .wmproject.properties in the reference zip. Is this a WaveMaker DesignSystem project?"* and re-ask.

If `REFERENCE_INPUT` is a folder (does not end in `.zip`):
- Verify the folder exists and contains `.wmproject.properties` — if not, tell the user and re-ask.
- Set `REFERENCE_DIR = REFERENCE_INPUT`

**Extract versions from `REFERENCE_DIR`:**

- Read `<REFERENCE_DIR>/pom.xml` and extract:
  - `PARENT_VERSION` — `<parent><version>…</version></parent>`
  - `RUNTIME_UI_VERSION` — `<wavemaker.app.runtime.ui.version>`
- Read `<REFERENCE_DIR>/.wmproject.properties` and extract:
  - `UPGRADE_VERSION` — `<entry key="studioProjectUpgradeVersion">`

Display the extracted values and ask: *"Use these versions? (yes/no)"*
  - Yes → proceed. **`REFERENCE_DIR` is now set and will be used in STEP 9 for mobile design-tokens.**
  - No → fall back to Option 3 (ask user to enter manually). Set `REFERENCE_DIR` = (unset).

**Post-extraction cleanup (zip only):**

If `REFERENCE_INPUT` was a `.zip` file, delete the extracted folder after versions have been fetched (regardless of yes/no above):
```bash
rm -rf "<REF_EXTRACT_DIR>"
```
If `REFERENCE_INPUT` was a folder, do **not** delete anything.

---

**Option 3 — Enter versions manually**

Prompt:

```
Please enter the target DesignSystem versions (press Enter to accept the recommended default):
  Parent POM version    [recommended: <RECOMMENDED_PARENT>]:  ___
  Runtime UI version    [recommended: <RECOMMENDED_RUNTIME>]: ___
  Studio upgrade ver    [recommended: <RECOMMENDED_UPGRADE>]: ___
```

Empty input for any field → use the recommended default for that field.

---

**Recommended defaults by platform:**

| Version field | WEB default | MOBILE default |
|---|---|---|
| Parent POM | `1.0.0-20260513150623` | `1.0.0-20260513150623` |
| Runtime UI | `1.0.0-next.27577` | `1.0.0-next.27601` |
| Studio upgrade | `1115.07` | `1115.08` |

> Mobile DesignSystem uses the **same** version family as web (the `1.0.0-*` line). The older `12.0.0-*` numbers in earlier skill versions were wrong — they came from a pre-DesignSystem mobile track.

After whichever option is chosen, set:
- `PARENT_VERSION`
- `RUNTIME_UI_VERSION`
- `UPGRADE_VERSION`

---

Now show the full scope so the user can review the complete plan before any files are written.


Show the versions chosen by the user(or defaults if user chose defaults):

```
Migrated Project Versions:
  Parent POM version:     <PARENT_VERSION>
  Runtime UI version:     <RUNTIME_UI_VERSION>
  Studio upgrade version: <UPGRADE_VERSION>
```

If `RUN_AUTOLAYOUT = true`, scan and display the autolayout scope:

Scan `<SOURCE_DIR>/src/main/webapp/pages/**/*.html` for `wm-layoutgrid` **and** `wm-linearlayout`.
If `PAGE_FILTER` is set, restrict to matching folder names.

```
Layout scope (Phase 2):
  Page              layoutgrids   gridrows   gridcolumns   linearlayouts   linearlayoutitems
  ──────────────    ───────────   ────────   ───────────   ─────────────   ─────────────────
  <page>               N             N           N               N                 N
  ...
  Total: N pages
```

If neither `wm-layoutgrid` nor `wm-linearlayout` is found and `RUN_AUTOLAYOUT = true`, warn:
*"No wm-layoutgrid or wm-linearlayout found in target pages — Phase 2 will be a no-op."*
Continue (don't abort).

If `RUN_THEME = true`, show the theme scope:

Check if `<SOURCE_DIR>/src/main/webapp/themes/<THEME_NAME>/style.css` exists.

```
Theme scope (Phase 3):
  Theme:         <THEME_NAME>
  Source file:   src/main/webapp/theme/<THEME_NAME>/style.css
  Output file:   src/main/webapp/design-tokens/app.override.css
  Status:        [READY | FILE NOT FOUND - Phase 3 will be skipped]
```

If style.css is missing and `RUN_THEME = true`, warn:
*"style.css not found in theme folder — Phase 3 will be skipped."*
Set `RUN_THEME = false` and continue (don't abort).

---

### STEP 3 · Copy project (only when TARGET_DIR ≠ SOURCE_DIR)

```bash
cp -r "<SOURCE_DIR>" "<TARGET_DIR>"
```

All remaining steps operate on `TARGET_DIR`.

---

## PHASE 1 — Project Conversion (skip entirely if RUN_DESIGNSYSTEM = false)

Use the **Read** tool to load `../wm-projectconversion/SKILL.md` (sibling skill folder).
**Do NOT use the Skill tool** — read the file directly and execute its steps inline.

Execute **STEP 4 through STEP 8.5** from that file inline, using the
variables already resolved in STEP 0–2 above:

| Variable | Source |
|---|---|
| Project directory | `TARGET_DIR` |
| `PARENT_VERSION` | Resolved in STEP 2 |
| `RUNTIME_UI_VERSION` | Resolved in STEP 2 |
| `UPGRADE_VERSION` | Resolved in STEP 2 |
| `PROJECT_NAME` | `--project-name` flag (pass `NONE` if not supplied) |
| `PLATFORM` | Detected in STEP 1 (`WEB` or `MOBILE`) |

**Skip** these wm-projectconversion steps — already handled by this orchestrator:

| Skip | Reason |
|---|---|
| STEP 0 — Parse arguments | Done in STEP 0 above |
| STEP 1 — Validate + detect platform | Done in STEP 1 above |
| STEP 2 — Confirm target versions | Done in STEP 2 above |
| STEP 3 — Copy project | Done in STEP 3 above |
| STEP 9 onwards — Handle after PHASE 3 | Theme tokens must be extracted BEFORE themes/ deletion |
| STEP 13b — Generate ZIP | Handled by PHASE 4 of this orchestrator |
| STEP 14 — Print summary | Handled by STEP 4 of this orchestrator |

Execute **STEP 4 through STEP 8.5** in order, including all per-step mandatory
verifications defined in each step (XML parse check after STEP 5b, grep check
after STEP 7, scope checks after STEP 10). Fix any verification failure before
proceeding to the next step.

**IMPORTANT:** Stop after STEP 8.5 — do NOT execute STEP 9 yet. STEP 9 (delete themes folder)
must happen AFTER PHASE 3 (theme token extraction) to preserve the source files.

---

## PHASE 2 — Layout Conversion (skip entirely if RUN_AUTOLAYOUT = false)

Use the **Read** tool to load `../wm-component-conversion/SKILL.md` (sibling skill folder).
**Do NOT use the Skill tool** — read the file directly and execute its steps inline.

Execute **STEP 1 and STEP 3** from that file inline, using the
variables already resolved in STEP 0 above:

| Variable | Source |
|---|---|
| Project directory | `TARGET_DIR` |
| `PAGE_FILTER` | `--pages` flag (empty list = all pages) |
| `ADD_RESPONSIVE` | `true` if `--responsive` was specified |
| `DRY_RUN` | always `false` when called from this orchestrator |

**Skip** these wm-component-conversion steps — already handled by this orchestrator:

| Skip | Reason |
|---|---|
| STEP 0 — Parse arguments | Done in STEP 0 above |
| STEP 2 — Show summary + confirm | Autolayout scope shown in STEP 2 above; user confirmed in STEP 0 |
| STEP 3b — Generate ZIP | Handled by PHASE 3 of this orchestrator |
| STEP 4 — Print summary | Handled by STEP 4 of this orchestrator |

Execute **STEP 1** (validate project and discover target HTML files) and
**STEP 3** (write the conversion script, run it, delete it). Parse the JSON
output to build the per-page counts for the unified summary.

---

## PHASE 3 — Theme Token Migration (skip entirely if RUN_THEME = false)

**EXECUTION ORDER CRITICAL:**
1. Extract tokens from `src/main/webapp/themes/<THEME_NAME>/style.css` (while themes/ folder still exists)
2. Write tokens to `src/main/webapp/design-tokens/app.override.css`
3. THEN delete themes/ folder (in PHASE 3.5)

Use the **Read** tool to load `../wm-theme-to-designsystem-conversion/SKILL.md` (sibling skill folder).
**Do NOT use the Skill tool** — read the file directly and execute its steps inline.

Execute **STEP 0 through STEP 7** from that file inline, using the
variables already resolved in STEP 0–1 above:

| Variable | Source |
|---|---|
| Project directory | `TARGET_DIR` |
| Theme name | `THEME_NAME` (detected in STEP 1) |
| `DRY_RUN` | always `false` when called from this orchestrator |

**Skip** these wm-theme-to-designsystem-conversion steps — already handled by this orchestrator:

| Skip | Reason |
|---|---|
| STEP 0 — Parse arguments | Done in STEP 0 above |
| STEP 1 — Validate project and theme | Done in STEP 1 above |

Execute **STEP 2 through STEP 7** in order:
- STEP 2: Read foundation.css and style.css
- STEP 3: Extract tokens (typography, colors, spacing)
- STEP 3b: Ask user about custom font family (interactive prompt)
- STEP 4: Match tokens against foundation
- STEP 5: Build override CSS with font imports
- STEP 6: Create design-tokens folder
- STEP 7: Print extraction summary (store results for unified summary)

Capture the token extraction summary and store for the unified STEP 4 report:
- Typography token count
- Color token count
- Spacing token count
- Font family customization (YES/NO + font name)
- Output file size

---

## PHASE 3.5 — Delete Legacy Themes (happens AFTER PHASE 3)

After theme tokens have been successfully extracted and written to `design-tokens/app.override.css`,
execute **STEP 9** from `wm-projectconversion/SKILL.md`:

**STEP 9:** Delete the legacy themes folder
```bash
rm -rf "<TARGET_DIR>/src/main/webapp/themes"
```

This is now safe because:
- All tokens have been extracted in PHASE 3
- design-tokens/app.override.css has been populated with the tokens
- The legacy themes folder is no longer needed

---

## PHASE 5 — Packaging (always runs)

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

## STEP 5 · Unified summary

```
Migration complete!

Project:   <TARGET_DIR>
Platform:  <WEB or MOBILE>
ZIP:       <ZIP_PATH>  (<ZIP_SIZE>)

════════════════════════════════════════════════════════
PHASE 1 — Project Conversion        [COMPLETE | SKIPPED]
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
PHASE 2 — Layout Conversion   [COMPLETE | SKIPPED]
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
PHASE 3 — Theme to DesignSystem Conversion   [COMPLETE | SKIPPED]
════════════════════════════════════════════════════════
Theme:     <THEME_NAME>
Source:    src/main/webapp/theme/<THEME_NAME>/style.css
Output:    src/main/webapp/design-tokens/app.override.css

Extracted Tokens:
  ✓ Typography  — N variables
    Font family: [CUSTOM — imported | DEFAULT — using foundation]
    Examples: --wm-font-family-brand ('Roboto'), --wm-h1-font-size (32px), ...
  
  ✓ Colors      — N variables
    Examples: --wm-color-primary (#FF7250), --wm-color-error (#F44336), ...
  
  ✓ Spacing     — N variables
    Examples: --wm-gap-base (8px), --wm-margin-base (4px), ...

Token Mapping:
  • M foundation overrides (e.g., --wm-color-primary, --wm-font-family-brand)
  • N custom tokens (no foundation match, kept as-is)
  • Total: M + N variables

Font Configuration:
  [Font family selection result]
  ✓ Font: <FONT_NAME> | Using foundation defaults
  ✓ Import: @import url() | system font | not needed

════════════════════════════════════════════════════════
Next steps
════════════════════════════════════════════════════════
  1. Import <ZIP_PATH> into WaveMaker Studio
  2. Studio auto-applies remaining DesignSystem migrations on first open
  3. Preview each page; adjust container gap/padding/alignment as needed
  4. Verify theme tokens applied correctly (colors, typography, spacing)
  5. Customise branding via Theme panel (design-tokens/app.override.css)
  6. Build and test the application

Known harmless Studio log lines:
  • "ResourceDoesNotExistException … wm_rn_config.json" — WEB projects don't
    have this file; Studio handles the 404 silently.
  • "ResourceDoesNotExistException … design-tokens/app.override.css" on first
    open — Studio creates it itself during initial generation (will load overrides
    if present).
```

---

## Quick reference — what each phase does

### Phase 1 (wm-projectconversion) → Phase 2 (wm-component-conversion) → Phase 3 (wm-theme-to-designsystem-conversion) → Phase 4 (Packaging)

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

### Phase 2 (wm-component-conversion)

| Widget | Converts to |
|---|---|
| `wm-layoutgrid` | `wm-container direction="row" wrap="true" width="fill" gap="0" columngap="0"` |
| `wm-gridrow` | `wm-container direction="row" wrap="true" width="fill" gap="0" columngap="0"` |
| `wm-gridcolumn columnwidth="N"` | `wm-container direction="row" wrap="true" width="<bootstrap%>"` |
| `wm-linearlayout direction="row/column"` | `wm-container direction="<same>" wrap="true" width="fill" gap="<spacing>" alignment="middle-<h>"` |
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

### Phase 3 (wm-theme-to-designsystem-conversion)

| Source | Category | Target | Example |
|---|---|---|---|
| `style.css` typography | font-family, font-size, font-weight, line-height, letter-spacing | `design-tokens/app.override.css` | `--wm-font-family-brand: 'Roboto', sans-serif` |
| `style.css` colors | primary, secondary, accent, error, success, surface, text, border | `design-tokens/app.override.css` | `--wm-color-primary: #FF7250` |
| `style.css` spacing | gap, margin, padding, space, size | `design-tokens/app.override.css` | `--wm-gap-base: 8px` |

**Token mapping strategy:**
- If legacy token matches foundation semantic name (e.g., `--my-primary` → `--wm-color-primary`), it overrides the foundation value
- If no foundation match, token is kept with original name as a custom override
- Font family extraction is **interactive**: user is prompted whether to import custom font or use foundation defaults
- Font imports are auto-detected: Google Fonts get `@import url()`, system fonts are used directly

**Output location:** `src/main/webapp/design-tokens/app.override.css`
- Created if missing
- Appended if exists (preserves prior customizations)
- Foundation values remain in `src/main/webapp/theme/<theme_name>/foundation.css`
