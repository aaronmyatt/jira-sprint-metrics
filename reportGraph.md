# Report With ASCII Graphs

Aggregate metrics across multiple sprints and render them as ASCII charts suitable
for CLI output, email bodies, or any monospace-friendly medium. Reads the same
reportable data shape as [reportTable.md](reportTable.md) — it simply chooses a
different visual representation.

This pipeline does **not** call the Jira API — it relies entirely on the structured
data returned by `sprintsOverview` (which itself reads from
`~/.cache/jira-sprints-metrics.json`).

```zod
// output
//
// Each field is a ready-to-print multi-line string containing an ASCII chart.
// We keep them as strings (rather than structured chart objects) so the caller
// can concatenate them directly into a report body without any further rendering.
const GraphDefs = z.object({
  grandTotalsGraph: z.string().optional(),
  sprintTrendsGraph: z.string().optional(),
  leaderboardGraph: z.string().optional(),
  developerSprintGraphs: z.map(z.string(), z.string()).optional(),
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

We draw the bars by hand using Unicode block characters — this avoids pulling in
a charting dep for what is essentially `'█'.repeat(n)`. Block elements give us
sub-character resolution via the "eighths" family (`▏▎▍▌▋▊▉█`), which looks much
smoother than ASCII `#`/`=`.
Ref: https://en.wikipedia.org/wiki/Block_Elements

- if: /grandTotals
  ```ts
  /**
   * Render a single horizontal bar line.
   *
   * @param {string} label  Left-aligned row label.
   * @param {number} value  The raw value to display next to the bar.
   * @param {number} max    The value that corresponds to a full-width bar.
   * @param {number} width  Total bar width in character cells (default 40).
   * @returns {string}      One line: "label | ███████░░░░ value".
   */
  const bar = (label, value, max, width = 40) => {
    // Guard against divide-by-zero when every sprint is empty — a zero-length
    // bar is more honest than NaN-length garbage.
    const ratio = max > 0 ? value / max : 0;
    // Full blocks + a trailing partial block for sub-cell precision.
    // 8 eighths per cell, so `ratio * width * 8` = total eighths to draw.
    const eighths = Math.round(ratio * width * 8);
    const full = Math.floor(eighths / 8);
    const remainder = eighths % 8;
    // U+2588 FULL BLOCK … U+258F LEFT ONE EIGHTH BLOCK
    // The partials go from wide (▉) to thin (▏) as `remainder` shrinks.
    const partials = [" ", "▏", "▎", "▍", "▌", "▋", "▊", "▉"];
    const filled = "█".repeat(full) + (remainder ? partials[remainder] : "");
    const padded = filled.padEnd(width, " ");
    return `${label.padStart(14)} │ ${padded} ${value}`;
  };

  const t = input.grandTotals;

  // Two conceptually distinct bar groups joined into a single code block:
  //   1. Completion breakdown  — how many tickets got done at all
  //   2. Commitment breakdown  — of the done tickets, how on-time they were
  // `max` is the total ticket count so the bars stay comparable row-to-row.
  input.grandTotalsGraph = [
    "Overall Totals Across All Sprints",
    "─".repeat(60),
    bar("Total",      t.total,      t.total),
    bar("Done",       t.done,       t.total),
    bar("Incomplete", t.incomplete, t.total),
    "",
    bar("Met",        t.met,        t.total),
    bar("Acceptable", t.acceptable, t.total),
    bar("Late",       t.late,       t.total),
    "",
    `Completion: ${t.completionPct.toFixed(1)}%   Commitment: ${t.commitmentPct.toFixed(1)}%`,
  ].join("\n");
  ```

## Present Sprint Trends

A time series of completion % and commitment % per sprint is the canonical
"is the team getting better or worse?" chart — so it deserves a real line plot.

