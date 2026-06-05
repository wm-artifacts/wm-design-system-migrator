# Rule 03: Attribute & Class Enhancements — UI Elements

## Overview

Individual visual components now strictly bind to a variant identifier and specific class name combinations.

---

## Buttons (`<wm-button>`, `<wm-form-action>`)

- **NDS Pattern:** Utilized standard Bootstrap contextual classes (e.g., `class="btn-default"` or `class="btn-primary"`).
- **DS Pattern:** Requires explicit mapping to `"filled"` or `"outline"` variants.

### Action

1. Add `btn-filled` to the class attribute.
2. Create a `variant` attribute mapped to the class context:
   - For `btn-default` → add `variant="filled:default"`
   - For `btn-primary` → add `variant="filled:primary"`

---

## Typography (`<wm-label>`)

- **NDS Pattern:** `class="p"` or `class="h5"`, along with `type="p"`.
- **DS Pattern:** Requires a `variant` attribute bound to the typographical class.

### Action

Read the class size (`p`, `h5`, etc.) and inject the matching variant:
- `class="p"` → `variant="default:p"`
- `class="h5"` → `variant="default:h5"`

---

## Icons (`<wm-icon>`)

- **NDS Pattern:** Basic icons `<wm-icon name="icon1"></wm-icon>`.
- **DS Pattern:** Requires size definition via both classes and variants.

### Action

Add specific Font Awesome size classes and map the variant to match:
- Add `class="fa-xs"`
- Add `variant="default:xs"`

---

## Images/Pictures (`<wm-picture>`)

- **NDS Pattern:** Handled shapes natively via `shape="circle"`.
- **DS Pattern:** Replaces the `shape` property with CSS classes and sets a crop mode.

### Action

1. Remove `shape="circle"`.
2. Inject `resizemode="cover"`.
3. Add `class="img-circle img-rounded"`.
4. Add `variant="default:rounded"`.

---

## Tables (`<wm-table>`)

- **NDS Pattern:** Basic instantiation `<wm-table editmode="inline"...>`
- **DS Pattern:** Adds base UI variant.

### Action

Append `variant="default"` to the `<wm-table>` tag.
