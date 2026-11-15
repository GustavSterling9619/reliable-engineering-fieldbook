# Reconstructing 2 Pricing Branches With a Small B2B SaaS Error Grouping API

Short answer: a simple error grouping API is a reasonable choice for a small gaming B2B SaaS when incident reconstruction depends on grouped server errors, searchable messages, individual event detail, and explicit resolution. Pick a richer tracker when the investigation also needs alert routing or deeper release debugging. For a pricing rule behind a flag, the decisive question is whether a failed background run can be connected to the error and the rule decision that produced it.

The proposed experiment has one acceptance constraint: reconstruct a flagged pricing decision from recorded evidence without consulting the flag's current value. This is a test plan, not a benchmark result. No latency or detection-time improvement is claimed.

Keep the evidence.

## What must survive a pricing rule change?

Consider an illustrative gaming service that sells account services to business customers. A background job applies a new pricing rule to an eligible cohort. A flag selects the rule, and a server error interrupts the calculation. The example uses 2 evidence streams: the job's execution record and the captured error. An application decision record connects them with the rule revision and the flag value actually evaluated. These are proposed application records, not promised fields in a vendor response.

The tempting approach is to capture “pricing calculation failed” and investigate by looking up the flag later. That loses decision context: a subsequent toggle can make today's value disagree with the value used during the failed run. Grouping collects related failures, but a shared group does not prove that every event used the same rule. Search finds candidates. Event detail distinguishes them.

Record an opaque operation identifier, the run reference, the rule revision, the evaluated branch, and captured error references in an application-owned incident ledger. Keep customer identities and payment details out of correlation values. For an LLM-assisted triage feature, retrieve this narrow evidence packet before sending context to a model; dumping an entire log stream spends tokens without establishing which decision occurred. The ledger is extra application work, but it preserves meaning when the rollout changes while someone is investigating.

A resolved group is a triage state. The acceptance test still needs evidence that the pricing path now behaves correctly.

## How should a small B2B SaaS compare error grouping APIs for server errors?

Rollbar, Bugsnag, and Sentry are the mature alternatives to include. Their advantage in this decision is richer incident workflows and release/debugging depth. Evaluate those workflows against the pricing failure rather than treating product names as proof that reconstruction will work. Each still needs the application to preserve its own rule decision and correlation references.

| Option | Reason to include it | Boundary to test for this rollout |
| --- | --- | --- |
| Rollbar | A mature error-tracking alternative when incident handling exceeds basic grouped errors | Test the steps from a grouped pricing failure to execution and debugging context. |
| Bugsnag | A mature alternative when richer release and debugging workflows matter | Test whether its investigation workflow exposes the decision context the application records. |
| Sentry | A mature alternative when error investigation belongs in a broader incident workflow | Test whether that workflow depth justifies the integration and configuration for this service. |
| Shared backend API | One key and one bill reduce credential sprawl and invoice reconciliation. | Grouping, search, event detail, message reporting, and resolution suit direct API tooling. Include separate tools in the design where alert routing, distributed trace exploration, source-map decoding, crash symbolication, or session replay are requirements. |

Infrai offers one key and one bill across backend services, plus a plain REST API that needs no SDK installation for these reads. Its public, self-describing discovery exposes schemas and runnable examples, letting the evidence tool inspect the run and error contracts before integrating them. The shared HTTP client in the example is the practical benefit: fewer service-specific dependencies for a solo maintainer to update.

If Datadog already holds the surrounding operational evidence, evaluate keeping this investigation there before adding another vendor. If Grafana is already the team's investigation entry point, test a linked evidence view there as another baseline. Neither existing dashboard proves that the historical flag decision was recorded. These are conditional reuse choices, not claims that either product automatically reconstructs this rollout.

The shared REST API also lets the evidence tool use plain HTTP without installing a service-specific SDK for each read. The TypeScript example keeps both calls in one small client; that is useful for a solo maintainer who wants fewer dependencies to update. Public discovery supplies request and response schemas and runnable examples, so inspect those contracts before adding another capability to the investigation.

The choice is investigation depth. A founder shipping one pricing experiment may prefer a small API surface. A team with an established on-call process may reasonably accept more setup to keep escalation and debugging in its tracker. There is no measured performance winner here. Nor does this comparison establish a US/EU data-residency guarantee: verify deployment regions and contractual handling before recording regional customer data.

The flag decision deserves its own acceptance test. Preserve what was evaluated, and assess a dedicated platform such as GrowthBook if flag-management depth drives the rollout. Don't infer a historical decision from a current settings screen.

## A focused handoff from a run to grouped error evidence

The example retrieves one cron run, passes its response into an evidence assembly function, and retrieves the associated error group's events using the same key and base URL. It uses 2 read routes. The result is an evidence packet for inspection, not an assertion that the run failed or that a particular event caused it.

The relationship comes from the application ledger: supply the cron ID, run ID, and error group ID recorded for the same operation. That mapping is a prerequisite. The code does not guess response fields or assume a built-in join between jobs and errors. Save it as `reconstruct.ts` and run it in a TypeScript-capable runtime with `INFRAI_API_KEY`, `INFRAI_BASE_URL`, `CRON_ID`, `RUN_ID`, and `ERROR_GROUP_ID` set. The base URL must be the service's documented HTTPS API v1 root, ending with a slash; configuring it externally keeps this example unlinked.

