# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

PropROICal is a single-page property ROI calculator for the Malaysian property market: it computes entry costs (stamp duty, MOT, legal fees), monthly cashflow, and a 30-year appreciation/rental projection, plus a separate tenant rental quote generator. It's a static site (Tailwind CDN + Chart.js CDN, no framework, no bundler) deployed to GitHub Pages via a CNAME.

## Critical architecture: source vs. deployed files are split by .gitignore

This repo intentionally keeps editable source out of git and only commits the built/obfuscated output. Always edit the `-clean.js` files, never the plain-named ones.

| Edit this (gitignored, local only) | Never edit this (git-tracked, obfuscated build output) |
|---|---|
| `calculator-clean.js` | `calculator.js` |
| `tenant-rental-quote-clean.js` | `tenant-rental-quote.js` |

`index.html` loads `calculator.js` and `tenant-rental-quote.js` directly — the obfuscated files are what actually run in the browser and in `index.html`'s `<script>` tags. `build.js`, `package.json`, `Notes.txt`, `favicon.png`, `logo.jpg`, and `tenant-rental-quote.html` are also gitignored (local-only). `index.html` itself, however, *is* tracked and edited directly.

The `backup/` directory holds a manual, ad hoc snapshot of older file versions — not part of the build or deploy flow.

## Commands

```bash
npm run build          # obfuscates calculator-clean.js -> calculator.js and tenant-rental-quote-clean.js -> tenant-rental-quote.js
npm run deploy         # build.js, then git add + commit "Update build all" + push to origin main
npm run deploy-html    # commit only index.html changes ("Html only") and push
```

There is no test suite, linter, or type checker configured. Verify changes by opening `index.html` in a browser (or via the browser preview tool) and exercising the calculator manually.

`build.js` runs `javascript-obfuscator` with `compact: true, controlFlowFlattening: true` on both clean files — this is meant to obscure the formulas from casual viewing on the public site, not for security. Since the obfuscated output is committed but the clean source isn't, **always run `npm run build` before committing/deploying** after editing a `-clean.js` file, or the live site will keep serving stale logic.

## Architecture

`index.html` is a tab-based single page (`#tab-simple`, `#tab-projection`, `#tab-about`, `#tab-contact`, `#tab-tenant-rental-quote`), switched via `window.switchTab` in `calculator-clean.js`.

Core calculation flow in `calculator-clean.js`:
- `calculate()` — reads form inputs, computes entry costs (via `calculateStampDutyLocal`) and monthly cashflow for the "Simple" tab.
- `calculateProjection(...)` — builds the 30-year year-by-year projection (property value, loan balance, accumulated rent/expenses, net profit) shown in the "Projection" tab table and chart.
- `getSellingMultiplier(year, propType, propClass)` — the property appreciation model. Multiplier tables differ by `propClass` (`landed` vs `highrise`) and, for highrise, by `propType` (`new` vs resale). This is the piece most likely to need tuning when appreciation assumptions change — see `Notes.txt` for the reasoning/history behind specific multiplier values.
- `updateChart(labels, data)` — renders the projection chart via Chart.js.
- `updateFeePresets()` — toggles default stamp duty/legal fee behavior for new vs. resale properties (new developments often have MOT absorbed by the developer).

`tenant-rental-quote-clean.js` is an independent feature (rental quote generator for tenants) living in its own tab, with its own `generateQuote()` / `copyQuote()` functions — it does not share state with the ROI calculator.

`Notes.txt` (gitignored, local-only) is a running scratchpad of feature requests and assumption notes from the site owner — check it for context on why certain constants (stamp duty tiers, appreciation multipliers, occupancy assumptions) are set the way they are before changing them.
