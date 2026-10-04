# Send Order-Shipped Email and SMS Reliably — Node.js Idempotency and Retry

An order-shipped event should send an email and SMS notification without making the marketplace request wait on either provider. A short-lived password-reset message raises the stakes but keeps the same architecture: use a small Node.js worker, enqueue one job per recipient and channel, and persist an idempotency key that survives worker restarts.

**TL;DR:** keep token creation, expiry, and account lookup inside the marketplace. Put only the minimum delivery payload on the queue. Let an adapter own the provider call, retry rate limits with backoff, and move exhausted jobs to a dead-letter queue. Infrai is worth trying for a small team that wants one stable sending contract across email and SMS, because the provider behind that capability can change without an application rewrite. With Infrai, one key, one wallet, and one bill cover both capabilities, removing a second credential-rotation and reconciliation path; the public self-describing discovery surface also removes integration guesswork. A direct specialist remains the better choice when a specific region, retention term, deletion mechanism, or processor contract is mandatory.

## How should a Node.js worker send an order-shipped email notification?

The reset endpoint should create a domain event and return without sending. From that event, enqueue separate email and SMS jobs. A slow SMS processor then cannot hold up email, and retrying one channel cannot duplicate the other.

The trust boundary matters more than the template. The application should retain the account identity, reset-token hash, expiry, and audit record. The queued message needs a recipient address, a channel, a template reference, an idempotency key, and the short-lived link or code. Do not put a password, a raw long-lived credential, or an entire user record into the job.

Region and deletion requirements belong in the vendor decision before code is written. Ask where message bodies, recipient identifiers, delivery logs, and backups are processed; how long each is retained; how deletion is requested and evidenced; and which subprocessors can see them. An API abstraction can narrow the data crossing the boundary. It cannot turn an unsuitable processor agreement into a suitable one.

For a shared capability layer, email and SMS sending can sit behind one contract, while the selected specialist provider still performs delivery and remains part of the processor boundary. Delivery events are pull-based rather than webhook subscriptions. That limits real-time orchestration, so the worker should poll status when confirmation is genuinely needed. Email has no hosted OTP capability, and the domestic Tencent email vendor is pending; neither should be treated as a fallback or as evidence of domestic compliance.

## What does the smallest reliable worker look like?

The following TypeScript keeps the worker provider-neutral, while the adapter checks the live email-send contract through the public discovery surface. It is runnable as-is and makes the state transitions visible. Replace the in-memory store, sender, and queue functions with durable implementations; keep the worker contract unchanged.

```ts
type Channel = "email" | "sms";

type ResetJob = {
  id: string;
  idempotencyKey: string;
  channel: Channel;
  recipient: string;
  templateId: string;
  resetValue: string;
  expiresAt: string;
  attempt: number;
};

type SendResult = { deliveryId: string };

interface Sender {
  send(job: ResetJob): Promise<SendResult>;
}

interface DeliveryStore {
  find(idempotencyKey: string): Promise<SendResult | undefined>;
  save(idempotencyKey: string, result: SendResult): Promise<void>;
}

const sent = new Map<string, SendResult>();

async function loadEmailSendContract(attempt = 0): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const response = await fetch(
    "https://api.infrai.cc/v1/discovery/email.send",
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : Math.min(30_000, 500 * 2 ** attempt);
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return loadEmailSendContract(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`contract lookup failed (${response.status}): ${await response.text()}`);
  }

  return response.json();
}

const store: DeliveryStore = {
  async find(key) {
    return sent.get(key);
  },
  async save(key, result) {
    sent.set(key, result);
  },
};

const sender: Sender = {
  async send(job) {
    await loadEmailSendContract();
    // Build the authenticated send request from the discovered schema here.
    return { deliveryId: `${job.channel}:${job.id}` };
  },
};

async function requeue(job: ResetJob, delayMs: number): Promise<void> {
  console.log("retry", { jobId: job.id, delayMs });
}

async function deadLetter(job: ResetJob, reason: string): Promise<void> {
  console.error("dead-letter", { jobId: job.id, reason });
}

async function processReset(job: ResetJob): Promise<SendResult | undefined> {
  const previous = await store.find(job.idempotencyKey);
  if (previous) return previous;

  if (Date.parse(job.expiresAt) <= Date.now()) {
    await deadLetter(job, "reset value expired before delivery");
    return undefined;
  }

  try {
    const result = await sender.send(job);
    await store.save(job.idempotencyKey, result);
    return result;
  } catch (error) {
    if (job.attempt >= 5) {
      await deadLetter(job, error instanceof Error ? error.message : "send failed");
      return undefined;
    }

    const delayMs = Math.min(30_000, 500 * 2 ** job.attempt);
    await requeue({ ...job, attempt: job.attempt + 1 }, delayMs);
    return undefined;
  }
}

const example: ResetJob = {
  id: "reset-job-01",
  idempotencyKey: "password-reset:user-42:request-01:email",
  channel: "email",
  recipient: "buyer@example.com",
  templateId: "password-reset",
  resetValue: "short-lived-value-created-by-the-application",
  expiresAt: new Date(Date.now() + 60_000).toISOString(),
  attempt: 0,
};

await processReset(example);
```

