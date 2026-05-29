---
name: wm-prism-conv
description: Use this skill to convert a WaveMaker DEFAULT-template project (WEB or
  NATIVE_MOBILE) to PRISM design system format. It updates pom.xml groupIds and versions
  from com.wavemaker.* to ai.wavemaker.*, marks the project as PRISM in
  .wmproject.properties, restructures page layouts (wm-left-panel moved outside wm-content,
  header/footer moved inside), renames legacy wm.*Variable action types to wm.*Action,
  replaces the themes directory with design-tokens, updates the NPM scope from @wavemaker
  to @wavemaker-ai, and produces a Studio-importable ZIP. Supports both WEB and
  NATIVE_MOBILE platforms. Versions can be entered manually, copied from a reference PRISM
  project, or accepted as recommended defaults. Use this skill when the user wants to
  upgrade an existing non-PRISM WaveMaker project to PRISM only. Do not use this skill to
  convert grid or linear layout widgets to flex containers (use wm-autolayout-conv for
  that), or when the user wants the full migration pipeline in one shot (use
  wm-studio-migrate instead).
metadata:
  version: 0.1.0
---

# /convert-to-prism — WaveMaker Non-Prism → Prism Converter

Convert a WaveMaker DEFAULT-template project (WEB or NATIVE_MOBILE) to PRISM (design system) format.

Templates are located at: `default-to-prism-conv-templates/web/`, `default-to-prism-conv-templates/mobile/`, `default-to-prism-conv-templates/shared/`

---

## Invocation

```
/convert-to-prism <project_path_or_zip>
/convert-to-prism <project_path_or_zip> -o <output_path>
/convert-to-prism <project_path_or_zip> --project-name <name>
```

| Argument | Required | Description |
|---|---|---|
| `<project_path_or_zip>` | Yes | Absolute path to the non-prism WaveMaker project folder **or** a `.zip` export of that project |
| `-o <output_path>` | No | Write converted project here; source stays untouched |
| `--project-name <name>` | No | Rename project (updates artifactId, displayName, packagePrefix, title) |

---

## Execution — follow every step in order

### STEP 0 · Parse arguments from $ARGUMENTS

Extract from `$ARGUMENTS`:
- `SOURCE_INPUT` — first positional value (may be a folder path or a `.zip` file path)
- `TARGET_DIR` — value after `-o` (default determined below)
- `PROJECT_NAME` — value after `--project-name` (optional)

If `SOURCE_INPUT` is missing, ask: *"Please provide the path to the non-prism WaveMaker project folder or zip file."*

**Zip detection and extraction:**

If `SOURCE_INPUT` ends with `.zip` (case-insensitive):
1. Verify the file exists — if not, abort: *"Zip file not found: `<SOURCE_INPUT>`."*
2. Set `SOURCE_ZIP_BASENAME` = zip filename without extension (e.g. `MyApp.zip` → `MyApp`). Used in STEP 13b to name the output zip `<SOURCE_ZIP_BASENAME>_conv_prism.zip`.
3. Determine `EXTRACT_DIR`:
   - `EXTRACT_DIR` = `<dirname(SOURCE_INPUT)>/<SOURCE_ZIP_BASENAME>/`
   - Example: `/tmp/MyApp.zip` → `EXTRACT_DIR = /tmp/MyApp/`
4. If `EXTRACT_DIR` already exists, remove it first:
   ```bash
   rm -rf "<EXTRACT_DIR>"
   ```
5. Extract the zip:
   ```bash
   unzip -q "<SOURCE_INPUT>" -d "<EXTRACT_DIR>"
   ```
6. After extraction, the zip may contain either:
   - A single top-level directory (e.g. `<EXTRACT_DIR>/MyApp/`) → set `SOURCE_DIR` to that inner directory.
   - Project files directly at `<EXTRACT_DIR>/` → set `SOURCE_DIR = EXTRACT_DIR`.

   Detect by checking whether `<EXTRACT_DIR>/.wmproject.properties` exists:
   - Yes → `SOURCE_DIR = EXTRACT_DIR`
   - No → find the single subdirectory and set `SOURCE_DIR` to it. If there are multiple subdirectories and none contain `.wmproject.properties`, abort: *"Could not locate .wmproject.properties inside the zip. Is this a WaveMaker project export?"*

7. Set `TARGET_DIR`:
   - If `-o` was supplied → use that value (unchanged).
   - Otherwise → `TARGET_DIR = SOURCE_DIR` (conversion happens in-place on the extracted copy; the original zip is untouched).

If `SOURCE_INPUT` is a directory path (does not end in `.zip`):
- Set `SOURCE_DIR = SOURCE_INPUT`
- Set `SOURCE_ZIP_BASENAME` = (unset — output zip will be named after the folder basename)
- Set `TARGET_DIR` = value after `-o` if supplied, otherwise `TARGET_DIR = SOURCE_DIR`

---

### STEP 1 · Validate source project + detect platform

Read `<SOURCE_DIR>/.wmproject.properties`.

- File missing → abort: *"Not a WaveMaker project — .wmproject.properties not found."*
- Contains `<entry key="template">PRISM</entry>` → abort: *"Already a PRISM project. Nothing to do."*
- Must contain `<entry key="template">DEFAULT</entry>` → continue.

