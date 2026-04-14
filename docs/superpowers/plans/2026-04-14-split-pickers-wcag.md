# Split Front/Back Pickers + WCAG Contrast Enforcement — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Separate the picker UI into two labeled rows (Front / Back) with independent color controls, add WCAG contrast ratio enforcement that hard-filters inaccessible options for text/logo pickers and warns on decorative pickers, and clear legacy localStorage data.

**Architecture:** Everything stays in `index.html`. The state model expands from 6 keys to 9. Two new utility functions (`relativeLuminance`, `contrastRatio`) power the WCAG logic. Picker construction becomes row-based with a `CONTRAST_RULES` table driving per-picker filtering. No new files, no build step, no dependencies.

**Tech Stack:** Vanilla HTML/CSS/JS, single file

---

### Task 1: Add WCAG contrast utility functions

**Files:**
- Modify: `index.html:811-815` (after `lighten()` function)

- [ ] **Step 1: Add `relativeLuminance` and `contrastRatio` functions after `lighten()`**

Insert after the closing brace of `lighten()` at line 815:

```js
function relativeLuminance(hex) {
  const r = parseInt(hex.slice(1,3),16) / 255;
  const g = parseInt(hex.slice(3,5),16) / 255;
  const b = parseInt(hex.slice(5,7),16) / 255;
  const sR = r <= 0.03928 ? r / 12.92 : Math.pow((r + 0.055) / 1.055, 2.4);
  const sG = g <= 0.03928 ? g / 12.92 : Math.pow((g + 0.055) / 1.055, 2.4);
  const sB = b <= 0.03928 ? b / 12.92 : Math.pow((b + 0.055) / 1.055, 2.4);
  return 0.2126 * sR + 0.7152 * sG + 0.0722 * sB;
}

function contrastRatio(hex1, hex2) {
  const l1 = relativeLuminance(hex1);
  const l2 = relativeLuminance(hex2);
  const lighter = Math.max(l1, l2);
  const darker = Math.min(l1, l2);
  return (lighter + 0.05) / (darker + 0.05);
}
```

- [ ] **Step 2: Verify by opening index.html in browser and running in dev console**

Run in browser console:
```js
contrastRatio('#15424A', '#FFFFFF')  // Should be ~9.1 (high contrast, dark on white)
contrastRatio('#C8B49B', '#FFFFFF')  // Should be ~1.7 (low contrast, Desert Sand on white)
contrastRatio('#C49A30', '#FFFFFF')  // Should be ~2.5 (low contrast, Citrus Gold on white)
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add WCAG contrast ratio utility functions"
```

---

### Task 2: Add BG_CHECK_COLORS lookup and CONTRAST_RULES table

**Files:**
- Modify: `index.html` (after `contrastRatio` function, before `let state`)

- [ ] **Step 1: Add `BG_CHECK_COLORS` — the worst-case background hex for each frontBg option**

Insert after the `contrastRatio` function:

```js
// Worst-case background hex for WCAG contrast checks per BG option
const BG_CHECK_COLORS = {
  'White / Cream':  '#FFFFFF',
  'Cool White':     '#FFFFFF',
  'Warm White':     '#FFFFFF',
  'Sage Tint':      '#FFFFFF',
  'Deep Teal':      '#0D2A30',
  'Dark Charcoal':  '#1A1614',
};

// Near-white back background for contrast checks
const BACK_BG_LIGHT = '#FDFCFA';
const BACK_BG_DARK  = '#FDFAF5';
```

- [ ] **Step 2: Add `CONTRAST_RULES` table — defines per-picker-key threshold and whether hard-filter or warn**

Insert after `BACK_BG_DARK`:

```js
// WCAG contrast rules per picker key
// mode: 'hard' = option disabled when failing, 'warn' = orange border on swatch
// bgSource: 'front' = check vs frontBg, 'back' = check vs back bg
const CONTRAST_RULES = {
  frontLogo:    { threshold: 3,   mode: 'hard', bgSource: 'front' },
  frontAccent1: { threshold: 3,   mode: 'warn', bgSource: 'front' },
  frontAccent2: { threshold: 4.5, mode: 'hard', bgSource: 'front' },
  backLogo:     { threshold: 3,   mode: 'hard', bgSource: 'back' },
  backAccent1:  { threshold: 3,   mode: 'hard', bgSource: 'back' },
  backAccent2:  { threshold: 4.5, mode: 'hard', bgSource: 'back' },
  backText:     { threshold: 4.5, mode: 'hard', bgSource: 'back' },
  backFooter:   { threshold: 3,   mode: 'warn', bgSource: 'back' },
};
```

