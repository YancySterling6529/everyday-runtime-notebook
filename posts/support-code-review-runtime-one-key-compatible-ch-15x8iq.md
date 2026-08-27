# Support Code Review Runtime: One-Key Compatible Chat Completions with Model Routing

Short answer: put one compatible chat-completions boundary in front of OpenAI, Claude, and Gemini models, but route each support-code review by an explicit quality-versus-latency policy and validate every finding before it reaches an agent.

A shared key and a familiar request shape remove SDK sprawl. They don't make model behavior interchangeable. For customer support, that distinction matters: a fast answer with a malformed file location is worse than a slower answer an agent can act on, while sending every tiny change through the slowest review path wastes the latency budget. The useful abstraction is therefore small — transport, routing policy, and schema validation — with model-specific behavior kept behind it.

The deciding constraint is the output contract, not brand preference.

The job is narrow: review a proposed code change and return structured findings for a support engineer. The input includes a patch, a repository hint, and a latency class. The output needs a severity, file, line, and explanation. Free-form prose isn't the contract.

Quality and latency pull in opposite directions, but a single global model setting hides that conflict. A typo in an internal reply template can take the fast lane. An authentication change, a payment handler, or a patch with no reliable file context should take the quality lane. This is workload routing, not vendor ranking.

The first design decision is to keep the route table in application code rather than scatter model strings through handlers. The second is to validate output after parsing. JSON mode, structured-output features, and prompt instructions can reduce formatting drift, yet the application still owns the boundary. A compatible API only promises a common way to send the request; it can't promise that different models interpret severity or evidence identically.

That is the catch.

Treat the gateway response as untrusted input. Reject a finding whose line is negative, whose file is blank, or whose severity falls outside the small vocabulary your support workflow understands. Also allow an empty findings array. Forcing a model to invent a problem is a quiet failure mode, and it is harder to spot than invalid JSON because the payload looks healthy.

I benchmark the whole decision, not just time to first token: request duration, valid-response rate, empty-review rate, and the share of findings later accepted by a human. I'm not sure which model will win on a given repository without that evaluation set. Nobody should be. Language mix, diff size, and the team's definition of a blocking issue can change the result, so a public leaderboard can't settle this local decision.

## How can one key Node.js API route chat completions across models?

Use one small function with a provider-neutral request shape. The example below uses the standard `POST /v1/chat/completions` path exposed by an OpenAI-compatible endpoint, sends one bearer key, selects a model from an application-owned policy, and validates the returned JSON. It deliberately uses the platform `fetch` API instead of an SDK. Less config. Fewer moving pieces.

