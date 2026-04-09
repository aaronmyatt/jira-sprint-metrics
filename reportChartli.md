# Report With Chartli Graphs

Aggregate metrics across multiple sprints and render them as terminal charts via
[`@atm/chartli`](https://jsr.io/@atm/chartli) — a small, dependency-free chart
renderer that exposes its renderers as plain functions (in addition to a CLI).
The output is the same shape as [reportGraph.md](reportGraph.md): a set of
ready-to-print multi-line strings, each containing one chart.

This pipeline does **not** call the Jira API — it relies entirely on the
structured data returned by `sprintsOverview` (which itself reads from
`~/.cache/jira-sprints-metrics.json`).

Why chartli rather than hand-rolled bars + `asciichart`? One dep, one mental
model: every chart type takes `{ normalized, options }` and returns a string.
Ref: https://jsr.io/@atm/chartli

```zod
// output
//
// Each field is a ready-to-print multi-line string containing a chart.
// We keep them as strings (rather than structured chart objects) so the caller
// can concatenate them directly into a report body without any further rendering.
const GraphDefs = z.object({
  grandTotalsGraph: z.string().optional(),
  sprintTrendsGraph: z.string().optional(),
  leaderboardGraph: z.string().optional(),
  developerSprintGraphs: z.map(z.string(), z.string()).optional(),
  developerTrendsGraph: z.string().optional(),
});
```

## Get Reportable Data

Delegate all data gathering + aggregation to `sprintsOverview`. This keeps the
presenter stateless with respect to caching, filtering, and cross-sprint math —
its only job is to turn numbers into pictures.

```ts
import sprintsOverview from "sprintsOverview";
Object.assign(input, await sprintsOverview.process());
```

## Present Grand Totals

Render the overall completion vs. commitment picture as a horizontal bar chart.
A bar chart (rather than a line chart) is the right choice here because we have
a handful of **independent categorical values**, not a time series.

We use `renderBars` from chartli. One important nuance: chartli's
`normalizeData()` helper normalises each *column* to its own `[0, 1]` range,
which is great for plotting unrelated series on a shared chart, but **wrong**
when every bar should be a fraction of the same denominator (here, the total
ticket count). So we build a `NormalizeResult` by hand — chartli is happy to
consume any object that matches the shape, since each renderer is just a pure
function over `{ normalized, options }`.
Ref: https://jsr.io/@atm/chartli/doc/~/renderBars

- if: /grandTotals
  ```ts
  // jsr: imports work natively in Deno — no install step required.
  // Ref: https://docs.deno.com/runtime/manual/basics/modules/#jsr
  import { renderBars } from "jsr:@atm/chartli@1.0.0";

  const t = input.grandTotals;

  // Two conceptually distinct bar groups, rendered as one chart each:
  //   1. Completion breakdown  — how many tickets got done at all
  //   2. Commitment breakdown  — of the done tickets, how on-time they were
  // Both groups share the same denominator (`t.total`) so the bar lengths are
  // directly comparable across the two groups. We render them as separate
  // chartli charts so each can carry its own yAxisLabel as a heading.
  const completionLabels = ["Total", "Done", "Incomplete"];
  const completionValues = [t.total, t.done, t.incomplete];

  const commitmentLabels = ["Met", "Acceptable", "Late"];
  const commitmentValues = [t.met, t.acceptable, t.late];

  /**
   * Build a chartli `NormalizeResult` whose every column is scaled against the
   * same maximum (so the bars are comparable). chartli's renderBars only looks
   * at the *last* row of `data` for the bar lengths, so a single-row dataset
   * is exactly what we need here.
   *
   * @param {number[]} values  Raw category values.
   * @param {number}   max     The shared denominator (full-width bar value).
   * @returns {{data: number[][], raw: number[][], min: number[], max: number[]}}
   */
  const buildNormalized = (values, max) => ({
    // Clamp into [0, 1]; guard against divide-by-zero on empty sprints.
    data: [values.map(v => (max > 0 ? Math.max(0, Math.min(1, v / max)) : 0))],
    raw: [values],
    // `min` / `max` arrays are per-column. renderBars doesn't read them, but
    // we populate them honestly for any other consumers (e.g. renderAscii).
    min: values.map(() => 0),
    max: values.map(() => max),
  });

  // `showDataLabels: true` makes renderBars print the *raw* value (from `raw`)
  // next to each bar instead of the normalised 0–1 ratio. This is exactly the
  // ticket-count display we want.
  // Ref: https://jsr.io/@atm/chartli/doc/~/BarsOptions
  const completionChart = renderBars({
    normalized: buildNormalized(completionValues, t.total),
    options: {
      width: 40,
      seriesLabels: completionLabels,
      showDataLabels: true,
      yAxisLabel: "Overall Totals — Completion",
    },
  });

  const commitmentChart = renderBars({
    normalized: buildNormalized(commitmentValues, t.total),
    options: {
      width: 40,
      seriesLabels: commitmentLabels,
      showDataLabels: true,
      yAxisLabel: "Overall Totals — Commitment",
    },
  });

  input.grandTotalsGraph = [
    completionChart,
    "",
    commitmentChart,
    "",
    `Completion: ${t.completionPct.toFixed(1)}%   Commitment: ${t.commitmentPct.toFixed(1)}%`,
  ].join("\n");
  ```

## Present Sprint Trends

A time series of completion % and commitment % per sprint is the canonical
"is the team getting better or worse?" chart — so it deserves a real line plot.
chartli's `renderAscii` is exactly that: a multi-series line chart drawn with
unicode marker characters (`●`, `○`, `◆`, …) and box-drawing axes.
Ref: https://jsr.io/@atm/chartli/doc/~/renderAscii

```ts
import { renderAscii } from "jsr:@atm/chartli@1.0.0";

// Sprints are already sorted chronologically by `sprintsOverview`, so we can
// just zip them into rows. chartli expects rows × cols where each row is one
// x-position and each col is one series.
const sprints = input.cachedSprints;
const raw = sprints.map(s => [s.totals.completionPct, s.totals.commitmentPct]);

// Anchor both series to the same [0, 100] scale by hand. If we used
// `normalizeData()` instead, each series would be normalised to its own
// observed min/max — which would *visually* exaggerate small fluctuations and
// destroy the comparison between completion % and commitment %.
const data = raw.map(([completion, commitment]) => [completion / 100, commitment / 100]);

const normalized = {
  data,
  raw,
  // Per-column min/max — renderAscii only uses these for tick labels when
  // there's a single series, so they're effectively decorative here, but we
  // set them honestly to keep the data structure self-describing.
  min: [0, 0],
  max: [100, 100],
};

// `height` controls the number of rows in the plot; 12 gives enough vertical
// resolution to see a 5–10% swing without eating the terminal. `width` is
// generous so dense sprint timelines stay readable.
// `xLabels` puts a sprint index under each data point — we keep them numeric
// (1, 2, 3 …) because real sprint names are too long to fit on one row;
// the legend below the chart maps the numbers back to names.
const chart = renderAscii({
  normalized,
  options: {
    width: 60,
    height: 12,
    seriesLabels: ["Completion %", "Commitment %"],
    yAxisLabel: "Sprint Trends — Completion vs Commitment (0–100%)",
    xLabels: sprints.map((_, i) => String(i + 1)),
    xAxisLabel: "Sprint #",
  },
});

// renderAscii's y-axis ticks are hard-coded to '1.00' / '0.50' / '0.00' when
// there's more than one series, so we add a small "× 100 = %" hint plus a
// numbered list of sprint names so a reader can map peaks/troughs to specific
// sprints.
const legend = [
  "(y-axis ticks are 0.00–1.00; multiply by 100 to read as %)",
  "Sprints (left → right):",
  ...sprints.map((s, i) => `  ${i + 1}. ${s.name}`),
].join("\n");

input.sprintTrendsGraph = [chart, "", legend].join("\n");
```

## Present Developer Leaderboard

One horizontal bar per developer, ranked by completion percentage. A bar chart
beats a line chart here because developers are **unordered categories**, not a
sequence — there is no meaning to "the line going up between Alice and Bob".

We render two `renderBars` charts side-by-side conceptually (one for completion
%, one for commitment %) so the eye can compare each developer on both axes
without needing a multi-series bar layout.

```ts
// Build leaderboard rows from the pre-aggregated Map produced by sprintsOverview.
// We compute completion % on the fly rather than trusting a possibly-missing
// `overallCompletionPct` field — defensive, and cheap.
const leaderboardRows = [...input.compileLeaderboard.values()]
  .map(dev => ({
    assignee: dev.assignee,
    completionPct: dev.total > 0 ? (dev.done / dev.total) * 100 : 0,
    commitmentPct: dev.total > 0 ? ((dev.met + dev.acceptable) / dev.total) * 100 : 0,
    total: dev.total,
  }))
  // Sort descending so the best performers are at the top of the chart — this
  // matches how readers scan leaderboards (top = winner).
  .sort((a, b) => b.completionPct - a.completionPct);

/**
 * Render one chartli horizontal-bar chart where each "column" is a developer.
 * Bars are scaled to 100 (not to the max value in the series) so the chart
 * reads as "% of perfect" rather than "% of the best developer". The latter
 * would make a team of equally-mediocre developers all look like heroes.
 *
 * @param {(r: object) => number} pick  Selector for the metric to plot.
 * @param {string} title                Heading rendered above the bars.
 * @returns {string}                    Multi-line ASCII chart.
 */
const renderDeveloperBars = (pick, title) => {
  const values = leaderboardRows.map(pick);
  return renderBars({
    normalized: {
      data: [values.map(v => Math.max(0, Math.min(1, v / 100)))],
      raw: [values.map(v => Number(v.toFixed(1)))],
      min: values.map(() => 0),
      max: values.map(() => 100),
    },
    options: {
      width: 40,
      seriesLabels: leaderboardRows.map(r => r.assignee),
      // showDataLabels prints the raw % next to each bar, so the reader gets
      // both a visual length comparison *and* the exact number.
      showDataLabels: true,
      yAxisLabel: title,
    },
  });
};

input.leaderboardGraph = [
  renderDeveloperBars(d => d.completionPct, "Developer Leaderboard — Completion %"),
  "",
  renderDeveloperBars(d => d.commitmentPct, "Developer Leaderboard — Commitment % (met + acceptable on time)"),
].join("\n");
```

## Developer Sprint Graphs

For each sprint, render a mini bar chart of per-developer completion %. This
gives the same granular view as the per-sprint tables in
[reportTable.md](reportTable.md), but in a form that lets the eye compare bar
lengths at a glance rather than scanning columns of numbers.

```ts
// Map<sprintName, chartString> — mirrors the shape of `developerSprintTables`
// in reportTable.md so the two presenters are interchangeable from the
// caller's perspective.
input.developerSprintGraphs = input.cachedSprints
  .filter(sprint => sprint.developerSummaries && sprint.developerSummaries.length > 0)
  .reduce((acc, sprint) => {
    const rows = sprint.developerSummaries
      .map(dev => ({
        assignee: dev.assignee,
        completionPct: dev.total > 0 ? (dev.done / dev.total) * 100 : 0,
      }))
      .sort((a, b) => b.completionPct - a.completionPct);

    const values = rows.map(r => r.completionPct);

    // Same hand-rolled NormalizeResult trick as in Grand Totals: every bar is
    // a fraction of 100 so they're comparable across developers within the
    // same sprint *and* across sprints.
    const chart = renderBars({
      normalized: {
        data: [values.map(v => Math.max(0, Math.min(1, v / 100)))],
        raw: [values.map(v => Number(v.toFixed(1)))],
        min: values.map(() => 0),
        max: values.map(() => 100),
      },
      options: {
        width: 30,
        seriesLabels: rows.map(r => r.assignee),
        showDataLabels: true,
        yAxisLabel: `Sprint: ${sprint.name}`,
      },
    });

    return acc.set(sprint.name, chart);
  }, new Map());
```

## Developer Trends

One sparkline per developer showing their completion % across sprints in
chronological order. This is the "is each individual improving or declining?"
view that the leaderboard (aggregate) and per-sprint graphs (snapshot) can't
answer on their own.

`renderSpark` is the right tool here: it produces one compact line per series
(here, per developer), so a team of N developers fits in N lines — far more
scannable than N stacked bar charts.
Ref: https://jsr.io/@atm/chartli/doc/~/renderSpark

```ts
import { renderSpark } from "jsr:@atm/chartli@1.0.0";

// Pick the developer set + ordering from the leaderboard so the sparklines
// list reads top-down in the same rank order as the leaderboard chart above.
// Visually, this lets the reader's eye flow from "rank" to "trend" without
// having to re-orient between the two sections.
const orderedDevs = [...input.compileLeaderboard.values()]
  .map(d => ({
    assignee: d.assignee,
    completionPct: d.total > 0 ? (d.done / d.total) * 100 : 0,
  }))
  .sort((a, b) => b.completionPct - a.completionPct)
  .map(d => d.assignee);

// Build a sprint × developer matrix of completion %.
//   rows[i] = sprint i (chronological — sprintsOverview already sorted these)
//   cols[j] = developer j (in leaderboard rank order)
// Devs that didn't ship anything in a given sprint get 0. Treating "absent"
// as "0%" is a small lie, but it keeps every sparkline the same length so
// they line up under each other — a gap-based representation would break
// alignment and make the chart much harder to scan.
const sprintRows = input.cachedSprints.map(sprint => {
  const byDev = new Map(
    (sprint.developerSummaries ?? []).map(d => [
      d.assignee,
      d.total > 0 ? (d.done / d.total) * 100 : 0,
    ]),
  );
  return orderedDevs.map(dev => byDev.get(dev) ?? 0);
});

// Anchor every column (= developer) to the same [0, 100] scale by hand.
// If we used chartli's normalizeData() the per-column auto-scaling would
// rescale each developer's series to its own observed min/max — so a dev
// who oscillated between 70% and 75% would *look* like the most volatile
// person on the team. The whole point of this chart is absolute trend, so
// we lock min=0, max=100 for every column.
const normalized = {
  data: sprintRows.map(row => row.map(v => Math.max(0, Math.min(1, v / 100)))),
  raw: sprintRows.map(row => row.map(v => Number(v.toFixed(1)))),
  min: orderedDevs.map(() => 0),
  max: orderedDevs.map(() => 100),
};

// `showDataLabels: true` makes renderSpark print the *latest* sprint's value
// at the end of each line (it reads `raw[lastRow][colIdx]`), so the reader
// gets both the historical trend and the current number on a single row.
input.developerTrendsGraph = [
  renderSpark({
    normalized,
    options: {
      seriesLabels: orderedDevs,
      showDataLabels: true,
      yAxisLabel: "Developer Completion % Trends (left = oldest sprint → right = newest)",
    },
  }),
  "",
  // Footer maps the leftmost-to-rightmost spark cells back to actual sprint
  // names, mirroring the legend pattern used in Sprint Trends above.
  "Sprints (left → right):",
  ...input.cachedSprints.map((s, i) => `  ${i + 1}. ${s.name}`),
].join("\n");
```

## Format Output

Validate that we produced exactly the shape promised in the `GraphDefs` schema
at the top of this file. Parsing (rather than `.safeParse()`) is intentional —
if a presenter section silently failed and left a field undefined *when it
should have been set*, we want to hear about it loudly at the boundary rather
than ship a half-rendered report.

The assembled report body, when printed top-to-bottom, reads as:

1. **Grand Totals Graph** — overall numbers as horizontal bars
2. **Sprint Trends Graph** — completion & commitment % over time as a line chart
3. **Developer Leaderboard Graph** — per-developer bars, ranked by aggregate
4. **Developer Sprint Graphs** — per-sprint per-developer breakdown
5. **Developer Trends Graph** — one sparkline per developer, showing each
   individual's completion % across sprints in chronological order

```ts
input = GraphDefs.parse(input);
```