- [ ] **Step 3: Add `LOGO_COLORS` — subset of BRAND for logo pickers (only colors with matching PNG files)**

Insert after `CONTRAST_RULES`:

```js
// Only BRAND colors that have matching logo PNG files
const LOGO_COLORS = Object.fromEntries(
  Object.keys(LOGO_MAP).map(k => [k, BRAND[k]])
);
```

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add BG_CHECK_COLORS, CONTRAST_RULES, and LOGO_COLORS"
```

---

### Task 3: Replace state model with 9-key split front/back

**Files:**
- Modify: `index.html` — `let state = { ... }` block (line ~802-809)

- [ ] **Step 1: Replace the state object**

Replace:
```js
let state = {
  frontBg:   'White / Cream',
  logo:      'San Gorgonio Blue',
  accent1:   'San Gorgonio Blue',
  accent2:   'Sage',
  textColor: 'San Gorgonio Blue',
  footer:    'San Gorgonio Blue',
};
```

With:
```js
let state = {
  frontBg:      'White / Cream',
  frontLogo:    'San Gorgonio Blue',
  frontAccent1: 'San Gorgonio Blue',
  frontAccent2: 'Sage',
  backLogo:     'San Gorgonio Blue',
  backAccent1:  'San Gorgonio Blue',
  backAccent2:  'Sage',
  backText:     'San Gorgonio Blue',
  backFooter:   'San Gorgonio Blue',
};
```

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "feat: expand state model from 6 keys to 9 (split front/back)"
```

---

### Task 4: Update `applyState()` to use new state keys

**Files:**
- Modify: `index.html` — `applyState()` function (lines ~817-869)

- [ ] **Step 1: Replace the full `applyState()` function**

Replace the entire `applyState()` function with:

