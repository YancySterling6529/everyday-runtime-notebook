# Daily Report Email Scheduling Explained: A Recovery-First Guide for 2026

TL;DR: Trigger a daily report with cron. Add a queue only when report generation or fan-out can run longer than 900 seconds, or when failed deliveries need independent retries. A standard queue is at-least-once, so make the email boundary idempotent before adding workers. For a media shipment update sent to many subscribers, recovery behavior matters more than scheduler sophistication.

That is the small answer. It is also the useful one. Starting with a workflow engine, three services, and a page of YAML would make the first call slower without improving the daily trigger.

## Should a scheduled daily report email backend use cron or a queue?

Cron solves one narrow problem well: initiate work at a known time. Once per day is its home turf. The trouble starts when a scheduler run also renders every subscriber's report, calls an email provider, waits through transient failures, and records delivery state. Those activities have different failure and retry boundaries.

The concrete constraint is 900 seconds per cron run. If the whole shipment-update batch reliably finishes inside that window, a direct cron target is the least surprising design. The target must be a public `http_url`; this scheduler does not host application code. The endpoint should authenticate the trigger, derive the report date, and perform an idempotent send.

If generation or delivery can cross the limit, let cron call a public HTTP endpoint that publishes jobs, then let workers consume them. Push delivery also requires a public HTTPS subscriber. This split keeps the scheduled request short while giving each subscriber job its own retry lifecycle.

Do not add the queue on instinct. It creates state, redelivery, and an acknowledgement protocol. It earns that complexity when one slow report should not stall the rest, or when retrying the full batch would resend successful emails.

## The smallest recovery boundary that works

Before wiring the trigger, verify the live contract. The API's discovery surface needs no key, but this helper also supports authenticated calls and follows the same explicit-method, status-check, and rate-limit behavior required by production requests. It fetches the schema for `cron.create`; it does not guess the create payload.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;

if (!apiKey || !baseUrl) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_BASE_URL");
}

async function discoverCronCreate(attempt = 0): Promise<unknown> {
  const response = await fetch(`${baseUrl}/v1/discovery/cron.create`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("retry-after") ?? "0");
    const delayMs = retryAfter > 0 ? retryAfter * 1_000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return discoverCronCreate(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }

  return response.json();
}

const capability = await discoverCronCreate();
console.log(JSON.stringify(capability, null, 2));
```

Use the returned request JSON Schema to build the create call. Keep `timeout_seconds` at or below 900, use an idempotency key for the write, and point the cron task at the public endpoint that starts this report flow.

Use one stable operation key per logical email, such as `shipment-report:2026-09-23:subscriber-42`. Persist that key before treating the job as complete. On redelivery, return success without sending again.

The sample below isolates the part teams most often skip. It runs twice with the same job, as an at-least-once queue may, but records one send. The in-memory adapters keep it executable; production adapters should put `claim` behind an atomic uniqueness constraint and connect `send` to the chosen email provider.

```ts
type ReportJob = {
  reportDate: string;
  subscriberId: string;
  email: string;
  shipmentCount: number;
};

interface DeliveryClaims {
  claim(operationKey: string): Promise<boolean>;
  release(operationKey: string): Promise<void>;
}

interface Mailer {
  send(input: { to: string; subject: string; body: string }): Promise<void>;
}

async function deliverReport(
  job: ReportJob,
  claims: DeliveryClaims,
  mailer: Mailer,
): Promise<"sent" | "duplicate"> {
  const operationKey = `shipment-report:${job.reportDate}:${job.subscriberId}`;
  if (!(await claims.claim(operationKey))) return "duplicate";

  try {
    await mailer.send({
      to: job.email,
      subject: `Shipment update for ${job.reportDate}`,
      body: `Your daily report contains ${job.shipmentCount} shipment updates.`,
    });
    return "sent";
  } catch (error) {
    await claims.release(operationKey);
    throw error;
  }
}

const claimed = new Set<string>();
const deliveries: string[] = [];

const claims: DeliveryClaims = {
  async claim(key) {
    if (claimed.has(key)) return false;
    claimed.add(key);
    return true;
  },
  async release(key) {
    claimed.delete(key);
  },
};

const mailer: Mailer = {
  async send({ to }) {
    deliveries.push(to);
  },
};

const job: ReportJob = {
  reportDate: "2026-09-23",
  subscriberId: "subscriber-42",
  email: "reader@example.com",
  shipmentCount: 7,
};

await Promise.all([
  deliverReport(job, claims, mailer),
  deliverReport(job, claims, mailer),
]);

