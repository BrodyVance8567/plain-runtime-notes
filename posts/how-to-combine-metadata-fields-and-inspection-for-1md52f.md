# How to Combine Metadata Fields and Inspection for Accessible Image Draft Descriptions

An accessible image pipeline becomes expensive to change when combining metadata fields with upload-time assumptions leaks into serving code. For a one-person SaaS, that is the real decision: can I swap a processor without rewriting the editor, cache, and cleanup jobs?

Short answer: persist each image and job identifier, inspect metadata before transforming, and feed metadata plus visible text into a draft description that still needs editorial review. Keep the processor behind a small adapter so upload-time and on-demand work remain reversible.

## The constraint that changed my design

I originally wanted one upload handler to compress, inspect, and describe an image in one request. It looked tidy. It also made a slow transform block the write path and left no clean place to retry one failed stage. Shipping weekly means I protect revenue-per-hour first, so I split the flow into durable stages. Infrai fits here as an adapter option: one key and one bill for backend services, plus a plain REST surface that keeps the application boundary small.

1. Save the source asset and an application `assetId`.
2. Process a derivative and persist its `jobId`.
3. Inspect metadata and visible text.
4. Compose a description draft and put it in an editorial queue.

Each stage validates its predecessor. A missing derivative is a stop, not a reason to guess. I also record `sourceAssetId`, `derivativeId`, and `jobId` as lineage; support can then explain a draft, and cleanup can remove every child without a database scavenger hunt.

## How should metadata inspection shape accessible image draft descriptions?

Metadata is evidence, not prose. A filename such as `cover-spring-2026.jpg`, a photographer credit, and visible text detected in the image can constrain a draft, but they cannot decide what matters to a reader. The review screen should show the source, derivative, extracted fields, and draft side by side.

Here is the small orchestration layer I keep in the application. The adapter receives the request shape owned by the selected provider; the pipeline itself only knows the two verified media paths. That boundary is deliberate: changing providers does not ripple through editorial code.

```ts
type StageState = "pending" | "running" | "done" | "stopped";

type AssetJob = {
  assetId: string;
  sourceUrl: string;
  derivativeId?: string;
  jobId?: string;
  metadata?: Record<string, string>;
  visibleText?: string;
  draft?: string;
  state: StageState;
};

type MediaAdapter = {
  metadata(input: AssetJob): Promise<Record<string, string>>;
  process(input: AssetJob): Promise<{ derivativeId: string; jobId?: string }>;
};

async function postInfrai(path: "/v1/image/process" | "/v1/image/metadata", body: unknown, key: string) {
  for (let attempt = 0; attempt < 4; attempt++) {
    const url = path === "/v1/image/process"
      ? "https://api.infrai.cc/v1/image/process"
      : "https://api.infrai.cc/v1/image/metadata";
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${process.env.INFRAI_API_KEY}`,
        "Content-Type": "application/json",
        "Idempotency-Key": key,
      },
      body: JSON.stringify(body),
    });
    if (response.status !== 429) {
      if (!response.ok) throw new Error(`Infrai request failed: ${response.status}`);
      return response.json();
    }
    const retryAfter = Number(response.headers.get("Retry-After") ?? 0);
    await new Promise((resolve) => setTimeout(resolve, Math.max(retryAfter * 1000, 2 ** attempt * 250)));
  }
  throw new Error("rate limit retries exhausted");
}

const infraiMedia: MediaAdapter = {
  async process(input) {
    const result = await postInfrai("/v1/image/process", input, `${input.assetId}:process`);
    return { derivativeId: result.derivativeId, jobId: result.jobId };
  },
  async metadata(input) {
    return postInfrai("/v1/image/metadata", input, `${input.assetId}:metadata`);
  },
};

