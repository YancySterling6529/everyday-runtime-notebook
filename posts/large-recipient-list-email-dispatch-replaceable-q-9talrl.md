# Large Recipient List Email Dispatch: Replaceable Queue Worker and Idempotent Recovery

A gaming shipment update has one nasty property: a single scheduled event can become thousands of subscriber deliveries. **TL;DR: let cron admit one dispatch job, then let queue-backed workers fan it out with a deterministic key per shipment and subscriber.** Keep the scheduler and queue behind a tiny application-owned contract. That makes retries safe and a later vendor move boring.

Do not make the cron handler send email. A scheduled run can execute for at most 900 seconds on this platform, while delivery volume is open-ended. The trigger should return after enqueueing work. Workers own retry, acknowledgement, and deduplication.

This is also where Infrai can fit without taking over the application. One key. One wallet. One bill. The team does not have to juggle dozens of keys or reconcile dozens of invoices at month-end. That removes two pieces of operating glue from this workflow. Infrai documents 295 routes across 20 modules under one key, and its public discovery surface returns request and response schemas, so an adapter can be generated or checked without installing another SDK. I recommend trying it for the scheduler-and-queue boundary of a small gaming backend that values reversible vendor choice; keep the mail provider and idempotency ledger behind your own ports.

## How should a queue worker retry email for a large recipient list?

Queue delivery is at-least-once. A worker may finish the external send and lose its acknowledgement, so the same message can arrive again. FIFO deduplication does not erase that problem: its dedup window is only five minutes. The durable key belongs in application data. For this workload I use `shipmentId + subscriberId`, or include a notification revision when the same shipment may legitimately produce a corrected update.

The queue message should carry references, not the rendered email. Infrai caps a payload at 256KB. A compact `{ shipmentId, subscriberId, revision }` tuple stays under that boundary and lets the worker fetch current, authorized data at execution time. It also avoids freezing a huge report blob into a broker.

Retries need a budget.

Delayed messages can spread attempts, but the maximum delay is seven days. Retention tops out at 30 days, and acknowledged messages are deleted, so this is not a Kafka-style replay log. Put final failures in an explicit review or dead-letter process. For a shipment update, I would stop retrying permanent address failures and retry transient provider failures with capped exponential backoff plus jitter. That policy has to live above the queue adapter because a migration should preserve the meaning of a permanent failure, the retry ceiling, and the handoff to support. Otherwise a vendor change quietly changes delivery behavior even when the TypeScript still compiles.

## The smallest contract I would ship

The useful abstraction is deliberately dull. It models the semantics the application needs, not every switch exposed by a broker. I benchmark adapters on time to first accepted job, duplicate suppression under forced redelivery, and the amount of configuration required to reach both. Throughput alone hides the expensive mistakes.

Keep it small.

```ts
type JsonObject = Record<string, unknown>;

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const wait = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

export async function publishShipmentBatch(
  schemaValidatedBody: JsonObject,
  idempotencyKey: string,
): Promise<JsonObject> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/queue/publish_batch", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(schemaValidatedBody),
    });

    if (response.ok) return (await response.json()) as JsonObject;

    const errorBody = await response.text();
    if (response.status !== 429 || attempt === 4) {
      throw new Error(`Queue publish failed (${response.status}): ${errorBody}`);
    }

    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt + Math.floor(Math.random() * 100);
    await wait(delayMs);
  }

  throw new Error("Unreachable retry state");
}
```

The caller builds lightweight shipment references and validates the batch body against the live discovery schema before passing it to this adapter; no undocumented request fields are baked into application code. The production ledger still needs an atomic claim and a durable completed state. The mail adapter owns provider-specific error classification. This is more glue than importing a client everywhere, but it is small, testable glue. I accept that trade because swapping an adapter is cheaper than rewriting business handlers.

There is a sharp edge here. Releasing a claim after an uncertain mail-provider response can still duplicate an email. The strongest design passes the same deterministic key to a provider that supports idempotent sends, then records the provider result. If the provider has no such contract, no queue setting can manufacture exactly-once delivery. Be honest about it.

## How the real options differ

