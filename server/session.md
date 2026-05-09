# Session Handler

```json
{
  "ttl": "PT168H"
}
```

## Std Lib

```ts
import { setCookie, getCookies, deleteCookie } from "jsr:@std/http/cookie";
```

## Storage

- not: /storage
  ```ts
  import StorageKv from "/storage/kv";
  input.storage = StorageKv;
  ```

## Init Response

- not: /response
  ```ts
  input.response = new Response();
  ```

## Check for sid cookie

```ts
const cookies = getCookies(input.request.headers, "sid");
const sid = $p.get(cookies, "/sid");
if (sid) {
    const result = await input.storage.process(Object.assign(input, { 
        action: { get: true}, 
        keyParts: ["session", sid] 
    }));

    if (result.value) {
        $p.set(input, "/session/sid", sid);
        $p.set(input, "/session/user", result.value);
    } else {
        console.log(`Session ID ${sid} not found in storage`);
    }
}
```

## Explicit session creation

We need to facilitate explicit session creation so that a new session can be created following a successful login. This should be a one time event (per session!). Subsequent requests will include the appropriate cookie and be handled by the existing session read logic.

- if: /session/create
  ```ts
  const email = $p.get(input, "/session/create");

  const sid = crypto.randomUUID();
  const relativeTtl = Temporal.Duration.from(input.ttl || $p.get(opts, "/config/ttl")).total("milliseconds");
  const user = { email, expiresAt: new Date(Date.now() + relativeTtl).toISOString(), sid };

  await input.storage.process(Object.assign(input, { 
    action: {set: true},
    keyParts: ["session", sid], 
    value: user, 
    ttl: input.ttl || $p.get(opts, "/config/ttl") 
  }));

  $p.set(input, "/session/sid", sid);
  $p.set(input, "/session/user", user);
  ```

## Set Cookie

Every authenticated request will reset the session cookie with a new expiry, so that user is logged in perpetually as long as they keep using the app within the TTL window.

- if: /session/user
    ```ts
    setCookie(input.response.headers, {
        name: "sid",
        value: $p.get(input, "/session/sid"),
        httpOnly: true,
        secure: input.mode?.deploy !== false, // Set secure flag only in deploy mode
        sameSite: "Lax",
        path: "/",
        maxAge: Temporal.Duration.from(input.ttl || $p.get(opts, "/config/ttl")).total("seconds"),
        // redundant with maxAge but easier to test!
        expires: new Date(Date.now() + Temporal.Duration.from(input.ttl || $p.get(opts, "/config/ttl")).total("milliseconds"))
    });
    ```

## Revoke Session

- if: /session/revoke
  ```ts
  const output = await input.storage.process(input);
  await output.delete(["session", input.session.sid]);

  // Clear session cookie
  deleteCookie(input.response.headers, "sid", {
    httpOnly: true,
    secure: input.mode?.deploy !== false, // Set secure flag only in deploy mode
    sameSite: "Lax",
    path: "/",
  });
  delete input.session;
  ```
