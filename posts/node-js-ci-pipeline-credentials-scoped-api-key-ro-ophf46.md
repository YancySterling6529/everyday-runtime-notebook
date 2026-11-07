# Node.js CI Pipeline Credentials: Scoped API Key Rotation With Billing Attribution

| Choice | Evaluate this boundary | Best fit for the marketplace |
| --- | --- | --- |
| Separate scoped service keys | Build capabilities versus production capabilities | Default when CI needs only one or two operations. |
| AWS IAM | Workload identity and policy within an AWS deployment | First alternative to evaluate when the billing worker already lives there. |
| HashiCorp Vault | Secret delivery and lifecycle across consumers | A candidate when several workloads need a shared credential-management process. |
| Unkey | API-key management as its own integration boundary | A candidate when issuing and managing consumer keys is the actual product requirement. |
| Kong Gateway | Access enforcement at a gateway | A candidate when that gateway already owns the request boundary. |

Choose a dedicated, narrowly scoped CI key before touching the marketplace's production credential. **TL;DR: Keep build access separate, then rotate production through a verified handoff that preserves billing attribution.** A replacement key should change authorization material without changing which marketplace consumer owns a charge. Treat a printed build secret as an expected rotation trigger.

The recommendation has two acceptance criteria: an exposed CI credential cannot reach production data, and requests made during the production handoff still reconcile to the correct billing consumer. Neither follows from a green build. For a Node.js worker deployed through GitHub Actions, the interesting evidence comes from the running worker and the accounting record, not from a successful secret update.

## What should a scoped API key let the CI pipeline do?

Only the capabilities the build exercises. A pipeline that needs one or two capabilities has no reason to inherit the main key's entire authority. Start narrow. Review a specific denied operation before widening access, and tighten scopes later when a build step disappears.

Names matter here. Use consumer names such as `marketplace-ci-build` and `marketplace-prod-billing`, rather than `temporary` and `new-key`. These are illustrative labels. They make the retirement decision readable: an operator needs to identify the consumer before deciding which credential can go away. An unnamed key becomes hard to revoke in practice because nobody can confidently say what will stop working.

Assume logs leak.

That assumption changes the acceptance test. Exercise the build's allowed operation, then attempt an operation outside its approved scope in a controlled environment and require denial. Keep production data out of that test. Log masking is useful hygiene, but the scope boundary has to stand even if a secret appears in output. Also keep account-management authority out of the build unless the build's reviewed job actually requires it; ordinary build access should not quietly become credential-administration access.

GitHub Actions is the workflow context, not proof of an external service key's permissions. Review those permissions at the external service boundary. An environment variable can carry a narrow credential or a dangerously broad one with exactly the same amount of YAML.

## Billing attribution must outlive the secret

Separate three identifiers in the application design: the stable consumer, the credential generation, and the business operation. For example, a marketplace billing worker remains the same consumer while moving from generation `g1` to `g2`. A retried order remains the same operation across that change. These names describe an application ledger design, not fields promised by a credential provider.

This distinction earns its keep during a rolling deployment. An older worker can still be using the previous credential while a replacement worker starts accepting work. If accounting groups solely by credential generation, one consumer appears in two buckets. If a retry gets a fresh business-operation identifier, a single order can look like two chargeable events. Keep generation information for the access audit, but reconcile the charge against the stable consumer and operation. Preserve the original operation identity across retries, including retries that cross the credential handoff. A successful authorization check cannot establish any of those accounting properties on its own.

Count unresolved consumers. Require zero in the rotation acceptance sample. That is a proposed acceptance threshold, not a reported benchmark.

Infrai offers one key, one bill across backend services, alongside a self-describing REST API. Public discovery needs no key and exposes request and response schemas plus runnable examples in 10 languages, so wiring a capability starts with reading its definition rather than learning another SDK. Its 295 routes across 20 modules reduce separate integration work; consolidated billing reduces vendor-invoice reconciliation, but still calls for distinct consumer credentials. Its specified per-call cost, vendor, latency, and request metadata gives the accounting pipeline useful reconciliation inputs. The application must still establish which marketplace consumer owns each operation; metadata isn't a tenant-to-invoice mapping.

Its idempotency convention is another useful advantage for a production handoff: the `Idempotency-Key` header and default 24h deduplication window let a supported write retain its retry identity across credential generations. Check the selected capability's idempotency declaration first. This convention doesn't make every operation idempotent, and the application ledger still has to reconcile the result. For a worker already using an OpenAI SDK, the OpenAI-compatible surface also avoids replacing that client during integration; that advantage contributes nothing to a build that never calls a model.

Avoid purchasing configuration for its own sake. The billing criterion is whether every accepted operation can be assigned correctly after replacement, not how many key-management settings a tool exposes.

## A Node.js attribution check before the handoff

Use a read-only key-inventory request as a connectivity preflight, then run the local accounting fixtures. Save the TypeScript below as `rotation-check.ts` and execute it with a TypeScript-capable Node.js setup. Supply `INFRAI_API_KEY` from an authorized administrative secret context and set `INFRAI_BASE_URL` to the provider's HTTPS API base ending in `/v1`. Don't grant inventory authority to CI just to run this check. The code needs no SDK and prints neither secrets nor inventory bodies.

