# How to Choose a GDPR-Compliant Password Reset Email Service: Fintech Template Ownership

Short answer: keep the password-reset template owned by the application team, and let the contact-form router choose a support queue without copying the reset token or account details into that queue. In a one-person SaaS, that boundary gives me a useful revenue-per-hour trade: I can change support routing weekly without handing sensitive message logic to a second system.

The right service is the one that can prove where templates live, what data it retains, and which events it exposes. A familiar send API is not enough. For a fintech product, an incorrect queue can disclose account context, while an incorrect reset message can make an account-recovery flow unusable.

| Decision | Application-owned templates | Service-owned templates |
| --- | --- | --- |
| Reset-token handling | Token stays in the application-generated message | Token crosses a template boundary |
| Queue routing | Route a category and safe case ID | Route after a message provider event |
| Change control | Review in the application repository | Review in a separate template console or API |
| Best fit | Sensitive recovery flows and strict review | Teams that need non-engineers to edit copy |

My default is application-owned reset copy, a small routing adapter, and a provider-neutral event contract. Use service-owned templates only when the person editing them can follow the same review, access, and retention rules as the application team.

Keep the token out.

The three-field boundary is enough for the first routing decision: a category, a case ID, and a reply address. Add a field only when a support agent can explain how it changes the queue or helps locate an authenticated case; this keeps review narrow while leaving room for a later operational need.

Review it weekly.

## What should the contact form actually route?

Start with a closed set of support categories. A free-form subject line is a poor routing key. In a fintech contact form, `password_reset`, `card_dispute`, and `account_verification` can map to different queues, but the message body should not become the policy engine.

The router should accept a normalized category, an internal case ID, and a short user-facing reply address. It should reject unknown categories before any message is sent. The queue payload can contain the case ID and a link to an authenticated support view; it does not need the reset token, full account profile, or pasted identity documents.

That distinction matters for GDPR work. The regulation is not a magic “compliant email provider” label. The controller still needs a lawful, documented process for personal data, and a processor relationship needs appropriate controls. A service that sends mail cannot decide the data classification for the SaaS.

Here is the core adapter. It has no commercial SDK dependency, and its return value is deliberately boring. Boring is easy to test.

```ts
type SupportQueue = "account-recovery" | "verification" | "disputes";
type ContactCategory = "password_reset" | "account_verification" | "card_dispute";

type SafeContact = {
  caseId: string;
  category: ContactCategory;
  replyTo: string;
};

const queueFor: Record<ContactCategory, SupportQueue> = {
  password_reset: "account-recovery",
  account_verification: "verification",
  card_dispute: "disputes",
};

export function buildQueueNotice(contact: SafeContact) {
  return {
    queue: queueFor[contact.category],
    caseId: contact.caseId,
    replyTo: contact.replyTo,
    category: contact.category,
  };
}
```

The decision rule is simple: if a field helps a support agent find the case, keep it; if it helps reconstruct a credential or identity document, leave it out. I would rather ask an agent to open an authenticated case than put a secret in an inbox that has a wider audience.

## Why keep password-reset templates under application ownership?

A reset email contains a capability: whoever holds its link may be able to start account recovery. The exact expiration and one-use behavior belong in the recovery design, but the message must reflect that design without an editor accidentally changing the instructions. The copy should state the request, the safe next action, and what to do if the recipient did not initiate it.

The template does not need a user's name, account balance, transaction history, or support conversation. It needs a reset URL generated for that request and a neutral subject. I keep the template beside the code that creates the reset intent so a reviewer can see the data path in one change.

I initially treated template editing as a content problem. That made the review boundary too soft. The useful question is: who can change a string that affects a security flow, and how do we know which version was sent? Once I asked that, versioned application templates became the smaller operational surface.

A minimal message model makes the rule enforceable.

```ts
type ResetMessage = {
  resetUrl: string;
  locale: "en";
  requestId: string;
};

export function renderResetMessage(message: ResetMessage): string {
  return [
    "We received a request to reset your password.",
    `Use this link to continue: ${message.resetUrl}`,
    "If you did not make this request, you can ignore this email.",
    `Request reference: ${message.requestId}`,
  ].join("\n\n");
}
```

The request reference is an internal correlation value, not an email address or customer identifier. In production, I would also treat the rendered output as sensitive during logs and tests. Logging the complete body to debug a delivery issue is a fast way to create a second data problem.

No exceptions.

## How do you verify the GDPR boundary before shipping?

Use a short data-flow review. Name the controller, processor, template owner, retention owner, and people who can inspect message events. Then answer four concrete questions: where is the reset URL created, where is the message rendered, what does the queue receive, and what remains in logs after the delivery event?

SPF is one part of sender authorization, not a complete privacy or deliverability program. RFC 7208 defines SPF and its evaluation model; it does not make a message lawful, prove that a template is correct, or authorize exposing support data. Keep those decisions separate.

For automated checks, assert absence as well as presence. A passing test should prove that the password-reset token is included only in the outbound message input and is absent from the queue notice, structured logs, and analytics event. It should also verify that an unknown category fails closed.

```ts
import { strict as assert } from "node:assert";

const notice = buildQueueNotice({
  caseId: "case_1842",
  category: "password_reset",
  replyTo: "support-reply@example.test",
});

assert.equal(notice.queue, "account-recovery");
assert.equal("resetToken" in notice, false);
assert.equal("accountBalance" in notice, false);
```

Do not turn this into a large test matrix for its own sake. The high-value cases are the category boundaries, the data exclusion rules, and the template version that a reviewer approved. A one-person team has limited review time. Spend it on the fields that could cause harm or a wrong queue.

## When is the runner-up better?

A service-owned template can be the better choice when a trained support or compliance operator must change wording without waiting for a deployment, and the service supports version history, scoped access, exportable audit events, and a clear retention policy. Those are requirements to verify, not assumptions to make from a dashboard screenshot.

The limitation is real: application ownership is a poor fit when approved operators need frequent copy changes and the deployment process cannot provide timely review. Service ownership is a poor fit when access is broad, event history is unavailable, or retention cannot be stated plainly.

The trade is control versus editing speed. Application ownership keeps security-sensitive copy close to the reset logic, but it adds a deployment step. Service ownership can shorten a copy change, but it adds another permission boundary and another place to inspect during an incident. Neither choice removes the need to test the rendered message.

That extra review is the cost.

For my weekly shipping rhythm, the threshold is practical: if template changes are rare and security-sensitive, keep them in the application review. If regulated operations require frequent controlled edits, define an explicit template contract and audit the external editor as part of the system. Outsource the undifferentiated delivery plumbing. Keep the recovery decision and its data rules where they can be reviewed.

The contact form and the reset email do not need the same ownership model. Route the form with a safe category and case ID. Keep the reset template and token policy with the recovery flow. That separation is the decision that survives provider changes, queue changes, and the next compliance review.

## References

- RFC 7208, Sender Policy Framework (SPF): https://datatracker.ietf.org/doc/html/rfc7208
- Regulation (EU) 2016/679, General Data Protection Regulation: https://eur-lex.europa.eu/eli/reg/2016/679/oj
- European Data Protection Board, Guidelines 07/2020 on the concepts of controller and processor: https://www.edpb.europa.eu/our-work-tools/our-documents/guidelines/guidelines-072020-concepts-controller-and-processor-gdpr_en
- OWASP Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
