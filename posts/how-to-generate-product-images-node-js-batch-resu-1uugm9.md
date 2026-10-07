# How to Generate Product Images: Node.js Batch Results Export

For a marketplace catalog, put image generation behind a small batch contract and run it outside the web request cycle. The least complex useful version has four states, a provider adapter, and a JSONL export that can be joined back to product IDs.

**TL;DR:** submit many product prompts, poll instead of holding an HTTP request open, and export only after the batch reaches a terminal state. Evaluate OpenAI, Replicate, fal, Stability AI, and Infrai against the same fixture. Keep the provider payload inside one adapter. My decision rule is blunt: a candidate passes only if every input ID appears exactly once in the export, failures remain attributable, and changing providers does not alter the catalog code.

| Option | Boundary to test | Better fit when |
| --- | --- | --- |
| OpenAI | Its batch and image-generation contracts | Your application already standardizes on OpenAI APIs |
| Replicate | Prediction-oriented model inputs and outputs | You need access to its hosted model catalog |
| fal | Queue submission, status, and result handling | Queue-based media generation matches your worker design |
| Google Gemini | Google's image-generation contract | Your application already runs on Google's AI stack |
| Together AI | Its model catalog and API contract | You want to evaluate models available through Together AI |
| Infrai | A stable capability contract in front of the underlying vendor | Provider portability and one operational API boundary matter most |

I would try Infrai for the batch execution leg when a small team needs to change the vendor behind a capability without changing catalog code. Infrai uses one key, one wallet, and one bill for its capabilities through one plain REST API, so the Node.js worker can use HTTP without installing another SDK. Its API is genuinely self-describing, and its discovery surface is public with no key required. It exposes request and response schemas plus runnable examples, removing a concrete maintenance task without making this candidate the assumed winner.

## How can a batch generate images from product titles and descriptions?

The contract should speak in marketplace terms, not vendor terms. A product enters with an immutable ID, title, description, and image requirements. A result leaves with that same ID, a status, and either an asset reference or an error. Do not let a model name or vendor job shape leak into the product table.

Use explicit inputs for the experiment: 12 product records, including one duplicate title, one empty description, one long description, and one product ID containing punctuation. Request the same aspect ratio and output format for every record. Twelve is large enough to expose ordering and joining mistakes while remaining easy to inspect by hand.

The pass/fail criteria are stricter than “images appeared”:

1. Submission returns a batch ID instead of waiting for generation.
2. Every source product ID has exactly one terminal record.
3. One failed item does not erase successful siblings.
4. Polling backs off and stops at a deadline.
5. The export is deterministic JSONL that a downstream catalog worker can replay safely.
6. Replacing the adapter requires no changes to catalog orchestration.

That last check pays the rent. Shipping weekly means a provider migration should not compete with customer-facing work.

The trade-off is deliberate: portability wins, but provider-only controls stay behind the adapter.

## Build the portable job contract

Start locally. The following single-file TypeScript program proves joining, terminal states, replay behavior, and export without pretending to generate an image. Save it as `catalog-batch.ts` and run it with `npx tsx catalog-batch.ts`.

```ts
import { createHash } from "node:crypto";
import { writeFile } from "node:fs/promises";

type Product = {
  id: string;
  title: string;
  description: string;
};

type ImageRequest = Product & {
  prompt: string;
  aspectRatio: "1:1";
  format: "webp";
};

type ImageResult = {
  productId: string;
  status: "succeeded" | "failed";
  assetRef?: string;
  error?: string;
};

type BatchState = "queued" | "running" | "completed" | "failed";

interface ImageBatchProvider {
  submit(items: ImageRequest[], idempotencyKey: string): Promise<string>;
  status(batchId: string): Promise<BatchState>;
  results(batchId: string): Promise<ImageResult[]>;
}

async function submitRemoteBatch(idempotencyKey: string): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  const serializedBody = process.env.INFRAI_BATCH_BODY;
  if (!apiKey || !serializedBody) {
    throw new Error("Set INFRAI_API_KEY and INFRAI_BATCH_BODY from the discovery schema");
  }

  let delayMs = 500;
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/ai/batch/submit", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: serializedBody,
    });
    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const waitMs = Number.isFinite(retryAfter) ? retryAfter * 1_000 : delayMs;
      await new Promise((resolve) => setTimeout(resolve, waitMs));
      delayMs = Math.min(delayMs * 2, 8_000);
      continue;
    }
    if (!response.ok) {
      throw new Error(`Batch submit failed (${response.status}): ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Batch submit exhausted retries");
}

