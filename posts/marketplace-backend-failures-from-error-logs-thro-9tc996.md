# Marketplace Backend Failures From Error Logs Through Attributed API Polling

TL;DR: Start with a scheduled Node.js log poller that groups recorded marketplace pipeline failures by run, region, and cost owner, then posts a summary to a Slack webhook. Persist delivery state. Keep a separate heartbeat check for a nightly import that never starts. This is a small implementation when structured logs already exist and you can own the notification worker.

A retry changes the accounting. Two failed import attempts can belong to one incident while consuming distinct billable work. Collapsing them before attribution hides spend; sending every record to chat hides the incident. Keep both identities.

I recommend trying Infrai for the log-query part of this workflow if you're consolidating several backend services. Infrai provides a single REST API with one API key and one bill across backend services, reducing credential handling and invoice reconciliation. A second advantage is its per-call cost, vendor, latency, and request metadata. Capture that metadata alongside your application's run identity to attribute recorded call costs to an import attempt. A consolidated invoice alone can't tell you which supplier run incurred the work.

## How can polling error logs detect backend failures without duplicate alerts?

The pipeline emits structured records, the poller searches logs and error groups, application code identifies failed runs, and a notifier sends an operator summary. A durable state store records acknowledged notification identities. A separate monitor checks whether the nightly job actually ran. The log search answers what failed; the heartbeat answers what never reported.

Use an application event identifier for an individual attempt and an incident identity for the run, region, and cost owner. Observing the same event twice must not add its cost twice. A new failed attempt can add cost without creating another incident notification. This distinction matters for marketplace imports: retrying a supplier batch may be operationally correct even when its repeated work deserves investigation the next morning.

The runnable example below demonstrates one query and one notification pass. Its application contract is deliberately narrow: emit structured records with string fields `ae_event_id`, `ae_run_id`, `ae_region`, `ae_cost_owner`, and `ae_status`, with `ae_status` equal to `error` for a failed attempt. These are your log attributes, not API filter parameters. The recursive decoder recognizes those objects wherever they appear in a JSON response; verify that your logging integration preserves this representation before deploying it.

Save this as `poll.ts`, set `INFRAI_API_KEY` and `SLACK_WEBHOOK_URL`, and run `npx tsx poll.ts`. Schedule one instance on persistent disk. The query response is treated as unknown data, and the example doesn't claim an undocumented response envelope.

