# Report

Aggregate metrics across multiple sprints to show trends and overall developer performance. Reads all cached sprint JSON files from `~/.cache/jira-sprints-metrics.json`, optionally excluding sprints by name pattern, and produces a cross-sprint summary.

This pipeline does **not** call the Jira API — it relies entirely on locally cached, or precomputed data passed via the `input` object.

## Load Cached Metrics

Scan `~/.cache/jira-sprints-metrics.json`. If `input.exclude` is set, filter out any sprint whose name includes the exclusion string. Skip this step if `input.cachedSprints` is already populated (e.g. by a previous step or test setup) to avoid unnecessary file reads during development.

- not: /cachedSprints
  ```ts
  import sprintMetrics  from "sprintMetrics"
  import { join } from "jsr:@std/path";

  const parseBoardId = (value) => {
    const parsed = Number(value);
    return Number.isFinite(parsed) ? parsed : undefined;
  };

  input.boardId = parseBoardId(input.boardId ?? $p.get(opts, "/config/BOARD_ID"));
  input.cachePaths = {
    metrics: [
      input.boardId !== undefined ? join(".cache", "boards", `${input.boardId}`, "jira-sprints-metrics.json") : "",
      join(".cache", "jira-sprints-metrics.json"),
    ].filter(Boolean),
    totals: [
      input.boardId !== undefined ? join(".cache", "boards", `${input.boardId}`, "jira-sprints-metrics-totals.json") : "",
      join(".cache", "jira-sprints-metrics-totals.json"),
    ].filter(Boolean),
  };

  const exclude = String(input.exclude || "").trim();
  let allSprints = null;

  for (const cachePath of input.cachePaths.metrics) {
    try {
      allSprints = JSON.parse(await Deno.readTextFile(cachePath));
      input.metricsCachePath = cachePath;
      break;
    } catch {
      // try the next cache candidate
    }
  }

  if (allSprints) {
    input.cachedSprints = exclude
      ? allSprints.filter(sprint => !sprint.name.includes(exclude))
      : allSprints;
  } else {
    const results = await sprintMetrics.process(input.boardId !== undefined ? { boardId: input.boardId } : {});
    input.cachedSprints = exclude
      ? results.sprints.filter(sprint => !sprint.name.includes(exclude))
      : results.sprints;
    input.grandTotals = results.totals;
  }
  ```

## Load Cached Totals

Also read the pre-computed totals from `~/.cache/jira-sprints-metrics-totals.json` if it exists, so we can include grand totals in the final report without needing to recompute them here.

- not: /grandTotals
  ```ts
  const cachePaths = input.cachePaths?.totals || [];
  let grandTotals = null;

  for (const cachePath of cachePaths) {
    try {
      grandTotals = JSON.parse(await Deno.readTextFile(cachePath));
      input.totalsCachePath = cachePath;
      break;
    } catch {
      // try the next cache candidate
    }
  }

  input.grandTotals = grandTotals;
  ```

## Ensure Chronological Order

This gives a chronological view of how the team's performance has changed over time.

```ts
input.cachedSprints = input.cachedSprints.toSorted((a, b) => new Date(a.endDate).getTime() - new Date(b.endDate).getTime());
```

## Present Developer Leaderboard

Aggregate per-developer stats across all sprints to create a leaderboard showing each developer's total tickets, completion percentage, and commitment percentage. This highlights individual contributions and can help identify top performers or those who may need support.

```ts
input.compileLeaderboard = input.cachedSprints
  .filter(sprint => sprint.developerSummaries && sprint.developerSummaries.length > 0)
  .reduce((outerAcc, sprint) => {
    return sprint.developerSummaries.reduce((devAcc, devSummary) => {
      const existing = devAcc.get(devSummary.assignee) || {
        assignee: devSummary.assignee,
        total: 0,
        done: 0,
        incomplete: 0,
        met: 0,
        acceptable: 0,
        late: 0,
        sprintCount: 0,
      };

      existing.total += devSummary.total;
      existing.done += devSummary.done;
      existing.incomplete += devSummary.incomplete;
      existing.met += devSummary.met;
      existing.acceptable += devSummary.acceptable;
      existing.late += devSummary.late;
      existing.sprintCount += 1;
      devAcc.set(devSummary.assignee, existing);
      return devAcc;
    }, outerAcc);
  }, new Map());
```
