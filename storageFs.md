# File System Storage Injectable

We will pass in a file system storage pipeline to abstract away these details and enable us to swap out for a `Deno.KV` implementation in preparation for deployment to Deno Deploy.

By default we will use the `Deno.FileSystem` API to store our data in a local directory. We can configure the root directory for our file system storage in the `config.json` file.

```json
{
    "root": ".cache/",
    "ttl": ""
}
```

```zod
export const schema = z.object({
    root: z.string().default(".cache/"),
    ttl: z.string().default(() => {
        const now = Temporal.Now.instant();
        
        // Add 7 days (as 168 hours, since Instant only accepts time units)
        // Ref: https://tc39.es/proposal-temporal/#sec-temporal-duration-from
        const expiresAt = now.add({
            hours: 7 * 24,  // 7 days = 168 hours
        });
        return expiresAt.toString();
    }),
    key: z.array(z.string()),
});
```

```ts
console.log(input.key)
```

## Create Cache Directory

```ts
import { join } from "jsr:@std/path";
import { ensureDir, ensureFile } from "jsr:@std/fs";
    
input.cacheDir = join(Deno.cwd(), opts.config.root);
await ensureDir(input.cacheDir);
```

## Get Item

```ts
// Parse stored ISO string back to Temporal.Instant for proper comparison
// Ref: https://tc39.es/proposal-temporal/#sec-temporal.instant.from
input.raiseWhenExpired = async (data: { value: any; expiresAt?: string }, filePath: string) => {
    if (data.expiresAt) {
        const now = Temporal.Now.instant();
        const expiry = Temporal.Instant.from(data.expiresAt);
        
        // Compare Temporal instants directly (returns -1, 0, or 1)
        // Ref: https://tc39.es/proposal-temporal/#sec-temporal.instant.prototype.compare
            console.log(`Data at ${filePath} has expired (expired at ${expiry.toString()}, now is ${now.toString()})`);
        if (Temporal.Instant.compare(now, expiry) > 0) {
            await Deno.remove(filePath); // Clean up expired file
            throw new Error("Data has expired");
        }
    }
};

input.get = async (key: string[]) => {
    const filePath = join(input.cacheDir, ...key);
    try {
        const raw = await Deno.readTextFile(filePath);
        console.log(`Attempting to read cache file: ${filePath}`, raw);
        const data = JSON.parse(raw);
        await input.raiseWhenExpired(data, filePath); // Check if the data has expired
        return {
            key: filePath,
            value: data.value,
            versionstamp: data.expiresAt, // ISO string from Temporal.Instant
        }
    } catch (error) {
        if (error instanceof Deno.errors.NotFound) {
            return {
                key: filePath,
                value: undefined,
                versionstamp: undefined,
            }; // Key not found
        }
        throw error; // Rethrow other errors
    }
};
```

## Handle Get

- if: /action/get
- ```ts
  input.result = await input.get(input.key);
  ```

## Delete Item

```ts
input.delete = async (key: string[]) => {
    const filePath = join(input.cacheDir, ...key);
    try {
        await Deno.remove(filePath);
        return { key: filePath };
    } catch (error) {
        if (error instanceof Deno.errors.NotFound) {
            return { key: filePath }; // Key not found, consider it deleted
        }
        throw error; // Rethrow other errors
    }
};
```

## Sweep Expired Items

Since it will be fairly cheap to read the directory and check for expired items, we will do this on every operation to ensure we are not keeping around expired data.

```ts
import { walk } from "jsr:@std/fs/walk";

for await (const entry of walk(input.cacheDir)) {
    if (entry.isFile) {
        const filePath = join(input.cacheDir, entry.name);
        try {
            input.get([entry.path]); // This will automatically delete expired files
        } catch (error) {
            console.error(`Error processing cache file ${filePath}:`, error);
        }
    }
}
```

## Set Item

We want to follow a tempfile rename strategy when writing files to ensure
atomic writes and avoid race conditions. We will write to a temporary file
first, then rename it to the final destination.

```ts
import { ensureDir, ensureFile } from "jsr:@std/fs";

input.set = async (key: string[], value: any, ttl?: number) => {
    const uuid = crypto.randomUUID();
    const tempFilePath = join(input.cacheDir, ...key, uuid) + ".tmp";
    const filePath = join(input.cacheDir, ...key) + ".json";
    const expiresAt = ttl || opts.config.ttl || input.ttl;
    const data = {
        value,
        expiresAt
    };

    await ensureFile(tempFilePath);
    await Deno.writeTextFile(tempFilePath, JSON.stringify(data), { create: true });
    await Deno.rename(tempFilePath, filePath);
    Deno.remove(tempFilePath).catch(() => {}); // Clean up temp file if it still exists
    return {
        versionstamp: expiresAt,
    }
}
```

## Handle Set

- if: /action/set
- ```ts
  input.result = await input.set(input.key, input.value, input.ttl);
  ```
