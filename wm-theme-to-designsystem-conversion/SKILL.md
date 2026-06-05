---
name: wm-theme-to-designsystem-conversion
description: Extract global design tokens (typography, colors, spacing) from legacy theme style.css using @wavemaker/foundation-css package reference, outputting to app.override.css for design system theme customization. Use this skill to migrate old theme configurations to the new design-token-based system during a DesignSystem template migration.
metadata:
  version: 0.1.0
---

# /wm-theme-to-designsystem-conversion — Legacy Theme → Design Tokens Converter

Convert legacy custom theme styles from `style.css` into design tokens that override
the foundation theme. Extracts global tokens for typography, colors, and spacing, then
writes them to `src/main/webapp/design-tokens/app.override.css`.

---

## Invocation

```
/wm-theme-to-designsystem-conversion <project_path> <theme_name>
/wm-theme-to-designsystem-conversion <project_path> <theme_name> --dry-run
/wm-theme-to-designsystem-conversion <project_path> <theme_name> --verbose
```

| Argument | Required | Description |
|---|---|---|
| `<project_path>` | Yes | Absolute path to the WaveMaker project |
| `<theme_name>` | Yes | Theme folder name (e.g., `default`, `light`, `custom`) |
| `--dry-run` | No | Preview what would be extracted — no files are written |
| `--verbose` | No | Show detailed extraction logs and token categorization |

---

## Execution — follow every step in order

### STEP 0 · Parse arguments from $ARGUMENTS

Extract:
- `PROJECT_DIR` — first positional value
- `THEME_NAME` — second positional value
- `DRY_RUN` — `true` if `--dry-run` is present
- `VERBOSE` — `true` if `--verbose` is present

If either `PROJECT_DIR` or `THEME_NAME` is missing, ask: *"Please provide the project path and theme name, e.g., `/wm-theme-to-designsystem-conversion /path/to/project default`"*

---

### STEP 1 · Validate project and theme

Check:
1. `<PROJECT_DIR>/src/main/webapp/themes/<THEME_NAME>/` exists
   - If missing → abort: *"Theme directory not found: `<PROJECT_DIR>/src/main/webapp/themes/<THEME_NAME>/`"*

2. `<PROJECT_DIR>/src/main/webapp/themes/<THEME_NAME>/style.css` exists
   - If missing → abort: *"No style.css found in theme folder."*

3. `<PROJECT_DIR>/src/main/webapp/` exists (for output path validation)
   - If missing → abort: *"Not a valid WaveMaker project — webapp directory not found."*

---

### STEP 2 · Install foundation.css package and read legacy style.css

**Foundation CSS package installation:**
1. Install the `@wavemaker/foundation-css` package into the assets folder:
   ```bash
   npm install @wavemaker/foundation-css --prefix <PROJECT_DIR>/src/main/webapp/design-tokens/
   ```

2. After installation, reference the following from the installed package:

   **Foundation CSS file:** `node_modules/@wavemaker/foundation-css/foundation/foundation.css`
   - Contains all `:root { --wm-*: ... }` semantic token definitions
   - Used as reference for token mapping

   **Global token definitions:** `node_modules/@wavemaker/foundation-css/src/tokens/web/global/`
   - `border.json` — border-radius, border-width, border-color tokens
   - `color.json` — color semantic tokens (primary, secondary, error, success, etc.)
   - `spacing.json` — gap, margin, padding, size tokens
   - `typography.json` — font-family, font-size, font-weight, line-height tokens
   - These JSON files define the complete semantic token structure

   **Component structure (reference only):** `node_modules/@wavemaker/foundation-css/src/tokens/web/components/`
   - Component-specific tokens if needed for advanced customization
   - Structure: `components/<component-name>/<component-name>.json`

**Legacy CSS location (from project):** `<PROJECT_DIR>/src/main/webapp/themes/<THEME_NAME>/style.css`

Read all files as text. Foundation definitions are used to identify semantic token names
and match legacy theme tokens against foundation tokens for intelligent override mapping.

---

### STEP 3 · Extract tokens from style.css

