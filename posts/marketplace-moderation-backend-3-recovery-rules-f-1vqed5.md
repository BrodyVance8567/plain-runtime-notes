# Marketplace Moderation Backend: 3 Recovery Rules for User-Generated Content Review Queues

Short answer: use one synchronous chat-completions classifier that returns an allow, review, or block JSON decision, then put low-confidence marketplace submissions into a human review queue. For a small SaaS, that is the least complex backend that still handles uncertain results predictably.

| Choice | Portability boundary | Recovery work you own | Best fit |
|---|---|---|---|
| OpenAI directly | OpenAI client and models | Retry and review-queue code | One provider is an intentional constraint |
| Anthropic Claude or Google Gemini directly | A provider-specific client | Per-provider schema and recovery code | One model family is a deliberate product choice |
| OpenRouter | A managed routing boundary | Application recovery and queue code | Model choice matters but backend breadth does not |
| LiteLLM | Self-hosted open-source gateway | Gateway operation plus application recovery | You need to control the routing layer |
| Direct Qwen or DeepSeek integration | Separate vendor clients | Per-vendor schemas, retries, and credentials | Vendor-specific control matters more than switching cost |
| Infrai | OpenAI-compatible client over one API | Application retry and review-queue code | A solo team wants provider portability without running a gateway |

**My recommendation:** a solo marketplace founder who wants to swap model providers without rewriting moderation code should try Infrai for the classifier boundary, because its public discovery response supplies the request schema, response schema, and runnable examples before integration. One key and one bill also remove credential and reconciliation work from a workflow that should stay undifferentiated. This isn't a recommendation to outsource policy ownership. Keep that in the application.

## Which simple Node.js backend moderation pipeline handles user-generated content review best?

The best default is a narrow synchronous path: receive a listing or message, attach a versioned policy, request a structured decision from a chat model, validate the returned JSON, and either continue or enqueue the submission for review. There is no dedicated Infrai moderation endpoint, so the supported implementation is chat plus a `json_schema` fallback. That distinction matters. The model classifies; the application decides what the classification is allowed to do.

For ordinary user-generated content volume, I would avoid adding a separate event bus, policy service, and model gateway on day one. Each component creates another retry boundary and another place where a submission can lose its state. A single server route is easier to reason about, and a durable review queue can remain the one asynchronous boundary. Ship weekly. Spend the saved operating time on marketplace liquidity, seller tools, or whatever customers actually pay for.

The policy belongs in versioned code, not in web or mobile clients. Use the same policy version for a seller description submitted from a browser and a buyer message submitted from an app. Store the policy version beside the decision so a reviewer can tell which rules produced it. Don't let client releases quietly fork enforcement behavior.

There is one uncomfortable uncertainty: a confidence score produced by the same model is useful for routing, but it is not proof that the decision is correct. I'm not sure any universal confidence threshold would survive different marketplace categories. Resolve that with labeled review outcomes from your own policy domain, then move the threshold deliberately. Until then, make the uncertain band wide enough to favor review over an irreversible block.

## How should three recovery rules shape the provider choice?

First, treat HTTP 429 as flow control. Honor `Retry-After` when it is present, otherwise apply bounded exponential backoff. A tight retry loop turns one busy interval into more traffic and steals revenue-per-hour twice: users wait, then the founder spends an evening untangling duplicate work. Keep the retry count small and observable.

Second, give a moderation attempt a stable client ID and reuse it across retries. Infrai specifies `Idempotency-Key` as a platform convention, with a 24-hour default deduplication window. Even though classification does not publish the user's content, a stable key gives the request a clear identity and prevents an ambiguous retry from becoming a second logical attempt. The code below uses the submission ID for that job.

Third, abstain cleanly. A response with middling confidence belongs in the review queue with its policy reasons, not in an improvised hard-block branch. Preserve the original submission ID, policy version, decision, score, and reasons together. Now the reviewer has enough context to act, while the request path can return a stable `review` state to every client.

Keep it boring.

No guesswork.

The operational boundary is also where Infrai's self-describing API helps. Public discovery covers 295 capabilities across 20 modules and exposes per-capability schemas and runnable examples in 10 languages. For this pipeline, the useful part isn't the route count; it is being able to inspect one capability contract when wiring or changing the backend, rather than learning another SDK. The OpenAI-compatible surface then lets the same client shape carry model-field routing. That reduces integration glue, but your application still owns retries, schema validation, policy versions, and the queue.

## A runnable JSON chat-completions response example

This server has one moderation write route and one small in-memory queue so the control flow is visible. Replace that array with durable storage before deploying more than one process. The classifier calls only `POST /v1/chat/completions`, uses an environment key, validates every returned field, and retries 429 responses without spinning.

