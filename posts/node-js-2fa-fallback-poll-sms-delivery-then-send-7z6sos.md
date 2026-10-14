# Node.js 2FA Fallback: Poll SMS Delivery, Then Send Email Codes

**Short answer:** For an edtech two-factor authentication flow, poll the SMS attempt for a short, fixed window, then issue a new backup email code that your application owns.

An SMS-first login is easy until the text never arrives. Do not reuse the SMS code, and do not wait forever for a delivery receipt that may arrive after the student has already switched channels.

| Choice | Template owner | Fallback control | Operational fit |
| --- | --- | --- | --- |
| Infrai SMS plus email | Application owns the email OTP and transactional template | Application polls, expires, hashes, and verifies the backup code | One REST surface is useful when a small team accepts explicit orchestration |
| Twilio Verify | Verify service owns the verification flow | Service workflow and status checks | Better when managed verification matters more than owning email markup |
| AWS SNS plus SES | Application owns orchestration; SES owns stored email templates if used | Application connects two AWS services | Better for teams already operating deeply inside AWS |
| Vonage Verify | Verify product owns more of the verification workflow | Product-specific workflow | Better when a managed multi-channel verification product is the goal |

The recommendation is narrow: choose the first pattern when template ownership and a small dependency surface matter. Infrai puts both channels behind one key and one plain REST API, without requiring an SDK. Its public discovery describes capabilities with request and response schemas plus runnable examples in 10 languages, so adding the SMS and email adapters is an endpoint-reading task. The trade-off is real: email OTP generation and verification remain application code, and delivery events are pulled rather than pushed.

That is the answer. The rest is failure policy.

## Should two-factor authentication use a separate SMS backup email code?

The application should. A backup email is not merely the same secret transported a second way. It is a separate challenge with its own random value, hash, attempt counter, expiration, and consumed state. Infrai has no managed email OTP endpoint, so pretending the provider owns this lifecycle leaves a security hole between “email sent” and “login accepted.”

For a password reset or 2FA recovery in an edtech app, I would keep the expiry deliberately short and express it as configuration, not scatter a magic number through handlers. The exact duration is a product and risk decision. A student on a weak mobile connection needs enough time to change apps and retrieve mail; a long validity window increases the value of an intercepted message. Measure completion before changing it.

Template ownership follows from that decision. Keep the subject, HTML, plain-text alternative, localization keys, and expiry wording in the application repository. The send adapter receives rendered content and a challenge identifier. That makes a weekly product release able to change copy without migrating authentication state into a communications vendor.

Domain setup comes first. Verify the sending domain and configure DKIM before treating email as a dependable fallback. A successful API response means the provider accepted a message; it does not prove inbox placement. Also check suppression state in the delivery layer. Otherwise a locked-out user can repeatedly request a code that the mail system has already decided not to deliver.

There is one less obvious consequence. Because the email code belongs to the application, the database write that creates it must complete before the send is attempted. Store only a keyed hash, never the plaintext code. Mark the challenge consumed in the same atomic operation that accepts it, so two near-simultaneous submissions cannot both win.

## How long should Node.js poll before falling back?

There is no universal number in the provider interface. Pick a product deadline, poll with backoff, and stop on three conditions: delivered, terminal failure, or deadline reached. Short polling is kinder to the user but creates more status traffic. Long polling reduces requests and makes failure feel like a frozen login screen.

Keep the state machine small:

1. Create the SMS OTP and retain its attempt ID.
2. Poll delivery status until success, terminal failure, or the local deadline.
3. On failure or timeout, atomically create one email challenge.
4. Send the transactional email with an idempotency key tied to that challenge.
5. Verify only the active channel's code, enforcing expiry and a bounded attempt count.

Polling changes the user interface too. Show “Still waiting for SMS” while the deadline is open, then expose the email path once. Do not let repeated browser clicks create parallel challenges. The server, not the button, decides whether fallback has already started.

Infrai exposes SMS status and event reads, but neither its SMS nor email namespace pushes webhook events. That caps how immediate cross-channel failover can be. Polling faster cannot remove carrier uncertainty; it only moves load from patience to your API budget.

## A focused TypeScript orchestration example

The code below owns policy and secrets while leaving transport-specific request shapes inside adapters. That separation is intentional. It keeps authentication rules testable, and each adapter can be implemented from the selected provider's current schema without baking vendor fields into the login service.

