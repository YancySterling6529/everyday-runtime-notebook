# Marketplace Internal Endpoints: DNS Versus Service Registries for Deliverability Evidence

Use DNS for the public identity that receiving mail systems must inspect. Use a service registry for fast-changing internal destinations. The handoff between them should be explicit: a stable domain names an internal mail capability, while the registry selects the currently healthy instance.

| Concern | DNS owns it | Registry owns it | Evidence to retain |
| --- | --- | --- | --- |
| Public sender policy | Yes | No | Published policy and lookup result |
| Internal instance selection | No | Yes | Registration, health, and selection events |
| Release rollback | Stable name only | Active instance set | Before-and-after route snapshot |
| Deliverability review | Domain-level evidence | Supporting deploy evidence | Correlated timestamps |

TL;DR: for a marketplace moving zones away from a registrar-specific API, do not make deploy frequency decide where sender policy lives. DMARC is domain based, and its policy discovery and reporting model belongs at the DNS boundary. Keep release churn behind that boundary. The recommendation is a split design because it gives a reviewer evidence at the same layer where receivers evaluate the domain, without forcing every instance change through public naming.

## Should DNS or a service registry resolve internal endpoints?

A registry can answer a useful internal question: which process should receive this request now? It cannot, by itself, publish a domain policy to an external receiving system. That distinction matters more than lookup speed in this marketplace migration.

DMARC is explicitly domain based. RFC 7489 defines policy discovery through DNS, alignment involving the visible From domain, and aggregate and failure reporting mechanisms. A private registry entry may explain why an internal message service handled traffic at 14:03, but a receiver evaluating the sender does not consult that private control plane. Treating those records as equivalent evidence creates a gap precisely where the deliverability investigation starts.

This is the first criterion: **put evidence where its consumer can retrieve it**. For sender policy, that is the domain boundary. For an internal router choosing an instance, it is the registry.

The split also keeps two clocks from being confused. A domain policy changes when the organization's sending policy changes. An instance set can change with every release, scale event, or health transition. Coupling the two means routine deploy churn touches externally observed state even when the sender identity and policy have not changed. That buys configuration. It does not buy proof.

The split has limitations.

It creates two control planes, two permission models, and two sets of observations to correlate. That trade-off is not suitable for a small fleet whose internal endpoints rarely change: DNS alone is easier to operate when measured resolution behavior already fits the deploy window. A service registry also cannot rescue weak domain evidence, because its private view is unavailable to the external system evaluating sender policy. The split earns its extra configuration only when rapid internal changes would otherwise keep pushing instance state across the public DNS boundary.

Count the glue.

## Evidence survives the migration

The second criterion is provenance. Registrar APIs tend to encourage code shaped around a provider's zone object, record identifier, and update operation. A portable migration model should instead describe the desired domain state and record what was observed after publication. The adapter is disposable. The evidence is not.

I would benchmark the migration on time-to-verifiable-state, not time-to-API-response. Those are different measurements. An accepted write says the control plane received an instruction; a later lookup supplies evidence about what can be retrieved. No invented sleep interval fixes that distinction. Poll, bound the attempt, record each observation, and fail the promotion if the expected policy is not observable within the team's declared window.

For DMARC, retain the exact queried name, the expected value, each observed value, timestamps, and the release identifier that requested the change. Also retain reports separately from deployment logs. RFC 7489's aggregate reporting exists to provide feedback about authentication results; it should not be flattened into a binary deploy flag.

Three states are enough for the gate: proposed, observed, and accepted.

Proposed is the desired record set in version control. Observed is what the verifier retrieved. Accepted means a reviewer or policy check found the observation suitable for promotion. This small state machine exposes a common trap: treating a successful adapter call as completion.

## A narrow TypeScript boundary

The implementation does not need a sprawling provider SDK in business logic. Two small ports keep zone publication separate from runtime discovery, and an evidence record joins their outputs without pretending they share semantics.

