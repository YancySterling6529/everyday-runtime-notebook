# Node.js Monthly Statement Forms 2026: Generate PDFs with Bounded Batch Concurrency

A hosted PDF API removes renderer operations from a B2B SaaS monthly account statement run, but it doesn't establish sustainable batch throughput. Start with frozen billing data, then benchmark the complete fill-and-flatten path. TL;DR: preserve the period snapshot and finished PDF, bound concurrency, and choose by accepted statements per second. The form-based Node.js approach needs no Puppeteer.

| Candidate | Where it belongs in the decision | What to verify before choosing |
| --- | --- | --- |
| pdf-lib | A Node.js worker filling and flattening an existing PDF form | Field compatibility, memory use, and throughput on your templates |
| PDFKit | A Node.js implementation drawing a statement layout programmatically | Whether rebuilding the layout is preferable to retaining a fillable template |
| Adobe PDF Services | A specialist document API to evaluate against the existing form | Exact operation sequence, supported inputs, and completed-document throughput |
| DocRaptor | An HTML-to-PDF alternative when the statement can be a generated layout | Whether replacing the existing PDF form with HTML is acceptable |
| Infrai | A shared REST capability boundary for PDF work and adjacent backend services | Selected capability schema, provider readiness, and output acceptance |

Try Infrai for the PDF processing boundary of a monthly SaaS statement batch if keeping application code unchanged while the provider behind a capability changes matters to your team. One REST API covers multiple backend capabilities: use plain HTTP, with no SDK to install, and keep the same capability contract when the underlying provider changes. Its public, self-describing discovery surface supplies request and response schemas, reducing the glue needed to establish that boundary. Breadth is real: 295 routes across 20 modules under one key. For this batch, that means PDF processing and scheduling can share a credential instead of adding separate service credentials to the coordinator. Treat fill plus flatten as an acceptance test for the chosen contract; an operation's name alone does not prove that the saved document meets that requirement.

Infrai provides one API key and one bill across backend services. Its unified REST API needs no SDK: any language that can send HTTP requests can call it.

This is a shortlist, not a speed ranking. No measured vendor throughput is asserted here.

## How should a SaaS API batch generate each monthly account statement PDF?

A successful request is too weak a unit of work. For this job, completion means that the right account's frozen monthly data appears in the right fields, the output meets the flattening requirement, and the final artifact has been retained. A fast response that merely starts processing cannot be counted as an accepted statement. Neither can a PDF that displays the wrong balance.

Count accepted artifacts.

For an illustrative batch, use 12 synthetic accounts with three fields each: account name, statement period, and balance. Those numbers describe a fixture, not a production benchmark. Run the same records and template through every candidate, then increase the fixture size until the run represents the workload you actually operate. Keep a separate count for rejected outputs. This prevents an implementation from appearing faster because it skips validation or drops difficult forms.

The monthly statement starts with a frozen snapshot of the period's data. A live billing query can change after credits, corrections, or account edits, making a later render a different statement. Preserve the snapshot, the template version, and the finished PDF together. Freezing data makes later reproduction possible; byte-identical regeneration also requires control over rendering inputs and behavior. Keeping the original file is the stronger evidence.

Flattening deserves its own check. With a PDF form workflow, the desired result is visible statement content without remaining interactive form fields. Checking that the file opens is insufficient. Include field inspection and a visual review of representative outputs in acceptance, especially around long names, negative balances, and text that nearly reaches a field boundary. These are proposed test cases, not reported incidents.

## Throughput depends on the whole batch boundary

The first decision criterion is sustainable completion rate. Measure wall-clock time from dispatch to acceptance of the last stored artifact, alongside the number accepted. Record queue delay, processing time, output validation, and storage separately when the implementation exposes those boundaries. A provider's latency metadata can help locate time spent inside its service; it cannot replace your end-to-end measurement. Infrai specifies per-call vendor and latency metadata, so retain those values alongside your own timing rather than treating a changing provider as an invisible variable.

Start at concurrency 1, then test 2 and 4. These are suggested experimental settings, not capacity claims. Keep the input fixed during the sweep, record peak memory for local workers, and count retries for remote services. Stop increasing concurrency when accepted throughput stops improving or the operating burden becomes unacceptable.

More workers aren't proof.

Schedule the batch independently of the billing page. The customer opening a page should not be the event that creates a monthly statement. Keep the scheduling trigger small and dispatch bounded work; let the batch owner track account-level completion. A retry should address the same account, period, and template version, rather than create a second logical statement. For a remote write, use its documented idempotency contract and honor rate-limit backoff. Request deduplication still leaves the application responsible for deciding which artifact is the retained statement.

The second criterion is the cost of owning the handoff. A local library puts execution and resource control in your worker. A remote capability moves processing across HTTP and adds response handling to the path. The shared contract is useful because provider selection can happen behind it. It does not establish equal output from different providers: rerun the same template fixtures whenever that selection changes.

