# Theme to Design System Conversion Skill

## Overview

The Theme to Design System Conversion Skill helps migrate legacy WaveMaker themes to the modern Design System architecture by:

* Extracting design tokens from the existing `style.css`
* Generating reusable global design tokens
* Converting frequently used component styles into component variants
* Migrating custom icon fonts (IcoMoon)
* Updating project configuration to consume Design System assets

---

# What This Skill Does

## 1. Design Token Extraction

The skill scans the legacy theme's `style.css` and extracts reusable design properties such as:

* Colors
* Typography
* Spacing

These properties are transformed into reusable Design System tokens and generated in `app.override.css`.

---

## 2. Component Variant Generation

The skill identifies frequently used component-specific styles and converts them into reusable Design System variants.

### Supported Components

The following components are analyzed during migration:

| Component    | Fallback Selector        |
| ------------ | ------------------------ |
| Accordion    | `.app-accordion`         |
| Anchor       | `.app-anchor`            |
| Badge        | `.app-badge`             |
| Button       | `.app-button`            |
| Button Group | `.app-button-group`      |
| Cards        | `.app-card`              |
| Container    | `.app-container-default` |
| Icon         | `.app-icon`              |
| Label        | `.app-label`             |
| List         | `.app-list`              |
| Page Header  | `.app-page-header`       |
| Panel        | `.app-panel`             |
| Picture      | `.app-picture`           |
| Popover      | `.app-popover`           |
| Tabs         | `.app-tabs`              |
| Tile         | `.app-tile`              |
| Wizard       | `.app-wizard`            |

### Variant Extraction Process

For each supported component, the migration:

1. Locates matching selectors in the legacy theme.
2. Extracts reusable visual properties.
3. Converts styles into Design System-compatible variants.
4. Preserves unsupported styles in `theme-style.css` as fallback overrides.

### Example

#### Legacy Theme

```css
.app-button {
   background: #1976d2;
   color: #ffffff;
   border-radius: 4px;
}

.app-card {
   background: #ffffff;
   box-shadow: 0 2px 8px rgba(0,0,0,.1);
}
```

#### Generated Design System Variant

```css
.wm-button.variant-primary {
   background: var(--wm-color-primary);
   color: var(--wm-color-on-primary);
}

.wm-card.variant-default {
   background: var(--wm-color-surface);
}
```

---

## 3. IcoMoon Icon Migration

Legacy themes often include custom icon fonts generated through IcoMoon.

The skill:

* Detects IcoMoon assets
* Extracts font files
* Copies assets into the project at:

```text
src/main/webapp/design-tokens/custom-icon/
├── icomoon.ttf
├── icomoon.woff
├── icomoon.woff2
└── icomoon.svg

icon.css
```

* Preserves existing icon mappings

---

## 4. Theme Style Integration

Instead of loading the entire legacy theme, only the required styling artifacts are preserved.

### theme-style.css

Contains:

* Frequently used component styles
* Variant mappings
* Theme-specific overrides

### icon.css

Contains:

* IcoMoon icon definitions
* Font-face declarations

---

## Generated Project Structure

```text
src/main/webapp/

├── design-tokens/
    └── app.override.css
    └──theme-style.css
    └── icon.css
├── custom-icon/
     ├── icomoon.ttf
     ├── icomoon.woff
     ├── icomoon.woff2
     └── icomoon.svg

└── index.html
```

---

## Migration Flow

```text
Legacy Theme
      │
      ▼
style.css Analysis
      │
      ├── Extract Colors
      ├── Extract Typography
      ├── Extract Spacing
      ├── Extract Component Styles
      └── Extract IcoMoon Assets
      │
      ▼
Generate Design Tokens
      │
      ▼
Create Component Variants
      │
      ▼
Copy Icon Assets
      │
      ▼
Update index.html
      │
      ▼
Design System Ready Project
```
