# Fintech Daily Report Queue: Rate-Limited Email Sending After Reservation Expiry

Use a durable queue with a rate-capped HTTPS consumer, but keep reservation state and email intent in the database. The queue is transport, not truth. For a fixed hold window, a small sweeper should claim expired reservations with a conditional update, record an outbox event in the same transaction, and let a separate dispatcher send the daily report under the provider's limit. That is the least complex design that still gives an operator a clean replay path.

A reservation begins in `held` with an `expiresAt` timestamp. A scheduled sweep finds due rows and enqueues their IDs. The public HTTPS consumer attempts a compare-and-set transition to `expired`; a duplicate delivery becomes a harmless no-op. That transition also appends a report item to an outbox. At report time, another queue drains those items into recipient-specific emails at a configured pace. The important separation is easy to miss: expiring money-related state and delivering a report about it have different failure and recovery rules.

The split matters.

## Should a daily report email queue use a public HTTPS subscriber?

Recovery should start from durable application state, not from whatever the subscriber happened to receive. The scheduler may run twice, a push may be redelivered, and an email provider may ask the sender to slow down. None of those events should reopen a reservation, extend a hold, or create a second report item. The database transition therefore needs a predicate equivalent to `status = held AND expires_at <= now`, plus a uniqueness rule for the outbox event.

Keep the endpoint boring. It authenticates the queue, parses one small command, runs the conditional transaction, and acknowledges only after commit. It doesn't send email inline. A fast acknowledgement avoids coupling queue delivery timeouts to the much slower and less predictable email call, while the outbox makes the later send recoverable.

Small handlers recover.

This also answers the push-versus-pull question. A public HTTPS subscriber is reasonable when the queue can cap dispatch rate, authenticate requests, and retry non-success responses. A pull worker is the better fit when operators need direct control over prefetch, long processing times, or a private network boundary. Neither choice changes the state machine. **The recoverable unit is the database command, not the HTTP request.**

One warning: don't use exponential backoff as the rate limiter. Backoff reacts to failed attempts; pacing controls normal throughput. Configure a steady dispatch ceiling below the documented provider quota, then honor a valid `Retry-After` response by deferring that item. Add jitter to retry delays so a batch does not wake up in lockstep. The exact safety margin depends on whether the provider's quota is global, per account, or per destination; I'm not sure a generic percentage is defensible without that contract.

Retries are not pacing.

## Govern expiry through durable reservation state

Three identifiers matter: `reservationId` for the business object, `expiryEventId` for the one-time state transition, and `deliveryKey` for one report sent to one recipient on one report date. They solve different duplicate paths. Reusing a queue message ID for all three looks convenient, but it ties correctness to retention and redelivery policies outside the application.

For example, suppose a hold expires at `14:05:00Z`. The sweeper publishes at 14:05:12, the consumer commits at 14:05:13, and its acknowledgement is lost. A redelivery at 14:06 sees `expired` and returns success without inserting another outbox row. Hours later, the report dispatcher claims the unique key `ops@example.test:2026-08-20`; if its process stops after the provider accepts the message but before the local success marker commits, the result is ambiguous. No queue can erase that ambiguity unless the email provider accepts an idempotency key. The practical response is to store the provider message ID when available, suppress confirmed duplicates locally, and make the report content tolerant of an occasional duplicate rather than pretending exactly-once email exists.

Ambiguity remains.

Short failure paths deserve explicit states. Use `pending`, `leased`, `sent`, and `dead` for delivery records, with lease expiry rather than a permanent `processing` flag. An operator can then reclaim abandoned leases after a deploy, inspect poison messages without blocking the whole batch, and replay a bounded date range. Keep attempts and `nextAttemptAt` beside the delivery record. The queue can still schedule work, but an audit query can explain what remains.

Retention is a cost decision. Keeping full rendered email bodies is convenient for replay but can duplicate sensitive report data and inflate storage. Keeping a template version plus immutable input references is leaner, provided those references remain available for the audit period. In fintech, I would choose the retention rule before choosing queue machinery because deletion and evidence requirements shape the schema.

## Implement the narrow TypeScript worker

The example below isolates queue semantics behind interfaces. It assumes the database implementation performs `expireAndRecord` atomically and enforces a unique expiry event per reservation. The HTTP framework and signature scheme are deliberately left at the edge because those choices depend on the deployment environment.

```ts
type ExpiryCommand = {
  reservationId: string;
  dueAt: string;
  attempt: number;
};

type ExpiryResult = "expired" | "already-final" | "not-due";

interface ReservationStore {
  expireAndRecord(input: {
    reservationId: string;
    dueAt: Date;
    observedAt: Date;
  }): Promise<ExpiryResult>;
}

interface RequestVerifier {
  verify(headers: Headers, rawBody: string): Promise<boolean>;
}

interface Clock {
  now(): Date;
}

type WorkerResponse = { status: number; body: string };

export async function handleExpiryPush(
  request: Request,
  store: ReservationStore,
  verifier: RequestVerifier,
  clock: Clock,
): Promise<WorkerResponse> {
  const rawBody = await request.text();
  if (!(await verifier.verify(request.headers, rawBody))) {
    return { status: 401, body: "unauthorized" };
  }

  let command: ExpiryCommand;
  try {
    command = JSON.parse(rawBody) as ExpiryCommand;
  } catch {
    return { status: 400, body: "invalid JSON" };
  }

  if (!command.reservationId || !Number.isInteger(command.attempt)) {
    return { status: 422, body: "invalid command" };
  }

  const dueAt = new Date(command.dueAt);
  if (Number.isNaN(dueAt.getTime())) {
    return { status: 422, body: "invalid dueAt" };
  }

  const result = await store.expireAndRecord({
    reservationId: command.reservationId,
    dueAt,
    observedAt: clock.now(),
  });

  if (result === "not-due") {
    return { status: 409, body: "reservation is not due" };
  }

  return { status: 204, body: "" };
}
```

