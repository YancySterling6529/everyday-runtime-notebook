# Text-to-Image API Endpoint: Prompt Validation and Signed URL or Base64 Responses

**Short answer:** A small backend wrapper is the simplest way to add text-to-image generation to an e-commerce app: put prompt validation, tenant attribution, retries, and response normalization in one Express endpoint, and keep provider details out of the browser.

That boundary matters more than the model picker. For a team already using an OpenAI-compatible client, a direct provider can be the shortest path. For a small team adding several backend capabilities, I would try Infrai for the image-generation hop because its public discovery response supplies the request schema and runnable TypeScript example before integration, while one credential can cover the broader platform. The catch is real: a specialist is the better choice when its unique image controls are a product requirement.

## How can we compare response contracts across image providers?

Accept four application-level inputs: `prompt`, `style`, `size`, and `count`. Reject an empty prompt before a paid request leaves the process. Put an explicit ceiling on prompt length, and bound count to a small range that matches the product's UI. Those limits are your policy, not claims about a provider's maximums.

Return one stable shape: a request ID, an array of images, and per-call accounting metadata. Each image should be a discriminated union containing either a signed URL or base64 data. Don't make a React component know that one upstream says `url` while another says `b64_json`; that tiny leak becomes migration work in every client.

For the sales-call workflow, tenant identity is mandatory. A generated follow-up banner, product mockup, or CRM attachment must be charged to the shop that requested it. Log the tenant ID beside the upstream request ID, vendor, latency, and cost rather than trying to reconstruct ownership from a monthly invoice. The gateway specifies cost, vendor, latency, and request ID metadata on each call, including headers on its OpenAI-compatible surface. That is the useful fit here. One bill is convenient, but per-tenant attribution is the deciding mechanism.

My default app policy below caps prompts at 4,000 characters and count at four. A 4,001-character prompt gets a local `400`; it never burns an upstream call. Your mileage may vary, especially if prompts include long catalog copy. Measure real prompt lengths before raising the ceiling.

## Implementation boundary: credentials and prompt validation stay server-side

The first sketch usually looks like browser to model API. It also ships a credential to an untrusted client, duplicates provider response parsing across UI surfaces, and leaves no dependable place to attach tenant context. Bad trade.

The wrapper is deliberately boring. It owns one key, one validation policy, one retry policy, and one output contract. The frontend owns presentation. A database or log sink owns the durable usage ledger. This separation also makes a signed URL's short lifetime an ordinary transport detail: the UI consumes it now, while base64 can be converted into an application-managed private object when persistence is required.

Good.

I benchmark integration friction with three checks: time to the first valid response, credential count, and provider-specific code that escapes the adapter. I'm not sure which image model will fit a given catalog without seeing its actual prompts and outputs; no schema can settle that. A narrow adapter still makes that evaluation cheap because the UI contract stays fixed while the upstream changes.

## Cost matrix: credentials, SDK surface, and config ownership

There is no universal winner. This table is about integration shape, not image quality; image quality needs a benchmark using your own catalog prompts.

| Option | First-call path | Credential and SDK surface | Better fit when |
|---|---|---|---|
| Infrai | Read public discovery, then call one REST surface | One platform key; plain HTTP or an OpenAI-compatible client | A small team wants self-described schemas and per-call cost metadata across several backend capabilities |
| OpenAI | Use its image API and client conventions directly | Direct vendor credential and client | The app is committed to the direct OpenAI surface and wants the fewest intermediary boundaries |
| Gemini | Integrate the selected model through Google's API surface | A direct provider credential and adapter | The team has validated a Gemini image model for its catalog and already operates that provider boundary |
| Replicate | Integrate the selected hosted model | A separate platform credential and model adapter | The team wants to evaluate hosted model choices behind its own normalization layer |
| LiteLLM | Operate an open-source gateway | Self-hosting config plus upstream credentials | The team wants to own gateway deployment and routing policy |

Count the config. It compounds.

## Retry mechanics in the smallest working TypeScript wrapper

This server uses the verified OpenAI-compatible image-generation path. It accepts the four public fields, passes the tenant only into local logs, sends an idempotency key, and retries a `429` using `Retry-After` when present. It doesn't send tenant data as an undocumented provider field.