Detect `PLATFORM`:
- `<entry key="platformType">WEB</entry>` → `PLATFORM = WEB`
- `<entry key="platformType">NATIVE_MOBILE</entry>` → `PLATFORM = MOBILE`
- Any other value → `PLATFORM = WEB` (default)

Read `<SOURCE_DIR>/pom.xml`. Missing → abort: *"Missing pom.xml."*

Extract from `pom.xml`:
- `CURRENT_PARENT_VERSION` — value inside `<parent><version>...</version></parent>`
- `CURRENT_RUNTIME_VERSION` — value of `<wavemaker.app.runtime.ui.version>`

Extract from `.wmproject.properties`:
- `CURRENT_UPGRADE_VERSION` — value of `<entry key="studioProjectUpgradeVersion">`

---

### STEP 2 · Confirm target versions with user

**This step is mandatory — do not skip or use defaults silently.**

Display the detected project type and current versions, then ask the user how they want to supply the target PRISM versions:

```
Detected platform: [WEB or MOBILE]
Current project versions:
  Parent POM version:     <CURRENT_PARENT_VERSION>
  Runtime UI version:     <CURRENT_RUNTIME_VERSION>
  Studio upgrade version: <CURRENT_UPGRADE_VERSION>

How would you like to set the target PRISM versions?
  1) Use recommended defaults
  2) Copy versions from a reference PRISM project
  3) Enter versions manually
```

**Wait for the user's choice, then follow the matching path:**

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

**Option 2 — Copy from a reference PRISM project**

Ask: *"Please provide the path to the reference PRISM project folder (or its zip)."*

- If the path is a `.zip`, extract it to a temp dir first (same logic as STEP 0 zip handling), then locate the project root.
- Read `pom.xml` from the reference project and extract:
  - `PARENT_VERSION` — `<parent><version>…</version></parent>`
  - `RUNTIME_UI_VERSION` — `<wavemaker.app.runtime.ui.version>`
- Read `.wmproject.properties` from the reference project and extract:
  - `UPGRADE_VERSION` — `<entry key="studioProjectUpgradeVersion">`
- Display the extracted values and ask: *"Use these versions? (yes/no)"*
  - Yes → proceed.
  - No → fall back to Option 3 (ask user to enter manually).

---

**Option 3 — Enter versions manually**

Prompt:

```
Please enter the target PRISM versions (press Enter to accept the recommended default):
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

> Mobile PRISM uses the **same** version family as web (the `1.0.0-*` line). The older `12.0.0-*` numbers in earlier skill versions were wrong — they came from a pre-PRISM mobile track.

After whichever option is chosen, set:
- `PARENT_VERSION`
- `RUNTIME_UI_VERSION`
- `UPGRADE_VERSION`

---

### STEP 3 · Copy project (only when TARGET_DIR ≠ SOURCE_DIR)

If `-o` was supplied:
```bash
cp -r "<SOURCE_DIR>" "<TARGET_DIR>"
```
All remaining steps operate on `TARGET_DIR`.

---

### STEP 4 · Update `pom.xml`

**Reference templates:** `default-to-prism-conv-templates/web/pom_changes.xml` (WEB) or `default-to-prism-conv-templates/mobile/pom_changes.xml` (MOBILE)

Read `<TARGET_DIR>/pom.xml` and apply:

**Both WEB and MOBILE** — inside the `<parent>` block:
```
<groupId>com.wavemaker.app</groupId>  →  <groupId>ai.wavemaker.app</groupId>
<version>ANY_OLD</version>            →  <version>PARENT_VERSION</version>
```
After `</parent>`:
```
<groupId>com.wavemaker.project</groupId>  →  <groupId>ai.wavemaker.project</groupId>
```

> Earlier versions of this skill claimed mobile keeps `com.wavemaker.*` — that was wrong. PRISM mobile uses the `ai.wavemaker.*` groupIds, same as PRISM web. Verified against a reference PRISM mobile project (sample_mobile) created by Studio.

**Both WEB and MOBILE** — in `<properties>`:
```
<wavemaker.app.runtime.ui.version>ANY_OLD</wavemaker.app.runtime.ui.version>
  →  <wavemaker.app.runtime.ui.version>RUNTIME_UI_VERSION</wavemaker.app.runtime.ui.version>
```

**If `--project-name` was given** — after `</parent>`:
```
<artifactId>OLD</artifactId>  →  <artifactId>PROJECT_NAME</artifactId>
<finalName>OLD</finalName>    →  <finalName>PROJECT_NAME</finalName>
```

Save the file.

---

### STEP 5 · Update `.wmproject.properties`

**Reference template:** `default-to-prism-conv-templates/shared/wmproject_properties_changes.txt`

Read `<TARGET_DIR>/.wmproject.properties` and apply (both WEB and MOBILE):
```
<entry key="template">DEFAULT</entry>
  →  <entry key="template">PRISM</entry>

<entry key="studioProjectUpgradeVersion">ANY_VALUE</entry>
  →  <entry key="studioProjectUpgradeVersion">UPGRADE_VERSION</entry>
