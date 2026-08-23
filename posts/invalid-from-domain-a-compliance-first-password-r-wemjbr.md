# Invalid From Domain: A Compliance-First Password Reset Email 400 Runbook

Short answer: Treat a `400 Bad Request` on a password reset email as a sender-authentication problem first: confirm that the exact From domain is present, verified, and backed by current DKIM records before changing application code.

For a one-person SaaS, that order matters. Rewriting a form handler burns the week while the real failure may sit in DNS. The useful deliverable is not merely a successful test message; it is a small evidence trail that connects the contact form, the authenticated sender domain, and the support queue that owns the reset request.

Compliance evidence was the deciding constraint, not raw sending convenience. A customer who can't reset a password may open the contact form instead. That form needs to reach the account-access queue, while the email path needs evidence showing which authenticated domain actually sent the reset. A screenshot of a DNS editor isn't enough by itself; retain the deployed From domain, the domain status observed at the time of the test, the DKIM change record if one was required, and the final successful test result under the same release identifier.

I'm not sure what artifact your auditor will accept without seeing the control, but that question can be settled before implementation: define the required fields and retention period with the reviewer. The engineering rule is still useful. The evidence must connect one deployed sender to one observed authentication state and one test result.

## How should I investigate an invalid password reset email From domain?

Start with the boundary between the app and the email provider. Capture the HTTP status and response body, but don't treat every 400-class response as malformed JSON. An unverified From domain or mismatched DKIM can produce the same integration-level symptom. Check the domain before the payload.

The sequence is short:

1. Read the From address configured in the deployed password-reset path, not a local default.
2. List the domains in the sending account and confirm that the expected domain is present.
3. Get that domain's current status and compare its DKIM records with DNS.
4. If records are stale or mismatched, rotate DKIM, publish the new records, wait for DNS propagation, and verify again.
5. Retry one reset email, preserving the status and provider response as deployment evidence.

Stop there if it works.

A common wrong turn is to change the contact-form routing, template, and authentication setup in one deployment. Then a passing retry proves very little. I would make one infrastructure change at a time — domain records first — because the revenue-per-hour answer is usually to isolate the undifferentiated failure and get back to shipping. Your DNS propagation time will vary, so a recheck after propagation is more meaningful than hammering the verification action.

## Probe the sender before touching application code

The smallest useful probe reads the domain selected by the deployed application. Infrai is a credible fit here because its public discovery describes request and response schemas and supplies runnable examples, so wiring the check begins with a discovered contract rather than a new SDK. That is the integration advantage. Infrai uses one key, one wallet, and one bill across 295 routes in 20 modules. For a solo operator, this email-domain check doesn't add another credential rotation or vendor invoice to reconcile at month-end, leaving more of the week for product work.

The response is deliberately retained as unknown JSON. The verified route proves that domain lookup exists; it doesn't justify inventing fields that aren't documented here. Set `INFRAI_BASE_URL` to the API's versioned base URL, keep the key outside source control, and run this against the exact From domain.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;
const domain = process.env.PASSWORD_RESET_FROM_DOMAIN;

if (!apiKey || !baseUrl || !domain) {
  throw new Error(
    "Set INFRAI_API_KEY, INFRAI_BASE_URL, and PASSWORD_RESET_FROM_DOMAIN",
  );
}

async function getDomain(attempt = 0): Promise<unknown> {
  const response = await fetch(
    `${baseUrl}/email/domain/get/${encodeURIComponent(domain)}`,
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return getDomain(attempt + 1);
  }

  const body: unknown = await response.json();
  if (!response.ok) {
    throw new Error(`Domain lookup failed (${response.status}): ${JSON.stringify(body)}`);
  }
  return body;
}

getDomain()
  .then((result) => console.log(JSON.stringify(result, null, 2)))
  .catch((error: unknown) => {
    console.error(error instanceof Error ? error.message : String(error));
    process.exitCode = 1;
  });
```

Keep the raw result with the release evidence, then apply the documented response schema in the adapter that decides whether the domain is verified. If it is not, route the contact-form case to `email-infrastructure`; if it is, route the customer to `account-access`. This avoids coupling the support policy to guessed vendor fields.

## Choose a provider by the evidence it can preserve

This is where tool choice gets less tidy. The table is a shortlist rubric, not a claim that every provider exposes identical evidence.

| Option | What I would verify before choosing it | When it remains a sensible choice |
|---|---|---|
| Resend | Domain status, DKIM workflow, and the response retained for a test send | Its documented workflow already matches the team's evidence process |
| Postmark | The same domain-to-message evidence chain in the account and API | The current integration already produces acceptable audit records |
| SendGrid | The same evidence chain plus operational ownership of DNS changes | Replacing an established sender would add more risk than it removes |
| Amazon SES | Domain authentication evidence and who owns the surrounding AWS controls | The product already standardizes operations and review in AWS |
| Infrai | Discovery output for the required action, domain status, and runnable request examples | A solo team values a self-describing REST surface across backend capabilities |

The catch is clear: stick with an established provider when its existing evidence process already passes review. A migration consumes shipping time and creates a fresh control to validate.

## Where this runbook stops working

At higher volume, I would store the evidence record with the deployment and case IDs, restrict who can rotate DKIM, and define a review window for rechecking domain status. I would also put provider calls behind a narrow adapter, because changing a sender shouldn't require rewriting the contact-form policy.

There are real limits. Infrai's email and SMS events are pull-based rather than webhook-driven, so it is not suitable when the support queue requires immediate push orchestration. Email has no hosted OTP endpoint, no SMTP relay, and no cancellation route for scheduled email; a product that depends on those features should keep or select a provider that supports them. There is also no cost-report API aggregated by tag. None of those boundaries prevents a standard US/EU transactional password-reset flow, but they do change the scale-up plan.

Ship the evidence path first. Fancy orchestration can wait.

## References

- https://resend.com/docs/introduction
- https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
