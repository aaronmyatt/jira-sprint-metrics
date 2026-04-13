# Sprint Metrics

The core analytics pipeline. Given a `sprintId`, fetches every issue in that sprint, retrieves each issue's changelog to extract status transitions, and computes per-ticket and per-developer metrics.

Two dimensions are measured:

1. **Completion** — did the ticket reach DONE status before the sprint ended?
2. **Commitment** — did the ticket hit its due date (with a ±1 day tolerance window)?

Results are cached locally as JSON so repeated runs skip the API calls.

```zod
const IssueMetric = z.object({
  key: z.string(),
  summary: z.string(),
  assignee: z.string(),
  status: z.string(),
  finalStatus: z.string(),
  dueDate: z.string().optional(),
  outcome: z.enum(["done", "incomplete"]),
  dueDateOutcome: z.enum(["met", "acceptable", "early", "late", "not-completed", "no-due-date"]),
  transitions: z.array(z.object({
    from: z.string(),
    to: z.string(),
    at: z.string(),
  })),
  sprintId: z.number(),
});

const DeveloperSummary = z.object({
  assignee: z.string(),
  total: z.number(),
  done: z.number(),
  incomplete: z.number(),
  completionPct: z.number(),
  met: z.number(),
  acceptable: z.number(),
  late: z.number(),
  commitmentPct: z.number(),
});

const JiraIssuesWithChangelogs = z.object({
  key: z.string(),
  fields: z.object({
    summary: z.string(),
    assignee: z.object({ displayName: z.string().default("Unassigned") }).nullable(),
    status: z.object({
      name: z.enum(["Ready", "Created", "To Do", "In Progress", "Done", "Closed", "Review", "Test", "Blocked / On Hold"]),
    }),
    dueDate: z.string().optional(),
  }),
  transitions: z.array(z.object({
    from: z.string(),
    to: z.string(),
    at: z.string(),
  })) 
});

// output
const sprintCheck = z.object({
  sprints: z.array(z.object({
    id: z.number(),
    name: z.string(),
    state: z.string(),
    originBoardId: z.number(),
    startDate: z.string().optional(), // some sprints may not have started yet
    endDate: z.string().optional(), // future sprints may not have an end date
    metrics: z.array(IssueMetric),
    developerSummaries: z.array(DeveloperSummary),
    issues: z.array(JiraIssuesWithChangelogs),
    totals: z.object({
      total: z.number(),
      done: z.number(),
      incomplete: z.number(),
      completionPct: z.number(),
      met: z.number(),
      acceptable: z.number(),
      late: z.number(),
      commitmentPct: z.number(),
    }),
  })),
  totals: z.object({
    total: z.number(),
    done: z.number(),
    incomplete: z.number(),
    completionPct: z.number(),
    met: z.number(),
    acceptable: z.number(),
    late: z.number(),
    commitmentPct: z.number(),
    avgCompletionTime: z.number().optional(),
  }),
})
```

```json
{
  "BOARD_ID": 2662,
  "EXCLUDE_SPRINT": [13538]
}
```

## Jira API Authentication

```ts
import JiraAuth from "JiraAuth";
const { fetchWithCache } = await JiraAuth.process()
input.fetchWithCache = fetchWithCache;
```

## Get Sprints For Board

```ts
import getSprints from "sprints";
const boardId = input.boardId ?? $p.get(opts, "/config/BOARD_ID");
const { sprints } = await getSprints.process({ boardId, state: "closed" });
input.boardId = boardId;
input.sprints = sprints.filter(s => !$p.get(opts, "/config/EXCLUDE_SPRINT", []).includes(s.id));
```

## Fetch Issues

Call `GET /rest/agile/1.0/sprint/{sprintId}/issue` with fields `summary,status,assignee,duedate`. Paginate through all results. Store the raw issue list in `input.issues`.