```

If `--project-name` was given:
```
<entry key="displayName">OLD</entry>  →  <entry key="displayName">PROJECT_NAME</entry>
<entry key="name">OLD</entry>          →  <entry key="name">PROJECT_NAME</entry>
<entry key="packagePrefix">com.OLD</entry>  →  <entry key="packagePrefix">com.PROJECT_NAME</entry>
```

#### STEP 5b · Ensure `supportedLanguages` entry exists (CRITICAL for STATIC bundles)

PRISM Studio's Angular codegen does `Object.values(t.supportedLanguages)` whenever
`languageBundleSources=STATIC`. If the entry is missing, codegen crashes with
`TypeError: Cannot convert undefined or null to object` in
`@wavemaker-ai/angular-codegen/src/update-angular-json.js`, and the generated
runtime app never builds.

Non-prism projects historically don't include `supportedLanguages` in
`.wmproject.properties` (the data lives in `i18n/<lang>.json` files). Studio
hadn't read it from a property in 11.x; PRISM now does.

**Apply this only if** `<entry key="languageBundleSources">STATIC</entry>` is
present AND no `<entry key="supportedLanguages">…</entry>` exists.

Build the JSON value:
1. List all `<TARGET_DIR>/i18n/*.json` files (these are the language bundles
   the project already ships).
2. For each `<lang>.json`, take its full JSON content as the value for the
   `<lang>` key in the `supportedLanguages` map.
3. If `i18n/` is empty or missing, fall back to `defaultLanguage` and use the
   minimal shape:
   ```json
   {
     "<defaultLanguage>": {
       "messages": {},
       "formats": {},
       "files": { "angular": "<defaultLanguage>", "fullCalendar": null, "moment": null },
       "prefabMessages": {}
     }
   }
   ```

Insert the entry next to other locale-related keys (e.g. just after
`<entry key="preferBrowserLang">`):

```xml
<entry key="supportedLanguages">{"en":{"messages":{},"formats":{},"files":{"angular":"en","fullCalendar":null,"moment":null},"prefabMessages":{}}}</entry>
```

The JSON must be a single line of valid JSON. **XML-escape `&`, `<`, `>`** in
the JSON value before inserting (`&` → `&amp;`, `<` → `&lt;`, `>` → `&gt;`).
i18n strings frequently contain HTML snippets like `<br/>` or `<span>...</span>` —
those raw angle brackets crash Studio import with:

```
java.util.InvalidPropertiesFormatException: Element type "br" must be declared.
  at PropertiesDefaultHandler.startElement
```

because Java's XML properties parser parses `<entry>` text content for markup.
Quotes (`"`) inside text content do NOT need escaping (they're not attribute
values), but angle brackets and ampersands DO. Verify with
`xml.etree.ElementTree.parse()` — if Python can parse it, Studio will too.

Save the file.

**Verification (mandatory — do not skip):**

Run both checks after saving:
```bash
grep 'key="supportedLanguages"' "<TARGET_DIR>/.wmproject.properties"
python3 -c "import xml.etree.ElementTree as ET; ET.parse('<TARGET_DIR>/.wmproject.properties'); print('XML valid ✓')"
```

- If the first grep returns **empty** → the entry was not written. Insert it and recheck.
- If Python raises a parse error → the JSON value contains unescaped `<`, `>`, or `&`. Escape them and recheck.

**Do not proceed to STEP 6 until both checks pass.**

---

### STEP 6 · Update `src/main/webapp/index.html`

Read `<TARGET_DIR>/src/main/webapp/index.html`. If not found, skip this step.

**WEB projects:**

**Change 1 — Remove @font-face style block** (if present):
```html
<style>
    @font-face { font-family: 'Roboto' ... }
    body { font-family: 'Roboto' ... }
</style>
```

**Change 2 — Remove old CSS links from `<head>`** (if present):
```html
<link rel="stylesheet" href="_cdnUrl_/wmapp/styles/css/wm-style.css">
<link rel="stylesheet" href="_cdnUrl_/wmapp/styles/css/wm-responsive.css" ...>
```
Also remove their `<link rel="preload" ... onload="...">` variants if present.

**Change 3 — Replace old body CSS block** with prism block (see `default-to-prism-conv-templates/web/prism_index_css_block.html`):

Old body block (remove):
```html
<link rel="preload" href="_cdnUrl_/wmapp/styles/css/wm-style.css" as="style" onload="...">
<link rel="preload" href="_cdnUrl_/wmapp/styles/css/wm-responsive.css" as="style" onload="...">
<link rel="preload" href="themes/_activeTheme_/style.css" as="style" onload="...">
<link rel="preload" href="app.css" as="style" onload="...">
```
Also handles direct (non-onload) variant:
```html
<link rel="stylesheet" href="themes/_activeTheme_/style.css">
<link rel="stylesheet" href="app.css">
```

New block (insert in body, keeping position relative to `<app-root>`):
```html
<link rel="preload" href="_cdnUrl_/wmapp/styles/foundation/foundation.css" as="style">
<link rel="stylesheet" href="_cdnUrl_/wmapp/styles/foundation/foundation.css">

<link rel="preload" href="design-tokens/app.override.css" as="style">
<link rel="stylesheet" href="design-tokens/app.override.css">

<link rel="preload" href="app.css" as="style">
<link rel="stylesheet" href="app.css">
```

**Important:** Keep the Roboto webfont preload line — do NOT remove:
```html
<link rel="stylesheet preload" ... Roboto-Regular-webfont.woff ...>
```

**MOBILE projects:**

Mobile `index.html` only contains a redirect script — no CSS to update.
If `--project-name` was given, update the title:
```html
<title>OLD</title>  →  <title>PROJECT_NAME</title>
```
Otherwise, skip this step for mobile.

Save the file if any changes were made.

---

### STEP 7 · Update `app.variables.json` and all page `*.variables.json`

**Reference template:** `default-to-prism-conv-templates/shared/variables_json_changes.txt`

Find all target files:
- `<TARGET_DIR>/src/main/webapp/app.variables.json`
- `<TARGET_DIR>/src/main/webapp/pages/**/*.variables.json`

For each file, replace all occurrences (in both `"category"` and `"_id"` values):
```
wm.NotificationVariable  →  wm.NotificationAction
wm.NavigationVariable    →  wm.NavigationAction
wm.LoginVariable         →  wm.LoginAction
wm.LogoutVariable        →  wm.LogoutAction
```

If any of these legacy `*Variable` categories survive into PRISM, Studio import fails with
`com.wavemaker.platform.exception.VariableException: Unknown variable type <Name>` and the
project never initializes (Angular codegen then fails downstream with
`TypeError: Cannot convert undefined or null to object` in `updateAngularJSON`).

Save each file that was modified.

**Verification (mandatory — do not skip):**

Run this grep after saving:
```bash
grep -r "wm\.\(NotificationVariable\|NavigationVariable\|LoginVariable\|LogoutVariable\)" \
  "<TARGET_DIR>/src/main/webapp" --include="*.variables.json"
