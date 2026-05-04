# Storage Test

## Imports

```ts
import { assert } from "jsr:@std/assert";

import StorageFs from "storageFs";
import StorageKv from "storageKv";
input.fs = StorageFs;
input.kv = StorageKv;
```

## FS Write Test

```ts
const testKey = ["test", "key"];
const testValue = { message: "Hello, world!", expiresAt: new Date().toISOString() };

await input.fs.process({
    keyParts: testKey,
    action: { set: true },
    value: testValue,
});

const data = await input.fs.process({
    keyParts: testKey,
    action: { get: true },
});

assert(data.value.message === testValue.message, "Retrieved value should match the stored value");
```

## KV Write Test

```ts
const testKey = ["test", "kv-key"];
const testValue = { message: "Hello, KV!", expiresAt: Temporal.Now.instant().add({ hours: 1 }).toString() };

const setWat = await input.kv.process({
    keyParts: testKey,
    action: { set: true },
    value: testValue,
});
console.log("Set in KV:", setWat);

const data = await input.kv.process({
    keyParts: testKey,
    action: { get: true },
});
console.log("Retrieved from KV:", data);

assert(data.value.message === testValue.message, "Retrieved value should match the stored value");
```
