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
  "inputs": [
    { "_name": "basic" }
  ],
  "MAX_RESULTS_PER_PAGE": 50,
  "BOARD_ID": 2662
}
```

## Build Auth Header

Construct the Base64-encoded Basic auth header and the base URL from the Jira credentials in `Deno.env`. Store them on `input` so every subsequent fetch step can reuse them without re-reading config.

Deno.env.get("JIRA_DOMAIN")
Deno.env.get("JIRA_EMAIL")
Deno.env.get("JIRA_TOKEN")

```ts
input.baseUrl = `https://${Deno.env.get("JIRA_DOMAIN")}`;
const authString = `${Deno.env.get("JIRA_EMAIL")}:${Deno.env.get("JIRA_TOKEN")}`;
input.authHeader = `Basic ${btoa(authString)}`;
```

## Fetch or Cache Helper

Wrap fetch calls in a helper function that checks for cached responses in `~/.cache/jira-boards/` first. Cache each response as `{endpoint}_{params}_{date}.json` so we can skip API calls during development and testing.

```ts
async function fetchWithCache(endpoint, params = {}) {
  const endpointKey = endpoint.replaceAll("/", "_").replace(/^_+|_+$/g, "");
  const paramsKey = Object.entries(params).map(([k,v]) => `${k}-${v}`).join("_");
  const dayKey = new Date().toISOString().split("T")[0]; // daily cache invalidation
  const cacheKey = `${endpointKey}${paramsKey ? `_${paramsKey}` : ""}_${dayKey}.json`;
  const cachePath = `.cache/jira-boards/${cacheKey}`;
  const cacheDir = `.cache/jira-boards`;

  try {
    const cached = await Deno.readTextFile(cachePath);
    return JSON.parse(cached);
  } catch {
    const url = new URL(`${input.baseUrl}${endpoint}`);
    Object.entries(params).forEach(([k, v]) => url.searchParams.append(k, v));
    
    const response = await fetch(url.toString(), {
      headers: {
        "Authorization": input.authHeader,
        "Accept": "application/json",
      },
    });
    
    const data = await response.json();
    await Deno.mkdir(cacheDir, { recursive: true });
    await Deno.writeTextFile(cachePath, JSON.stringify(data));
    return data;
  }
}

input.fetchWithCache = fetchWithCache;
```

## Fetch Boards

Call `GET /rest/agile/1.0/board` with pagination. Jira returns pages of 50 by default — keep fetching while `startAt + maxResults < total`. Collect every board's `id`, `name`, and `type` into `input.boards`.

```ts mock
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

const totalPages = Math.ceil(input.boards.length / MAX_RESULTS_PER_PAGE) || 1;
input.fetchResults = `Fetched ${input.boards.length} boards across ${totalPages} page(s).`;
```

## Clear Old Cache
Optionally, implement a step to clear out old cache files in `~/.cache/jira-boards/` that are more than 7 days old to prevent stale data from accumulating.

```ts
const cacheDir = `.cache/jira-boards`;
try {
  for await (const entry of Deno.readDir(cacheDir)) {
    if (entry.isFile && entry.name.endsWith(".json")) {
      const filePath = `${cacheDir}/${entry.name}`;
      const stats = await Deno.stat(filePath);
      const ageInDays = (Date.now() - stats.mtime.getTime()) / (1000 * 60 * 60 * 24);
      if (ageInDays > 7) {
        await Deno.remove(filePath);
      }
    }
  }
} catch {
// ignore if cache directory doesn't exist or any file errors
}
```

## Narrow to board of interest

aka `GBG Insights Portal`

```ts
input.board = input.boards.find(b => b?.id === $p.get(opts, "/config/BOARD_ID"));
input.boardId = input.board?.id || $p.get(opts, "/config/BOARD_ID");
```

## Format Output

Shape `input.body` as a simple table-friendly array so the CLI output is scannable. Each row should show the board ID and name.

```ts
input.boards = (input.boards ?? []).map((board) => ({
  id: board.id,
  name: board.name,
  type: board.type ?? "",
}));
```
