# Report With Tables

Aggregate metrics across multiple sprints to show trends and overall developer performance. Reads all cached sprint JSON files from `~/.cache/jira-sprints-metrics.json`, optionally excluding sprints by name pattern, and produces a cross-sprint summary.

This pipeline does **not** call the Jira API — it relies entirely on locally cached, or precomputed data passed via the `input` object.

```zod
// output 
const TableDefs = z.object({
  grandTotalsTitle: z.string().optional(),
  grandTotalsTable: z.string().optional(),
  sprintTrendsTable: z.string().optional(),
  leaderboardTable: z.string().optional(),
  developerSprintTables: z.map(z.string(), z.string()).optional()
});
```

## Get Reportable Data

```ts
import sprintsOverview from "sprintsOverview";
Object.assign(input, await sprintsOverview.process());
```

## Present Totals
If `input.grandTotals` is available, it can be included in the final report to show the overall completion and commitment percentages across all sprints, giving a high-level summary of team performance.

- if: /grandTotals
  ```ts
  import formatTableAs from "jsr:@dep/table";
  input.grandTotalsTable = new formatTableAs.Markdown()
    .add("Overall Totals Across All Sprints", "", "", "", "", "", "", "")
    .add(
      "Total Tickets", "Done", "Incomplete", "Completion %", 
      "Met Due Date", "Acceptable", "Late", "Commitment %")
    .add(
      input.grandTotals.total, 
      input.grandTotals.done, 
      input.grandTotals.incomplete, 
      `${input.grandTotals.completionPct.toFixed(1)}%`, 
      input.grandTotals.met, 
      input.grandTotals.acceptable, 
      input.grandTotals.late, 
      `${input.grandTotals.commitmentPct.toFixed(1)}%`)
    .build();
  ```

## Present Sprint Trends
Format `input.cachedSprints` into a readable table showing each sprint's name, end date, completion percentage, and commitment percentage. This allows stakeholders to quickly see how the team's performance has evolved over time and identify any trends or patterns.

```ts
input.sprintTrendsTable = new formatTableAs.Markdown()
  .add("Sprint Name", "End Date", "Completion %", "Commitment %");

input.cachedSprints.forEach(sprint => input.sprintTrendsTable.add(
    sprint.name,
    sprint.endDate || "On going or future sprint",
    `${sprint.totals.completionPct.toFixed(1)}%`,
    `${sprint.totals.commitmentPct.toFixed(1)}%`
  )
);
input.sprintTrendsTable = input.sprintTrendsTable.build();
```

## Present Developer Leaderboard

Aggregate per-developer stats across all sprints to create a leaderboard showing each developer's total tickets, completion percentage, and commitment percentage. This highlights individual contributions and can help identify top performers or those who may need support.

```ts
input.leaderboardTable = new formatTableAs.Markdown()
  .add("Assignee", "Total Tickets", "Done", "Incomplete", "Completion %", "Met Due Date", "Acceptable", "Late", "Commitment %");

[...input.compileLeaderboard.values()]
  .sort((a, b) => b.overallCompletionPct - a.overallCompletionPct)
  .forEach(dev => input.leaderboardTable.add(
    dev.assignee,
    dev.total,
    dev.done,
    dev.incomplete,
    `${(dev.done / dev.total * 100).toFixed(1)}%`,
    dev.met,
    dev.acceptable,
    dev.late,
    `${((dev.met + dev.acceptable) / dev.total * 100).toFixed(1)}%`
  )
);
input.leaderboardTable = input.leaderboardTable.build();
```

## Developer Sprint Tables

Optionally, for each developer, create a table showing their performance in each sprint (tickets completed, completion %, commitment %). This provides a more granular view of individual performance over time.

```ts
input.developerSprintTables = input.cachedSprints
  .filter(sprint => sprint.developerSummaries && sprint.developerSummaries.length > 0)
  .reduce((acc, sprint) => {
    const sprintTable = sprint.developerSummaries.reduce((tableAcc, devSummary) => {
      return tableAcc.add(
        devSummary.assignee,
        devSummary.total,
        devSummary.done,
        devSummary.incomplete,
        `${(devSummary.done / devSummary.total * 100).toFixed(1)}%`,
        devSummary.met,
        devSummary.acceptable,
        devSummary.late,
        `${((devSummary.met + devSummary.acceptable) / devSummary.total * 100).toFixed(1)}%`
      );
    },
    new formatTableAs.Markdown()
    .add("Sprint Name", sprint.name, "", "", "", "", "", "", "")
    .add("Assignee", "Total Tickets", "Done", "Incomplete", "Completion %", "Met Due Date", "Acceptable", "Late", "Commitment %"))

    return acc.set(sprint.name, sprintTable.build());
}, new Map());

```

## Format Output

Shape `input.body` with three sections:

1. **Grand Totals Table** — the overall numbers
2. **Sprint Trends Table** — one row per sprint showing completion and commitment percentages
3. **Developer Leaderboard Table** — aggregated per-developer stats across all sprints
4. **Developer Sprint Tables** — tables showing each developer's performance in each sprint

```ts
input = TableDefs.parse(input);
```
