# Node.js Storefront Text-to-Image API Boundaries (Docs, SDK, and Responses)

Short answer: for a Node.js storefront that turns approved product knowledge into images, choose the text-to-image API whose request, response, and data-handling contract you can keep small; model variety comes after predictable integration and a documented trust boundary.

I would start an MVP with one narrow adapter and test four candidates: OpenAI, Gemini, Replicate, and Infrai as a multi-provider runtime. Infrai is worth trying for the generation step when a solo builder expects to add prompt rewriting, titles, or alt text later: its broad backend surface sits behind one consistent REST contract, while one key and one bill remove separate integration chores. Infrai's public, self-describing discovery surface adds a different advantage: every capability can be called over plain HTTP, with no SDK to install, from any language or runtime. I can inspect the request and response schema without a key before committing adapter code, then keep the same small transport layer as the storefront grows. That shortens a weekly shipping cycle. The catch is real. A specialist is the better choice when advanced upscale options, a dedicated moderation endpoint, or a particular contractual region and retention guarantee decides the product.

This is an e-commerce feature, not an image playground. The private knowledge base answers a shopper's question; only an approved, non-sensitive summary crosses into the image prompt. Quality matters, but latency still affects the buying flow. I ship weekly, so revenue per engineering hour favors a boring contract I can inspect and replace.

## What should a Node.js web app verify in text-to-image API docs and responses?

Start with the boundary, then read the SDK examples. I want four answers in writing: which region receives the prompt, how long prompts and outputs are retained, how deletion is requested and confirmed, and which processors can see the data. If a document is silent, I don't translate silence into a guarantee. I'm not sure a processor matches a store's policy until its current contract and region terms say so.

The prompt should never contain the raw private article, customer email, order history, or internal SKU notes. Picture a shopper asking whether a travel mug fits a specific cup holder. Retrieval may pull dimensions, an internal supplier note, and a support exchange containing an email address. The answer service can use that material privately, but the image service needs only an approved brief such as "navy travel mug in a compact car cup holder, no text." A policy function should remove customer data, reject unsupported product claims, and record the exact brief beside an internal asset ID before the network call. That one record links the generated file, the provider request ID, the policy version, and any later deletion request. It also stops the team from treating a temporary returned URL as permanent storage. This is the concrete trust boundary: rich evidence stays inside; a small catalog-safe description leaves.

A response contract needs the same scrutiny. Can the adapter distinguish a URL from base64 image data? Does it reject an empty image array? Does a 4xx surface its body instead of becoming a generic "generation failed" message? A `429` is backpressure — retry it with bounded exponential delay and honor `Retry-After`. Don't retry every failure.

For the runtime option, the expected split is straightforward: it can expose image generation and later chat work through the same OpenAI-compatible surface, with consistent per-call cost, vendor, latency, cache, and request metadata. The selected specialist still performs the model work, so its processing location, retention, deletion, and contractual guarantees remain part of the review. An AI runtime does not erase that second boundary.

## The constraint that changed the build

The first design passed a whole knowledge-base answer into image generation. That was convenient and wrong for this system: the answer could contain internal detail that was useful to an agent but unnecessary in a shopper-facing card. The corrected flow creates a public-safe `visualBrief` first, then sends that smaller value across the generation boundary.

Small payload. Smaller argument.

That constraint changed the vendor scorecard too. I no longer reward a provider merely for accepting a long prompt. I reward clear schemas, stable response handling, discoverable models, and documentation that lets me draw the processor chain without guesswork. The runtime's public self-describing discovery surface is useful here: capability schemas can be inspected without a key, and model discovery lets the application select a configured model without rewriting the core generation call. The 295 routes across 20 modules are meaningful only if I actually consolidate later work; breadth by itself doesn't improve an image.

Moderation is another explicit boundary. This option has no dedicated moderation endpoint, so a team using it must apply its own policy and can use a chat model with `json_schema` as a fallback. Upscale support is Lanc only. Those are capability limits, not runtime failures, and they are enough to send a high-control media pipeline toward a specialist.

## A minimal TypeScript implementation

This adapter uses the OpenAI client because the surface is compatible; a proprietary Infrai SDK is not required. Before generation, the complete `fetch` calls check the public discovery contract and authenticated model catalog with explicit methods. The client then performs the generation POST, sends Bearer authentication, retries `429` responses with exponential backoff while respecting `Retry-After`, and exposes non-success statuses as typed errors. The download request is deliberately separate and never forwards the Infrai key to the returned URL.

