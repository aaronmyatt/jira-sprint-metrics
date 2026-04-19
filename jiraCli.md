# Jira CLI

Interactive CLI entrypoint for setting up Jira auth, verifying a magic-code login, selecting a board, listing sprints, and rendering sprint reports.

```zod
const JiraAuthSchema = z.object({
  JIRA_DOMAIN: z.string({ message: "JIRA_DOMAIN must be a valid URL" }),
  JIRA_EMAIL: z.string().email({ message: "JIRA_EMAIL must be a valid email address" }),
  JIRA_TOKEN: z.string().min(1, { message: "JIRA_TOKEN cannot be empty" }),
  updatedAt: z.string().refine((value) => !isNaN(Date.parse(value)), { message: "updatedAt must be a valid date string" }),
});
```

```json
{
  "jiraAuthPath": ".cache/jira-auth.json",
  "selectedBoardPath": ".cache/selected-board.json"
}
```

## Load Config and Cache

Read Jira credentials from `input`, then `.cache/jira-auth.json`, then `Deno.env`.
```ts
import { existsSync } from "jsr:@std/fs/exists";

try {
  input.jiraAuthConfig = JSON.parse(await Deno.readTextFile($p.get(opts, "/config/jiraAuthPath")));
} catch {
  input.jiraAuthConfig = {
    JIRA_DOMAIN: $p.get(opts, '/config/JIRA_DOMAIN') || Deno.env.get("JIRA_DOMAIN") || "",
    JIRA_EMAIL: $p.get(opts, '/config/JIRA_EMAIL') || Deno.env.get("JIRA_EMAIL") || "",
    JIRA_TOKEN: $p.get(opts, '/config/JIRA_TOKEN') || Deno.env.get("JIRA_TOKEN") || "",
  };
}

try {
  input.selectedBoard = JSON.parse(await Deno.readTextFile($p.get(opts, "/config/selectedBoardPath")));
} catch {
  input.selectedBoard = null;
}

input.boardId = input.boardId || input.selectedBoard?.id || $p.get(opts, "/config/BOARD_ID") || Deno.env.get("BOARD_ID");
```

## Setup

- flags: /setup
  ```ts
  const jiraAuthPath = $p.get(opts, "/config/jiraAuthPath");
  if (existsSync(jiraAuthPath)) {
    const raw = await Deno.readTextFile(jiraAuthPath);
    const existingAuth = JSON.parse(raw);
    input.body = [
      "# Jira CLI is already set up",
      "",
      "Existing credentials found in cache:",
      `- Domain: ${existingAuth.JIRA_DOMAIN}`,
      `- Email: ${existingAuth.JIRA_EMAIL}`,
      `- Token: ${existingAuth.JIRA_TOKEN}`,
      "",
    ].join("\n");
    return;
  }

  console.log("No existing Jira credentials found. Starting setup...");
  const jiraDomain = prompt("Enter your Jira domain (e.g. your-domain.atlassian.net):") || "";
  const jiraEmail = prompt("Enter your Jira email:") || "";
  const jiraToken = prompt("Enter your Jira API token:") || "";

  const jiraAuth = JiraAuthSchema.safeParse({
      JIRA_DOMAIN: jiraDomain,
      JIRA_EMAIL: jiraEmail,
      JIRA_TOKEN: jiraToken,
      updatedAt: new Date().toISOString(),
  });

  if (!jiraAuth.success) {
    console.error("Invalid input:");
    console.error(jiraAuth.error.format());
    return;
  }

  const writeRaw = JSON.stringify(jiraAuth.data, null, 2);
  await Deno.writeTextFile(jiraAuthPath, writeRaw, { create: true });
  input.body = [
      "# Jira setup saved",
      "",
      `- Domain: ${jiraAuth.data.JIRA_DOMAIN}`,
      `- Email: ${jiraAuth.data.JIRA_EMAIL}`,
      `- Token: ${jiraAuth.data.JIRA_TOKEN}`,
      `- Cache file: ${jiraAuthPath}`,
  ].join("\n");
  ```

## Login