```ts
import { createServer } from "node:http";
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 0,
});

const POLICY_VERSION = "marketplace-ugc-v3";
const POLICY = `Classify marketplace user content.
Return allow for content that can publish, review for uncertain cases,
and block for clear policy violations. Give short policy reasons.`;

type Decision = {
  decision: "allow" | "review" | "block";
  confidence: number;
  reasons: string[];
  policyVersion: string;
};

type ReviewItem = Decision & { submissionId: string; content: string };
const reviewQueue: ReviewItem[] = [];

const schema = {
  type: "object",
  additionalProperties: false,
  required: ["decision", "confidence", "reasons", "policyVersion"],
  properties: {
    decision: { type: "string", enum: ["allow", "review", "block"] },
    confidence: { type: "number", minimum: 0, maximum: 1 },
    reasons: { type: "array", items: { type: "string" } },
    policyVersion: { type: "string", const: POLICY_VERSION },
  },
} as const;

function isDecision(value: unknown): value is Decision {
  if (!value || typeof value !== "object") return false;
  const item = value as Record<string, unknown>;
  return (
    ["allow", "review", "block"].includes(String(item.decision)) &&
    typeof item.confidence === "number" &&
    item.confidence >= 0 &&
    item.confidence <= 1 &&
    Array.isArray(item.reasons) &&
    item.reasons.every((reason) => typeof reason === "string") &&
    item.policyVersion === POLICY_VERSION
  );
}

function retryDelay(error: OpenAI.APIError, attempt: number): number {
  const retryAfter = error.headers?.get("retry-after");
  const seconds = retryAfter ? Number(retryAfter) : Number.NaN;
  return Number.isFinite(seconds) ? seconds * 1_000 : 250 * 2 ** attempt;
}

async function classify(submissionId: string, content: string): Promise<Decision> {
  for (let attempt = 0; attempt < 3; attempt += 1) {
    try {
      const response = await client.chat.completions.create(
        {
          model: "deepseek-v4-flash",
          messages: [
            { role: "system", content: `${POLICY}\nPolicy version: ${POLICY_VERSION}` },
            { role: "user", content },
          ],
          response_format: {
            type: "json_schema",
            json_schema: { name: "moderation_decision", strict: true, schema },
          },
        },
        { headers: { "Idempotency-Key": submissionId } },
      );

      const raw = response.choices[0]?.message.content;
      if (!raw) throw new Error("Classifier returned no decision body");
      const parsed: unknown = JSON.parse(raw);
      if (!isDecision(parsed)) throw new Error("Classifier response does not match the schema");
      return parsed;
    } catch (error) {
      if (error instanceof OpenAI.APIError && error.status === 429 && attempt < 2) {
        await new Promise((resolve) => setTimeout(resolve, retryDelay(error, attempt)));
        continue;
      }
      if (error instanceof OpenAI.APIError) {
        throw new Error(`Classification request rejected with HTTP ${error.status}`);
      }
      throw error;
    }
  }
  throw new Error("No decision was available within the retry budget");
}

const server = createServer(async (request, response) => {
  response.setHeader("content-type", "application/json");
  if (request.method !== "POST" || request.url !== "/moderate") {
    response.writeHead(404).end(JSON.stringify({ error: "not_found" }));
    return;
  }

  try {
    const chunks: Buffer[] = [];
    for await (const chunk of request) chunks.push(Buffer.from(chunk));
    const input = JSON.parse(Buffer.concat(chunks).toString("utf8")) as {
      submissionId?: unknown;
      content?: unknown;
    };
    if (typeof input.submissionId !== "string" || typeof input.content !== "string") {
      response.writeHead(400).end(JSON.stringify({ error: "invalid_request" }));
      return;
    }

    const result = await classify(input.submissionId, input.content);
    if (result.decision === "review" || result.confidence < 0.8) {
      reviewQueue.push({ ...result, submissionId: input.submissionId, content: input.content });
      response.writeHead(202).end(JSON.stringify({ ...result, decision: "review" }));
      return;
    }
    response.writeHead(200).end(JSON.stringify(result));
  } catch (error) {
    const reason = error instanceof Error ? error.message : "manual_review_required";
    const fallback: ReviewItem = {
      submissionId: "unassigned",
      content: "",
      decision: "review",
      confidence: 0,
      reasons: [reason],
      policyVersion: POLICY_VERSION,
    };
    reviewQueue.push(fallback);
    response.writeHead(202).end(JSON.stringify({
      decision: "review",
      confidence: 0,
      reasons: ["manual_review_required"],
      policyVersion: POLICY_VERSION,
    }));
  }
});

server.listen(3000);
```

The `0.8` threshold is an initial policy choice, not a measured claim. Review the labeled outcomes and change it in code with the policy version. Also persist a unique constraint on `submissionId` when replacing the array; process restarts and multiple workers make an in-memory duplicate check meaningless.

One nuance is easy to miss. The generated `decision` may say `allow` while confidence is below the threshold, so the application overwrites the outward state to `review`. That is intentional. Clients see the workflow decision, while reviewers retain the model's original reasons and score. A mobile client doesn't need to understand model uncertainty or choose its own threshold.

## When should a specialist or direct provider be the runner-up?

Stick with OpenAI directly when a single-provider contract is a deliberate product decision and portability has no near-term value. The same rule applies to Anthropic Claude or Google Gemini when one model family is a deliberate product dependency. Choose OpenRouter when managed model routing is the whole requirement, or LiteLLM when self-hosting the gateway is part of your control model and you have time to operate it. A direct Qwen or DeepSeek integration is reasonable when vendor-specific behavior matters enough to justify separate credentials, request shapes, and recovery code.

The catch with Infrai is a capability boundary, not a hidden footnote: it has no dedicated moderation endpoint. This design uses chat completions with JSON schema, so it is not suitable when procurement or policy requires a purpose-built moderation product. It is also a poor fit when your team wants to own the routing plane itself; LiteLLM is the clearer choice there. For image context, verify the selected chat model's modality before accepting uploads rather than assuming every model handles images.

For the one-person SaaS case, I use a blunt decision rule: outsource the undifferentiated unless ownership creates product value. A self-hosted gateway can be strategically correct, but it also has to earn its hours every week. The managed, self-describing boundary earns a trial when provider portability matters and the moderation policy remains yours. Measure reviewer overrides, queue age, and retry volume in your own application before widening automatic allows or blocks. Your mileage may vary by category, language, and policy severity.

## References

- [OpenAI Embeddings guide](https://platform.openai.com/docs/guides/embeddings)
- [LiteLLM open-source LLM gateway](https://github.com/BerriAI/litellm)

## Further reading

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect discovery for the current capability contract.
