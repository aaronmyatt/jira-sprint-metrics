# Report

Aggregate metrics across multiple sprints to show trends and overall developer performance. Reads all cached sprint JSON files from `~/.cache/jira-sprints-metrics.json`, optionally excluding sprints by name pattern, and produces a cross-sprint summary.

This pipeline does **not** call the Jira API — it relies entirely on locally cached, or precomputed data passed via the `input` object.

## Load Metrics

Read the cached metrics from `["boards", `${boardId}`, "metrics|totals.json"]`

- if: /storage
  ```ts
  const boardId = input.boardId ?? $p.get(opts, "/config/BOARD_ID");

  input.sprintKeyParts = ["boards", `${boardId}`, "sprints.json"];
  input.totalsKeyParts = ["boards", `${boardId}`, "totals.json"];

  const sprints = await input.storage.process({
    keyParts: input.sprintKeyParts,
    action: { get: true },
  });

  const totals = await input.storage.process({
    keyParts: input.totalsKeyParts,
    action: { get: true },
  });

  if(sprints.value && Array.isArray(sprints.value)) {
    input.sprints = sprints.value
    input.totals = totals.value
    console.log('Loaded cached metrics');
  }
  ```

## Fetch Metrics

Each call to the Jira API is being cached so generating the sprint report from
scratch each time should be fairly fast and cheap.

- not: /sprints
  ```ts
  import sprintMetrics  from "sprintMetrics"
  console.log("# Fetching sprint metrics from Jira API...");
  Object.assign(input, await sprintMetrics.process(input));
  ```

## Cache Metrics

Write the computed data (`sprint`, `metrics`, `developerSummaries`, `totals`) as pretty-printed JSON to `~/.cache/jira-sprints-metrics.json`. Create the directory if it doesn't exist.

- if: /storage
  ```ts
  import { join } from "jsr:@std/path";

  input.storage.process({
    keyParts: input.sprintKeyParts,
    action: { set: true },
    value: input.sprints,
  });

  input.storage.process({
    keyParts: input.totalsKeyParts,
    action: { set: true },
    value: input.totals,
  });
  ```

## Present Developer Leaderboard

Aggregate per-developer stats across all sprints to create a leaderboard showing each developer's total tickets, completion percentage, and commitment percentage. This highlights individual contributions and can help identify top performers or those who may need support.

```ts
input.compileLeaderboard = input.sprints
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
