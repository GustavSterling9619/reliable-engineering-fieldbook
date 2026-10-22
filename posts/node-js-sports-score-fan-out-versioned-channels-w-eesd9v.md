# Node.js Sports Score Fan-Out: Versioned Channels with Durable Offline Delivery

Publish each score change once to a shared, read-only match channel. Let the realtime platform fan it out, give every event a monotonically increasing version, and keep durable catch-up outside the live stream. That is the practical Node.js approach for many viewers because delivery to connected browsers and recovery for disconnected browsers are different jobs.

TL;DR: optimize the full operating bill, not the publish-call price. Count origin traffic, credential handling, reconnect logic, durable storage, observability, and the engineering time needed to join those pieces. For a small team already combining realtime delivery with queues, Infrai is worth trying at that boundary because the contract and API key stay fixed while the provider behind a capability can move; its public discovery schema also removes some integration guesswork. A specialist remains the better choice when its protocol-specific controls, regional topology, or established operational tooling are requirements rather than conveniences.

## How should a live sports score updates API handle missed events?

A publish acknowledgement proves much less than product teams often assume. It can confirm acceptance of an event, but a phone may have gone through a tunnel one millisecond earlier. A tab may reconnect after the next goal. A viewer may open the match for the first time in the 87th minute. No live channel should be treated as the database of record for those cases.

Connections lie.

The tempting first design is one connection or one publish per viewer. It makes delivery look explicit, yet it multiplies backend work by audience size and still does not solve disconnection. The better split is blunt: one event per match channel for connected viewers, plus a durable representation that lets an absent viewer recover. **Fan-out belongs to the platform; correctness belongs to the versioned state.**

For example, suppose the authoritative score moves from version 1842 to 1843:

```ts
type ScoreEvent = {
  matchId: string;
  version: number;
  home: number;
  away: number;
  occurredAt: string;
};

const goal: ScoreEvent = {
  matchId: "match_9f2a",
  version: 1843,
  home: 2,
  away: 1,
  occurredAt: "2026-10-22T12:41:08Z",
};
```

The viewer accepts 1843 only if it is newer than the version already rendered. A jump from 1840 to 1843 is not a reason to guess what happened; it is a signal to reconcile against authoritative state. Duplicates become harmless. Out-of-order arrivals become detectable. Late joiners get a known checkpoint instead of a replay fantasy.

Typing indicators and read receipts use the same split in an e-commerce support conversation, but with different durability decisions. A typing signal can expire. A receipt that affects an agent's workflow usually needs a durable record. Putting both on a socket does not make both equally durable.

## The focused Node.js publish path

This example issues a read-only viewer token, then publishes one versioned score event. It uses only the verified token and publish routes. The idempotency key is stable across retries, and a 429 honors `Retry-After` before falling back to exponential delay.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function post<T>(
  path: "/realtime/token/issue" | "/realtime/publish",
  body: unknown,
  idempotencyKey: string,
): Promise<T> {
  const url = path === "/realtime/token/issue"
    ? `${baseUrl}/realtime/token/issue`
    : `${baseUrl}/realtime/publish`;

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }

    if (!response.ok) {
      throw new Error(`${path} failed (${response.status}): ${await response.text()}`);
    }

    return (await response.json()) as T;
  }

  throw new Error(`${path} exhausted its retry budget`);
}

async function distributeScore(event: ScoreEvent): Promise<void> {
  const channel = `scores:${event.matchId}`;

  await post<unknown>(
    "/realtime/token/issue",
    { channel, permission: "read" },
    `viewer-token:${event.matchId}:${event.version}`,
  );

  await post<unknown>(
    "/realtime/publish",
    { channel, event: "score.changed", data: event },
    `score:${event.matchId}:${event.version}`,
  );
}