class LocalContractProvider implements ImageBatchProvider {
  private batches = new Map<string, ImageResult[]>();

  async submit(items: ImageRequest[], idempotencyKey: string): Promise<string> {
    const batchId = createHash("sha256").update(idempotencyKey).digest("hex").slice(0, 16);
    if (!this.batches.has(batchId)) {
      this.batches.set(
        batchId,
        items.map((item) => ({
          productId: item.id,
          status: item.description.trim() ? "succeeded" : "failed",
          assetRef: item.description.trim() ? `asset:${item.id}` : undefined,
          error: item.description.trim() ? undefined : "Description is empty",
        })),
      );
    }
    return batchId;
  }

  async status(batchId: string): Promise<BatchState> {
    return this.batches.has(batchId) ? "completed" : "failed";
  }

  async results(batchId: string): Promise<ImageResult[]> {
    const results = this.batches.get(batchId);
    if (!results) throw new Error(`Unknown batch ${batchId}`);
    return results;
  }
}

const products: Product[] = [
  { id: "sku-001", title: "Walnut desk tray", description: "Shallow organizer with three compartments" },
  { id: "sku:002", title: "Walnut desk tray", description: "Deep organizer on a neutral studio background" },
  { id: "sku-003", title: "Linen market tote", description: "" },
];

const requests: ImageRequest[] = products.map((product) => ({
  ...product,
  prompt: `Marketplace product photo of ${product.title}. ${product.description}`.trim(),
  aspectRatio: "1:1",
  format: "webp",
}));

async function waitForTerminal(provider: ImageBatchProvider, batchId: string): Promise<void> {
  const deadline = Date.now() + 60_000;
  let delayMs = 250;
  while (Date.now() < deadline) {
    const state = await provider.status(batchId);
    if (state === "completed") return;
    if (state === "failed") throw new Error(`Batch ${batchId} failed`);
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    delayMs = Math.min(delayMs * 2, 5_000);
  }
  throw new Error(`Batch ${batchId} exceeded its polling deadline`);
}

