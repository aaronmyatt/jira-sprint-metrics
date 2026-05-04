# Plan: Magic-Code-Auth Jira Sprint Report Web App on Deno Deploy

This plan is to be saved (at implementation time) as `TODO.md` at the repo root of `/Users/aaronmyatt/pipes/jira-sprint-metrics/` so it can serve as a running checklist while the work is done.

---

## Context

`jira-sprint-metrics` is today a Pipedown CLI with a rich report pipeline
(`report.md` → `reportTable`/`reportChartli` → markdown output). Login uses a
filesystem-backed magic-code flow (`magicCodeAuth.md` + `jiraCli.md --login`).

Goal: stand up a **very basic, single-purpose website**, deployed on
[Deno Deploy](https://docs.deno.com/deploy/manual/), that lets an authenticated
user:

1. Log in by email + magic code.
2. Pick one of the boards returned by `reportBoards.md`.
3. View the rendered sprint report produced by `report.md` for that board.

Nothing else — no settings, no history, no admin. The website is a thin Mithril
SPA that delegates all business logic to the existing Pipedown pipes.

Two constraints shape the work:

- **Deno Deploy has no writable filesystem.** Session + challenge state must
  move to [Deno KV](https://docs.deno.com/deploy/kv/manual/) (`Deno.openKv()`),
  which is zero-config on Deploy.
- **Existing CLI must keep working.** The CLI continues to use filesystem
  caching exactly as today.

To satisfy both without forking the auth pipe, **storage is extracted as a
Pipedown pipe** that `magicCodeAuth` (and `session`) consume through
**dependency injection**: the caller imports the desired backend and passes it
on `input.storage`. The auth pipe calls `input.storage.process({action, key, ...})`
and is agnostic to whether it's hitting the filesystem or Deno KV.

Frontend patterns are lifted from [`$HOME/pipes/core/pipedown/pdCli/frontend`](/Users/aaronmyatt/pipes/core/pipedown/pdCli/frontend):
namespaced globals (`WEB.state` / `WEB.actions` / `WEB.components`), Mithril
(`m.mount`), shared theme tokens (`shared/base.css`), and fetch wrappers.

---

## Architecture Overview

```
Browser (SPA in /web)
  │  fetch JSON
  ▼
webApp.md  (pd serve, entrypoint .pd/webApp/server.ts)
  ├─ static: "./web"                        ← SPA + CSS + JS
  ├─ imports storageKv  → passes as input.storage to magicCodeAuth + session
  ├─ POST /api/auth/send             → magicCodeAuth  (action.send)
  ├─ POST /api/auth/verify           → magicCodeAuth  (action.verify) + session.create
  ├─ POST /api/auth/logout           → session.destroy
  ├─ GET  /api/me                    → session.read
  ├─ GET  /api/boards     (auth)     → reportBoards
  └─ POST /api/report     (auth)     → report (boardId from body)

jiraCli.md  (existing --login step)
  └─ imports storageFs → passes as input.storage to magicCodeAuth

magicCodeAuth.md  (rewritten; storage-agnostic)
  └─ input.storage.process({ action: {get|set|delete}, key, value, ttlMs })

session.md  (new helper; storage-agnostic)
  └─ same storage contract as magicCodeAuth

storageFs.md  (new; filesystem backend)
storageKv.md  (new; Deno KV backend)
```

Pipes consumed by `webApp.md` use Pipedown's sub-pipe import pattern already
demonstrated in [`jiraCli.md:112`](jiraCli.md#L112) —
`import x from "pipeName"; await x.process({...})`. Imported pipes can be
placed on `input` and passed along to downstream pipes because `input.storage`
is just `{ process: fn }` — the same JS object Pipedown hands you on import.

The `pd serve` template already handles static serving, JSON body parsing,
HTML content-type inference, and `input.response` escape-hatch; see
[`templates/server.ts:88`](templates/server.ts#L88) and
[`templates/server.ts:131`](templates/server.ts#L131). No template changes needed.

---

## Storage Contract (the DI seam)

Every storage pipe (`storageFs`, `storageKv`, future alternatives) must accept
the following `input` and mutate it identically:

**Input shape**

| Field | Type | Purpose |
|-------|------|---------|
| `action` | `{ get: true } \| { set: true } \| { delete: true }` | Which operation to run. |
| `key` | `string[]` | Hierarchical key, e.g. `["magic-code", challengeId]` — matches Deno KV's array-key convention so `storageKv` can pass it through untouched. `storageFs` flattens it to `join(root, ...key) + ".json"`. |
| `value` | `unknown` | Required for `set`. Serialised as JSON by both backends. |
| `ttlMs` | `number?` | Optional expiry. `storageKv` passes it to `kv.set(key, value, { expireIn: ttlMs })`. `storageFs` writes `{ value, expiresAt }` and filters expired on `get`. |
| `storage` | `{ root?: string }` | Backend config. FS uses `root` (default `.cache/pd-storage`). KV uses `storage.kvPath` optionally. |

**Output shape**

| Field | Type | Purpose |
|-------|------|---------|
| `result` | `{ value: unknown \| null, versionstamp?: string }` | Returned by `get`. `null` when missing/expired (mirrors `kv.get`'s `{value: null, versionstamp: null}` semantics — ref [KV operations](https://docs.deno.com/deploy/kv/manual/operations)). |

Rationale for the shape: aligning on Deno KV's native conventions (array keys,
`{value, versionstamp}` results, `expireIn` for TTL) means `storageKv` is a
thin pass-through and `storageFs` does the adaptation work.

---

## TODO (save this as `TODO.md` at implementation time)

### Phase 0 — Prep

- [x] Confirm Deno version ≥ 1.38 (KV GA). Run `deno --version`.
- [ ] Create a Deno Deploy project (empty) — record its name and slug.
- [ ] Generate strong secrets for `MAGIC_CODE_PEPPER` and `SESSION_SECRET`
      (each 32+ random bytes, hex-encoded).

### Phase 1 — Storage pipes (the DI backends)

#### `storageFs.md` (new)

- [x] `json` config: `{ "root": ".cache/pd-storage" }` — base directory for all
      files, callers can override by placing `storage.root` on `input`.
- [x] Step `Resolve Path`: flatten `input.key` → `path.join(root, ...key) + ".json"`;
      ensure parent directory exists. Uses
      [`jsr:@std/path/join`](https://jsr.io/@std/path/doc/~/join) and
      [`Deno.mkdir`](https://docs.deno.com/api/deno/~/Deno.mkdir).
- [x] Step `Get` — `if: /action/get`:
  - [x] Read file, JSON-parse `{ value, expiresAt }`.
  - [x] If `expiresAt` is in the past, delete the file and return `null`.
  - [x] `input.result = { value, versionstamp: stats.mtime.toISOString() }`.
- [x] Step `Set` — `if: /action/set`:
  - [x] Compute `expiresAt = ttlMs ? Date.now() + ttlMs : null`.
  - [x] Atomic write via temp file + rename so concurrent readers don't see
        half-written JSON. Ref:
        [`Deno.rename`](https://docs.deno.com/api/deno/~/Deno.rename).
- [x] Step `Delete` — `if: /action/delete`:
  - [x] `Deno.remove(path)`; swallow "not found" errors.
- [x] Step `Sweep` (optional, runs on every `set`): walk `root` and remove files
      past their `expiresAt`. Cheap because the tree is small; replaces the old
      `Clear Old Challenge Cache` step in `magicCodeAuth`. Uses
      [`jsr:@std/fs/walk`](https://jsr.io/@std/fs/doc/~/walk).
- [x] Add a `dry-run set + get` test input so `pd test` exercises it.

#### `storageKv.md` (new)

- [x] `json` config: `{ "kvPath": null }` — `null` ⇒ hosted KV on Deploy,
      explicit path ⇒ local SQLite file when needed.
- [x] Step `Open`: `input._kv = globalThis.__sharedKv ??= await Deno.openKv(kvPath)`
      so repeated calls in the same process reuse the same connection (open is
      cheap but caching avoids churn).
- [x] Step `Get` — `if: /action/get`:
  - [x] `input.result = await kv.get(input.key)` — already `{value, versionstamp}`.
- [x] Step `Set` — `if: /action/set`:
  - [x] `await kv.set(input.key, input.value, input.ttlMs ? { expireIn: input.ttlMs } : undefined)`.
- [x] Step `Delete` — `if: /action/delete`:
  - [x] `await kv.delete(input.key)`.
- [ ] Test input: a round-trip `set` with `ttlMs: 50`, wait 100 ms, confirm `get` returns `null`.

### Phase 2 — Rewrite `magicCodeAuth.md` around `input.storage`

Reference: current file at [`magicCodeAuth.md`](magicCodeAuth.md). Keep the same
public contract on `input` (`action.send|verify`, `email`, `code`, `challengeId`,
`auth`, `error`) so `jiraCli.md` needs only a one-line change.

- [x] Top of the pipe: if `!input.storage?.process`, throw with a clear message
      — the caller must inject a storage backend.
- [x]  Remove `cacheDir` / `challengePath` / `clear-old-challenge-cache` logic —
      TTL and sweeping now belong to the storage pipe.
- [x] `Send Code` step:
  - [x] Build challenge object (same shape as today: `challengeId`, `email`,
        `codeHash`, `expiresAt`, `attempts`, `maxAttempts`, `createdAt`, `consumedAt`).
  - [x] `await input.storage.process({ action: { set: true }, key: ["magic-code", challengeId], value: challenge, ttlMs })`.
  - [x] Expose `challengeId` + `challengeExpiresAt` on `input` exactly as today.
  - [x] Keep the `dryRun` short-circuit and `sendEmail.process(...)` call unchanged.
- [x] `Verify Magic Code` step:
  - [x] `const { result } = await input.storage.process({ action: { get: true }, key: ["magic-code", input.challengeId] })`.
  - [x] Treat `result.value === null` as `reason: "expired"` (KV already returns
        that after TTL; `storageFs` mimics via `expiresAt`).
  - [x] On wrong code, `set` the updated record back (preserve `expiresAt` by
        passing the remaining TTL — KV needs a fresh `expireIn` each write, and
        FS recomputes).
  - [x] On success, `set` with `consumedAt` populated.
  - [x] Produce the same `input.auth` shape as today.
- [x] Trim the `json` config block — remove `cacheDir`; leave `ttlMinutes`,
      `length`, `maxAttempts`, `pepper`.
- [x] Update test `inputs[]` to include a `storage` stub (in-memory Map) so the
      dry-run test doesn't touch the real filesystem or KV.

### Phase 3 — New `session.md` helper (also DI-based)

- [ ] Contract mirrors `magicCodeAuth`: requires `input.storage`, ignores where
      it points.
- [ ] `json` config: `sessionTtlDays` (default 7), `cookieName` (default `sid`).
- [ ] Three actions gated by `if: /action/create|read|destroy`.
- [ ] `create`:
  - [ ] `sid = crypto.randomUUID()`.
  - [ ] `storage.set(["session", sid], { email, createdAt }, ttlMs)`.
  - [ ] Produce `input.setCookie` with the full `Set-Cookie` value:
        `sid=<sid>; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=<s>`.
        Set `Secure` off only when `input.mode?.deploy` is false AND the request
        origin is `http://localhost`. Ref:
        [MDN Set-Cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie).
- [ ] `read`:
  - [ ] Parse `Cookie:` header from `input.request.headers`.
  - [ ] `storage.get(["session", sid])` — populate `input.session = { email, sid }` or leave absent.
- [ ] `destroy`:
  - [ ] `storage.delete(["session", sid])`.
  - [ ] Produce `input.setCookie` with `Max-Age=0` to clear the browser copy.

### Phase 4 — Update CLI to inject `storageFs`

- [ ] Edit the `Login` step in [`jiraCli.md`](jiraCli.md#L104):
  - [ ] Add `import storage from "storageFs";` alongside the existing
        `import magicCodeAuth from "magicCodeAuth";`.
  - [ ] Pass `storage` through both `magicCodeAuth.process({ ..., storage })` calls
        (send + verify).
- [ ] Smoke test: `pd run jiraCli.md -- --login` completes end-to-end,
      challenges land in `.cache/pd-storage/magic-code/<id>.json`.
- [ ] Remove the now-dead `.cache/magic-codes/` directory path from any docs.

### Phase 5a — New `webApp.md` entry point (API routes)

Set up HTTP API routes that delegate to existing pipes (`magicCodeAuth`,
`session`, `reportBoards`, `report`). This entry point lives alongside
`jiraCli.md` and shares no business logic — only imported pipes. Reference
[`templates/server.ts`](templates/server.ts) for the request/response
conventions: `input.request`, `input.requestBody` (auto-parsed at
[`server.ts:61-83`](templates/server.ts#L61-L83)), `input.body`,
`input.responseOptions`, and the optional `input.response` escape hatch
([`server.ts:153-167`](templates/server.ts#L153-L167)).

#### Pipe config

- [ ] `json` config:
  ```json
  {
    "static": "./web",
    "defaultContentType": "application/json",
    "parseBody": true,
    "cors": false
  }
  ```
  - `static`: directory served by `serveDir` before pipe execution
    ([`server.ts:88-102`](templates/server.ts#L88-L102)). Frontend wiring is
    Phase 5b; the directory can be empty for now and routes will still work.
    Ref: [`serveDir`](https://jsr.io/@std/http/doc/file-server/~/serveDir).
  - `defaultContentType`: API surface is JSON-first; the template
    auto-stringifies object bodies at
    [`server.ts:188-195`](templates/server.ts#L188-L195).
  - `parseBody`: opt-in JSON/form parsing populates `input.requestBody`.
  - `cors`: `false` because the SPA is served from the same origin as the
    API. Flip to `true` (or an origin string) only if a separate frontend
    host is introduced later.
    Ref: [MDN CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS).

#### Step `Bootstrap` (always runs)

- [ ] Inject the storage backend once for every downstream sub-pipe call:
  ```ts
  import storage from "storageKv";
  input.storage = storage;
  ```
- [ ] Parse the request URL once and pre-compute method + path booleans.
      Pipedown's `if:` directive evaluates JSON-pointer truthiness on `input`,
      so derived booleans are the idiomatic way to branch in a flat pipe (no
      nested conditionals, no shared regex on each step).
  ```ts
  // URL parsing — Ref: https://developer.mozilla.org/en-US/docs/Web/API/URL
  input.url = new URL(input.request.url);
  input.method = input.request.method;

  const path = input.url.pathname;
  input.isPost = input.method === "POST";
  input.isGet  = input.method === "GET";
  input.pathIsAuthSend   = path === "/api/auth/send";
  input.pathIsAuthVerify = path === "/api/auth/verify";
  input.pathIsAuthLogout = path === "/api/auth/logout";
  input.pathIsMe         = path === "/api/me";
  input.pathIsBoards     = path === "/api/boards";
  input.pathIsReport     = path === "/api/report";
  ```

#### Step `Load Session` (always runs)

- [ ] Read the session cookie on every request so downstream auth-gated
      steps can branch on `input.session`. Unauthenticated requests simply
      leave `input.session` undefined.
  ```ts
  import session from "session";
  Object.assign(input, await session.process({
    action: { read: true },
    request: input.request,
    storage: input.storage,
  }));
  ```

#### Step `Send Magic Code` — `if: /isPost`, `if: /pathIsAuthSend`

- [ ] Validate the email and dispatch the challenge. The shape of
      `magicCodeAuth`'s response is the same as the CLI uses at
      [`jiraCli.md:123`](jiraCli.md#L123).
  ```ts
  import magicCodeAuth from "magicCodeAuth";
  const { email } = input.requestBody || {};
  if (!email) {
    input.responseOptions.status = 400;
    input.body = { ok: false, reason: "missing-email" };
    return;
  }
  const result = await magicCodeAuth.process({
    action: { send: true },
    email,
    storage: input.storage,
  });
  if (result.error) {
    input.body = { ok: false, reason: result.error[0]?.reason || "send-failed" };
    return;
  }
  // challengeId is opaque to the SPA; it just round-trips it back on /verify.
  input.body = {
    ok: true,
    challengeId: result.challengeId,
    expiresAt: result.challengeExpiresAt,
  };
  ```

#### Step `Verify And Create Session` — `if: /isPost`, `if: /pathIsAuthVerify`

- [ ] Verify the code, then mint a session. `setCookie` is parked on
      `input` and applied in `Assemble Response`; this keeps cookie wiring
      in one place instead of scattered across handlers.
  ```ts
  import magicCodeAuth from "magicCodeAuth";
  import session from "session";

  const { email, code, challengeId } = input.requestBody || {};
  const verify = await magicCodeAuth.process({
    action: { verify: true },
    email, code, challengeId,
    storage: input.storage,
  });
  if (!verify.auth?.authenticated) {
    input.responseOptions.status = 401;
    input.body = {
      ok: false,
      reason: verify.error?.[0]?.reason || "verify-failed",
      // Surface attemptsLeft so the SPA can show "2 attempts remaining".
      attemptsLeft: verify.error?.[0]?.attemptsLeft,
    };
    return;
  }
  const created = await session.process({
    action: { create: true },
    email: verify.auth.email,
    storage: input.storage,
    mode: input.mode, // session.md uses mode.deploy to decide the Secure flag
  });
  input.setCookie = created.setCookie;
  input.body = { ok: true, email: verify.auth.email };
  ```

#### Step `Logout` — `if: /isPost`, `if: /pathIsAuthLogout`

- [ ] Destroy the server record and emit a `Max-Age=0` cookie so the
      browser drops its copy. Idempotent: missing/expired cookies still
      return `{ ok: true }`.
  ```ts
  import session from "session";
  const destroyed = await session.process({
    action: { destroy: true },
    request: input.request,
    storage: input.storage,
  });
  input.setCookie = destroyed.setCookie;
  input.body = { ok: true };
  ```

#### Step `Me` — `if: /isGet`, `if: /pathIsMe`

- [ ] No 401 here. The SPA hits `/api/me` on load to decide whether to
      show `LoginView` or `BoardsView`; a `null` email is the expected
      "not logged in" signal, not an error.
  ```ts
  input.body = input.session
    ? { email: input.session.email }
    : { email: null };
  ```

#### Step `Boards` — `if: /isGet`, `if: /pathIsBoards`

- [ ] Auth gate, then delegate to `reportBoards`. The pipe currently uses
      shared Jira credentials (env-resolved); per-user Jira creds are an
      open question deferred to post-MVP (see Open Questions).
  ```ts
  if (!input.session) {
    input.responseOptions.status = 401;
    input.body = { error: "unauthorized" };
    return;
  }
  import reportBoards from "reportBoards";
  const result = await reportBoards.process(input);
  input.body = { boards: result.boards };
  ```

#### Step `Report` — `if: /isPost`, `if: /pathIsReport`

- [ ] Same 401 gate. Call shape mirrors the CLI at
      [`jiraCli.md:269`](jiraCli.md#L269) so the CLI and webApp emit
      byte-identical reports — useful for diffing during the migration.
  ```ts
  import report from "report";

  if (!input.session) {
    input.responseOptions.status = 401;
    input.body = { error: "unauthorized" };
    return;
  }
  const { boardId } = input.requestBody || {};
  if (!boardId) {
    input.responseOptions.status = 400;
    input.body = { error: "missing-boardId" };
    return;
  }
  // format.all = true ⇒ table + chart sections, same as `--report` flag.
  const result = await report.process({ ...input, boardId, format: { all: true } });
  input.body = { markdown: result.body };
  ```

#### Step `Assemble Response`

- [ ] Apply any pending `Set-Cookie` header. Centralised here so handlers
      only stash `input.setCookie` and don't poke at `responseOptions`
      directly. 404 fallback is handled by the template at
      [`server.ts:197-207`](templates/server.ts#L197-L207); no work needed.
  ```ts
  if (input.setCookie) {
    // Ref: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie
    input.responseOptions.headers["set-cookie"] = input.setCookie;
  }
  ```

#### Smoke test pipe — `webAppSmoke.md` (new)

A standalone pipe that imports `webApp` and exercises each route
programmatically. No HTTP server, no browser. Validates route wiring
before any frontend exists; runs in CI via `pd test`.

- [ ] `json` config: `{}` — no app config; uses an in-memory storage stub.
- [ ] Step `Setup`:
  - [ ] Build a `Map`-backed `storage` stub matching the storage contract
        (`get`/`set`/`delete` over `keyParts`). This bypasses both `storageFs`
        and `storageKv` so the smoke test doesn't touch the filesystem or
        require Deno KV.
  - [ ] Drive `magicCodeAuth` with `dryRun: true` so `sendEmail` is skipped
        and the plaintext code is returned on the response (existing
        behaviour — see `magicCodeAuth.md` test inputs).
- [ ] Step `Helper`:
  - [ ] Define `input.callRoute = async (method, path, body, cookie) => { ... }`
        that builds a `Request`, calls `webApp.process(...)`, and returns
        `{ status, body, setCookie }`. Avoids 7 copies of the same scaffolding.
- [ ] Step `Send`:
  - [ ] `POST /api/auth/send` with `{ email }`. Assert `body.ok === true`,
        capture `challengeId`.
- [ ] Step `Verify`:
  - [ ] Read the challenge record from the stub storage to recover the
        plaintext code (only possible in dry-run mode).
  - [ ] `POST /api/auth/verify` with `{ email, code, challengeId }`. Assert
        `status === 200`, capture `set-cookie` and extract the `sid`.
- [ ] Step `Me With Cookie`:
  - [ ] `GET /api/me` with `Cookie: sid=<sid>`. Assert `body.email === email`.
- [ ] Step `Boards Unauthed`:
  - [ ] `GET /api/boards` with no cookie. Assert `status === 401`.
- [ ] Step `Boards Authed`:
  - [ ] `GET /api/boards` with cookie. Assert response shape `{ boards: [...] }`.
        Stub `reportBoards` via the storage layer's existing Jira fixture
        cache, or skip with a `BOARDS_FIXTURE` env flag if Jira creds aren't
        present in CI.
- [ ] Step `Logout`:
  - [ ] `POST /api/auth/logout` with cookie. Assert `set-cookie` contains
        `Max-Age=0` and a follow-up `GET /api/me` returns `{ email: null }`.
- [ ] Wire into `pd test webAppSmoke.md` and the project's existing test
      command so a CI break flags any route regression.

### Phase 5b — `webApp.md` static / frontend handling

_Placeholder — to be fleshed out alongside Phase 6 once the API surface in
5a is verified by the smoke test. Will cover SPA delivery via the
`templates/server.ts` static handler, the `index.html` shell, and any
SPA-fallback routing needed for client-side navigation._

### Phase 6 — Frontend (`./web/`)

Mirror the three-file Mithril pattern from
[`pdCli/frontend/home`](/Users/aaronmyatt/pipes/core/pipedown/pdCli/frontend/home):

```
web/
├── index.html
├── app.js
├── state.js
├── styles.css
├── shared/
│   └── base.css          ← copy of pdCli/frontend/shared/base.css
└── components/
    ├── Layout.js
    ├── LoginView.js
    ├── BoardsView.js
    └── ReportView.js
```

- [ ] `index.html`:
  - [ ] `<script src="https://unpkg.com/mithril/mithril.js">`.
  - [ ] `<script src="https://unpkg.com/markdown-it/dist/markdown-it.min.js">` —
        SPA renders `result.markdown` client-side.
  - [ ] `<link rel="stylesheet" href="/shared/base.css">`, `/styles.css`.
  - [ ] `<div id="app"></div>` + `<script src="/state.js">`, `/app.js`.
- [ ] `state.js`:
  - [ ] `globalThis.WEB = { state: { me: null, boards: [], report: null, stage: 'loading' }, actions: {}, components: {} };`.
  - [ ] `WEB.actions.me()` → GET `/api/me`.
  - [ ] `WEB.actions.sendCode(email)` → POST `/api/auth/send`.
  - [ ] `WEB.actions.verify(code)` → POST `/api/auth/verify` (server sets cookie).
  - [ ] `WEB.actions.logout()` → POST `/api/auth/logout`.
  - [ ] `WEB.actions.loadBoards()` → GET `/api/boards`.
  - [ ] `WEB.actions.runReport(boardId)` → POST `/api/report`.
- [ ] `app.js`:
  - [ ] On load, call `WEB.actions.me()` to pick initial view.
  - [ ] `m.mount(document.getElementById("app"), { view: () => m(WEB.components.Layout) })`.
- [ ] Components:
  - [ ] `Layout.js` — topbar (logo, logout if `state.me`), stage switch.
  - [ ] `LoginView.js` — two-step form: email → code. Shows inline errors from `error.reason`.
  - [ ] `BoardsView.js` — grid of `{ id, name }`, click to run report.
  - [ ] `ReportView.js` — renders `state.report.markdown` through markdown-it; back-button returns to boards.
- [ ] Styles:
  - [ ] `shared/base.css` → copied verbatim from the pdCli frontend to inherit the green theme tokens.
  - [ ] `styles.css` → small additions for the login form + boards grid.

### Phase 7 — Deno Deploy deployment

- [ ] Add/extend a root `deno.json` so Deploy can resolve imports. Deploy's
      build step runs `pd build` to regenerate `.pd/`.
- [ ] Entry point: `.pd/webApp/server.ts` (produced by `pd build`).
- [ ] Required Deploy environment variables:
  - `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `SMTP_FROM`
  - `MAGIC_CODE_PEPPER`
  - (Do not set `DENO_KV_PATH` — Deploy provides managed KV automatically.)
- [ ] First deployment: `deployctl deploy --project=<slug> --entrypoint=.pd/webApp/server.ts`
      after `pd build`. Ref: [deployctl](https://docs.deno.com/deploy/manual/deployctl).
- [ ] Optional: link the GitHub repo with build command `pd build` for auto-deploy.
      Ref: [Git integration](https://docs.deno.com/deploy/manual/git-integration).

### Phase 8 — Local dev loop

- [ ] `pd serve webApp.md` starts the site at `http://localhost:8000`.
- [ ] Static assets in `./web` are served by the template's built-in handler
      (see [`templates/server.ts:88`](templates/server.ts#L88)).
- [ ] For local SMTP testing, point `SMTP_HOST` at
      [MailHog](https://github.com/mailhog/MailHog) (port 1025).
- [ ] `pd test` exercises `storageFs`, `storageKv`, and `magicCodeAuth` via
      their `inputs[]` cases with in-memory storage stubs.

### Phase 9 — Verification checklist

- [ ] Local: `pd run jiraCli.md -- --login` still works end-to-end (proves
      `storageFs` + DI wiring).
- [ ] Local: `pd serve webApp.md`, open `http://localhost:8000`:
  - [ ] See the login view.
  - [ ] Submit email → receive code in MailHog → submit code → land on boards.
  - [ ] Pick a board → report renders.
  - [ ] Reload the page → session cookie keeps you logged in.
  - [ ] Logout → returns to login, cookie cleared.
  - [ ] `curl /api/report` without cookie returns 401.
- [ ] Deploy: same walkthrough against the live `*.deno.dev` URL.
- [ ] Swap backends sanity-check: temporarily change `webApp.md` to
      `import storage from "storageFs"` and confirm the site still works
      locally — proves the DI seam is clean.

---

## Files Touched / Created

| File | Status | Notes |
|------|--------|-------|
| `storageFs.md` | new | Filesystem backend exposing get/set/delete/sweep |
| `storageKv.md` | new | Deno KV backend, same contract |
| [`magicCodeAuth.md`](magicCodeAuth.md) | rewrite | Storage-agnostic, requires `input.storage` |
| [`jiraCli.md`](jiraCli.md) | edit | Import `storageFs` and inject into auth calls |
| `session.md` | new | Cookie + session helper, storage-agnostic |
| `webApp.md` | new | Server pipe, imports `storageKv` once and injects everywhere |
| `web/index.html`, `web/app.js`, `web/state.js`, `web/styles.css` | new | SPA |
| `web/shared/base.css` | new (copied) | from `pdCli/frontend/shared` |
| `web/components/Layout.js`, `LoginView.js`, `BoardsView.js`, `ReportView.js` | new | Mithril views |
| `TODO.md` | new | this plan, committed as the running checklist |
| `deno.json` (root) | touch | entries for Deploy + import map if missing |

---

## Open Questions (safe to defer past first deploy)

- Rate limiting on `/api/auth/send` — Deploy has no built-in limiter. MVP: a
  KV-backed counter keyed by email with a short TTL, trivial to add behind the
  same storage contract.
- Board ACL — any authenticated email can view any board's report. Gate with
  `ALLOWED_EMAIL_DOMAINS` in `session.create` if needed.
- `currentLoginPath` / `pendingLoginPath` CLI convenience files — unchanged for
  now; consider consolidating in a later pass.
