# Webhook Queue Push/Subscribe for Rate-Limited Node.js Consumers: ACK and NACK

Short answer: put every marketplace shipment update in a durable queue, let a rate-limited Node.js consumer push it to each subscriber's HTTPS endpoint, and ACK only after a terminal delivery result; NACK retryable failures with backoff, then move exhausted deliveries to a dead-letter queue.

| Choice | Latency | Cost and operations | Best fit |
|---|---:|---|---|
| Queue plus continuous consumers | Lowest | Pays for idle capacity or managed delivery | Shipment updates that should arrive quickly |
| Queue plus scheduled batches | Bounded by the schedule | Reduces idle compute but creates bursts | Updates where minutes of delay are acceptable |
| Database outbox plus polling workers | Depends on polling interval | Reuses the database; adds table maintenance | Atomic shipment writes with a small workload |

For marketplace fan-out, start with a durable queue and continuous consumers. The recommendation is about failure isolation and predictable latency: one slow subscriber must not hold every other shipment update. Scheduled batching is the runner-up when delivery latency is negotiable and traffic is sparse.

## What should a webhook queue push/subscribe Node.js consumer optimize first?

Optimize the latency budget and the subscriber-specific rate limit together. A global cap is easy, but it lets one busy endpoint consume the whole allowance. Key concurrency and pacing by subscriber while also imposing a lower global ceiling that protects outbound connections and memory.

The useful latency budget has separate parts: time waiting in the queue, time waiting for that subscriber's permit, HTTPS request time, and retry delay. Record them separately. An end-to-end average can look healthy while one subscriber spends most of its time behind another subscriber's backlog. Percentiles by subscriber and queue age expose that failure mode.

Cost follows utilization.

A consumer sized for a peak shipment burst sits idle between bursts; a batch worker trades that idle time for deliberate queueing delay. For a one-person SaaS, the revenue-per-hour question is blunt: will another day tuning autoscaling sell more subscriptions than a clear delivery SLO and a conservative concurrency cap? Usually, outsource undifferentiated queue operation, ship weekly, and keep the delivery contract portable. A managed component still needs an exit plan and a measured bill.

Don't confuse rate with concurrency. Ten requests in flight can finish in 100 milliseconds or 30 seconds, so a concurrency limit alone cannot enforce ten requests per second. Use a token bucket or a next-allowed-at timestamp for rate, plus a separate semaphore for in-flight work. Your mileage may vary because subscribers publish different limits; the missing input is each endpoint's documented quota and response behavior.

## How should the consumer handle ACK, NACK, and dead letters?

Treat ACK and NACK as queue state transitions, not HTTP aliases. ACK a delivery after a `2xx` response because the subscriber accepted it. NACK a timeout, connection failure, `408`, or `429`, attach the next attempt time, and retry with exponential backoff plus jitter. A permanent contract failure such as `400`, `401`, `403`, `404`, `405`, `410`, `413`, `415`, or `422` should go to the dead-letter queue without repeated delivery.

Be cautious with broad rules. A subscriber can document semantics that differ from this baseline, and I'm not sure a generic consumer can infer those semantics safely. Store a per-subscriber policy when the contract says, for example, that `409` is an accepted duplicate. Otherwise, retain the response metadata and dead-letter the ambiguous result for inspection rather than inventing success.

Delivery must be idempotent because a worker can send the HTTPS request and lose its queue lease before recording the ACK. Include a stable event ID in every attempt. The receiving endpoint should persist that ID with its business update in one transaction and return the same successful outcome for a duplicate. Exactly-once delivery is not a realistic transport promise here; idempotent effects are the practical target.

Short leases matter. Extend a lease only while the request is active, put a hard timeout on the request, and never ACK before reading the response. On worker shutdown, stop leasing new jobs, wait briefly for active requests, then release unfinished leases.

Ack last.

A dead-letter record needs enough evidence to replay safely: event ID, subscriber ID, destination identifier, attempt count, first and last attempt timestamps, final status or error class, and a payload reference. Avoid storing authorization headers. Replay should create a new delivery attempt linked to the old record, not erase the audit trail.

## A focused TypeScript consumer

The example keeps transport, queue, and rate policy behind interfaces. Its endpoint comes from trusted subscriber configuration, not from the event payload; accepting arbitrary destinations would turn the worker into an SSRF proxy. The queue implementation must lease atomically so two workers cannot own the same available delivery. A relational implementation can select due rows with `FOR UPDATE SKIP LOCKED`, update their lease in the same transaction, and let competing workers skip locked rows.

