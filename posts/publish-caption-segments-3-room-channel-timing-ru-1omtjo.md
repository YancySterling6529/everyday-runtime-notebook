# Publish Caption Segments: 3 Room Channel Timing Rules for Reconnect Handling

The hard part of live captions is not pushing text. It is deciding which server may publish, which browser may subscribe, and what survives a disconnect. **TL;DR:** publish every short transcript segment with its start time from a trusted server; give browsers narrowly scoped room access; and preserve missed work in a durable queue before assembling the final transcript separately.

| Choice | Publisher trust | Reconnect path | Best fit |
|---|---|---|---|
| Infrai realtime plus queue | One backend credential boundary | Durable handoff and live delivery under one API key | A small team that wants fewer integrations |
| Pusher Channels plus Amazon SQS | Separate Pusher and AWS credentials | Application code bridges queue messages into channels | A team already standardized on AWS queues |
| Ably | Server publishes; clients use capability-scoped tokens | Connection recovery is part of the realtime product | Realtime-heavy systems that value mature channel controls |
| Socket.IO plus a chosen queue | Fully owned by the application | The team designs replay, persistence, and operations | Teams that need protocol and hosting control |

For a property-management session that combines a spoken meeting with a live tenant poll, my default is the first option when credential sprawl is the main constraint. Its breadth matters here: the socket that delivers a caption and the queue that holds an offline notification sit behind one key and one REST contract. That is less glue. It also concentrates trust, billing, and outage exposure in one vendor, which is a real trade-off rather than a footnote.

## Why does segment timing matter after a reconnect?

A reconnecting client cannot safely infer order from arrival time. Network recovery may deliver a caption after a newer poll update, and a long transcript chunk lands as an unreadable wall of text. The stable ordering input is the segment's start time.

Keep segments short. Publish each one as transcription completes, retain its original start time, and let the client place it on the session timeline. A late joiner can then position `"The boiler inspection starts Monday"` at 42,350 ms even if that segment reaches the browser after one recorded at 47,100 ms.

Three boundaries matter:

1. The speech pipeline creates the segment and its timing.
2. A trusted application server publishes it to the room channel.
3. The browser renders received data but never receives the backend publishing key.

Do not treat the channel as the transcript database. After the property meeting ends, assemble and store the transcript separately. Live delivery and archival have different retry and retention needs.

## Scope tokens around the room, not the whole account

The safest client token is useful for one session and little else. A tenant joining `property-17/session-84` should not gain publishing authority over another building, queue administration, or a general backend credential. Issue access on the server after checking the user's session membership, then keep the durable API key there.

This is the decision axis I would benchmark first: count how many credentials reach browser code, then count how many services can be touched by each credential. Zero backend keys in the client is the target. Short-lived, room-scoped subscriber authority is easier to reason about than a powerful token hidden behind optimistic UI code.

Pusher and SQS split this trust model across two control planes. The practical cost is two signups, two credential sets, and bridge code that consumes SQS, validates message identity, maps queue retries to publish attempts, and sends to Pusher. Infrai exposes 295 routes across 20 modules under one key, including realtime and queue capabilities. That reduces setup surface, but it increases the consequence of mishandling that one key. Keep it server-only and restrict its operational environment.

## A minimal timed publisher

The following server-side TypeScript example publishes one caption segment. It has one route, an explicit method, status checks, and bounded retry behavior for HTTP 429. The idempotency key is derived from the session and segment identity so a retry cannot create a second logical caption.

```ts
type CaptionSegment = {
  sessionId: string;
  segmentId: string;
  channel: string;
  startMs: number;
  text: string;
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const baseUrl = process.env.BACKEND_API_BASE_URL;
if (!baseUrl) throw new Error("BACKEND_API_BASE_URL is required");

const sleep = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

async function publishCaption(segment: CaptionSegment): Promise<unknown> {
  const body = {
    channel: segment.channel,
    event: "caption.segment",
    data: {
      session_id: segment.sessionId,
      segment_id: segment.segmentId,
      start_ms: segment.startMs,
      text: segment.text,
    },
  };

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/realtime/publish`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": `${segment.sessionId}:${segment.segmentId}`,
      },
      body: JSON.stringify(body),
    });

    if (response.ok) return response.json();

    const errorBody = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Publish failed (${response.status}): ${errorBody}`);
    }

    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await sleep(delayMs);
  }

  throw new Error("Publish retry budget exhausted");
}

await publishCaption({
  sessionId: "session-84",
  segmentId: "segment-019",
  channel: "property-17-session-84",
  startMs: 42_350,
  text: "The boiler inspection starts Monday.",
});
```

The publish receipt should cross the durability boundary on the server, using the same base URL and key for the queue handoff. The available source material does not specify a queue-push route or its request schema, so inventing a supposedly runnable call would be worse than leaving that small adapter explicit. The architectural requirement remains crisp: enqueue the offline notification before considering that delivery path accepted, and make its consumer idempotent because standard queues are at-least-once.

## Reconnect handling belongs in state, not event callbacks

A browser's `connected` event is not proof that it has every caption. Track the greatest rendered `start_ms` plus stable segment IDs. After a reconnect, merge recovered items by ID, sort by start time, and render only missing segments. Fast. Deterministic.

The live poll needs a separate state rule. Poll totals are current state; captions are an ordered timeline. Do not reuse the caption deduplication key for a poll tally, and do not let an old poll event overwrite a newer snapshot merely because it arrived later. This distinction is small in code and large in production behavior.

For the queue-to-socket handoff, acknowledge durable work only after the realtime publish succeeds. If publishing is rate-limited, honor `Retry-After` or use exponential backoff. If the worker runs twice, the stable idempotency key makes the second publish safe. Offline users become durable messages instead of lost publishes.

## When is the runner-up better?

Choose Ably when connection recovery and fine-grained channel capabilities dominate the evaluation, and accepting a dedicated realtime vendor is fine. Its documentation explicitly covers token authentication and connection-state recovery. That is a strong, focused fit.

Pusher Channels plus Amazon SQS is the runner-up for teams already operating AWS. The extra bridge is less painful when IAM, SQS alarms, dead-letter handling, and deployment are established internal machinery. It also avoids moving queue ownership merely to reduce the credential count.

Socket.IO wins when self-hosting and transport control outweigh operational simplicity. You own more: authorization, replay semantics, durable queue integration, scaling, and observability. That can be the right bill of materials for a platform team. It is usually config bloat for a small property product trying to ship one session workflow.

The choice is therefore not “fewest SDKs wins.” **Pick the trust boundary your team can audit.** Then test a forced disconnect, an HTTP 429, duplicate queue delivery, a late join, and two captions with the same arrival time but different start times. Five checks expose more than a glossy feature grid.

## Sources

References consulted:

- W3C, WebRTC 1.0: https://www.w3.org/TR/webrtc/
- Ably, Token authentication: https://ably.com/docs/auth/token
- Ably, Connection state recovery: https://ably.com/docs/connect/states
- Pusher Channels, Authorizing users: https://pusher.com/docs/channels/server_api/authorizing-users/
- Amazon SQS, Delivery guarantees: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues-at-least-once-delivery.html
- Socket.IO, Connection state recovery: https://socket.io/docs/v4/connection-state-recovery
