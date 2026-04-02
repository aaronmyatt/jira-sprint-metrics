# Sprint Metrics

The core analytics pipeline. Given a `sprintId`, fetches every issue in that sprint, retrieves each issue's changelog to extract status transitions, and computes per-ticket and per-developer metrics.

Two dimensions are measured:

1. **Completion** — did the ticket reach DONE or TEST status before the sprint ended?
2. **Commitment** — did the ticket hit its due date (with a ±1 day tolerance window)?

Results are cached locally as JSON so repeated runs skip the API calls.

```zod
import { z } from "npm:zod";

const StatusTransition = z.object({
  from: z.string(),
  to: z.string(),
  at: z.string(),
});

const TicketOutcome = z.enum(["done", "test", "incomplete"]);

const DueDateOutcome = z.enum([
  "met",
  "acceptable",
  "early",
  "late",
  "not-completed",
  "no-due-date",
]);

const IssueMetric = z.object({
  key: z.string(),
  summary: z.string(),
  assignee: z.string().default("Unassigned"),
  status: z.string(),
  finalStatus: z.string().default(""),
  dueDate: z.string().optional(),
  outcome: TicketOutcome.default("incomplete"),
  dueDateOutcome: DueDateOutcome.default("no-due-date"),
  transitions: z.array(StatusTransition).default([]),
});

const DeveloperSummary = z.object({
  assignee: z.string(),
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

export const schema = z.object({
  sprintId: z.number(),
  force: z.boolean().default(false),
  sprint: z.object({
    id: z.number(),
    name: z.string(),
    startDate: z.string(),
    endDate: z.string(),
  }).optional(),
  issues: z.array(z.any()).default([]),
  metrics: z.array(IssueMetric).default([]),
  developerSummaries: z.array(DeveloperSummary).default([]),
  totals: z.object({
    total: z.number(),
    done: z.number(),
    test: z.number(),
    incomplete: z.number(),
    completionPct: z.number(),
    met: z.number(),
    acceptable: z.number(),
    late: z.number(),
    commitmentPct: z.number(),
  }).optional(),
  cached: z.boolean().default(false),
});
```

```json
{
  "inputs": [
    { "_name": "basic", "sprintId": 1 },
    { "_name": "force refresh", "sprintId": 1, "force": true }
  ]
}
```

## Build Auth Header

Construct the Base64-encoded Basic auth header and base URL from `opts.config`, just like the other pipelines.

```ts
```

## Check Cache

Look for a cached JSON file at `~/.cache/jira-sprint/{sprintId}.json`. If it exists and `input.force` is not `true`, parse it, populate `input.metrics`, `input.developerSummaries`, `input.totals`, and `input.sprint`, set `input.cached = true`.

```ts
```

## Fetch Sprint Details

- check: /sprintId
- not: /cached

If we don't have cached data, call `GET /rest/agile/1.0/sprint/{sprintId}` to get the sprint name, start date, and end date. Store the result in `input.sprint`.

```ts
```

## Fetch Issues

- not: /cached

Call `GET /rest/agile/1.0/sprint/{sprintId}/issue` with fields `summary,status,assignee,duedate`. Paginate through all results. Store the raw issue list in `input.issues`.

```ts
```

## Fetch Changelogs

- not: /cached

For each issue in `input.issues`, call `GET /rest/api/3/issue/{issueKey}/changelog` and paginate. Extract all status-category transitions (items where `field === "status"`). Attach the transitions array to each issue object so the next step can access them.

```ts
```

## Compute Ticket Outcomes

- not: /cached

For each issue, determine its `finalStatus` — the last status it held before `input.sprint.endDate`. Then classify:

- **done** — final status category is "Done"
- **test** — final status is "In QA", "In Review", "Test", or similar test-phase statuses
- **incomplete** — anything else

Also compute `dueDateOutcome` by comparing the date the ticket first reached TEST or DONE against its `dueDate`:

- **met** — reached target status on the due date
- **acceptable** — within ±1 calendar day
- **early** — before the due date
- **late** — after the due date
- **not-completed** — never reached TEST/DONE during the sprint
- **no-due-date** — issue had no due date set

Build the full `IssueMetric` object for each issue and store in `input.metrics`.

```ts
```

## Summarise By Developer

- not: /cached

Group `input.metrics` by `assignee`. For each developer compute:

- `total`, `done`, `test`, `incomplete` counts
- `completionPct` = `(done + test) / total * 100`
- `met`, `acceptable`, `late` counts (excluding `no-due-date` and `not-completed`)
- `commitmentPct` = `(met + acceptable) / (met + acceptable + late) * 100` (0 if denominator is 0)

Store the results in `input.developerSummaries`.

```ts
```

## Compute Sprint Totals

- not: /cached

Aggregate the same counts across all issues (not grouped by developer) and store in `input.totals`. This gives the sprint-level headline numbers.

```ts
```

## Save Cache

- not: /cached

Write the computed data (`sprint`, `metrics`, `developerSummaries`, `totals`) as pretty-printed JSON to `~/.cache/jira-sprint/{sprintId}.json`. Create the directory if it doesn't exist.

```ts
```

## Format Output

Shape `input.body` with the sprint name, totals, and per-developer breakdown so the CLI output gives a complete single-sprint report.

```ts
```