```js
function applyState() {
  const bgOpt = BG_OPTIONS[state.frontBg];
  const isDark = bgOpt.dark;
  const fc1 = BRAND[state.frontAccent1];
  const fc2 = BRAND[state.frontAccent2];
  const bc1 = BRAND[state.backAccent1];
  const bc2 = BRAND[state.backAccent2];
  const txt = BRAND[state.backText];
  const ftr = BRAND[state.backFooter];

  // Front
  document.getElementById('custom-front').style.background = bgOpt.bg;
  document.getElementById('custom-strip').style.background = `linear-gradient(90deg, ${fc1}, ${lighten(fc1,40)}, ${fc2}, ${lighten(fc2,40)}, ${fc1})`;
  document.getElementById('custom-rule').style.background = `linear-gradient(90deg, ${fc1}, ${fc2})`;
  document.getElementById('custom-tagline').style.color = isDark ? lighten(fc2, 80) : fc2;

  // Front logo
  const logoFile = isDark ? 'White' : (LOGO_MAP[state.frontLogo] || state.frontLogo);
  document.getElementById('custom-front-logo').src = logoBase + logoFile + '.png';

  // Back logo
  const backLogoFile = LOGO_MAP[state.backLogo] || LOGO_MAP[state.frontLogo];
  if (backLogoFile) document.getElementById('custom-back-logo').src = logoBase + backLogoFile + '.png';

  // Waves
  const waveAlpha = isDark ? 0.05 : 0.04;
  document.getElementById('custom-wave1').setAttribute('fill', isDark ? `rgba(200,180,155,${waveAlpha})` : `${fc1}11`);
  document.getElementById('custom-wave2').setAttribute('fill', isDark ? `rgba(200,180,155,${waveAlpha * 0.7})` : `${fc1}09`);

  // Back bar
  document.getElementById('custom-bar').style.background = `linear-gradient(90deg, ${bc1}, ${lighten(bc1,40)}, ${bc2}, ${lighten(bc2,40)}, ${bc1})`;

  // Back text
  document.getElementById('custom-name').style.color = txt;
  document.getElementById('custom-jobtitle').style.color = bc1;
  document.getElementById('custom-identity').style.borderRight = `2px solid ${lighten(bc1, 180)}`;

  // Contact items
  document.querySelectorAll('#custom-contact .contact-icon').forEach(el => el.style.color = bc1);
  document.querySelectorAll('#custom-contact .contact-text').forEach(el => el.style.color = txt);
  document.querySelectorAll('#custom-contact .contact-label').forEach(el => el.style.color = bc2);
  document.querySelectorAll('#custom-contact .contact-text a').forEach(el => {
    el.style.color = txt;
    el.style.borderBottom = `1px solid ${txt}33`;
  });

  // Back + footer
  document.getElementById('custom-back').style.background = isDark ? '#FDFAF5' : '#FDFCFA';
  document.getElementById('custom-footer-wave').setAttribute('fill', ftr);
  document.querySelectorAll('#custom-socials .social-icon').forEach(el => el.style.color = 'rgba(255,255,255,0.8)');

  // Auto-scale editable name
  const nameEl = document.getElementById('custom-name');
  if (nameEl) autoScaleName(nameEl);
}
```

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "feat: update applyState() to use split front/back state keys"
```

---

### Task 5: Update `applyStateToCard()` to use new state keys

**Files:**
- Modify: `index.html` — `applyStateToCard()` function (lines ~888-941)

- [ ] **Step 1: Replace the full `applyStateToCard()` function**

Replace the entire function with:

```js
function applyStateToCard(cardEl, s) {
  const bgOpt = BG_OPTIONS[s.frontBg];
  if (!bgOpt) return;
  const isDark = bgOpt.dark;
  const fc1 = BRAND[s.frontAccent1];
  const fc2 = BRAND[s.frontAccent2];
  const bc1 = BRAND[s.backAccent1];
  const bc2 = BRAND[s.backAccent2];
  const txt = BRAND[s.backText];
  const ftr = BRAND[s.backFooter];
  if (!fc1 || !fc2 || !bc1 || !bc2 || !txt || !ftr) return;

  // Front
  const front = cardEl.querySelector('.card-front');
  front.style.background = bgOpt.bg;
  cardEl.querySelector('.saved-flip-front .accent-strip').style.background = `linear-gradient(90deg, ${fc1}, ${lighten(fc1,40)}, ${fc2}, ${lighten(fc2,40)}, ${fc1})`;
  cardEl.querySelector('.saved-flip-front .front-rule').style.background = `linear-gradient(90deg, ${fc1}, ${fc2})`;
  cardEl.querySelector('.saved-flip-front .front-tagline').style.color = isDark ? lighten(fc2, 80) : fc2;

  // Front logo
  const logoFile = isDark ? 'White' : (LOGO_MAP[s.frontLogo] || s.frontLogo);
  cardEl.querySelector('.saved-flip-front .front-logo').src = logoBase + logoFile + '.png';

  // Waves
  const waveAlpha = isDark ? 0.05 : 0.04;
  cardEl.querySelector('.saved-flip-front .wave-path-1').setAttribute('fill', isDark ? `rgba(200,180,155,${waveAlpha})` : `${fc1}11`);
  cardEl.querySelector('.saved-flip-front .wave-path-2').setAttribute('fill', isDark ? `rgba(200,180,155,${waveAlpha * 0.7})` : `${fc1}09`);

  // Back
  const back = cardEl.querySelector('.card-back');
  back.style.background = isDark ? '#FDFAF5' : '#FDFCFA';
  cardEl.querySelector('.saved-flip-back .back-bar').style.background = `linear-gradient(90deg, ${bc1}, ${lighten(bc1,40)}, ${bc2}, ${lighten(bc2,40)}, ${bc1})`;

  // Back logo
  const backLogoFile = LOGO_MAP[s.backLogo] || LOGO_MAP[s.frontLogo];
  if (backLogoFile) cardEl.querySelector('.saved-flip-back .back-logo-small').src = logoBase + backLogoFile + '.png';

  // Back text
  cardEl.querySelector('.saved-flip-back .back-name').style.color = txt;
  cardEl.querySelector('.saved-flip-back .back-jobtitle').style.color = bc1;
  cardEl.querySelector('.saved-flip-back .back-identity').style.borderRight = `2px solid ${lighten(bc1, 180)}`;

  // Contact
  cardEl.querySelectorAll('.saved-flip-back .contact-icon').forEach(el => el.style.color = bc1);
  cardEl.querySelectorAll('.saved-flip-back .contact-text').forEach(el => el.style.color = txt);
  cardEl.querySelectorAll('.saved-flip-back .contact-label').forEach(el => el.style.color = bc2);
  cardEl.querySelectorAll('.saved-flip-back .contact-text a').forEach(el => {
    el.style.color = txt;
    el.style.borderBottom = `1px solid ${txt}33`;
  });

  // Footer
  cardEl.querySelector('.saved-flip-back .footer-wave-path').setAttribute('fill', ftr);
  cardEl.querySelectorAll('.saved-flip-back .social-icon').forEach(el => el.style.color = 'rgba(255,255,255,0.8)');
}
```

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "feat: update applyStateToCard() to use split front/back state keys"
```

