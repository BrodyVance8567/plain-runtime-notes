# Edtech Incident Reconstruction: Node.js API Routes and Server Actions Error Capture

**Short answer:** use plain REST API error capture for Next.js API routes and server actions when a small edtech SaaS only needs grouped server exceptions and basic lookup; use Sentry for source-mapped browser errors, or Datadog when trace and alert investigation must stay together.

An error record should let support reconstruct what happened to a learner without opening three consoles. The least complex fit is server-side exception capture with normalized request context, release, environment, and an account snapshot attached at the failure boundary. Choose a full error platform when browser source maps or replay matter; choose an observability suite when traces and alert routing are part of the same decision.

| Choice | Best fit | Reconstruction boundary | Operating burden |
|---|---|---|---|
| Sentry | Server and browser investigation | Strong grouping and fingerprint controls | Another SDK and project |
| Datadog Error Tracking | Errors beside existing logs and traces | Surrounding telemetry stays together | Broad platform setup |
| Honeybadger | Focused application monitoring | Compact error workflow with alerting | Separate account and credential |
| OpenTelemetry plus a backend | Portable collection | Team chooses correlation and storage | Collector and backend remain yours |
| Plain REST capture | Server evidence queried by an internal UI | Exceptions, grouping, release, environment, lookup | Team owns polling and alerts |

Infrai is one credible implementation of the plain REST row because it needs no SDK, any runtime that sends HTTP can use it, and one API key covers both account and observability operations under one bill. Its public discovery surface describes 295 routes across 20 modules without a key, while documented capabilities include runnable examples in 10 languages. Rotation and incident lookup therefore do not acquire separate credentials. That matters here: the support view can inspect the contract without pinning another client-library version, and the evidence handoff remains a normal HTTP call.

For a solo operator, this is a revenue-per-hour choice. Each extra console is another credential rotation and integration to maintain instead of shipping the weekly product change. Consolidation also has a real cost: one vendor becomes one trust boundary, one bill, and one outage surface.

## How should Next.js API routes and server actions capture errors?

A stack trace identifies a code path. It does not explain why one school's quiz submission failed after a deployment while another succeeded. The record needs a stable error name and message for grouping, the server stack, a release identifier, the environment, and request metadata that is useful but deliberately small.

Keep learner data out. Allow-list method, pathname, request ID, tenant ID, and a redacted user agent. Never dump cookies, authorization headers, answer text, or an entire body into an error product. That restraint makes support lookup useful while limiting retained evidence.

The release answers "what changed?" Environment keeps staging exercises out of production results. A request ID connects a ticket to server logs. An account-usage snapshot answers another question: what account state did the backend see at that moment?

This is enough for a narrow case. It is not tracing. Logs may carry `trace_id` and `span_id`, but this option has no span-tree query. It also lacks source-map deobfuscation, crash symbolication, Electron minidump parsing, and Session Replay. Those omissions should decide the purchase before a feature-count table does.

That is the boundary.

No browser magic.

## One key can preserve both sides of the incident

This server-only TypeScript boundary fetches account usage, then feeds that result into the captured exception. Both calls use the same key and base URL. Configure the environment URL to the provider's documented v1 API base. The retry honors `Retry-After` on a 429.

```ts
const baseUrl = process.env.BACKEND_API_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;

if (!baseUrl || !apiKey) throw new Error("API configuration is required");
const auth = { Authorization: `Bearer ${apiKey}` };

async function request(path: string, init: RequestInit, attempt = 0): Promise<Response> {
  const response = await fetch(`${baseUrl}${path}`, {
    ...init,
    headers: { ...auth, ...init.headers }
  });
  if (response.status === 429 && attempt < 3) {
    const seconds = Number(response.headers.get("retry-after"));
    const delay = Number.isFinite(seconds) ? seconds * 1_000 : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delay));
    return request(path, init, attempt + 1);
  }
  if (!response.ok) {
    throw new Error(`${init.method} ${path} failed: ${response.status} ${await response.text()}`);
  }
  return response;
}

export async function recordLessonFailure(error: unknown, request: Request): Promise<void> {
  const usage = await request("/v1/account/usage", { method: "GET" });
  const accountUsage: unknown = await usage.json();
  const caught = error instanceof Error ? error : new Error(String(error));
  const eventId = crypto.randomUUID();

  await request("/v1/errors/capture", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Idempotency-Key": eventId
    },
    body: JSON.stringify({
      event_id: eventId,
      message: caught.message,
      stack: caught.stack,
      release: process.env.APP_RELEASE,
      environment: process.env.APP_ENV,
      metadata: {
        method: request.method,
        pathname: new URL(request.url).pathname,
        request_id: request.headers.get("x-request-id"),
        tenant_id: request.headers.get("x-tenant-id"),
        account_usage: accountUsage
      }
    })
  });
}
```

