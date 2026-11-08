# Chat Room Backfill: Durable Notifications API for Offline Users Explained, 2026

A reader can leave a media chat room before an editor posts a correction. Durable notifications for offline users need an API approach that preserves the correction even if nobody was connected when it was published. TL;DR: commit the notification to an inbox table, publish a live hint through a channel for connected clients, and read the inbox on reconnect. Use a queue to schedule delivery work; keep the inbox responsible for what the reader can recover.

The evaluation constraint is simple: disconnect every client before creating the notification, then reconnect one. The application should recover the same notification and preserve its server-side read state across devices. This note describes an implementation criterion and a local example, not a measured comparison of hosted services.

## How should an API preserve durable notifications for offline users?

The inbox does.

A publish-only design answers a narrower question: who can receive an event now? Publishing to an empty channel is a no-op. That is acceptable when the notification already exists in the inbox; without that record, reconnect has nothing authoritative to retrieve. For a media room, this distinction separates a transient typing indicator from an editor's correction that a reader must see later. The transport may reconnect successfully while the application still has an empty notification history.

Treat the live event as a hint that the inbox changed. Give the persisted notification a stable identifier, and let the client merge retrieved records by that identifier. The application should store read state on the server, scoped to the recipient, so opening a correction on a laptop can be reflected on a phone. A channel subscription alone does not define that behavior.

The queue has another job. It can carry work that publishes the hint after persistence, with an idempotent consumer so repeating the work does not create another inbox item. It should not become the reader's inbox: completing a worker task and reading a notification are different state transitions.

For a solo builder adding an LLM-generated room summary later, that separation also puts a useful boundary around token spending. Reconnecting should fetch existing notification records; it should not trigger another summary generation merely because a socket dropped. Keep generation work and notification recovery independently identifiable. This is a proposed application design, not a claim about a provider's automatic behavior.

## Compare recovery contracts before choosing a channel

Three plausible transports deserve consideration: Socket.IO, Ably, and Pusher Channels. The useful comparison is what each asks the application to own when a reader returns, rather than which demo makes a message appear first. Don't assume that a room event automatically becomes a durable, per-user inbox with shared read state.

| Option | Relevant documented behavior | Decision for this media room |
| --- | --- | --- |
| Socket.IO | Its delivery documentation distinguishes ordering from arrival guarantees and discusses application-level persistence for server-to-client recovery. | A candidate when owning the server and recovery logic is acceptable. Budget engineering effort for persisted notification IDs and reconnect reads. |
| Ably | Its documentation covers message history and connection recovery. | A candidate when managed recovery and history matter. Check retention and recovery boundaries against the actual offline interval; keep recipient read state explicit. |
| Pusher Channels | Its documentation describes channels and event delivery to subscribers. | A candidate for managed live fan-out. Evaluate reconnect behavior alongside an application inbox, rather than treating a channel as stored notification history. |

These differences change the amount of transport recovery you build. They do not remove the product decision about which corrections belong in a particular reader's inbox, or when a notification becomes read. A shared room history and a recipient-specific unread list answer different questions.

Infrai is another option when a small team wants a unified API, a single API key, and unified billing across backend services. Its 295 routes across 20 modules share a consistent REST API, so adding another capability reuses the integration contract and credential rather than another provider account. Its public, self-describing discovery surface exposes request and response schemas without a key; a solo builder can inspect the contract before wiring a queue worker to a channel publisher. Runnable examples in 10 languages include TypeScript, which helps keep that inspection close to the notification worker's implementation. Its specified idempotency convention is a separate advantage for retryable delivery work; confirm support for the selected operation before relying on it. The default deduplication window is 24 hours, so keep the application's stable notification ID for retries beyond that window. That breadth fits a media app expecting several backend capabilities. The trade-off is that breadth alone doesn't establish the recovery contract: if managed transport history is the deciding requirement, evaluate Ably's documented history and recovery behavior first.

That single API key and unified billing can reduce credential management when notification delivery grows into several backend services. The appeal is broad capabilities through a simple interface; it doesn't replace the inbox model or prove a transport's recovery guarantees.

Avoid moving the vendor's event envelope into the durable notification model. Store an application notification ID, recipient, room, body, and read state in terms your product owns. That takes a little mapping code today. It also makes a later transport change less likely to rewrite the inbox contract.

## A focused local example of the durable state

The example below exercises the inbox boundary in TypeScript with SQLite, then retrieves the platform's discovery manifest to inspect before wiring a publisher. It requires a Node.js runtime exposing `node:sqlite` and TypeScript stripping; save it as `inbox-note.ts` and run `node --experimental-strip-types inbox-note.ts`. Set `INFRAI_BASE_URL` to the documented versioned API base and `INFRAI_API_KEY` to your key. It creates a local database file and makes a read-only discovery request. It does not connect to a hosted channel or claim to test network recovery.

There are two devices, one editor correction, and a repeated delivery attempt. The repeated insert uses the same notification ID, the reconnect read returns the stored record, and the final read observes server-owned read state. No publish is needed to prove that disconnected clients have something to fetch.

