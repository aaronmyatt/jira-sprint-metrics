# Report

Aggregate metrics across multiple sprints to show trends and overall developer performance. Reads all cached sprint JSON files from `~/.cache/jira-sprints-metrics.json`, optionally excluding sprints by name pattern, and produces a cross-sprint summary.

This pipeline does **not** call the Jira API — it relies entirely on locally cached, or precomputed data passed via the `input` object.

## Tables

- if: /format/tables
- if: /format/all
  ```ts
  import reportTable from "reportTable";
  Object.assign(input, await reportTable.process());
  ```

## Graphs

- if: /format/graphs
- if: /format/all
  ```ts
  import reportGraphs from "reportChartli";
  Object.assign(input, await reportGraphs.process());
  ```

## Assemble Body

```ts
const sections = [];

const pushSection = (title, content) => {
  if (content) {
    sections.push(`## ${title}\n\n${content}`);
  }
};

const pushNamedSections = (title, blocks) => {
  const entries = blocks instanceof Map
    ? [...blocks.entries()]
    : Object.entries(blocks || {});

  if (!entries.length) return;

  sections.push([
    `## ${title}`,
    ...entries.map(([name, content]) => `### ${name}\n\n${content}`),
  ].join("\n\n"));
};

pushSection("Grand Totals Table", input.grandTotalsTable);
pushSection("Sprint Trends Table", input.sprintTrendsTable);
pushSection("Developer Leaderboard Table", input.leaderboardTable);
pushNamedSections("Developer Sprint Tables", input.developerSprintTables);
pushSection("Grand Totals Graph", input.grandTotalsGraph);
pushSection("Sprint Trends Graph", input.sprintTrendsGraph);
pushSection("Developer Leaderboard Graph", input.leaderboardGraph);
pushNamedSections("Developer Sprint Graphs", input.developerSprintGraphs);
pushSection("Developer Trends Graph", input.developerTrendsGraph);

const header = [
  "# Jira Sprint Report",
  input.boardId !== undefined ? `Board ID: ${input.boardId}` : "",
  input.exclude ? `Excluded sprint filter: ${input.exclude}` : "",
].filter(Boolean).join("\n\n");

input.body = sections.length > 0
  ? [header, ...sections].filter(Boolean).join("\n\n")
  : [header || "# Jira Sprint Report", "No report sections were generated."].join("\n\n");
```