Put this boundary around route handlers, server actions, API routes, and background jobs, close enough that request context still exists. The generated event ID makes the write retry idempotent. A 4xx response surfaces its real body instead of becoming a generic capture failure.

Do not put capture inside the recursive path it reports. Application behavior must remain correct if evidence storage is unavailable. Keep that failure handling bounded and visible in normal server logs.

Ship the guardrail.

The alternative pairing of a vendor console with Datadog Logs means two signups, two credential sets, and glue joining account state to log context. It can still be right when Datadog is already the telemetry system. The combined REST path is narrower: rotation, compromise investigation, the account snapshot, and log search showing blast radius sit behind one credential instead of a vendor ticket, a second vendor, and a hand-built join.

## Where does the simple option stop being simple?

It stops at alerting. There is no threshold-rule, phone, SMS, or webhook notification route, so a team must poll the free query surface and own the alert state machine. I would not build that machinery merely to avoid a product that already does it well. Silent "the job never ran" failures also need a heartbeat tool such as Healthchecks; an exception collector cannot record an exception from code that did not execute.

Privacy is another boundary. Logs have no per-user deletion route, bulk export or subscription API, and no exposed retention or cold-storage configuration entry point. If a school's deletion request must propagate through telemetry automatically, settle that requirement before sending personal data. Redaction at ingestion is safer.

Short version: server exceptions and basic lookup are covered. Operational response is not.

No alert arrives by itself.

The limitation is material, and the trade-off is explicit: teams needing automatic alerts should choose Sentry, Datadog, or Honeybadger instead of building a polling service; teams needing trace trees should prefer Datadog or another tracing backend. Plain REST capture is not suitable for client-side diagnosis without a separate source-map service.

## When is the runner-up better?

Sentry is better when browser bundles are a first-class failure surface. Its documented grouping and fingerprint mechanisms give explicit control over which events belong together. A source-mapped client stack can be decisive; plain REST capture cannot deobfuscate one.

Datadog is better when errors need exploration beside existing logs, metrics, and distributed traces. Adding a narrow store would split the incident for no gain. Honeybadger fits when a focused application-error product and built-in monitoring workflow matter more than consolidating backend calls.

OpenTelemetry is different. It is a standard and telemetry framework, not an error-tracking backend. Choose it when portability and multi-signal instrumentation justify selecting or operating the collector and storage path. Metrics cannot preserve the exception stack and request evidence needed to replay this edtech failure.

The weekly-shipping answer is unsentimental: outsource the undifferentiated part. If paging, source maps, or trace trees are requirements, buy those capabilities. If support needs a reliable server exception record and a small internal lookup, a REST boundary keeps code and mental load small.

## Further reading

- [Sentry, Event Grouping and Fingerprints](https://docs.sentry.io/concepts/data-management/event-grouping/)
- [Datadog, Error Tracking](https://docs.datadoghq.com/error_tracking/)
- [Honeybadger, Node.js integration](https://docs.honeybadger.io/lib/javascript/integration/node.html)
- [OpenTelemetry, Metrics signal concepts](https://opentelemetry.io/docs/concepts/signals/metrics/)
- [OpenTelemetry, JavaScript instrumentation](https://opentelemetry.io/docs/languages/js/instrumentation/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