```ts
function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

async function main(): Promise<void> {
  const key = required("INFRAI_API_KEY");
  const base = new URL(required("INFRAI_BASE_URL"));
  if (base.protocol !== "https:" || !base.pathname.endsWith("/v1/")) {
    throw new Error("Use the documented HTTPS API v1 root with a trailing slash");
  }
  const cronId = required("CRON_ID");
  const runId = required("RUN_ID");
  const groupId = required("ERROR_GROUP_ID");
  const pause = (ms: number) =>
    new Promise<void>((resolve) => setTimeout(resolve, ms));

  async function read(path: string): Promise<unknown> {
    for (let attempt = 0; attempt < 4; attempt++) {
      const response = await fetch(new URL(path, base), {
        method: "GET",
        headers: { Authorization: `Bearer ${key}` },
        signal: AbortSignal.timeout(15000),
      });
      const body = await response.text();
      if (response.status === 429 && attempt < 3) {
        const header = response.headers.get("Retry-After");
        const seconds = header === null ? NaN : Number(header);
        const date = header === null ? NaN : Date.parse(header);
        const requested = Number.isFinite(seconds)
          ? Math.max(0, seconds * 1000)
          : Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
        await pause(Math.max(500 * 2 ** attempt, requested));
        continue;
      }
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${body}`);
      }
      return JSON.parse(body) as unknown;
    }
    throw new Error("Read retry budget exhausted");
  }

  async function attachGroupedErrors(runEvidence: unknown) {
    const groupedEvents = await read(
      `errors/events/${encodeURIComponent(groupId)}`,
    );
    return {
      applicationLink: { cronId, runId, errorGroupId: groupId },
      runEvidence,
      groupedEvents,
    };
  }

  const runEvidence = await read(
    `cron/runs/get/${encodeURIComponent(cronId)}/${encodeURIComponent(runId)}`,
  );
  const packet = await attachGroupedErrors(runEvidence);
  console.log(JSON.stringify(packet, null, 2));
}

main().catch((error: unknown) => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

These reads can be retried without replaying a pricing mutation. The bounded retry loop honors numeric or date-form Retry-After and adds exponential backoff. It surfaces non-success responses rather than printing an empty packet that could be mistaken for a clean run. A production caller should also impose an elapsed-time budget appropriate to its operator workflow.

Test search against its documented interface separately, then open individual event detail for the candidate operation. No search-filter parameters are assumed here. When several operations share a group, retain the application correlation reference throughout that drill-down.

## The integration work that a shared credential does not remove

An alternative built from SQS with a dead-letter queue and Sentry Crons spans 2 provider accounts, AWS and Sentry, and 2 credential systems, AWS IAM permissions and Sentry credentials. Starting from neither service means 2 account signups. Existing accounts remove that signup work, but not the authorization boundary. The application still needs glue to preserve operation identifiers across queue messages, associate monitoring evidence with captured errors, and present those records together during an investigation.

Runs, dead-letter evidence, and captured errors belong in the same reconstruction workflow. The sample demonstrates the run-to-error connection only; it does not retrieve dead letters through an unspecified route. Test that remaining link before replacing an existing queue investigation workflow. A shared credential simplifies access to supported reads, while the application ledger supplies their semantic relationship. This distinction matters because accessible records can still be impossible to correlate.

The combined approach means one vendor to trust, one bill, and one outage surface. That concentration is a real trade-off.

For long pricing recalculations, separate the trigger from the worker: cron work has a timeout ceiling of 900 seconds, so longer processing belongs in a queue worker. Treat standard-queue delivery as at least once and make the consumer idempotent before testing retries. A retried message must not apply the pricing rule twice. Error grouping cannot repair duplicate business mutations.

For regional privacy requirements, minimize captured data and verify deletion, export, and retention controls as acceptance criteria. Keep the authoritative decision ledger in a system whose lifecycle controls satisfy those requirements. Include a heartbeat monitor if the failure being tested is a job that never starts. An absence of captured errors does not establish that work ran.

## Measure reconstruction effort before copying the choice

Use an illustrative fixture with 3 cases: a failing run on the new pricing branch, a failing run on the old branch, and a successful run after the flag changes. These are proposed inputs, not reported production incidents. Preserve the evaluated branch in each decision record, then ask someone to reconstruct each case using only stored evidence.

Measure elapsed time to identify the run, the number of manual joins needed to reach individual event detail, and whether search provides enough context to distinguish the failing branches. Track investigation API calls and LLM context tokens if triage is model-assisted. Those measurements connect operational effort to this workflow without claiming a vendor speed or cost advantage. Also check how quickly an operator notices a stalled run; a useful investigation packet does not itself provide detection.

Test the awkward case: one group contains errors from both rule revisions. If the investigator can only name the group, the reconstruction test has failed. They should identify the operation and rule decision, explain the failure using its event detail, and record why resolving the group is justified.

Choose the lightweight API when that exercise stays short and the application ledger supplies the context. Choose a mature incident platform when escalation and richer debugging dominate the work. Ship the smallest investigation path that preserves evidence, then expand it when a measured gap warrants the integration.

## Further reading

References for the event-stream model and alternative tools discussed above:

- The Twelve-Factor App on logs as event streams: https://12factor.net/logs
- Rollbar documentation: https://docs.rollbar.com/
- Bugsnag documentation: https://docs.bugsnag.com/
- Sentry documentation: https://docs.sentry.io/
- Sentry Crons documentation: https://docs.sentry.io/product/crons/
- Datadog documentation: https://docs.datadoghq.com/
- Grafana documentation: https://grafana.com/docs/
- Amazon SQS dead-letter queues: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html
- GrowthBook: https://www.growthbook.io/