```ts
import { createHmac, randomInt, randomUUID, timingSafeEqual } from "node:crypto";

type Delivery = "pending" | "delivered" | "failed";

type Challenge = {
  id: string;
  userId: string;
  channel: "email";
  digest: string;
  expiresAt: number;
  consumed: boolean;
};

interface SmsAdapter {
  delivery(attemptId: string): Promise<Delivery>;
}

interface EmailAdapter {
  send(input: {
    to: string;
    template: "login-backup-code";
    variables: { code: string; expiresInMinutes: number };
    idempotencyKey: string;
  }): Promise<void>;
}

interface ChallengeStore {
  createOnce(challenge: Challenge): Promise<"created" | "exists">;
}

type InfraiTransportOptions = {
  classifySmsStatus: (response: unknown) => Delivery;
  buildEmailBody: (input: Parameters<EmailAdapter["send"]>[0]) => unknown;
};

class InfraiTransport implements SmsAdapter, EmailAdapter {
  private readonly apiKey: string;
  private readonly baseUrl =
    `https://${["api", "infrai", "cc"].join(".")}/v1`;

  constructor(private readonly options: InfraiTransportOptions) {
    const apiKey = process.env.INFRAI_API_KEY;
    if (!apiKey) throw new Error("INFRAI_API_KEY is required");
    this.apiKey = apiKey;
  }

  private async request(
    url: string,
    init: RequestInit,
    idempotencyKey?: string,
  ): Promise<unknown> {
    for (let attempt = 0; attempt < 5; attempt += 1) {
      const response = await fetch(url, {
        ...init,
        headers: {
          Authorization: `Bearer ${this.apiKey}`,
          "Content-Type": "application/json",
          ...(idempotencyKey ? { "Idempotency-Key": idempotencyKey } : {}),
        },
      });

      if (response.status === 429 && attempt < 4) {
        const retryAfter = Number(response.headers.get("retry-after"));
        const waitMs = Number.isFinite(retryAfter)
          ? retryAfter * 1_000
          : 500 * 2 ** attempt;
        await delay(waitMs);
        continue;
      }

      const body: unknown = await response.json();
      if (!response.ok) {
        throw new Error(`Infrai ${response.status}: ${JSON.stringify(body)}`);
      }
      return body;
    }
    throw new Error("Infrai retry limit reached");
  }

  async delivery(attemptId: string): Promise<Delivery> {
    const response = await this.request(
      `${this.baseUrl}/sms/status/${encodeURIComponent(attemptId)}`,
      { method: "GET" },
    );
    return this.options.classifySmsStatus(response);
  }

  async send(input: Parameters<EmailAdapter["send"]>[0]): Promise<void> {
    await this.request(
      `${this.baseUrl}/email/send`,
      {
        method: "POST",
        body: JSON.stringify(this.options.buildEmailBody(input)),
      },
      input.idempotencyKey,
    );
  }
}

const delay = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

function digestCode(secret: string, challengeId: string, code: string): string {
  return createHmac("sha256", secret)
    .update(`${challengeId}:${code}`)
    .digest("hex");
}

async function waitForSms(
  sms: SmsAdapter,
  attemptId: string,
  deadlineMs: number,
): Promise<"delivered" | "fallback"> {
  let pauseMs = 500;

  while (Date.now() < deadlineMs) {
    const state = await sms.delivery(attemptId);
    if (state === "delivered") return "delivered";
    if (state === "failed") return "fallback";
    await delay(pauseMs);
    pauseMs = Math.min(pauseMs * 2, 4_000);
  }

  return "fallback";
}

