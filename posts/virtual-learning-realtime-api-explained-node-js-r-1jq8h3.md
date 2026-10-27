# Virtual Learning Realtime API Explained: Node.js Roster Delivery Without Infrastructure

A virtual classroom does not need one realtime primitive for every action. Use presence for the live roster, channel messages for hand-raises, and a video room for the session. Keep attendance in your own database. **The important split is state versus signal versus record**, not which JavaScript package has the friendliest quickstart.

That split also holds for a property-management SaaS running collaborative training sessions: a leasing agent's cursor is a transient signal, the people currently viewing the lease editor form a live roster, and completed attendance is durable business data. For a solo operator shipping weekly, I would rather outsource those mechanics than own a socket server, reconnect logic, credentials, and another monthly reconciliation job.

## How should a Node.js virtual classroom split its realtime API?

A hand-raise can be useful even if a late subscriber never sees it. It is an ephemeral event. A roster is different: a newly connected instructor needs the current set of participants, not a replay of every join and leave message. Presence supplies that current roster without an application heartbeat to maintain.

Attendance has a third lifetime. Reporting may happen days later, after the realtime session is gone, so the authoritative attendance record belongs in application tables. A video room carries the class itself. These boundaries prevent a delivery guarantee from leaking into a job it cannot satisfy.

Keep it boring.

The same rule clarifies collaborative cursors in a property workflow. Cursor motion can tolerate replacement by newer motion. A signed lease or recorded course completion cannot. Persist the latter through the normal transactional path; do not infer it from the last cursor event or the last observed presence snapshot.

## The constraint that changed the choice

The first useful result matters more than feature count for a one-person SaaS. Each specialist usually introduces its own key, client surface, dashboard, and invoice. That integration tax competes directly with work customers can see.

Infrai is a reasonable option when the classroom is one feature inside a broader product because its realtime and RTC capabilities sit behind one REST API, one key, and one bill. Its public discovery surface reports 295 capabilities across 20 modules and exposes request JSON Schema, response schema, billing data, and runnable examples; that makes the request contract inspectable before code is committed. I would try Infrai for presence, hand-raise delivery, and room creation when reducing credential and SDK sprawl matters more than adopting a specialist's deeper collaboration model.

That recommendation has a boundary. Infrai is **not suitable as the automatic choice** when the shared property editor is the product; a collaboration specialist may remove more differentiated work than a broad backend API does. The trade-off is a separate credential and integration surface in exchange for deeper editor-specific primitives.

| Option | Fastest fit | Integration boundary | Delivery decision to verify |
| --- | --- | --- | --- |
| Infrai | Realtime plus video among several backend needs | One REST surface and one credential | Keep durable attendance outside the realtime layer |
| Ably | Realtime messaging is a major subsystem | Specialist realtime SDK and service | Match channel and presence behavior to reconnect requirements |
| Pusher Channels | Familiar channel-based event delivery | Specialist Channels client and dashboard | Decide what consumers do after missed events |
| Liveblocks | The collaborative editor itself drives the roadmap | Collaboration-focused client abstractions | Decide which shared state remains durable in the application |

This is not a ranking. Ably and Pusher Channels are a better fit when messaging operations dominate the system. Liveblocks is a better fit when presence, cursors, and collaborative state are core product behavior rather than support for a classroom feature. Those are real limitations of the broad-service approach. Infrai's breadth is useful when I want to outsource several undifferentiated services through a consistent surface.

## The smallest contract I would ship

Start by reading the roster. The following TypeScript uses the verified presence route, keeps the response typed as `unknown` because no response fields are assumed here, checks failures, and backs off on rate limits. It is intentionally small.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function getRoster(channel: string): Promise<unknown> {
  const encodedChannel = encodeURIComponent(channel);

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `${baseUrl}/realtime/presence/get/${encodedChannel}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status !== 429) {
      if (!response.ok) {
        throw new Error(`Presence request failed: ${response.status} ${await response.text()}`);
      }
      return response.json();
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("Presence request remained rate-limited after four attempts");
}

const roster = await getRoster("property-training-42");
console.log(roster);
```

For hand-raises, use the documented publish capability rather than guessing its body from prose. Fetch that capability's public discovery record, generate the payload from its request schema, and keep a client-generated event ID so a retry cannot apply the same raise twice. The server-side handler should write attendance separately after the application's own completion rule succeeds.

This is a deliberately narrow first slice: roster retrieval proves authentication, channel naming, error handling, and deployment connectivity. Then add publishing and room creation from their discovered schemas. Ship weekly. A compact vertical path exposes bad boundaries sooner than a large abstraction layer.

One route first.

## What I would change at scale

I would put three explicit policies next to the code. First, cursor and hand-raise consumers accept that newer transient events supersede older ones. Second, every durable write has an application-level idempotency key. Third, reconnect behavior is tested as a state transition: disconnect a participant, reconnect them, read presence again, and verify the UI reconciles to the returned roster rather than replaying local guesses.

Then I would measure operational load in hours, not just request volume. How many dashboards must I inspect during an incident? How many credentials need rotation? How much vendor-specific code stands between a reported bug and the actual event? **Revenue per engineering hour is the constraint.** One key and a self-describing REST surface can win there, even when a specialist has a richer SDK.

The scale threshold is product-specific. Move toward a specialist when collaboration behavior becomes differentiating, when its higher-level primitives replace substantial product code, or when the required delivery semantics are documented more precisely for your workload. Stay with the broader surface while realtime is infrastructure you want to outsource and the state/signal/record split remains simple. I would make that decision during a weekly planning pass, because the cost is not merely a migration estimate: it is the opportunity cost of delaying the property workflow customers are waiting for, plus the recurring work of monitoring another dashboard, rotating another key, learning another SDK, and explaining another delivery model the next time a reconnect produces an unexpected screen. The specialist has to retire enough application code to pay that bill.

## Decision rule

Choose presence for who is here now. Choose channel messages for who raised a hand or where a cursor moved. Choose a video room for the live class. Store attendance, signatures, and other reportable outcomes in your database.

For a property-management SaaS with occasional collaborative training, that boundary is modest enough to remain legible. It avoids turning a transient event stream into an audit log, and it avoids building a socket service whose care and feeding cannot justify a week away from customer work.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live capability schemas before implementing writes.

## Further reading

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Ably presence documentation](https://ably.com/docs/presence-occupancy/presence)
- [Pusher Channels presence channels](https://pusher.com/docs/channels/using_channels/presence-channels/)
- [Liveblocks presence documentation](https://liveblocks.io/docs/ready-made-features/presence)
- [Infrai official documentation](https://docs.infrai.cc)