---

### Task 6: Add picker row CSS and warning swatch style

**Files:**
- Modify: `index.html` — CSS section, after `.builder-pickers` styles (lines ~319-365)

- [ ] **Step 1: Replace `.builder-pickers` CSS and add new row/warning styles**

Replace the existing `.builder-pickers` rule:
```css
    /* Swatch-dropdown pickers below the cards */
    .builder-pickers {
      display: flex;
      justify-content: center;
      gap: 1rem;
      flex-wrap: wrap;
      margin-top: 2rem;
      margin-bottom: 2rem;
    }
```

With:
```css
    /* Swatch-dropdown pickers below the cards */
    .builder-pickers {
      display: flex;
      flex-direction: column;
      align-items: center;
      gap: 1.2rem;
      margin-top: 2rem;
      margin-bottom: 2rem;
    }

    .picker-row {
      display: flex;
      align-items: flex-start;
      gap: 1rem;
      flex-wrap: wrap;
      justify-content: center;
    }

    .picker-row-label {
      font-family: 'Libre Franklin', sans-serif;
      font-weight: 600;
      font-size: 0.6rem;
      letter-spacing: 0.15em;
      text-transform: uppercase;
      color: #7A7063;
      writing-mode: horizontal-tb;
      min-width: 40px;
      text-align: right;
      padding-top: 0.6rem;
    }

    .picker-swatch.contrast-warn {
      border-color: #D97706;
      box-shadow: 0 0 0 2px rgba(217, 119, 6, 0.3);
    }
```

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "feat: add picker row layout CSS and contrast warning style"
```

---

### Task 7: Rebuild picker construction with two rows and WCAG filtering

**Files:**
- Modify: `index.html` — `PICKER_DEFS`, picker construction loop, `getSwatchColor`, `refreshSwatchUI` (lines ~989-1051)

- [ ] **Step 1: Replace the entire picker construction section**

Replace from `// ═══ Build dropdown pickers ═══` through the end of `refreshSwatchUI()` function with:

```js
// ═══ Build dropdown pickers ═══
const PICKER_ROWS = [
  {
    label: 'Front',
    pickers: [
      { key: 'frontBg',      label: 'Background', options: BG_OPTIONS },
      { key: 'frontLogo',    label: 'Logo',       options: LOGO_COLORS },
      { key: 'frontAccent1', label: 'Accent 1',   options: BRAND },
      { key: 'frontAccent2', label: 'Accent 2',   options: BRAND },
    ],
  },
  {
    label: 'Back',
    pickers: [
      { key: 'backLogo',    label: 'Logo',     options: LOGO_COLORS },
      { key: 'backAccent1', label: 'Accent 1', options: BRAND },
      { key: 'backAccent2', label: 'Accent 2', options: BRAND },
      { key: 'backText',    label: 'Text',     options: BRAND },
      { key: 'backFooter',  label: 'Footer',   options: BRAND },
    ],
  },
];

// Flat list for iteration
const PICKER_DEFS = PICKER_ROWS.flatMap(row => row.pickers);

const pickersContainer = document.getElementById('builder-pickers');

function getSwatchColor(name, options) {
  const val = options[name];
  if (typeof val === 'string') return val;
  if (val && val.bg) return val.dark ? '#15424A' : '#F0E2CA';
  return '#ccc';
}

function getCheckBg(pickerKey) {
  const rule = CONTRAST_RULES[pickerKey];
  if (!rule) return null;
  if (rule.bgSource === 'front') return BG_CHECK_COLORS[state.frontBg] || '#FFFFFF';
  const isDark = BG_OPTIONS[state.frontBg] && BG_OPTIONS[state.frontBg].dark;
  return isDark ? BACK_BG_DARK : BACK_BG_LIGHT;
}

function passesContrast(pickerKey, colorName) {
  const rule = CONTRAST_RULES[pickerKey];
  if (!rule) return true;
  const hex = BRAND[colorName];
  if (!hex) return true;
  const bgHex = getCheckBg(pickerKey);
  return contrastRatio(hex, bgHex) >= rule.threshold;
}

// Build the two-row picker UI
PICKER_ROWS.forEach(rowDef => {
  const row = document.createElement('div');
  row.className = 'picker-row';

  const rowLabel = document.createElement('span');
  rowLabel.className = 'picker-row-label';
  rowLabel.textContent = rowDef.label;
  row.appendChild(rowLabel);

  rowDef.pickers.forEach(def => {
    const picker = document.createElement('div');
    picker.className = 'picker';
    picker.id = `picker-${def.key}`;

    const swatch = document.createElement('div');
    swatch.className = 'picker-swatch';
    swatch.style.background = getSwatchColor(state[def.key], def.options);

    const select = document.createElement('select');
    select.className = 'picker-select';
    select.title = def.label;

    // For frontLogo when dark BG, show locked "White (auto)"
    const isDarkBg = BG_OPTIONS[state.frontBg] && BG_OPTIONS[state.frontBg].dark;
    if (def.key === 'frontLogo' && isDarkBg) {
      const opt = document.createElement('option');
      opt.value = state[def.key];
      opt.textContent = 'White (auto)';
      opt.selected = true;
      select.appendChild(opt);
      select.disabled = true;
      swatch.style.background = '#FFFFFF';
    } else {
      Object.keys(def.options).forEach(name => {
        const opt = document.createElement('option');
        opt.value = name;
        opt.textContent = name;
        if (name === state[def.key]) opt.selected = true;
        // Hard-filter: disable options that fail contrast
        const rule = CONTRAST_RULES[def.key];
        if (rule && rule.mode === 'hard' && !passesContrast(def.key, name)) {
          opt.disabled = true;
          opt.textContent = name + ' (low contrast)';
        }
        select.appendChild(opt);
      });
    }

    select.addEventListener('change', () => {
      state[def.key] = select.value;
      swatch.style.background = getSwatchColor(select.value, def.options);
      // When frontBg changes, rebuild all dropdowns (contrast thresholds shift)
      if (def.key === 'frontBg') {
        refreshSwatchUI();
      } else {
        updateContrastWarnings();
      }
      applyState();
    });

    const label = document.createElement('span');
    label.className = 'picker-label';
    label.textContent = def.label;

    picker.appendChild(swatch);
    picker.appendChild(select);
    picker.appendChild(label);
    row.appendChild(picker);
  });

  pickersContainer.appendChild(row);
});

function updateContrastWarnings() {
  PICKER_DEFS.forEach(def => {
    const rule = CONTRAST_RULES[def.key];
    if (!rule) return;
    const picker = document.getElementById(`picker-${def.key}`);
    const swatch = picker.querySelector('.picker-swatch');
    const passes = passesContrast(def.key, state[def.key]);
    swatch.classList.toggle('contrast-warn', !passes);
  });
}

function refreshSwatchUI() {
  PICKER_DEFS.forEach(def => {
    const picker = document.getElementById(`picker-${def.key}`);
    const select = picker.querySelector('select');
    const swatch = picker.querySelector('.picker-swatch');

    // Rebuild options to reflect current contrast state
    select.innerHTML = '';
    select.disabled = false;

    const isDarkBg = BG_OPTIONS[state.frontBg] && BG_OPTIONS[state.frontBg].dark;
    if (def.key === 'frontLogo' && isDarkBg) {
      const opt = document.createElement('option');
      opt.value = state[def.key];
      opt.textContent = 'White (auto)';
      opt.selected = true;
      select.appendChild(opt);
      select.disabled = true;
      swatch.style.background = '#FFFFFF';
    } else {
      Object.keys(def.options).forEach(name => {
        const opt = document.createElement('option');
        opt.value = name;
        opt.textContent = name;
        if (name === state[def.key]) opt.selected = true;
        const rule = CONTRAST_RULES[def.key];
        if (rule && rule.mode === 'hard' && !passesContrast(def.key, name)) {
          opt.disabled = true;
          opt.textContent = name + ' (low contrast)';
        }
        select.appendChild(opt);
      });
      swatch.style.background = getSwatchColor(state[def.key], def.options);
    }
  });
  updateContrastWarnings();
}
```

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "feat: rebuild picker UI with two rows, WCAG filtering, and contrast warnings"
```

---

### Task 8: Update Randomize to use new state keys and enforce WCAG

**Files:**
- Modify: `index.html` — randomize click handler (lines ~1054-1078)

- [ ] **Step 1: Replace the randomize handler**

Replace the entire `btn-randomize` click handler with:

```js
document.getElementById('btn-randomize').addEventListener('click', () => {
  const bgKeys = Object.keys(BG_OPTIONS);
  const brandKeys = Object.keys(BRAND);
  const logoKeys = Object.keys(LOGO_MAP);

  state.frontBg = bgKeys[Math.floor(Math.random() * bgKeys.length)];

  const isDark = BG_OPTIONS[state.frontBg].dark;
  const frontBgHex = BG_CHECK_COLORS[state.frontBg] || '#FFFFFF';
  const backBgHex = isDark ? BACK_BG_DARK : BACK_BG_LIGHT;

  function pickPassing(keys, bgHex, threshold) {
    const passing = keys.filter(k => {
      const hex = BRAND[k];
      return hex && contrastRatio(hex, bgHex) >= threshold;
    });
    return passing[Math.floor(Math.random() * passing.length)];
  }

  // Front pickers
  state.frontLogo = isDark
    ? logoKeys[Math.floor(Math.random() * logoKeys.length)]
    : pickPassing(logoKeys, frontBgHex, 3) || logoKeys[0];
  state.frontAccent1 = brandKeys[Math.floor(Math.random() * brandKeys.length)]; // warn-only
  state.frontAccent2 = pickPassing(brandKeys, frontBgHex, 4.5) || brandKeys[0];

  // Back pickers
  state.backLogo    = pickPassing(logoKeys, backBgHex, 3) || logoKeys[0];
  state.backAccent1 = pickPassing(brandKeys, backBgHex, 3) || brandKeys[0];
  state.backAccent2 = pickPassing(brandKeys, backBgHex, 4.5) || brandKeys[0];
  state.backText    = pickPassing(brandKeys, backBgHex, 4.5) || brandKeys[0];
  state.backFooter  = brandKeys[Math.floor(Math.random() * brandKeys.length)]; // warn-only

  refreshSwatchUI();
  applyState();
});
```

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "feat: update randomize to use split state keys and enforce WCAG contrast"
```

