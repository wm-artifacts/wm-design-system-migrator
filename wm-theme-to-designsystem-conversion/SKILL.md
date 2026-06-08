---
name: wm-theme-to-designsystem-conversion
description: Extract global design tokens (typography, colors, spacing) from legacy theme style.css using @wavemaker/foundation-css package reference, outputting to app.override.css for design system theme customization. Use this skill to migrate old theme configurations to the new design-token-based system during a DesignSystem template migration.
metadata:
  version: 0.2.0
---

# /wm-theme-to-designsystem-conversion — Legacy Theme → Design Tokens Converter

Convert legacy custom theme styles from `style.css` and `app.css` into design tokens and component variants.

**Output:**
- **Global tokens** → `src/main/webapp/design-tokens/app.override.css`
- **Component variants** → `src/main/webapp/design-tokens/overrides/components/<component>/<component>.json`
- **CSS rules** → appended to `app.override.css`

---

## Quick Start

```bash
# Basic extraction
/wm-theme-to-designsystem-conversion /path/to/project default

# Preview without writing files
/wm-theme-to-designsystem-conversion /path/to/project default --dry-run

# Show detailed logs
/wm-theme-to-designsystem-conversion /path/to/project default --verbose
```

---

## Invocation Reference

| Argument | Required | Description |
|---|---|---|
| `<project_path>` | ✅ Yes | Absolute path to WaveMaker project |
| `<theme_name>` | ✅ Yes | Theme folder name (e.g., `default`, `light`, `custom`) |
| `--dry-run` | ❌ No | Preview extraction — no files written |
| `--verbose` | ❌ No | Show detailed extraction logs |

---

## Execution Flow

### STEP 0 · Parse Arguments

**Extract from `$ARGUMENTS`:**

| Variable | Source | Required | Example |
|---|---|---|---|
| `PROJECT_DIR` | 1st positional | ✅ Yes | `/Users/dev/my-project` |
| `THEME_NAME` | 2nd positional | ✅ Yes | `default` |
| `DRY_RUN` | `--dry-run` flag | ❌ No | `false` (default) |
| `VERBOSE` | `--verbose` flag | ❌ No | `false` (default) |

**If missing arguments:**
```
❌ "Please provide the project path and theme name, e.g., 
    /wm-theme-to-designsystem-conversion /path/to/project default"
```

---

### STEP 1 · Validate Project & Theme

**Check existence of these paths:**

| Path | Status | Error Message |
|---|---|---|
| `<PROJECT_DIR>/src/main/webapp/themes/<THEME_NAME>/` | MUST EXIST | "Theme directory not found: `<PROJECT_DIR>/src/main/webapp/themes/<THEME_NAME>/`" |
| `<PROJECT_DIR>/src/main/webapp/themes/<THEME_NAME>/style.css` | MUST EXIST | "No style.css found in theme folder." |
| `<PROJECT_DIR>/src/main/webapp/` | MUST EXIST | "Not a valid WaveMaker project — webapp directory not found." |

**If all checks pass:** → Continue to STEP 2

---

### STEP 2 · Foundation CSS Package Setup

**The `@wavemaker/foundation-css` package is required for token mapping and component detection.**

#### 2.1 — User Prompt

```
The @wavemaker/foundation-css package is required for token mapping and component detection.

Do you want to:
  [1] Check and install (if missing) — only install if not found
  [2] Reinstall/Update — always download latest version
  [3] Skip — assume package is already installed

→ User selects option
```

#### 2.2 — Installation Based on User Choice

| User Choice | Action | Behavior |
|---|---|---|
| **[1] Check & Install** | Look for package | If exists → skip installation<br>If missing → install |
| **[2] Reinstall/Update** | Always install | `npm install @wavemaker/foundation-css --prefix <SKILL_ASSETS>/` |
| **[3] Skip** | Assume installed | If missing → fail at STEP 2b with error |

**Installation command:**
```bash
npm install @wavemaker/foundation-css --prefix <SKILL_ASSETS>/
```

#### 2.3 — Verify Installation

| Check | Pass | Fail |
|---|---|---|
| File exists: `<SKILL_ASSETS>/node_modules/@wavemaker/foundation-css/foundation/foundation.css` | Continue ✅ | Abort: "Failed to install @wavemaker/foundation-css" ❌ |

---

### STEP 2b · Build Component Selector Map

**Build lookup map for detecting WaveMaker components in CSS rules.**

Source: `<SKILL_ASSETS>/node_modules/@wavemaker/foundation-css/src/tokens/web/components/`

**Supported Basic Components:**

| Component | Class Names | Example |
|---|---|---|
| **button** | `.btn`, `.app-button` | `.dark-btn.app-button` |
| **label** | `.label`, `.app-label` | `.text-ellipsis.app-label` |
| **message** | `.message`, `.app-message` | `.success-alert.app-message` |
| **search** | `.search`, `.app-search` | `.search-inline.app-search` |
| **progress** | `.progress`, `.app-progress` | `.striped.app-progress` |
| **progress-circle** | `.progress-circle`, `.app-progress-circle` | `.large.app-progress-circle` |
| **icon** | `.icon`, `.app-icon` | `.icon-sm.app-icon` |
| **anchor** | `.anchor`, `.app-anchor`, `.link` | `.external.app-anchor` |
| **picture** | `.picture`, `.app-picture`, `.img` | `.mobile-card.app-picture` |
| **bottomsheet** | `.bottomsheet`, `.app-bottomsheet` | `.custom.app-bottomsheet` |
| **spinner** | `.spinner`, `.app-spinner` | `.large.app-spinner` |
| **skeleton** | `.skeleton`, `.app-skeleton` | `.wave.app-skeleton` |

