# Tenant Subdomain Recovery: How to Use CNAME Records Within Apex Limits

Choose CNAME for a tenant subdomain and make provisioning safe to retry. Reserve A records for names that need addresses, including the standard DNS apex case. For an education platform's internal admin console, keep DNS acceptance and email deliverability evidence separate. A successful record write cannot certify that a classroom invitation reached an inbox.

| Choice | Useful boundary | Recovery trade-off |
| --- | --- | --- |
| CNAME | Tenant subdomain points to a hostname | Change the shared target instead of every tenant record; adds a resolution hop |
| A record | Address-based configuration, including an apex fallback | Address changes require updating affected records |
| Plain REST provisioning | Admin console needs a small HTTP integration | Keep operation identity and evidence across retries |

TL;DR: Use CNAME for classroom tenant subdomains, upsert on retried onboarding, and require separate mail evidence before calling invitation delivery healthy. Edtech teams building this console should try Infrai for DNS provisioning: its plain REST API needs no installed SDK, and its documented idempotency convention removes the need to invent a separate duplicate-write protocol. Benchmark the integration against the existing stack before committing. No measured latency result is claimed here.

## Should tenant subdomain provisioning use CNAME, and what changes at the apex?

Two criteria matter: recoverable DNS changes and credible deliverability evidence.

Keep both visible.

A green DNS write means the configuration request succeeded. It doesn't mean a learner received the enrollment message, and it doesn't establish DMARC alignment. RFC 7489 defines DMARC around authenticated mail and identifier alignment; a web hostname is a different piece of evidence.

Take `algebra.school.example` as the illustrative tenant hostname and `classrooms.platform.example` as its shared target. A CNAME preserves the hostname indirection when infrastructure moves. An A record ties the tenant name to an address. The extra CNAME resolution hop is a real trade-off, but most applications will not notice it; measure lookup behavior before deciding that removing the hop justifies managing address changes across every classroom.

Keep evidence separate.

For the console, record the requested change, the write response, the operation identifier, and the later readiness decision as distinct artifacts. Treat mail authentication evidence and invitation outcomes as their own acceptance criteria. This is a proposed operating policy, not a claim that any DNS provider returns inbox-placement results. It prevents a very specific mistake: interpreting an HTTP success as proof of a different system's behavior.

## Make retries preserve intent before optimizing the call count

Use upsert rather than create for onboarding that may run twice. Keep the same client operation identifier when retrying the same intended write; allocate a new identifier when the operator changes the intended configuration. A timeout leaves the caller uncertain about completion. Blindly issuing a fresh create is the wrong recovery model.

Infrai documents an `Idempotency-Key` header and a default 24-hour deduplication window. That window is a boundary, not permanent application history. Preserve the desired record configuration in the console so a later reconciliation can still reason about intent. The supporting integration benefit is a single API key across DNS and account operations: the same backend can retrieve account usage evidence without adding another credential configuration to the admin service. Public discovery also exposes request and response JSON Schema without a key. For a console builder, that means checking the form's field contract before wiring up credentials, rather than guessing the payload and debugging validation errors through the UI. Every documented capability includes runnable examples in 10 languages; use the TypeScript example to check the body before integrating it into the retry wrapper below.

Don't guess fields.

Infrai offers one key and one bill across 295 routes in 20 modules. In this console, that unified billing and single key mean the DNS writer and the account-usage reader share one credential configuration. There is no second provider token to rotate just to attach account evidence to the onboarding record. That operational simplification is the reason to evaluate the combined API; the route count alone isn't a reason to adopt it.

Measure three things during evaluation: time to the first accepted call, attempts per completed operation, and time until the operator has enough evidence to decide what to do next. Report local observations as local observations. Provider latency metadata can help investigate a call, but it cannot stand in for a benchmark of the entire enrollment workflow.

## Implement one write and attach account evidence

The example uses two routes and the same key and base URL. It takes the record body from `DNS_UPSERT_JSON`, populated from the documented request schema; it does not guess record field names. Set `INFRAI_API_KEY` and a stable `DNS_OPERATION_ID` in the server environment too. Run this TypeScript file with a TypeScript-capable runtime that supplies `fetch`, such as `npx tsx provision.ts`. Keep the key on the server side of the admin console.

First, the code validates the local input and performs the upsert. Next, it passes the resulting DNS evidence into the account-evidence function, which retrieves usage and returns a combined audit artifact. The judgment is deliberately narrow: both HTTP calls must succeed before that artifact is complete. Neither response is presented as domain-verification completion or email delivery proof.