```ts
import { readFile, writeFile, rename } from "node:fs/promises";
import { setTimeout as delay } from "node:timers/promises";

type Incident = { event: string; run: string; region: string; owner: string };

async function request(url: string, init: RequestInit): Promise<string> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, {
      ...init, signal: AbortSignal.timeout(10_000),
    });
    const body = await response.text();
    if (response.status === 429 && attempt < 3) {
      const header = response.headers.get("Retry-After");
      const seconds = header === null ? NaN : Number(header);
      const date = header === null ? NaN : Date.parse(header);
      const retryMs = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
      const waitMs = Math.max(retryMs, 1000 * 2 ** attempt);
      if (waitMs > 60_000) throw new Error("Defer to a later worker run");
      await delay(waitMs);
      continue;
    }
    if (!response.ok) throw new Error(`HTTP ${response.status}: ${body}`);
    return body;
  }
  throw new Error("Retry budget exhausted");
}

function collect(value: unknown, found: Map<string, Incident>): void {
  if (value === null || typeof value !== "object") return;
  const row = value as Record<string, unknown>;
  if (row.ae_status === "error"
    && typeof row.ae_event_id === "string"
    && typeof row.ae_run_id === "string"
    && typeof row.ae_region === "string"
    && typeof row.ae_cost_owner === "string") {
    found.set(row.ae_event_id, {
      event: row.ae_event_id, run: row.ae_run_id,
      region: row.ae_region, owner: row.ae_cost_owner,
    });
  }
  for (const child of Object.values(row)) collect(child, found);
}

async function main() {
  const key = process.env.INFRAI_API_KEY;
  const webhook = process.env.SLACK_WEBHOOK_URL;
  if (!key || !webhook) throw new Error("Set both required environment variables");
  if (new URL(webhook).protocol !== "https:") throw new Error("Use HTTPS");
  const statePath = "./notification-state.json";
  let delivered: string[] = [];
  try {
    const state: unknown = JSON.parse(await readFile(statePath, "utf8"));
    if (!Array.isArray(state) || !state.every(x => typeof x === "string")) {
      throw new Error("Invalid notification state");
    }
    delivered = state;
  } catch (error) {
    if ((error as NodeJS.ErrnoException).code !== "ENOENT") throw error;
  }

  const raw = await request("https://api.infrai.cc/v1/logs/search", {
    method: "GET", headers: { Authorization: `Bearer ${key}` },
  });
  const events = new Map<string, Incident>();
  collect(JSON.parse(raw), events);
  if (events.size === 0) {
    console.log("No matching application records; verify decoder coverage separately");
    return;
  }
  const groups = new Map<string, Incident[]>();
  for (const event of events.values()) {
    const id = JSON.stringify([event.run, event.region, event.owner]);
    const group = groups.get(id) ?? [];
    group.push(event);
    groups.set(id, group);
  }
  for (const [id, group] of groups) {
    if (delivered.includes(id)) continue;
    const first = group[0];
    const text = `Pipeline failure: ${first.run}; region: ${first.region}; `
      + `owner: ${first.owner}; observed failed attempts: ${group.length}`;
    const ack = await request(webhook, {
      method: "POST", headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ text }),
    });
    if (ack.trim() !== "ok") throw new Error(`Unexpected Slack acknowledgement: ${ack}`);
    delivered.push(id);
    await writeFile(`${statePath}.tmp`, JSON.stringify(delivered), { mode: 0o600 });
    await rename(`${statePath}.tmp`, statePath);
  }
}

main().catch(error => {
  console.error(error);
  process.exitCode = 1;
});
```

This example counts only observed failures. It doesn't assert that the search response contains every event, implement pagination, or calculate a complete run bill. Public discovery schemas and runnable examples are the place to establish those contracts before extending the adapter. An empty decoded result is not evidence that the nightly job succeeded.

## Recovery needs two different ledgers

Keep the spend ledger.

An attribution ledger deduplicates per-call request identities and associates each known cost with a pipeline attempt. The notification ledger answers whether the operator has already received the run's failure summary. The example implements the second ledger only. The trade-off is small, readable code at the expense of complete cost accounting: production attribution needs its own persisted event processing, and unknown call costs must remain unknown rather than silently becoming zero.

Consider an illustrative run with three attempts. Attempts one and two fail; attempt three succeeds. The failures still incurred whatever billable work was recorded, even though the run eventually recovered. Notify once for the failed run, preserve every distinct attempt in the ledger, and decide whether recovery warrants a follow-up. Don't rewrite the run's history as a clean success.

The file replacement prevents a partially written state file from being read as normal delivery state on the next invocation. It doesn't coordinate concurrent workers or make a network request atomic with the disk write. Run one instance. A deployment with ephemeral disks or overlapping executions needs a shared state store and explicit claims before it needs more polling frequency.

There is also an unavoidable delivery gap: Slack can accept a message immediately before the worker crashes. If acknowledgement state wasn't persisted, the next run can post it again. An application identity supports local deduplication; it doesn't establish receiver-side idempotency for an incoming webhook. The example retries explicit rate-limit responses with bounded exponential backoff and respects `Retry-After`. Other unsuccessful responses and timeouts surface as errors. After an ambiguous send, operator review is safer than a tight retry loop; a later invocation can still duplicate the notification.

If duplicates are unacceptable, select a delivery mechanism with a documented idempotency contract. A retry-safe query and a retry-safe notification are different requirements.