We use [`asciichart`](https://github.com/kroitor/asciichart) for this. It's a
tiny, dependency-free library that draws multi-series line charts with axis
labels using only box-drawing characters. Importing via `npm:` works in Deno
out of the box.
Ref: https://docs.deno.com/runtime/fundamentals/node/#using-npm-packages

```ts
import asciichart from "npm:asciichart@1.5.25";

// Sprints are already sorted chronologically by `sprintsOverview`, so we can
// just map to the two series we want. `asciichart.plot` expects arrays of
// numbers — one array per series.
const completionSeries = input.cachedSprints.map(s => s.totals.completionPct);
const commitmentSeries = input.cachedSprints.map(s => s.totals.commitmentPct);

// `height` controls the number of rows in the plot. 10 rows gives us enough
// vertical resolution to see a 5–10% swing without eating the terminal.
// `colors` is intentionally omitted — we want plain text that survives copy/paste
// into Slack, email, markdown, etc. without ANSI escape sequences.
const plot = asciichart.plot([completionSeries, commitmentSeries], {
  height: 10,
  format: (x) => (`      ${x.toFixed(1)}%`).slice(-7),
});

// asciichart doesn't label the x-axis, so we append a compact legend + a list
// of sprint names in chronological order. This is usually enough context for a
// reader to map peaks/troughs back to specific sprints.
const legend = [
  "Legend:  line 1 = Completion %   line 2 = Commitment %",
  "Sprints (left → right):",
  ...input.cachedSprints.map((s, i) => `  ${i + 1}. ${s.name}`),
].join("\n");

input.sprintTrendsGraph = [
  "Sprint Trends",
  "─".repeat(60),
  plot,
  "",
  legend,
].join("\n");
```

## Present Developer Leaderboard

One horizontal bar per developer, ranked by completion percentage. A bar chart
beats a line chart here because developers are **unordered categories**, not a
sequence — there is no meaning to "the line going up between Alice and Bob".

We reuse the same `bar()` helper idea as in Grand Totals but inline it here so
each section of this pipeline stays independently readable (these are separate
code blocks in the pipe runtime, so top-level `const`s don't cross boundaries).

```ts
const bar = (label, value, max, width = 40) => {
  const ratio = max > 0 ? value / max : 0;
  const eighths = Math.round(ratio * width * 8);
  const full = Math.floor(eighths / 8);
  const remainder = eighths % 8;
  const partials = [" ", "▏", "▎", "▍", "▌", "▋", "▊", "▉"];
  const filled = "█".repeat(full) + (remainder ? partials[remainder] : "");
  return `${label.padStart(20)} │ ${filled.padEnd(width, " ")} ${value.toFixed(1)}%`;
};

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

// Bars are scaled to 100 (not to the max value in the series) so the chart
// reads as "% of perfect" rather than "% of the best developer". The latter
// would make a team of equally-mediocre developers all look like heroes.
input.leaderboardGraph = [
  "Developer Leaderboard (by Completion %)",
  "─".repeat(60),
  ...leaderboardRows.map(d => bar(d.assignee, d.completionPct, 100)),
  "",
  "Commitment % (met + acceptable on time):",
  ...leaderboardRows.map(d => bar(d.assignee, d.commitmentPct, 100)),
].join("\n");
```

## Developer Sprint Graphs

For each sprint, render a mini bar chart of per-developer completion %. This
gives the same granular view as the per-sprint tables in
[reportTable.md](reportTable.md), but in a form that lets the eye compare bar
lengths at a glance rather than scanning columns of numbers.

```ts
const bar = (label, value, max, width = 30) => {
  const ratio = max > 0 ? value / max : 0;
  const eighths = Math.round(ratio * width * 8);
  const full = Math.floor(eighths / 8);
  const remainder = eighths % 8;
  const partials = [" ", "▏", "▎", "▍", "▌", "▋", "▊", "▉"];
  const filled = "█".repeat(full) + (remainder ? partials[remainder] : "");
  return `${label.padStart(20)} │ ${filled.padEnd(width, " ")} ${value.toFixed(1)}%`;
};

// Map<sprintName, asciiChartString> — mirrors the shape of
// `developerSprintTables` in reportTable.md so the two presenters are
// interchangeable from the caller's perspective.
input.developerSprintGraphs = input.cachedSprints
  .filter(sprint => sprint.developerSummaries && sprint.developerSummaries.length > 0)
  .reduce((acc, sprint) => {
    const rows = sprint.developerSummaries
      .map(dev => ({
        assignee: dev.assignee,
        completionPct: dev.total > 0 ? (dev.done / dev.total) * 100 : 0,
      }))
      .sort((a, b) => b.completionPct - a.completionPct);

    const chart = [
      `Sprint: ${sprint.name}`,
      "─".repeat(60),
      ...rows.map(d => bar(d.assignee, d.completionPct, 100)),
    ].join("\n");

    return acc.set(sprint.name, chart);
  }, new Map());
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
3. **Developer Leaderboard Graph** — per-developer bars, ranked
4. **Developer Sprint Graphs** — per-sprint per-developer breakdown

```ts
input = GraphDefs.parse(input);
```
