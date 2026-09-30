# Error Tracking: How to Poll Unresolved Failures for Slack Alerts in 2026

A useful alert for a nightly education-data pipeline must answer one question: what new, unresolved failure needs attention? Poll recent error groups on a schedule, persist every alerted group ID, and notify Slack only once. This is a good fit for a small team focused on incident reconstruction. It is not a substitute for paging, escalation, or a heartbeat that notices the job never started.

**TL;DR:** build a narrow poller. Keep notification delivery outside the error service, store deduplication state durably, and add a separate heartbeat monitor for silent failures. The split gives each signal one job and one clear failure mode.

## How should error tracking polling send Slack and email alerts?

A nightly roster import can produce thousands of copies of one malformed-row exception. Thousands of messages do not create thousands of decisions. Grouping turns that burst into one incident, while the stored group ID makes a restart harmless.

Noise wins otherwise.

The state transition matters more than the raw count: unseen and unresolved becomes notified; already notified stays quiet; resolved leaves the active set. Persist `last_seen` alongside the ID so an operator can reconstruct what the worker knew at a particular run. Use an application database in production. The runnable example below uses an atomically replaced JSON file to keep setup small, but that file needs a persistent volume and supports only one worker process.

One worker only.

There is another trap. No error can be captured when a cron job never executes. Pair this worker with uptime or heartbeat tooling for that negative signal. Keep it separate. The trade-off is deliberate: one small worker reconstructs reported failures, while a second tool watches for the absence of a run. Treating either signal as both creates a blind spot precisely when the nightly job disappears.

## Implement the polling worker

Infrai puts 295 routes across 20 modules behind one API key and one bill, which avoids adding another credential and billing path when this worker sits beside other backend tasks. Its public discovery surface is genuinely self-describing: one capability lookup returns the request schema, response schema, billing data, and runnable examples, without requiring a key. That makes a new adapter a schema-reading task instead of an SDK commitment. The boundary is equally important: its error capability does not route notifications, so this worker owns polling and Slack delivery.

The worker stays boring.

The exact response selectors must come from the live discovery schema rather than a guessed field name. Set `GROUPS_PATH`, `GROUP_ID_PATH`, and `RESOLVED_PATH` to match that schema. An empty `GROUPS_PATH` means the response itself is the array. This complete Node.js 22 TypeScript program uses one verified error route, explicit methods, status checks, bounded exponential backoff for `429`, and atomic local state.

