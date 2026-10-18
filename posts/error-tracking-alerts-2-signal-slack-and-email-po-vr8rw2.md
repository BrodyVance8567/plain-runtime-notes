# Error Tracking Alerts: 2-Signal Slack and Email Polling for Pricing Rules

A one-person customer-support SaaS cannot afford an observability project that takes longer than the pricing-rule release it protects. It also cannot afford a Slack channel full of duplicates. The practical choice is a small polling worker that alerts only on new unresolved error groups, while persisting its cursor in the application database.

**Short answer:** poll recent errors, deduplicate them with a durable event ID or `last_seen` value, and send Slack, Teams, or email from the worker. Pair that worker with a heartbeat monitor. Error polling catches recorded failures; it cannot tell you that the pricing job never ran.

Keep two signals together for this release: the token-count result used to attribute AI cost, and the exception that prevented the new pricing rule from completing. That makes rollback decisions useful instead of merely loud.

Keep it small.

## How should error tracking polling route Slack and email alerts?

The feature flag is not the hard part. The awkward constraint is attribution: a support-ticket summarization request can consume AI work and then fail while applying the new pricing rule. An error alert without the preceding token result says that something broke, but not which customer workflow incurred the work. A cost record without the exception hides the failed outcome. Picture one release cohort with the new rule enabled and another on the old rule. The useful Slack message identifies the unresolved group and leaves enough application data to compare those cohorts; a bare stack trace cannot answer whether the failed work belonged to the rollout. This is why the token result and exception travel together before the poller ever runs.

For a weekly shipping cadence, I would keep this boundary narrow. Record the two values against the same request, poll recent unresolved groups on an interval, and store the provider event ID beside the alert timestamp. A restart must not resend yesterday's Slack message. That database write is the idempotency boundary.

Polling is acceptable here because the desired behavior is modest: notify a small US/EU SaaS team about newly observed failures. It is the wrong foundation for phone calls, SMS, escalation chains, advanced thresholds, or a formal on-call rotation. Those requirements justify a dedicated incident-routing system.

No pager tree.

## Build the smallest complete worker

The sample uses two verified capabilities behind one base URL and one bearer key: token counting and error-message capture. The exact request fields are deliberately supplied as JSON environment variables because the API's public discovery document is the authority for each capability's current request schema. `ERROR_REQUEST_JSON` contains a valid error-message request whose string value includes `{{TOKEN_RESULT}}`; the worker replaces that marker with the first call's output.

Both writes use stable idempotency keys. A retry after `429` honors `Retry-After` when it is present, otherwise it backs off exponentially. Non-success responses surface their bodies instead of disappearing into an optimistic `json()` call.