```

If this returns **any output**, those files were missed. Apply the rename to every matched file and re-run the grep. **Do not proceed to STEP 8 until the grep returns empty.**

Common miss: `pages/Common/Common.variables.json` and deep prefab paths like
`WEB-INF/prefabs/<name>/webapp/pages/**/*.variables.json` — the glob
`pages/**/*.variables.json` must cover ALL subdirectories, not just top-level
page folders.

---

### STEP 8 · Restructure page layout HTML files

Find all files: `<TARGET_DIR>/src/main/webapp/pages/*/*.html`

---

#### STEP 8-WEB · Web page layout restructuring

**Reference template:** `default-to-prism-conv-templates/web/prism_page_layout.html`

For each page HTML file:
1. Skip if it does not contain `<wm-page`.
2. Check if `<wm-left-panel` is **inside** `<wm-content>` — if so, restructure.
3. If `<wm-left-panel` is already **outside** `<wm-content>` — skip.

**Rules:**
- `<wm-left-panel>` — move from inside `<wm-content>` to direct child of `<wm-page>` (before `<wm-content>`). Add `navtype="rail" navheight="full"` if not already present. Preserve all existing attributes.
- `<wm-header>` — move from outside `<wm-content>` to **first** child inside `<wm-content>`. Preserve all attributes.
- `<wm-footer>` — move from outside `<wm-content>` to **last** child inside `<wm-content>`. Preserve all attributes.
- `<wm-top-nav>` — remove entirely.
- All other elements (especially `<wm-page-content>` and its children) — preserve exactly.
- Dialogs (`<wm-dialog>`) outside `<wm-content>` stay outside `<wm-content>`.

**OLD layout:**
```html
<wm-page name="mainpage" pagetitle="Main">
    <wm-header content="header" name="header1" height="auto"></wm-header>
    <wm-top-nav name="topnav" content="topnav"></wm-top-nav>
    <wm-content name="content">
        <wm-left-panel columnwidth="2" name="leftpanel" content="leftnav"></wm-left-panel>
        <wm-page-content columnwidth="10" name="mainContent"></wm-page-content>
    </wm-content>
    <wm-footer name="footer1" content="footer"></wm-footer>
</wm-page>
```

**NEW layout:**
```html
<wm-page name="mainpage" pagetitle="Main">
    <wm-left-panel columnwidth="2" name="leftpanel" content="leftnav" navtype="rail" navheight="full"></wm-left-panel>
    <wm-content name="content">
        <wm-header content="header" name="header1" height="auto"></wm-header>
        <wm-page-content columnwidth="10" name="mainContent"></wm-page-content>
        <wm-footer name="footer1" content="footer"></wm-footer>
    </wm-content>
</wm-page>
```

---

#### STEP 8-MOBILE · Mobile page layouts

For PRISM mobile, `wm-layoutgrid` / `wm-gridrow` / `wm-gridcolumn` are
**kept as-is** — they are valid PRISM mobile widgets. The reference
PRISM mobile project (`sample_mobile/pages/Login/Login.html`) uses
`<wm-layoutgrid>` natively under `<wm-page-content>`.

The only mobile-specific structural fixup is for `wm-linearlayout` /
`wm-linearlayoutitem`, which are legacy and should be converted to
`wm-container`. **Do NOT rewrite `wm-layoutgrid`/`wm-gridrow`/`wm-gridcolumn`.**

**Before converting, ask the user:**

First, scan all page HTML files and count the total number of `<wm-linearlayout` occurrences.

If the count is zero → skip this section entirely (no prompt needed).

If the count is one or more → ask:

```
Found <N> wm-linearlayout element(s) across <M> page(s).
Convert wm-linearlayout → wm-container? (yes/no)
```

- User answers **yes** (or `y`) → proceed with the conversions below.
- User answers **no** (or `n`) → skip all linearlayout conversion; do NOT modify any page HTML files in this step. Note the skip in the STEP 14 summary.

**Wait for the user's answer before continuing.**

For each page HTML file, apply only the layout-conversion below. Skip files
with no `wm-linearlayout`.

**Conversion 1 — `wm-linearlayout` → `wm-container`:**

```html
<!-- BEFORE -->
<wm-linearlayout name="layout1" direction="column" horizontalalign="center" verticalalign="top" spacing="8">
  ...children...
</wm-linearlayout>

<!-- AFTER -->
<wm-container name="container{N}" direction="column" alignment="top-center" gap="8" width="fill" class="app-default-container" variant="default" autoLayout="true">
  ...children...
</wm-container>
```

Attribute mapping:
- `name` → auto-assign `container{N}` (sequential counter per file, starting at 1)
- `direction` → copy as-is (default `"column"` if absent)
- `horizontalalign` + `verticalalign` → `alignment="{v}-{h}"`:
  - vertical: `top`→`top`, `center`→`middle`, `bottom`→`bottom` (default `top`)
  - horizontal: `left`→`left`, `center`→`center`, `right`→`right` (default `left`)
  - If neither: `alignment="top-left"`
- `spacing` → `gap` (copy value; default `"4"` if absent)
- Add: `width="fill"` `class="app-default-container"` `variant="default"` `autoLayout="true"`
- Remove: `horizontalalign`, `verticalalign`, `spacing`

> **Removed:** The previous `wm-layoutgrid → nested wm-container` conversion
> has been deleted. PRISM mobile uses `wm-layoutgrid`/`wm-gridrow`/`wm-gridcolumn`
> natively (verified against `sample_mobile`); rewriting them to `wm-container`
> with `autoLayout="true"` breaks bindings and visual layout. Leave layoutgrid
> trees untouched.

**Conversion 2 — `wm-linearlayoutitem` → unwrap or container:**

- Single child inside item → **unwrap**: move child up to parent, remove the `<wm-linearlayoutitem>` tag
- Multiple children → convert to `wm-container` (same attributes as parent linearlayout); note the page needs review

Save each file that was modified. Track pages needing review (had multi-child linearlayoutitems).

---

### STEP 8c · Prefab inventory (BOTH WEB and MOBILE) — read-only

**Do NOT modify any prefab content.**

WaveMaker prefabs are platform-agnostic artifacts: a single prefab build runs
against both DEFAULT and PRISM consumer projects. Prefab-internal variables,
templates and CSS are handled by the prefab runtime — not the host project's
loaders — so a non-PRISM-looking cache under
`src/main/webapp/WEB-INF/prefabs/<prefabname>/` is normal and not a defect.
**No prefab-related conversion is required.**

**Scan (informational only):**

1. List `<TARGET_DIR>/src/main/webapp/WEB-INF/prefabs/*/` directories. If the
   parent dir does not exist or is empty, report "No prefabs detected" and
   omit the section from the summary.
2. Cross-reference each prefab name against
   `<TARGET_DIR>/src/main/webapp/pages/**/*.html` for
   `<wm-prefab prefabname="...">` to list consumption sites.

**Add to the STEP 14 summary** (only when prefabs are detected):

```
Prefabs detected (passthrough — no conversion required, runs as-is in PRISM):
  • <prefab1>   used in: <page>, <page>
  • <prefab2>   used in: <page>
