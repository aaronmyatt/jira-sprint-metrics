# Session Test

We need a basic session management cookie handler to persist a users login state across requests. This is a simple test to verify that we can set and read cookies in our server.

We will lean on standard browser `Request` / `Response` objects, supported by Deno, so that we can easily migrate to Deno Deploy without needing to change our underlying request handling code.

```json
{
  "kvPath": ":memory:"
}
```


## Test Setup

```ts
import { assert } from "jsr:@std/assert";
import session from 'session'

input.now = Temporal.Now.instant();
input.$kv = await Deno.openKv(":memory:");
```

## Input object has storage

```ts
const output = await session.process(input);
assert(output.storage, "❌ Expected session process to inject storage");
console.log("✅ Session process injected storage");

const output1 = await session.process(Object.assign(input, { storage: {} }));
assert(output1.storage, "❌ Expected empty object to be replaced with storage implementation");
console.log("✅ Session process injected storage: { storage: {} }");

const output2 = await session.process(Object.assign(input, { storage: null }));
assert(output2.storage, "❌ Expected null to be replaced with storage implementation");
console.log("✅ Session process injected storage: { storage: null }");
```

## TTL Defaults to 7 days

```ts
assert(session.json.config.ttl === 'PT168H', "❌ Expected default TTL to be 7 days (PT168H)");
console.log("✅ Session default TTL is 7 days (PT168H)");
```

## Pass through when no sid/action

```ts
const output = await session.process(Object.assign(input, { request: new Request("https://example.com/test") }));
assert(output.session?.user === undefined, "❌ Expected session process to pass through when no sid/action");
console.log("✅ Session process passed through when no sid/action");
```

## Explicit session creation

```ts
const output = await session.process(Object.assign(input, {
    request: new Request("https://example.com/test"),
    session: { create: 'test@test.com' }
}));

assert(output.session.sid, "❌ Expected session process to create session with provided sid");
assert(output.session.user.email === 'test@test.com', "❌ Expected session process to create session with provided email");
assert(output.response.headers.get("Set-Cookie").includes(`sid=${output.session.sid}`), "❌ Expected session process to set cookie with provided sid");
console.log("✅ Session process created session with provided sid and set cookie");

delete input.session.create; // Clean up for next test
```

## Mock session for cookie test

```ts
input._testSid = crypto.randomUUID();
console.log("Generated dummy session ID:", input._testSid);

await input.$kv.set(["session", input._testSid], { email: "test@test.com" });
```

## Already logged in

```ts
const output = await session.process(Object.assign(input, {
    request: new Request("https://example.com/test", {
        headers: {
            "Cookie": `sid=${input._testSid}`
        }
    })
}));

assert(output.session.sid === input._testSid, "❌ Expected session process to recognize existing session from cookie");
console.log("✅ Session process recognized existing session from cookie");
assert(output.session.user.email === "test@test.com", "❌ Expected session process to recognize existing session user email");
console.log("✅ Session process recognized existing session user email");
```

## Extend session on activity

```ts
import { getSetCookies } from "jsr:@std/http/cookie";

const output = await session.process(Object.assign(input, {
    request: new Request("https://example.com/test", {
        headers: {
            "Cookie": `sid=${input._testSid}`
        }
    })
}));

const cookies = getSetCookies(output.response.headers);
assert(cookies[0].maxAge > 0, "❌ Expected maxAge to be greater than 0");

// wait 1 second and verify that the expires has been extended
await new Promise(resolve => setTimeout(resolve, 1000));

const output1 = await session.process(Object.assign(input, {
    request: new Request("https://example.com/test", {
        headers: {
            "Cookie": `sid=${input._testSid}`
        }
    })
}));

const cookies1 = getSetCookies(output1.response.headers);

assert(cookies1[0].expires > cookies[0].expires, "❌ Expected expires to be extended on activity");
console.log("✅ Session process extended session on activity");
```

## Revoke session on logout

```ts
const output = await session.process(Object.assign(input, {
    request: new Request("https://example.com/test", {
        headers: {
            "Cookie": `sid=${input._testSid}`
        }
    }),
}));

// revoke session signal/convention
$p.set(output, "/session/revoke", true);

const output1 = await session.process(output);

assert(!output1.session?.sid, "❌ Expected session process to revoke session on logout");
console.log("✅ Session process revoked session on logout");

// cookie should be max-age=0
const cookies = getSetCookies(output1.response.headers);


assert(cookies.at(-1).expires < new Date(), "❌ Expected expires to be in the past on logout");
console.log("✅ Session process set cookie expires in the past on logout");

const result = await input.$kv.get(["session", input._testSid]);

assert(result.value === null, "❌ Expected session to be deleted from storage on logout");
console.log("✅ Session process deleted session from storage on logout");
```
