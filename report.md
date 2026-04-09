# Report

Aggregate metrics across multiple sprints to show trends and overall developer performance. Reads all cached sprint JSON files from `~/.cache/jira-sprints-metrics.json`, optionally excluding sprints by name pattern, and produces a cross-sprint summary.

This pipeline does **not** call the Jira API — it relies entirely on locally cached, or precomputed data passed via the `input` object.

## Tables

- if: /format/tables
  ```ts
  import reportTable from "reportTable";
  Object.assign(input, await reportTable.process());
  ```

## Graphs

- if: /format/graphs
```ts
import reportGraphs from "reportGraph";
Object.assign(input, await reportGraphs.process());
```