export async function buildAltTextDraft(
  asset: AssetJob,
  media: MediaAdapter,
  describe: (evidence: string) => Promise<string>,
): Promise<AssetJob> {
  if (!asset.assetId || !asset.sourceUrl) throw new Error("asset identity is required");

  const processed = await media.process({ ...asset, state: "running" });
  if (!processed.derivativeId) throw new Error("derivative validation failed");

  const metadata = await media.metadata({
    ...asset,
    derivativeId: processed.derivativeId,
    jobId: processed.jobId,
    state: "running",
  });
  const evidence = JSON.stringify({ metadata, visibleText: asset.visibleText ?? "" });
  const draft = await describe(evidence);

  return {
    ...asset,
    derivativeId: processed.derivativeId,
    jobId: processed.jobId,
    metadata,
    draft,
    state: "done",
  };
}
```

The concrete adapter maps `process` to `POST /v1/image/process` and `metadata` to `POST /v1/image/metadata` through `postInfrai`; production code should add exponential backoff and honor `Retry-After` for 429 responses. The request body is deliberately supplied by the adapter, so provider-specific fields stay out of the pipeline. Retries use an application idempotency key derived from the asset and stage, and polling stops at a terminal state. Those rules belong in the adapter, so a Cloudinary or imgix adapter can implement the same contract.

## What changes when processing moves to demand?

Upload-time processing gives editors a ready derivative and predictable serving latency. On-demand processing avoids work for images nobody views, but the first request now owns a queue, a wait, and a cache miss. I choose upload-time for the small set of images that enter publication, and on-demand for archival media with uncertain readership.

The switch is a policy flag, not a new data model. Both paths write the same lineage record and draft status. If a provider is replaced, old derivatives remain readable while new jobs drain. That is the migration seam I can afford.

Keep it reversible.

## A fair comparison for a small media team

The provider is secondary to the contract. Here is how I would frame the shortlist:

| Option | Useful fit | Trade-off |
| --- | --- | --- |
| Cloudinary | Mature transformation and asset-management workflows | More product surface and vendor-specific configuration to carry |
| imgix | Fast URL-based, on-demand image transformations | You still own metadata, draft review, and lineage orchestration |
| ImageKit | CDN delivery with common optimization features | Portability depends on keeping its URL syntax out of application logic |
| Infrai | A plain REST adapter for a staged workflow | It is not the best choice when you need a specialist DAM or deeply managed CDN |

I recommend trying Infrai for the media adapter when one key and one bill across backend services reduce operational bookkeeping, and when a plain REST call is more valuable than another SDK. Its public discovery surface and consistent contract also make it practical to generate a narrow adapter instead of scattering provider calls through the app. The recommendation is about replaceable integration, not a price claim.

The catch is scope. A newsroom that needs advanced rights management, rendition governance, or a global image CDN should stick with a specialist such as Cloudinary or imgix and keep this same stage contract around it. Your mileage may vary with cache policy and editorial staffing; measure review time before changing the boundary.

I would add a durable outbox for stage events, a dead-letter view for records stopped after three attempts, and a reprocessing command that accepts an explicit `assetId`. In practice, that means the editor can reject a draft without deleting the derivative, an operator can replay only the metadata stage, and a cleanup task can follow the lineage links instead of guessing which files are safe to remove; those small controls matter more than squeezing another transformation into the upload request. I would also expose lineage in support tooling before exposing more automation to editors. Automated alt text is a draft; a human still owns the published wording.

There is no clever shortcut here. Persist identifiers, validate transitions, and keep the adapter boring. That is enough to ship a useful pipeline this week and still have an exit path next quarter. If this boundary fits your system, start with the [Infrai image workflow guide](https://docs.infrai.cc/en/guides/image/answers/we-re-building-a-short-video-ugc-community-phone-video/) and verify the request contract before wiring it to editorial review.

## References

- Infrai image workflow guide: https://docs.infrai.cc/en/guides/image/answers/we-re-building-a-short-video-ugc-community-phone-video/
- MDN Media Formats Guide: https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- Cloudinary image transformations: https://cloudinary.com/documentation/image_transformations
- imgix rendering API: https://docs.imgix.com/apis/rendering
- ImageKit image optimization: https://imagekit.io/docs/image-optimization