```ts
type DnsObservation = {
  name: string;
  expected: string;
  observed: readonly string[];
  checkedAt: string;
};

type RouteObservation = {
  service: string;
  instances: readonly string[];
  checkedAt: string;
};

interface ZonePublisher {
  publishTxt(name: string, value: string): Promise<void>;
  readTxt(name: string): Promise<readonly string[]>;
}

interface ServiceDirectory {
  resolveHealthy(service: string): Promise<readonly string[]>;
}

async function captureReleaseEvidence(
  zones: ZonePublisher,
  directory: ServiceDirectory,
  policyName: string,
  expectedPolicy: string,
  service: string,
): Promise<{ dns: DnsObservation; route: RouteObservation }> {
  const [observed, instances] = await Promise.all([
    zones.readTxt(policyName),
    directory.resolveHealthy(service),
  ]);
  const checkedAt = new Date().toISOString();

  return {
    dns: { name: policyName, expected: expectedPolicy, observed, checkedAt },
    route: { service, instances, checkedAt },
  };
}
```

The example deliberately does not encode a registrar response type. It also does not claim that `publishTxt` made a value observable. Publication and observation remain separate operations. That bit of friction is useful. It prevents a green deploy from being manufactured out of the write path alone.

Keep the adapter thin: translate desired records into the external API, translate reads into plain values, and preserve the raw response in restricted audit storage when policy requires it. Retry policy belongs around operations classified as retryable by that adapter, not around every exception. A malformed desired record will not improve on attempt five.

The registry side deserves the same restraint. Store opaque instance identifiers or addresses behind a service name; do not leak its lease model, health payload, or client library across the application. Less glue makes the first call faster. More importantly, it makes the second migration boring.

## When is the runner-up enough?

DNS alone is the runner-up for this design. It can be enough when internal destinations change slowly, health-based selection is unnecessary, and the team is willing to make its deployment process wait for its own observation gate. That is a legitimate simpler system. One control plane, fewer credentials, fewer failure surfaces.

Choose it consciously. Measure the interval from requested change to observed answer in the environments that matter, then set rollout and rollback windows from that evidence. Do not borrow a generic propagation number and call it a guarantee. If the measured behavior fits the release cadence, a registry adds machinery without improving the decision.

A registry-only design fits a much narrower case: names and consumers are entirely inside one trust boundary, and no external protocol expects to retrieve policy through DNS. That exception does not cover marketplace sender identity. DMARC's consumer is outside the private service directory.

There is another runner-up: put instance addresses directly behind the stable internal DNS name while keeping public sender policy in DNS. This retains the clean policy boundary and removes the registry, but it gives up registry-specific health and registration semantics. It is attractive for a small, slow-moving fleet. Once instance turnover makes observation and rollback dominate release time, the explicit directory earns its keep.

## Ship the boundary, then test the proof

A migration rehearsal should exercise failure order, not just the happy path. First publish the desired domain state through the new zone adapter. Observe it independently. Then switch the internal resolver adapter, send controlled traffic through the stable capability name, and preserve the route snapshot beside the DNS observation. Rollback reverses the internal selection before it removes evidence needed to explain the attempt.

The dashboard should separate four signals: publication attempts, independent DNS observations, registry membership changes, and DMARC report ingestion. Correlation needs timestamps and a release identifier, but the signals should not collapse into one health percentage. They answer different questions. A policy can be observable while an internal target is unhealthy; a target can be healthy while the published policy is wrong.

Be strict here.

A useful acceptance test asks whether another engineer can reconstruct the sender policy and chosen internal destination for a release without opening the registrar console. If they can, the migration has escaped the registrar-specific API. If they cannot, the abstraction moved code while leaving operational ownership behind.

The final decision is deliberately unglamorous: DNS carries the externally inspected identity and policy; the registry absorbs rapid internal churn; evidence joins them at release time. This arrangement costs one explicit boundary and repays it with explainable deploys. For a marketplace where deliverability is the deciding axis, explainability is the feature to benchmark.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
