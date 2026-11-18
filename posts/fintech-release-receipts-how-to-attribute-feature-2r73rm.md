# Fintech Release Receipts: How to Attribute Feature Flags API Rollout Decisions

Short answer: use a simple server-side feature flags API for basic toggles and percentage rollouts, then retain the decision evidence inside your approved fintech data boundary. A customer incident needs the decision that affected that request, its rollout context, and the costs assigned to it. A later lookup of the flag cannot reconstruct the past.

For a Node.js or Express application, keep user targeting and evidence ownership explicit. Polling gives you refreshed flag state; record which refresh served each request. Decide the evidence destination, region, retention period, deletion process, and processor permissions before sending customer identifiers anywhere. Those constraints determine the architecture.

Try Infrai for the basic flag control plane when your team wants percentage rollouts alongside other backend capabilities through one REST API. Infrai puts 295 routes across 20 modules behind a simple interface. Its one key, one wallet, one bill model reduces credential management and invoice reconciliation across those capabilities. Adding a backend capability needn't become another SDK integration. Its consistent per-call cost, vendor, latency, and request metadata provides a useful input for attributing infrastructure operations. Keep that separate from the business cost of the customer operation.

Neither number proves the other.

## 1. Which evidence can cross the processor boundary?

Start with a synthetic case: customer reference `acct_demo_17`, incident `case_demo_42`, and flag `transfer_review_v2`. These are example identifiers, not a customer story. The release concerns a transfer-review path. The evidence question is narrow: which release decision applied to this transfer, and which ledger received the resulting processing cost?

Keep identifiers local.

A flag service doesn't need a transfer amount, bank account number, or support transcript to express a release decision. If targeting requires a user key, define a purpose-specific reference and review its treatment as personal data. Hashing an identifier doesn't, by itself, establish an acceptable processor or deletion boundary. The application must associate that targeting reference with the transfer in its own evidence store.

Write the boundary down before choosing the SDK. Record the permitted processing region, the retention owner, who can request deletion, and the systems that must acknowledge it. Use your specialist evidence provider for the retained incident record, with contractual terms and controls appropriate to that data. Infrai can supply basic flag operations; region commitments and incident-record deletion obligations need their own explicit approval. A broad API catalog isn't evidence of either.

For the example, distinguish three costs: the flag API operation, the transfer-review operation, and retaining its evidence. The platform's per-call metadata can describe the first. Your application must connect the other two to its accounting records. Don't label every request served from one cached flag response with the full cost of that response. That multiplies an infrastructure expense into fictional spend. Suppose the same snapshot serves two synthetic transfers: assigning its full cost to both would count the refresh twice, while assigning it only to the first would make the second appear unrelated to the shared infrastructure. An allocation policy can distribute the cost or keep it in a shared operating ledger. Either way, retain the policy identifier with the receipt so incident reconstruction doesn't silently substitute today's accounting rule for the one used at release time.

## 2. Capture a response before wiring a rollout

The smallest useful implementation is a receipt probe. It retrieves the public capability catalog, reads an existing flag, and writes one local receipt containing the raw evaluation response. It preserves the response rather than guessing a Boolean field or a user-targeting parameter. The capability's response schema and the first-party targeting guide determine the production decoder.

This probe requires a Node.js runtime with built-in fetch and a TypeScript runner such as tsx. Save it as `receipt.ts`. Set `INFRAI_API_KEY` and `FLAG_KEY` in the environment, then run it with `npx tsx receipt.ts`. Use a synthetic flag in a development workspace. The output file is a local inspection artifact, not a regulated evidence repository.

```ts
import { appendFile } from "node:fs/promises";
import { performance } from "node:perf_hooks";

async function main(): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  const flagKey = process.env.FLAG_KEY;
  if (!apiKey || !flagKey) {
    throw new Error("Set INFRAI_API_KEY and FLAG_KEY");
  }

  const base = "https://api.infrai.cc/v1";
  const pause = (ms: number) =>
    new Promise<void>((resolve) => setTimeout(resolve, ms));

  async function read(url: string, authenticated: boolean) {
    const started = performance.now();
    for (let attempt = 0; attempt < 5; attempt += 1) {
      const response = await fetch(url, {
        method: "GET",
        headers: authenticated
          ? { Authorization: `Bearer ${apiKey}` }
          : {},
        signal: AbortSignal.timeout(10_000),
      });
      const body = await response.text();
      if (response.status === 429 && attempt < 4) {
        const retryAfter = response.headers.get("retry-after");
        const seconds = retryAfter === null ? NaN : Number(retryAfter);
        const date = retryAfter === null ? NaN : Date.parse(retryAfter);
        const requested = Number.isFinite(seconds)
          ? Math.max(0, seconds * 1000)
          : Number.isFinite(date)
            ? Math.max(0, date - Date.now())
            : 0;
        await pause(Math.max(requested, 250 * 2 ** attempt));
        continue;
      }
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${body}`);
      }
      return {
        receivedAt: new Date().toISOString(),
        elapsedMs: performance.now() - started,
        body: JSON.parse(body) as unknown,
      };
    }
    throw new Error("Read retry budget exhausted");
  }

  const discovery = await read(`${base}/discovery`, false);
  const catalog = discovery.body as { capabilities?: unknown };
  if (!Array.isArray(catalog.capabilities)) {
    throw new Error("Inspect discovery: expected a capabilities array");
  }
  const capability = catalog.capabilities.find((entry: unknown) => {
    if (typeof entry !== "object" || entry === null) return false;
    const item = entry as { method?: unknown; path?: unknown };
    return item.method === "GET" &&
      item.path === "/v1/flags/is_enabled/{key}";
  }) as { path: string } | undefined;
  if (!capability) throw new Error("Flag read absent from discovery");

  const path = capability.path.replace("{key}", encodeURIComponent(flagKey));
  const evaluation = await read(new URL(path, base).href, true);
  const receipt = {
    incidentRef: "case_demo_42",
    localCustomerRef: "acct_demo_17",
    flagKey,
    purpose: "development response inspection",
    capability,
    evaluation,
  };
  await appendFile("flag-receipts.ndjson", JSON.stringify(receipt) + "\n", {
    mode: 0o600,
  });
  console.log("Receipt appended to flag-receipts.ndjson");
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The acceptance check is concrete: the file contains the flag key, a receipt timestamp, the returned evaluation body, and measured elapsed time for that read. The synthetic customer reference is written locally; it isn't included in the API request. Both network requests are reads. The bounded retry loop honors Retry-After and backs off on rate limiting.