Parse style.css to identify and extract design tokens. Use **TWO extraction strategies**:

#### Strategy 1: CSS Variables in `:root {}`

If `:root { --var: value; }` exists, extract CSS variable declarations:
- Variable name (e.g., `--brand-primary`)
- Variable value (e.g., `#FF7250`)
- Category (typography / color / spacing based on name patterns)

#### Strategy 2: Actual CSS Property Values (fallback)

If `:root` variables are minimal or absent, **extract actual CSS property values** from the stylesheet:

**Colors:**
- Scan all selectors for `color:`, `background-color:`, `border-color:` properties
- Extract hex (`#FF7250`), rgb (`rgb(255, 114, 80)`), named colors
- Map to semantic foundation names:
  - Primary colors → `--wm-color-primary`
  - Secondary colors → `--wm-color-secondary`
  - Error/danger colors → `--wm-color-error`
  - Success colors → `--wm-color-success`
  - Warning colors → `--wm-color-warning`
  - Info colors → `--wm-color-info`
  - Neutral/gray colors → `--wm-color-surface`, `--wm-color-on-surface`
  - Custom colors → `--wm-<custom-name>` (with `--wm-` prefix)

**Typography:**
- Scan for `font-family:`, `font-size:`, `font-weight:`, `line-height:`, `letter-spacing:`
- Extract from heading selectors (h1, h2, h3, .heading, .title)
- Extract from body text selectors (body, p, .text, .label)
- Map to semantic foundation names:
  - Primary font family → `--wm-font-family-brand`
  - Fallback font family → `--wm-font-family-plain`
  - H1 size → `--wm-h1-font-size`, `--wm-h1-font-weight`, `--wm-h1-line-height`
  - Body size → `--wm-font-size-base`, `--wm-font-weight-normal`
  - Custom typography → `--wm-<custom-name>`

**Spacing:**
- Scan for `padding:`, `margin:`, `gap:`, `border-radius:`, `width:`, `height:` properties
- Extract numeric values with units (px, em, rem, %)
- Map to semantic foundation names:
  - Base gaps → `--wm-gap-base`
  - Base margins → `--wm-margin-base`
  - Base padding → `--wm-padding-base`, `--wm-padding-vertical-base`, `--wm-padding-horizontal-base`
  - Border radius → `--wm-border-radius-sm`, `--wm-border-radius-md`, `--wm-border-radius-lg`
  - Component dimensions → `--wm-<component>-<dimension>` (e.g., `--wm-input-height`, `--wm-header-padding`)

#### Extraction Priority

1. **Prefer existing `:root` CSS variables** if present (explicit intent)
2. **Fall back to actual property values** if `:root` has fewer than 10 variables
3. **Combine both** if both exist (CSS variables + additional properties)

#### Output Structure

Store in a structured object with foundation naming applied:

```python
tokens = {
    'typography': [
        {'name': '--wm-font-family-brand', 'value': 'Arial, sans-serif', 'source': 'style.css'},
        {'name': '--wm-h1-font-size', 'value': '32px', 'source': 'h1 selector'},
        {'name': '--wm-h1-font-weight', 'value': '700', 'source': 'h1 selector'},
        ...
    ],
    'colors': [
        {'name': '--wm-color-primary', 'value': '#FF7250', 'source': 'style.css or .primary selector'},
        {'name': '--wm-color-error', 'value': '#F44336', 'source': '.error selector'},
        ...
    ],
    'spacing': [
        {'name': '--wm-gap-base', 'value': '8px', 'source': 'style.css or .gap selector'},
        {'name': '--wm-padding-base', 'value': '16px', 'source': 'body selector'},
        {'name': '--wm-border-radius-md', 'value': '8px', 'source': '.rounded-md selector'},
        ...
    ],
}
```

#### Semantic Mapping Rules

