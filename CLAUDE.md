# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A single-file business card color builder for the **Beaumont Library District (BLD)**. Users customize card colors via dropdown pickers and save schemes to localStorage. Saved schemes appear below the builder as 3D flip cards in a responsive grid.

## Development

No build step, no dependencies, no framework. Open `index.html` directly in a browser. Refresh to see changes. No tests, no linter.

External CDN libraries: `html2canvas` (card-to-image capture) and `jsPDF` (print PDF export with crop marks and bleed).

## Architecture

Everything lives in **index.html** (~1879 lines total):

1. **CSS** (~760 lines): Card system (700x400 at 1.75:1 ratio), builder UI (pickers, buttons, export dropdown), saved section (CSS 3D flip cards in auto-fill grid), responsive breakpoints at 1480px/740px/380px, print styles
2. **HTML** (~92 lines): Builder preview (front + back card with `id="custom-*"` elements), picker containers under each card, action buttons, saved-section grid with flip-all/import/export controls
3. **JavaScript** (~1020 lines): Brand palette, background options, WCAG contrast checking, state management, picker UI construction, localStorage persistence, URL hash sharing, scheme name generator, PDF export with crop marks

### Key Patterns

- **Builder preview** uses `id`-based selectors (e.g., `#custom-front`, `#custom-strip`) -- there's only one preview card
- **Saved cards** use class-based selectors scoped to each card element (e.g., `cardEl.querySelector('.saved-flip-front .accent-strip')`) -- multiple cards coexist
- These are two parallel rendering paths: `applyState()` updates the builder preview via `getElementById`, `applyStateToCard(cardEl, s)` updates a saved card via `querySelector` within that card's DOM subtree
- Saved cards render at full 700x400 size then `transform: scale(0.5)` inside 350x200 containers
- Logo PNGs live in `BLD Logo Suite (2)/BLD Logo Suite/Horizontal/PNG Transparent Background/` -- the path has spaces and parentheses, always use the `logoBase` variable
- `LOGO_MAP` translates brand color names to logo filenames (they don't always match, e.g., `'Lavender Purple'` maps to `'Lavender Fields Purple'`)
- The Name and Job Title fields on the back face are `contenteditable` with auto-scaling (max 2 lines, font shrinks from 2.1rem to 0.8rem)

### WCAG Contrast System

Each picker has a `CONTRAST_RULES` entry defining a minimum contrast ratio and enforcement mode:
- `mode: 'hard'` -- the randomizer will only pick colors that pass (logos, text, back-face labels)
- `mode: 'warn'` -- swatch gets a `.contrast-warn` CSS ring, but the color remains selectable (accent strip, footer)

`BG_CHECK_COLORS` maps each background option to its worst-case hex for contrast math. `passesContrast()` and `contrastRatio()` implement WCAG relative luminance checking.

When `frontBg` changes, all picker dropdowns are rebuilt via `refreshSwatchUI()` because contrast thresholds shift between light and dark backgrounds. Individual picker changes only call `updateContrastWarnings()`.

### Dark Background Behavior

When `BG_OPTIONS[frontBg].dark` is true: front logo forces `White.png` (picker locked/disabled), tagline uses `lighten(c2, 80)`, and wave SVGs use different alpha values. The back face always uses a light background (`#FDFAF5` for dark themes, `#FDFCFA` for light).

### State Model

```js
{
  frontBg,                     // Background option key (from BG_OPTIONS)
  frontLogo, frontAccent1, frontAccent2,  // Front face: logo + 2 accents (from BRAND/LOGO_COLORS)
  backLogo, backAccent1, backAccent2,     // Back face: logo + 2 accents
  backText, backFooter                    // Back face: text color + footer wave color
}
```

9 keys total. Front and back have independent pickers. `frontAccent1` drives: accent strip primary, divider rule. `frontAccent2` drives: tagline text. `backAccent1` drives: top bar, job title, icons, divider. `backAccent2` drives: contact labels. `backText` drives: name, contact text. `backFooter` drives: footer wave fill.

### Persistence

- **localStorage** key: `bld-card-versions`. Array of `{ name, date, state }` objects. Max 20 saved schemes. A migration on load clears pre-v2 saves (old 6-key state format without `frontLogo`).
- **URL hash sharing**: `#custom=` + URI-encoded JSON array of the 9 state values in order. Loaded on page init.
- **Collection export/import**: JSON file download/upload of the full saved array.

### PDF Export

Uses `html2canvas` to capture front and back at 3x scale with 0.125" bleed, then `jsPDF` to compose pages with crop marks, registration marks, and a slug line. Output is a 2-page landscape PDF at the print-ready trim size (3.5" x 2").

### Brand Palette

The canonical palette is the `BRAND` object in JS. Key colors: San Gorgonio Blue (#15424A), Sage (#5A7A6A), Rust (#8B5E3C), Lavender Purple (#7A5480). The default state matches the Original BLD Website Theme (mybld.org).

`LOGO_COLORS` is a filtered subset of `BRAND` -- only colors that have matching logo PNG files in `LOGO_MAP`.

### Fonts

Three Google Fonts loaded via CDN: `DM Serif Display` (headings/names), `News Cycle` (body text), `Libre Franklin` (labels/taglines/subtitles).

### Scheme Name Generator

`generateSchemeName()` builds evocative names from `COLOR_WORDS` (per-brand-color word lists), `BG_WORDS` (per-background word lists), and geographic `SUFFIXES`. Used as the default suggestion in the save prompt and as the PDF filename.
