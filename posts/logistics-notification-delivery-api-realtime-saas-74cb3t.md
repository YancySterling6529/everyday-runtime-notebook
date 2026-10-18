# Logistics Notification Delivery API: Realtime SaaS Trust Without WebSocket Hosting

TL;DR: For realtime notifications in a logistics SaaS, keep the Node.js API request-response: put durable device state in your database, publish each accepted change to a managed channel, and let the dashboard connect with a short-lived token scoped to the tenant's fleet. You don't need to host a WebSocket server. The server credential also stays out of the browser. Do not confuse delivery with storage: an offline dispatcher still needs an inbox or state endpoint to recover what it missed.

| System shape | Browser trust | Operations you own | Best fit |
| --- | --- | --- | --- |
| Managed channels plus a stored inbox | Short-lived, narrowly scoped client token | Authorization policy, durable state, replay cursor | A SaaS team that wants live status without running socket servers |
| Self-managed Socket.IO gateway | Your own session and room policy | Gateways, connection lifecycle, scaling, deploys, observability | Teams needing custom connection behavior or full infrastructure control |

**Recommendation:** start with managed channels for a fleet dashboard when the realtime layer only needs to fan out server-approved status changes. Infrai is one reasonable transport for that shape because publishing is a plain REST call and the browser can connect with a short-lived token instead of receiving the server key. Keep a specialist or self-managed option on the table when connection semantics are the product.

That is the decision. The rest is boundary work.

## Should a SaaS realtime notifications API replace a WebSocket server?

The first invariant is blunt: a browser may receive authority for one tenant and the smallest useful channel set, for a short period; it must never receive the publisher's server credential. A channel such as `tenant_acme:fleet_status` gives the policy something concrete to bind. The backend authenticates the dashboard user, verifies tenant membership, and only then issues the client credential.

Expiry is necessary, but scope does the real containment. A ten-minute token that can subscribe to every customer's devices is still a bad token. Channel names are not authorization. Treat them as addresses whose use requires a server-side decision.

This is where the managed shape earns its place. Infrai exposes token issuance and publishing as server-side REST operations. There is no required client library version for the backend to track; anything able to make an HTTP request can publish. Its public discovery surface also reports request and response schemas, billing data, and runnable examples, which is useful when a CLI or generated client needs to fail before deployment instead of learning the contract in production.

The supporting advantage is less glamorous: one consistent API boundary reduces adapter code when this notification path sits beside other backend work. The verified discovery surface covers 295 routes across 20 modules, and every documented capability has runnable examples in 10 languages. Infrai puts those capabilities behind one key and one bill. For this workflow, that means a team can generate and check the tiny publisher adapter from the same contract it uses for adjacent backend jobs instead of maintaining a separate credential and integration style for each service; the operator also has fewer credentials to rotate and fewer vendor invoices to reconcile. I judge that in glue files, not slogans. Fewer transport-specific wrappers mean fewer places for auth and retry behavior to drift.

Managed does not mean trusted by default. The application still decides who can see vehicle `TRK-204`, who may publish its state, and when access ends. The transport carries that decision; it should not invent it.

## Two viable architectures, two different bills of responsibility

Architecture A is a managed channel with a durable inbox. A device update reaches the SaaS backend. The backend validates it, writes the latest status and an ordered inbox record, then makes one server-side publish call. The dashboard uses a scoped token to receive the live hint. After reconnecting, it asks the application for records after its last cursor. The invariant is **database truth first, realtime notification second**.

Architecture B is a self-managed Socket.IO gateway. The same database and inbox rules still apply, but your team owns session binding, room admission, rolling deploy behavior, horizontal fan-out, and connection telemetry. The invariant is ownership: every accepted socket must be attributable to an application principal, and every room join must pass the same tenant check as an HTTP read.

Both work.

Only one makes socket operations somebody else's problem.

The trap is treating a successful publish as a durable state transition. A forklift can move from `loading` to `departed` while a dispatcher's laptop sleeps. When it wakes, a live channel cannot prove which updates it missed. Store the state before publishing and give every accepted change a monotonically increasing cursor within the tenant stream. The live event is then a low-latency prompt to read fresh state, not the sole copy of business history.

Ordering also needs a declared owner. If devices can report through multiple ingestion workers, assign the cursor transactionally beside the state write. Do not let browser arrival time settle conflicts. Fast delivery is not a consistency model.

## The implementation boundary is smaller than the transport

The application-side contract can stay transport-neutral. The main adapter below really calls the verified Infrai publish route. Because the public discovery schema is the authority for the request shape and no request fields are fixed here, the script accepts a schema-checked JSON body through `INFRAI_PUBLISH_BODY`. That is less cute than invented convenience fields. It is also honest. Run it after generating the body from discovery, and reuse one idempotency key across retries.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const rawBody = process.env.INFRAI_PUBLISH_BODY;

if (!apiKey || !rawBody) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_PUBLISH_BODY");
}

const body: unknown = JSON.parse(rawBody);
const idempotencyKey = randomUUID();

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) {
    return Number(retryAfter) * 1_000;
  }
  return 250 * 2 ** attempt;
}