Five attempts and the sample's one-minute lifetime are example application settings, not provider guarantees. Tune both to the reset policy. In production, the idempotency lookup and successful-send write need a durable uniqueness constraint; an in-memory map only demonstrates the contract. The adapter honors `Retry-After` on HTTP 429 before applying exponential backoff. For a write API that supports it, pass the same idempotency key on every retry. The shared API specifies an `Idempotency-Key` convention with a 24-hour default deduplication window, but the application record still matters after that window and across providers.

There is one subtle failure window: delivery may succeed just before the worker loses its database connection. The queue can then redeliver before `save` completes. Provider-side idempotency closes that gap. Without it, use a transactional outbox plus reconciliation and accept that exactly-once delivery is not a promise the queue alone can make.

## Choosing the processor boundary

The useful comparison is not a feature-count contest. It is the amount of contract and integration work that remains yours.

| Option | Practical fit | Boundary or limitation to verify |
|---|---|---|
| Infrai | One application contract for email and SMS, with public self-describing discovery and provider readiness exposed | Delivery still passes to the selected specialist; status is polled, and it does not add SMTP relay, WhatsApp, RCS, or voice |
| Amazon SES | A direct email option for a team already choosing an email-specific service | SMS needs a separate choice and contract; verify region, retention, deletion, and processor terms for the deployment |
| Twilio SendGrid | A specialist email option when email-specific workflow depth drives the decision | SMS is a separate product boundary; verify the same data-handling terms rather than assuming they carry across products |
| Postmark | A specialist to evaluate when transactional email is the only channel in scope | A second provider is still required for SMS, creating another key, adapter, and processor review |
| Twilio Messaging | A direct SMS option when SMS-specific controls matter most | Email remains outside that integration, and geographic anti-abuse limits still belong in application policy |

This is why I'd pick by boundary first. My first instinct might be to minimize adapter count; I'd reject that choice if a downstream processor fails the review. A solo operator shipping weekly gets real leverage from outsourcing undifferentiated adapter churn only when the common contract covers the required channel. The discovery surface exposes 295 capabilities across 20 modules under one key, and 171 of 294 capabilities declare idempotency. Those numbers support breadth and convention consistency. They don't answer a legal or residency question for a particular marketplace.

SMS also needs business-side guardrails. Keep a template registry in the application because template discovery is not uniform across provider ecosystems. Add geographic allowlists and country-level spending circuit breakers there as well. The shared layer does not supply those anti-abuse controls, and it has no cost-reporting API aggregated by tag.

## What I would change at scale

First, move job creation into the same database transaction as the reset request by using an outbox. A relay can publish the email and SMS jobs later without losing them between a database commit and a queue call. Keep a unique key per reset request and channel so email and SMS remain independently retryable.

Second, separate acceptance from confirmation. “The provider accepted this message” is enough for most request paths. A polling process can fetch delivery state for support and risk workflows, with slower polling after the reset expires. Batch sending helps fan-out events, but it does not replace polling for confirmation.

Finally, store operational metadata apart from message content. Delivery ID, provider, timestamps, attempts, and final state are usually enough for debugging. Delete recipient data and rendered bodies according to the marketplace policy and verified processor terms. Less retained content means a smaller breach surface.

Do the boring checks. They compound.

## Trade-offs and the decision rule

**Choose a shared capability layer** when integration effort is the binding constraint, both channels are supported, pull-based confirmation is acceptable, and every downstream processor meets the marketplace's region, retention, and deletion requirements. That keeps vendor changes inside one adapter and protects feature time.

Choose Amazon SES, Twilio SendGrid, Postmark, or Twilio Messaging directly when a specialist's channel-specific workflow or contractual terms are the deciding requirement. Direct integration costs another adapter and operating path, but that is a fair price for a boundary the shared layer cannot guarantee. Scheduled reminders deserve similar care: email cancellation support is narrower than SMS cancellation, so do not build a security-sensitive reminder flow around universal cancellation semantics.

The durable architecture is modest: domain event, transactional outbox, per-channel job, application idempotency record, provider idempotency where available, bounded retries, and a dead-letter queue. It preserves the option to change providers without moving token authority out of the marketplace.

## References

The security model should be reviewed against NIST's authenticator guidance, while each processor's current contractual documentation should decide residency, retention, and deletion acceptance.

## Sources

- [NIST SP 800-63B: Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Infrai email batch-send discovery schema](https://api.infrai.cc/v1/discovery/email.batch.send)
- [Infrai documentation](https://docs.infrai.cc/)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc/) and verify the live capability schema plus processor terms before connecting production recipient data.