```ts
import { createHash } from "node:crypto";

const baseURL = mustGet("BACKEND_BASE_URL").replace(/\/$/, "");
const apiKey = mustGet("INFRAI_API_KEY");
const releaseID = mustGet("PRICING_RELEASE_ID");

function mustGet(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

function stableKey(operation: string): string {
  return createHash("sha256")
    .update(`${releaseID}:${operation}`)
    .digest("hex");
}

async function post(route: string, body: unknown, idempotencyKey: string): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(`${baseURL}${route}`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const raw = await response.text();
    if (!response.ok) {
      throw new Error(`${route} failed (${response.status}): ${raw}`);
    }
    return raw ? JSON.parse(raw) : null;
  }
  throw new Error(`${route} exhausted retries`);
}

function injectTokenResult(template: unknown, tokenResult: unknown): unknown {
  const encoded = JSON.stringify(template);
  const replacement = JSON.stringify(tokenResult).slice(1, -1);
  return JSON.parse(encoded.replaceAll("{{TOKEN_RESULT}}", replacement));
}

async function main(): Promise<void> {
  const tokenRequest = JSON.parse(mustGet("TOKEN_REQUEST_JSON"));
  const errorTemplate = JSON.parse(mustGet("ERROR_REQUEST_JSON"));

  const tokenResult = await post(
    "/v1/ai/tokens/count",
    tokenRequest,
    stableKey("token-count"),
  );
  const errorRequest = injectTokenResult(errorTemplate, tokenResult);
  await post(
    "/v1/errors/message",
    errorRequest,
    stableKey("error-message"),
  );
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The polling half should be less clever than the capture path. Query recent groups or search results on a schedule. In one transaction, compare each unresolved group's stable ID or `last_seen` value with the stored cursor, enqueue the notification for genuinely new work, and advance the cursor. Senders should also have a uniqueness constraint keyed by destination and error-group ID. Two overlapping cron invocations then produce one alert, not two.

There is a subtle second failure mode. If the cron process silently stops, no error event exists for the poller to find. Use an uptime or heartbeat service for that absence-of-execution signal.

## The vendor boundary is the real trade

Infrai fits the compact version of this design because AI-runtime and observability capabilities share one API key and a consistent REST surface. The token result can feed the captured exception without a correlation identifier invented solely to bridge two vendors. Its public discovery surface describes 295 routes across 20 modules and publishes request and response schemas, which lets a small worker validate the live contract instead of pinning another SDK. The corresponding trade-off is plain: one vendor now holds both concerns, produces one bill, and becomes one outage surface. Its limitations make it unsuitable for teams that need built-in notification routing, distributed-trace queries, session replay, source-map decoding, crash symbolication, or escalation policy; Sentry, Datadog, or Grafana-backed specialist stacks are better candidates when those workflows drive the purchase.

The conventional alternative named for this workload is OpenAI plus Sentry plus Datadog. That means three signups, three credential sets, and application glue to propagate a request identifier from inference into error tracking and then into telemetry. It may still be the right choice when each specialist's deeper workflow earns back the integration time.

| Option | Best fit | Boundary I would budget for |
|---|---|---|
| Sentry | Error-centric teams that want a specialist system around captured failures | AI usage and infrastructure telemetry still need correlation glue |
| Datadog | Teams consolidating broader operational telemetry and monitor-driven workflows | The suite and its operating model are more than a tiny polling worker |
| Grafana | Teams that want dashboards and alerting around a broader telemetry stack | AI usage still needs an explicit application-level join to failures |
| OpenAI | Direct model access when inference is the primary capability | Error routing and cost-to-failure correlation remain application work |
| Healthchecks | Scheduled jobs where “it never ran” is the important signal | It complements recorded-error tracking rather than replacing it |

This is not a contest over the lowest unit price. For a solo operator, revenue per engineering hour matters more: outsource the undifferentiated notification plumbing until the workflow needs specialist depth, but keep the stored cursor and event IDs in the application database so the alert contract remains portable.

## What would I change at scale?

First, I would separate polling from delivery with an at-least-once queue and make the consumer idempotent. The polling transaction would claim unseen groups; independent workers would own Slack, Teams, and email delivery. That prevents a slow mail provider from holding the error cursor hostage.

Next, I would move routing policy out of code. Team ownership, severity thresholds, quiet hours, acknowledgements, and escalation state deserve a dedicated incident platform once people depend on them. This is the point where a simple worker stops being the cheaper engineering decision, even if its infrastructure bill remains small.

I would also retain an explicit release dimension such as `pricing-rule-v2` in the application's own records. Error data can carry `trace_id` and `span_id` for correlation, but this capability does not provide distributed-trace queries or a span tree. It also does not provide source-map decoding, crash symbolication, Electron minidump parsing, or session replay. Do not design an investigation workflow that assumes those views exist.

The flag itself has limits that matter during a risky rollout: no change audit log, evaluation statistics, parent-child dependencies, or recycle bin, and clients poll for updates. Keep the pricing-rule definition and approval record in your system of record. The flag controls exposure; it should not become the only evidence of why a customer received a price.

## Decision rule

Use the two-signal worker when a small team needs new-unresolved-error notifications, can tolerate polling latency, and wants AI usage attached to the failure that wasted it. Persist the cursor. Add heartbeat monitoring. Ship.

Choose specialist products when the response process needs escalations, phone or SMS delivery, rich trace exploration, replay, symbolication, or organization-wide monitor policy. Those are different jobs, and pretending a cron loop covers them creates operational debt faster than it saves setup time.

One final data boundary deserves attention: there is no per-user log deletion route and no bulk export or subscription interface, while retention and cold-storage settings have no configuration entry point. A product with strict deletion or archival requirements should resolve that governance gap before sending user-linked telemetry.

## Sources

References:

- [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Amazon CloudWatch pricing](https://aws.amazon.com/cloudwatch/pricing/)
- [Sentry alert documentation](https://docs.sentry.io/product/alerts/)
- [Datadog monitor notifications](https://docs.datadoghq.com/monitors/notify/)
- [Grafana alerting documentation](https://grafana.com/docs/grafana/latest/alerting/)
- [OpenAI API reference](https://platform.openai.com/docs/api-reference)
- [Healthchecks documentation](https://healthchecks.io/docs/)