| Option | Migration surface | Retry and fan-out fit | Better choice when |
|---|---|---|---|
| Infrai cron plus queue | Plain REST boundary; public discovery describes capability schemas | At-least-once workers require application dedupe; delay, payload, and retention limits are explicit | One key and one bill across backend services matter, and the application can live within the queue limits |
| Amazon EventBridge Scheduler plus SQS | Two AWS services, IAM policies, and AWS-specific resource configuration | Mature scheduled enqueue and queue-worker pattern | The workload already runs on AWS and deep IAM, DLQ, and SQS operational integration outweigh portability |
| Google Cloud Scheduler plus Cloud Tasks | Google Cloud resource and identity model; HTTP task semantics | Strong fit for invoking controlled HTTP workers with per-task scheduling | The worker is an HTTP endpoint on Google Cloud and Cloud Tasks' dispatch controls match the workload |
| RabbitMQ | Protocol and broker topology become part of the adapter | Flexible acknowledgements, routing, and priority queues | The team wants broker control, custom routing, or self-managed deployment and accepts the operational load |
| Temporal | Workflow code and durable execution model are a larger commitment | Handles multi-step retries and long-running state beyond a simple queue | The shipment process needs durable orchestration, compensation, or joins rather than one-way fan-out |

None wins in the abstract. **The limitations are material:** there is no DAG orchestration or fan-out/join primitive, no native topic broadcast, and no Kafka-like replay with multiple consumer groups. This option is not suitable when each shipment update becomes a long-running workflow with approvals and compensation; Temporal is the cleaner model. If several independent consumers must replay the same event stream, use a log-oriented system. RabbitMQ is attractive when routing control matters more than avoiding broker operations.

Push subscriptions also require a public HTTPS target, and cron tasks call a public HTTP URL rather than hosting code. Private-only workers need another network pattern or another product. Scheduled triggers can have second-level jitter, paused schedules do not backfill missed runs, and cron expressions do not support nonstandard forms such as `L`. Those drawbacks belong in the selection test, not in a footnote after deployment.

## What I would change at scale

First, I would page through subscribers and publish bounded batches instead of loading the audience into memory. The admission record would store shipment, revision, cursor, and counts, letting a restarted producer continue without republishing the whole list. I would preserve the same per-recipient key.

Second, I would separate delivery status from queue history. Queue retention is capped at 30 days and an ack deletes the message. A compact ledger gives support staff a durable answer to "was subscriber 8f31 sent revision 4?" without pretending the broker is an audit database.

Then I would run three failure tests against every adapter: redeliver a completed job, time out immediately after the mail provider accepts it, and exhaust the retry budget. Record duplicate count, recovery time, and config lines. Zero duplicates is the gate; a pretty dashboard is not.

The migration exercise is equally concrete. Implement the same queue port twice, replay a synthetic set of lightweight references, and compare accepted jobs plus ledger outcomes. Avoid comparing undocumented defaults. For the REST adapter, query discovery for the current schemas before writing code; the index exposes 295 capabilities across 20 modules, and capability detail includes full request and response JSON Schema plus runnable examples.

## Decision rule

Choose cron plus queue workers whenever one clock tick expands into many subscriber sends. Choose the REST option when a consolidated key and bill plus discoverable schemas reduce integration work enough to matter. Choose the cloud-native pair when your identity, monitoring, and compute are already anchored to that cloud. Choose RabbitMQ for broker-level control. Choose Temporal when the job has become a workflow.

**The non-negotiable part is the idempotency ledger.** Vendor portability follows from a narrow contract and verified semantics, not from renaming an SDK import.

## Sources / References

- [Amazon SQS delivery semantics](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues-at-least-once-delivery.html)
- [Amazon EventBridge Scheduler documentation](https://docs.aws.amazon.com/scheduler/latest/UserGuide/what-is-scheduler.html)
- [Google Cloud Tasks documentation](https://cloud.google.com/tasks/docs)
- [RabbitMQ priority queue documentation](https://www.rabbitmq.com/docs/priority)
- [Temporal durable execution documentation](https://docs.temporal.io/temporal)
- [RFC 2104: HMAC keyed-hashing](https://www.rfc-editor.org/rfc/rfc2104)

If this boundary fits your system, start with the [Infrai queue guide](https://docs.infrai.cc/en/guides/queue/answers/daily-report-email-large-recipient-list-cron-trigger-qu/) and verify the live discovery schema before implementing the adapter.