| Legacy/Found Value | Foundation Token | Rule |
|---|---|---|
| Primary brand color | `--wm-color-primary` | Most prominent color in design |
| Secondary brand color | `--wm-color-secondary` | Second most prominent |
| Error/danger color | `--wm-color-error` | Red/danger tones |
| Success/check color | `--wm-color-success` | Green/success tones |
| Warning color | `--wm-color-warning` | Yellow/orange warning tones |
| Info/blue color | `--wm-color-info` | Blue/info tones |
| Primary font family | `--wm-font-family-brand` | Main heading font |
| Fallback font family | `--wm-font-family-plain` | Body/system font |
| 8px spacing | `--wm-gap-base` | Base unit for gaps |
| 4px spacing | `--wm-margin-base` | Base unit for margins |
| 4-8px border radius | `--wm-border-radius-sm` | Small radius |
| 8px border radius | `--wm-border-radius-md` | Medium radius |
| 16px+ border radius | `--wm-border-radius-lg` | Large radius |

---

### STEP 3b · Ask user about font family customization

If typography tokens include a **custom font family** (e.g., `--my-font-family` or `--font-family-brand`), 
prompt the user:

```
Found custom font family in theme:
  Font: <EXTRACTED_FONT_FAMILY>

Do you want to use this font family in the design system?
  [Y/n]
```

**If user answers YES (Y or Enter):**
1. Store the font family value for later import in STEP 5
2. Plan to add appropriate `@import` or `@font-face` declaration to app.override.css

**If user answers NO (n):**
1. Skip font family customization
2. Use foundation defaults (--wm-font-family-brand and --wm-font-family-plain)

---

### STEP 4 · Match against foundation tokens (reference)

For each extracted token from style.css, check if a corresponding foundation variable exists
in the reference foundation.css:

- **Match rule**: If foundation has a token with a "similar" semantic purpose (e.g., both are primary colors, both are heading fonts), flag it as "overrides foundation"
- **No match**: Flag as "new custom token"

Foundation.css is the **reference standard** bundled with the migration tool. It defines the semantic
token names that all DesignSystem projects use. Legacy theme tokens are mapped to these names.

Example:
```
--my-primary-color: #FF7250
  ↓ (matches semantic purpose in reference foundation.css)
  foundation: --wm-color-primary: #FF7250  ← map to this foundation name
  
--custom-accent: #E91E63
  ↓ (no foundation match in reference)
  custom-only: new token (keep original name)
```

---

### STEP 5 · Build override CSS

Write to `<PROJECT_DIR>/src/main/webapp/design-tokens/app.override.css`:

**If user approved custom font family in STEP 3b:**

Add font import statements at the top of the file. Determine import method based on font value:

1. **If font is a Google Font** (e.g., `'Roboto', sans-serif`):
   ```css
   @import url('https://fonts.googleapis.com/css2?family=Roboto:wght@300;400;500;600;700&display=swap');
   ```

2. **If font is a web font URL** (e.g., `url('...')`):
   Use as-is or wrap in `@font-face` if needed.

3. **If font is a system font** (e.g., `'Segoe UI', Tahoma, sans-serif`):
   No import needed; just use in variable.

The output file structure:

```css
/**
 * Design Token Overrides — Migrated from legacy theme
 * Theme: <THEME_NAME>
 * Source: <PROJECT_DIR>/src/main/webapp/themes/<THEME_NAME>/style.css
 * 
 * These tokens override foundation.css values.
 * Foundation tokens are defined in src/main/webapp/theme/<THEME_NAME>/foundation.css
 */

/* Font imports (if custom font approved by user) */
@import url('https://fonts.googleapis.com/css2?family=...');

:root {
  /* Typography Overrides */
  <typography tokens here, one per line>

  /* Color Overrides */
  <color tokens here, one per line>

  /* Spacing Overrides */
  <spacing tokens here, one per line>
}

/* Custom tokens (no foundation equivalent) */
:root {
  <custom tokens here if any>
}
```

**Rules for writing tokens:**
1. If a token name from style.css matches a foundation semantic name (e.g., `--my-primary` and `--wm-color-primary`), **map it to the foundation name**:
   ```css
   /* Original: --my-primary-color: #FF7250; */
   --wm-color-primary: #FF7250;  /* Overridden from legacy theme */
   ```