---

### Task 9: Update Reset, URL hash, and localStorage clearing

**Files:**
- Modify: `index.html` — reset handler (lines ~1080-1089), `stateToHash` (lines ~1100-1103), `hashToState` (lines ~1105-1116), and add localStorage clearing near `STORAGE_KEY` (line ~1209)

- [ ] **Step 1: Replace the reset handler**

Replace:
```js
document.getElementById('btn-reset').addEventListener('click', () => {
  state = {
    frontBg: 'White / Cream', logo: 'San Gorgonio Blue',
    accent1: 'San Gorgonio Blue', accent2: 'Sage',
    textColor: 'San Gorgonio Blue', footer: 'San Gorgonio Blue',
  };
  refreshSwatchUI();
  applyState();
});
```

With:
```js
document.getElementById('btn-reset').addEventListener('click', () => {
  state = {
    frontBg: 'White / Cream',
    frontLogo: 'San Gorgonio Blue', frontAccent1: 'San Gorgonio Blue', frontAccent2: 'Sage',
    backLogo: 'San Gorgonio Blue', backAccent1: 'San Gorgonio Blue', backAccent2: 'Sage',
    backText: 'San Gorgonio Blue', backFooter: 'San Gorgonio Blue',
  };
  refreshSwatchUI();
  applyState();
});
```

- [ ] **Step 2: Replace `stateToHash()`**

Replace:
```js
function stateToHash() {
  const s = [state.frontBg, state.logo, state.accent1, state.accent2, state.textColor, state.footer];
  return '#custom=' + encodeURIComponent(JSON.stringify(s));
}
```

