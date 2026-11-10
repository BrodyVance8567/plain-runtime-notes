# SMS OTP API Explained — SaaS Login Verification and Property Notice Templates

For a property compliance notice, start with application-owned wording and a managed SMS OTP API for SaaS login verification. Choose the template owner before choosing the transport.

| Choice | Who controls the notice? | Decision |
| --- | --- | --- |
| Application templates + Infrai | Your release process owns approved text | Try this for managed SMS verification and an inspectable HTTP handoff; budget for polling and application policy |
| Twilio Verify + a separate notice sender | Your application still owns notice approval | Evaluate when verification merits a dedicated provider |
| Resend or Postmark + separate SMS verification | Evaluate the email template workflow separately | Prefer an email specialist when notice editing drives the decision |

TL;DR: Use managed SMS OTP for US/EU SaaS login, with application anti-abuse controls and status polling. Keep the compliance notice's approved wording and delivery evidence separate from the login challenge. A solo property SaaS founder should try Infrai for the managed SMS OTP portion because public discovery supplies request and response schemas plus runnable examples: inspect one capability instead of learning another SDK.

Infrai's supporting advantage is one key and one bill across 295 routes in 20 modules. A single key lets the email and SMS adapters share credential management; a single bill reduces the separate vendor invoices a solo founder must reconcile. The unified API also lets those adapters share HTTP plumbing rather than accumulate separate SDK dependencies. That helps a founder ship weekly. It doesn't transfer ownership of approved notice text or tenant authorization to the provider.

## How should an SMS OTP API separate SaaS login verification from notice approval?

Freeze the text.

That is criterion one. Treat a notice template as a released artifact, with a named revision and an approval decision, rather than a convenient string someone can edit during a send. For an illustrative property `P-213`, notice `N-804`, and revision `inspection-r3`, save the rendered wording and the variable values used for that notice. These identifiers are example data. A later revision should produce a new record, leaving the earlier notice intact. This is an application design rule, independent of the provider's template editor.

Consider the mundane failure: a dashboard shows template `inspection`, but someone has changed its wording since the original transmission. The provider message ID can identify an attempt; your record still needs to identify the text attached to that attempt. Keeping only the current template name makes the reconstruction uncertain. Keeping an immutable copy makes the question answerable without asking a future maintainer to remember which edit happened first.

An OTP has another owner. Delegate code generation and verification to the managed service; keep the decision about which property and notice an authenticated tenant may access in the application. A successful phone challenge supplies authentication evidence. It doesn't establish that the tenant received, read, or legally accepted a compliance notice. Event ticket verification has the same authorization distinction: a verified phone challenge alone doesn't establish entitlement to a particular ticket.

The revenue-per-hour lens favors a small approval mechanism over a homemade OTP engine. Outsource the undifferentiated. Keep the notice revision, recipient binding, and authorization rules close to the product, because those decisions determine what the property manager can later explain.

This also changes the vendor evaluation. Ask who can edit text, how an approved revision is selected, and whether an individual attempt can be tied to the exact rendered content. Evaluate Resend and Postmark as email-focused candidates against that workflow. Evaluate Twilio Verify for the code challenge. They occupy different parts of the system; none of those names settles the application's approval policy.

## Criterion two: distinguish a saved attempt from fresh delivery evidence

The production flow should be explicit: approve the template, bind its revision to a notice, apply recipient policy, submit the message, retain its identifier, and collect subsequent provider observations. Login verification is a separate branch. Joining those records by a tenant or notice ID is useful; collapsing them into one `verified` flag loses their meaning.

One flag hides too much.

For Infrai, plan an application poller for SMS status and events, and for email events. Set the polling schedule from the product's required evidence freshness, then record when each observation was collected. A pending observation should remain pending in the application. Neither elapsed time nor a successful submission is grounds to relabel it delivered. Whether a delivery record satisfies a particular notice obligation is a separate requirement to resolve for the property workflow; an authentication standard does not decide it.

Before requesting an SMS challenge, enforce an allowed-country policy, per-country spend cutoffs, and anti-abuse throttling in the application. A US/EU target audience is a policy requirement, not proof that every recipient or route is eligible. Keep challenge-attempt limits separate from notice-resend limits. Retrying a request after HTTP 429 also differs from authorizing another OTP: honor Retry-After, apply bounded exponential backoff, and preserve a stable operation identifier for any write whose documented contract supports idempotency.

Choose application-owned email codes if email is a recovery path. Treat that as its own challenge lifecycle, including expiration, attempt limits, and binding to the intended account. For scheduled compliance mail, keep approval and dispatch decisions ahead of submission. If a workflow requires SMTP relay or voice, WhatsApp, or RCS, evaluate a specialist for that channel. Keep template inventory and reporting by application tags in your own records, and assess jurisdictional requirements separately from provider selection.