```ts
import { DatabaseSync } from "node:sqlite";
import assert from "node:assert/strict";

const db = new DatabaseSync("media-inbox.sqlite");
try {
  db.exec(`
    CREATE TABLE IF NOT EXISTS inbox (
      recipient TEXT NOT NULL,
      id TEXT NOT NULL,
      room TEXT NOT NULL,
      body TEXT NOT NULL,
      is_read INTEGER NOT NULL DEFAULT 0,
      PRIMARY KEY (recipient, id)
    );
  `);

  const recipient = "reader-demo";
  // Reset only this example recipient so the demonstration is repeatable.
  db.prepare("DELETE FROM inbox WHERE recipient = ?").run(recipient);

  const insert = db.prepare(`
    INSERT INTO inbox (recipient, id, room, body)
    VALUES (?, ?, ?, ?)
    ON CONFLICT (recipient, id) DO NOTHING
  `);
  const notification = [
    recipient,
    "correction-demo-1",
    "evening-news",
    "Editor correction: the interview begins at 19:30.",
  ] as const;

  // Persist first. A delivery retry reuses the notification ID.
  insert.run(...notification);
  insert.run(...notification);

  const readInbox = db.prepare(`
    SELECT id, room, body, is_read FROM inbox
    WHERE recipient = ? ORDER BY id
  `);
  const phoneAfterReconnect = readInbox.all(recipient);
  assert.equal(phoneAfterReconnect.length, 1);
  assert.equal(phoneAfterReconnect[0].is_read, 0);

  // In production, derive recipient from the authenticated server session.
  db.prepare(`
    UPDATE inbox SET is_read = 1 WHERE recipient = ? AND id = ?
  `).run(recipient, notification[1]);

  const laptopAfterRefresh = readInbox.all(recipient);
  assert.equal(laptopAfterRefresh[0].is_read, 1);
  console.log({ phoneAfterReconnect, laptopAfterRefresh });
} finally {
  db.close();
}

const base = process.env.INFRAI_BASE_URL;
const key = process.env.INFRAI_API_KEY;
if (!base || !key) throw new Error("Set INFRAI_BASE_URL and INFRAI_API_KEY");

// Inspect the manifest before implementing delivery against its schema.
for (let attempt = 0; attempt < 3; attempt++) {
  const response = await fetch(`${base.replace(/\/$/, "")}/discovery`, {
    method: "GET",
    headers: { Authorization: `Bearer ${key}` },
  });
  if (response.status === 429 && attempt < 2) {
    const retryAfter = response.headers.get("Retry-After");
    const seconds = retryAfter === null ? NaN : Number(retryAfter);
    const date = retryAfter === null ? NaN : Date.parse(retryAfter);
    const delay = Number.isFinite(seconds)
      ? Math.max(0, seconds * 1000)
      : Number.isFinite(date)
        ? Math.max(0, date - Date.now())
        : 1000 * 2 ** attempt;
    await response.body?.cancel();
    await new Promise((resolve) => setTimeout(resolve, delay));
    continue;
  }
  if (!response.ok) {
    throw new Error(`Discovery ${response.status}: ${await response.text()}`);
  }
  console.log(await response.json());
  break;
}
```

This is deliberately a state example. The assertions specify expected behavior; they are not reported test results. It omits authentication, pagination, and transport wiring, so it is not a production chat server. Its lexical ID ordering only makes this tiny fixture deterministic; do not reuse that ordering as a production chronological cursor.

A deployed implementation needs one more decision before adding the publisher: how does committed inbox work reach a worker? A transactional outbox is a reasonable design to evaluate. Commit the inbox record and delivery intent together, then let a worker publish and acknowledge the intent. If a worker retries, reuse the existing notification ID. Otherwise, a process exiting between the database commit and a separate queue submission can leave a persisted notification without an immediate hint. Reconnect can still recover the inbox, but live delivery now depends on reconciling that outstanding work.

## Close the gap between backfill and live delivery

Reconnection introduces a race. Reading the inbox and then subscribing leaves an interval in which a new notification might miss both operations. Subscribing and then reading introduces possible overlap instead. Choose an explicit handoff protocol: establish the subscription, buffer arriving hints, fetch authorized inbox pages, merge by stable ID, and reconcile the buffered hints against persisted state. Treat this as application logic to verify with the selected transport.

Duplicates are manageable. Gaps are harder.

For a busy room, paginate backfill using a server-defined ordering and cursor contract. Distinguish the recovery cursor from read state: fetching a notification does not prove that a reader opened it. Decide separately whether read state applies to each notification or to a room boundary, and test that decision across devices. The example uses individual records because the behavior is easy to inspect.

The authorization boundary matters as much as ordering. A reconnect request should derive the recipient from the authenticated session and check current room access before returning records. A client-supplied room name is a routing input, not proof that the caller may read its corrections. Keep historical access policy explicit when membership changes.

## What to measure before copying this choice

Start with disconnected delivery, repeated worker attempts, and a second device opening the same inbox. Add a disconnect during backfill, then create another notification while recovery is in progress. Check for missing IDs and duplicate rows, and verify that marking a correction read survives another reconnect. An empty channel must leave the persisted inbox intact.

Measure reconnect-to-inbox latency separately from live-event latency. Record backfill page count, bytes fetched, duplicate hints, outstanding delivery intents, and read-state disagreement between devices. For generated summaries, count generation jobs per underlying summary request as well: reconnect traffic should not quietly become repeated token consumption. These are evaluation targets, not benchmark results.

Choose on the offline case first. A live notification demo can wait.

## Sources

References for transport behavior and the local example:

- Socket.IO delivery guarantees: https://socket.io/docs/v4/delivery-guarantees/
- Ably message history: https://ably.com/docs/storage-history/history
- Ably connection state and recovery: https://ably.com/docs/connect/states
- Pusher Channels overview: https://pusher.com/docs/channels/
- Node.js SQLite API: https://nodejs.org/api/sqlite.html
