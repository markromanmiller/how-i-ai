*Written 2026-08-04, based on an interview.*

## Why a waffle chart instead of a bar or pie chart

The core problem with a table of numbers, or even a bar/pie chart, is that reading it requires either doing mental math or mentally comparing bar lengths/pie slices — neither gives an immediate, visceral sense of magnitude. A waffle chart made of many small unit squares lets you *feel* how big a category is by roughly counting or eyeballing area, the same way a pile of coins feels different from a number printed on a receipt. That's the whole reason for choosing this chart type over more conventional options: the tool exists to build intuition during scenario exploration, not to present precise numbers (the numbers are already right there as text).

## Why blocks lean wide, never tall

`WIDE_CAP = 2` (a block can be twice as wide as tall) and `TALL_CAP = 1` (never taller than wide) come directly from the shape of the screen the chart lives in: the editor's chart area has much more horizontal room than vertical room, so a block that's allowed to spread wide fits the available canvas better than one that's forced into a tall, narrow shape. The secondary reason is that wide, "chunky" blocks make relative size differences easier to read at a glance than thin slivers would — a block's shape should help you compare it to its neighbors, not just satisfy a packing constraint.

## Why $20 per square (and why it changed from $100)

The square value started at $100 per full square. At that size, on a typical desktop-width chart, there wasn't enough resolution — most categories collapsed into one or two squares, and the chart didn't use the available space well. Dropping it to $20 gave enough squares per category for the packing algorithm and the "feel the size" goal to actually work. The value was tuned by look and feel against typical monthly budget magnitudes, not derived from any formula — if the chart ever again feels too sparse or too dense, that's the knob to revisit.

The quarter-square ($5) partial-fill rounding granularity is a holdover from the $100-per-square era, when a $25 remainder was visually significant enough to deserve a partial square. At $20/square it's arguably finer than necessary, but it was left as-is rather than being reconsidered when the square value changed.

## Why everything is local-only (localStorage + JSONL, no backend)

This was a deliberate boundary, not a "haven't gotten to it yet" gap. The data in this app is a personal or household budget, and the author did not want that data to live anywhere online — even behind the password protection already used elsewhere for personal projects. Keeping storage entirely local (browser `localStorage`, with JSONL files for backup/portability) means the numbers themselves never leave the machine. This is reinforced by what the tool is actually *for*: the valuable output of using it isn't the budget numbers, it's the decision a person reaches after playing with scenarios — so there's no real loss in not persisting or syncing the numbers anywhere else.

## Why scenarios aren't scoped to a specific budget

Scenarios are stored as a flat, budget-agnostic list rather than being tied to the budget they were built against. This was a simplicity-of-development choice: a scenario row (`{ category, type, current, proposed, method }`) intentionally carries no linking metadata back to a specific budget — just enough fields to describe "this category changes from X to Y, for this reason." Adding a budget reference wasn't judged necessary at the time. This does mean a scenario can be applied against a mismatched budget (matching is by category name only), but that trade-off was accepted implicitly rather than weighed against alternatives — it's the simplest data shape that supported the feature, not a considered design decision to keep scenarios reusable across budgets.

## Why category matching is by exact string name

Budget items and scenario rows are linked purely by matching `category` string, with no shared ID between a budget's item and a scenario's row. This was simply the simplest mechanism that worked when the feature was built, not a considered trade-off around portability or any other concern.

## What the "method / reason" field is for

The free-text method field on each scenario row (e.g. "Starting consulting business - requires 5h/wk") isn't read by any calculation — it exists purely as documentation. Its purpose is to preserve the reasoning behind a proposed change for whoever revisits the scenario later, whether that's the same person after time has passed or someone else the scenario is shared with. Without it, a proposed number change would be indistinguishable from an arbitrary guess once the context that motivated it is forgotten.