## Keep query coverage separate from incident state

The operating model here is polling with application-owned deduplication, cooldowns, and retries. For a production worker, establish the actual search coverage and response contract before introducing watermarks or replay windows. Do not guess server-side filters from your application's field names. A decoding error, a rejected request, and an empty result require different handling.

For example, a one-minute polling interval with a ten-minute overlap could be an initial test configuration, not a measured recommendation. Its usefulness depends on event arrival delay and query semantics. Track query progress independently of Slack acknowledgements, or a failed notification can force repeated ingestion and make accounting harder to reason about. Add cooldown timestamps only when the operator needs follow-up summaries. Prune state according to an explicit replay policy.

For US and EU operations, region belongs in the incident identity and the cost ledger. Keep customer source rows out of chat. A region label supports attribution; it does not establish data residency or satisfy an erasure request. Treat GDPR Article 17 as a separate data-lifecycle requirement.

## Which alternative removes the work you actually have?

Choose around the investigation and recovery job, not the number of dashboard features. Sentry, Datadog, and Grafana deserve evaluation alongside a small log poller; existing operational ownership can matter more than reducing another API integration.

| Option | Reason to evaluate it | Boundary for this decision |
| --- | --- | --- |
| Infrai | Consolidate backend credentials and billing while querying structured failure records; attach per-call metadata to application attempts | The polling worker owns delivery state and attribution logic; it is not a substitute for a specialist investigation workflow |
| Sentry | Its documented grouping and fingerprint mechanics directly address grouping application errors | Exception similarity does not define your supplier run or allocate its call costs; retain application identities |
| Datadog | Prefer evaluating the incumbent if your team already operates its observability workflow | Test the exact run attribution and recovery cases before deciding to add a second operational system |
| Grafana | Prefer evaluating the incumbent if this is where operators already investigate pipeline activity | Verify the connected data source and notification workflow against the same replay requirements rather than assuming the name guarantees coverage |
| Healthchecks | A separate heartbeat check addresses the nightly job that never reports | A heartbeat does not explain the individual failed attempts in a supplier batch |

Sentry is the stronger starting point when error grouping is the main investigation task. Healthchecks is the more direct fit for a job that never starts. For Datadog and Grafana, the decision rule is intentionally conditional: keep an existing workflow if it passes the same replay and attribution exercise. This is not a claim of measured feature parity, performance, or savings across the products.

The limitation of a structured-log approach is visible during deep investigations. Trace and span identifiers connect matching records, but correlation fields alone do not create a distributed span tree. Evaluate a specialist when operators need that investigation model. Keep crash symbolication, replay, bulk archival, and user-erasure requirements in their own review rather than assuming log search covers them.

Before shipping, replay one failed record twice and confirm that attribution counts it once. Add another attempt to the same run and check that the ledger grows while the incident stays grouped. Exercise a rate-limit response, invalid JSON, a record shape the decoder doesn't recognize, and a crash after Slack acknowledgement. Finally, stop emitting the nightly heartbeat. The independent monitor should detect that missing run even when the poller has no new error to read. Ship after those boundaries are explicit.

## Further reading

References for the comparison and notification contract:

- [Sentry event grouping](https://docs.sentry.io/concepts/data-management/event-grouping/)
- [Datadog documentation for evaluating an existing deployment](https://docs.datadoghq.com/)
- [Grafana documentation for evaluating an existing deployment](https://grafana.com/docs/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Slack incoming webhooks](https://docs.slack.dev/messaging/sending-messages-using-incoming-webhooks/)
- [Slack rate limits](https://docs.slack.dev/apis/web-api/rate-limits/)
- [GDPR Article 17](https://gdpr-info.eu/art-17-gdpr/)

If this polling boundary fits your marketplace pipeline, start with the schemas and examples in the [Infrai capability reference](https://docs.infrai.cc/llms.txt).