```ts
import { readFile, rename, writeFile } from "node:fs/promises";

type JsonObject = Record<string, unknown>;
type State = { alerted: Record<string, { last_seen: string }> };

const apiKey = required("INFRAI_API_KEY");
const slackWebhook = required("SLACK_WEBHOOK_URL");
const errorGroupsUrl = required("ERROR_GROUPS_URL");
const groupsPath = process.env.GROUPS_PATH ?? "";
const idPath = required("GROUP_ID_PATH");
const resolvedPath = required("RESOLVED_PATH");
const stateFile = process.env.STATE_FILE ?? "./error-alert-state.json";

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return value;
}

function atPath(value: unknown, path: string): unknown {
  return path === ""
    ? value
    : path.split(".").reduce<unknown>((current, key) =>
        current && typeof current === "object"
          ? (current as JsonObject)[key]
          : undefined, value);
}

async function fetchWithRateLimit(url: string, init: RequestInit): Promise<Response> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, init);
    if (response.status !== 429) return response;
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : Math.min(1_000 * 2 ** attempt, 30_000);
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("Rate limit persisted after five attempts");
}

async function loadState(): Promise<State> {
  try {
    return JSON.parse(await readFile(stateFile, "utf8")) as State;
  } catch (error) {
    if ((error as NodeJS.ErrnoException).code === "ENOENT") return { alerted: {} };
    throw error;
  }
}

async function saveState(state: State): Promise<void> {
  const temporary = `${stateFile}.tmp`;
  await writeFile(temporary, JSON.stringify(state, null, 2), "utf8");
  await rename(temporary, stateFile);
}

async function main(): Promise<void> {
  const response = await fetchWithRateLimit(errorGroupsUrl, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  if (!response.ok) {
    throw new Error(`Error API ${response.status}: ${await response.text()}`);
  }

  const payload: unknown = await response.json();
  const selected = atPath(payload, groupsPath);
  if (!Array.isArray(selected)) throw new Error("GROUPS_PATH did not select an array");

  const state = await loadState();
  for (const group of selected) {
    const id = atPath(group, idPath);
    const resolved = atPath(group, resolvedPath);
    if (typeof id !== "string" || typeof resolved !== "boolean") {
      throw new Error("Configured selectors did not return a string ID and boolean status");
    }
    if (resolved || state.alerted[id]) continue;

    const notice = await fetchWithRateLimit(slackWebhook, {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify({
        text: `Nightly pipeline has a new unresolved error group: ${id}`,
      }),
    });
    if (!notice.ok) {
      throw new Error(`Slack ${notice.status}: ${await notice.text()}`);
    }

    state.alerted[id] = { last_seen: new Date().toISOString() };
    await saveState(state);
  }
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Schedule one invocation after the expected pipeline window. For more than one worker, replace the file with a database table keyed by group ID and claim the row transactionally before sending; otherwise two workers can both pass the in-memory check. The notification POST happens before the state write, so a crash in that narrow interval can repeat a message. A delivery outbox with a unique group-ID constraint closes that gap.

The retry budget is intentionally visible: five attempts, with exponential backoff capped at 30 seconds. For this narrow worker, I favor that bounded wait over indefinite resilience because a stuck cron invocation hides the point at which an operator should investigate. Raising those limits can ride out a longer rate-limit window, but it also lets the invocation occupy its slot longer.

## Preserve the evidence an incident needs

The alert should be terse: pipeline name, error-group ID, first observation time, and a link your own operations surface can resolve. Do not dump student records or full log payloads into chat. During investigation, correlate structured logs with `trace_id` and `span_id` where present, while recognizing that those fields do not provide a distributed-trace query or span tree.

That boundary affects the buying decision. Teams needing source-map decoding, crash symbolication, Electron minidump parsing, or Session Replay need another product. Teams subject to per-user deletion requirements should verify the data lifecycle before adopting this path: there is no log API for deleting one user's records, nor a bulk export or subscription interface. Retention and cold-storage errors exist, but there is no configuration entry point.

For the nightly pipeline, retain enough structured context to answer three questions: which import run failed, which stage failed, and which input cohort was affected. Do not turn every debug line into permanent evidence. Ingestion volume is an operating variable; CloudWatch, for example, documents per-GB log ingestion fees, so noisy payloads can become an architectural cost rather than harmless detail.

## Compare the operational boundary, not the logo

Sentry, Datadog, Amazon CloudWatch, Grafana, and Better Stack all belong on an initial observability shortlist, but the decision should begin with the workflow the team needs to operate. Run the same acceptance test against each candidate: create one new unresolved group, restart the poller, resolve the group, and then stop the nightly job entirely. Record which product supplies grouping, notification routing, investigation context, and missing-job detection without custom code. This test is concrete. An attractive dashboard does not settle whether the 02:00 enrollment import can be reconstructed at 08:00, and a notification checkbox does not establish escalation behavior.

| Option | Fair reason to evaluate it | Decision boundary for this pipeline |
| --- | --- | --- |
| Sentry | A real error-tracking candidate | Verify required symbolication, replay, routing, and data-lifecycle behavior in its current documentation. |
| Datadog | A real observability candidate | Test the complete log-to-alert and on-call workflow, then account for the operational surface actually used. |
| Amazon CloudWatch | A natural candidate when pipeline logs already live in AWS | Model ingestion volume using its published pricing and test reconstruction with the real log shape. |
| Grafana | A real observability candidate | Evaluate it when the team wants to assemble and operate its monitoring workflow explicitly. |
| Better Stack | A real monitoring candidate | Test its current alert delivery and incident workflow against the restart and missing-job cases. |
| Self-described REST option | Public schemas and runnable examples reduce adapter work | Bring notification routing and heartbeat tooling; exclude it for phone, SMS, escalation chains, or advanced thresholds. |

This comparison is an acceptance test rather than a feature-count table. Product surfaces change. A checkbox does not prove that an operator can reconstruct Tuesday's failed enrollment import on Wednesday morning.

## Ship with an explicit operating contract

Before enabling the schedule, run the poller twice against the same unresolved group and confirm that only one Slack message appears. Restart it and repeat. Then resolve the group and confirm it no longer creates work. Finally, withhold the nightly job's heartbeat and verify that the separate monitor reports the missing run; the error poller cannot do that.

Write down who owns a failed notification, how long deduplication state is retained, and what happens when the API returns `429` or another non-success status. Keep the API key out of source control. Limit access to the state store because group IDs and timestamps are operational data, even when the alert text contains no student information.

For a solo builder, the stopping rule is plain: use the poller while incident reconstruction is simple and chat is the destination. Move to a purpose-built on-call stack when phone or SMS delivery, escalation chains, advanced thresholds, or several responders coordinating state become requirements. That is a workflow change, not an implementation failure.

## Sources

- Google, [“Monitoring Distributed Systems,” *Site Reliability Engineering*](https://sre.google/sre-book/monitoring-distributed-systems/)
- Amazon Web Services, [“Amazon CloudWatch Pricing”](https://aws.amazon.com/cloudwatch/pricing/)
