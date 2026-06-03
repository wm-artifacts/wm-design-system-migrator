---
name: wm-theme-conv
description: Extract global design tokens (typography, colors, spacing) from legacy theme style.css and merge with foundation.css, outputting to app.override.css for design system theme customization. Use this skill to migrate old theme configurations to the new design-token-based system during a DesignSystem template migration.
metadata:
  version: 0.1.0
---

# /wm-theme-conv — Legacy Theme → Design Tokens Converter

Convert legacy custom theme styles from `style.css` into design tokens that override
the foundation theme. Extracts global tokens for typography, colors, and spacing, then
writes them to `src/main/webapp/design-tokens/app.override.css`.

---

## Invocation

```
/wm-theme-conv <project_path> <theme_name>
/wm-theme-conv <project_path> <theme_name> --dry-run
/wm-theme-conv <project_path> <theme_name> --verbose
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

If either `PROJECT_DIR` or `THEME_NAME` is missing, ask: *"Please provide the project path and theme name, e.g., `/wm-theme-conv /path/to/project default`"*

---

### STEP 1 · Validate project and theme

Check:
1. `<PROJECT_DIR>/src/main/webapp/theme/<THEME_NAME>/` exists
   - If missing → abort: *"Theme directory not found: `<PROJECT_DIR>/src/main/webapp/theme/<THEME_NAME>/`"*

2. `<PROJECT_DIR>/src/main/webapp/theme/<THEME_NAME>/style.css` exists
   - If missing → abort: *"No style.css found in theme folder."*

3. `<PROJECT_DIR>/src/main/webapp/` exists (for output path validation)
   - If missing → abort: *"Not a valid WaveMaker project — webapp directory not found."*

---

### STEP 2 · Read foundation.css and legacy style.css

**Foundation CSS location:** `<PROJECT_DIR>/src/main/webapp/theme/<THEME_NAME>/foundation.css`
(This is the base design token file with `:root { --wm-*: ... }` variables)

**Legacy CSS location:** `<PROJECT_DIR>/src/main/webapp/theme/<THEME_NAME>/style.css`

If foundation.css is missing, log a warning but continue (assume default foundation).

Read both files as text.

---

### STEP 3 · Extract tokens from style.css

Parse style.css to identify global CSS variables in `:root {}` selector.
Categorize each variable into:
- **Typography**: patterns matching `*font-*`, `*text-*`, `*line-height*`, `*letter-spacing*`
- **Colors**: patterns matching `*color*`, `*-bg*`, `*-text*`, `*-border*`
- **Spacing**: patterns matching `*gap*`, `*margin*`, `*padding*`, `*space*`, `*-size*` (when unit is length)

For each extracted variable, record:
- Variable name (e.g., `--my-primary-color`)
- Variable value (e.g., `#FF7250`)
- Category (typography / color / spacing)

Store in a structured object:
```python
tokens = {
    'typography': [
        {'name': '--my-font-family', 'value': 'Arial, sans-serif', 'source': 'style.css'},
        ...
    ],
    'colors': [
        {'name': '--my-primary', 'value': '#FF7250', 'source': 'style.css'},
        ...
    ],
    'spacing': [
        {'name': '--my-gap', 'value': '8px', 'source': 'style.css'},
        ...
    ],
}
```

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

### STEP 4 · Match against foundation tokens

For each extracted token, check if a corresponding foundation variable exists:
- **Match rule**: If foundation has a token with a "similar" semantic purpose (e.g., both are primary colors, both are heading fonts), flag it as "overrides foundation"
- **No match**: Flag as "new custom token"

Example:
```
--my-primary-color: #FF7250
  ↓ (matches purpose of)
  foundation: --wm-color-primary: #FF7250  ← same token, can override
  
--custom-accent: #E91E63
  ↓ (no foundation match)
  custom-only: new token (no override)
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
 * Source: <PROJECT_DIR>/src/main/webapp/theme/<THEME_NAME>/style.css
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
Source:      <PROJECT_DIR>/src/main/webapp/theme/<THEME_NAME>/style.css
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

Token Mapping:
  • M foundation overrides (e.g., --wm-color-primary, --wm-font-family-brand)
  • N custom tokens (no foundation match, kept as-is)
  • Total: M + N variables

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
 * Source: /path/to/project/src/main/webapp/theme/default/style.css
 * 
 * These tokens override foundation.css values.
 * Foundation tokens are defined in src/main/webapp/theme/default/foundation.css
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

- **Semantic mapping**: The converter tries to infer semantic meaning from variable names (e.g., `--my-primary` → `--wm-color-primary`). For ambiguous names, tokens are treated as custom (kept as-is).
- **Foundation reference**: If a token value in style.css **already uses a foundation variable** (e.g., `--my-gap: var(--wm-gap-base)`), it is skipped (no override needed).
- **Font family handling**:
  - **Google Fonts** (e.g., `'Roboto', sans-serif`): Auto-detect and import from `https://fonts.googleapis.com/css2`
  - **Web fonts** (e.g., URLs): Use as-is or wrap in `@font-face` if needed
  - **System fonts** (e.g., Arial, Helvetica): No import needed, directly use in variable
  - **User approval**: Always ask before importing to avoid unnecessary external requests
- **Dark mode variants**: If style.css has `:root[color='dark']` selectors, process them separately and write to a `:root[color='dark']` section in app.override.css.
- **Incremental writing**: If app.override.css already exists, append new tokens with a clear separator so prior customizations are preserved.
- **Font import detection**: If font value contains a known Google Font name or a URL, auto-suggest the appropriate import method.
