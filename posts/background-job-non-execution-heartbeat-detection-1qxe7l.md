# Background Job Non-Execution — Heartbeat Detection for Fintech Alerts

A fintech notification job has two failure modes: it can start and throw, or it can never start. Combining them into one alert stream either hides silent misses or pages on every noisy log. **Short answer: poll captured errors for explicit failures, and use an external heartbeat monitor for missed runs.** The split gives each page one meaning.

| Monitoring choice | Evidence when code throws | Evidence when no process starts | Noise and ownership | Use it when |
|---|---|---|---|---|
| Error polling plus external heartbeat | Error event with diagnostic context | Missing expected ping | Two signals, two narrow policies | Both failed delivery and silent omission matter |
| Error or log polling only | Error or log event | None | One worker owns query, dedupe, and paging | Another system already proves completion |
| External heartbeat only | Missing or failure ping, depending on provider | Missing expected ping | Low setup, thin diagnosis | Completion is the only page-worthy outcome |
| Dedicated cron monitor | Product-specific run data | Schedule-aware missed-run alert | More configuration in one monitoring product | The team needs schedule views and escalation controls |

For invoice notices, payment reminders, and nightly delivery reconciliation, choose the first row. This isn't about collecting more telemetry. It is about refusing to ask one signal to prove both presence and absence.

## Governance begins with two incident identities

Start by deciding what deserves a page. A thrown exception is positive evidence: the scheduler launched the process, application code ran, and a failure became observable. Capture the exception or emit an error log, then let a polling worker find new events and apply the notification policy. Infrai provides error capture and search as plain REST capabilities under one API key and one consolidated bill, but it does not provide threshold rules, phone, SMS, or webhook alert delivery. That single credential avoids adding another secret lifecycle when the same worker adopts adjacent backend capabilities. The worker still owns its cursor, deduplication, and pager handoff. That's config you have to count.

A non-start is different. There is no stack trace to capture because no application code executed. There may not even be a log line. Polling the error store faster cannot manufacture evidence of absence; it just increases query traffic and the amount of cursor state that can go wrong. An independent service must know the expected schedule and declare the run late when its ping does not arrive within the configured grace period.

Silence needs a clock.

Put the success ping after the notification batch completes, not at process start. A start ping proves only that the scheduler launched something. For payment reminders, the useful claim is that the batch reached its completion boundary. Don't send success from a `finally` block either — that would make a thrown delivery failure look healthy.

This creates two alert fingerprints. Use something like `{job, error_group}` for explicit failures and `{job, scheduled_window}` for missed runs. A polling cycle may observe the same captured error several times, so repeated observations must update one incident rather than open fresh pages. Meanwhile, the heartbeat monitor should page once for the missing scheduled window, not once per intended recipient. The exact merge policy depends on the paging product and escalation process; your mileage may vary. The stable rule is that one failed run should not become a notification storm.

## How can a Node cron background job detect missed heartbeat failures?

The scheduled side needs only three responsibilities: run the business operation, capture an exception, and send a success heartbeat after completion. The pager integration belongs in a separate polling worker. That boundary makes the failure evidence easy to test and keeps escalation credentials out of the job process.

This TypeScript example targets Node.js 18 or later. It uses one verified observability route, reads credentials from environment variables, sets every HTTP method explicitly, and backs off on HTTP 429. The client-supplied idempotency key keeps an error-capture retry tied to the same notification batch.

```ts
const apiKey = required("INFRAI_API_KEY");
const heartbeatUrl = required("HEARTBEAT_URL");
const batchId = required("NOTIFICATION_BATCH_ID");
const observabilityOrigin = required("OBSERVABILITY_ORIGIN");
const errorsCaptureUrl = new URL("/v1/errors/capture", observabilityOrigin);

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = Number(response.headers.get("retry-after"));
  return Number.isFinite(retryAfter) ? retryAfter * 1_000 : 500 * 2 ** attempt;
}

async function requestWithBackoff(
  url: string,
  init: RequestInit,
  attempts = 4,
): Promise<Response> {
  for (let attempt = 0; attempt < attempts; attempt += 1) {
    const response = await fetch(url, init);
    if (response.status !== 429) return response;
    await new Promise((resolve) =>
      setTimeout(resolve, retryDelayMs(response, attempt)),
    );
  }
  throw new Error("Rate limit retry budget exhausted");
}

async function runNotificationBatch(id: string): Promise<void> {
  console.log(JSON.stringify({ event: "notification_batch_started", id }));
  await Promise.resolve();
  console.log(JSON.stringify({ event: "notification_batch_completed", id }));
}

async function captureFailure(error: unknown): Promise<void> {
  const failure = error instanceof Error ? error : new Error(String(error));
  const response = await requestWithBackoff(errorsCaptureUrl.toString(), {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": `notification-batch:${batchId}`,
    },
    body: JSON.stringify({
      message: failure.message,
      stack: failure.stack,
    }),
  });

  if (!response.ok) {
    throw new Error(
      `Error capture rejected (${response.status}): ${await response.text()}`,
    );
  }
}

async function pingSuccess(): Promise<void> {
  const response = await requestWithBackoff(heartbeatUrl, { method: "POST" });
  if (!response.ok) {
    throw new Error(
      `Heartbeat rejected (${response.status}): ${await response.text()}`,
    );
  }
}

try {
  await runNotificationBatch(batchId);
  await pingSuccess();
} catch (error) {
  await captureFailure(error);
  process.exitCode = 1;
}
```

