# File System Storage Injectable

We will pass in a file system storage pipeline to abstract away storage details
and enable us to swap out for a `Deno.KV` implementation in preparation for
deployment to Deno Deploy.

By default we will use the `Deno.FileSystem` API to store our data in a
local directory. We can configure the root directory for our
file system storage in the `config.json` file.

```json
{
    "root": ".cache/",
    "ttl": "PT168H"
}
```

```zod
export const schema = z.object({
    root: z.string().default(".cache/"),
    ttl: z.string().default(() => {
        const now = Temporal.Now.instant();
        
        // Add 7 days (as 168 hours, since Instant only accepts time units)
        // Ref: https://tc39.es/proposal-temporal/#sec-temporal-duration-from
        const expiresAt = now.add('PT168H');
        return expiresAt.toString();
    }),
    keyParts: z.array(z.string()),
});
```

## Utils

```ts
// Parse stored ISO string back to Temporal.Instant for proper comparison
// Ref: https://tc39.es/proposal-temporal/#sec-temporal.instant.from
input.raiseWhenExpired = async (data: { value: any; expiresAt?: string }, filePath: string) => {
    if (data.expiresAt) {
        const now = Temporal.Now.instant();
        const expiry = Temporal.Instant.from(data.expiresAt);
        
        // Compare Temporal instants directly (returns -1, 0, or 1)
        // Ref: https://tc39.es/proposal-temporal/#sec-temporal.instant.prototype.compare
        if (Temporal.Instant.compare(now, expiry) > 0) {
            await Deno.remove(filePath); // Clean up expired file
            throw new Error("Data has expired");
        }
    }
};

input.readOrRaise = async (filePath: string) => {
    const raw = await Deno.readTextFile(filePath);
    const data = JSON.parse(raw);
    await input.raiseWhenExpired(data, filePath); // Check if the data has expired
    return data;
}

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
input.get = async (keyParts: string[]) => {
    const filePath = join(input.cacheDir, ...keyParts);
    try {
        const data = await input.readOrRaise(filePath);
        return {
            key: keyParts,
            value: data.value,
            versionstamp: data.expiresAt, // ISO string from Temporal.Instant
        }
    } catch (error) {
        console.error(`Error reading cache file ${filePath}:`, error);
        return {
            key: keyParts,
            value: null,
            versionstamp: null,
        };
    }
};
```

## Handle Get

- if: /action/get
- ```ts
  Object.assign(input, await input.get(input.keyParts));
  ```

## Delete Item

```ts
input.delete = async (keyParts: string[]) => {
    const filePath = join(input.cacheDir, ...keyParts);
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

Since it will be fairly cheap to read the directory and check for expired items,
we will do this on every operation to ensure we are not keeping around expired data.

I was keep to reuse the get method here but it would require
wrangling the absolute path returned by `walk`, so I'll lean
on the extracted readOrRaise method

```ts
import { walk } from "jsr:@std/fs/walk";

for await (const entry of walk(input.cacheDir)) {
    if (entry.isFile) {
        await input.readOrRaise(entry.path)
            .catch((error) => {
                if (error.message === "Data has expired") {
                    console.debug(`Removing expired cache file: ${entry.path}`);
                } else {
                    console.error(`Error reading cache file ${entry.path}:`, error);
                }
            });
    }
}
```

## Set Item

We want to follow a tempfile rename strategy when writing files to ensure
atomic writes and avoid race conditions. We will write to a temporary file
first, then rename it to the final destination.

```ts
import { ensureDir, ensureFile } from "jsr:@std/fs";

input.set = async (keyParts: string[], value: any, ttl?: number) => {
    const uuid = crypto.randomUUID();
    const tempFilePath = join(input.cacheDir, ...keyParts) + `-${uuid}.tmp`;
    const filePath = join(input.cacheDir, ...keyParts);
    const expiresAt = ttl || input.ttl || opts.config.ttl;
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
        ok: true,
    }
}
```

## Handle Set

- if: /action/set
- ```ts
  Object.assign(input, await input.set(input.keyParts, input.value));
  ```
