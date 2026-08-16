# Customer Inbox Gateway API — One Key, Rate Limits, Fallback Routing Across 2 Regions

Short answer: choose a one-key gateway API only if one stable operation identity survives client retries, rate limits, fallback routing, and cancellation across Europe and the US.

For a one-person developer-tools SaaS, the deciding constraint is not how short the setup guide looks. It is whether the same support ticket can be processed twice when a browser retries while the primary provider is being rate-limited. OpenAI, Claude, and Gemini can sit behind one credential, but that credential is merely authentication at the gateway boundary. It does not define request identity, preserve regional policy, or decide which interrupted request is safe to replay.

That changes the selection exercise. I would spend one shipping cycle building a thin concurrency harness, then keep the gateway only if its event trail reconciles with the application ledger. Don't start with a feature matrix. Start with one ticket delivered twice.

## How can one gateway API use fallback routing without duplicate tickets?

Treat request admission, fallback routing, and accounting as one state machine. A ticket arrives with a tenant ID, a residency class, an operation ID, and a budget policy. The application claims that operation before calling an upstream. The router selects an eligible provider family, records the attempt, and either returns a structured triage result or evaluates a narrowly defined fallback condition. A repeated delivery joins the existing operation instead of opening another one. The customer-facing API should not need to know which upstream handled the work.

The hard part is classification. A capacity signal can justify trying another eligible upstream. An invalid request cannot. An authentication failure cannot. A client disconnect should not quietly become a second billable attempt. If the gateway flattens all of these into a generic error, simple setup has purchased complicated incident response.

Keep two limits separate. The upstream limit protects a provider account or model pool. The tenant limit protects the economics of the SaaS. A noisy tenant can still consume the shared upstream allowance while remaining under a poorly chosen local cap, so the local policy needs both a tenant dimension and a provider dimension. This is where one key helps operationally — there is one credential to rotate in the application — but it must not collapse those dimensions in telemetry.

Retries need a budget too. For support triage, I allow at most one alternate-provider attempt in the policy model below. That is a design choice, not an industry constant. A second fallback might improve completion in some workloads, but it also makes latency, duplicate work, and cost attribution harder to reason about. Your mileage may vary; a replay test with representative payload sizes and real quota responses is what would settle it.

One subtle failure mode matters more than it first appears: a successful fallback can erase the evidence of the failed primary attempt. The user sees a category and priority, so the workflow looks healthy, while the operator sees only the final provider and assigns the whole cost to one call. The ledger must retain both attempts under the same operation ID. Otherwise per-tenant margin is fiction.

Use a deterministic drill. Give the primary adapter a `rate_limited` result after it records 800 input units and zero output units, then let the alternate adapter return a valid category after 800 input units and 24 output units. While that alternate is running, deliver the original ticket again with the same operation ID. The second delivery must not start a third provider attempt. Next, cancel the waiting client and verify that cancellation propagates to work that no caller or durable workflow still needs. The final ledger should explain one logical operation, two ordered upstream attempts, 1,600 input units, 24 output units, and the tenant responsible for them. This is not a performance benchmark. It is a concurrency contract that can pass or fail without a subjective score.

One ticket. Two deliveries. One result.

## Per-tenant accounting is the concurrency lock

The gateway adapter should return normalized outcomes, not provider-shaped exceptions. That keeps policy in one place and makes the test harness independent of any commercial SDK. The example is intentionally small, but it carries the fields needed to answer who spent what, where, and why.

```ts
type Region = "eu" | "us";
type Provider = "openai" | "claude" | "gemini";
type Outcome = "ok" | "rate_limited" | "invalid" | "auth_failed" | "cancelled";

type Ticket = {
  tenantId: string;
  operationId: string;
  region: Region;
  subject: string;
  body: string;
};

type Triage = {
  category: string;
  priority: "low" | "normal" | "high";
};

type Attempt = {
  tenantId: string;
  operationId: string;
  region: Region;
  provider: Provider;
  sequence: number;
  outcome: Outcome;
  inputUnits: number;
  outputUnits: number;
  durationMs: number;
};

type AdapterResult =
  | { outcome: "ok"; value: Triage; usage: Pick<Attempt, "inputUnits" | "outputUnits"> }
  | { outcome: Exclude<Outcome, "ok">; usage: Pick<Attempt, "inputUnits" | "outputUnits"> };

type ProviderAdapter = {
  run(provider: Provider, ticket: Ticket, signal: AbortSignal): Promise<AdapterResult>;
};

type Ledger = {
  append(attempt: Attempt): Promise<void>;
};

async function triageTicket(
  ticket: Ticket,
  providers: readonly Provider[],
  adapter: ProviderAdapter,
  ledger: Ledger,
  signal: AbortSignal,
): Promise<Triage> {
  for (const [index, provider] of providers.slice(0, 2).entries()) {
    const startedAt = Date.now();
    const result = await adapter.run(provider, ticket, signal);

    await ledger.append({
      tenantId: ticket.tenantId,
      operationId: ticket.operationId,
      region: ticket.region,
      provider,
      sequence: index + 1,
      outcome: result.outcome,
      inputUnits: result.usage.inputUnits,
      outputUnits: result.usage.outputUnits,
      durationMs: Date.now() - startedAt,
    });

    if (result.outcome === "ok") return result.value;
    if (result.outcome !== "rate_limited") {
      throw new Error(`Triage stopped after ${result.outcome}`);
    }
  }

  throw new Error("Triage capacity exhausted");
}
```

