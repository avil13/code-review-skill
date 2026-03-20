# CSS / Less / Sass Review Guide

Guidance for reviewing CSS and preprocessor code: performance, maintainability, responsive design, and browser support.

## CSS variables vs hard-coded values

### When to use variables

```css
/* ❌ Hard-coded — hard to keep consistent */
.button {
  background: #3b82f6;
  border-radius: 8px;
}
.card {
  border: 1px solid #3b82f6;
  border-radius: 8px;
}

/* ✅ CSS custom properties */
:root {
  --color-primary: #3b82f6;
  --radius-md: 8px;
}
.button {
  background: var(--color-primary);
  border-radius: var(--radius-md);
}
.card {
  border: 1px solid var(--color-primary);
  border-radius: var(--radius-md);
}
```

### Naming variables

```css
/* Suggested groupings */
:root {
  /* Color */
  --color-primary: #3b82f6;
  --color-primary-hover: #2563eb;
  --color-text: #1f2937;
  --color-text-muted: #6b7280;
  --color-bg: #ffffff;
  --color-border: #e5e7eb;

  /* Spacing */
  --spacing-xs: 4px;
  --spacing-sm: 8px;
  --spacing-md: 16px;
  --spacing-lg: 24px;
  --spacing-xl: 32px;

  /* Typography */
  --font-size-sm: 14px;
  --font-size-base: 16px;
  --font-size-lg: 18px;
  --font-weight-normal: 400;
  --font-weight-bold: 700;

  /* Radius */
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 12px;
  --radius-full: 9999px;

  /* Shadow */
  --shadow-sm: 0 1px 2px rgba(0, 0, 0, 0.05);
  --shadow-md: 0 4px 6px rgba(0, 0, 0, 0.1);

  /* Motion */
  --transition-fast: 150ms ease;
  --transition-normal: 300ms ease;
}
```

### Scoping variables

```css
/* ✅ Component-scoped tokens — less global pollution */
.card {
  --card-padding: var(--spacing-md);
  --card-radius: var(--radius-md);

  padding: var(--card-padding);
  border-radius: var(--card-radius);
}

/* ⚠️ Avoid thrashing custom properties from JS too often — cost adds up */
```

### Review checklist

- [ ] Colors expressed as variables (or design tokens)?
- [ ] Spacing aligned with a design system?
- [ ] Repeated literals extracted?
- [ ] Names semantic, not only `color1` / `blue2`?

---

## `!important` policy

### When it can be OK

```css
/* ✅ Utilities that must win */
.hidden { display: none !important; }
.sr-only { position: absolute !important; }

/* ✅ Third-party overrides you can’t fix at source */
.third-party-modal {
  z-index: 9999 !important;
}

/* ✅ Print */
@media print {
  .no-print { display: none !important; }
}
```

### When to avoid it

```css
/* ❌ Papering over specificity — fix the cascade */
.button {
  background: blue !important;  /* why? */
}

/* ❌ Fighting your own earlier rule */
.card { padding: 20px; }
.card { padding: 30px !important; }  /* edit the first rule */

/* ❌ Inside component CSS */
.my-component .title {
  font-size: 24px !important;  /* breaks encapsulation */
}
```

### Better alternatives

```css
/* Goal: override .btn */

/* ❌ !important */
.my-btn {
  background: red !important;
}

/* ✅ Higher specificity */
button.my-btn {
  background: red;
}

/* ✅ More specific context */
.container .my-btn {
  background: red;
}

/* ✅ :where() keeps base rule easy to beat */
:where(.btn) {
  background: blue;  /* specificity 0 for the selector list */
}
.my-btn {
  background: red;
}
```

### Sample review comments

```markdown
🔴 [blocking] "15 uses of !important — justify each or remove"
🟡 [important] "This can be fixed by selector order/specificity instead"
💡 [suggestion] "Consider @layer for predictable cascade control"
```

---

## Performance

### High-impact issues

#### 1. `transition: all`

```css
/* ❌ Browser must consider every interpolable property */
.button {
  transition: all 0.3s ease;
}

/* ✅ List animated properties */
.button {
  transition: background-color 0.3s ease, transform 0.3s ease;
}

/* ✅ Shared duration token */
.button {
  --transition-duration: 0.3s;
  transition:
    background-color var(--transition-duration) ease,
    box-shadow var(--transition-duration) ease,
    transform var(--transition-duration) ease;
}
```

#### 2. Animating `box-shadow`

```css
/* ❌ Heavy repaint each frame */
.card {
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  transition: box-shadow 0.3s ease;
}
.card:hover {
  box-shadow: 0 8px 16px rgba(0,0,0,0.2);
}

/* ✅ Pseudo-element + opacity (often cheaper) */
.card {
  position: relative;
}
.card::after {
  content: '';
  position: absolute;
  inset: 0;
  box-shadow: 0 8px 16px rgba(0,0,0,0.2);
  opacity: 0;
  transition: opacity 0.3s ease;
  pointer-events: none;
  border-radius: inherit;
}
.card:hover::after {
  opacity: 1;
}
```

