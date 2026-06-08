# Rule 07: Advanced Component Migrations

## Overview

Advanced components (Carousel, Marquee, Login) require minimal direct transformation. The inner `<wm-picture>` components within carousels are automatically upgraded by the Basic rules (execution order 2).

---

## Carousel (`<wm-carousel>`)

- **NDS Pattern:** `<wm-carousel height="480" name="carousel1">` with `<wm-picture>` children.
- **DS Pattern:** Identical `<wm-carousel>` and `<wm-carousel-content>` wrappers; inner `<wm-picture>` elements gain `class="img-rounded"` and `variant="default:rounded"`.
- **Action:** No direct change to `<wm-carousel>` or `<wm-carousel-content>`. The Basic picture rule handles the `<wm-picture>` upgrade automatically.

---

## Marquee (`<wm-marquee>`)

- **NDS Pattern:** `<wm-marquee name="marquee1"></wm-marquee>`
- **DS Pattern:** Identical — no structural change required.
- **Action:** None.

---

## Login (`<wm-login>`)

- **NDS Pattern:** `<wm-login name="login1"></wm-login>`
- **DS Pattern:** Identical — no structural change required.
- **Action:** None.

---

## Script

Extracted by the skill assembler into `wm_comp_conv_tmp.py`. Edit this block to add or change
advanced-component conversion behaviour — the skill picks it up automatically on the next run.

Signature contract: `apply_advanced_rules(text) -> (text, counts_dict)`.
Use `parse_attrs`, `build_attrs`, and `merge_class` — they are injected by the assembler header.

```python
# Execution order: 7
# Components: wm-carousel, wm-marquee, wm-login
# Note: wm-picture elements inside carousels are handled by apply_basic_rules.

def apply_advanced_rules(text):
    counts = {}
    # No transformations required for advanced container components.
    # Inner wm-picture elements are upgraded by apply_basic_rules.
    return text, counts
```