```ts
for (const sprint of input.sprints) {
  sprint.issues = sprint.issues || [];
  const MAX_RESULTS_PER_PAGE = $p.get(opts, "/config/MAX_RESULTS_PER_PAGE") ?? 50;
  let startAt = 0;
  let isLastPage = false;

  while (!isLastPage) {
    const page = await input.fetchWithCache(`/rest/agile/1.0/sprint/${sprint.id}/issue`, {
      startAt,
      maxResults: MAX_RESULTS_PER_PAGE,
      fields: "summary,status,assignee,duedate",
    }, [".cache", 'sprint', `${sprint.id}`, 'issues']);

    const values = page.issues ?? [];
    sprint.issues.push(...values);
    startAt += values.length;

    const total = page.total ?? 0;
    isLastPage = page.isLast === true || startAt >= total || values.length === 0;
  }
}
```

## Fetch Changelogs

For each issue in `input.sprints[].issues`, call `GET /rest/api/3/issue/{issueKey}/changelog` and paginate. Extract all status-category transitions (items where `field === "status"`). Attach the transitions array to each issue object so the next step can access them.

Ref: https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-issues/#api-rest-api-3-issue-issueidorkey-changelog-list-post

```ts
for (const sprint of input.sprints) {
  for (const issue of sprint.issues) {
    const transitions = [];
    const MAX_RESULTS_PER_PAGE = $p.get(opts, "/config/MAX_RESULTS_PER_PAGE") ?? 50;
    let startAt = 0;
    let isLastPage = false;

    while (!isLastPage) {
      const page = await input.fetchWithCache(`/rest/api/3/issue/${issue.key}/changelog`, {
        startAt,
        maxResults: MAX_RESULTS_PER_PAGE,
      }, [".cache", "sprint", `${sprint.id}`, "issues", "changelog", `${issue.key}`]);

      const values = page.values ?? [];
      for (const history of values) {
        for (const item of history.items) {
          if (item.field === "status") {
            transitions.push({
              from: item.fromString,
              to: item.toString,
              at: history.created,
            });
          }
        }
      }

      startAt += values.length;
      const total = page.total ?? 0;
      isLastPage = page.isLast === true || startAt >= total || values.length === 0;
    }

    issue.transitions = transitions;
  }
}
```

## Compute Ticket Outcomes

For each issue, determine its `finalStatus` — the last status it held before `input.sprints[].endDate`.
For the purposes of these sprint metric, a developer is consider DONE once they stop actively working on the ticket.

Therefore, we classify:

- **done** — final status category is "Done", "Review", "Test", or "Closed" statuses
- **incomplete** — anything else (e.g. still in "In Progress" or "To Do" at sprint end)

Also compute `dueDateOutcome` by comparing the date the ticket first reached DONE against its `dueDate`:

- **met** — reached target status on the due date
- **acceptable** — within ±1 calendar day
- **early** — before the due date
- **late** — after the due date
- **not-completed** — never reached DONE during the sprint
- **no-due-date** — issue had no due date set

Build the full `IssueMetric` object for each issue and store in `input.metrics`.

TODO: extract this to it's own pipe