With:
```js
function stateToHash() {
  const s = [state.frontBg, state.frontLogo, state.frontAccent1, state.frontAccent2,
             state.backLogo, state.backAccent1, state.backAccent2, state.backText, state.backFooter];
  return '#custom=' + encodeURIComponent(JSON.stringify(s));
}
```

- [ ] **Step 3: Replace `hashToState()`**

Replace:
```js
function hashToState(hash) {
  try {
    const raw = hash.replace('#custom=', '');
    const s = JSON.parse(decodeURIComponent(raw));
    if (s.length === 6) {
      state.frontBg = s[0]; state.logo = s[1]; state.accent1 = s[2];
      state.accent2 = s[3]; state.textColor = s[4]; state.footer = s[5];
      return true;
    }
  } catch(e) {}
  return false;
}
```

With:
```js
function hashToState(hash) {
  try {
    const raw = hash.replace('#custom=', '');
    const s = JSON.parse(decodeURIComponent(raw));
    if (s.length === 9) {
      state.frontBg = s[0]; state.frontLogo = s[1]; state.frontAccent1 = s[2]; state.frontAccent2 = s[3];
      state.backLogo = s[4]; state.backAccent1 = s[5]; state.backAccent2 = s[6];
      state.backText = s[7]; state.backFooter = s[8];
      return true;
    }
  } catch(e) {}
  return false;
}
```

- [ ] **Step 4: Add localStorage clearing after `STORAGE_KEY` definition**

Insert after `const STORAGE_KEY = 'bld-card-versions';`:

```js
// Clear pre-v2 saved schemes (old 6-key state format)
try {
  const existing = JSON.parse(localStorage.getItem(STORAGE_KEY));
  if (Array.isArray(existing) && existing.length > 0 && existing[0].state && !existing[0].state.frontLogo) {
    localStorage.removeItem(STORAGE_KEY);
  }
} catch(e) {}
```

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: update reset, URL hash (9 values), and clear legacy localStorage"
```

---

### Task 10: Update Copy Full Spec to show split front/back sections

**Files:**
- Modify: `index.html` — `btn-spec` click handler (lines ~1128-1198)

- [ ] **Step 1: Replace the spec generation**

Replace the entire `btn-spec` click handler with:

```js
document.getElementById('btn-spec').addEventListener('click', () => {
  const bgOpt = BG_OPTIONS[state.frontBg];
  const isDark = bgOpt.dark;
  const fc1 = BRAND[state.frontAccent1];
  const fc2 = BRAND[state.frontAccent2];
  const bc1 = BRAND[state.backAccent1];
  const bc2 = BRAND[state.backAccent2];
  const txt = BRAND[state.backText];
  const ftr = BRAND[state.backFooter];
  const frontLogoFile = isDark ? 'White' : (LOGO_MAP[state.frontLogo] || state.frontLogo);
  const backLogoFile = LOGO_MAP[state.backLogo] || LOGO_MAP[state.frontLogo];

  const spec = [
    `BEAUMONT LIBRARY DISTRICT`,
    `Business Card Color Specification`,
    `Generated: ${new Date().toLocaleDateString('en-US', { year:'numeric', month:'long', day:'numeric' })}`,
    ``,
    `══════════════════════════════════════`,
    `  FRONT FACE`,
    `══════════════════════════════════════`,
    `Background:       ${state.frontBg}${isDark ? ' (dark)' : ' (light)'}`,
    `Logo Color:       ${frontLogoFile}`,
    `Accent 1:         ${fc1}  (${state.frontAccent1})`,
    `  Accent Strip:   linear-gradient ${state.frontAccent1} → ${state.frontAccent2}`,
    `  Divider Rule:   linear-gradient ${state.frontAccent1} → ${state.frontAccent2}`,
    `Accent 2:         ${fc2}  (${state.frontAccent2})`,
    `  Tagline Text:   ${isDark ? lighten(fc2, 80) : fc2}  (${state.frontAccent2})`,
    ``,
    `══════════════════════════════════════`,
    `  BACK FACE`,
    `══════════════════════════════════════`,
    `Background:       ${isDark ? '#FDFAF5 (dark theme)' : '#FDFCFA (light theme)'}`,
    `Logo Color:       ${backLogoFile}  (${state.backLogo})`,
    `Accent 1:         ${bc1}  (${state.backAccent1})`,
    `  Top Bar:        linear-gradient ${state.backAccent1} → ${state.backAccent2}`,
    `  Job Title:      ${bc1}  (${state.backAccent1})`,
    `  Icons:          ${bc1}  (${state.backAccent1})`,
    `  Divider:        ${lighten(bc1, 180)}`,
    `Accent 2:         ${bc2}  (${state.backAccent2})`,
    `  Contact Labels: ${bc2}  (${state.backAccent2})`,
    `Text:             ${txt}  (${state.backText})`,
    `  Staff Name:     ${txt}  (${state.backText})`,
    `  Contact Text:   ${txt}  (${state.backText})`,
    `Footer:           ${ftr}  (${state.backFooter})`,
    `  Wave Fill:      ${ftr}`,
    ``,
    `══════════════════════════════════════`,
    `  BRAND PALETTE REFERENCE`,
    `══════════════════════════════════════`,
    ...Object.entries(BRAND).map(([name, hex]) => `  ${name.padEnd(22)} ${hex}`),
    ``,
    `══════════════════════════════════════`,
    `  SHARE LINK`,
    `══════════════════════════════════════`,
    `${window.location.origin + window.location.pathname + stateToHash()}`,
  ].join('\n');

  navigator.clipboard.writeText(spec).then(() => {
    showToast('Full spec copied to clipboard');
  }).catch(() => {
    const w = window.open('', '_blank', 'width=600,height=700');
    w.document.write(`<pre style="font-family:monospace;font-size:13px;padding:2rem;white-space:pre-wrap">${spec}</pre>`);
    w.document.title = 'BLD Card Spec';
    showToast('Spec opened in new tab — copy from there');
  });
});
```

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "feat: update Copy Full Spec output for split front/back sections"
```