2. If a token has no foundation equivalent, **keep the original name prefixed**:
   ```css
   --my-custom-accent: #E91E63;  /* Custom token */
   ```

3. **For font-family tokens that user approved**: Use the extracted font family value:
   ```css
   /* Original: --font-family-brand: 'Roboto', sans-serif; */
   --wm-font-family-brand: 'Roboto', sans-serif;  /* Custom font from legacy theme */
   ```

4. Add comments above token groups (Typography, Colors, Spacing, Fonts) for clarity.

5. Sort tokens alphabetically within each category.

6. **CRITICAL: Update variable references** — When a token value references another CSS variable (e.g., `var(--brand-primary)`), **always update the reference to point to the new mapped token name**:
   ```css
   /* WRONG - references old token name: */
   --wm-header-active-text-color: var(--brand-primary);
   
   /* CORRECT - references mapped token name: */
   --wm-header-active-text-color: var(--wm-color-primary);
   ```
   This ensures all references resolve to the new DesignSystem naming scheme and eliminates dead reference chains.

7. **CRITICAL: Eliminate duplicate tokens** — Do NOT emit the same token name twice. When processing style.css:
   - Track all emitted token names as you build the output
   - If a token name already appears in the output, skip it (keep the first occurrence)
   - Report duplicates found to the user for awareness
   ```css
   /* WRONG - duplicates: */
   --wm-color-primary: #2294ef;
   --wm-color-primary: color-mix(in srgb, var(--brand-primary), var(--light-mixer) 9%);
   
   /* CORRECT - keep only one: */
   --wm-color-primary: #2294ef;  /* from --brand-primary */
   ```

8. **CRITICAL: Use mapped references for dependent tokens** — For tokens whose values depend on other tokens, use the mapped destination name:
   ```css
   /* WRONG - references original token before mapping: */
   --wm-btn-primary-hover: color-mix(in srgb, var(--brand-primary), var(--light-mixer) 9%);
   
   /* CORRECT - uses mapped token for clarity: */
   --wm-btn-primary-hover: color-mix(in srgb, var(--wm-color-primary), var(--wm-light-mixer) 9%);
   ```

---

### STEP 6 · Create design-tokens folder if needed

If `<PROJECT_DIR>/src/main/webapp/design-tokens/` does not exist:
```bash
mkdir -p "<PROJECT_DIR>/src/main/webapp/design-tokens/"
```

If `app.override.css` already exists, **append** the new tokens instead of overwriting
(preserve any prior overrides). Add a separator comment: `/* --- Legacy theme tokens appended <DATE> --- */`

---

### STEP 7 · Print summary and report

Display a summary:

```
Theme Token Extraction — [DRY RUN: no files written | COMPLETE]

Project:     <PROJECT_DIR>
Theme:       <THEME_NAME>
Source:      <PROJECT_DIR>/src/main/webapp/themes/<THEME_NAME>/style.css
Output:      <PROJECT_DIR>/src/main/webapp/design-tokens/app.override.css

Extracted Tokens:
  ✓ Typography — N variables
    • font-family, font-size, font-weight, line-height, letter-spacing, etc.
    • Examples: --my-heading-font (Arial), --my-body-size (14px), ...
    • Custom font: [YES - imported | NO - using foundation defaults]
  
  ✓ Colors — N variables
    • primary, secondary, accent, surface, border, text, etc.
    • Examples: --my-primary (#FF7250), --my-error (#F44336), ...
  
  ✓ Spacing — N variables
    • gap, margin, padding, space, size, etc.
    • Examples: --my-gap (8px), --my-margin (16px), ...

Token Mapping (against reference foundation.css):
  • M foundation overrides (e.g., --wm-color-primary, --wm-font-family-brand)
  • N custom tokens (no foundation match, kept as-is)
  • Total: M + N variables
  
Reference Used:
  • Foundation: @wavemaker/foundation-css/foundation/foundation.css (installed via npm)
  • Global tokens: @wavemaker/foundation-css/src/tokens/web/global/*.json
  • Legacy theme: <PROJECT_DIR>/src/main/webapp/themes/<THEME_NAME>/style.css

Font Configuration:
  [if user approved custom font]
  ✓ Font family: <FONT_FAMILY>
  ✓ Import method: @import url() | @font-face | system font
  ✓ Added to: app.override.css line 1

Output File:
  Created: <PROJECT_DIR>/src/main/webapp/design-tokens/app.override.css
  Size: <SIZE>
  Status: Ready for Studio import

Next Steps:
  1. Open the project in WaveMaker Studio
  2. Studio will automatically load app.override.css on app startup
  3. If font was imported, verify it loads correctly in page preview
  4. All design system components will use the new tokens
  5. Verify colors, typography, and spacing in the page preview
  6. Adjust tokens in app.override.css if needed (no re-import required)
```