```ts

for (const sprint of input.sprints) {
  sprint.metrics = [];
  for (const issue of sprint.issues) { 
    const metric = {
      key: issue.key,
      summary: issue.fields.summary,
      assignee: issue.fields.assignee?.displayName || "Unassigned",
      status: issue.fields.status.name,
      finalStatus: "",
      dueDate: issue.fields.duedate || "",
      outcome: "incomplete",
      dueDateOutcome: "no-due-date",
      transitions: issue.transitions || [],
      sprintId: sprint.id,
    };

    // Categories that count as "done" for sprint completion purposes.
    // An issue is considered complete when it *first* enters one of these statuses
    // before the sprint ends — later transitions (e.g. reopened then re-closed)
    // don't change the completion timestamp.
    const doneCategories = ["Done", "Closed", "Review", "Test"];
    const sprintEnd = new Date(sprint.endDate);

    // Find the first transition into a done category that occurred before
    // or on the sprint end date. This is the moment the issue was "completed"
    // for sprint-metric purposes.
    const firstDoneTransition = metric.transitions.find(
      (t) => new Date(t.at) <= sprintEnd && doneCategories.includes(t.to)
    );

    // finalStatus reflects the first done status reached, or falls back to
    // the last status seen before sprint end (to capture in-progress work).
    if (firstDoneTransition) {
      metric.finalStatus = firstDoneTransition.to;
      metric.outcome = "done";
    } else {
      // No done transition found — walk transitions up to sprint end to get
      // the most recent status (e.g. "In Progress", "Blocked").
      let lastStatus = metric.status;
      for (const t of metric.transitions) {
        if (new Date(t.at) > sprintEnd) break;
        lastStatus = t.to;
      }
      metric.finalStatus = lastStatus;
      metric.outcome = "incomplete";
    }

    // Classify due date outcome based on when the issue *first* reached a
    // done category relative to its due date. What matters for developer
    // performance is whether that first completion landed within ±1 day of
    // the due date — that's "acceptable" delivery.
    if (metric.dueDate && firstDoneTransition) {
      const dueDate = new Date(metric.dueDate);
      const completionDate = new Date(firstDoneTransition.at);

      // diffDays is positive when completed after the due date (late),
      // negative when completed before it (early), zero when same day.
      const diffDays = Math.round(
        (completionDate - dueDate) / (1000 * 60 * 60 * 24)
      );

      if (diffDays === 0) {
        metric.dueDateOutcome = "met";
      }
      else if (Math.abs(diffDays) == 1) {
        // First done transition fell within ±1 day of the due date —
        // this is the target window for good developer performance.
        metric.dueDateOutcome = "acceptable";
      } else if (diffDays < -1) {
        metric.dueDateOutcome = "early";
      } else {
        // diffDays > 1 — completed more than a day after due date
        metric.dueDateOutcome = "late";
      }
    } else if (metric.dueDate && !firstDoneTransition) {
      // Has a due date but was never completed within the sprint.
      metric.dueDateOutcome = "not-completed";
    }

    sprint.metrics.push(metric);
  }
}
```

## Summarise By Developer

Group `input.sprints[].metrics` by `assignee`. For each developer compute:

- `total`, `done`, `acceptable`, `incomplete` counts
- `completionPct` = `done / total * 100`
- `met`, `acceptable`, `late` counts (excluding `no-due-date` and `not-completed`)
- `commitmentPct` = `(met + acceptable) / (met + acceptable + late) * 100` (0 if denominator is 0)

Store the results in `input.developerSummaries`.

```ts
for (const sprint of input.sprints) {
  sprint.developerSummaries = [];
  const metricsByAssignee = Map.groupBy(sprint.metrics || [], m => m.assignee);

  for (const [assignee, metrics] of metricsByAssignee) {
    const total = metrics.length;
    const done = metrics.filter(m => m.outcome === "done").length;
    const incomplete = metrics.filter(m => m.outcome === "incomplete").length;
    const completionPct = total > 0 ? (done / total) * 100 : 0;

    const met = metrics.filter(m => ["met"].includes(m.dueDateOutcome)).length;
    const acceptable = metrics.filter(m => ["acceptable"].includes(m.dueDateOutcome)).length;
    const late = metrics.filter(m => ["late"].includes(m.dueDateOutcome)).length;
    const commitmentDenom = met + acceptable + late;
    const commitmentPct = commitmentDenom > 0 ? ((met + acceptable) / commitmentDenom) * 100 : 0;

    sprint.developerSummaries.push({
      assignee,
      total,
      done,
      incomplete,
      completionPct,
      met,
      acceptable,
      late,
      commitmentPct,
    });
  }
}
```

## Compute Sprint Totals

  Aggregate the same counts across all issues (not grouped by developer `input.sprints[].metrics`) and store in `input.sprints[].totals`. This gives the sprint-level headline numbers.

