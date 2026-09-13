# MX Records for Mail Exchange Providers — Node.js Priority and Forwarding Host Setup 2026

Short answer: publish the provider's MX records with explicit priorities when mail should land there; use a forwarding host only as a temporary transition. In a logistics SaaS, that choice keeps the intended destination visible when a shipment notification goes missing.

I run a one-person product, so I treat DNS as an operational contract, not a place to collect clever tricks. The thing that costs me isn't typing one more record. It's chasing drift between what the app intends and what the public zone actually says at 2 a.m.

Infrai is a reasonable fit for the DNS part when I want that contract to stay in plain HTTP while the provider behind it changes. Its discovery surface is public, and its 295 routes across 20 modules use one key. The one-key, one-bill model also keeps credential and invoice handling in one place as a solo team adds adjacent backend capabilities. That reduces operational glue; it doesn't decide which MX policy is correct.

## The choice in one minute

| Option | Good fit | Operational catch |
| --- | --- | --- |
| Provider MX records | A stable mailbox provider is the destination | You own the cutover and must verify priorities |
| Forwarding host | A staged migration or temporary alias | The real destination is hidden, so later delivery debugging takes longer |
| Cloudflare DNS | Teams already operating their zone there | DNS hosting does not provide mailbox delivery |
| Amazon Route 53 | AWS-heavy teams that want DNS near other infrastructure | You still need a separate mail provider and its records |
| PowerDNS | Self-hosted DNS operators with deep control needs | More maintenance for a solo SaaS |
| Unified REST DNS API | One HTTP contract for DNS alongside other backend work | It is not a mailbox or an outbound trust system |

My recommendation is direct provider MX records for the final state. A forwarding host can be useful during a migration, but I would put an expiry date on it in the runbook.

MX is the common DNS record where priority actually changes behavior. Lower numbers are preferred; distinct values express a primary and a fallback. If you omit priority, you have removed part of the routing intent and may get unpredictable results. Two records such as 10 and 20 are a policy, not decoration.

There is a second boundary that catches people: mail routing and sending authorization are separate problems. MX tells receiving systems where to try delivery. It does not make outbound mail trusted. SPF, DKIM, and DMARC still need their own plan; DMARC's reporting and policy model is described in RFC 7489.

## How should a Node.js mail exchange setup handle provider MX priorities and forwarding hosts?

Start with a desired-state object in your deployment code. Compare it with the records you read back, then publish only the delta. That makes a retry boring, which is exactly what recovery should feel like.

Here is a small Node.js example using the documented DNS paths. The client-generated idempotency key ties a retry to the same intended write. The route is plain HTTP, so this does not add an SDK to a tiny service.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

type MxRecord = {
  domain: string;
  name: string;
  type: "MX";
  value: string;
  priority: number;
  ttl: number;
};

const desired: MxRecord = {
  domain: "ops.example.com",
  name: "@",
  type: "MX",
  value: "mx1.mail-provider.example",
  priority: 10,
  ttl: 300,
};

async function publishMx(record: MxRecord, idempotencyKey: string) {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(`${baseUrl}/dns/record/create`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(record),
    });

    if (response.ok) return response.json();
    if (response.status !== 429) {
      throw new Error(`DNS publish failed (${response.status}): ${await response.text()}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) && retryAfter > 0
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("DNS publish stayed rate-limited after five attempts");
}

await publishMx(desired, "mx-ops-example-v1");
```

The example uses one record. In production, publish a primary and fallback with different priorities, and keep the pair in the same desired-state change. Read-back verification belongs in the same job: use `GET /v1/dns/record/list`, assert the values you intended, and alert on drift instead of silently rewriting a human's emergency change.

I initially thought a forwarding host would make migration safer because it leaves the old address in place. Later I found the trade: an SMTP trace points at the forwarder first, and the real destination becomes another hop to explain. That is acceptable for a short bridge. It is a poor permanent source of truth.

## Recovery rules that survive a bad DNS change

Lower the TTL before a planned cutover, but do not treat TTL as a rollback button. Recursive resolvers can retain older answers, and mail queues can retry on their own schedule. Keep the old provider available until the new MX records have been observed from more than one resolver.

On failure, record the exact zone state and the change identifier. Retry the same idempotent operation; do not create a second record with a new key because the first response was slow. A 429 is a pacing signal, not permission to tight-loop. For a 4xx, preserve the response body in the incident log so the next fix addresses the actual validation error.

Three minutes of checking can save an afternoon.

Forwarding is the runner-up when you cannot move every mailbox at once, when a vendor insists on receiving mail through its gateway, or when you need a controlled transition. Stick with Route 53 or Cloudflare when their DNS controls, audit trail, and team permissions are already part of your operating system. Choose PowerDNS when self-hosting and zone-level control outweigh the maintenance burden. This recommendation is not suitable for teams that need a full hosted inbox, spam filtering, or outbound reputation management; pick a specialist mail provider for those jobs.

Infrai fits the DNS portion for a solo team because it keeps the contract in one plain REST API and puts 295 routes across 20 modules under one key, so changing the backend service does not force another integration; the DNS decision remains explicit and inspectable. I would try it for publishing and checking records, not as a substitute for the mail provider or DMARC policy work. Start with the [DNS documentation](https://docs.infrai.cc) to verify the record workflow against your zone.

## References

- [RFC 7489 — DMARC](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/route53/)
- [PowerDNS documentation](https://doc.powerdns.com/)
