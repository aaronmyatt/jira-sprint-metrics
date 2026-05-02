# Email Magic Code Authentication

Issues and verifies short-lived email magic codes for passwordless sign-in.

This pipe supports two actions:

1. `send` - generates a challenge and emails a code
2. `verify` - validates the code and marks the challenge as consumed

Set SMTP credentials in environment variables (or pass them on `input`) similarly to `JiraAuth.md`.

```json
{
  "magicCode": {
    "ttlMinutes": 10,
    "length": 6,
    "maxAttempts": 5
  },
  "inputs": [
    {
      "_name": "dry-run send",
      "action": {"send": true},
      "email": "dev@example.com",
      "dryRun": true
    }
  ]
}
```

## Storage

- not: /storage
  ```ts
  import StorageFs from "storageFs";
  input.storage = StorageFs;
  ```

## Build Auth + SMTP Context

Read runtime settings from `input`, pipe config, and `Deno.env` (in that precedence order), and expose normalized values on `input` for later steps.

Environment keys:
- `MAGIC_CODE_TTL_MINUTES`
- `MAGIC_CODE_LENGTH`
- `MAGIC_CODE_MAX_ATTEMPTS`
- `MAGIC_CODE_PEPPER`

```ts
const cfg = $p.get(opts, "/config/magicCode") || {};

const readSetting = (inputValue, configValue, envKey) => {
  return [inputValue, configValue, Deno.env.get(envKey)]
    .find(v => v !== undefined && v !== null && v !== "");
};

const toNumber = (value, fallback) => {
  const n = Number(value);
  return Number.isFinite(n) ? n : fallback;
};

input.action = (input.action || {"send": true});
input.email = String(input.email || "").trim().toLowerCase();
input.code = String(input.code || "").trim();
input.challengeId = String(input.challengeId || "").trim();

input.magicCode = {
  ttlMinutes: toNumber(readSetting(input.ttlMinutes, $p.get(cfg, "/ttlMinutes"), "MAGIC_CODE_TTL_MINUTES"), 10),
  length: toNumber(readSetting(input.codeLength, $p.get(cfg, "/length"), "MAGIC_CODE_LENGTH"), 6),
  maxAttempts: toNumber(readSetting(input.maxAttempts, $p.get(cfg, "/maxAttempts"), "MAGIC_CODE_MAX_ATTEMPTS"), 5),
  cacheDir: String(readSetting(input.cacheDir, $p.get(cfg, "/cacheDir"), "MAGIC_CODE_CACHE_DIR") || ".cache/magic-codes"),
  pepper: String(readSetting(input.pepper, $p.get(cfg, "/pepper"), "MAGIC_CODE_PEPPER") || ""),
};

if (!input.email) {
  throw new Error("Missing email. Provide input.email.");
}

const encoder = new TextEncoder();
input.sha256 = async (value) => {
  const digest = await crypto.subtle.digest("SHA-256", encoder.encode(value));
  return Array.from(new Uint8Array(digest)).map((b) => b.toString(16).padStart(2, "0")).join("");
};
```

## Send Code

- if: /action/send
  ```ts
  import sendEmail from "sendEmail";

  const buildCode = (length = 6) => {
    const digits = "0123456789";
    const bytes = crypto.getRandomValues(new Uint8Array(length));
    return Array.from(bytes).map((b) => digits[b % 10]).join("");
  };

  const now = Date.now();

  const code = buildCode(input.magicCode.length);
  const challengeId = crypto.randomUUID();
  const expiresAt = new Date(now + input.magicCode.ttlMinutes * 60 * 1000).toISOString();
  const codeHash = await input.sha256(`${input.email}:${code}:${input.magicCode.pepper}:${challengeId}`);

  const challenge = {
      challengeId,
      email: input.email,
      codeHash,
      expiresAt,
      attempts: 0,
      maxAttempts: input.magicCode.maxAttempts,
      createdAt: new Date(now).toISOString(),
      consumedAt: null,
  };

  await input.storage.process({
    keyParts: ["magic-code-challenges", challengeId+".json"],
    action: { set: true },
    value: challenge,
    ttl: expiresAt,
  });

  input.challengeId = challengeId;
  input.challengeExpiresAt = expiresAt;

  if (!input.dryRun) {
    input.emailSent = await sendEmail.process({
      email: input.email,
      subject: "Your magic sign-in code",
      text: `Your code is ${code}. It expires in ${input.magicCode.ttlMinutes} minutes.`,
      html: `<p>Your code is <strong>${code}</strong>.</p><p>It expires in ${input.magicCode.ttlMinutes} minutes.</p>`,
    });
  }
  ```

## Verify Magic Code

Generate/send a code for `send`, or validate a challenge for `verify`.

- if: /action/verify
  ```ts
  const challenge = await input.storage.process({
    keyParts: ["magic-code-challenges", input.challengeId+".json"],
    action: { get: true },
  });

  if (challenge.value.consumedAt) {
    input.error = [{ action: input.action, authenticated: false, reason: "already-used" }];
    return;
  }

  if (challenge.value.email !== input.email) {
    input.error = [{ action: input.action, authenticated: false, reason: "email-mismatch" }];
    return;
  }

  if (Date.now() > new Date(challenge.value.expiresAt).getTime()) {
    input.error = [{ action: input.action, authenticated: false, reason: "expired" }];
    return;
  }

  if (challenge.value.attempts >= challenge.value.maxAttempts) {
    input.error = [{ action: input.action, authenticated: false, reason: "max-attempts" }];
    return;
  }

  const candidateHash = await input.sha256(`${input.email}:${input.code}:${input.magicCode.pepper}:${input.challengeId}`);
  const ok = candidateHash === challenge.value.codeHash;

  if (!ok) {
    challenge.value.attempts += 1;
    await input.storage.process({
      keyParts: ["magic-code-challenges", input.challengeId+".json"],
      action: { set: true },
      value: challenge.value,
    });
    input.error = [{
      action: input.action,
      authenticated: false,
      reason: "invalid-code",
      attemptsLeft: Math.max(0, challenge.value.maxAttempts - challenge.value.attempts),
    }];
    return;
  }

  challenge.value.consumedAt = new Date().toISOString();
  await input.storage.process({
    keyParts: ["magic-code-challenges", input.challengeId+".json"],
    action: { set: true },
    value: challenge.value,
  });

  input.auth = {
    action: input.action,
    authenticated: true,
    email: input.email,
    challengeId: input.challengeId,
    authenticatedAt: challenge.value.consumedAt,
  };
  ```