```

Do not flag a prefab as needing migration based on the cache contents.
A prefab failing at runtime in PRISM is a prefab-build bug — file it against
the prefab project; it is not something this conversion can or should fix.

Save nothing; this step is read-only.

---

### STEP 9 · Replace `themes/` with `design-tokens/`

**WEB projects** — Studio generates `design-tokens/app.override.css` on first
open in prism mode. Just remove the legacy themes/:
```bash
rm -rf "<TARGET_DIR>/src/main/webapp/themes"
```
A blank `design-tokens/app.override.css` is sufficient if you want to give
Studio a head start, or to carry over custom CSS variables from `themes/<active>/css-variable.less`.

**MOBILE projects** — `themes/` must be replaced by a fully-formed
`design-tokens/` directory **before** import; mobile Studio expects it
on disk (it does not auto-generate the foundation skeleton from a blank slate).

The reference layout (from a fresh PRISM mobile project):
```
src/main/webapp/design-tokens/
├── app.studio.override.css         # stub:  :root {}
├── dependencies.json               # []
├── themes-config.json              # {"activeTheme": "foundation-theme"}
└── foundation/
    ├── index.html                  # preview canvas
    ├── theme.png
    ├── android/
    │   ├── style.css   style.js   tokens.js   variables.js
    │   └── assets/
    └── ios/
        ├── style.css   style.js   tokens.js   variables.js
        └── assets/