If `--dry-run`: note that no files were written.
If `--verbose`: show the full token list with mappings and font import URL.

---

## Edge Cases and Error Handling

### Missing style.css File

**Behavior:**
- STEP 1 validation checks for `src/main/webapp/themes/<THEME_NAME>/style.css`
- If missing → abort with error: *"No style.css found in theme folder."*

**Solution:**
- Ensure the theme folder contains a `style.css` file
- If the theme has CSS in other files (`.less`, `.scss`), compile them to `style.css` first
- If no theme styling exists, create a blank `style.css` with an empty `:root { }`

---

### Empty :root Variables (No CSS Variables Defined)

**Behavior:**
- If `style.css` exists but has no `:root { --var: value; }` definitions:
  - **STEP 3 uses Strategy 2 (Fallback)** — extracts actual CSS property values
  - Scans selectors for `color:`, `font-family:`, `font-size:`, `padding:`, `margin:`, `border-radius:`, etc.
  - Maps found values to foundation semantic tokens automatically
  - Creates `app.override.css` with extracted tokens using `--wm-*` naming

**Example:**
```css
/* Input: style.css with properties but no :root variables */
body { font-family: Arial, sans-serif; font-size: 14px; }
h1 { color: #2294ef; font-size: 32px; font-weight: 700; }
.primary-btn { background-color: #2294ef; padding: 10px 18px; }
.error { color: #ff6464; }

/* Output: app.override.css with foundation tokens */
:root {
  /* Typography Tokens */
  --wm-font-family-plain: Arial, sans-serif;
  --wm-font-size-base: 14px;
  --wm-h1-font-size: 32px;
  --wm-h1-font-weight: 700;
  
  /* Color Tokens */
  --wm-color-primary: #2294ef;
  --wm-color-error: #ff6464;
  
  /* Spacing Tokens */
  --wm-btn-padding: 10px 18px;
}
```

**Next step:**
- No manual work needed! All values automatically extracted and mapped
- User can fine-tune token names in `app.override.css` if needed
- Studio will immediately use the extracted tokens

---

### CSS Variables Defined Outside :root Selector

**Behavior:**
- Only variables defined in `:root { }` scope are extracted (global scope)
- Variables defined in other selectors (`.class-name { --var: value; }`) are **ignored**
- This is intentional: component-scoped variables are not design tokens

**Example:**
```css
:root {
  --primary-color: #FF7250;        /* ✓ EXTRACTED */
}

.header {
  --header-padding: 16px;          /* ✗ IGNORED (not in :root) */
}

body {
  --body-margin: 0;                /* ✗ IGNORED (not in :root) */
}
```

**Note:** Only `:root` variables are global design tokens. Component-specific CSS variables should remain in component stylesheets.

---

### Referenced Variable Does Not Exist

**Behavior:**
- If a token value contains `var(--referenced-name)` but that variable is not defined:
  - The reference is **preserved as-is** in the output
  - The browser CSS engine will fall back to initial value if reference fails at runtime

**Example:**
```css
/* Input style.css */
:root {
  --primary-color: #FF7250;
  --btn-hover-color: color-mix(in srgb, var(--primary-color), #000 20%);
  --undefined-ref: var(--does-not-exist);  /* Reference to undefined variable */
}

/* Output app.override.css */
:root {
  --wm-color-primary: #FF7250;
  --wm-btn-hover-color: color-mix(in srgb, var(--wm-color-primary), #000 20%);  /* ✓ Mapped correctly */
  --wm-undefined-ref: var(--does-not-exist);  /* ✗ Kept as-is; will fail at runtime */
}
```

