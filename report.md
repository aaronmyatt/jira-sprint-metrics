# Report

Aggregate metrics across multiple sprints to show trends and overall developer performance. Reads all cached sprint JSON files from `~/.cache/jira-sprint/`, optionally excluding sprints by name pattern, and produces a cross-sprint summary.

This pipeline does **not** call the Jira API — it relies entirely on locally cached data produced by the Sprint Metrics pipeline. Run `sprintMetrics.md` for each sprint first.

```zod
import { z } from "npm:zod";

const SprintTotals = z.object({
  sprintName: z.string(),
  endDate: z.string(),
  total: z.number(),
  done: z.number(),
  test: z.number(),
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
  totalDone: z.number(),
  totalTest: z.number(),
  totalIncomplete: z.number(),
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

Scan `~/.cache/jira-sprint/` for all `.json` files. Parse each one and collect them into `input.cachedSprints`, sorted by sprint end date ascending. If `input.exclude` is set, filter out any sprint whose name includes the exclusion string.

```ts
```

## Build Sprint Trends

Map each cached sprint into a `SprintTotals` row (sprint name, end date, completion %, commitment %). Store in `input.sprintTrends`. This gives a chronological view of how the team's performance has changed over time.

```ts
```

## Aggregate Developer Performance

Flatten all per-developer summaries from every cached sprint. Group by assignee and sum up their ticket counts, completion counts, and commitment counts across all sprints. Compute `overallCompletionPct` and `overallCommitmentPct` for each developer. Store in `input.aggregatedDevelopers`, sorted by overall completion % descending.

```ts
```

## Compute Grand Totals

Sum across all sprints to produce a single `grandTotals` row — the headline numbers for the entire reporting window.

```ts
```

## Format Output

Shape `input.body` with three sections:

1. **Sprint Trends** — one row per sprint showing completion and commitment percentages
2. **Developer Leaderboard** — aggregated per-developer stats across all sprints
3. **Grand Totals** — the overall numbers

Format for readable CLI output.

```ts
```
