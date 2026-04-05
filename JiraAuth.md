# Jira API Authentication

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