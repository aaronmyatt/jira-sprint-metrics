# Jira API Authentication

## Build Auth Header

Construct the Base64-encoded Basic auth header and the base URL from Jira credentials resolved from `input`, `.cache/jira-auth.json`, or `Deno.env`. Store them on `input` so every subsequent fetch step can reuse them without re-reading config.

Deno.env.get("JIRA_DOMAIN")
Deno.env.get("JIRA_EMAIL")
Deno.env.get("JIRA_TOKEN")

```ts
const readCachedAuth = async () => {
  try {
    return JSON.parse(await Deno.readTextFile(".cache/jira-auth.json"));
  } catch {
    return {};
  }
};

const readSetting = (...values) => values.find((value) => value !== undefined && value !== null && String(value).trim() !== "");

const normalizeBaseUrl = (value) => {
  const raw = String(value || "").trim();
  if (!raw) return "";

  if (raw.startsWith("http://") || raw.startsWith("https://")) {
    return raw.replace(/\/+$/, "");
  }

  return `https://${raw.replace(/^\/+/, "").replace(/\/+$/, "")}`;
};

const cachedAuth = await readCachedAuth();
const jiraDomain = String(readSetting(input.jiraDomain, input.JIRA_DOMAIN, cachedAuth.JIRA_DOMAIN, Deno.env.get("JIRA_DOMAIN")) || "").trim();
const jiraEmail = String(readSetting(input.jiraEmail, input.JIRA_EMAIL, cachedAuth.JIRA_EMAIL, Deno.env.get("JIRA_EMAIL")) || "").trim();
const jiraToken = String(readSetting(input.jiraToken, input.JIRA_TOKEN, cachedAuth.JIRA_TOKEN, Deno.env.get("JIRA_TOKEN")) || "").trim();
const baseUrl = normalizeBaseUrl(readSetting(input.jiraBaseUrl, cachedAuth.jiraBaseUrl, jiraDomain));

if (!jiraEmail || !jiraToken) {
  throw new Error("Missing JIRA_EMAIL or JIRA_TOKEN. Run the CLI setup flow or set environment variables.");
}

if (!baseUrl) {
  throw new Error("Missing JIRA_DOMAIN. Run the CLI setup flow or set environment variables.");
}

input.jiraDomain = jiraDomain;
input.jiraEmail = jiraEmail;
input.jiraToken = jiraToken;
input.baseUrl = baseUrl;
input.authHeader = `Basic ${btoa(`${jiraEmail}:${jiraToken}`)}`;
```

## Clear Old Cache

Walk the cache dir and remove any .json files older than 24hrs.

```ts
import { walk } from "jsr:@std/fs/walk";
import { join } from "jsr:@std/path";

const cacheDir = `.cache/`;
try {
  for await (const entry of walk(cacheDir, { maxDepth: 1 })) {
    if (!entry.isFile || !entry.path.endsWith(".json")) continue;

    const stats = await Deno.stat(entry.path);
    if (!stats.mtime) continue;

    const ageInDays = (Date.now() - stats.mtime.getTime()) / (1000 * 60 * 60 * 24);
    if (ageInDays > 1) {
      await Deno.remove(entry.path);
    }
  }
} catch {
// ignore if cache directory doesn't exist or any file errors
}
```

## Fetch or Cache Helper

Wrap fetch calls in a helper function that checks for cached responses in `~/.cache/jira-boards/` first. Cache each response as `{endpoint}_{params}_{date}.json` so we can skip API calls during development and testing.

```ts
async function fetchWithCache(endpoint, params = {}, cacheParts = [".cache", "jira"]) {
  const endpointKey = endpoint.replaceAll("/", "_").replace(/^_+|_+$/g, "");
  const paramsKey = Object.entries(params).map(([k,v]) => `${k}-${v}`).join("_");
  const dayKey = new Date().toISOString().split("T")[0]; // daily cache invalidation
  const cacheKey = `${endpointKey}${paramsKey ? `_${paramsKey}` : ""}_${dayKey}.json`;
  const cachePath = join(...cacheParts, cacheKey);

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
    await Deno.mkdir(join(...cacheParts), { recursive: true });
    await Deno.writeTextFile(cachePath, JSON.stringify(data));
    return data; 
  }
}

input.fetchWithCache = fetchWithCache;
```