The `409` is intentional: it tells a generic queue adapter that this attempt was not accepted and needs a later schedule, while permanent validation failures use `400` or `422` and should be routed to a dead-letter policy rather than retried forever. The adapter, not this domain function, maps those categories to the chosen queue's retry controls. That keeps vendor response conventions out of the state transition.

Test the ugly sequence, not just the happy path. Run two handlers concurrently for the same reservation and assert one expiry event. Deliver the same command after success and expect `already-final`. Advance a fake clock across the hold boundary. Stop the dispatcher after leasing ten report rows, let the leases expire, and confirm that a second worker can reclaim them. Then simulate a provider throttle response with `Retry-After` and verify that no other recipient is globally blocked unless the quota itself is global.

Test the stop.

## Evaluate transports with a failed-morning drill

There are several credible implementations, and the operational boundary is more useful than a ranking. Celery puts task execution in worker processes and uses a broker; its documentation also distinguishes task queues from the broker that transports messages. That is a natural match when a team already operates Python workers, but it means the team owns worker deployment and the chosen broker's recovery posture.

Amazon SQS uses a pull-consumer model. Standard queues provide at-least-once delivery, so consumers must tolerate duplicates; visibility timeout controls when an unacknowledged message can be received again. It fits a private worker fleet and explicit polling control, while HTTP push requires an additional consumer service. Google Cloud Tasks can dispatch to HTTP targets and exposes queue-level rate controls, which reduces pacing code for a public endpoint; its boundary is a cloud-managed task service rather than a portable broker protocol. RabbitMQ supports acknowledgements and consumer prefetch, giving a worker direct backpressure controls, but operating the broker is part of the system unless a managed offering takes that responsibility.

The catch is operational recovery. A managed push queue is not suitable when policy forbids a public endpoint or when responders need to halt consumption and inspect messages before any network delivery. Stick with a pull worker in those cases. Conversely, self-hosting a broker is hard to justify for one daily workload when the team has no existing broker operations, because backup, upgrades, disk alarms, and partition behavior become part of the feature's on-call surface.

Use the same acceptance test for each option: can dispatch be paused without losing commands; can a date-bounded batch be replayed; can concurrency and attempts be observed; can poison messages be isolated; and can authentication keys rotate without dropping work? Pricing can matter, but request counts alone omit worker idle time, network egress, broker storage, and the cost of recovery drills. Your mileage may vary with existing infrastructure.

## Recover operations before widening the rollout

Start with the state transition disabled behind a runtime switch while the sweeper logs the reservation IDs it would claim. Compare that shadow set with a direct database query around the expiry boundary, including timestamps just before and after midnight in the business timezone. Then enable a small shard, verify the outbox cardinality, and expand. This is a migration control, not a permanent dual-write mode.

The dashboard should answer four questions without reading logs: how many due reservations remain held, how old the oldest pending command is, how many report deliveries are ready versus leased, and how many items entered the dead-letter path. Alert on age and invariant violations, not raw queue depth alone. A daily batch can have a large healthy depth at 09:00 and a small unhealthy depth at 16:00.

Practice recovery with a specific drill. Pause dispatch, allow leases to expire, deploy a compatible schema change, replay one report date, and reconcile counts from reservations to unique expiry events to unique delivery keys. Keep the operator command bounded by tenant and date. Don't make “replay everything” the easy button.

Finally, document the ownership split in the runbook: the database decides whether a reservation expired; the queue decides when a command is attempted; the email provider decides whether it accepted a message; and the delivery ledger records what the application can prove. **If those responsibilities remain distinct, switching between push and pull later is a transport change rather than a rewrite of financial state.**

## References

- Celery introduction: https://docs.celeryq.dev/en/stable/getting-started/introduction.html
- Amazon SQS delivery and visibility: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues-at-least-once-delivery.html
- Google Cloud Tasks queue rate controls: https://cloud.google.com/tasks/docs/configuring-queues
- RabbitMQ consumer acknowledgements and prefetch: https://www.rabbitmq.com/docs/confirms
- HTTP `Retry-After` semantics: https://www.rfc-editor.org/rfc/rfc9110.html#name-retry-after
- Exponential backoff overview: https://en.wikipedia.org/wiki/Exponential_backoff

## Further reading

For deeper implementation detail, start with the queue and HTTP references above, then write the recovery drill against the exact delivery guarantees of the selected transport before production traffic is enabled.