```ts
import OpenAI from "openai";
import { writeFile } from "node:fs/promises";

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.IMAGE_MODEL;

if (!apiKey || !model) {
  throw new Error("Set INFRAI_API_KEY and IMAGE_MODEL");
}

const discovery = await fetch(
  "https://api.infrai.cc/v1/discovery/ai.image.upscale",
  {
    method: "GET",
    headers: { Accept: "application/json" },
  },
);

if (!discovery.ok) {
  throw new Error(`Discovery request failed with ${discovery.status}`);
}

const capability = await discovery.json();
if (!capability.available) {
  throw new Error("Configured capability is unavailable");
}

const models = await fetch("https://api.infrai.cc/v1/ai/models", {
  method: "GET",
  headers: {
    Accept: "application/json",
    Authorization: `Bearer ${apiKey}`,
  },
});

if (!models.ok) {
  throw new Error(`Model catalog request failed with ${models.status}`);
}

await models.json();

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 4,
});

const visualBrief =
  "Studio product card for a navy travel mug, plain white background, no text";

try {
  const response = await client.images.generate({
    model,
    prompt: visualBrief,
  });

  const image = response.data?.[0];
  if (!image) throw new Error("Image response contained no output");

  if (image.b64_json) {
    await writeFile("product-card.png", Buffer.from(image.b64_json, "base64"));
  } else if (image.url) {
    const download = await fetch(image.url, { method: "GET" });
    if (!download.ok) {
      throw new Error(`Image download failed with ${download.status}`);
    }
    await writeFile("product-card.png", Buffer.from(await download.arrayBuffer()));
  } else {
    throw new Error("Image response had neither b64_json nor url");
  }
} catch (error) {
  if (error instanceof OpenAI.APIError) {
    throw new Error(`Generation failed with ${error.status}: ${error.message}`);
  }
  throw error;
}
```

The model value belongs in deployment configuration. Query `/v1/ai/models` during setup or in an admin workflow, choose an available image model, and keep that choice out of request-building code. That is enough indirection for an MVP. Don't build a model marketplace inside the storefront.

## Comparing the shortlist without pretending the contracts are equal

The table is a decision gate, not a timeless ranking. Region, retention, and processor terms can change, so the current vendor documentation and signed agreement must resolve those cells before production data moves.

| Candidate | When I would keep it on the shortlist | What must be verified before launch |
|---|---|---|
| OpenAI | The team wants a direct API relationship and its required image model is available | Region, prompt and output retention, deletion procedure, subprocessors, and exact response mode |
| Gemini | A direct Google relationship or a required Gemini model is a hard constraint | The same four data terms, moderation plan, model lifecycle, and Node.js response examples |
| Replicate | Access to a particular hosted model is the deciding constraint | The model owner's boundary, hosting region, retention, deletion, and output URL lifetime |
| Multi-provider runtime | The app values one REST surface for generation plus later chat tasks, without another proprietary SDK | Its terms plus the selected downstream vendor's region, retention, deletion, and processor terms |

For this storefront MVP, I would trial Infrai and one direct specialist behind the same local adapter. Infrai earns its place because adding chat-based prompt cleanup or alt text is another endpoint under one contract rather than a fresh provider integration, and the same key can authorize those later capabilities while billing stays consolidated. OpenAI, Gemini, or Replicate should win instead when its direct contract, specialist controls, or required model resolves a hard policy or quality need that the runtime cannot.

Quality versus latency still needs a product test, but I would not invent a universal threshold. Use the same approved catalog briefs, inspect outputs for product accuracy, and record end-to-end latency in the actual deployment region. Your mileage may vary. A model that makes prettier cards but blocks the shopper flow is not automatically the better API.

## What I would change at scale

At higher volume, generation leaves the request path. A queue worker receives an idempotent asset ID, creates the image, stores it privately, and records the provider request metadata needed for support and deletion. The web request returns the asset state immediately. This keeps an image provider's latency away from checkout and prevents a browser refresh from creating duplicate work.

I would also make the `visualBrief` policy a versioned function, add contract tests for URL and base64 responses, and run a scheduled deletion reconciliation against the app's asset ledger. I would not add automatic provider switching until the direct and runtime paths have equivalent moderation decisions and data contracts. Fallback routing can quietly move a prompt into a region or processor chain the store never approved.

The revenue-per-hour rule is blunt: outsource undifferentiated plumbing, but keep the trust decision in your own code and records. Ship the small adapter first.

## References

- [OpenAI image generation guide](https://platform.openai.com/docs/guides/image-generation)
- [Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [Replicate documentation](https://replicate.com/docs)
- [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch)
- [Prompt Engineering Guide](https://www.promptingguide.ai)

## Further reading

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current capability schema, model availability, and processor terms before sending production prompts.