```ts
for (const sprint of input.sprints) {
  const metrics = sprint.metrics || [];
  const total = metrics.length;
  const done = metrics.filter(m => m.outcome === "done").length;
  const incomplete = metrics.filter(m => m.outcome === "incomplete").length;
  const completionPct = total > 0 ? (done / total) * 100 : 0;

  const met = metrics.filter(m => ["met"].includes(m.dueDateOutcome)).length;
  const acceptable = metrics.filter(m => ["acceptable"].includes(m.dueDateOutcome)).length;
  const late = metrics.filter(m => ["late"].includes(m.dueDateOutcome)).length;
  const commitmentDenom = met + acceptable + late;
  const commitmentPct = commitmentDenom > 0 ? ((met + acceptable) / commitmentDenom) * 100 : 0;

  sprint.totals = {
    total,
    done,
    incomplete,
    completionPct,
    met,
    acceptable,
    late,
    commitmentPct,
  };
}
```

## Throughput Metrics

Calculate throughput metrics like average completion time (from first transition to done) and average time in each status for all completed tickets. This can help identify bottlenecks in the workflow.

```ts
import { format as formatDuration } from "jsr:@std/fmt/duration";

for (const sprint of input.sprints) {
  const completedMetrics = (sprint.metrics || []).filter(m => m.outcome === "done");
  const totalCompletionTime = completedMetrics.reduce((sum, m) => {
    const doneTransition = m.transitions.find(t => t.to === m.finalStatus);
    const firstTransition = m.transitions[0];
    if (doneTransition && firstTransition) {
      const completionTime = new Date(doneTransition.at).getTime() - new Date(firstTransition.at).getTime();
      return sum + completionTime;
    }
    return sum;
  }, 0);
  const avgCompletionTime = completedMetrics.length > 0 ? totalCompletionTime / completedMetrics.length : 0;
  sprint.totals.avgCompletionTime = formatDuration(avgCompletionTime, { ignoreZero: true});
}
```

## Compute Overall Totals

Aggregate totals across all sprints into `input.totals` for an overall summary.

```ts
input.totals = {
  total: input.sprints.reduce((sum, s) => sum + (s.totals?.total || 0), 0),
  done: input.sprints.reduce((sum, s) => sum + (s.totals?.done || 0), 0),
  incomplete: input.sprints.reduce((sum, s) => sum + (s.totals?.incomplete || 0), 0),
  completionPct: 0, // will compute after met/acceptable/late
  met: input.sprints.reduce((sum, s) => sum + (s.totals?.met || 0), 0),
  acceptable: input.sprints.reduce((sum, s) => sum + (s.totals?.acceptable || 0), 0),
  late: input.sprints.reduce((sum, s) => sum + (s.totals?.late || 0), 0),
};
const commitmentDenom = input.totals.met + input.totals.acceptable + input.totals.late;
input.totals.commitmentPct = commitmentDenom > 0 ? ((input.totals.met + input.totals.acceptable) / commitmentDenom) * 100 : 0;
input.totals.completionPct = input.totals.total > 0 ? (input.totals.done / input.totals.total) * 100 : 0;
```

## Verify Sprint Data

```ts
sprintCheck.parse(input);
```

## Save Cache

Write the computed data (`sprint`, `metrics`, `developerSummaries`, `totals`) as pretty-printed JSON to `~/.cache/jira-sprints-metrics.json`. Create the directory if it doesn't exist.

```ts
import { join } from "jsr:@std/path";

const boardId = input.boardId ?? $p.get(opts, "/config/BOARD_ID");
const cacheDir = join(".cache", "boards", `${boardId}`);
const cachePath = join(cacheDir, "jira-sprints-metrics.json");
await Deno.mkdir(cacheDir, { recursive: true });
await Deno.writeTextFile(cachePath, JSON.stringify(input.sprints, null, 2));
```

## Save aggregate totals

```ts
const boardId = input.boardId ?? $p.get(opts, "/config/BOARD_ID");
const cacheDir = join(".cache", "boards", `${boardId}`);
const totalsCachePath = join(cacheDir, "jira-sprints-metrics-totals.json");
await Deno.mkdir(cacheDir, { recursive: true });
await Deno.writeTextFile(totalsCachePath, JSON.stringify(input.totals, null, 2));
```
