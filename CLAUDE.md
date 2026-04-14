# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A single-file business card color builder for the **Beaumont Library District (BLD)**. Users customize card colors via dropdown pickers and save schemes to localStorage. Saved schemes appear below the builder as 3D flip cards in a responsive grid.

## Development

No build step, no dependencies, no framework. Open `index.html` directly in a browser. Refresh to see changes.

## Architecture

Everything lives in **index.html** (~1335 lines total):

1. **CSS** (~648 lines): Card system (700x400 at 1.75:1 ratio), builder UI (pickers, buttons), saved section (CSS 3D flip cards in auto-fill grid), responsive breakpoints at 1480px/740px/380px, print styles
2. **HTML** (~80 lines): Builder preview (front + back card with `id="custom-*"` elements), pickers container, action buttons, saved-section grid
3. **JavaScript** (~600 lines): Brand palette, background options, state management, localStorage persistence, URL hash sharing, scheme name generator

### Key Patterns

- **Builder preview** uses `id`-based selectors (e.g., `#custom-front`, `#custom-strip`) -- there's only one preview card
- **Saved cards** use class-based selectors scoped to each card element (e.g., `cardEl.querySelector('.saved-flip-front .accent-strip')`) -- multiple cards coexist
- These are two parallel rendering paths: `applyState()` updates the builder preview via `getElementById`, `applyStateToCard(cardEl, s)` updates a saved card via `querySelector` within that card's DOM subtree
- Saved cards render at full 700x400 size then `transform: scale(0.5)` inside 350x200 containers
- Logo PNGs live in `BLD Logo Suite (2)/BLD Logo Suite/Horizontal/PNG Transparent Background/` -- the path has spaces and parentheses, always use the `logoBase` variable
- `LOGO_MAP` translates brand color names to logo filenames (they don't always match, e.g., `'Lavender Purple'` maps to `'Lavender Fields Purple'`)

### Dark Background Behavior

When `BG_OPTIONS[frontBg].dark` is true: front logo forces `White.png`, tagline uses `lighten(c2, 80)`, and wave SVGs use different alpha values. The back face always uses a light background (`#FDFAF5` for dark themes, `#FDFCFA` for light).

### State Model

```
{ frontBg, logo, accent1, accent2, textColor, footer }
```

`accent1` drives: accent strip primary, icons, job title, divider. `accent2` drives: tagline, labels, name label. Not every original BLD color mapping is reproducible through this 2-accent model.

### Persistence

- **localStorage** key: `bld-card-versions`. Array of `{ name, date, state }` objects. Max 20 saved schemes.
- **URL hash sharing**: `#custom=` + URI-encoded JSON array of the 6 state values in order. Loaded on page init.

### Brand Palette

The canonical palette is the `BRAND` object in JS. Key colors: San Gorgonio Blue (#15424A), Sage (#5A7A6A), Rust (#8B5E3C), Lavender Purple (#7A5480). The default state matches the Original BLD Website Theme (mybld.org).

### Fonts

Three Google Fonts loaded via CDN: `DM Serif Display` (headings/names), `News Cycle` (body text), `Libre Franklin` (labels/taglines/subtitles).