There is no automatic retry loop hidden in the adapter. Good. The policy permits fallback only after a normalized `rate_limited` outcome, and every attempt is appended before the next decision. The stable `operationId` joins the records; `sequence` preserves their order. In production, the ledger write also needs a uniqueness rule on the operation, provider, and sequence tuple so a repeated delivery cannot create a second accounting row.

The adapter contract leaves unit names generic because different capability families do not necessarily report usage the same way. The gateway can normalize those units for analysis, but the raw response metadata should remain available in a restricted audit record. I'm not sure a universal normalized unit is useful once a support workflow adds reranking or speech. Cohere documents reranking as a separate search-quality operation, while ElevenLabs documents speech capabilities; both are reminders that a future “AI bill” may contain unlike work. The evidence needed to resolve the schema is a week of actual operation-level usage, not a prettier interface.

## Backpressure belongs before provider choice

At low volume, an append-only database table and a daily reconciliation query are enough. At higher volume, put a tenant-aware admission queue before provider selection, move attempt events through durable delivery, make ledger consumers idempotent, and separate operational retention from billing retention. Preserve the operation ID across all of them. A queue admission decision needs a reason code and policy version just like a provider attempt, because work rejected before an upstream call still explains the customer experience. Add alerts for a rising fallback ratio by tenant, region, and provider; a global average can hide a single customer's bad week.

I would also replace the fixed provider array with policy data: eligible providers by region, tenant tier, data class, and operation. Deploy policy changes separately from application code, but version every decision so an old ticket can be explained later. Ship weekly, yet make routing changes boring: test fixtures, a dry evaluation against recorded metadata, and an immediate rollback to the prior policy version.

Don't put raw support text into cost events. The accounting path needs identifiers, usage, timing, outcome, and policy version. Content belongs in the smallest system that actually needs it, with its own retention rules. This separation lowers the blast radius of analytics access and keeps financial queries fast.

The larger design needs concurrency controls before it needs smarter routing. Queue depth, tenant fairness, and cancellation propagation determine how much useless work enters the system. A sophisticated fallback score cannot repair a queue that continues an upstream request after the customer has abandoned it.

## Comparing three ownership boundaries

The same operation contract can sit behind three ownership boundaries. They do not create the same work for a solo operator.

| Boundary | Application owns | Main fit | Main limitation |
| --- | --- | --- | --- |
| Managed gateway | Tenant admission, operation identity, reconciliation | One credential and less adapter maintenance | Provider-specific cancellation controls or metadata may be abstracted |
| Self-hosted gateway | Routing runtime, credentials, upgrades, policy, ledger | Strict control over policy and event retention | The operator owns deployment and on-call work |
| Direct adapters | Each provider integration and the shared contract | Provider-native controls and audit fields | Rotation, normalization, and fallback stay in application scope |

A managed boundary is suitable when one credential, consistent adapters, regional eligibility controls, and attempt-level telemetry remove enough undifferentiated work to protect the weekly shipping cadence. If a regulated tenant requires direct contractual control, provider-native audit fields, or a data path the gateway cannot document, the direct boundary behind an internal adapter is the clearer fit.

Self-hosting the routing layer offers tighter control over policy and logs, but it transfers credential rotation, quota coordination, upgrades, and on-call responsibility to the same person building customer features. That can be rational at sufficient scale or under strict compliance requirements. For a small developer-tools business, revenue per engineering hour is the cleaner decision rule: outsource the undifferentiated only when the resulting event trail remains auditable.

Provider families should remain replaceable upstream candidates. The acceptable design preserves one operation identity through duplicate delivery and fallback in both regions, then makes every attempt explainable at the tenant boundary. If no implementation does, keep the adapter and ledger, reduce the provider set, and postpone the one-key migration.

## References

- https://docs.cohere.com/docs/rerank-overview
- https://elevenlabs.io/docs