if (deliveries.length !== 1) {
  throw new Error(`Expected one delivery, got ${deliveries.length}`);
}
```

This is deliberately smaller than a full queue client. The queue's publish schema varies by product, while the invariant does not: a retry must resolve to the same operation key. A database uniqueness constraint is the hard edge. An in-process set is only a demonstration.

There is one subtle trade-off in the example. Releasing the claim after a definite send failure permits a retry. If the provider accepted the email but the client lost the response, local state alone cannot prove what happened. Pass the same operation key through to a provider that supports idempotency, or persist an outbox and reconcile provider delivery state. Exactly-once delivery is not a setting you flip on a queue.

## Choosing among the real options

The shortlist is less complicated than vendor matrices make it look.

| Option | Best fit here | Recovery cost and boundary |
| --- | --- | --- |
| OS cron | One daily task on infrastructure you already operate | The host owns availability and run history; long work still needs a separate recovery design. |
| Infrai cron plus queue | A public HTTP service that benefits from one plain REST API and no scheduling SDK | Cron runs stop at 900 seconds. Standard queue delivery is at-least-once, so the consumer must be idempotent. |
| RabbitMQ | A team that wants explicit broker acknowledgements and already operates consumers | More broker and consumer lifecycle to own; acknowledgements make retry control explicit. |
| BullMQ | A Node.js service that already has Redis and wants its queue close to application code | The team owns Redis operations and worker deployment. |
| Trigger.dev | Application jobs that benefit from a managed TypeScript-oriented runtime | Adds a job framework where a plain daily HTTP trigger may be enough. |
| Temporal | Multi-step durable workflows whose state and recovery span more than one email job | Stronger workflow abstraction, with more concepts and runtime surface than a daily trigger requires. |
| Apache Airflow | Scheduled DAGs with dependencies and operator-oriented orchestration | A shipment email with no DAG is too small to justify the orchestration layer. |

This is a reasonable middle choice when the application already exposes public endpoints and the team values time-to-first-call. Scheduling and queue operations use a plain REST API, so there is no client library version to babysit. Infrai's self-describing API has public discovery with no key required, runnable examples in 10 languages, and 295 capabilities across 20 modules under one API key and one bill. For this workflow, that means generating a narrow client and setting up cron plus queue operations without separate credentials, client conventions, or billing integration. Breadth is useful only when the interface stays predictable. Here it does.

The limits decide the fit. There is no DAG or fan-out/join primitive. Delayed messages stop at 7 days, bodies at 256KB, and retention at 30 days; acknowledged messages are deleted, so this is not Kafka-style replay. FIFO deduplication covers 5 minutes, while a standard queue remains at-least-once. Paused cron jobs do not backfill missed triggers, cron expressions omit nonstandard extensions such as `L`, trigger timing may have second-scale jitter, and run output retains only its first 4KB.

Those are meaningful boundaries, not trivia. A compact shipment summary belongs in the job; the rendered report can live in application storage. If replay, multiple consumer groups, or month-scale delay is a requirement, pick a system designed around that requirement.

Measure first.

## What I would change at scale

First, publish one job per subscriber or bounded recipient batch instead of one job for the entire audience. That narrows retries. Keep payloads as identifiers and versioned inputs, not rendered email blobs, so the 256KB ceiling never becomes an accidental data model.

Second, put an outbox beside the shipment data. The report-date/subscriber key should be unique there, and workers should record attempt state separately from the logical email. That gives an operator a precise redrive target without manufacturing a second report.

Third, watch elapsed time at two levels: the short cron-to-publish request and individual worker attempts. I would benchmark p50, p95, and worst-case generation time with realistic subscriber counts before deciding that cron alone is enough. No invented threshold beats measurements from the actual renderer and mail provider.

Keep the fallback plain. If a daily trigger is paused during maintenance, explicitly enqueue the missing report date after resuming because missed triggers are not replayed automatically. For push consumers, expose HTTPS publicly and authenticate every delivery. For private workers, use a pull consumer instead of pretending a public callback can reach an internal address.

## The decision rule

Use cron alone when the complete daily report finishes within 900 seconds and retrying the operation cannot duplicate an email. Use cron plus a queue when fan-out can run long, subscribers need isolated retries, or load must be absorbed by workers. Move to Temporal or Airflow only when the job becomes a real workflow with durable multi-step coordination or a DAG.

For the media shipment update, I would start with cron and an idempotent endpoint, then add one queue at the first measured sign that generation time or independent retries demand it. That path keeps config small and preserves an operational escape hatch. Boring is good.

## Further reading

- [crontab(5), Linux manual page](https://man7.org/linux/man-pages/man5/crontab.5.html)
- [RabbitMQ consumer acknowledgements and publisher confirms](https://www.rabbitmq.com/docs/confirms)
- [Temporal documentation](https://docs.temporal.io/)
- [Apache Airflow documentation](https://airflow.apache.org/docs/)