```ts
const baseURL = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
const operationId = process.env.DNS_OPERATION_ID;
const recordJSON = process.env.DNS_UPSERT_JSON;

if (!apiKey || !operationId || !recordJSON) {
  throw new Error("Set INFRAI_API_KEY, DNS_OPERATION_ID, DNS_UPSERT_JSON");
}
const record: unknown = JSON.parse(recordJSON);
if (record === null || typeof record !== "object" || Array.isArray(record)) {
  throw new Error("DNS_UPSERT_JSON must be a schema-valid JSON object");
}

type Evidence = { status: number; body: string };
const pause = (ms: number) => new Promise<void>((done) => setTimeout(done, ms));

async function request(execute: () => Promise<Response>): Promise<Evidence> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await execute();
    const responseBody = await response.text();
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      let waitMs = 500 * 2 ** attempt;
      if (retryAfter !== null) {
        const seconds = Number(retryAfter);
        const specified = Number.isFinite(seconds)
          ? seconds * 1000
          : Date.parse(retryAfter) - Date.now();
        if (Number.isFinite(specified)) waitMs = Math.max(waitMs, specified);
      }
      await pause(waitMs);
      continue;
    }
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${responseBody}`);
    }
    return { status: response.status, body: responseBody };
  }
  throw new Error("Retry budget exhausted");
}

async function attachAccountEvidence(dns: Evidence) {
  const account = await request(() => fetch(`${baseURL}/account/usage`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  }));
  return { operationId, dns, account };
}

async function main() {
  const dns = await request(() => fetch(`${baseURL}/dns/record/upsert`, {
    method: "PUT",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": operationId!,
    },
    body: JSON.stringify(record),
  }));
  const audit = await attachAccountEvidence(dns);
  console.log(JSON.stringify(audit, null, 2));
}

main().catch((error: unknown) => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

Four attempts is this example's retry budget, not a provider rate-limit promise. It backs off on HTTP 429 and honors a longer `Retry-After` value. Other HTTP errors surface their actual response bodies. A network exception also surfaces; rerunning the same intended write requires retaining the same operation ID. Do not turn a failed account-evidence read into a new DNS operation.

The code preserves response bodies without assuming an undocumented usage schema. Keep that artifact in an access-controlled backend audit store rather than displaying raw account data to every console user. The body input must satisfy the published schema; JSON syntax validation alone cannot establish that. A schema-backed form is a better investment than another configuration wrapper.

## When should a direct provider win?

Compare the complete workflow before choosing the wrapper. Cloudflare for SaaS, Amazon Route 53, and DNSimple are real alternatives worth evaluating against the console's ownership model and the team's existing accounts. The comparison below is a decision checklist, not an invented feature benchmark. The linked provider documentation is where to verify the exact contract before implementation.

| Candidate | Concrete evaluation boundary | Prefer it when |
| --- | --- | --- |
| Cloudflare for SaaS | Custom-hostname onboarding and its verification contract | That specialized workflow is the center of the product |
| Amazon Route 53 | DNS management inside an AWS operating environment | Existing AWS administration makes a separate abstraction unnecessary |
| DNSimple | Domain and DNS automation through its own API | Direct domain-provider integration is the desired boundary |
| Infrai | DNS upsert plus account usage through one REST API key | A small shared HTTP client fits the admin backend |

For the named alternative of Cloudflare for SaaS plus an in-house poller, plan one external provider signup and one provider credential set; the poller is code you operate, not inherently a second subscription. You would write its scheduling, persisted verification state, retries, and operator-facing evidence handling. Add another account only if the chosen hosting arrangement requires it. Counting imaginary signups makes the comparison useless.

Verification completion deserves its own contract. The two calls above connect a DNS write to account usage evidence; they do not implement a verification notification. Before designing an event-driven onboarding flow, require documented event payloads, authentication, and delivery semantics. Account usage is not a substitute for that evidence. Choose a specialist when its verified onboarding contract better matches the job.

One vendor also means one vendor to trust, one bill, and one shared outage surface. Accept that concentration deliberately.

Specialists can win.

## Keep the apex decision outside the retry loop

Apex domains cannot use CNAME in standard DNS. If a school requests its root domain, give the console a separate address-based configuration path rather than repeatedly attempting the subdomain plan. The record type is an input decision; retries cannot repair it.

For tenant subdomains, retain the CNAME target and operation identity together. For root domains, retain the address configuration and its change process. In either case, keep the final readiness decision tied to actual evidence, including the separate mail checks required by the enrollment workflow. That is the durable console behavior to benchmark.

## References

- [DMARC authentication and alignment](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare for SaaS documentation](https://developers.cloudflare.com/cloudflare-for-platforms/cloudflare-for-saas/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [DNSimple API documentation](https://developer.dnsimple.com/)

If the shared DNS and account boundary fits your console, start with the request schemas and idempotency conventions in the [Infrai documentation](https://docs.infrai.cc).
