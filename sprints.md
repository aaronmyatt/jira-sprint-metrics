# Sprints

List sprints belonging to a given Jira board. Accepts a `boardId` and an optional `state` filter (`active`, `closed`, or `future`). Returns sprint metadata including start/end dates which are essential for the metrics pipeline downstream.

```zod
import { z } from "npm:zod";

export const schema = z.object({
  boardId: z.number(),
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
    { "_name": "active sprints", "boardId": 1, "state": "active" },
    { "_name": "closed sprints", "boardId": 1, "state": "closed" }
  ]
}
```

## Build Auth Header

Construct the Base64-encoded Basic auth header and base URL from `opts.config`. Identical pattern to the Boards pipeline — reuse the same credential shape from `config.json`.

```ts
```

## Fetch Sprints

Call `GET /rest/agile/1.0/board/{boardId}/sprint` with the `state` query parameter. Paginate through all results. Collect each sprint's `id`, `name`, `state`, `startDate`, `endDate`, and `completeDate` into `input.sprints`.

```ts
```

## Format Output

Shape `input.body` as a table-friendly list. Each row should show sprint ID, name, state, and date range so the user can pick which sprint to analyse.

```ts
```