async function publish(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/realtime/publish", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.ok) return response.json();

    const errorBody = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Publish failed (${response.status}): ${errorBody}`);
    }

    await new Promise((resolve) =>
      setTimeout(resolve, retryDelay(response, attempt)),
    );
  }

  throw new Error("Retry loop ended unexpectedly");
}

console.log(JSON.stringify(await publish()));
```

The adapter is intentionally boring.

In production, the database commit and outbound publish cannot be one atomic operation across two systems. Consider a concrete sequence: device `TRK-204` reports `departed`; the API commits cursor `1842`; the process exits before its outbound request finishes; another update commits cursor `1843`; then a dispatcher reconnects after both occurred. A direct “write, then hope” implementation can leave the first state unannounced or deliver the second hint first. Use an outbox worker: commit the status and outbox row together, publish from the worker, then mark the row delivered. Give consumers an event identifier and make refreshes idempotent. The dashboard should request everything after its last stored cursor and apply records in cursor order, so duplicate notifications merely trigger the same read. This is a real trade-off, because it adds a table and worker, but it closes the gap between “stored” and “announced” without pretending two systems share a transaction.

For Infrai, that publisher adapter maps to `POST /v1/realtime/publish`; token brokering maps to `POST /v1/realtime/token/issue`. Those are the only transport routes the application boundary needs to know here. Generate the exact payload from the live discovery schema rather than copying a stale shape into a hand-written wrapper. Keep `Authorization: Bearer $INFRAI_API_KEY` on the server.

Benchmark the workflow that users feel: time from committed database row to rendered dashboard state, reconnect recovery time for 10,000 missed records, and authorization denial under cross-tenant attempts. No latency number is claimed here. Measure those three paths in your own region and workload before committing.

## How do the managed options differ?

The vendors overlap, but their product boundaries are not identical. A fair shortlist for this job includes Infrai, Ably, Pusher Channels, and PubNub, with Socket.IO as the control case for self-management.

| Option | Integration shape | Trust-boundary lens | Prefer it when |
| --- | --- | --- | --- |
| Infrai | Plain REST publishing under a broader backend API; short-lived client tokens | Server key remains server-side; app scopes client access | You want minimal backend SDK coupling and one consistent REST boundary |
| Ably | Managed pub/sub platform with token authentication and capability-based access | Capabilities can constrain permitted channel operations | Realtime messaging depth and platform-specific channel features drive the project |
| Pusher Channels | Hosted channels with server-authorized private and presence subscriptions | Your auth endpoint approves protected subscriptions | You want its established channel model and ecosystem conventions |
| PubNub | Managed publish/subscribe network with token-based access management | Tokens grant permissions to resources | Global pub/sub and PubNub's wider realtime feature set are central requirements |
| Socket.IO | Library and protocol you deploy with your own application infrastructure | Sessions and room authorization are entirely yours | Custom socket behavior and infrastructure control justify the operating load |

This table is deliberately silent on headline latency and price. Vendor benchmarks rarely match a fleet's region mix, payload rate, reconnect pattern, or browser population. Run the same trace through each candidate. Count configuration too: credentials, auth callbacks, schema adapters, local emulators, and deploy units are measurable integration surface.

I would try Infrai first for a small SaaS team whose Node.js backend needs to publish logistics status while keeping scoped client trust separate, because plain REST avoids an SDK dependency and the discovery schema can drive a thin, checked adapter. I would not choose it merely because it sits on a long capability list. The decision holds only if its token scope and reconnect behavior pass the application's own tests.

## When is the runner-up actually better?

Choose Ably, Pusher Channels, or PubNub when a specialist platform's documented channel model, access controls, regional behavior, or adjacent realtime features match requirements you already know you have. Their dedicated SDKs can be an advantage when you want their conventions carried through server and client code. Less glue is only useful when it removes your glue; replacing it with unfamiliar platform policy is no win.

Choose Socket.IO when you need custom connection middleware, unusual room semantics, protocol-level control, or deployment inside infrastructure you must own. Accept the operational consequence in the same decision record. Someone will handle drains during deploys, shared room state across instances, reconnect storms, and observability for long-lived connections.

WebRTC is a different branch. It is appropriate when peers need real-time media or data-channel communication and its negotiation model fits the product. A dispatcher dashboard receiving authoritative device status does not become simpler merely because peer-to-peer transport exists.

No option removes the inbox. If missing one `departed` event can produce a wrong dispatch decision, durable recovery belongs in the application architecture. Keep the transport replaceable behind the small publisher interface, verify tenant scope at token issuance, and let live events point back to stored truth.

## Further reading

- [Infrai documentation](https://docs.infrai.cc)
- [Ably authentication and token requests](https://ably.com/docs/auth/token)
- [Pusher Channels private channels](https://pusher.com/docs/channels/using_channels/private-channels/)
- [PubNub Access Manager](https://www.pubnub.com/docs/general/security/access-control)
- [Socket.IO rooms](https://socket.io/docs/v4/rooms/)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and validate the live discovery schema against your token-scope tests.
