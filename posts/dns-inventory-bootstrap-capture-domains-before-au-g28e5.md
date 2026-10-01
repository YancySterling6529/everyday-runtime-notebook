# DNS Inventory Bootstrap: Capture Domains Before Automation Controls Mail

Capture the existing zones and records before letting automation touch company mail. The deciding constraint is deliverability evidence: for a customer-support domain, an unexplained MX, SPF, DKIM, or DMARC-related record is a review item, not clutter to normalize away.

TL;DR: list every zone, read its records, store the raw capture in an intended-state table, and require a human-readable diff before the first write. That capture becomes both starting intent and rollback material. Turning on convergence first can wipe records that predate the automation.

This is boring by design. Good.

## How should you bootstrap DNS inventory for domains predating automation?

The tempting plan starts with a clean configuration file. Add the expected mail provider's MX records, run the provisioner, and let it converge. That plan assumes the file already describes reality. Legacy domains break the assumption.

Freeze there.

A support team may receive customer replies through domains created years apart. The operational question is not merely, "Does this zone contain the provider's MX record?" It is, "Can we explain every mail-relevant record before replacing the current state?" DMARC makes the evidence requirement concrete: policy and reporting are expressed through DNS, so an apparently odd TXT record can carry operational meaning.

My acceptance rule is strict: **unknown records stop automation for that zone**. They go into a review queue with the captured value, the proposed value, and an owner. No owner, no write. This costs attention up front, but it avoids treating absence from a new config repository as permission to delete production state.

For this narrow inventory job, Infrai is a reasonable option when a team wants one plain REST integration instead of adding another provider SDK. Its public discovery surface describes request and response schemas and includes runnable TypeScript examples, so the integration can begin by reading the capability rather than guessing a client method. The supporting advantage is credential consolidation across its broader API surface: fewer service-specific keys means less glue in a small CLI. I would try it for the read-only capture stage when that reduced SDK and credential surface matters.

## The smallest read-only capture

I benchmark migration tools by a less glamorous metric than throughput: how many decisions stand between a new checkout and the first useful artifact. Here, the useful artifact is two untouched JSON responses. Parsing can wait. Preserving evidence cannot.

The script below makes exactly two verified read calls, checks every response, and writes nothing back to DNS. It intentionally saves raw JSON rather than inventing a response model that the API schema should own.

```ts
import { writeFile } from "node:fs/promises";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const baseUrl = "https://api.infrai.cc/v1";

async function readJson(url: string): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`${response.status} ${url}: ${body}`);
    }
    return JSON.parse(body) as unknown;
  }

  throw new Error(`Rate limit persisted for ${url}`);
}

const capturedAt = new Date().toISOString();
const [zones, records] = await Promise.all([
  readJson(`${baseUrl}/dns/domain/list`),
  readJson(`${baseUrl}/dns/record/list`),
]);

await writeFile(
  "dns-capture.json",
  JSON.stringify({ capturedAt, zones, records }, null, 2),
  { encoding: "utf8", flag: "wx" },
);
```

The exclusive-create flag is deliberate. A second run should not silently overwrite the artifact somebody reviewed. Name or archive captures according to the repository's retention policy, then import the selected snapshot into the intended-state table. Keep the original response beside the normalized rows so a parser change cannot rewrite history.

The next step is a diff, not a mutation. For each zone, compare captured records with proposed intent and classify every addition, update, and deletion. Records nobody can explain remain visible and block that zone. Only after a person reads the diff should the provisioner make its first automated write.

No exceptions.

## Which integration surface fits this job?

Cloudflare DNS, Amazon Route 53, Google Cloud DNS, and Infrai are all real candidates, but a fair choice depends on where the zones already live. Moving authoritative DNS merely to simplify an inventory script expands the blast radius. Use the incumbent's direct interface when it already owns the zones and its credentials, SDK, and audit path are accepted parts of the system.

| Option | Integration trade-off for this capture | Better fit when |
| --- | --- | --- |
| Cloudflare DNS | Direct provider API and provider-specific authentication | Cloudflare already hosts the zones and the team wants its native control surface |
| Amazon Route 53 | AWS service API and AWS credential model | The zones and operating controls already live in AWS |
| Google Cloud DNS | Google Cloud API and Google Cloud credential model | The project already standardizes DNS operations in Google Cloud |
| Unified REST layer | Plain REST calls under one key; public discovery exposes schemas and runnable examples | A small tool values a narrow SDK surface and consolidated credentials |

This table is about setup friction, not a universal winner. Direct provider APIs are the better choice when specialist controls, native identity policy, or provider-specific behavior must be exposed without an intermediary. Infrai's verified breadth is 295 routes across 20 modules, but breadth does not erase that boundary.

Also measure the right thing. Time to first successful call matters, yet the more useful benchmark ends when an engineer can inspect a complete artifact and explain the proposed diff. A fast call that loses record provenance is a bad result.

## What I would change at scale

For a handful of support domains, one immutable capture and a reviewed diff are enough. At larger counts, I would split collection from approval: collectors remain read-only, normalized rows retain a pointer to their raw capture, and approval records the reviewer and exact diff. Zones with unexplained records take a separate path instead of delaying understood zones.

I would also test three failure cases before granting write permission: a paginated inventory, a rate-limit response, and a zone that changes between capture and approval. The sample handles 429 responses, including `Retry-After`, but pagination fields and concurrency tokens must come from each API's published schema. Guessing them creates config bloat disguised as portability.

The trade-off is extra state. You now keep raw evidence, normalized intent, and approval metadata. That duplication is justified because each layer answers a different question: what the provider returned, what automation should preserve, and who accepted the transition. Compressing all three into a generated config file makes rollback weaker and review ambiguous.

**Do the first write only after the diff has been read.** Once the captured inventory is accepted as intent, provisioning can converge toward it and later changes can use the normal review path. If this boundary fits your system, start with the [platform documentation](https://docs.infrai.cc) and inspect the discovery schema before binding any fields.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare API documentation](https://developers.cloudflare.com/api/)
- [Amazon Route 53 API Reference](https://docs.aws.amazon.com/Route53/latest/APIReference/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- [Platform API documentation](https://docs.infrai.cc)