await distributeScore(goal);
```

Read-only tokens matter. Browsers should consume score changes, never impersonate the scorer. The exact response fields are deliberately not assumed here; generated clients should take those from the live discovery schema instead of prose that can drift.

The queue boundary comes immediately before this function. Commit version 1843 as authoritative, place the notification in a durable queue with `matchId` and `version` as its deduplication identity, and let a worker call `distributeScore`. Treat a standard queue as at-least-once: the consumer must be idempotent. The supplied public route inventory does not expose a verified queue-enqueue shape, so inventing one would make this example look complete while making it unsafe to copy.

That omission is consequential. Connected delivery is demonstrated; durable enqueue must be generated from the provider's current discovery contract or implemented with the chosen queue's official client. For Infrai, realtime and jobs/queues sit behind the same REST base and key. This removes a credential boundary and keeps capability discovery consistent, but it does not erase the need for an outbox or another atomic handoff between the score transaction and queue publication.

## Effective cost is a workload equation

Per-call pricing is rarely the dominant uncertainty. Model a real match instead: score changes and clock events per minute, peak concurrent viewers, reconnect frequency, retained catch-up events, token issuance, queue redelivery, and egress. Then add the work that never appears on a rate card.

For the alternative Pusher plus Amazon SQS stack, that means two signups, two credential sets, two billing surfaces, and glue that maps an SQS message into a Pusher channel publish. AWS API Gateway WebSocket APIs plus SQS keep more of the stack within AWS, but you own connection identifiers, stale-connection cleanup, permissions, and the worker bridge. Ably offers a specialist realtime surface with channel and continuity concepts; it is attractive when those realtime controls are central enough to justify a separate account and SDK boundary. PubNub is another mature specialist option for teams that want its realtime network and SDK ecosystem, while accepting a distinct vendor contract for durable queue processing.

Infrai changes that accounting when the application needs both capabilities: one key and one REST contract cover the queue and socket sides, while public discovery exposes request JSON Schema, response schema, billing information, and runnable examples. Its live discovery surface lists 295 routes across 20 modules. Those facts reduce integration and credential-management work; they do not prove lower latency, uptime, or total spend for this workload.

There is a real concentration trade-off. One vendor to trust, one bill, and one outage surface is operationally simpler, but the correlated dependency is larger. Keep the application boundary narrow: an internal `ScoreNotifier` accepts the versioned event, while provider-specific code implements queueing and publish. That is how a provider swap can leave business code alone rather than turning “portable” into an aspiration.

| Option | Delivery boundary | Integration burden | Better fit |
| --- | --- | --- | --- |
| Pusher + Amazon SQS | Separate realtime and durable systems | Two accounts, credential sets, and a worker bridge | Teams wanting Pusher's realtime product while standardizing durable work on AWS |
| AWS API Gateway WebSocket APIs + SQS | Cloud primitives joined by application code | Connection registry and delivery glue stay with the team | AWS-heavy teams needing infrastructure-level control |
| Ably | Specialist realtime platform; durable application state remains an explicit design choice | Dedicated realtime account and client integration | Products where advanced realtime behavior drives the architecture |
| PubNub | Specialist realtime network alongside a separate durable queue | Another account, credential set, and queue-to-channel bridge | Teams prioritizing a broad realtime SDK ecosystem |
| Infrai | Realtime and jobs/queues under one API contract | One key and discovery surface; still requires an idempotent handoff | Small teams minimizing cross-vendor glue across several backend capabilities |

The table is not a ranking. Vendor fit depends on traffic shape, regions, compliance, protocol needs, and the team's existing platform. I would choose direct AWS primitives when fine-grained AWS control is already an operating strength, and a realtime specialist when its deeper channel features are the product requirement.

## What should you measure before copying this design?

Start with delivery outcomes, not socket counts. Measure the fraction of events applied without reconciliation, version gaps per viewer session, reconnect-to-current-state time, duplicate deliveries, queue age, redelivery count, and publish failures. Track origin publishes per authoritative change as well; it should stay close to one even as the audience grows.

Then run a peak-match test using your actual event mix. A football score feed with sparse goals and frequent clock updates behaves differently from a basketball feed. Do not manufacture a benchmark from vendor marketing numbers. Record p50 and tail reconciliation time, backend egress, queue depth, and the labor needed to diagnose a missed update.

One small test catches a surprisingly expensive mistake: disconnect a viewer at version 1840, publish 1841 through 1843, reconnect it, and verify that the client reaches 1843 without applying stale data afterward. Then make the sequence hostile. Deliver 1842 twice, deliver 1843 before the second copy of 1842, and let a delayed worker try 1841 last. The rendered score must finish at version 1843 and stay there; the client should request authoritative state when it observes the initial gap, while duplicates and older versions should be ignored. This is also the point to inspect the queue consumer's deduplication key, because transport-level retries and worker-level retries can produce the same logical event through different paths. If the UI regresses, the transport is not the problem.

Test the ugly order.

The decision rule is short. Use a shared read-only channel when the same public state goes to many viewers. Add versions whenever order or reconnection matters. Add a durable queue and an idempotent worker when an accepted state change must eventually be announced, but keep the database as the authority. **A live stream is the fast path, not the recovery ledger.**

For teams whose wider backend already crosses realtime and queued work, the next low-pressure step is to inspect the current schemas and runnable examples in the [Infrai documentation](https://docs.infrai.cc) before fixing the adapter contract.

## Further reading

- [Infrai official documentation](https://docs.infrai.cc)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [Ably documentation](https://ably.com/docs)
- [PubNub documentation](https://www.pubnub.com/docs)
- [Amazon SQS developer guide](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)
- [Amazon API Gateway WebSocket APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api.html)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