---

### Task 11: Update `generateSchemeName()` to use new state keys

**Files:**
- Modify: `index.html` — `generateSchemeName()` function (lines ~1329-1345)

- [ ] **Step 1: Update the color word lookups**

Replace:
```js
  const word1 = pick(COLOR_WORDS[s.accent1] || COLOR_WORDS['San Gorgonio Blue']);
  const word2 = pick(COLOR_WORDS[s.accent2] || COLOR_WORDS['Sage']);
```

With:
```js
  const word1 = pick(COLOR_WORDS[s.frontAccent1] || COLOR_WORDS['San Gorgonio Blue']);
  const word2 = pick(COLOR_WORDS[s.backAccent1] || COLOR_WORDS['Sage']);
```

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "feat: update generateSchemeName() to use split state keys"
```

---

### Task 12: Smoke test the full flow in browser

**Files:**
- No changes — testing only

- [ ] **Step 1: Open `index.html` in browser and verify default state renders correctly**

Both card faces should show the original BLD theme (San Gorgonio Blue / Sage).

- [ ] **Step 2: Verify picker layout**

Two rows should appear: "FRONT" with 4 pickers (Background, Logo, Accent 1, Accent 2) and "BACK" with 5 pickers (Logo, Accent 1, Accent 2, Text, Footer).

- [ ] **Step 3: Test WCAG hard-filtering**

Open the Back Text dropdown — Desert Sand and Citrus Gold should show as disabled with "(low contrast)" suffix. Select a dark front background (Deep Teal) — front Logo picker should lock to "White (auto)".

- [ ] **Step 4: Test WCAG warning**

Select Desert Sand for Front Accent 1 (warn-only) on a light background — the swatch should get an orange border but remain selectable.

- [ ] **Step 5: Test Randomize**

Click Randomize several times. All generated combinations should have compliant text/logo contrast. No disabled-color options should appear as selected values.

- [ ] **Step 6: Test Reset**

Click Reset. All pickers should return to default BLD theme values.

- [ ] **Step 7: Test Save / Load cycle**

Click Save Scheme, name it, verify it appears in the saved grid with correct colors on both faces. Click Load on a saved scheme, verify pickers update.

- [ ] **Step 8: Test Share Link**

Click Copy Share Link, paste the URL in a new tab, verify the 9-value hash loads correctly.

- [ ] **Step 9: Test Copy Full Spec**

Click Copy Full Spec, verify the output shows separate "FRONT FACE" and "BACK FACE" sections with independent color assignments.

- [ ] **Step 10: Commit final verified state**

```bash
git add index.html
git commit -m "feat: split front/back pickers with WCAG contrast enforcement"
```