#### 3. Properties that trigger layout (reflow)

```css
/* ❌ Animating layout */
.bad-animation {
  transition: width 0.3s, height 0.3s, top 0.3s, left 0.3s, margin 0.3s;
}

/* ✅ Prefer transform + opacity (compositor-friendly) */
.good-animation {
  transition: transform 0.3s, opacity 0.3s;
}

/* Position: translate instead of top/left */
.move {
  transform: translateX(100px);  /* ✅ */
  /* left: 100px; */             /* ❌ */
}

/* Size feel: scale instead of width/height when possible */
.grow {
  transform: scale(1.1);  /* ✅ */
  /* width: 110%; */      /* ❌ */
}
```

### Medium-impact issues

#### Complex selectors

```css
/* ❌ Deep chains — slower matching */
.page .container .content .article .section .paragraph span {
  color: red;
}

/* ✅ Flatter, purposeful classes */
.article-text {
  color: red;
}

/* ❌ Universal / broad attribute selectors */
* { box-sizing: border-box; }           /* hits everything */
[class*="icon-"] { display: inline; }   /* attribute scan */

/* ✅ Scope resets */
.icon-box * { box-sizing: border-box; }
```

#### Heavy shadows and filters

```css
/* ⚠️ Stacked shadows cost paint */
.heavy-shadow {
  box-shadow:
    0 1px 2px rgba(0,0,0,0.1),
    0 2px 4px rgba(0,0,0,0.1),
    0 4px 8px rgba(0,0,0,0.1),
    0 8px 16px rgba(0,0,0,0.1),
    0 16px 32px rgba(0,0,0,0.1);
}

/* ⚠️ Filters and backdrop-filter are GPU-heavy */
.blur-heavy {
  filter: blur(20px) brightness(1.2) contrast(1.1);
  backdrop-filter: blur(10px);
}
```

### Tuning tips

```css
/* will-change — use sparingly, remove after animation */
.animated-element {
  will-change: transform, opacity;
}

.animated-element.idle {
  will-change: auto;
}

/* contain — limit invalidation blast radius */
.card {
  contain: layout paint;
}
```

### Performance checklist

- [ ] Any `transition: all`?
- [ ] Animating `width` / `height` / `top` / `left`?
- [ ] Animating `box-shadow`?
- [ ] Selector depth > ~3 meaningful levels?
- [ ] Leftover `will-change` on idle elements?

---

## Responsive design

### Mobile-first

```css
/* ✅ Base = small screens, enhance upward */
.container {
  padding: 16px;
  display: flex;
  flex-direction: column;
}

@media (min-width: 768px) {
  .container {
    padding: 24px;
    flex-direction: row;
  }
}

@media (min-width: 1024px) {
  .container {
    padding: 32px;
    max-width: 1200px;
    margin: 0 auto;
  }
}

/* ❌ Desktop-first — more overrides to undo */
.container {
  max-width: 1200px;
  padding: 32px;
  flex-direction: row;
}

@media (max-width: 1023px) {
  .container {
    padding: 24px;
  }
}

@media (max-width: 767px) {
  .container {
    padding: 16px;
    flex-direction: column;
    max-width: none;
  }
}
```

### Breakpoint tokens

```css
/* Prefer content-driven breakpoints */
:root {
  --breakpoint-sm: 640px;
  --breakpoint-md: 768px;
  --breakpoint-lg: 1024px;
  --breakpoint-xl: 1280px;
  --breakpoint-2xl: 1536px;
}

@media (min-width: 768px) { /* md */ }
@media (min-width: 1024px) { /* lg */ }
```

### Responsive checklist

- [ ] Mobile-first min-width media queries?
- [ ] Breakpoints tied to layout breaks, not device SKUs?
- [ ] Avoid redundant overlapping ranges?
- [ ] Type in relative units (`rem` / `em`) where it helps?
- [ ] Touch targets ≥ ~44px where relevant?
- [ ] Tested orientation changes?

### Common fixes

```css
/* ❌ Fixed width */
.container {
  width: 1200px;
}

/* ✅ Fluid + cap */
.container {
  width: 100%;
  max-width: 1200px;
  padding-inline: 16px;
}

/* ❌ Fixed height on text */
.text-box {
  height: 100px;
}

/* ✅ min-height */
.text-box {
  min-height: 100px;
}

/* ❌ Tiny tap targets */
.small-button {
  padding: 4px 8px;
}

/* ✅ Minimum touch size */
.touch-button {
  min-height: 44px;
  min-width: 44px;
  padding: 12px 16px;
}
```