- if: /command/login
  ```ts
  import magicCodeAuth from "magicCodeAuth";

  const existingLogin = await input.cli.readJson(input.cli.paths.login) || {};
  const email = input.cli.promptRequired("Email", String(existingLogin.email || ""));
  const sendResult = await magicCodeAuth.process({ email, action: { send: true } });
  const sendIssues = [
    ...(sendResult.errors || []),
    ...(sendResult.error || []),
  ];

  if (sendIssues.length > 0 || !sendResult.challengeId || !sendResult.emailSent) {
    input.body = [
      "# Login failed",
      "",
      `Could not send a verification code to ${email}.`,
      ...(sendIssues.length > 0
        ? ["", "Issues:", ...sendIssues.map((issue) => `- ${issue.message || issue.reason || JSON.stringify(issue)}`)]
        : []),
    ].join("\n");
    return;
  }

  const pendingLogin = {
    email,
    challengeId: sendResult.challengeId,
    createdAt: new Date().toISOString(),
  };

  await input.cli.writeJson(input.cli.paths.pendingLogin, pendingLogin);
  const code = input.cli.promptRequired("Verification code");
  const verifyResult = await magicCodeAuth.process({
    email,
    challengeId: pendingLogin.challengeId,
    code,
    action: { verify: true },
  });

  const verifyIssue = verifyResult.error?.[0];
  if (!verifyResult.auth?.authenticated) {
    input.body = [
      "# Login not verified",
      "",
      `- Email: ${email}`,
      `- Challenge ID: ${pendingLogin.challengeId}`,
      `- Reason: ${verifyIssue?.reason || "verification-failed"}`,
      ...(verifyIssue?.attemptsLeft !== undefined ? [`- Attempts left: ${verifyIssue.attemptsLeft}`] : []),
      "",
      `Pending login remains cached in ${input.cli.paths.pendingLogin}.`,
    ].join("\n");
    return;
  }

  await input.cli.writeJson(input.cli.paths.login, {
    ...verifyResult.auth,
    updatedAt: new Date().toISOString(),
  });

  try {
    await Deno.remove(input.cli.paths.pendingLogin);
  } catch {
    // ignore if the pending file is already missing
  }

  input.auth = verifyResult.auth;
  input.body = [
    "# Login verified",
    "",
    `- Email: ${verifyResult.auth.email}`,
    `- Challenge ID: ${verifyResult.auth.challengeId}`,
    `- Authenticated at: ${verifyResult.auth.authenticatedAt}`,
    `- Session cache: ${input.cli.paths.login}`,
  ].join("\n");
  ```

## Boards

- flags: /boards
  ```ts
  import boardsTable from "reportBoards"
  
  const output = await boardsTable.process();

  console.log([
    "# Select a board",
    "",
    "Enter a board ID to select a board for subsequent commands.",
    "This will be cached for future runs until you select a different board.",
    "",
    output.body,
    "",
    `Current selected board ID: ${input.selectedBoard?.id || "none"}`,
  ].join("\n"));
  const selectedBoardId = prompt("Board ID: ");
  const selectedBoard = output.boards.find(board => String(board.id) === String(selectedBoardId));
  if (!selectedBoard) {
    input.body = `# Board not found\n\nNo board was found with ID "${selectedBoardId}".`;
    return;
  }

  const writeRaw = JSON.stringify(selectedBoard, null, 2);
  await Deno.writeTextFile($p.get(opts, "/config/selectedBoardPath"), writeRaw, { create: true });
  ```

## Sprints

- flags: /sprints
  ```ts
  import formatTableAs from "jsr:@dep/table";
  import getSprints from "sprints";

  if (!input.boardId) {
    console.error("# Sprints\n\nNo board ID is available. Run --boards first or pass --boardId.");
    return;
  }
  
  const results = await getSprints.process({ boardId: input.boardId, state: 'closed' });

  input.body = new formatTableAs.Markdown()
    .add("Sprint ID", "Name", "State", "Start", "End");

  results.sprints.forEach(sprint => input.body.add(
    sprint.id,
    sprint.name,
    sprint.state,
    sprint.startDate || "",
    sprint.endDate || sprint.completeDate || "",
  ));
  input.body = [
    "# Sprints",
    `Board ID: ${input.boardId}`,
    `State: closed`,
    results.fetchResults,
    results.sprints.length > 0 ? input.body.build() : "No sprints were returned for this board/state.",
  ].join("\n\n");
  ```

## Report

- flags: /report
  ```ts
  import getReport from "report";

  if (input.boardId === undefined) {
    input.body = "# Jira Sprint Report\n\nNo board ID is available. Run --boards first or pass --boardId.";
    return;
  }

  const results = await getReport.process({ boardId: input.boardId, format: { all: true } });
  input.body = results.body || "# Jira Sprint Report\n\nNo report sections were generated.";
  ```

## Help

- flags: /help
  ```ts
  const helpLines = [
    "# Jira CLI",
    "",
    ...(input.helpReason ? [input.helpReason, ""] : []),
    "Use one command flag at a time:",
    "",
    "- --setup Prompt for JIRA_DOMAIN, JIRA_EMAIL, and JIRA_TOKEN and save them to .cache/jira-auth.json",
    "- --login Prompt for email, send a magic code, prompt for the verification code, and cache the verified session",
    "- --boards List boards interactively and save the selected board ID to .cache/selected-board",
    "- --sprints List sprints in a table using the selected board first, then --boardId, then config",
    "- --report Render the markdown report using the selected board first, then --boardId, then config",
    "",
    "Optional flags:",
    "",
    "- --boardId 2662 Override the board only when there is no cached selected board",
    "- --state active|closed|future Filter the --sprints command",
    "- --format all|tables|graphs Choose report output sections",
    "",
    "Examples:",
    "",
    "- pd run jiraCli.md -- --setup",
    "- pd run jiraCli.md -- --login",
    "- pd run jiraCli.md -- --boards",
    "- pd run jiraCli.md -- --sprints --state closed",
    "- pd run jiraCli.md -- --report --format tables",
  ];

  input.body = helpLines.join("\n");
  ```

## Logit
- if: /body
  ```ts
  console.log(input.body);
  ```
