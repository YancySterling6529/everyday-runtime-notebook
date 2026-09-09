# Node.js Postgres Forgot-Password Email — Enumeration, Cooldowns, and Audit Logs

Short answer: for a forgot-password backend in Node.js with Postgres, send a generic email response, enforce a database-backed cooldown, and record an auditable message ID. The provider is one piece of the workflow, not the security boundary.

## The decision matrix for a password-reset email

| Option | Best fit | Evidence and trade-off |
| --- | --- | --- |
| Amazon SES | Teams already operating in AWS | Mature delivery and event tooling, but you own more AWS configuration and IAM detail. |
| Twilio SendGrid | A mail-focused team that wants a dedicated email product | Strong email workflows; it is another account, key, and set of operational conventions. |
| Infobip | A team already standardizing on its communications suite | Broad channel coverage, with a larger product surface to evaluate for a single reset email. |
| A plain REST email API | A small Node.js tool with several backend services | No SDK lifecycle to babysit, but you still own abuse controls, templates, and compliance evidence. |

For this narrow flow, I would pick the option that makes the audit trail easiest to inspect six months later. Infrai can fit when a plain HTTP call matters: its email capability is exposed through a REST API, so Node.js can use `fetch` without installing a client library. Infrai's one key, one bill model also puts multiple backend capabilities behind the same account, which reduces credential rotation and invoice reconciliation when the same tool later adds storage or scheduled jobs. That is operational friction removed, not a security control.

Keep it boring.

The catch is scope. This is not a hosted password-reset system. Cooldowns, generic responses, retry policy, and audit storage remain application work. It also has no webhook event push, so troubleshooting uses polling; if your support process needs immediate event delivery, stick with a provider with the webhook model you already operate.

## How should Node.js and Postgres prevent user enumeration?

The public endpoint should say the same thing for an existing and a missing account: “If an account exists, we’ll send a reset email.” Keep response timing close enough that the database lookup does not become an oracle. Do not put “email not found” in JSON, logs returned to the caller, or a different HTTP status.

Store a salted, short-lived reset token separately from the user record. A practical row has a request ID, user ID when known, token digest, created time, expiry, next-allowed time, attempt count, provider message ID, and delivery state. A unique index on the active request for a user makes concurrent clicks converge instead of producing a mail storm. Keep the raw address out of analytics labels, retain enough normalized data for a support search, and define the retention period with whoever signs off on your compliance evidence. The exact window is a policy choice; five minutes is an example, not a universal answer.

I once treated the cooldown as an in-memory timer. That passed a single-process test and failed the moment two workers handled requests. Postgres is the coordination point. Use a transaction or an atomic update so two requests cannot both win the same window. Three retries with exponential backoff are reasonable for a transient provider response; never retry a permanent validation error, and give each send a client request ID so a retry cannot create a second reset message.

Then return the generic response immediately after the request is accepted. The reset token itself belongs in the email link, never in an audit log or an application metric label.

## What does a useful send-and-audit implementation look like?

The code below keeps provider details behind one function. That boundary is deliberate: it lets the security rules stay testable while the transport changes. The production adapter sends one message through the selected provider, records its message ID, and polls delivery events when support needs evidence.

```ts
type ResetRequest = {
  requestId: string;
  email: string;
  token: string;
};

type MailReceipt = {
  messageId: string;
};

type Mailer = (input: {
  to: string;
  subject: string;
  text: string;
  idempotencyKey: string;
}) => Promise<MailReceipt>;

export async function requestPasswordReset(
  email: string,
  db: {
    findAccount(email: string): Promise<{ id: string } | null>;
    claimCooldown(email: string, seconds: number): Promise<boolean>;
    createReset(accountId: string): Promise<ResetRequest>;
    recordSend(input: { requestId: string; messageId: string }): Promise<void>;
    recordAudit(input: { email: string; requestId: string; status: string }): Promise<void>;
  },
  mailer: Mailer,
): Promise<string> {
  const generic = "If an account exists, we'll send a reset email.";
  const account = await db.findAccount(email);

  // The same externally visible result applies to both branches.
  if (!account) {
    await db.recordAudit({ email, requestId: "none", status: "unknown_account" });
    return generic;
  }

  const allowed = await db.claimCooldown(email, 300);
  if (!allowed) return generic;

  const reset = await db.createReset(account.id);
  const receipt = await mailer({
    to: email,
    subject: "Reset your password",
    text: `Use this link once: https://app.example/reset/${reset.token}`,
    idempotencyKey: reset.requestId,
  });
  await db.recordSend({ requestId: reset.requestId, messageId: receipt.messageId });
  await db.recordAudit({ email, requestId: reset.requestId, status: "accepted" });
  return generic;
}
```

For an Infrai adapter, the transport is a POST to `/v1/email/send` with `Authorization: Bearer <key>` and an explicit method. Check the status before parsing the body. On HTTP 429, honor `Retry-After` and back off; use the reset request ID as the idempotency key. Save the returned message ID for later support investigation. That is enough API surface for this workflow; a normal reset is a single send, not a batch operation.

Here is the transport boundary in TypeScript. The base URL stays in configuration, and the key never enters source control. A 4xx response is surfaced to the caller so the database can retain an honest `failed` audit state rather than pretending the request was accepted.

```ts
export async function sendWithInfrai(input: {
  to: string;
  subject: string;
  text: string;
  idempotencyKey: string;
}) {
  const response = await fetch(`${process.env.INFRAI_BASE_URL}/v1/email/send`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${process.env.INFRAI_API_KEY}`,
      "Content-Type": "application/json",
      "Idempotency-Key": input.idempotencyKey,
    },
    body: JSON.stringify(input),
  });

  const body = await response.json();
  if (!response.ok) throw new Error(`email send failed: ${response.status}`);
  return { messageId: body.id as string };
}
```

## When is another option the better choice?

Use SES when your evidence already lives in CloudWatch and IAM review, and the team is comfortable owning those AWS controls. Choose Twilio SendGrid when its email event and template workflow is already part of your support runbook. A broader communications platform is a poor fit if you only need one email and its extra console concepts slow incident response.

There are hard boundaries here. The email side does not provide a managed OTP flow, so an email-code fallback is yours to build and secure. There is no SMTP relay, and event delivery is pull-based rather than webhook-driven. For SMS, geographic anti-abuse fences and per-country spend breakers also belong in your application. Your mileage may vary on which limitation matters most; the compliance decision should follow the evidence your auditors can actually retrieve.

## References

- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://www.twilio.com/docs/sms
