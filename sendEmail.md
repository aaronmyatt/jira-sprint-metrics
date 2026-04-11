# Send Emails

A simple wrapper around npm:nodemailer to send emails with a configured SMTP server. This is used for sending magic codes for authentication, but can be used for any email-sending needs.

Ref: https://nodemailer.com/

## Pull in SMTP Config

```ts
const cfg = $p.get(opts, "/config/smtp") || {};

const readSetting = (inputValue, configValue, envKey) => {
  return [inputValue, configValue, Deno.env.get(envKey)]
    .find(v => v !== undefined && v !== null && v !== "");
};

const toNumber = (value, fallback) => {
  const n = Number(value);
  return Number.isFinite(n) ? n : fallback;
};

input.smtp = {
    host: String(readSetting(input.smtpHost, $p.get(cfg, "/smtpHost"), "SMTP_HOST") || ""),
    port: toNumber(readSetting(input.smtpPort, $p.get(cfg, "/smtpPort"), "SMTP_PORT"), 587),
    user: String(readSetting(input.smtpUser, $p.get(cfg, "/smtpUser"), "SMTP_USER") || ""),
    pass: String(readSetting(input.smtpPass, $p.get(cfg, "/smtpPass"), "SMTP_PASS") || ""),
    from: String(readSetting(input.smtpFrom, $p.get(cfg, "/smtpFrom"), "SMTP_FROM") || ""),
}
```

## Validate Email Address

```ts
const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
if (!emailRegex.test(input.email)) {
  throw new Error("Invalid email address");
}
```

## Create Nodemailer Transporter

```ts
import nodemailer from "npm:nodemailer";

input.transporter = nodemailer.createTransport({
  host: input.smtp.host,
  port: input.smtp.port,
  secure: input.smtp.port === 465, // true for 465, false for other ports
  auth: {
    user: input.smtp.user,
    pass: input.smtp.pass,
  },
});
```

## Verify SMTP Connection

> Before sending emails, you can verify that Nodemailer can connect to your SMTP server. This is useful for catching configuration errors early. Ref: https://nodemailer.com/#verify-the-connection-optional

```ts
await input.transporter.verify();
```

## Send Email

```ts
input.sentEmail = await input.transporter.sendMail({
  from: input.smtp.from,
  to: input.email,
  subject: input.subject || "No Subject",
  text: input.text,
  html: input.html,
});
```