---

## Browser compatibility

### Features to verify

| Feature | Support | Note |
|---------|---------|------|
| CSS Grid | Modern ✅ | IE needs Autoprefixer + testing |
| Flexbox | Broad ✅ | Very old engines may need prefixes |
| CSS variables | Modern ✅ | No IE — provide fallbacks |
| `gap` in flex | Newer ⚠️ | Safari 14.1+ |
| `:has()` | Newer ⚠️ | Check baseline (e.g. Firefox 121+) |
| Container queries | Newer ⚠️ | ~2023+ baselines |
| `@layer` | Newer ⚠️ | Match project browsers |

### Fallback patterns

```css
/* Custom property with solid fallback */
.button {
  background: #3b82f6;
  background: var(--color-primary);
}

/* Flex gap fallback */
.flex-container {
  display: flex;
  gap: 16px;
}
.flex-container > * + * {
  margin-left: 16px;
}

/* Grid progressive enhancement */
.grid {
  display: flex;
  flex-wrap: wrap;
}
@supports (display: grid) {
  .grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  }
}
```

### Autoprefixer

```javascript
// postcss.config.js
module.exports = {
  plugins: [
    require('autoprefixer')({
      grid: 'autoplace',
      flexbox: 'no-2009',
    }),
  ],
};

// package.json
{
  "browserslist": [
    "> 1%",
    "last 2 versions",
    "not dead",
    "not ie 11"
  ]
}
```

### Compatibility checklist

- [ ] Checked [Can I Use](https://caniuse.com) for new syntax?
- [ ] Fallbacks or `@supports` where needed?
- [ ] Autoprefixer wired to `browserslist`?
- [ ] `browserslist` matches product requirements?
- [ ] Smoke-tested in target browsers?

---

## Less / Sass specifics

### Nesting depth

```scss
/* ❌ Too deep — bloated compiled selectors */
.page {
  .container {
    .content {
      .article {
        .title {
          color: red;
        }
      }
    }
  }
}

/* ✅ Aim for ≤ ~3 levels; BEM-style helps */
.article {
  &__title {
    color: red;
  }

  &__content {
    p { margin-bottom: 1em; }
  }
}
```

### Mixins vs `@extend` vs variables

```scss
/* Variables — single values */
$primary-color: #3b82f6;

/* Mixins — parameterized snippets */
@mixin button-variant($bg, $text) {
  background: $bg;
  color: $text;
  &:hover {
    background: darken($bg, 10%);
  }
}

/* @extend — shared rules (easy to misuse) */
%visually-hidden {
  position: absolute;
  width: 1px;
  height: 1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
}

.sr-only {
  @extend %visually-hidden;
}

/* ⚠️ @extend pitfalls: surprise selector unions, no @media extend,
   often prefer mixins or plain classes */
```

### Preprocessor checklist

- [ ] Nesting ≤ ~3 levels?
- [ ] `@extend` used rarely and deliberately?
- [ ] Mixins kept focused, not mega-blocks?
- [ ] Output CSS size reasonable (no explosion from extends)?

---

## Quick review pass

### Must fix

```markdown
□ transition: all
□ Animating width/height/top/left/margin
□ Many !important uses
□ Same hard-coded colors/spacing 3+ times
□ Selector nesting >4 levels
```

### Should fix

```markdown
□ Missing responsive behavior
□ Desktop-first only
□ Complex animated box-shadow
□ No compatibility fallback for new features
□ Over-broad global custom properties
```

### Nice to have

```markdown
□ Grid could simplify layout
□ Tokens/variables for repeated literals
□ @layer for cascade clarity
□ contain for isolated widgets
```

---

## Tools

| Tool | Use |
|------|-----|
| [Stylelint](https://stylelint.io/) | Lint CSS / SCSS |
| [PurgeCSS](https://purgecss.com/) | Strip unused CSS |
| [Autoprefixer](https://autoprefixer.github.io/) | Vendor prefixes |
| [CSS Stats](https://cssstats.com/) | Size / rule stats |
| [Can I Use](https://caniuse.com/) | Feature support |

---

## References

- [CSS Performance Optimization - MDN](https://developer.mozilla.org/en-US/docs/Learn_web_development/Extensions/Performance/CSS)
- [What a CSS Code Review Might Look Like - CSS-Tricks](https://css-tricks.com/what-a-css-code-review-might-look-like/)
- [How to Animate Box-Shadow - Tobias Ahlin](https://tobiasahlin.com/blog/how-to-animate-box-shadow/)
- [Media Query Fundamentals - MDN](https://developer.mozilla.org/en-US/docs/Learn_web_development/Core/CSS_layout/Media_queries)
- [Autoprefixer - GitHub](https://github.com/postcss/autoprefixer)