```ts
interface Delivery {
  id: string;
  eventId: string;
  subscriberId: string;
  endpoint: URL;
  payload: unknown;
  attempt: number;
}

interface Queue {
  lease(signal: AbortSignal): Promise<Delivery | null>;
  ack(deliveryId: string): Promise<void>;
  retry(deliveryId: string, availableAt: Date, reason: string): Promise<void>;
  deadLetter(deliveryId: string, reason: string): Promise<void>;
}

interface RateGate {
  wait(subscriberId: string, signal: AbortSignal): Promise<void>;
}

const permanentStatuses = new Set([
  400, 401, 403, 404, 405, 410, 413, 415, 422,
]);

function retryAt(attempt: number): Date {
  const capMs = 15 * 60_000;
  const baseMs = Math.min(capMs, 1_000 * 2 ** attempt);
  const jitterMs = Math.floor(Math.random() * Math.max(1, baseMs / 4));
  return new Date(Date.now() + baseMs + jitterMs);
}

async function deliver(
  queue: Queue,
  rates: RateGate,
  delivery: Delivery,
): Promise<void> {
  await rates.wait(delivery.subscriberId, AbortSignal.timeout(30_000));

  try {
    const response = await fetch(delivery.endpoint, {
      method: "POST",
      headers: {
        "content-type": "application/json",
        "idempotency-key": delivery.eventId,
      },
      body: JSON.stringify(delivery.payload),
      signal: AbortSignal.timeout(10_000),
    });

    if (response.ok) {
      await queue.ack(delivery.id);
      return;
    }

    if (permanentStatuses.has(response.status)) {
      await queue.deadLetter(delivery.id, `HTTP ${response.status}`);
      return;
    }

    await queue.retry(
      delivery.id,
      retryAt(delivery.attempt),
      `HTTP ${response.status}`,
    );
  } catch (error) {
    const reason = error instanceof Error ? error.name : "network_error";
    await queue.retry(delivery.id, retryAt(delivery.attempt), reason);
  }
}

async function run(queue: Queue, rates: RateGate): Promise<void> {
  const shutdown = new AbortController();
  process.once("SIGTERM", () => shutdown.abort());

  while (!shutdown.signal.aborted) {
    const delivery = await queue.lease(shutdown.signal);
    if (delivery) await deliver(queue, rates, delivery);
  }
}
```

This is intentionally one delivery at a time. Production code can run a fixed worker pool, but the pool size must stay bounded and the per-subscriber gate must be shared across workers. A `Retry-After` value can inform scheduling for a `429` response after careful parsing; cap it locally so a malformed or extreme value cannot strand a delivery forever.

The event producer needs protection too. Write the marketplace shipment change and an outbox row in one database transaction, then relay the outbox row to the queue. Without that boundary, a process crash between updating the shipment and publishing the event silently loses a notification. Consumers should receive an immutable event snapshot or a versioned reference, not a mutable object whose meaning changes between retries.

## Testing and operating the delivery path

Test state transitions before throughput. A deterministic fake endpoint should return a sequence such as `429`, timeout, then `204`; assert that the delivery remains unacknowledged until the final response, attempt times increase, and the same event ID appears every time. Add cases for a permanent `410`, duplicate delivery, lease expiry, shutdown during a request, and two workers racing for one job. Then test the rate gate with a fake clock. Wall-clock tests are slow and flaky — a virtual clock lets a suite prove that subscriber A cannot consume subscriber B's permits and that the global cap still applies. Keep the invariant explicit: no delivery disappears without either an ACK, a future retry time, or a dead-letter record. Deploy the producer first so it writes backward-compatible events, then deploy consumers that understand the new version. During rollback, older consumers must ignore fields they don't know. Track queue age, lease expirations, attempts per delivery, `429` rate, terminal status class, dead-letter growth, and successful delivery latency. Alert on sustained queue age rather than a momentary queue depth spike; depth rises naturally during a marketplace shipment burst, while age says subscribers are actually waiting. Security belongs in the workflow as well: require HTTPS, resolve subscriber destinations through an allowlisted configuration path, block private and link-local address ranges after DNS resolution, sign the exact request body with a timestamp, rotate secrets, redact response bodies from routine logs, and limit payload and response sizes. These checks cost less than explaining why a customer-controlled URL reached an internal service.

## When is scheduled batching or an outbox poller better?

The catch is that continuous consumers are not suitable when traffic is near zero and the business accepts a delivery window measured in minutes. A scheduled batch worker is easier to budget and can drain several updates per subscriber under one controlled run. Cron is a time-based job scheduler, so it supplies the wake-up mechanism; it does not provide queue durability, ACK/NACK state, rate limiting, or dead-letter handling. Those still need explicit storage and transitions.

Stick with an outbox poller when the shipment write and notification intent must be atomic, operational capacity is limited, and the existing relational database can absorb the polling load. `FOR UPDATE SKIP LOCKED` lets multiple workers claim different due rows without waiting on one another, but PostgreSQL documents that `SKIP LOCKED` provides an inconsistent view and is suitable for queue-like access rather than general reporting. The trade-off is table churn, vacuum pressure, and another hot path in the primary database.

Choose the smallest system that meets the delivery SLO. For a low-volume marketplace, that may be one outbox table, a bounded Node.js worker, and a cron-triggered sweep. Move to continuous queue consumers when measured queue age violates the SLO or bursts interfere with transaction processing. The deciding evidence is queue age and database load, not a diagram.

## References

- https://en.wikipedia.org/wiki/Cron
- https://www.postgresql.org/docs/current/sql-select.html
