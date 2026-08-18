# SYSARCH — Budget Audit

## Overview

Budget Audit is a single-page, client-side-only web app for building a monthly income/expense budget and exploring "what if" modifications to it. Users paste category/amount pairs from a spreadsheet, save the result as a named **budget**, then in a second tab visualize that budget as a "waffle chart" (a grid of squares, each worth a fixed dollar amount) and layer named **scenarios** of proposed changes on top — e.g. "cut groceries by $200, reason: meal prep." Everything lives in the browser: no server, no build step, no external dependencies. Data persists in `localStorage` and can be exported/imported as JSONL files for backup or sharing.

## How it works

The app is a static HTML page (`index.html`) with two tabs, switched by plain show/hide (`.tab-panel.active`):

- **Upload tab** — paste income and expenses (tab- or comma-separated `category, amount` lines), preview the parsed rows, save them as a named budget in `localStorage`, and manage (load/rename/export/delete) previously saved budgets. Budgets can also be imported from a `.jsonl` file.
- **Editor tab** — pick an active budget and an active scenario, view both as waffle charts (income row and expenses row), and add/edit/delete per-category modifications via a modal ("Proposed Modifications" table in the sidebar). Modifications are the diff between a budget's baseline amounts and a scenario's proposed amounts, each with a free-text "method/reason" field.

All modules attach themselves to a shared global namespace, `window.BA`, as IIFEs (`BA.storage`, `BA.color`, `BA.waffle`, `BA.upload`, `BA.editor`, `BA.main`). Script load order in `index.html` (storage → color → waffle → upload → editor → main) establishes the dependency order; there is no module bundler.

`BA.main` wires up tab switching and a debounced window-resize handler that re-renders the editor's waffle charts when active. `BA.upload.saveBudget()` and other mutating actions call `BA.main.onBudgetsChanged()` to keep the editor's dropdowns and chart in sync if the editor tab becomes active later.

## Data model

Stored in `localStorage` under four keys (`js/storage.js`):

| Key | Shape | Purpose |
|---|---|---|
| `ba_budgets` | `{ [budgetName]: { income: Item[], expenses: Item[] } }` | All saved budgets |
| `ba_activeBudget` | `string \| null` | Name of the currently selected budget |
| `ba_scenarios` | `{ [scenarioName]: ScenarioRow[] }` | All saved scenarios (global, not scoped to a budget) |
| `ba_activeScenario` | `string \| null` | Name of the currently selected scenario |

```
Item          = { id, category, amount }
ScenarioRow   = { id, category, type: "income"|"expense", current, proposed, method }
```

IDs are generated with `BA.storage.makeId()` (random base36 + timestamp, not a UUID — collision risk is treated as negligible for a single-user local app).

**Budgets** hold baseline amounts only. **Scenarios** are a flat, budget-agnostic list of proposed overrides — a scenario is not tied to the budget it was created against, so switching budgets while a scenario is active can pair mismatched category names (see Known issues).

The editor computes what's actually displayed via `effectiveItems(type)` (`js/editor.js`): start from the active budget's items, then overlay any scenario row of the matching type by category name, replacing the amount with `proposed` and flagging it `modified: true`. A scenario row whose category doesn't exist in the budget effectively adds a new category to the chart.

### JSONL interchange format

Both budgets and scenarios can be exported/imported as newline-delimited JSON:

- Budget line: `{"type":"income"|"expense","category":"...","amount":123}`
- Scenario line: `{"category":"...","type":"income"|"expense","current":123,"proposed":100,"method":"..."}`

Import re-mints IDs; malformed lines (bad JSON, missing category, non-numeric amount for budgets) are silently skipped.

## Files / components

| File | Responsibility |
|---|---|
| `index.html` | Page structure, both tabs, the modification modal, script includes |
| `styles.css` | All styling; dark theme via CSS custom properties in `:root` |
| `js/storage.js` | `localStorage` read/write, budget/scenario CRUD, paste parsing, JSONL (de)serialization, file download, ID generation |
| `js/color.js` | Deterministic per-category color: hashes the category string to a hue (0–359), returns a dimmer HSL for baseline squares and a brighter one for modified squares |
| `js/waffle.js` | The waffle-chart engine: converts a dollar amount to a count of squares, packs categories into a treemap-like grid ("strip" packing), and renders/tooltips/labels the result. See below. |
| `js/upload.js` | Upload tab behavior: paste preview, save/list/rename/export/delete budgets, budget import |
| `js/editor.js` | Editor tab behavior: budget/scenario selection, effective-item computation, chart rendering trigger, scenario table, add/edit modal, scenario export/import |
| `js/main.js` | Tab switching, resize handling, app bootstrap (`DOMContentLoaded`), cross-module refresh signaling |

### The waffle chart engine (`js/waffle.js`)

This is the most algorithmically involved part of the app:

1. **Cell sizing** — each category's dollar amount is converted to a number of "cells" at `SQUARE_VALUE = $20` per full square, with a quarter-square-granularity partial fill for the remainder (`computeCellInfo`). Amounts under one quarter-square still get a single hollow "marker" cell so they remain visible/clickable.
2. **Packing** — categories with 2+ cells ("big" entries) are grouped into roughly square-ish horizontal strips using a greedy strip-treemap algorithm (`groupIntoStrips` / `layoutStrip` / `worstBadness`), tuned so blocks can be up to 2x wider than tall but never taller than wide (`WIDE_CAP` / `TALL_CAP`). This lets one dominant category (e.g. a salary) spread wide instead of being forced tall.
3. **Gap filling** — single-cell categories are dropped into leftover cells inside the big blocks' bounding boxes first (`gapPool`), and any that don't fit are shelf-packed after (`shelfPack`).
4. **Shared scale** — `computeLayout` returns an abstract cell-unit grid (no pixel size yet). The editor computes layouts for the income and expense rows independently, then takes the *smaller* of the two rows' ideal cell-pixel sizes (`idealCellPx`) so one square represents the same dollar amount in both rows.
5. **Rendering** — `draw()` renders the shared-scale layout to absolutely-positioned DOM blocks of CSS grid squares, with an instant custom tooltip (bypassing the native `title` delay), inline category labels on wide-enough blocks, and a double-click handler that opens the modification modal.
6. **Totals** — `renderTotals()` shows income/expense/net numeric totals plus a small waffle of the net amount using the same shared cell scale.

## External dependencies

None. No `package.json`, no build tooling, no CDN scripts or fonts. Pure HTML/CSS/vanilla JS, run by opening `index.html` directly or serving the directory statically.

## Operational commands

There is no build or test suite. To run the app, serve or open `index.html` in a browser, e.g.:

```
python3 -m http.server 8000   # then open http://localhost:8000
```

## Known issues / gaps

- **Scenarios are not scoped to a budget.** The `ba_scenarios` store is a single flat namespace independent of `ba_budgets`, so a scenario built for one budget can be silently applied against a different budget with different (or absent) categories, since matching is purely by category-name string.
- **No delete-scenario UI.** `BA.storage.deleteScenario` exists but nothing in the UI calls it (only create/rename/import/export are wired up).
- **No validation on import.** JSONL import silently drops malformed lines rather than surfacing errors to the user.
- **ID collisions** are possible in principle (`makeId()` is not a cryptographic or monotonic UUID) but low-risk for a single-user local app.