**Store as:** `COMPONENT_SELECTOR_MAP` (used in STEP 8)

---

### STEP 3 · Extract Tokens from style.css

**Two extraction strategies:**

#### Strategy 1: CSS Variables in `:root {}`

| Element | Extract | Example |
|---|---|---|
| Variable name | `--var-name` | `--brand-primary` |
| Variable value | `value` | `#FF7250` |
| Category | Auto-detect | color / typography / spacing |

**Patterns for auto-detection:**

| Pattern | Category | Example |
|---|---|---|
| `*color*`, `*-bg*`, `*-text*`, `*-border*` | **Color** | `--my-primary`, `--bg-dark` |
| `*font*`, `*size*`, `*weight*`, `*height*`, `*spacing*` | **Typography** | `--heading-font`, `--body-size` |
| `*gap*`, `*margin*`, `*padding*`, `*space*`, `*-size*` | **Spacing** | `--gap-base`, `--padding-lg` |

#### Strategy 2: CSS Property Values (Fallback)

**If `:root` variables are minimal (< 10), extract actual CSS property values:**

| Property | Pattern | Example |
|---|---|---|
| Colors | `color:`, `background-color:`, `border-color:` | `#FF7250`, `rgb(255, 114, 80)` |
| Typography | `font-family:`, `font-size:`, `font-weight:`, `line-height:` | `Arial, sans-serif`, `32px`, `700` |
| Spacing | `padding:`, `margin:`, `gap:`, `border-radius:`, `width:`, `height:` | `8px`, `16px`, `4px` |

#### Output Structure

```python
tokens = {
    'typography': [
        {'name': '--wm-font-family-brand', 'value': 'Arial, sans-serif', 'source': 'style.css'}
    ],
    'colors': [
        {'name': '--wm-color-primary', 'value': '#FF7250', 'source': 'h1 selector'}
    ],
    'spacing': [
        {'name': '--wm-gap-base', 'value': '8px', 'source': 'style.css'}
    ]
}
```

---

### STEP 3b · Ask User About Custom Font

**If custom font family found:**

```
Found custom font family in theme:
  Font: <EXTRACTED_FONT_FAMILY>

Do you want to use this font family in the design system?
  [Y/n]
```

| Answer | Action |
|---|---|
| **Y** (or Enter) | Store font for import in STEP 5 |
| **n** | Skip font customization |

---

### STEP 4 · Match Against Foundation Tokens

**For each extracted token, check if foundation equivalent exists:**

| Legacy Token | Foundation Match | Action |
|---|---|---|
| `--my-primary-color: #FF7250` | `--wm-color-primary` | Map to foundation name |
| `--custom-accent: #E91E63` | (no match) | Keep as custom token |

**Output:** Categorized token list with "foundation override" or "custom-only" flags

---

### STEP 5 · Build override CSS

**Output file:** `<PROJECT_DIR>/src/main/webapp/design-tokens/app.override.css`

#### Structure

```css
/**
 * Design Token Overrides — Migrated from legacy theme
 * Theme: <THEME_NAME>
 * Source: <PROJECT_DIR>/src/main/webapp/themes/<THEME_NAME>/style.css
 */

/* Font imports (if user approved custom font) */
@import url('https://fonts.googleapis.com/css2?family=Roboto:wght@...');

:root {
  /* Typography Overrides */
  --wm-font-family-brand: 'Roboto', sans-serif;
  --wm-h1-font-size: 32px;
  
  /* Color Overrides */
  --wm-color-primary: #FF7250;
  --wm-color-error: #F44336;
  
  /* Spacing Overrides */
  --wm-gap-base: 8px;
  --wm-padding-base: 16px;
}
```

#### Writing Rules

| Rule | Correct | Wrong |
|---|---|---|
| **Map to foundation names** | `--wm-color-primary: #FF7250;` | `--my-primary: #FF7250;` |
| **Update var() references** | `var(--wm-color-primary)` | `var(--brand-primary)` |
| **No duplicates** | Keep first occurrence | Emit same token twice |
| **Sort tokens** | Alphabetically per category | Random order |

---

### STEP 6 · Create design-tokens folder

**If missing:**
```bash
mkdir -p "<PROJECT_DIR>/src/main/webapp/design-tokens/"
```