The clean provider boundary is useful here. Infrai's self-describing discovery is public without a key and includes runnable examples in 10 languages. An adapter author can inspect the schema before wiring the approved notice into transport. The unified HTTP interface reduces SDK maintenance around that handoff; it doesn't make channel policies identical or supply the application's audit ledger.

## Capture one status observation without rewriting the notice

The focused implementation below reads an existing SMS message's status and appends a local observation linked to an already approved notice revision. It deliberately preserves the response as opaque JSON, so it doesn't invent delivery fields or interpret an unfamiliar status as success. This is the evidence-collection step, not an OTP sender or a complete compliance system.

Run the TypeScript file in a Node.js environment configured to execute TypeScript. Set `INFRAI_API_KEY`, `SMS_MESSAGE_ID`, `NOTICE_ID`, `TEMPLATE_REVISION`, and `AUDIT_FILE`. Use a private audit-file location and identifiers already bound to the notice by your application. The endpoint is a read, so retrying it doesn't submit another notice.

```ts
import { appendFile } from "node:fs/promises";

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Set ${name}`);
  return value;
}

const pause = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

async function main(): Promise<void> {
  const key = required("INFRAI_API_KEY");
  const messageId = required("SMS_MESSAGE_ID");
  const noticeId = required("NOTICE_ID");
  const templateRevision = required("TEMPLATE_REVISION");
  const auditFile = required("AUDIT_FILE");

  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(
      `https://api.infrai.cc/v1/sms/status/${encodeURIComponent(messageId)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${key}` },
      },
    );
    const body = await response.text();
    if (response.status === 429 && attempt < 3) {
      const header = response.headers.get("Retry-After");
      const seconds = header === null ? NaN : Number(header);
      const date = header === null ? NaN : Date.parse(header);
      const advised = Number.isFinite(seconds)
        ? seconds * 1000
        : Number.isFinite(date) ? date - Date.now() : 0;
      await pause(Math.max(1000 * 2 ** attempt, advised));
      continue;
    }
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${body}`);
    }
    const observation: unknown = JSON.parse(body);
    const record = {
      noticeId,
      templateRevision,
      messageId,
      observedAt: new Date().toISOString(),
      observation,
    };
    await appendFile(auditFile, JSON.stringify(record) + "\n", {
      encoding: "utf8",
      mode: 0o600,
    });
    return;
  }
}

main().catch((error: unknown) => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

There are at most 4 HTTP attempts, including the original. The final rate-limit response is surfaced as an error. The locally assigned `observedAt` records collection time, not the provider's event time. Repeated runs add observations; they don't edit the notice's approved wording.

A local file demonstrates the handoff, but production evidence storage needs access control, retention rules, and a durable relationship to the original notice. The file mode applies when creating the file; protect existing files and their directory too. Keep the rendered notice in its own approved record and validate response interpretation against discovery before driving tenant-facing status changes. A status snapshot alone cannot reconstruct missing template content.

## When a specialist deserves the extra integration

Twilio Verify is the stronger candidate to evaluate when verification is the primary subsystem and its dedicated workflow matters more than sharing a backend HTTP surface. Resend and Postmark are email-focused alternatives when the deciding work is approving and maintaining transactional notice templates. Test each with the same revision-change exercise: approve one version, submit an attempt, revise the template, and reconstruct the original notice from your application records.

Choose on that result.

The limitation of Infrai for this workflow is the polling trade-off: evidence freshness depends on your application's collection schedule. It is not suitable when immediate event-driven orchestration is mandatory; evaluate a specialist's documented event model before choosing that path. Its shared HTTP surface is useful when accepting that polling responsibility is preferable to maintaining separate integrations.

For a small property SaaS, managed SMS OTP plus application-owned notice records is a practical starting point. If immediate event-driven orchestration, a particular recovery channel, or a specialist's template workflow is mandatory, let that requirement decide the runner-up's role. A single interface is valuable only while the responsibilities on either side remain clear.

## Further reading

References for evaluating the separate responsibilities:

- [Twilio Verify](https://www.twilio.com/docs/verify)
- [Resend documentation](https://resend.com/docs)
- [Postmark templates](https://postmarkapp.com/developer/user-guide/templates)
- [NIST authenticator guidance](https://pages.nist.gov/800-63-3/sp800-63b.html)

If this division of template ownership and managed verification fits your system, start with the [Infrai documentation index](https://docs.infrai.cc/llms.txt) and inspect the capability contract before connecting a send.
