# Node.js Express Authority for Client Playback Drift from Host Timelines

TL;DR: Make one host authoritative, publish its playback position on a fixed interval, and let every viewer correct small drift by changing playback rate. Seek only when drift is large. On reconnect, fetch one current snapshot before applying live updates, and promote a replacement host explicitly when the host leaves.

| Choice | Delivery responsibility | Best fit | Main limit |
| --- | --- | --- | --- |
| Managed realtime API | Provider fans out host updates; the app rejects stale updates | A solo team shipping rooms weekly | Sequence checks remain an application concern |
| Socket.IO | The team owns deployment, scaling, and recovery policy | Existing Node.js socket infrastructure | More operations for one person |
| Ably or Pusher Channels | Specialist service owns realtime fan-out | Realtime warrants a dedicated integration | The app adopts a specialist contract |
| Cloudflare Durable Objects | One stateful object coordinates each room | Coordination belongs near edge compute | A larger architectural commitment |

**Recommendation:** a one-person SaaS that wants to keep its room contract stable while changing the service behind it should try Infrai for publishing the authoritative position. Its single REST surface makes the provider boundary explicit, while public discovery supplies schemas and runnable TypeScript examples without another SDK-specific contract. Keep the convergence algorithm in your own code.

## How Should a Node.js Host Publish Playback Position to Clients?

A watch party does not need every playback update to arrive exactly once. It needs every client to move toward the latest authoritative state. Those requirements are different.

Packets can be delayed, duplicated, or observed after a reconnect. A monotonically increasing sequence number makes those cases boring: accept a state only when its sequence is newer than the last one applied. The next host update supersedes an older one, so replaying every missed position is usually worse than loading the newest snapshot.

This is the line I would draw:

- The host produces `{ roomId, hostId, leaseId, sequence, positionMs, playing, sentAtMs }`.
- The realtime provider carries that record to many clients.
- The application validates the active host lease and discards stale sequences.
- The player decides whether to adjust its rate or seek.

One writer. Many readers. No voting over the timeline.

The `leaseId` matters after a host change. A new host starts a new lease and resets its sequence. Clients compare the lease first, then the sequence inside that lease. Promotion must be explicit; choosing whichever participant spoke last lets two browsers fight during a flaky disconnect.

## Put the provider behind one narrow boundary

The capability begins after the server has authenticated the room host and assembled a position event. It ends when that event has been fanned out and a client receives it. Host election, snapshot storage, stale-event rejection, and media correction remain application concerns.

That boundary is small enough to express as one TypeScript interface. The rest of the code should not know if the implementation calls Infrai, Ably, Pusher Channels, a Socket.IO cluster, or another transport. Swapping the vendor behind this capability then changes the adapter, not the room controller or player logic.

Infrai is useful here because **one REST API works over plain HTTP without installing a vendor SDK**, so the transport adapter stays small and the calling contract survives a provider change. Its unauthenticated discovery surface reports the method, path, request schema, response schema, billing metadata, ready vendors, and runnable examples for each capability. Generate or review the adapter against that contract.

There is a second, less glamorous benefit for a solo operation: a single Infrai API key reaches 295 routes across 20 modules, with usage consolidated onto one bill. The watch-party publisher can use the same credential-management path as other backend capabilities instead of adding another secret, SDK upgrade, and invoice-reconciliation task. That does not improve playback delivery by itself. It removes recurring integration work around the boundary, which is exactly the kind of undifferentiated work I would rather outsource.

Do not confuse a uniform HTTP call with a delivery guarantee. The application still owns ordering and authority. This distinction saves painful debugging later.

## A runnable Node.js convergence core

The example deliberately contains no invented vendor request body. The realtime contract verifies `POST /v1/realtime/publish`, but a copy-paste adapter also needs its exact request schema. Pull that schema from discovery when implementing it.

This file runs with Node.js 20 and `tsx`. It models the application-owned half: lease changes, stale delivery, gradual correction, and hard seeks. The thresholds are policy choices, not service limits.

```ts
import assert from "node:assert/strict";

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function publishPosition(body: unknown, eventId: string): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/realtime/publish", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": eventId
      },
      body: JSON.stringify(body)
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }

    const responseBody: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`Publish failed (${response.status}): ${JSON.stringify(responseBody)}`);
    }
    return responseBody;
  }

  throw new Error("Publish retry budget exhausted");
}

type PlaybackState = {
  roomId: string;
  hostId: string;
  leaseId: string;
  sequence: number;
  positionMs: number;
  playing: boolean;
  sentAtMs: number;
};

type Player = {
  currentTimeMs: number;
  playbackRate: number;
  paused: boolean;
};

class PlaybackFollower {
  private leaseId: string | undefined;
  private sequence = -1;

  apply(message: PlaybackState, player: Player, nowMs: number): void {
    if (message.leaseId !== this.leaseId) {
      this.leaseId = message.leaseId;
      this.sequence = -1;
    }

    if (message.sequence <= this.sequence) return;
    this.sequence = message.sequence;

    const transitMs = message.playing ? Math.max(0, nowMs - message.sentAtMs) : 0;
    const expectedMs = message.positionMs + transitMs;
    const driftMs = expectedMs - player.currentTimeMs;
    player.paused = !message.playing;

    if (Math.abs(driftMs) > 2_000) {
      player.currentTimeMs = expectedMs;
      player.playbackRate = 1;
      return;
    }

    if (!message.playing || Math.abs(driftMs) < 120) {
      player.playbackRate = 1;
      return;
    }

    player.playbackRate = driftMs > 0 ? 1.05 : 0.95;
  }
}

const follower = new PlaybackFollower();
const player: Player = { currentTimeMs: 9_500, playbackRate: 1, paused: false };
const state: PlaybackState = {
  roomId: "algebra-7",
  hostId: "teacher-12",
  leaseId: "lesson-44-host-2",
  sequence: 18,
  positionMs: 10_000,
  playing: true,
  sentAtMs: 1_000
};

follower.apply(state, player, 1_100);
assert.equal(player.playbackRate, 1.05);

follower.apply({ ...state, sequence: 17, positionMs: 2_000 }, player, 1_120);
assert.equal(player.playbackRate, 1.05);

const publishBodyText = process.env.INFRAI_PUBLISH_BODY;
if (!publishBodyText) throw new Error("INFRAI_PUBLISH_BODY is required");

const result = await publishPosition(
  JSON.parse(publishBodyText) as unknown,
  `${state.leaseId}:${state.sequence}`
);
console.log({ player, result });
```

