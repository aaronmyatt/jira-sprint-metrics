# Boards

List all Jira boards accessible to the authenticated user. This is the entry point for discovering which board contains the sprints you want to analyse.

The pipeline authenticates against the Jira Cloud REST API using credentials stored in `opts.config` (sourced from `config.json`). It paginates through all results and returns a flat list of boards with their IDs and names.

```zod
import { z } from "npm:zod";

export const schema = z.object({
  boards: z.array(z.object({
    id: z.number(),
    name: z.string(),
    type: z.string().default(""),
  })).default([]),
});
```

```json
{
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

## Fetch Boards

Call `GET /rest/agile/1.0/board` with pagination. Jira returns pages of 50 by default — keep fetching while `startAt + maxResults < total`. Collect every board's `id`, `name`, and `type` into `input.boards`.

Ref: https://docs.atlassian.com/jira-software/REST/7.0.4/#agile/1.0

```ts
const MAX_RESULTS_PER_PAGE = $p.get(opts, "/config/MAX_RESULTS_PER_PAGE") ?? 50;
input.boards = [];
let startAt = 0;
let isLastPage = false;

while (!isLastPage) {
  const page = await input.fetchWithCache(`/rest/agile/1.0/board`, {
    startAt,
    maxResults: MAX_RESULTS_PER_PAGE,
  });

  const values = page.values ?? [];
  input.boards.push(...values);
  startAt += values.length;

  const total = page.total ?? 0;
  isLastPage = page.isLast === true || startAt >= total || values.length === 0;
}
```

## Narrow to board of interest

aka `GBG Insights Portal`

```ts
input.board = input.boards.find(b => b?.id === $p.get(opts, "/config/BOARD_ID"));
input.boardId = input.board?.id || $p.get(opts, "/config/BOARD_ID");
```

## Format Output

```ts
input.boards = (input.boards ?? []).map((board) => ({
  id: board.id,
  name: board.name,
  type: board.type ?? "",
}));
```