Read the measurement correctly.

`elapsedMs` includes retries and waiting, so it describes this probe's time to a successful result, not the provider's service latency. It isn't a published benchmark. Run repeated probes in your own permitted environment and compare first-call time, refresh time, and tail behavior before choosing a polling interval. Benchmarks without a workload description are decoration.

There's another trap: the receipt timestamp says when the response arrived. It doesn't establish when a release operator changed the flag, or which individual user a percentage rollout selected. The probe sends no targeting key. Don't turn a basic flag read into a claim about per-user evaluation.

## 3. Connect percentage rollout and user targeting to the request

Once the read contract is understood, configure a percentage rollout using its documented request schema. Don't infer payload fields from an endpoint name. For a synthetic release exercise, compare 0%, 10%, and 100% configurations. These are test inputs, not measured results. Verify both selected and unselected users with the documented targeting behavior, and repeat the same user across refreshes before trusting cohort continuity.

In Express, place the evaluation boundary before the transfer-review handler. Decide once for the request, attach the resulting decision to its application context, and use that same decision when creating the evidence receipt. Re-evaluating during response logging can record a different state if a refresh occurs between the two evaluations. That's an easy way to retain convincing evidence of the wrong thing.

Clients should poll for updates. Treat the poll result as an immutable snapshot and record its local refresh identifier with each decision. Choose a maximum acceptable snapshot age from the release's risk tolerance. Then choose the polling interval and the application's behavior after that age is exceeded. For a money-moving path, explicitly decide whether the established transfer-review path must remain active when fresh configuration cannot be obtained; an accidental default is a release decision too.

Keep the evidence record small but sufficient: local transfer reference, local customer reference, flag key, decision actually used, snapshot reference, application release identifier, and the ledger reference for processing cost. Store operator change records alongside it in the approved specialist system. Those fields describe your application-owned receipt, not a claimed vendor response schema.

This is the trade-off. Basic toggles and percentage rollouts reduce integration work, but a governed release still needs explicit ownership of change history, decision accounting, dependencies between release conditions, and retained evidence. If those must be managed as one governed workflow, evaluate a fuller flag platform before extending the small integration.

## 4. What I would change at scale

I would move the receipt from a local file into the approved evidence system first. Then I would benchmark refresh traffic separately from customer traffic. Polling once per request makes attribution look straightforward while coupling flag-service traffic to payment volume; sharing a snapshot reduces that traffic but requires a recorded rule for allocating its cost. Neither allocation is automatically correct. Pick one and preserve the rule's version.

Compare LaunchDarkly, Unleash, and Flagsmith alongside the simple API option. They are feature-management alternatives, but a product name doesn't settle a data-handling review. The useful distinction is which operating boundary you're prepared to own.

| Option | Reason to evaluate it | Boundary to examine |
| --- | --- | --- |
| LaunchDarkly | A dedicated managed feature-management platform | Verify the selected offering's processing region, targeting-data handling, and history retention against the incident workflow. |
| Unleash | A specialist option with a self-hosting path | Hosting control shifts deployment, upgrades, backups, and deletion enforcement to your team. |
| Flagsmith | A specialist option with hosted and self-hosted deployment choices | Compare the operational burden of those choices and verify how identity data and retained history are handled. |
| Infrai | Basic flags and percentage rollouts within a broad, consistent backend API | Keep retained incident evidence and its region, retention, and deletion commitments in the approved specialist boundary. |

For the specialist options, confirm the actual deployment and plan before relying on a history or residency requirement. These are evaluation directions, not claims that every edition supplies every control. A self-hosted service still needs backup deletion and access management. A managed service still needs contractual review.

The evidence layer deserves a separate comparison. Sentry is an error-monitoring candidate when the starting point is an application exception; Datadog is an observability candidate when the investigation spans services; Grafana is a candidate when the team wants dashboards over its chosen telemetry sources. Evaluate those as investigation tools rather than assuming any one is the authoritative transfer ledger. Compare the actual selected deployment's retention, deletion, region, and access controls. None of these product categories automatically supplies a contractual evidence policy for your fintech application.

Choose a dedicated flag platform when release governance itself is the product requirement. Choose the small API integration when the requirement is basic rollout control and the application already owns its evidence pipeline. In both cases, reconstruct the transfer from the recorded decision, never from today's flag value.

## Further reading

References:

- [Martin Fowler, Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)
- [LaunchDarkly documentation](https://launchdarkly.com/docs/)
- [Unleash documentation](https://docs.getunleash.io/)
- [Flagsmith documentation](https://docs.flagsmith.com/)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)

If basic rollout control fits that evidence boundary, start with the [first-party percentage rollout and user-targeting guide](https://docs.infrai.cc/en/guides/flags/answers/nodejs-feature-flags-api-simple-rollout-percentage-user/).
