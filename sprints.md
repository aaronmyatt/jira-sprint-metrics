# Sprints

List sprints belonging to a given Jira board. Accepts a `boardId` and an optional `state` filter (`active`, `closed`, or `future`). Returns sprint metadata including start/end dates which are essential for the metrics pipeline downstream.

```zod
import { z } from "npm:zod";

export const schema = z.object({
  boardId: z.number().default(2662),
  state: z.enum(["active", "closed", "future"]).default("active"),
  sprints: z.array(z.object({
    id: z.number(),
    name: z.string(),
    state: z.string(),
    startDate: z.string().default(""),
    endDate: z.string().default(""),
    completeDate: z.string().optional(),
  })).default([]),
});
```

```json
{
  "inputs": [
    { "_name": "active sprints", "boardId": 2662, "state": "active" },
    { "_name": "closed sprints", "boardId": 2662, "state": "closed" }
  ],
  "MAX_RESULTS_PER_PAGE": 50,
  "BOARD_ID": 2662
}
```

## Jira API Authentication

```ts
import JiraAuth from "JiraAuth";
const { fetchWithCache } = await JiraAuth.process()
input.fetchWithCache = fetchWithCache;
```

## Fetch Sprints

Call `GET /rest/agile/1.0/board/{boardId}/sprint` with the `state` query parameter. Paginate through all results. Collect each sprint's `id`, `name`, `state`, `startDate`, `endDate`, and `completeDate` into `input.sprints`.

Ref: https://docs.atlassian.com/jira-software/REST/7.0.4/#agile/1.0/board/%7BboardId%7D/sprint

```ts
const MAX_RESULTS_PER_PAGE = $p.get(opts, "/config/MAX_RESULTS_PER_PAGE") ?? 50;
const boardId = input.boardId || $p.get(opts, "/config/BOARD_ID") || Deno.env.get("BOARD_ID");
const state = String(input.state || "active").trim().toLowerCase();

input.boardId = boardId;
input.state = state;
input.sprints = [];
let startAt = 0;
let isLastPage = false;

while (!isLastPage) {
  const page = await input.fetchWithCache(`/rest/agile/1.0/board/${boardId}/sprint`, {
    startAt,
    maxResults: MAX_RESULTS_PER_PAGE,
    state,
  }, [".cache", 'boards', `${boardId}`, 'sprints']);

  const values = page.values ?? [];
  input.sprints.push(...values);
  startAt += values.length;

  const total = page.total ?? 0;
  isLastPage = page.isLast === true || startAt >= total || values.length === 0;
}

const totalPages = Math.ceil(input.sprints.length / MAX_RESULTS_PER_PAGE) || 1;
input.fetchResults = `Fetched ${input.sprints.length} sprints across ${totalPages} page(s).`;
```

## Format Output

Shape `input.body` as a table-friendly list. Each row should show sprint ID, name, state, and date range so the user can pick which sprint to analyse.

```ts
import formatTableAs from "jsr:@dep/table";

const sprintTable = new formatTableAs.Markdown()
  .add("Sprint ID", "Name", "State", "Start", "End");

for (const sprint of input.sprints) {
  sprintTable.add(
    sprint.id,
    sprint.name,
    sprint.state,
    sprint.startDate || "",
    sprint.endDate || sprint.completeDate || "",
  );
}

input.body = [
  "# Sprints",
  `Board ID: ${input.boardId}`,
  `State: ${input.state}`,
  input.fetchResults,
  input.sprints.length > 0 ? sprintTable.build() : "No sprints were returned for this board/state.",
].join("\n\n");
```