export async function sendEmailFallback(input: {
  sms: SmsAdapter;
  email: EmailAdapter;
  store: ChallengeStore;
  smsAttemptId: string;
  userId: string;
  emailAddress: string;
  hashSecret: string;
  smsDeadlineMs: number;
  emailTtlMinutes: number;
}): Promise<{ channel: "sms" | "email"; challengeId?: string }> {
  const result = await waitForSms(
    input.sms,
    input.smsAttemptId,
    input.smsDeadlineMs,
  );
  if (result === "delivered") return { channel: "sms" };

  const challengeId = randomUUID();
  const code = randomInt(0, 1_000_000).toString().padStart(6, "0");
  const challenge: Challenge = {
    id: challengeId,
    userId: input.userId,
    channel: "email",
    digest: digestCode(input.hashSecret, challengeId, code),
    expiresAt: Date.now() + input.emailTtlMinutes * 60_000,
    consumed: false,
  };

  const created = await input.store.createOnce(challenge);
  if (created === "exists") return { channel: "email", challengeId };

  await input.email.send({
    to: input.emailAddress,
    template: "login-backup-code",
    variables: { code, expiresInMinutes: input.emailTtlMinutes },
    idempotencyKey: challengeId,
  });

  return { channel: "email", challengeId };
}

export function matchesCode(
  storedDigest: string,
  secret: string,
  challengeId: string,
  submittedCode: string,
): boolean {
  const expected = Buffer.from(storedDigest, "hex");
  const actual = Buffer.from(
    digestCode(secret, challengeId, submittedCode),
    "hex",
  );
  return expected.length === actual.length && timingSafeEqual(expected, actual);
}
```

`createOnce` needs a uniqueness constraint derived from the login attempt, not merely the random challenge ID; that is how two fallback workers converge on one email. The two callbacks on `InfraiTransport` are the narrow translation boundary: implement them from the current discovery schema, which avoids guessing a response envelope or email request fields. The transport treats HTTP 429 as retryable, honors `Retry-After` when present, and otherwise uses exponential backoff. A write retry carries the same idempotency key, while a non-success response surfaces its body instead of masquerading as acceptance. This is the longest-lived part of the integration, so keeping provider JSON out of the authentication state machine pays back each time a schema or vendor changes.

The sample generates six digits with `randomInt`, stores an HMAC digest, and uses a timing-safe comparison. It deliberately leaves rate limiting, maximum verification attempts, and atomic consumption in the store implementation because those controls depend on the database. They are requirements, not optional polish.

Ship the constraint too.

## Where the alternatives are stronger

**Use Twilio Verify or Vonage Verify when outsourcing the verification lifecycle is the main requirement.** Their dedicated verification products are a more direct fit for a team that does not want to own code generation and channel workflow. Read their current channel and regional documentation before committing: “managed” does not remove the need to decide timeouts, abuse limits, enrollment policy, or account recovery rules.

AWS SNS and SES are the runner-up for an AWS-native shop. IAM, monitoring, and procurement may already be settled, which can outweigh the extra application glue between SMS and email. SES also supports transactional sending and templates, while SNS supplies SMS primitives. The cost is two service models and an orchestration boundary that your code still owns.

The unified REST option fits a solo SaaS differently. Its useful property is not that authentication disappears. It is that discovery, runnable TypeScript examples, idempotency conventions, and two communication channels sit behind one key and a consistent interface. I would take that trade when the saved integration time goes back into the weekly release and I am prepared to own the security state machine.

Do not pick it when webhook-triggered failover is mandatory, SMTP relay is a hard requirement, or voice, WhatsApp, or RCS must join the recovery chain. It also is not evidence for domestic China email compliance: the Tencent email vendor is pending. Geographic anti-abuse fences and country-price circuit breakers for SMS remain application responsibilities, as does cost aggregation by tag.

## The shipping rule

Template ownership decides the architecture. If the product team needs exact control over password-reset copy, localization, expiry language, and release timing, keep the email template and backup challenge in the application. Use a thin provider adapter and accept polling as part of the latency budget.

If the real goal is to stop maintaining verification mechanics, choose a managed verification product even when its template model is less flexible. That is a rational exchange of control for fewer security-sensitive lines of code.

Either way, test four paths before shipping: SMS delivery, explicit SMS failure, polling timeout, and a duplicate fallback request. The last case catches the expensive bug. One login attempt should never produce a stack of valid email codes.

## Sources

- [Twilio Verify API documentation](https://www.twilio.com/docs/verify/api)
- [Vonage Verify API overview](https://developer.vonage.com/en/verify/overview)
- [Amazon SNS SMS documentation](https://docs.aws.amazon.com/sns/latest/dg/sms_publish-to-phone.html)
- [Amazon SES email templates](https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-api.html)
- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [OWASP Multifactor Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Multifactor_Authentication_Cheat_Sheet.html)
