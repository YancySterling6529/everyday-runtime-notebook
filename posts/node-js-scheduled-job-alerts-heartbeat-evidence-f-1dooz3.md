# Node.js Scheduled Job Alerts: Heartbeat Evidence for Missed Notification Runs

A scheduled Node.js job can fail without running at all: the process that should start a healthtech notification sweep may never wake up. Short answer: pair a heartbeat monitoring signal with an independent expected-window ledger, then alert when the cron task misses its deadline or an executed attempt fails. An alternative to success-only healthchecks must cover both paths. Keep enough evidence to reconstruct which notifications were eligible and which were actually handed to a delivery provider.

Silence is ambiguous.

## What changed the design?

A heartbeat sent only after successful work leaves a useful gap, but it does not say which dispatch window was missed. A heartbeat at process startup has the opposite problem: it can report green before any notification leaves the queue. For incident reconstruction, neither is enough by itself. The unit of evidence should be a scheduled window with a stable ID, plus attempts attached to that window. This matters when a late retry overlaps the next sweep. Two successes must not turn into an unexplained double send. Consider an 08:00 sweep still retrying at 08:05, when the next window becomes eligible: a single global "last success" timestamp might advance while the 08:00 notification set remains unresolved. A responder needs both window IDs and the outcome of each attempt to tell delay from omission.

I would separate four times: the scheduled start, the actual attempt start, the attempt finish, and the alert deadline. The deadline is a policy decision, not an observed failure time. For example, a five-minute sweep might allow a ten-minute completion window; those numbers illustrate the calculation, not a recommended clinical delivery target. A delayed alert should say "no completion recorded for window 08:00 by 08:10," not "every patient notification failed." That distinction matters.

## How should a Node.js scheduled job alert when its heartbeat misses a run?

Create expected windows independently of the worker, using a durable store and a separate periodic checker. The checker asks which expected windows passed their deadline without a successful completion. A worker may update the same window repeatedly, but it should use a stable window key and durable attempt IDs. An alert record should carry that key, the deadline, the last attempt state, and a correlation ID; avoid putting patient names or message bodies in alert payloads. This monitoring design works for a cron-triggered task or another scheduler because it checks the obligation, not the scheduler's claim that it fired.

Here is the smallest interface I would build around a Node.js worker. The scheduler and checker can both derive the same UTC window key. The store implementation must make window creation and attempt writes durable, and the checker must run outside the worker process. This contract does not pretend an in-memory timer survives a restart.

```ts
type WindowKey = string; // e.g. 2026-09-20T08:00:00.000Z
type Outcome = { status: 'completed' } | { status: 'failed'; errorCode: string };

interface DispatchStore {
  expect(key: WindowKey, deadline: Date): Promise<void>; // idempotent insert
  begin(key: WindowKey, attemptId: string, startedAt: Date): Promise<void>;
  finish(key: WindowKey, attemptId: string, outcome: Outcome, at: Date): Promise<void>;
  overdue(now: Date): Promise<Array<{ key: WindowKey; deadline: Date; lastStatus: string | null }>>;
}

interface AlertSink {
  notify(event: { key: WindowKey; reason: 'missed' | 'failed'; detail: string }): Promise<void>;
}

async function checkWindows(store: DispatchStore, alerts: AlertSink, now: Date): Promise<void> {
  for (const window of await store.overdue(now)) {
    await alerts.notify({
      key: window.key,
      reason: window.lastStatus === 'failed' ? 'failed' : 'missed',
      detail: `No successful dispatch recorded by ${window.deadline.toISOString()}`,
    });
  }
}
```

The checker needs its own schedule and health signal; otherwise it can fail silently alongside the worker. A repeated scan must deduplicate alerts by window key and alert state in durable storage. Likewise, an attempt marked completed should mean the defined unit of work finished, not merely that a callback started. If the delivery provider accepts a request but later reports a bounce, record that as a separate delivery event. Do not rewrite the dispatch attempt's history.

No ping proves delivery.

## What evidence survives a replay?

For each window, persist the schedule version, UTC key, deadline, attempt ID, outcome, and timestamps. For each notification, retain an internal correlation ID and delivery state subject to the organization's retention and privacy rules. Replay is a separate operation: use an idempotency key scoped to the notification and delivery action so a recovery sweep does not resend a message already accepted. The exact idempotency guarantee depends on the provider and your own storage transaction boundaries; test it rather than inferring it from a successful HTTP response.

Error severity should also stay honest. RFC 5424 defines severity levels for syslog messages, but a severity label alone cannot establish that a scheduled run occurred or that a notification arrived. Capture structured outcomes first, then map them to local alert policy. A failed attempt can trigger an immediate alert; a missed window requires waiting until its deadline. Those are different detection paths with different timestamps.

Test with a stopped worker, a slow but ultimately successful worker, a failed attempt followed by a successful retry, and a checker restart. Assert which alert fires and whether the incident timeline still identifies the original window. Check deployment behavior too: a schedule change needs an explicit effective time, or old expected windows may look like new failures.

## What would I change at scale?

Partition window records by schedule and time, and move alert deduplication into an atomic state transition. Measure detection lag from deadline to alert, plus the share of windows whose completion status remains unknown. Keep the core contract small. More configuration knobs will not fix a missing event boundary.

The trade-off is one extra durable ledger and an independent checker. It costs storage and another failure mode, but it gives an incident responder an answer that a lone success ping cannot: which scheduled window was due, whether work started, how many attempts ran, and where the evidence stops.

## References

- https://datatracker.ietf.org/doc/html/rfc5424
- https://nodejs.org/api/process.html

## Sources

- https://datatracker.ietf.org/doc/html/rfc5424
- https://nodejs.org/api/process.html
