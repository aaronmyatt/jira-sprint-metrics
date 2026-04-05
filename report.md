# Report

Aggregate metrics across multiple sprints to show trends and overall developer performance. Reads all cached sprint JSON files from `~/.cache/jira-sprints-metrics.json`, optionally excluding sprints by name pattern, and produces a cross-sprint summary.

This pipeline does **not** call the Jira API — it relies entirely on locally cached, or precomputed data passed via the `input` object.

```
import { z } from "npm:zod";

const SprintTotals = z.object({
  sprintName: z.string(),
  endDate: z.string(),
  total: z.number(),
  done: z.number(),
  incomplete: z.number(),
  completionPct: z.number(),
  met: z.number(),
  acceptable: z.number(),
  late: z.number(),
  commitmentPct: z.number(),
});

const AggregatedDeveloper = z.object({
  assignee: z.string(),
  totalTickets: z.number(),
  totalDone: z.number(),  totalIncomplete: z.number(),
  overallCompletionPct: z.number(),
  totalMet: z.number(),
  totalAcceptable: z.number(),
  totalLate: z.number(),
  overallCommitmentPct: z.number(),
  sprintCount: z.number(),
});

export const schema = z.object({
  boardId: z.number().optional(),
  exclude: z.string().default(""),
  cachedSprints: z.array(z.any()).default([]),
  sprintTrends: z.array(SprintTotals).default([]),
  aggregatedDevelopers: z.array(AggregatedDeveloper).default([]),
  grandTotals: SprintTotals.optional(),
});
```

```json
{
  "inputs": [
    { "_name": "basic" },
    { "_name": "with exclusion", "exclude": "Planning Sprint" }
  ]
}
```

## Load All Cached Sprints

Scan `~/.cache/jira-sprints-metrics.json`. If `input.exclude` is set, filter out any sprint whose name includes the exclusion string. Skip this step if `input.cachedSprints` is already populated (e.g. by a previous step or test setup) to avoid unnecessary file reads during development.

- not: /cachedSprints
  ```ts
  const allSprints = JSON.parse(await Deno.readTextFile(".cache/jira-sprints-metrics.json"));
  input.cachedSprints = allSprints.filter(sprint => !sprint.name.includes(input.exclude));
  ```


## Load Cached Totals

Also read the pre-computed totals from `~/.cache/jira-sprints-metrics-totals.json` if it exists, so we can include grand totals in the final report without needing to recompute them here.

- not: /totals
  ```ts
  try {
    input.grandTotals = JSON.parse(await Deno.readTextFile(".cache/jira-sprints-metrics-totals.json"));
  } catch {
    input.grandTotals = null;
  }
  ```

## Build Sprint Trends

Map each cached sprint into a `SprintTotals` row (sprint name, end date, completion %, commitment %). Store in `input.sprintTrends`. This gives a chronological view of how the team's performance has changed over time.

```ts
input.cachedSprints = input.cachedSprints.toSorted((a, b) => new Date(a.endDate).getTime() - new Date(b.endDate).getTime());
input.sprintTrends = input.cachedSprints.map(sprint => ({
  sprintName: sprint.name,
  endDate: sprint.endDate || "On going or future sprint",
  ...sprint.totals,
}));
```

## Present Totals
If `input.grandTotals` is available, it can be included in the final report to show the overall completion and commitment percentages across all sprints, giving a high-level summary of team performance.

- if: /grandTotals
  ```ts
  import { Text } from "jsr:@dep/table";

  input.grandTotalsTitle = "Overall Totals Across All Sprints";
  input.grandTotalsTable = new Text()
    .add("Total Tickets", "Done", "Incomplete", "Completion %", "Met Due Date", "Acceptable", "Late", "Commitment %")
    .add(input.grandTotals.total, input.grandTotals.done, input.grandTotals.incomplete, `${input.grandTotals.completionPct.toFixed(1)}%`, input.grandTotals.met, input.grandTotals.acceptable, input.grandTotals.late, `${input.grandTotals.commitmentPct.toFixed(1)}%`)
    .build();


  ```

## Format Output

Shape `input.body` with three sections:

1. **Sprint Trends** — one row per sprint showing completion and commitment percentages
2. **Developer Leaderboard** — aggregated per-developer stats across all sprints
3. **Grand Totals** — the overall numbers

Format for readable CLI output.

```ts

```