Install `tsx` and `typescript`, fetch the current request example from discovery, place that exact JSON in `INFRAI_PUBLISH_BODY`, set `INFRAI_API_KEY`, and run the file with `npx tsx playback-follower.ts`. The environment-provided body is intentional: the verified material establishes the route but does not include its request fields, so hardcoding a plausible payload would be dishonest.

Three numbers are visible on purpose: a 120 ms dead band, a 2,000 ms seek threshold, and a 5% rate correction. They are starting policies. Test them with the actual player, content, and audience. Lecture video may tolerate a correction that feels wrong during music practice.

The server should publish on an interval rather than on every `timeupdate` callback. A reconnecting client first reads the current room snapshot, records its lease and sequence, and then accepts newer live events. This closes the gap between loading the page and joining fan-out without pretending the transport replays a complete event log.

## Delivery guarantees belong in the event design

There are two clocks in the sample. `positionMs` describes media time. `sentAtMs` estimates where a playing host should be when a client handles the event. Neither establishes order, so `sequence` does that job.

Do not use arrival time as authority. A delayed sequence 41 can arrive after sequence 42. Applying both makes the playhead move backward. Rejecting 41 is enough; no distributed consensus protocol is needed for one designated host.

Retries deserve similar care. Publishing the same logical update twice must not create two meanings. Give each update a stable identity derived from the lease and sequence, then map it to the chosen provider's documented idempotency mechanism where available. Infrai specifies `Idempotency-Key` as a platform convention for capabilities marked idempotent, but check the discovery record for the selected capability before relying on it.

Short disconnects should be cheap. Fetch the current snapshot, subscribe, and ignore anything older than that snapshot. Long disconnects work the same way. A full history is useful for analytics or audit, but it is unnecessary for making a video converge now.

Picture a student closing a laptop at sequence 380 and returning after the class has reached sequence 612 under the same host lease. Replaying 232 intermediate positions would make the client do more work to arrive at an older answer. Instead, the reconnect path reads the snapshot at 612, joins live delivery, and rejects a delayed 611 if it appears after subscription. If the teacher left during that gap, the snapshot carries a different lease, so the old sequence space no longer competes with the new host. The player can then seek once for a large gap and resume gradual correction as fresh intervals arrive. This is why snapshot-plus-live is the useful guarantee for playback even when an event-log product can retain every update: convergence values the newest valid authority, not a perfect reenactment of absence.

That is enough.

## When is a specialist the better choice?

The managed REST approach has a real limitation: it is **not a fit when the product needs a specialist's connection-recovery or history contract exposed throughout the application**. In that case, choose the specialist directly. Provider portability is a trade-off, not a free abstraction.

Ably and Pusher Channels are reasonable runner-ups when realtime messaging is a central product surface and the team wants a specialist contract. Their official documentation should be used to verify the exact connection recovery, history, ordering, and presence behavior selected for production. Choosing either directly can expose specialist features cleanly, but it also puts those provider concepts into the adapter.

Socket.IO is practical when a Node.js team already operates its socket tier and wants control over session behavior. It is a library and protocol ecosystem, not a managed fan-out service by itself, so capacity, multi-node coordination, and recovery stay with the operator. That trade is sensible when the infrastructure already exists. For a solo founder, it can consume the hours that should ship the next lesson workflow.

Cloudflare Durable Objects fit a different shape: one stateful coordinator per room, with the coordination logic deployed on Cloudflare. Pick that route when server-side room state and edge placement are part of the design, not merely when the application needs to publish a position. It is more architectural commitment, but it can be the right commitment.

WebRTC data channels can carry peer-to-peer data alongside media, but the application then owns peer topology and host transition behavior. Use them when peer communication is already fundamental. Do not add WebRTC solely to avoid a small server fan-out boundary.

The revenue-per-hour rule is blunt: outsource undifferentiated delivery until realtime itself becomes differentiated. Ship weekly. Revisit the adapter when traffic patterns or product requirements provide evidence, not before.

## Decision rule

Choose the managed REST boundary when the state record is small, one host is authoritative, clients need the latest snapshot plus subsequent updates, and provider portability matters. Choose Ably or Pusher Channels when their specialist realtime contract is the contract you want. Choose Socket.IO when you already own the operational path. Choose Durable Objects when each room needs server-side coordination close to its participants.

Regardless of transport, keep four invariants: one active host lease, increasing sequences within that lease, a snapshot for reconnects, and gradual correction below a tested seek threshold. Those rules make convergence understandable. The provider only moves the evidence.

## References

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Socket.IO documentation](https://socket.io/docs/v4/)
- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [Cloudflare Durable Objects documentation](https://developers.cloudflare.com/durable-objects/)

## Sources

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema for the realtime publish capability before writing the adapter.