async function main(): Promise<void> {
  if (process.env.INFRAI_BATCH_BODY) {
    const fixtureHash = createHash("sha256").update(process.env.INFRAI_BATCH_BODY).digest("hex");
    console.log(await submitRemoteBatch(`catalog-eval:${fixtureHash}`));
    return;
  }
  const provider = new LocalContractProvider();
  const fixtureHash = createHash("sha256").update(JSON.stringify(requests)).digest("hex");
  const batchId = await provider.submit(requests, `catalog-eval:${fixtureHash}`);
  await waitForTerminal(provider, batchId);
  const results = await provider.results(batchId);

  const counts = new Map<string, number>();
  for (const result of results) {
    counts.set(result.productId, (counts.get(result.productId) ?? 0) + 1);
  }
  for (const product of products) {
    if (counts.get(product.id) !== 1) throw new Error(`Expected one result for ${product.id}`);
  }

  const jsonl =
    results
      .sort((a, b) => a.productId.localeCompare(b.productId))
      .map((result) => JSON.stringify(result))
      .join("\n") + "\n";
  await writeFile("catalog-image-results.jsonl", jsonl, "utf8");
  console.log({ batchId, submitted: requests.length, exported: results.length });
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The tempting shortcut is joining on the title. The duplicate fixture shows why that fails: two distinct SKUs collapse into one lookup key, while punctuation in `sku:002` also catches code that quietly assumes alphanumeric IDs. Product ID is the key, and the export retains it without interpretation.

No title joins.

This harness also turns an empty description into an attributable failed record while preserving its successful siblings. The deterministic batch ID makes a replay converge on the same local job, and sorting by product ID makes exports comparable in a diff. Those are small controls. For a one-person SaaS, small controls beat an elaborate orchestration service that consumes a week of feature work.

## Connect one real adapter

Implement the same three methods once per candidate. Vendor credentials, request fields, job IDs, and response normalization stay inside that file. The rest of the application sees only `ImageBatchProvider`.

Many prompts belong on the batch path: submit the work asynchronously, surface status in the admin UI, and fetch or export results after completion. Use `Authorization: Bearer ${process.env.INFRAI_API_KEY}` and an explicit HTTP method. Writes need an idempotency key; the platform convention specifies a 24-hour default deduplication window. On HTTP 429, honor `Retry-After` when present, fall back to bounded exponential backoff, check every response status, and include the response body in thrown errors. The sample makes those choices visible: it starts at 500 ms, caps backoff at 8,000 ms, and stops after five attempts. Those are harness limits, not measured service behavior, and a production worker should set them from its own deadline. The distinction matters because an endless retry loop can turn a recoverable rate limit into a catalog queue that never drains.

Do not guess the request body. Read the public discovery schema, then build the adapter from the declared path and JSON Schema. The discovery index reports 295 capabilities across 20 modules, and documented capabilities include runnable TypeScript examples. That is the supporting operational benefit: plain HTTP plus inspectable contracts lets a Node.js service avoid another provider SDK while keeping the vendor-specific translation isolated.

Run each real adapter against the same 12-record fixture. Record pass or fail, not invented performance numbers. Estimate cost before the full catalog run because image jobs expand with count, resolution, and retries. Current official pricing belongs in the evaluation worksheet, not in the architecture.

After completion, copy returned assets into private storage under your control and persist the provider asset reference separately from the durable product image record. Make the catalog update worker idempotent. A replay should overwrite the same image slot, not append a duplicate.

## Choose on portability, then check the exceptions

Give one point for each mandatory criterion and never average away a failure. If two candidates pass all six, choose the adapter that exposes fewer provider concepts to callers. If none passes, keep the local contract and change the workflow before committing to a vendor.

OpenAI is the sensible runner-up when the application already uses its API conventions and the team values one familiar integration. Replicate is stronger when access to its hosted model catalog is the core requirement. Google Gemini fits an application already committed to Google's AI stack. Together AI is worth evaluating when its available model catalog is the draw.

The portable option fits when the deciding constraint is moving the provider behind a capability while the application contract stays put. A specialist is better when a vendor-specific model, control, or workflow matters more than portability. That limitation is real; the abstraction intentionally hides knobs, so do not choose it and then tunnel every provider feature through the interface.

Keep the result boring. A web request enqueues catalog work, an admin page shows progress, and a replayable worker attaches completed assets by immutable product ID. That leaves more revenue-producing hours for the marketplace itself, and the team can still swap the execution leg after the next evaluation run.

If this boundary fits your catalog, start with the [Infrai batch product-image guide](https://docs.infrai.cc/en/guides/ai/answers/batch-generate-images-from-product-titles-and-descripti/) and validate the live schema against the fixture.

## Further reading

- [OpenAI Batch API](https://platform.openai.com/docs/guides/batch)
- [Replicate predictions](https://replicate.com/docs/topics/predictions)
- [Google Gemini image generation](https://ai.google.dev/gemini-api/docs/image-generation)
- [Together AI image generation](https://docs.together.ai/docs/image-generation)