**If `app.override.css` exists:**
- Append new tokens (don't overwrite)
- Add separator: `/* --- Legacy theme tokens appended <DATE> --- */`

---

### STEP 7 · Print Summary

**Sample report:**

```
Theme Token Extraction — [COMPLETE | DRY RUN]

Project:     /path/to/project
Theme:       default
Source:      /path/to/project/src/main/webapp/themes/default/style.css
Output:      /path/to/project/src/main/webapp/design-tokens/app.override.css

Extracted Tokens:
  ✓ Typography — 8 variables
    • font-family, font-size, font-weight, line-height, letter-spacing
    • Custom font: YES — Roboto (imported)
  
  ✓ Colors — 6 variables
    • primary, secondary, error, success, surface, text
  
  ✓ Spacing — 5 variables
    • gap, margin, padding, border-radius, size

Token Mapping (vs. foundation.css):
  • 14 foundation overrides
  • 2 custom tokens
  • Total: 16 variables

Output File:
  Created: src/main/webapp/design-tokens/app.override.css
  Size: 2.5 KB
  Status: Ready for Studio import

Next Steps:
  1. Open project in WaveMaker Studio
  2. Studio will automatically load app.override.css
  3. Verify colors, typography, spacing in preview
  4. Adjust tokens in app.override.css if needed
```

---

### STEP 8 · Extract Component Variants (Basic Components Only)

**Focus on basic components from foundation CSS.**

#### 8.1 — Scan Sources (in order)

| Source | Priority | Status |
|---|---|---|
| `style.css` | Primary | Scanned ✅ |
| `app.css` | Secondary | Scanned if exists ✅ |
| `pages/{pageName}/{pageName}.css` | Tertiary | Future phase |

#### 8.2 — Detection Categories

**Category A — Basic Component Variants**

Pattern: `.wm-app` + **basic component class** + **custom modifier class**

```css
/* ✅ EXTRACT — .app-button is basic, .dark-btn is custom modifier */
.wm-app .dark-btn.app-button { 
  background-color: #222; 
  color: #fff; 
}

/* ✅ EXTRACT — .app-label is basic, .text-ellipsis is custom modifier */
.wm-app .text-ellipsis.app-label { 
  overflow: hidden; 
  text-overflow: ellipsis; 
}

/* ❌ SKIP — .app-input is NOT a basic component */
.wm-app .custom-input.app-input { 
  border: 1px solid #ccc; 
}
```

**Category B — Custom Utility Classes**

Pattern: `.wm-app` + **no component class**

```css
/* ✅ EXTRACT — no component class found */
.wm-app .card-wrapper { 
  background: linear-gradient(...); 
  border-radius: 16px; 
}
```

#### 8.3 — Extraction Algorithm

| Step | Action |
|---|---|
| 1 | Skip `:root {}`, `@font-face`, `@keyframes` blocks |
| 2 | Skip non-basic components (input, container, data, etc.) |
| 3 | Parse class tokens from selector |
| 4 | Match against `COMPONENT_SELECTOR_MAP` |
| 5 | If basic component + custom class → **Category A** |
| 6 | If no component class → **Category B** |
| 7 | Update `var()` references (legacy → foundation names) |
| 8 | Deduplicate across source files |

#### 8.4 — Output A: Component Variant JSON

**File path:** `src/main/webapp/design-tokens/overrides/components/<component>/<component>.json`

**IMPORTANT: One JSON file per component, not per variant**
- If `button.json` exists and you find another button variant → **ADD to existing file**
- Do NOT create `button-v2.json` or `button_dark.json`
- Add new variants as sibling nodes under the `"appearances"` object

**When to create:**
- **First variant of a component** → Create new `<component>.json` file
- **Additional variants of same component** → Add to existing `<component>.json` (as sibling appearance)

**Structure Reference:** Follow the foundation CSS component structure from:
- Reference: `@wavemaker/foundation-css/src/tokens/web/components/<component>/<component>.json`
- Example file structure from foundation button/label/anchor components

**JSON Format:**

```json
{
  "<component-name>": {
    "appearances": {
      "<variant-name>": {
        "mapping": {
          /* CSS properties and their token references */
          "<property>": { "value": "{token.reference.value}" },
          "<property-with-nested>": {
            "<sub-property>": { "value": "{token.reference.value}" }
          },
          "states": {
            "hover": { /* hover state properties */ },
            "focus": { /* focus state properties */ },
            "active": { /* active state properties */ },
            "disabled": { /* disabled state properties */ }
          }
        }
      }
    },
    "meta": {
      "appearances": {
        "<variant-name>": {
          "source": "user"
        }
      }
    }
  }
}
```

**Example: Adding Multiple Variants to Same Component**

When you find multiple button variants in CSS, add them all to a **single** `button.json` file:

```css
/* First variant */
.wm-app .dark-btn.app-button {
  color: #fff;
  background: #000;
  font-size: 16px;
  padding: 12px 16px;
}

/* Second variant */
.wm-app .outline-btn.app-button {
  color: #2294ef;
  background: transparent;
  border: 2px solid #2294ef;
  font-size: 14px;
  padding: 10px 14px;
}

/* Third variant */
.wm-app .ghost-btn.app-button {
  color: #666;
  background: transparent;
  font-size: 14px;
  padding: 8px 12px;
}
```

**Single button.json with all three variants:**

```json
{
  "btn": {
    "appearances": {
      "dark_btn": {
        "mapping": {
          "color": { "value": "{color.white.@.value}" },
          "background": { "value": "{color.black.@.value}" },
          "font-size": { "value": "{label.large.font-size.value}" },
          "padding": { "value": "{space.3.value} {space.4.value}" }
        }
      },
      "outline_btn": {
        "mapping": {
          "color": { "value": "{color.primary.@.value}" },
          "background": { "value": "transparent" },
          "border": {
            "width": { "value": "2px" },
            "style": { "value": "solid" },
            "color": { "value": "{color.primary.@.value}" }
          },
          "font-size": { "value": "{label.medium.font-size.value}" },
          "padding": { "value": "{space.2.value} {space.3.value}" }
        }
      },
      "ghost_btn": {
        "mapping": {
          "color": { "value": "{color.gray.@.value}" },
          "background": { "value": "transparent" },
          "font-size": { "value": "{label.medium.font-size.value}" },
          "padding": { "value": "{space.2.value} {space.2.value}" }
        }
      }
    },
    "meta": {
      "appearances": {
        "dark_btn": { "source": "user" },
        "outline_btn": { "source": "user" },
        "ghost_btn": { "source": "user" }
      }
    }
  }
}
```

**Complete Example — Anchor Component with btn_primary Variant:**

```json
{
  "anchor": {
    "appearances": {
      "btn_primary": {
        "mapping": {
          "color": {
            "@": {
              "value": "{color.primary.@.value}"
            }
          },
          "font-size": {
            "value": "{body.medium.font-size.value}"
          },
          "font-family": {
            "value": "{body.medium.font-family.value}"
          },
          "font-weight": {
            "value": "{body.medium.font-weight.value}"
          },
          "line-height": {
            "value": "{body.medium.line-height.value}"
          },
          "letter-spacing": {
            "value": "{body.medium.letter-spacing.value}"
          },
          "text-transform": {
            "value": "none"
          },
          "text": {
            "decoration": {
              "@": {
                "value": "none"
              }
            }
          },
          "gap": {
            "value": "{space.1.value}"
          },
          "icon": {
            "size": {
              "value": "{icon.size.@.value}"
            }
          },
          "image": {
            "size": {
              "value": "{icon.size.@.value}"
            },
            "radius": {
              "value": "{radius.circle.value}"
            }
          },
          "states": {
            "hover": {
              "color": {
                "@": {
                  "value": "~\"color-mix(in srgb, {color.primary.@.value}, {color.black.@.value} {opacity.hover.value})\""
                }
              },
              "text": {
                "decoration": {
                  "@": {
                    "value": "none"
                  }
                }
              }
            },
            "focus": {
              "color": {
                "@": {
                  "value": "~\"color-mix(in srgb, {color.primary.@.value}, {color.black.@.value} {opacity.focus.value})\""
                }
              },
              "text": {
                "decoration": {
                  "@": {
                    "value": "none"
                  }
                }
              }
            },
            "active": {
              "color": {
                "@": {
                  "value": "~\"color-mix(in srgb, {color.primary.@.value}, {color.black.@.value} {opacity.active.value})\""
                }
              },
              "text": {
                "decoration": {
                  "@": {
                    "value": "none"
                  }
                }
              }
            }
          }
        }
      }
    },
    "meta": {
      "appearances": {
        "btn_primary": {
          "source": "user"
        }
      }
    }
  }
}
```

**Mapping Rule Details:**

| Element | Description | Example |
|---|---|---|
| `<component-name>` | Top-level key matches component name | `"button"`, `"label"`, `"anchor"` |
| `"appearances"` | Object containing all variants | Contains `"dark-btn"`, `"outline-btn"`, etc. |
| `<variant-name>` | Custom variant name (snake_case) | `"dark_btn"`, `"text_ellipsis"`, `"btn_primary"` |
| `"mapping"` | CSS properties mapped to token refs | `"color"`, `"background"`, `"font-size"`, etc. |
| `"@"` | Default value key for properties | Used for single values or top-level defaults |
| `"{...value}"` | Token reference (foundation format) | `"{color.primary.@.value}"`, `"{space.2.value}"` |
| `"states"` | Interactive states | `"hover"`, `"focus"`, `"active"`, `"disabled"` |
| `"meta"` | Metadata about the variant | Always include `"source": "user"` |

**Property Nesting Examples:**

| CSS Property | JSON Structure |
|---|---|
| `color: red;` | `"color": { "@": { "value": "{color.primary.@.value}" } }` |
| `font-size: 16px;` | `"font-size": { "value": "{body.medium.font-size.value}" }` |
| `text-decoration: none;` | `"text": { "decoration": { "@": { "value": "none" } } }` |
| `border-color: blue;` | `"border": { "color": { "@": { "value": "{color.blue.@.value}" } } }` |

**State Definitions (hover, focus, active, disabled):**

States contain property overrides that apply during specific interactions:

```json
"states": {
  "hover": {
    "color": { "@": { "value": "~\"color-mix(...)\"" } },
    "background": { "@": { "value": "{color.primary.darken.value}" } }
  },
  "focus": {
    "outline": { "@": { "value": "2px solid {color.primary.@.value}" } }
  },
  "active": {
    "background": { "@": { "value": "{color.primary.active.value}" } }
  },
  "disabled": {
    "opacity": { "@": { "value": "0.38" } },
    "cursor": { "@": { "value": "not-allowed" } }
  }
}
```

**Token Reference Conversion Rules:**

| CSS Value | Foundation Token Reference |
|---|---|
| `#2294ef` (color) | `{color.primary.@.value}` |
| `8px` (spacing) | `{space.2.value}` |
| `4px` (small spacing) | `{space.1.value}` |
| `16px` (large spacing) | `{space.4.value}` |
| `8px` (border-radius) | `{radius.md.value}` |
| `16px` (large radius) | `{radius.lg.value}` |
| `Arial` (font) | `{body.medium.font-family.value}` |
| `16px` (font-size) | `{body.medium.font-size.value}` |
| `400` (font-weight) | `{body.medium.font-weight.value}` |
| `24px` (line-height) | `{body.medium.line-height.value}` |
| `0.5px` (letter-spacing) | `{body.medium.letter-spacing.value}` |

**How to Extract Values from CSS:**

When you find a CSS rule like:

```css
.wm-app .dark-btn.app-button {
  color: #2294ef;
  font-size: 16px;
  background-color: #000;
  padding: 12px 16px;
  border-radius: 8px;
}
```

Convert to JSON mapping:

```json
"mapping": {
  "color": {
    "@": {
      "value": "{color.primary.@.value}"  /* #2294ef → primary color token */
    }
  },
  "font-size": {
    "value": "{body.medium.font-size.value}"  /* 16px → body medium font-size */
  },
  "background": {
    "@": {
      "value": "{color.black.@.value}"  /* #000 → black color token */
    }
  },
  "padding": {
    "value": "{space.3.value}"  /* 12px → space.3 token; combined 12px 16px use largest */
  },
  "radius": {
    "value": "{radius.md.value}"  /* 8px → medium radius token */
  }
}
```

#### 8.5 — Output B: CSS Rules to app.override.css

**Append to:** `src/main/webapp/design-tokens/app.override.css`

**Purpose:** Convert the variant mapping into CSS custom properties (CSS variables) that components can consume.

**Relationship to JSON:**
- JSON `mapping` → CSS custom properties
- Each property in mapping becomes a `--wm-<component>-<property>` variable
- States (hover, focus, active, disabled) become `:hover`, `:focus`, `:active`, `[disabled]` selectors

**CSS Output Format:**

```css
/* ============================================================
 * Component Variants (migrated from legacy theme / app CSS)
 * ============================================================ */

/* --- Button variants --- */
.wm-app .dark-btn.app-button {
  --wm-btn-background: var(--wm-color-black);
  --wm-btn-color: var(--wm-color-background);
  --wm-btn-font-size: var(--wm-label-large-font-size);
  --wm-btn-font-family: var(--wm-label-large-font-family);
  --wm-btn-font-weight: var(--wm-label-large-font-weight);
  --wm-btn-line-height: var(--wm-label-large-line-height);
  --wm-btn-letter-spacing: var(--wm-label-large-letter-spacing);
  --wm-btn-text-transform: none;
  --wm-btn-border-color: var(--wm-color-surface-container-highest);
  --wm-btn-cursor: pointer;
  --wm-btn-radius: var(--wm-radius-sm);
  --wm-btn-padding: var(--wm-space-0) var(--wm-space-6);
  --wm-btn-min-width: auto;
  --wm-btn-min-height: var(--wm-space-10);
  --wm-btn-gap: var(--wm-space-2);
  --wm-btn-shadow: none;
  --wm-btn-icon-size: var(--wm-icon-size-md);
  --wm-btn-state-layer-color: var(--wm-color-on-surface);
}

.wm-app .dark-btn.app-button:hover,
.wm-app .dark-btn.app-button:hover::before,
.wm-app .dark-btn.app-button.hover::before {
  --wm-btn-state-layer-opacity: var(--wm-opacity-hover);
}

.wm-app .dark-btn.app-button:focus,
.wm-app .dark-btn.app-button:focus::before,
.wm-app .dark-btn.app-button.focus::before {
  --wm-btn-state-layer-opacity: var(--wm-opacity-focus);
}

.wm-app .dark-btn.app-button:active,
.wm-app .dark-btn.app-button:active::before,
.wm-app .dark-btn.app-button:active:hover::before,
.wm-app .dark-btn.app-button:active.focus::before {
  --wm-btn-state-layer-opacity: var(--wm-opacity-active);
}

.wm-app .dark-btn.app-button[disabled] {
  --wm-btn-color: var(--wm-color-on-surface);
  --wm-btn-background: var(--wm-color-surface-container-highest);
  --wm-btn-border-color: var(--wm-color-surface-container-highest);
  --wm-btn-opacity: 0.38;
  --wm-btn-shadow: none;
  --wm-btn-cursor: not-allowed;
}
```

**CSS Variable Naming Convention:**

| Element | Format | Example |
|---|---|---|
| Component prefix | `--wm-<component>-` | `--wm-btn-`, `--wm-label-`, `--wm-anchor-` |
| Property name | `--wm-<component>-<property>` | `--wm-btn-background`, `--wm-btn-color` |
| Nested property | `--wm-<component>-<parent>-<child>` | `--wm-btn-border-color`, `--wm-label-text-decoration` |
| State variant | `:state` pseudo-class | `:hover`, `:focus`, `:active`, `[disabled]` |

**Conversion Examples (JSON → CSS):**

| JSON Mapping | CSS Variable | CSS Rule |
|---|---|---|
| `"color": { "@": { "value": "{color.primary.@.value}" } }` | `--wm-btn-color` | `--wm-btn-color: var(--wm-color-primary);` |
| `"background": { "@": { "value": "{color.black.@.value}" } }` | `--wm-btn-background` | `--wm-btn-background: var(--wm-color-black);` |
| `"font-size": { "value": "{body.medium.font-size.value}" }` | `--wm-btn-font-size` | `--wm-btn-font-size: var(--wm-body-medium-font-size);` |
| `"states": { "hover": { "color": ... } }` | `--wm-btn-color` in `:hover` | `.btn:hover { --wm-btn-color: ...; }` |

**State Pseudo-Classes Mapping:**

| State | CSS Selector | Applied When |
|---|---|---|
| `hover` | `:hover`, `:hover::before`, `.hover::before` | User hovers over element |
| `focus` | `:focus`, `:focus::before`, `.focus::before` | Element receives keyboard focus |
| `active` | `:active`, `:active::before`, `:active:hover::before`, `:active.focus::before` | Element is pressed/clicked |
| `disabled` | `[disabled]` | Element has disabled attribute |

#### 8.6 — Token Reference Mapping (CSS Property → Foundation Token)

**Based on foundation CSS button structure (`button.less`), map CSS properties to foundation tokens:**

| CSS Property | Foundation Token | JSON Value | CSS Variable |
|---|---|---|---|
| **color** | Primary/semantic color | `{color.primary.@.value}` | `var(--wm-btn-color)` |
| **background-color** | Surface color | `{color.surface.@.value}` | `var(--wm-btn-background)` |
| **font-size** | Label scale size | `{label.large.font-size.value}` | `var(--wm-btn-font-size)` |
| **font-family** | Label scale family | `{label.large.font-family.value}` | `var(--wm-btn-font-family)` |
| **font-weight** | Label scale weight | `{label.large.font-weight.value}` | `var(--wm-btn-font-weight)` |
| **line-height** | Label scale height | `{label.large.line-height.value}` | `var(--wm-btn-line-height)` |
| **letter-spacing** | Label scale spacing | `{label.large.letter-spacing.value}` | `var(--wm-btn-letter-spacing)` |
| **border-radius** | Radius scale | `{radius.md.value}` | `var(--wm-btn-radius)` |
| **padding** | Space scale(s) | `{space.2.value} {space.4.value}` | `var(--wm-btn-padding)` |
| **height** | Space scale | `{space.10.value}` | `var(--wm-btn-height)` |
| **border-color** | Semantic color | `{color.surface.container.highest.@.value}` | `var(--wm-btn-border-color)` |
| **border-width** | Literal value | `1px` | `var(--wm-btn-border-width)` |
| **border-style** | Literal value | `solid` | `var(--wm-btn-border-style)` |
| **box-shadow** | Literal value | `none` | `var(--wm-btn-shadow)` |
| **gap** | Space scale | `{space.2.value}` | `var(--wm-btn-gap)` |
| **cursor** | Literal value | `pointer` | `var(--wm-btn-cursor)` |
| **opacity** | Literal value | `1` | `var(--wm-btn-opacity)` |

**Color Value Mapping:**

| CSS Value | Semantic Meaning | Foundation Token | JSON Reference |
|---|---|---|---|
| `#2294ef` | Primary brand color | Primary | `{color.primary.@.value}` |
| `#fff`, `#ffffff` | White background | White | `{color.white.@.value}` |
| `#000`, `#000000` | Black background | Black | `{color.black.@.value}` |
| `#f44336` | Error/danger state | Error | `{color.error.@.value}` |
| `#4caf50` | Success state | Success | `{color.success.@.value}` |
| `#ff9800` | Warning state | Warning | `{color.warning.@.value}` |
| `#2196f3` | Info state | Info | `{color.info.@.value}` |
| `#e0e0e0` | Disabled/border | Surface container | `{color.surface.container.highest.@.value}` |

**Spacing Value Mapping (px → Foundation Token):**

| CSS Value | Space Token | JSON Reference | Use Case |
|---|---|---|---|
| `0px` | space.0 | `{space.0.value}` | No space |
| `4px` | space.1 | `{space.1.value}` | Extra small spacing |
| `8px` | space.2 | `{space.2.value}` | Small spacing |
| `12px` | space.3 | `{space.3.value}` | Medium-small spacing |
| `16px` | space.4 | `{space.4.value}` | Medium spacing |
| `20px` | space.5 | `{space.5.value}` | Medium-large spacing |
| `24px` | space.6 | `{space.6.value}` | Large spacing |
| `32px` | space.8 | `{space.8.value}` | Extra large spacing |
| `40px` | space.10 | `{space.10.value}` | XXL spacing |

**Border Radius Mapping (px → Foundation Token):**

| CSS Value | Radius Token | JSON Reference | Use Case |
|---|---|---|---|
| `2px`, `4px` | radius.sm | `{radius.sm.value}` | Small border radius |
| `8px` | radius.md | `{radius.md.value}` | Medium border radius |
| `12px`, `16px` | radius.lg | `{radius.lg.value}` | Large border radius |
| `50%`, `999px` | radius.circle | `{radius.circle.value}` | Circular/fully rounded |

**Font Scale Mapping (Label & Body Scales):**

| Scale | Font Size | Font Weight | Line Height | Example Use |
|---|---|---|---|---|
| `label.small` | `{label.small.font-size.value}` | `{label.small.font-weight.value}` | `{label.small.line-height.value}` | Captions, hints |
| `label.medium` | `{label.medium.font-size.value}` | `{label.medium.font-weight.value}` | `{label.medium.line-height.value}` | Normal labels |
| `label.large` | `{label.large.font-size.value}` | `{label.large.font-weight.value}` | `{label.large.line-height.value}` | Emphasis, headings |
| `body.small` | `{body.small.font-size.value}` | `{body.small.font-weight.value}` | `{body.small.line-height.value}` | Small body text |
| `body.medium` | `{body.medium.font-size.value}` | `{body.medium.font-weight.value}` | `{body.medium.line-height.value}` | Regular body text |
| `body.large` | `{body.large.font-size.value}` | `{body.large.font-weight.value}` | `{body.large.line-height.value}` | Large body text |

**Icon Size Mapping:**

| CSS Value | Icon Size Token | JSON Reference | Use Case |
|---|---|---|---|
| `16px` | icon.size.sm | `{icon.size.sm.value}` | Small icons |
| `24px` | icon.size.md | `{icon.size.md.value}` | Medium icons (default) |
| `32px` | icon.size.lg | `{icon.size.lg.value}` | Large icons |
| `48px` | icon.size.xl | `{icon.size.xl.value}` | Extra large icons |

**Opacity/State Mapping:**

| CSS Value | State Token | JSON Reference | When Applied |
|---|---|---|---|
| `0.38` | opacity.disabled | `{opacity.disabled.value}` | When element is disabled |
| Hover overlay | opacity.hover | `{opacity.hover.value}` | On mouse hover |
| Focus overlay | opacity.focus | `{opacity.focus.value}` | On keyboard focus |
| Active overlay | opacity.active | `{opacity.active.value}` | When being clicked/pressed |

---

### STEP 9 · Report Extracted Variants

**Report what was extracted (source files remain intact):**

```
Component Variants Extracted (Basic Components):
  ✓ button — 3 variants: (dark-btn, outline-btn, ghost-btn)
  ✓ label — 2 variants: (text-ellipsis, ellipsis-lg)
  ✓ message — 1 variant: (success-alert)

Custom Utility Classes Extracted:
  ✓ card-mobile-img-wrapper
  ✓ badge-custom
  ✓ text-underline

Source Files Scanned:
  • src/main/webapp/themes/default/style.css — 5 rules migrated
  • src/main/webapp/app.css — 3 rules migrated

Output Files Created:
  ✓ src/main/webapp/design-tokens/overrides/components/button/button.json
  ✓ src/main/webapp/design-tokens/overrides/components/label/label.json
  ✓ src/main/webapp/design-tokens/app.override.css (CSS rules appended)

NOTE: Source CSS files remain intact with original content preserved.
```

---

## Token Extraction Rules

### Color Tokens

| Legacy Pattern | Foundation Token | Example |
|---|---|---|
| `*primary*` | `--wm-color-primary` | `--my-primary: #FF7250` |
| `*secondary*` | `--wm-color-secondary` | `--brand-secondary: #656DF9` |
| `*error*`, `*danger*` | `--wm-color-error` | `--error-color: #F44336` |
| `*success*`, `*check*` | `--wm-color-success` | `--success-color: #5AC588` |
| `*warning*` | `--wm-color-warning` | `--warning-color: #FFC107` |
| `*info*` | `--wm-color-info` | `--info-color: #2196F3` |
| `*surface*` | `--wm-color-surface` | `--bg-color: #FFFFFF` |
| `*text*`, `*on-surface*` | `--wm-color-on-surface` | `--text-color: #35363B` |

### Typography Tokens

| Legacy Pattern | Foundation Token | Example |
|---|---|---|
| Heading font | `--wm-font-family-brand` | `--heading-font: 'Segoe UI', sans-serif` |
| Body font | `--wm-font-family-plain` | `--body-font: Arial, sans-serif` |
| H1 size | `--wm-h1-font-size` | `--h1-size: 32px` |
| H1 weight | `--wm-h1-font-weight` | `--h1-weight: 700` |
| Body size | `--wm-font-size-base` | `--body-size: 14px` |
| Body weight | `--wm-font-weight-normal` | `--body-weight: 400` |

### Spacing Tokens

| Legacy Pattern | Foundation Token | Example |
|---|---|---|
| Base gap (8px) | `--wm-gap-base` | `--gap: 8px` |
| Small gap (4px) | `--wm-gap-1` | `--gap-sm: 4px` |
| Medium gap (8px) | `--wm-gap-2` | `--gap-md: 8px` |
| Large gap (16px) | `--wm-gap-3` | `--gap-lg: 16px` |
| Base margin | `--wm-margin-base` | `--margin: 16px` |
| Base padding | `--wm-padding-base` | `--padding: 16px` |
| Small radius | `--wm-radius-sm` | `--radius-sm: 4px` |
| Medium radius | `--wm-radius-md` | `--radius-md: 8px` |
| Large radius | `--wm-radius-lg` | `--radius-lg: 16px` |

---

## Edge Cases & Error Handling

### Missing style.css

| Scenario | Behavior | Solution |
|---|---|---|
| File missing | STEP 1 validation fails | Create `style.css` with empty `:root {}` |
| .less file instead | File not found error | Compile `.less` to `.css` first |

### Empty :root Variables

| Scenario | Behavior | Solution |
|---|---|---|
| No `:root` defined | Use Strategy 2 (property extraction) | Extracts from selectors automatically |
| Mixed variables | Use both strategies | Combines CSS vars + properties |

### CSS Variable References

| Scenario | Example | Handling |
|---|---|---|
| Valid reference | `var(--primary-color)` | Maps to foundation name |
| Undefined reference | `var(--does-not-exist)` | Preserved as-is (browser fallback) |
| Circular reference | A → B → A | Reported as warning |

### Duplicate Tokens

| Scenario | Behavior | Solution |
|---|---|---|
| Same var appears twice | First occurrence kept | Subsequent ones discarded |
| Reported in verbose | Shows count of duplicates | User can fix source CSS |

---

## Output Example

**Input: style.css**

```css
:root {
  --primary-color: #FF7250;
  --secondary-color: #656DF9;
  --font-family: 'Roboto', sans-serif;
  --gap-base: 8px;
}
```

**Output: app.override.css**

```css
/**
 * Design Token Overrides — Migrated from legacy theme
 * Theme: default
 * Source: /path/to/project/src/main/webapp/themes/default/style.css
 */

@import url('https://fonts.googleapis.com/css2?family=Roboto:wght@300;400;500;600;700&display=swap');

:root {
  /* Color Overrides */
  --wm-color-primary: #FF7250;
  --wm-color-secondary: #656DF9;

  /* Spacing Overrides */
  --wm-gap-base: 8px;

  /* Typography Overrides */
  --wm-font-family-brand: 'Roboto', sans-serif;
}
```

---

## Quick Reference: Component Variant Creation

**When you find a custom CSS variant, follow this process:**

### 1. Identify Component & Variant Name

**From CSS rule:**
```css
.wm-app .dark-btn.app-button {
  color: #2294ef;
  background: #000;
}
```

**Extract:**
- Component: `button` (from `.app-button`)
- Variant name: `dark_btn` (from `.dark-btn`, converted to snake_case)

### 2. Create JSON File Path

```
src/main/webapp/design-tokens/overrides/components/button/button.json
```

### 3. Map CSS Properties to Token References

**CSS Property → Token Reference Conversion:**

| CSS Property | Value | Foundation Token | JSON Mapping |
|---|---|---|---|
| `color` | `#2294ef` | primary blue color | `"color": { "@": { "value": "{color.primary.@.value}" } }` |
| `background` | `#000` | black color | `"background": { "@": { "value": "{color.black.@.value}" } }` |
| `font-size` | `16px` | body medium | `"font-size": { "value": "{body.medium.font-size.value}" }` |
| `padding` | `12px 16px` | space.3 (largest value) | `"padding": { "value": "{space.3.value}" }` |
| `border-radius` | `8px` | radius.md | `"radius": { "value": "{radius.md.value}" }` |

### 4. Create JSON Structure

**Basic Template:**

```json
{
  "button": {
    "appearances": {
      "dark_btn": {
        "mapping": {
          "color": { "@": { "value": "{color.primary.@.value}" } },
          "background": { "@": { "value": "{color.black.@.value}" } },
          "font-size": { "value": "{body.medium.font-size.value}" },
          "padding": { "value": "{space.3.value}" },
          "radius": { "value": "{radius.md.value}" }
        }
      }
    },
    "meta": {
      "appearances": {
        "dark_btn": {
          "source": "user"
        }
      }
    }
  }
}
```

### 5. Add State Definitions (if applicable)

**If CSS has hover/focus/active states:**

```css
.wm-app .dark-btn.app-button:hover {
  background: #1a1a1a;
  color: #fff;
}
```

**Add to JSON:**

```json
"states": {
  "hover": {
    "background": { "@": { "value": "{color.black.darken.value}" } },
    "color": { "@": { "value": "{color.white.@.value}" } }
  }
}
```

### 6. Generate CSS Variables

**Auto-generate CSS rules for app.override.css:**

```css
.wm-app .dark-btn.app-button {
  --wm-button-color: var(--wm-color-primary);
  --wm-button-background: var(--wm-color-black);
  --wm-button-font-size: var(--wm-body-medium-font-size);
  --wm-button-padding: var(--wm-space-3);
  --wm-button-radius: var(--wm-radius-md);
}

.wm-app .dark-btn.app-button:hover {
  --wm-button-background: var(--wm-color-black-darken);
  --wm-button-color: var(--wm-color-white);
}
```

### Common Token Reference Patterns

**For different property types:**

| Type | Value | Token Reference | Example |
|---|---|---|---|
| **Primary Color** | `#2294ef`, `rgb(34, 148, 239)` | `{color.primary.@.value}` | `--btn-color: var(--wm-color-primary)` |
| **Neutral Colors** | `#000`, `#fff`, `#ccc` | `{color.black.@.value}`, `{color.white.@.value}` | `--bg: var(--wm-color-black)` |
| **Semantic Colors** | Error, success, warning | `{color.error.@.value}`, `{color.success.@.value}` | `--error-bg: var(--wm-color-error)` |
| **Spacing** | `4px`, `8px`, `16px`, `24px` | `{space.1.value}`, `{space.2.value}`, `{space.3.value}`, `{space.4.value}` | `--padding: var(--wm-space-3)` |
| **Border Radius** | `4px`, `8px`, `16px` | `{radius.sm.value}`, `{radius.md.value}`, `{radius.lg.value}` | `--radius: var(--wm-radius-md)` |
| **Font Family** | Arial, Roboto | `{body.medium.font-family.value}` | `--font: var(--wm-body-medium-font-family)` |
| **Font Size** | `14px`, `16px`, `18px` | `{body.small.font-size.value}`, `{body.medium.font-size.value}`, `{body.large.font-size.value}` | `--size: var(--wm-body-medium-font-size)` |
| **Font Weight** | `400`, `500`, `700` | `{body.medium.font-weight.value}` | `--weight: var(--wm-body-medium-font-weight)` |
| **Line Height** | `20px`, `24px` | `{body.medium.line-height.value}` | `--height: var(--wm-body-medium-line-height)` |
| **Letter Spacing** | `0.5px` | `{body.medium.letter-spacing.value}` | `--spacing: var(--wm-body-medium-letter-spacing)` |
| **Icon Size** | `16px`, `24px`, `32px` | `{icon.size.sm.value}`, `{icon.size.md.value}`, `{icon.size.lg.value}` | `--icon-size: var(--wm-icon-size-md)` |
| **Opacity** | `0.5`, `0.38` | `{opacity.hover.value}`, `{opacity.focus.value}`, `{opacity.active.value}` | `--opacity: var(--wm-opacity-hover)` |

### Nested Property Examples

**For compound CSS properties:**

```json
/* text-decoration */
"text": {
  "decoration": {
    "@": { "value": "underline" }
  }
}

/* border-color */
"border": {
  "color": {
    "@": { "value": "{color.primary.@.value}" }
  }
}

/* icon-size */
"icon": {
  "size": {
    "value": "{icon.size.md.value}"
  }
}
```

---

## FAQ

| Question | Answer |
|---|---|
| **Will source CSS files be modified?** | No. Source files remain intact. Only output files are created/appended. |
| **Can I re-run the skill?** | Yes. STEP 2 will ask about foundation CSS installation each time. |
| **What if app.override.css already exists?** | Tokens are appended with a separator comment to preserve prior customizations. |
| **How do I know which tokens were mapped?** | Use `--verbose` flag to see detailed mapping report. |
| **Can I extract from multiple source files?** | Yes. STEP 8 scans style.css, app.css, and page CSS in order. |
| **Are advanced components (input, data) supported?** | Not in this version. Only basic components are extracted. Future phases will add support. |

---

## Notes

- **Foundation reference:** `@wavemaker/foundation-css` npm package provides semantic token standards
- **Semantic mapping:** Infers meaning from variable names (e.g., `--my-primary` → `--wm-color-primary`)
- **Incremental updates:** Tokens appended to existing `app.override.css` if file exists
- **Font handling:** Custom fonts require user approval before import URL is added
- **Basic components only:** Current version extracts variants for button, label, message, search, progress, icon, anchor, picture, bottomsheet, spinner, skeleton
- **Future phases:** Input, data, container, dialogs, navigation, chart, and other advanced component types
