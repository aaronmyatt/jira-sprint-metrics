# Jira API Authentication

## Build Auth Header

Construct the Base64-encoded Basic auth header and the base URL from the Jira credentials in `Deno.env`. Store them on `input` so every subsequent fetch step can reuse them without re-reading config.

Deno.env.get("JIRA_DOMAIN")
Deno.env.get("JIRA_EMAIL")
Deno.env.get("JIRA_TOKEN")

```ts
const makeAuthString = (email, token) => {
  const _email = email || Deno.env.get("JIRA_EMAIL");
  const _token = token || Deno.env.get("JIRA_TOKEN");
  if (!_email || !_token) {
    throw new Error("Missing JIRA_EMAIL or JIRA_TOKEN in environment variables");
  }
  return `${_email}:${_token}`;
};

input.baseUrl = `https://${Deno.env.get("JIRA_DOMAIN")}` || input.jiraBaseUrl;
const authString = makeAuthString(input.jiraEmail, input.jiraToken);
input.authHeader = `Basic ${btoa(authString)}`;
```

## Clear Old Cache
Walk the cache dir and remove any .json files older than 24hrs.

```ts
import { walk } from "jsr:@std/fs/walk";
import { join } from "jsr:@std/path";

const cacheDir = `.cache/`;
try {
  for await (const entry of walk(cacheDir, { maxDepth: 1 })) {
    if(entry.path.endsWith(".json")) {
      const stats = await Deno.stat(entry.path);
      const ageInDays = (Date.now() - stats.mtime.getTime()) / (1000 * 60 * 60 * 24);
      if (ageInDays > 1) {
        await Deno.remove(entry.path);
      }
    }
  }
} catch {
// ignore if cache directory doesn't exist or any file errors
}
```

## Fetch or Cache Helper

Wrap fetch calls in a helper function that checks for cached responses in `~/.cache/jira-boards/` first. Cache each response as `{endpoint}_{params}_{date}.json` so we can skip API calls during development and testing.

```ts
async function fetchWithCache(endpoint, params = {}, cacheDir = ".cache/jira-boards") {
  const endpointKey = endpoint.replaceAll("/", "_").replace(/^_+|_+$/g, "");
  const paramsKey = Object.entries(params).map(([k,v]) => `${k}-${v}`).join("_");
  const dayKey = new Date().toISOString().split("T")[0]; // daily cache invalidation
  const cacheKey = `${endpointKey}${paramsKey ? `_${paramsKey}` : ""}_${dayKey}.json`;
  const cachePath = `${cacheDir}/${cacheKey}`;

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