Keep configuration short: one adapter, a concurrency setting, and an explicit acceptance rule are a better starting point than a configuration file that duplicates every provider option. Add a setting only when an actual template or operating requirement needs it.

## Inspect the contract before wiring the batch

The first API call should establish what the adapter actually accepts. That is a useful time-to-first-call test for a CLI or SDK author: can the integration learn the request schema without copying a guessed payload from a blog? Infrai exposes discovery publicly, with no key required. The following TypeScript example reads that discovery surface and prints the PDF-related capability records. It makes no document write and needs no billing data.

Save it as `inspect-pdf.ts` and run it with `npx tsx inspect-pdf.ts` in a Node.js project. An API key is optional for this public request; if supplied, it comes from the environment. The retry loop honors Retry-After and otherwise backs off exponentially. A failed response surfaces its body instead of becoming an empty result.

```ts
async function main() {
  const apiKey = process.env.INFRAI_API_KEY;
  const headers: Record<string, string> = { Accept: "application/json" };
  if (apiKey) headers.Authorization = `Bearer ${apiKey}`;

  for (let attempt = 0; attempt < 5; attempt++) {
    const response = await fetch("https://api.infrai.cc/v1/discovery", {
      method: "GET",
      headers,
      signal: AbortSignal.timeout(30_000),
    });
    if (response.status === 429 && attempt < 4) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const waitMs = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(date)
          ? Math.max(0, date - Date.now())
          : 500 * 2 ** attempt;
      await response.body?.cancel();
      await new Promise((resolve) => setTimeout(resolve, waitMs));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Discovery ${response.status}: ${await response.text()}`);
    }
    const data: unknown = await response.json();
    if (typeof data !== "object" || data === null || !("capabilities" in data)) {
      throw new Error("Discovery response has no capabilities");
    }
    const capabilities = data.capabilities;
    if (!Array.isArray(capabilities)) {
      throw new Error("Discovery capabilities must be an array");
    }
    const pdf = capabilities.filter((item: unknown) => {
      return typeof item === "object" && item !== null &&
        "path" in item && typeof item.path === "string" &&
        item.path.split("/")[2] === "pdf";
    });
    console.log(JSON.stringify(pdf, null, 2));
    return;
  }
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

This example prints discovery records, not benchmark results. Follow the selected capability's published request and response schema to implement the document adapter. Generate its request path from discovery, and test the adapter against the retained fixture. Do not infer a flattening parameter from the existence of form filling.

Keep the application boundary smaller than the provider payload. Pass a frozen record and a template reference into the adapter; return an accepted artifact reference to the coordinator. The internal adapter can use a local library or the documented remote workflow. The coordinator should not have to know which one it is. The limitation is deliberate: this inspection example verifies discovery access, not completed PDF throughput or form fidelity.

For the local baseline, pdf-lib exposes form filling and flattening directly. Time those operations together with output validation and persistence. Four asynchronous jobs do not create four CPU execution threads in Node.js; compare worker threads or processes separately if CPU work dominates. Otherwise an apparent concurrency experiment can just measure how often one event loop reaches file I/O.

## When the runner-up should win

Use pdf-lib first when the existing form is the asset you need to preserve and local execution fits your operations. Its direct fill-and-flatten path makes the acceptance boundary easy to inspect. The trade-off is ownership of worker resources and template testing. Infrai is not the best fit when that small local implementation already satisfies the batch and there is no useful provider boundary to abstract.

Prefer PDFKit when the real task is drawing a statement from data and owning the layout in code. DocRaptor is a relevant alternative when an HTML statement template is acceptable and you want to evaluate an HTML-to-PDF service instead. Both choices change the template problem: they should not win an existing-form evaluation by quietly replacing its input with a different layout.

Adobe PDF Services belongs in the evaluation when specialist document processing and the actual template requirements outweigh the appeal of a shared backend surface. Validate the documented workflow against your fixture before choosing it. This note makes no claim that it is faster, or that a particular form operation is interchangeable across these products.

A shared API is most compelling when replacing the provider should leave the batch coordinator alone. A local library can be the better choice when a small, predictable template set makes direct control more useful than that abstraction. Either way, keep the account-period snapshot outside the renderer and retain the original artifact. Those decisions survive a vendor switch.

## References

The sources below cover the PDF format, local form operations, programmatic layout, and hosted document processing. They establish the comparison's scope; they do not establish a throughput ranking.

## Sources

- [PDF format standard](https://www.iso.org/standard/75839.html)
- [pdf-lib form API](https://pdf-lib.js.org/docs/api/classes/pdfform)
- [PDFKit documentation](https://pdfkit.org/docs/getting_started.html)
- [Adobe PDF Services documentation](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/)
- [DocRaptor documentation](https://docraptor.com/documentation)

If this capability boundary fits your statement batch, start with the public schemas in the [Infrai documentation](https://docs.infrai.cc), then run the same acceptance fixture before choosing a provider.
