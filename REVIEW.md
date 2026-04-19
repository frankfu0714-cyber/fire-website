# FIRE Calculator — Code Review

**Scope:** `index.html` (single-file HTML/CSS/JS) hosted at `frankfu0714-cyber.github.io/fire-website/`.
**Date:** 2026-04-19

## Overall Assessment

This is a polished and thoughtful project. The dark theme is attractive, the five-language i18n is genuinely complete (not just a stub), and the two-mode split (perpetual vs. drawdown-to-zero) is a smart pedagogical choice — most online FIRE calculators only show one. The math is mostly right, inputs are reasonable, and the code is readable.

The main issues called out below: (1) one real math bug in the drawdown target that silently over-estimates how much you need to retire, (2) some missing guard rails around edge cases (savings running out, realReturn ≤ 0), and (3) accessibility/responsiveness gaps that are easy wins. Code organization is the other thing to fix — a 2,300-line single file will hurt the moment you try to add the next feature.

---

## Critical

### C1. `pvAnnuity` Math.max killed the drawdown discount (FIXED)

```js
// Before
return Math.max(pv, annualExpense * years);
```

The PV of an annuity at positive real return is **always** less than `annualExpense × years` — that's the whole point: the portfolio keeps earning while you draw it down. Taking `Math.max` forced the target back up to the no-return case whenever `realReturn > 0`. Concrete example: $40K/year for 40 years at 4% real return → true PV ≈ $791K, but the old code returned $1.6M (≈2× over-estimate).

**Fix applied:** dropped the `Math.max`, kept the early-exit for `|realReturn| < 0.0001`, added a conservative branch when `realReturn < 0`.

### C2. Drawdown chart hid when savings ran out (FIXED)

`portfolio.push(Math.max(drawBal, 0))` clamped the drawdown line at zero if the balance went negative. The chart then showed a flat zero line for the rest of life expectancy with no warning — a user whose plan silently failed saw a "successful" chart.

**Fix applied:** added `estimateDepletionAge()` that simulates spend-down month-by-month. If the balance hits zero before life expectancy, the alert tells the user the depletion age.

### C3. `realReturn ≤ 0` warned but still computed nonsense (FIXED)

When inflation ≥ return, the code showed a warning but the target and chart still rendered with broken numbers (negative annuity PVs, infinite perpetual target). Users saw both the warning and the broken output simultaneously.

**Fix applied:** when `isPerp && realReturn <= 0`, the alert fires AND `valid = false` short-circuits downstream rendering.

---

## High

### H1. Accessibility (FIXED)

- Range sliders had no `aria-label` — screen reader announced them as "slider" with no context. Now each slider has a descriptive `aria-label`.
- Mode tabs were plain `<button>` — now `role="tablist"` / `role="tab"` / `aria-selected` (and `setMode` flips `aria-selected` on switch).
- Muted color was `#888899` on `#0b0b0f` ≈ 3.8:1 contrast (fails WCAG AA). Bumped to `#a0a0b3` for compliance.

### H2. Mobile breakpoint for small phones (FIXED)

The 800px media query covered tablets, but nothing targeted 320–480px. Numeric inputs could overflow on small phones. Added a `@media (max-width: 480px)` block: stacks mode tabs vertically, tightens padding, shrinks the big progress number, and tightens the comparison table.

### H3. Inline `onchange=` handlers in rendered expense rows (NOT FIXED)

`renderExpenseList` builds rows with `onchange="updateExpenseName(...)"` in the HTML string. Current escaping makes it XSS-safe, but inline handlers block adoption of a strict Content Security Policy. Future refactor: swap to `addEventListener` after `insertAdjacentHTML`.

### H4. No validation that age < lifeExp (FIXED)

A user could set age=85, lifeExp=80. The code masked it with `Math.max(lifeExp - age, 1)` — nonsense output. Now a top-priority alert fires when `age >= lifeExp`, and `valid` short-circuits rendering.

---

## Medium (not yet fixed)

### M1. Leftover `.claude/worktrees/` folders

There are four agent worktrees (`competent-gould`, `modest-bose`, `peaceful-joliot`, `peaceful-yonath`) under `.claude/worktrees/`, each with its own copy of `index.html`. These are Cowork agent sandboxes from prior sessions, not something you need to keep. Safe to delete.

### M2. Split the monolith

A single 2,300-line file is manageable but scales badly. Suggested split:
- `index.html` — structure + `<link>` and `<script src>`
- `styles.css` — the ~700 lines of CSS
- `translations.js` — the 5-language i18n tables (~550 lines)
- `app.js` — logic, state, chart rendering

GitHub Pages serves plain static files; no bundler required.

### M3. Magic numbers

`for (let m = 0; m < 1200; m++)` and `Math.min(Math.ceil(yearsToRetire) + 10, 60)` — pull these into named constants:

```js
const MAX_SOLVER_MONTHS = 1200;  // 100-year ceiling on time-to-retire solver
const CHART_BUFFER_YEARS = 10;
const MAX_CHART_YEARS = 60;
```

### M4. Inline styles in HTML

Mixed inline `style=""` and CSS classes. Move inline ones (e.g., `style="display:inline-flex"`) to classes for consistency.

### M5. Variable naming clarity

`income` is monthly, `annualExp = expenses * 12`, but the monthly/annual split isn't obvious. Renaming to `monthlyIncome` and `monthlyExpense` is a small change with big clarity benefit.

---

## Low (nice-to-haves)

### L1. Debounce slider input

Every slider tick re-runs `calculate()` and re-renders Chart.js. On older phones this can stutter. A 150–200ms debounce on `input` would help.

### L2. JSDoc for the math

`solveYearsToTarget` and the main simulation loop deserve 3-line comments explaining the financial meaning.

### L3. Canvas null check

`$('wealthChart').getContext('2d')` assumes the canvas exists. Wrap in `if (!canvas) return;` to fail gracefully if the DOM is ever modified.

---

## Math Verification

- ✅ Fisher equation for real return: `(1 + nominal) / (1 + inflation) - 1`
- ✅ Perpetual target: `annualExp / realReturn` (the standard 25× annual expenses at 4% real return)
- ✅ Compound growth loop: monthly compounding with annual salary step-up
- ✅ `pvAnnuity` (after fix): $40K × 40yr @ 4% real → $791K (correct), $40K × 25yr @ 7% real → $466K (correct)
- ✅ Savings rate: `monthlySav / monthlyIncome × 100`
- ✅ Progress %: `curSav / target × 100`, clamped to 100

---

## Suggested Next Steps

1. ✅ ~~C1: pvAnnuity~~
2. ✅ ~~C2: depletion alert~~
3. ✅ ~~C3 / H4: input guard rails~~
4. ✅ ~~H1: accessibility~~
5. ✅ ~~H2: mobile breakpoint~~
6. M1: clean up `.claude/worktrees/`
7. M2: split into separate CSS/JS files
8. H3: replace inline `onchange=` with `addEventListener`
