# Storage KV Injectable

In preparation for deployment to Deno Deploy, we will implement a `Deno.KV` storage pipeline as an injectable. This will allow us to abstract away storage details and easily swap out our file system storage for a key-value store implementation. It should adhear to the same interface as our file system storage to minimize changes needed in the rest of our codebase.

> ISO 8601 duration format for 7 days
```json
{
  "ttl": "PT168H"
}
```

## Ensure KV

https://kapeli.com/dash_share?path=https://docs.deno.com/deploy/kv/

- not: /$kv
  ```ts
  input.$kv = await Deno.openKv()
    .catch(() => {
        input.body = [
            "# Deno.KV not available",
            "",
            "The Deno.KV API is not available in this environment. Please ensure you are running in an environment that supports Deno.KV, such as Deno Deploy.",
        ].join("\n");
        throw new Error("Deno.KV not available");
    });
  ```

## Get Item

```ts
input.get = async (keyParts: string[]) => {
    const result = await input.$kv.get(keyParts);
    if (result.value?.expiresAt) {
        const now = Temporal.Now.instant();
        const expiry = Temporal.Instant.from(result.value.expiresAt);
        if (Temporal.Instant.compare(now, expiry) > 0) {
            await input.$kv.delete(keyParts); // Clean up expired entry
            result.value = undefined; // Treat as not found since it's expired
        }
    }
    return result;
};
```

## Set Item

```ts
input.set = async (keyParts: string[], value: any, ttl?: string) => {
    const now = Temporal.Now.instant();
    const ttlDuration = Temporal.Duration.from(ttl || input.ttl || $p.get(opts, "/config/ttl"));
    const expiresAt = now.add(ttlDuration);
    const result = await input.$kv.set(keyParts, Object.assign({ expiresAt: expiresAt.toString() }, value), { expiresIn: ttlDuration.total("milliseconds") });
    return { keyParts, value, expiresAt: expiresAt.toString(), ...result };
};
```

## Delete Item

```ts
input.delete = async (keyParts: string[]) => {
    await input.$kv.delete(keyParts);
    return keyParts;
};
```

## Handle Get

- if: /action/get
  ```ts
  Object.assign(input, await input.get(input.keyParts));
  ```

## Handle Set

- if: /action/set
  ```ts
  Object.assign(input, await input.set(input.keyParts, input.value));
  ```