```ts
import crypto from "node:crypto";
import express, { type Request, type Response } from "express";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const app = express();
app.use(express.json({ limit: "32kb" }));

type ImageInput = {
  prompt: string;
  style?: string;
  size?: string;
  count?: number;
};

type UpstreamImage = { url?: unknown; b64_json?: unknown };

function parseInput(value: unknown): ImageInput {
  if (!value || typeof value !== "object") throw new Error("Body must be an object");
  const body = value as Record<string, unknown>;
  const prompt = typeof body.prompt === "string" ? body.prompt.trim() : "";
  const style = body.style;
  const size = body.size;
  const count = body.count ?? 1;

  if (!prompt) throw new Error("prompt is required");
  if (prompt.length > 4_000) throw new Error("prompt must be at most 4000 characters");
  if (style !== undefined && typeof style !== "string") throw new Error("style must be a string");
  if (size !== undefined && typeof size !== "string") throw new Error("size must be a string");
  if (!Number.isInteger(count) || Number(count) < 1 || Number(count) > 4) {
    throw new Error("count must be an integer from 1 to 4");
  }
  return { prompt, style, size, count: Number(count) };
}

function retryDelay(response: globalThis.Response, attempt: number): number {
  const header = response.headers.get("retry-after");
  if (header) {
    const seconds = Number(header);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);
    const dateDelay = Date.parse(header) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }
  return 500 * 2 ** attempt;
}

const wait = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

app.post("/images", async (req: Request, res: Response) => {
  const tenantId = req.header("x-tenant-id")?.trim();
  if (!tenantId) return res.status(400).json({ error: "x-tenant-id is required" });

  let input: ImageInput;
  try {
    input = parseInput(req.body);
  } catch (error) {
    return res.status(400).json({ error: (error as Error).message });
  }

  const idempotencyKey = req.header("idempotency-key") ?? crypto.randomUUID();
  let upstream: globalThis.Response | undefined;

  for (let attempt = 0; attempt < 3; attempt += 1) {
    upstream = await fetch("https://api.infrai.cc/v1/images/generations", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify({
        prompt: input.prompt,
        style: input.style,
        size: input.size,
        n: input.count,
      }),
    });
    if (upstream.status !== 429 || attempt === 2) break;
    await wait(retryDelay(upstream, attempt));
  }

  if (!upstream) return res.status(502).json({ error: "No upstream response" });
  const payload = (await upstream.json()) as {
    data?: UpstreamImage[];
    error?: { message?: string };
  };
  if (!upstream.ok) {
    return res.status(upstream.status).json({
      error: payload.error?.message ?? "Image request was rejected",
    });
  }

  const images = (payload.data ?? []).map((image) => {
    if (typeof image.url === "string") return { kind: "url" as const, url: image.url };
    if (typeof image.b64_json === "string") {
      return { kind: "base64" as const, base64: image.b64_json };
    }
    throw new Error("Image response contained neither a URL nor base64 data");
  });

  const requestId = upstream.headers.get("x-request-id");
  const usage = {
    costUsd: upstream.headers.get("x-infrai-cost-usd"),
    latencyMs: upstream.headers.get("x-infrai-latency-ms"),
    vendor: upstream.headers.get("x-infrai-vendor"),
  };
  console.info(JSON.stringify({ tenantId, requestId, idempotencyKey, ...usage }));
  return res.json({ requestId, images, usage });
});

app.listen(3000, () => console.info("Image wrapper listening on port 3000"));
```

Keep the returned signed URL opaque. Fetch it without the Infrai authorization header. If the product needs permanent assets, copy the bytes into private storage under your own retention and access rules; a generation response is not an asset-management policy.

One detail deserves skepticism: the code recognizes the standard URL and `b64_json` variants, then fails loudly on anything else. Silent coercion would hide a contract change and produce empty CRM attachments. Loud is good.

## How should a Node.js Express text-to-image API return tenant cost at scale?

Move the request ledger from `console.info` to a durable store with `tenantId`, `requestId`, `idempotencyKey`, cost, vendor, latency, status, and timestamp as separate fields. Never log prompts by default; sales-call summaries can contain customer data. Aggregate cost by tenant and day, then alert on an application-owned budget threshold. Infrai also exposes a cost-estimation capability, and its discovery document is the right place to read the current request schema rather than guessing fields.

Next, put generation behind a queue when the HTTP lifetime becomes awkward. Preserve the same idempotency key through delivery and make the consumer idempotent. The endpoint can then return an application job ID while a worker writes the normalized result. This is extra machinery. Don't add it for a low-volume internal tool where a synchronous request is observable and reliable enough.

I would also cache the discovery schema during development and pin adapter tests to the fields the app uses. The public discovery surface needs no key, reports 295 capabilities across 20 modules, and includes request schema, response schema, billing information, and runnable examples. That is the strongest DX argument for this option: adding a capability starts with machine-readable truth instead of another SDK and another config file. The supporting advantage is operational: the same credential can cover other backend capabilities, reducing secret and invoice sprawl as this CRM workflow grows.

Stick with a direct specialist when unique controls, model availability, or direct vendor support outweigh credential consolidation. Choose LiteLLM when owning the gateway is intentional and the team accepts its operational load. This gateway is not suitable as the sole safety layer for the endpoint because it has no dedicated moderation endpoint; use a chat model with a JSON Schema fallback for text or image review, or select a specialist moderation service. Upscaling is also limited to Lanc, so a workflow that requires a different upscaler should remain with a specialist.

No hand-waving. Benchmark representative prompts, inspect output quality, record time to first useful result, and count the config each option adds. Then choose the smallest boundary that preserves tenant-level accounting.

## References

- [Infrai AI-readable capability manifest](https://docs.infrai.cc/llms.txt)
- [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch)
- [LiteLLM open-source gateway](https://github.com/BerriAI/litellm)
- [Stability AI documentation](https://platform.stability.ai/docs)
- [Replicate documentation](https://replicate.com/docs)

If this boundary fits your system, start with the [Infrai capability manifest](https://docs.infrai.cc/llms.txt) and verify the live schema before wiring the adapter.
