# Rule 04: Dialog Component Migrations

## Overview

Most dialog components adopt a standardized modal sizing pattern in the design system, requiring `class="modal-dialog modal-xs"` and `variant="default:xs"`. The `<wm-pagedialog>` is the only dialog that does not change.

---

## Alert Dialog (`<wm-alertdialog>`)

- **NDS Pattern:** `<wm-alertdialog on-ok="..." name="alertdialog1"></wm-alertdialog>`
- **DS Pattern:** `<wm-alertdialog on-ok="..." name="alertdialog1" class="modal-dialog modal-xs" variant="default:xs"></wm-alertdialog>`

### Action

1. Add `class="modal-dialog modal-xs"` (append to any existing classes).
2. Add `variant="default:xs"`.

---

## Confirm Dialog (`<wm-confirmdialog>`)

- **NDS Pattern:** `<wm-confirmdialog on-ok="..." on-cancel="..." name="confirmdialog1"></wm-confirmdialog>`
- **DS Pattern:** `<wm-confirmdialog ... name="confirmdialog1" class="modal-dialog modal-xs" variant="default:xs"></wm-confirmdialog>`

### Action

1. Add `class="modal-dialog modal-xs"`.
2. Add `variant="default:xs"`.

---

## Iframe Dialog (`<wm-iframedialog>`)

- **NDS Pattern:** `<wm-iframedialog on-ok="..." name="iframedialog1"></wm-iframedialog>`
- **DS Pattern:** `<wm-iframedialog on-ok="..." name="iframedialog1" class="modal-dialog modal-xs" variant="default:xs"></wm-iframedialog>`

### Action

1. Add `class="modal-dialog modal-xs"`.
2. Add `variant="default:xs"`.

---

## Login Dialog (`<wm-logindialog>`)

- **NDS Pattern:** `<wm-logindialog modal="false" caption="Login" ... name="logindialog1"></wm-logindialog>`
- **DS Pattern:** `<wm-logindialog modal="false" caption="Login" ... name="logindialog1" class="modal-dialog modal-xs" variant="default:xs"></wm-logindialog>`

### Action

1. Add `class="modal-dialog modal-xs"`.
2. Add `variant="default:xs"`.

> **Note:** Buttons inside `<wm-dialogactions>` receive `btn-filled` and `variant` via the Basic rules (execution order 2).

---

## Dialog (`<wm-dialog>`)

- **NDS Pattern:** `<wm-dialog dialogtype="design-dialog" modal="true" ... name="dialog1"></wm-dialog>`
- **DS Pattern:** `<wm-dialog ... name="dialog1" class="modal-dialog modal-xs" variant="default:xs"></wm-dialog>`

### Action

1. Add `class="modal-dialog modal-xs"`.
2. Add `variant="default:xs"`.

---

## Page Dialog (`<wm-pagedialog>`)

- **NDS Pattern:** `<wm-pagedialog on-ok="..." name="pagedialog1"></wm-pagedialog>`
- **DS Pattern:** Identical — no structural change required.
- **Action:** None.

---

## Script

Extracted by the skill assembler into `wm_comp_conv_tmp.py`. Edit this block to add or change
dialog-component conversion behaviour — the skill picks it up automatically on the next run.

Signature contract: `apply_dialogs_rules(text) -> (text, counts_dict)`.
Use `parse_attrs`, `build_attrs`, and `merge_class` — they are injected by the assembler header.

```python
# Execution order: 4
# Components: wm-alertdialog, wm-confirmdialog, wm-iframedialog, wm-logindialog, wm-dialog

DIALOG_TAGS = [
    'wm-alertdialog',
    'wm-confirmdialog',
    'wm-iframedialog',
    'wm-logindialog',
    'wm-dialog',
]

def apply_dialogs_rules(text):
    counts = {tag: 0 for tag in DIALOG_TAGS}

    def patch_dialog(m):
        tag, attr_str = m.group(1), m.group(2)
        attrs = parse_attrs(attr_str)
        if 'variant' not in attrs:
            attrs['class'] = merge_class(attrs.get('class', ''), 'modal-dialog modal-xs')
            attrs['variant'] = 'default:xs'
            counts[tag] += 1
        return f'<{tag} {build_attrs(attrs)}>'

    # wm-dialog uses \b to avoid matching wm-dialogactions
    tag_pattern = '|'.join(re.escape(t) for t in DIALOG_TAGS)
    text = re.sub(rf'<({tag_pattern})\b([^>]*)>', patch_dialog, text)
    return text, counts
```