```

Steps:
1. `rm -rf "<TARGET_DIR>/src/main/webapp/themes"`
2. Copy the `design-tokens/` skeleton from a known-good reference PRISM mobile
   project (e.g. `sample_mobile/src/main/webapp/design-tokens/`) into
   `<TARGET_DIR>/src/main/webapp/design-tokens/`.
3. Leave `theme.variables.js` at webapp root untouched (it stays
   `export default {};`).

Studio will repaint the foundation on the first project open, but having the
skeleton present at import time is what prevents the import-time validation
error that would otherwise surface as
`Failed to read properties input stream` / generic "missing artifact" failure.

---

### STEP 10 · Update NPM scope `@wavemaker` → `@wavemaker-ai` — BOTH WEB and MOBILE

The PRISM runtime ships under `@wavemaker-ai`. Three places need updating:

**10a — `<TARGET_DIR>/ui-build.js`** (if present) — apply this as a direct
string replacement BEFORE running the bulk regex in 10b:
```js
const NPM_PACKAGE_SCOPE = '@wavemaker';
```
→
```js
const NPM_PACKAGE_SCOPE = '@wavemaker-ai';
```

> **Critical:** `NPM_PACKAGE_SCOPE` is a bare string literal with no trailing
> slash. The 10b regex (`@wavemaker/`) will NOT match it. If this line is left
> unchanged, Studio's `ui-build.js` constructs the rn-codegen / rn-app paths
> under `@wavemaker/` at runtime, causing:
> `Cannot find module .../node_modules/@wavemaker/rn-codegen/index.js`
> even though every import path was correctly migrated.

**10b — Project source files** that `import` / `require` from the old scope.
Most commonly hit on mobile: monkey-patch files under
`src/main/webapp/resources/patches/*.js` reference
`@wavemaker/app-rn-runtime/...` paths directly. If any of those survive into
the bundle the Metro/React Native build fails with
`Unable to resolve module @wavemaker/app-rn-runtime/...`.

Walk the project tree (excluding `node_modules/`, `target/`, `coverage/`,
`.git/`, `build-src/`) and replace `@wavemaker/` → `@wavemaker-ai/` in files
with these extensions: `.js .ts .tsx .jsx .json .html .css .md .yml .yaml`.

**Use the trailing-slash pattern `@wavemaker/`** — anything else risks breaking
unrelated tokens:
- `@wavemaker.com` → email addresses (in Google OAuth response samples, query
  test values, contact email fields). Do **NOT** touch.
- `@wavemaker-ai` → already migrated, would create `@wavemaker-ai-ai`.
- `wavemaker.app` / `wavemaker.project` in pom.xml — Maven groupIds, handled
  separately in STEP 4.

Python implementation (safe):
```python
import os, re
PATTERN = re.compile(r'@wavemaker/')
SKIP_DIRS = {'node_modules', 'target', 'coverage', '.git', 'build-src'}
EXTS = {'.js', '.ts', '.tsx', '.jsx', '.json', '.html', '.css', '.md', '.yml', '.yaml'}
for dirpath, dirnames, filenames in os.walk(TARGET_DIR):
    dirnames[:] = [d for d in dirnames if d not in SKIP_DIRS]
    for fn in filenames:
        if os.path.splitext(fn)[1].lower() not in EXTS: continue
        path = os.path.join(dirpath, fn)
        with open(path, encoding='utf-8') as f: c = f.read()
        if '@wavemaker/' not in c: continue
        with open(path, 'w', encoding='utf-8') as f:
            f.write(PATTERN.sub('@wavemaker-ai/', c))
```

After the pass, verify emails survived and ui-build.js is clean:
```bash
grep -r '@wavemaker\.com' <TARGET_DIR>   # should still match (preserved)
grep -r '@wavemaker/' <TARGET_DIR>       # should be empty
grep -n "@wavemaker[^-]" <TARGET_DIR>/ui-build.js  # must be empty — catches bare literal missed by / pattern
```

If the last grep returns any lines, the NPM_PACKAGE_SCOPE constant was not
updated in 10a — apply the direct edit now before continuing.

> Earlier skill versions said "skip for mobile" and "only touch ui-build.js" —
> both were wrong. Mobile projects routinely have hand-written `resources/patches/`
> JS that imports from `@wavemaker/app-rn-runtime`; leaving those at the old
> scope is the #1 cause of "Build failed" in Studio after a mobile conversion
> even when import itself succeeded.

---

### STEP 11 · Update `wm_rn_config.json` — MOBILE ONLY

**Skip this step for WEB projects.**

**Reference template:** `default-to-prism-conv-templates/mobile/wm_rn_config_changes.txt`

Read `<TARGET_DIR>/src/main/webapp/wm_rn_config.json`. If not found, skip with warning.

Find the `"preferences"` object. If it doesn't exist, add `"preferences": {}`.

Add (or update) inside `"preferences"`:
```json
"enableDesignTokens": true,
"enableHermes": true
```

Both keys are present in `sample_mobile/src/main/webapp/wm_rn_config.json`.
`enableHermes` opts the React Native bundle into the Hermes JS engine, which
PRISM mobile expects. Preserve all other fields exactly. Save the file.

---

### STEP 12 · Create `migration_info.json`

Read the template: `default-to-prism-conv-templates/shared/migration_info.json`

If `<TARGET_DIR>/migration_info.json` already exists and already contains migration entry `1115.07`, keep the existing file (it has the full project history).

Otherwise, write the template content to `<TARGET_DIR>/migration_info.json`.

---

### STEP 13 · Create `migration_info.md`

Read the template: `default-to-prism-conv-templates/shared/migration_info.md`

If `<TARGET_DIR>/migration_info.md` already exists, keep it.

Otherwise, write the template content to `<TARGET_DIR>/migration_info.md`.

---

### STEP 13b · Generate the importable ZIP (always)

**This step is always run.** The whole point of the skill is to produce a zip
the user can import into WaveMaker Studio — never leave them to zip it
themselves.

Let:
- `PARENT_DIR` = directory containing `TARGET_DIR`
- `FOLDER_BASENAME` = basename of `TARGET_DIR`
- `ZIP_NAME` — determined by input type:
  - Input was a zip → `ZIP_NAME = <SOURCE_ZIP_BASENAME>_conv_prism`  (e.g. `MyApp_conv_prism`)
  - Input was a folder → `ZIP_NAME = <FOLDER_BASENAME>`
- `ZIP_PATH` = `<PARENT_DIR>/<ZIP_NAME>.zip`

Run (don't include the parent path in the archive; the zip's top-level entry
must be `<FOLDER_BASENAME>/...` so Studio's import detects the project root):

```bash
cd "<PARENT_DIR>" \
  && rm -f "<ZIP_NAME>.zip" \
  && zip -rq "<ZIP_NAME>.zip" "<FOLDER_BASENAME>" -x "*.DS_Store"
```

Notes:
- `rm -f` first so a re-run of the skill overwrites cleanly.
- `-x "*.DS_Store"` keeps macOS folder metadata out of the archive.
- `-rq` = recursive + quiet (errors still print).
- Do NOT include `node_modules/`, `target/`, `.git/` if you ever see them — but
  a standard WaveMaker project export shouldn't have these. Only add
  `-x` patterns for them if you spot them at conversion time.

After zip:
- Capture `ZIP_SIZE` (human-readable, e.g. via `ls -lh`).
- Pass `ZIP_PATH` and `ZIP_SIZE` into the STEP 14 summary.

If `zip` is not available on the user's system → fall back to `python3 -c
"import shutil; shutil.make_archive(...)"` rather than asking the user to
install anything. Either way, end with a single file at `ZIP_PATH`.

---

### STEP 14 · Print conversion summary

```
Conversion complete!

Project:  <TARGET_DIR>
Platform: <WEB or MOBILE>
ZIP:      <ZIP_PATH>  (<ZIP_SIZE>)         ← from STEP 13b — always include

Versions applied:
  Parent POM:       PARENT_VERSION
  Runtime UI:       RUNTIME_UI_VERSION
  Studio upgrade:   UPGRADE_VERSION

Files changed:
  ✓ pom.xml                        groupIds → ai.wavemaker.* + versions (both WEB and MOBILE)
  ✓ .wmproject.properties          template=PRISM, studioProjectUpgradeVersion=UPGRADE_VERSION
                                   supportedLanguages JSON (XML-escaped) when STATIC
  ✓ src/main/webapp/index.html     [WEB: foundation.css + design-tokens block | MOBILE: title only]
  ✓ src/main/webapp/app.variables.json   Variable→Action rename
  ✓ <N> page HTML files            [WEB: layout restructured | MOBILE: linearlayout→container (layoutgrid kept) — or "skipped (user declined linearlayout conversion)" if user said no]
  ✓ src/main/webapp/themes/        [WEB: removed | MOBILE: replaced with design-tokens/ skeleton]
  ✓ ui-build.js                    @wavemaker → @wavemaker-ai (both WEB and MOBILE)
  ✓ src/main/webapp/wm_rn_config.json   [MOBILE: enableDesignTokens=true + enableHermes=true | WEB: skipped]
  ✓ migration_info.json            Created / kept existing
  ✓ migration_info.md              Created / kept existing

Prefabs detected (from STEP 8c; omit this section if none — informational only,
no action required):
  • <name>   used in: <pages>

Known harmless Studio log lines (WEB):
  • "ResourceDoesNotExistException ... wm_rn_config.json" — Studio probes every project
    for mobile config; WEB projects never have this file. Studio handles the 404 silently.
  • "ResourceDoesNotExistException ... design-tokens/app.override.css" on first open —
    Studio creates this file itself during initial app generation.

Next steps:
  1. Open the project in WaveMaker Studio
  2. Studio auto-applies remaining prism migrations on first open
  3. [MOBILE] Studio generates design-tokens/ on first open
  4. Customise branding via Theme panel (colors, fonts, radii via design-tokens)
  5. Build and test the application
```

List any pages needing review (mobile multi-child linearlayoutitem), files skipped, or warnings separately.

---

## Quick reference — change map

### WEB projects

| File | Old value | New value |
|---|---|---|
| `pom.xml` parent groupId | `com.wavemaker.app` | `ai.wavemaker.app` |
| `pom.xml` parent version | (asked) | `PARENT_VERSION` |
| `pom.xml` project groupId | `com.wavemaker.project` | `ai.wavemaker.project` |
| `pom.xml` runtime UI version | (asked) | `RUNTIME_UI_VERSION` |
| `.wmproject.properties` template | `DEFAULT` | `PRISM` |
| `.wmproject.properties` upgradeVersion | (asked) | `UPGRADE_VERSION` |
| `.wmproject.properties` supportedLanguages | (missing, when STATIC) | JSON map from `i18n/*.json` |
| `index.html` head CSS | wm-style.css + wm-responsive.css | removed |
| `index.html` body CSS | themes + app.css | foundation.css + design-tokens + app.css |
| `*.variables.json` | `wm.NotificationVariable` | `wm.NotificationAction` |
| `*.variables.json` | `wm.NavigationVariable` | `wm.NavigationAction` |
| `*.variables.json` | `wm.LoginVariable` | `wm.LoginAction` |
| `*.variables.json` | `wm.LogoutVariable` | `wm.LogoutAction` |
| Page `*.html` layout | left-panel inside content | left-panel outside, header/footer inside |
| `WEB-INF/prefabs/*` cache | (scanned, not modified) | listed in summary; prefabs are platform-agnostic, no conversion needed |
| **Output ZIP** | — | `<TARGET_DIR>.zip` written alongside the project, ready to import into Studio (STEP 13b) |
| `themes/` directory | present | deleted |
| `ui-build.js` `NPM_PACKAGE_SCOPE` constant | `'@wavemaker'` (no slash — not caught by 10b regex) | `'@wavemaker-ai'` — direct edit in STEP 10a |
| `ui-build.js` scope (imports) | `@wavemaker/` | `@wavemaker-ai/` (STEP 10b bulk regex) |
| Source files (`.js .ts .json .html`) imports | `@wavemaker/...` | `@wavemaker-ai/...` (slash pattern; emails like `@wavemaker.com` preserved) |

### MOBILE projects (NATIVE_MOBILE)

| File | Old value | New value |
|---|---|---|
| `pom.xml` parent groupId | `com.wavemaker.app` | `ai.wavemaker.app` |
| `pom.xml` parent version | (asked) | `PARENT_VERSION` |
| `pom.xml` project groupId | `com.wavemaker.project` | `ai.wavemaker.project` |
| `pom.xml` runtime UI version | (asked) | `RUNTIME_UI_VERSION` |
| `.wmproject.properties` template | `DEFAULT` | `PRISM` |
| `.wmproject.properties` upgradeVersion | (asked) | `UPGRADE_VERSION` |
| `.wmproject.properties` supportedLanguages | (missing, when STATIC) | JSON map from `i18n/*.json`, XML-escape `<` `>` `&` |
| `index.html` | redirect-only HTML | title only (if `--project-name`) |
| `*.variables.json` | `wm.NotificationVariable` | `wm.NotificationAction` |
| `*.variables.json` | `wm.NavigationVariable` | `wm.NavigationAction` |
| `*.variables.json` | `wm.LoginVariable` | `wm.LoginAction` |
| `*.variables.json` | `wm.LogoutVariable` | `wm.LogoutAction` |
| Page `*.html` `wm-linearlayout` | legacy | `wm-container` with `autoLayout="true"` |
| Page `*.html` `wm-layoutgrid` family | (keep as-is) | **unchanged** — native PRISM mobile widget |
| `themes/` directory | present | replaced by `design-tokens/` skeleton (foundation/, app.studio.override.css, dependencies.json, themes-config.json) |
| `ui-build.js` `NPM_PACKAGE_SCOPE` constant | `'@wavemaker'` (no slash — not caught by 10b regex) | `'@wavemaker-ai'` — direct edit in STEP 10a |
| `ui-build.js` scope (imports) | `@wavemaker/` | `@wavemaker-ai/` (STEP 10b bulk regex) |
| Source files (`.js .ts .json .html`) imports — esp. `resources/patches/*.js` | `@wavemaker/app-rn-runtime/...` | `@wavemaker-ai/app-rn-runtime/...` (slash pattern; emails preserved) |
| `wm_rn_config.json` preferences | no design-token / hermes flags | `"enableDesignTokens": true, "enableHermes": true` |
| `WEB-INF/prefabs/*` cache | (scanned, not modified) | listed in summary; prefabs are platform-agnostic, no conversion needed |
| **Output ZIP** | — | `<TARGET_DIR>.zip` written alongside the project, ready to import into Studio (STEP 13b) |

---

## Templates reference

| Template file | Platform | Used in step |
|---|---|---|
| `default-to-prism-conv-templates/web/pom_changes.xml` | WEB | Step 4 |
| `default-to-prism-conv-templates/web/prism_index_css_block.html` | WEB | Step 6 |
| `default-to-prism-conv-templates/web/prism_page_layout.html` | WEB | Step 8 |
| `default-to-prism-conv-templates/mobile/pom_changes.xml` | MOBILE | Step 4 |
| `default-to-prism-conv-templates/mobile/wm_rn_config_changes.txt` | MOBILE | Step 11 |
| `default-to-prism-conv-templates/mobile/prism_mobile_layout.txt` | MOBILE | Step 8 |
| `default-to-prism-conv-templates/shared/wmproject_properties_changes.txt` | Both | Step 5 |
| `default-to-prism-conv-templates/shared/variables_json_changes.txt` | Both | Step 7 |
| `default-to-prism-conv-templates/shared/migration_info.json` | Both | Step 12 |
| `default-to-prism-conv-templates/shared/migration_info.md` | Both | Step 13 |
