# After Payment: Email API Event Polling for US/EU SaaS Order Receipts Without Webhooks

For an edtech SaaS, an order receipt is part of the payment experience. The least complex design that meets a delivery-reliability requirement is a durable send queue, a custom authenticated domain, and a pull-based event worker when inbound webhooks are forbidden. Keep the receipt request and the provider message ID in your database before polling for outcomes.

That is the short answer. A green send response is acceptance by an email service, not proof that a student received a receipt.

The decision changes if the network can accept inbound traffic. A webhook can deliver status changes quickly, but it brings endpoint authentication, replay, deduplication, and queue retention into the application. Polling removes the public receiver and adds cursor storage, schedule lag, pagination handling, and a reconciliation job. Those are engineering choices, not API cosmetics.

## Which email API shape fits an order receipt flow?

Start with the operating boundary. A webhook-to-queue path is usually the simplest way to get near-real-time events when a public ingress endpoint is acceptable. A complete event feed that can be polled is a better match for a private US/EU SaaS network. A per-message lookup is a narrow recovery tool: it can recheck IDs the application already knows, but it cannot reveal a send that was lost before its ID was persisted.

| Integration shape | Application owns | Good fit | Main trade-off |
|---|---|---|---|
| Webhook to durable queue | Receiver auth, deduplication, retention | Immediate status updates and controlled public ingress | More exposed infrastructure |
| Pull-based event feed | Cursor, scheduler, pagination, reconciliation | No inbound webhooks and a scheduled worker | Delayed feedback and repeated polls |
| Per-message lookup | Message inventory and scan schedule | Low volume or targeted diagnosis | Poor discovery of unknown events |
| Managed queue notification | Permissions, visibility timeout, dead letters | An existing queue platform | More policy to operate |

Resend, Postmark, SendGrid, Amazon SES, and Mailgun are useful comparison points because their documentation reflects different product boundaries and operating models. Resend presents a focused developer email workflow. Postmark separates transactional delivery from broadcast concerns. SendGrid and Mailgun are relevant when a team already uses broader sending and suppression tooling. Amazon SES fits an AWS-centered team that is prepared to own more of the delivery operations. None of those labels answers the acceptance question by itself.

For my revenue-per-hour calculation, the winner is the option that makes a failed receipt explainable without a midnight mailbox hunt. Ship weekly. Outsource the undifferentiated transport work, but retain the state machine and the audit trail.

## How should a SaaS handle custom domain, DKIM, suppression, and polling?

Treat domain setup as part of the delivery contract. DKIM signs a message with a domain key; DMARC checks alignment and policy. A custom domain therefore needs DNS ownership, a test that inspects the domain visible to the recipient, and a rotation owner for selectors. RFC 6376 and RFC 7489 define the relevant mechanisms. A passing DNS check is necessary. It is not a mailbox-delivery guarantee.

Suppression belongs before retry policy. A permanent failure, complaint, and unsubscribe should remain distinguishable in the internal record, even when all three prevent a later non-required send. A temporary failure takes another branch. For a payment receipt, be explicit about the required-message policy and its legal review; do not silently turn a failed transactional send into an unbounded retry loop.

Polling needs the same discipline. Store a cursor, process events idempotently, and commit the cursor with the event effects. Alert on cursor age and event-to-ingest lag, not just worker liveness. A scheduler that wakes up successfully can still leave the receipt ledger stale.

The US/EU question is larger than the provider's region label. Review where recipient addresses, message bodies, event records, backups, support access, and your polling database are processed or retained. I’m not sure a regional badge resolves every privacy question; the provider's current terms and your counsel have to settle what the architecture diagram cannot.

## What does a pull-only event worker need to prove?

The worker should make a crash boring. It reads from the saved cursor, inserts each event only once, updates suppression state where appropriate, and advances the cursor in the same transaction. If the process dies before commit, the next run reads the page again. If it dies after commit, the stored cursor prevents a second state transition.

No shortcuts.

For an order receipt, that transaction is the boundary between payment truth and mailbox evidence. The payment row should already contain the idempotency key before the send request is made. The send result should add the provider message ID without overwriting the order ID. Each later event should be attached to both identifiers, while a duplicate event ID should become a no-op. If a page contains an event without a recipient, the adapter should preserve its event ID and kind, place it in a quarantine table, and leave the cursor decision explicit rather than throwing away the whole page. That gives an operator something concrete to inspect when a student says the receipt never arrived, and it prevents a malformed record from repeatedly blocking unrelated orders.

```ts
type EventKind =
  | "delivered"
  | "temporary_failure"
  | "permanent_failure"
  | "complaint"
  | "unsubscribe";

type DeliveryEvent = {
  id: string;
  messageId: string;
  recipient?: string;
  kind: EventKind;
  occurredAt: string;
};

type EventPage = {
  events: DeliveryEvent[];
  nextCursor: string | null;
};

type EventSource = {
  readPage(cursor: string | null): Promise<EventPage>;
};

const suppressingKinds = new Set<EventKind>([
  "permanent_failure",
  "complaint",
  "unsubscribe",
]);

async function ingestPage(db: any, source: EventSource, stream: string) {
  const cursor = await db.getCursor(stream);
  const page = await source.readPage(cursor);

  await db.transaction(async (tx: any) => {
    for (const event of page.events) {
      const firstSeen = await tx.insertEventIfAbsent({
        eventId: event.id,
        messageId: event.messageId,
        kind: event.kind,
        occurredAt: event.occurredAt,
      });

      if (firstSeen && event.recipient && suppressingKinds.has(event.kind)) {
        await tx.upsertSuppression({
          address: event.recipient,
          reason: event.kind,
          sourceEventId: event.id,
        });
      }
    }

    await tx.setCursor(stream, page.nextCursor);
  });

  return page.events.length > 0 || page.nextCursor !== null;
}
```

The important fields are boring: order ID, message ID, event ID, event kind, occurred-at time, and cursor. Boring is good. A receipt support ticket should be traceable from payment settlement to the send attempt to the terminal event without searching raw provider logs.

## Where do the alternatives stop fitting?

The catch is that polling is not a universal substitute for webhooks. It is unsuitable when the business needs second-level escalation and the event feed's retention window cannot cover a worker outage. In that case, use a webhook-capable integration backed by durable storage. Polling is also a poor fit when the team will not own cursor recovery, pagination tests, and lag alerts; a managed notification path may be the better operational choice.

The reverse limitation matters too. A webhook design is a bad choice when the SaaS cannot expose a verified public receiver or when inbound traffic needs a security review that the launch schedule cannot absorb. Stick with a pull feed then, provided its documented retention and pagination behavior cover the recovery window.

Your mileage may vary on provider comparisons because plan limits, regional terms, and event retention can change. Record the version of each acceptance test and rerun it before a migration. The choice is conditional, and that is healthier than a permanent winner.

For an edtech launch, I would test a custom-domain receipt after payment settlement, duplicate worker execution, a cursor reset, a page boundary, a temporary failure, a permanent failure, a complaint, an unsubscribe, and a US/EU data-flow review. Measure cursor age, event-to-ingest lag, duplicate events, quarantined records, and suppression-write latency. Use the results to decide whether the API's event model matches the reliability promise.

## References

- https://www.rfc-editor.org/rfc/rfc6376
- https://www.rfc-editor.org/rfc/rfc7489
- https://www.rfc-editor.org/rfc/rfc8058
- https://resend.com/docs/introduction
- https://postmarkapp.com/developer
- https://docs.sendgrid.com/for-developers/sending-email
- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://documentation.mailgun.com/docs/mailgun/
- https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