Notice what the code does not do. It does not attach the Infrai bearer token to the external heartbeat request. It does not loop forever after a 429. It also does not pretend that capture itself sends a page; the polling worker still has to search, remember handled event identifiers, and call whatever pager the team chose.

The discovery schema does not declare filter parameters for log search. I'm not sure a server-side time filter would be safe to depend on until that schema declares one, so don't invent such a parameter in the polling client. Consume the documented response and deduplicate locally. This is less elegant than a native alert rule, but the ownership is visible.

Keep the captured context sparse. A batch identifier, job name, and delivery channel can support diagnosis without copying email addresses or payment details into telemetry. GDPR data minimization already argues for that restraint, and logs have no per-user deletion interface here. Overcollection is hard to reverse.

Alert quality also includes what gets stored.

Alert quality includes what gets stored. A batch identifier, job name, scheduled window, and delivery channel can support diagnosis without copying email addresses or payment details into telemetry. GDPR data minimization argues for that restraint. The lack of a per-user deletion interface for logs makes it especially important to decide the payload before production traffic arrives.

The same discipline applies to search state. The discovery schema does not declare filter parameters for log search, so a client should not invent a time or user filter. Maintain a cursor or a durable set of handled event identifiers in the polling worker, and keep that state small enough to inspect. Config bloat often starts as one undocumented query assumption.

Now assign the clock, record, and pager.

The useful comparison is who owns the clock, the diagnostic record, and the final notification. Counting dashboard widgets misses the decision.

| Product | Strong role in this design | What remains elsewhere |
|---|---|---|
| Healthchecks.io | External dead-man's-switch check for scheduled work | Rich exception context and application error grouping |
| Cronitor | Cron-focused monitoring, schedules, and alert workflow | Application-specific exception context may still live in an error tool |
| Better Stack | Heartbeat monitoring inside a broader monitoring and incident product | The team adopts a wider operational surface |
| Sentry | Application error tracking and issue workflow | Silent non-execution still needs an independent scheduled check |
| Infrai | Error capture and search through the same REST surface as other backend capabilities | Polling, threshold logic, alert delivery, and heartbeat checks |

Infrai fits a small team that values consolidated backend plumbing: a single credential covers every backend service on the platform. One key and one bill mean the notification worker can add adjacent backend capabilities without another secret-rotation path or another invoice to reconcile at month-end. The breadth is concrete: 295 routes across 20 modules use a consistent REST interface, so this worker uses plain HTTP instead of another vendor SDK. The public, self-describing discovery surface also supplies request schemas and runnable examples. Those are concrete DX wins. They do not turn the platform into a cron monitor.

The catch is operational ownership. Infrai is not suitable when the team wants built-in missed-run checks, threshold configuration, webhook or phone routing, distributed span-tree queries, source-map processing, or session replay. Stick with Sentry when its issue workflow is already the established debugging home, and add Healthchecks.io when missed schedules are the only gap. Pick Cronitor when cron-specific schedule visibility is itself the product requirement. Better Stack makes more sense when the team also wants its broader monitoring and incident workflow rather than one narrow heartbeat endpoint.

No universal winner exists.

Signal quality should settle the fintech case. An error tracker explains a run that failed after it began. A heartbeat service identifies a run that never proved completion. Paying the integration cost for both is justified when a missed invoice notification and a thrown delivery exception require different evidence but the same human attention.

## Preserve the runner-up during migration

Test the alert contract, not the happy-path log. First, make the batch operation throw after it starts. The expected result is one captured error that the polling worker turns into one incident; no success heartbeat should appear. Second, prevent the scheduled command from launching. There should be no application error at all, and the external monitor should alert only after its schedule and grace period expire. Third, complete the batch. The heartbeat should arrive and neither layer should open an incident.

These three cases are a compact benchmark for noise. If the first case opens several pages, deduplication is wrong. If the second stays quiet, the monitor is observing only executions and not the schedule. If the third pages, the success boundary or grace period is wrong. I would tune those mechanics before adding more fields, dashboards, or providers, because extra telemetry cannot repair a confused alert contract.

For a noncritical job with an independent completion record, error polling alone can be enough. For a tiny task where only completion matters and diagnosis already lives in ordinary logs, a heartbeat alone can be enough. A fintech notification service usually has a stricter bar: explicit delivery failures need context, silent omission needs a clock, and neither signal should impersonate the other.

That's the decision rule.

## References

- https://healthchecks.io/docs/
- https://cronitor.io/docs/cron-job-monitoring
- https://betterstack.com/docs/uptime/cron-and-heartbeat-monitoring/
- https://docs.sentry.io/product/issues/
- https://gdpr-info.eu/art-5-gdpr/
- https://gdpr-info.eu/art-17-gdpr/
