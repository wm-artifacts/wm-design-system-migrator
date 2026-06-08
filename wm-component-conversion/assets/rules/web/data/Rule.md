# Rule 01: Structural Replacements — Forms & Live Filters

## Overview

The most significant DOM structural change involves removing legacy grid systems in favor of responsive flex containers.

---

## Form & LiveFilter Wrappers

- **Pattern Change:** The parent `<wm-form>` and `<wm-livefilter>` tags must now explicitly define responsive row constraints based on their original grid column count.
- **Action:** If the original layout had `columns="3"`, inject `itemsperrow="xs-3 sm-3 md-3 lg-3"` directly into the `<wm-form>` or `<wm-livefilter>` tag.

---

## Removing `<wm-layoutgrid>`

- **NDS Pattern:** Used `<wm-layoutgrid>`, `<wm-gridrow>`, and `<wm-gridcolumn>` to structure fields.
- **DS Pattern:** Flattens this hierarchy into a single Flexbox `<wm-container>`.

### Action

1. Delete `<wm-layoutgrid>`, `<wm-gridrow>`, and `<wm-gridcolumn>` wrappers.
2. Wrap the child input fields (`wm-form-field`, `wm-filter-field`) in a newly structured container:

```html
<!-- Replace Grid Hierarchy With: -->
<wm-container direction="row" alignment="top-left" wrap="true" width="fill" columns="{original_column_count}" name="containerX" class="app-container-default" variant="default">
    <!-- Fields go here -->
</wm-container>
```

---

## `<wm-list>` and `<wm-listtemplate>`

- **NDS Pattern:** Relied on `listclass="list-group"` and internal components having `class="media-left"`, `class="media-body"`.
- **DS Pattern:** Uses explicit flex properties.

### Action on `<wm-list>`

1. Remove `listclass="list-group"`.
2. Inject Flex attributes:
   - `direction="column"`
   - `alignment="top-left"`
   - `gap="4"`
   - `wrap="false"`

### Action on `<wm-listtemplate>`

1. Inject Flex attributes:
   - `direction="row"` (or `column` depending on list type)
   - `alignment="top-left"`
   - `gap="4"`
   - `width="fill"`

---

## Tables (`<wm-table>`)

- **NDS Pattern:** Basic instantiation `<wm-table editmode="inline"...>`
- **DS Pattern:** Adds base UI variant.

### Action

Append `variant="default"` to the `<wm-table>` tag.

---