**Risk:** At runtime, `--wm-undefined-ref` will not resolve. If this causes rendering issues:
1. Check the source `style.css` for typos in variable names
2. Verify the referenced variable was extracted
3. Manually fix the reference in `app.override.css` or add the missing variable definition

---

### Duplicate Variable Names

**Behavior:**
- If the same variable name appears multiple times in `:root`:
  - **First occurrence is kept** (CSS cascade: later values override)
  - **Subsequent duplicates are discarded** with count reported in verbose mode
- This is safe: CSS naturally handles duplicates via cascade

**Example:**
```css
/* Input style.css */
:root {
  --primary-color: #FF7250;        /* KEPT */
  --primary-color: #E91E63;        /* DISCARDED (duplicate) */
  --primary-color: #2196F3;        /* DISCARDED (duplicate) */
}

/* Output app.override.css */
:root {
  --wm-color-primary: #FF7250;  /* Only the first value is kept */
}
```

**Behavior reported in VERBOSE mode:**
```
ℹ Deduplication: Discarded 2 duplicate token(s)
  - --wm-color-primary (kept first value: #FF7250, discarded: #E91E63, #2196F3)
```

---

### CSS Variables With Complex Values

**Behavior:**
- Variables with complex values are extracted and preserved exactly:
  - `calc()` expressions
  - `color-mix()` functions
  - `linear-gradient()` values
  - Quoted strings with special characters

**Handling:**
- Values are extracted as-is (no simplification)
- Variable references within values are **updated** to use mapped names
- Quotes and special characters are preserved

**Example:**
```css
/* Input */
:root {
  --gradient: linear-gradient(90deg, var(--primary) 0%, var(--secondary) 100%);
  --calc-value: calc(var(--base-size) * 1.5);
  --font-stack: "Roboto", "Arial", sans-serif;
}

/* Output */
:root {
  --wm-gradient: linear-gradient(90deg, var(--wm-color-primary) 0%, var(--wm-color-secondary) 100%);
  --wm-calc-value: calc(var(--wm-base-size) * 1.5);
  --wm-font-stack: "Roboto", "Arial", sans-serif;
}
```

---

## Token extraction rules

### Typography tokens

**Patterns to match:**
- `font-family`, `font-weight`, `font-size`, `line-height`, `letter-spacing`
- `text-decoration`, `text-transform`, `text-align`
- Selectors: heading styles (h1–h6, .heading, .title), body styles (body, p, .text, .label), etc.

**Examples from style.css:**
```css
:root {
  --my-heading-font: 'Segoe UI', Tahoma, sans-serif;
  --my-body-font: Arial, sans-serif;
  --my-h1-size: 32px;
  --my-h1-weight: 700;
  --my-h1-line-height: 40px;
}
```

**Mapped to foundation (if match found):**
```css
--wm-font-family-brand: 'Segoe UI', Tahoma, sans-serif;       /* from --my-heading-font */
--wm-font-family-plain: Arial, sans-serif;                     /* from --my-body-font */
--wm-h1-font-size: 32px;                                       /* from --my-h1-size */
--wm-h1-font-weight: 700;                                      /* from --my-h1-weight */
```

---

### Color tokens

**Patterns to match:**
- `*color*`, `*-bg*`, `*-text*`, `*-border*`, `*-shadow*`, `*-outline*`
- Semantic names: primary, secondary, success, error, warning, info, neutral, surface, background, etc.

**Examples from style.css:**
```css
:root {
  --my-primary: #FF7250;
  --my-error: #F44336;
  --my-surface: #FFFFFF;
  --my-text-color: #35363B;
  --my-border-light: #E4E4E4;
}
```

**Mapped to foundation (if match found):**
```css
--wm-color-primary: #FF7250;          /* from --my-primary */
--wm-color-error: #F44336;            /* from --my-error */
--wm-color-surface: #FFFFFF;          /* from --my-surface */
--wm-color-on-surface: #35363B;       /* from --my-text-color (semantic: "on-surface") */
--wm-color-border: #E4E4E4;           /* from --my-border-light */
```

