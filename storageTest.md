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
console.log("✅ FS Write Test passed");
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

const data = await input.kv.process({
    keyParts: testKey,
    action: { get: true },
});

assert(data.value.message === testValue.message, "Retrieved value should match the stored value");
console.log("✅ KV Write Test passed");
```

## Test FS TTL

```ts
const testKey = ["test", "key"];
const testValue = { message: "Hello, world!", expiresAt: new Date().toISOString() };

await input.fs.process({
    ttl: Temporal.Now.instant().subtract({ seconds: 1 }).toString(), // Set TTL to 1 second in the past
    keyParts: testKey,
    action: { set: true },
    value: testValue,
});

await (new Promise((resolve) => setTimeout(resolve, 1000)));

const data = await input.fs.process({
    keyParts: testKey,
    action: { get: true },
});

assert(data.value?.message !== testValue.message, "Retrieved value should not match the expired value");
assert(data.value === null, "Retrieved value should be null after expiration");
console.log("✅ FS TTL Test passed");
```

## Test KV TTL

```ts
const testKey = ["test", "kv-key"];
const testValue = { message: "Hello, KV!", expiresAt: Temporal.Now.instant().subtract({ seconds: 1 }).toString() };

const $kv = await Deno.openKv(':memory:');

await input.kv.process({
    $kv,
    ttl: 100, // Set TTL to 0.1 second
    keyParts: testKey,
    action: { set: true },
    value: testValue,
});

await (new Promise((resolve) => setTimeout(resolve, 1000)));

const data = await input.kv.process({
    $kv,
    keyParts: testKey,
    action: { get: true },
});

assert(data.value?.message !== testValue.message, "Retrieved value should not match the expired value");
assert(data.value === null, `Retrieved value: ${JSON.stringify(data.value)} should be null after expiration`);
console.log("✅ KV TTL Test passed");
```

