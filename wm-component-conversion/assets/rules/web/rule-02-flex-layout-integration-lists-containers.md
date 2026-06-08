# Rule 02: Flex Layout Integration — Lists & Containers

## Overview

Elements that previously relied on internal CSS classes to determine layout flow now require hardcoded Flex attributes.

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