```ts
async function checkInventory(): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  const baseURL = process.env.INFRAI_BASE_URL;
  if (!apiKey || !baseURL) throw new Error("Missing API configuration");
  const url = new URL(`${baseURL.replace(/\/$/, "")}/account/keys/list`);
  if (url.protocol !== "https:" || url.pathname !== "/v1/account/keys/list") {
    throw new Error("Expected an HTTPS versioned API base");
  }
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      redirect: "error",
    });
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const requested = Number.isFinite(seconds) ? seconds * 1000
        : Number.isFinite(date) ? date - Date.now() : 0;
      await response.body?.cancel();
      await new Promise(resolve => setTimeout(resolve,
        Math.max(1000 * 2 ** attempt, requested)));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Inventory HTTP ${response.status}: ${await response.text()}`);
    }
    await response.json();
    return;
  }
}

type Entry = {
  consumer: string;
  generation: string;
  operation: string;
};

function reconcile(
  entries: Entry[],
  expectedConsumer: string,
  allowedGenerations: Set<string>,
): number {
  const operations = new Set<string>();
  for (const entry of entries) {
    if (entry.consumer !== expectedConsumer) {
      throw new Error("Unexpected billing consumer");
    }
    if (!allowedGenerations.has(entry.generation)) {
      throw new Error("Credential generation outside this phase");
    }
    if (entry.operation.trim().length === 0) {
      throw new Error("Missing business operation identity");
    }
    operations.add(entry.operation);
  }
  return operations.size;
}

const consumer = "marketplace-prod-billing";
const overlap = new Set(["g1", "g2"]);
const afterRetirement = new Set(["g2"]);
const attempts: Entry[] = [
  { consumer, generation: "g1", operation: "order-1042" },
  { consumer, generation: "g2", operation: "order-1042" },
  { consumer, generation: "g2", operation: "order-1043" },
];

if (reconcile(attempts, consumer, overlap) !== 2) {
  throw new Error("Retry changed the operation count");
}

function expectRejected(entry: Entry, generations: Set<string>): void {
  let rejected = false;
  try {
    reconcile([entry], consumer, generations);
  } catch {
    rejected = true;
  }
  if (!rejected) throw new Error("Invalid accounting record accepted");
}

expectRejected(
  { consumer: "marketplace-ci-build", generation: "g2", operation: "order-1042" },
  overlap,
);
expectRejected(attempts[0]!, afterRetirement);
console.log("Attribution fixtures passed; production handoff still needs evidence");

checkInventory().then(() => {
  console.log("Inventory accepted; inspect records in a trusted admin context");
}).catch(error => {
  console.error(error instanceof Error ? error.message : "Inventory failed");
  process.exitCode = 1;
});
```

The local result is two distinct operations from three attempts. It isn't a billing engine: the set checks operation identity within this sample, and does not persist accounting state or prevent a remote write from happening twice. Production writes need their own supported idempotency mechanism and a durable reconciliation record. Nor does rejecting `g1` in this function revoke a real credential. The inventory request separately checks administrative connectivity, with at most 4 attempts and exponential backoff honoring `Retry-After`; it doesn't prove a production worker has adopted a replacement. Run it in a protected context because service error details belong there, not in public build logs.

The trap is easy to miss: a success-only smoke test can pass while CI records are being assigned to production or retired credentials still appear in accepted work. The two negative fixtures target those mistakes directly.

## Rotate with an overlap window you can account for

For a planned production rotation, first establish whether the service supports an overlap between independently usable credentials. Verify its replacement and retirement semantics before scheduling the rollout. Do not assume that a button labeled “rotate” preserves the previous key. If overlap is available, provision the replacement with the intended narrow scope, deliver it to replacement workers, and confirm an allowed operation from those workers before retiring the previous credential. Keep both generations attached to the same stable consumer in accounting.

Measure the transition. Record replacement availability, the first accepted request from the new generation, the last observed request from the previous generation, unresolved-consumer count, and repeated-operation count. These are a benchmark plan, not measured results. Compare them with the normal workload before declaring the handoff complete. Key creation time alone says nothing about a long-running process that still holds its old environment variable.

End the overlap only after workers have demonstrably moved, then verify that the retired credential is denied. Check accounting again across the whole interval. **Availability and correct attribution need separate evidence.** For a suspected exposure, decide explicitly how much overlap is acceptable: keeping a compromised credential usable to protect availability carries a different risk from a planned rotation. The CI key's narrow scope is what limits that decision's blast radius.

## When is the runner-up a better fit?

AWS IAM deserves the first evaluation when the production worker already sits within AWS. Compare its identity and policy boundary against the deployment you have. The limitation is scope of responsibility: an AWS access design does not automatically define permissions for an unrelated external API. Verify both boundaries rather than treating one credential system as universal.

HashiCorp Vault is worth the extra integration when multiple consumers need a common secret-delivery and lifecycle process. That is a different trade-off from minimizing time to a first API call. A central delivery process still leaves the external service's scopes and the marketplace's accounting identity to verify independently.

Unkey is a closer candidate when API-key management itself is the requirement, while Kong Gateway belongs on the shortlist when an existing gateway should own enforcement. Evaluate consumer identity, retirement behavior, and usage attribution at the boundary each would actually control. Neither choice settles how a retried marketplace order appears on an invoice.

For a CLI or SDK integration, benchmark more than the first successful call: count required configuration entries and manual steps through replacement and retirement. Then test the denied path. No measured vendor winner is established here. The useful choice is the smallest integration that can demonstrate narrow CI authority and preserve production billing identity through the handoff.

## References

- OWASP Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- GitHub Actions security guidance: https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions
- AWS IAM best practices: https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html
- HashiCorp Vault documentation: https://developer.hashicorp.com/vault/docs
- Unkey documentation: https://www.unkey.com/docs
- Kong Gateway documentation: https://developer.konghq.com/gateway/