```ts
type ReviewClass = "fast" | "quality";
type Severity = "low" | "medium" | "high";

type ReviewInput = {
  patch: string;
  repository: string;
  reviewClass: ReviewClass;
};

type Finding = {
  severity: Severity;
  file: string;
  line: number;
  explanation: string;
};

type ReviewResult = {
  findings: Finding[];
};

type ChatResponse = {
  choices?: Array<{
    message?: {
      content?: string | null;
    };
  }>;
};

const modelByClass: Record<ReviewClass, string> = {
  fast: process.env.FAST_REVIEW_MODEL ?? "",
  quality: process.env.QUALITY_REVIEW_MODEL ?? "",
};

function isFinding(value: unknown): value is Finding {
  if (typeof value !== "object" || value === null) return false;
  const item = value as Record<string, unknown>;
  const severities: Severity[] = ["low", "medium", "high"];

  return (
    typeof item.severity === "string" &&
    severities.includes(item.severity as Severity) &&
    typeof item.file === "string" &&
    item.file.length > 0 &&
    Number.isInteger(item.line) &&
    (item.line as number) >= 0 &&
    typeof item.explanation === "string" &&
    item.explanation.length > 0
  );
}

function parseReview(content: string): ReviewResult {
  const value: unknown = JSON.parse(content);
  if (typeof value !== "object" || value === null) {
    throw new Error("Review output must be an object");
  }

  const findings = (value as Record<string, unknown>).findings;
  if (!Array.isArray(findings) || !findings.every(isFinding)) {
    throw new Error("Review output has invalid findings");
  }

  return { findings };
}

export async function reviewChange(input: ReviewInput): Promise<ReviewResult> {
  const baseUrl = process.env.AI_BASE_URL;
  const apiKey = process.env.AI_API_KEY;
  const model = modelByClass[input.reviewClass];

  if (!baseUrl || !apiKey || !model) {
    throw new Error("AI_BASE_URL, AI_API_KEY, and review models are required");
  }

  const response = await fetch(`${baseUrl}/v1/chat/completions`, {
    method: "POST",
    headers: {
      authorization: `Bearer ${apiKey}`,
      "content-type": "application/json",
    },
    body: JSON.stringify({
      model,
      temperature: 0,
      messages: [
        {
          role: "system",
          content:
            "Review the patch. Return JSON only: {findings: [{severity, file, line, explanation}]}. Return an empty findings array when no issue is supported by the patch.",
        },
        {
          role: "user",
          content: JSON.stringify({
            repository: input.repository,
            patch: input.patch,
          }),
        },
      ],
    }),
  });

  if (!response.ok) {
    throw new Error(`Chat request failed with status ${response.status}`);
  }

  const payload = (await response.json()) as ChatResponse;
  const content = payload.choices?.[0]?.message?.content;
  if (!content) throw new Error("Chat response did not contain content");

  return parseReview(content);
}
```

The environment carries two model identifiers because model catalogs vary across compatible services. No provider name belongs in this function. Switching the upstream service means changing the base URL, key, and configured identifiers; the application contract stays put. That is the practical meaning of a drop-in replacement here, and it is intentionally modest. It does not claim identical token accounting, tool behavior, safety policy, rate limits, or output quality.

There is one sharp edge in this tiny version: `JSON.parse` fails closed. Good. In a support workflow, a rejected machine response can enter a controlled retry or manual-review queue, while a half-parsed finding can quietly point an agent at the wrong line. Don't shave a few milliseconds by accepting ambiguous output.

## Failure handling for support review

Start with a fixed set of real, redacted support patches and expected review outcomes. Each case should say which findings are required, which are acceptable, and which would be harmful. Then run every candidate model through the same endpoint, prompt, timeout, and validator. Record latency distributions rather than one average; a p50 hides the long wait that an on-call support engineer actually remembers.

The test harness should separate four outcomes: valid and useful, valid but empty, structurally invalid, and semantically wrong. HTTP `429` belongs in an operational bucket, not the quality score. Likewise, a parser rejection is a contract failure even if the discarded prose sounded convincing. This separation prevents routing policy from rewarding a model merely because infrastructure errors and weak reviews were blended into one failure percentage.

Use repository strata too. Consider a fixture where a patch changes an authorization guard on line 48, removes the early return for an unauthenticated request, and leaves the rest of a 200-line handler untouched. The required outcome is one high-severity finding tied to that file and line; an extra style complaint is acceptable but irrelevant; a claim about a database call absent from the patch is harmful. Run the same fixture repeatedly through both route classes, preserve the raw validated result for the evaluation window, and ask a reviewer to mark the security finding as accepted or missed. Then contrast it with a 12-line reply-template edit where speed matters and a false high-severity alarm would interrupt support work. These cases expose something a blended score cannot: the quality route may justify its delay for the authorization patch while adding no useful signal for the template. For each stratum, compare p50 and p95 end-to-end latency with reviewer acceptance. Move a workload to the quality class only when its acceptance gain is worth the measured delay — a product decision, not a universal constant.

OpenAI, Anthropic, and Google each document ways to work through OpenAI-shaped clients or APIs, but their compatibility guidance also preserves provider-specific boundaries. That makes all three reasonable candidates for the same harness, not evidence that their behavior is identical.