---

### Spacing tokens

**Patterns to match:**
- `*gap*`, `*margin*`, `*padding*`, `*space*`, `*-size*` (px, em, rem, %)
- Base units: 4px, 8px, 16px multiples (common design system approach)

**Examples from style.css:**
```css
:root {
  --my-gap-base: 8px;
  --my-gap-1: 4px;
  --my-gap-2: 8px;
  --my-gap-3: 12px;
  --my-margin-default: 16px;
  --my-padding: 12px;
}
```

**Mapped to foundation (if match found):**
```css
--wm-gap-base: 8px;                   /* from --my-gap-base */
--wm-gap-1: 4px;                      /* from --my-gap-1 */
--wm-gap-2: 8px;                      /* from --my-gap-2 */
--wm-gap-3: 12px;                     /* from --my-gap-3 */
--wm-margin-base: 16px;               /* from --my-margin-default (semantic) */
```

---

## Output example

**Input style.css** (legacy):
```css
:root {
  --primary-color: #FF7250;
  --secondary-color: #656DF9;
  --success-color: #5AC588;
  --error-color: #F44336;
  --font-family: 'Roboto', sans-serif;
  --font-size-h1: 32px;
  --font-size-body: 14px;
  --gap-sm: 8px;
  --gap-md: 16px;
}
```

**User prompt for font family:**
```
Found custom font family in theme:
  Font: 'Roboto', sans-serif

Do you want to use this font family in the design system?
  [Y/n]
→ User answers: Y
```

**Output app.override.css** (design tokens with font import):
```css
/**
 * Design Token Overrides — Migrated from legacy theme
 * Theme: default
 * Source: /path/to/project/src/main/webapp/themes/default/style.css
 * 
 * These tokens override foundation.css values.
 * Foundation reference: @wavemaker/foundation-css/foundation/foundation.css (npm package)
 */

/* Font imports */
@import url('https://fonts.googleapis.com/css2?family=Roboto:wght@300;400;500;600;700&display=swap');

:root {
  /* Color Overrides */
  --wm-color-error: #F44336;
  --wm-color-primary: #FF7250;
  --wm-color-secondary: #656DF9;
  --wm-color-success: #5AC588;

  /* Spacing Overrides */
  --wm-gap-base: 8px;
  --wm-gap-md: 16px;

  /* Typography Overrides */
  --wm-font-family-brand: 'Roboto', sans-serif;
  --wm-h1-font-size: 32px;
  --wm-body-medium-font-size: 14px;
}
```

---

## Notes

- **Foundation reference**: The `@wavemaker/foundation-css` npm package is installed during STEP 2 to provide foundation token definitions, global token JSON schemas, and component structure. This ensures the latest Design System standards are used for intelligent token mapping.
- **Semantic mapping**: The converter tries to infer semantic meaning from variable names (e.g., `--my-primary` → `--wm-color-primary`). For ambiguous names, tokens are treated as custom (kept as-is).
- **Foundation reference matching**: If a token value in style.css **already uses a foundation variable** (e.g., `--my-gap: var(--wm-gap-base)`), it is skipped (no override needed).
- **Font family handling**:
  - **Google Fonts** (e.g., `'Roboto', sans-serif`): Auto-detect and import from `https://fonts.googleapis.com/css2`
  - **Web fonts** (e.g., URLs): Use as-is or wrap in `@font-face` if needed
  - **System fonts** (e.g., Arial, Helvetica): No import needed, directly use in variable
  - **User approval**: Always ask before importing to avoid unnecessary external requests
- **Dark mode variants**: If style.css has `:root[color='dark']` selectors, process them separately and write to a `:root[color='dark']` section in app.override.css.
- **Incremental writing**: If app.override.css already exists, append new tokens with a clear separator so prior customizations are preserved.
- **Font import detection**: If font value contains a known Google Font name or a URL, auto-suggest the appropriate import method.