| API surface | Integration path | Useful evaluation role | Main boundary |
| --- | --- | --- | --- |
| OpenAI | Native chat-completions API | Baseline for the shared request shape | Native behavior is not a promise about other services |
| Anthropic | OpenAI SDK compatibility or native API | Exercise Claude models behind the adapter | Compatibility does not erase provider-specific behavior |
| Gemini | OpenAI compatibility or native API | Exercise Gemini models behind the adapter | Model identifiers and supported behavior remain service-specific |

Keep native adapters available when a required capability isn't represented faithfully by the common chat-completions surface. Compatibility is useful until it erases a feature the workload needs.

One more check matters: replay the evaluation after a model identifier, prompt, validator, or routing rule changes. Pinning the application contract is not the same as pinning model behavior. Store the request policy version and selected model beside each result so an accepted finding can be reconstructed later without logging raw secrets or needlessly retaining customer code.

## Governance belongs in the measurement loop

The smallest implementation makes one synchronous call. At scale, I would put a bounded queue ahead of it, attach a deadline and idempotency key to each review job, and keep routing policy in a versioned configuration object. Retries should be limited to failures that are safe to repeat, use backoff, and never multiply a request after its deadline has expired. The support UI needs a distinct state for “review unavailable”; silence must not masquerade as “no findings.”

Observability should follow the contract: route class, model identifier, policy version, duration, response status, validation result, and final human disposition. Avoid patch bodies in routine logs. They can contain customer data, credentials accidentally committed to a diff, or internal implementation details. A stable hash can correlate duplicate inputs without turning the log system into a second code archive.

I would also replace the handwritten validator with JSON Schema or a schema library once more result types appear. The handwritten version is readable for four fields. It becomes config bloat in disguise when nested suggestions, ranges, categories, and confidence evidence arrive — exactly the kind of glue that makes an allegedly simple compatibility layer expensive to own.

Keep it boring.

## Rollout without config sprawl

Ship the adapter in shadow mode first: create the structured review without showing it to support agents, validate it, and collect the same disposition labels used by the evaluation harness. Next, expose it only for the fast, low-risk patch class. The quality path follows after its acceptance and latency thresholds are written down. A versioned policy object should be the only place that maps workload classes to configured model identifiers; handlers should never accumulate provider branches.

For operational retries, honor `429` responses with bounded exponential backoff and jitter. Carry an idempotency key on queued review jobs so a retried write to your own job system doesn't create duplicate work. The model call itself still gets a deadline. Once that deadline has passed, route the item to the declared unavailable state or manual review instead of extending the queue invisibly.

## The decision boundary

A one-key compatible layer is not suitable when the application depends on a provider-native feature that the shared request and response contract cannot express. In that case, keep the neutral `reviewChange` interface but implement a native adapter behind it. The rest of the support system should consume validated findings, not provider payloads.

Stick with direct provider integrations when separate credentials, billing boundaries, compliance controls, or independent failure domains are requirements rather than inconveniences. One credential reduces setup friction — a real time-to-first-call win — but it also concentrates access. The right credential topology comes from the threat model and organizational boundary, not from a prettier `.env` file.

The final routing rule is plain: choose the fastest path that clears the measured acceptance threshold for that patch class, escalate sensitive or ambiguous changes to the quality path, and retain manual review when neither path clears it. Don't turn “multi-model” into random distribution. Routing earns its complexity only when a named workload class has evidence for a different choice.

## References

- https://platform.openai.com/docs/api-reference/chat/create
- https://platform.openai.com/docs/guides/embeddings
- https://docs.anthropic.com/en/api/openai-sdk
- https://ai.google.dev/gemini-api/docs/openai
- https://json-schema.org/draft/2020-12/json-schema-core
- https://nodejs.org/api/globals.html#fetch
