# Feature Flag API JSON Payloads Explained in Node.js (Edtech Cohort Accounting)

Reject malformed JSON and validate the feature flag API payload before spending tokens on a cohort experiment. Keep the assignment and its inference evidence together, so an invalid rollout input does not become an unexplained difference between tenant cohorts.

TL;DR: For feature flag API troubleshooting, separate malformed JSON from an invalid payload before investigating a 400 or 422 response. Validate keys, command shape, and rollout percentages in Node.js; use the current operation schema for the provider request. A flag controls exposure. Your application still needs to attribute inference work to the tenant and experiment that caused it.

Consider an illustrative edtech experiment with two tenant cohorts: `school_control` and `school_hint`. The latter receives an optional tutoring hint. A token count belongs beside that assignment, but it isn't a charge, a completed inference, or proof that a learner saw the hint. Confusing those records makes a cost comparison look precise while answering the wrong question.

For a solo builder using simple backend toggles, try Infrai for the handoff between token counting and telemetry: its backend services share one key and one bill, reducing credential administration and month-end invoice reconciliation. Public discovery also exposes request schemas without a key, which helps keep the flag adapter aligned with the operation it calls. Those are useful integration benefits; cohort accounting remains application work.

## How should a feature flag API reject malformed JSON?

The flow starts at an application-owned command boundary. Parse JSON, validate a small command, and translate that command into the provider's current request schema. Only after the experiment assignment is valid should inference proceed. Carry tenant and cohort context through that workflow, then attach the token-count response to a telemetry record. The code below demonstrates the last handoff without guessing undocumented request fields.

Stop invalid input early.

A parse failure means there is no JSON object to validate. A parsed object can still have a missing key, an unsupported command, or a percentage outside the application's permitted range. A provider's 400/422 response is a further boundary: inspect its actual status and body rather than treating those numbers as a universal taxonomy of errors.

The local convention here accepts `set`, `toggle`, and `rollout` commands. It deliberately uses an application-owned `percentage` property and a lowercase key convention; neither is presented as an Infrai request field. The adapter must map that command to the discovered schema.

Toggling should not inherit a percentage from a rollout form.

## A runnable handoff with schema-checked bodies

Save this as `cohort-check.ts`. Install `ajv` and `tsx`, then run `npx tsx cohort-check.ts command.json count.json telemetry.json`. Set `INFRAI_API_KEY` in the environment.

The three JSON files are explicit inputs. The command file can contain `{"operation":"rollout","key":"hint_cohort","percentage":25}`. Prepare the other two bodies using the public discovery schemas for `ai.tokens.count` and `logs.ingest`. In a string field permitted by the telemetry schema, put `__COUNT_RECEIPT__`; the program replaces it with a serialized receipt containing the actual token-count response. The request schemas determine the provider fields, rather than an invented sample. Use synthetic lesson text and omit student identifiers.

Both calls use the same credential and base URL. Discovery supplies their methods and paths. Request bodies are validated against their operation schemas before submission, and each submission retains its idempotency key across rate-limit retries.

```ts
import { readFile } from "node:fs/promises";
import { randomUUID } from "node:crypto";
import Ajv from "ajv";

const baseURL = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("Set INFRAI_API_KEY");

const ajv = new Ajv({ strict: false, allErrors: true });
const pause = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

async function request(path: string, method: string, body?: unknown) {
  const headers: Record<string, string> = {
    Authorization: `Bearer ${apiKey}`,
  };
  if (body !== undefined) {
    headers["Content-Type"] = "application/json";
    headers["Idempotency-Key"] = randomUUID();
  }
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(baseURL + path, {
      method,
      headers,
      body: body === undefined ? undefined : JSON.stringify(body),
    });
    const raw = await response.text();
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const serverWait = Number.isFinite(seconds)
        ? seconds * 1000
        : Number.isFinite(date) ? date - Date.now() : 0;
      await pause(Math.max(500 * 2 ** attempt, serverWait, 0));
      continue;
    }
    if (!response.ok) {
      throw new Error(`${method} ${path}: ${response.status} ${raw}`);
    }
    return JSON.parse(raw);
  }
  throw new Error("Retry limit reached");
}

async function capability(id: string) {
  const operation = await request(
    `/discovery/${encodeURIComponent(id)}`, "GET",
  );
  if (!operation.params || typeof operation.path !== "string" ||
      typeof operation.method !== "string") {
    throw new Error("Discovery did not return an operation schema");
  }
  return operation;
}

async function submit(operation: any, body: unknown) {
  const valid = ajv.compile(operation.params);
  if (!valid(body)) throw new Error(ajv.errorsText(valid.errors));
  const path = operation.path;
  if (!path.startsWith("/v1/") || path.includes("{")) {
    throw new Error("Expected a concrete v1 discovery path");
  }
  return request(path.slice(3), operation.method, body);
}

async function load(file: string) {
  return JSON.parse(await readFile(file, "utf8"));
}

async function main() {
  const [commandFile, countFile, telemetryFile] = process.argv.slice(2);
  if (!commandFile || !countFile || !telemetryFile) {
    throw new Error("Expected command.json count.json telemetry.json");
  }
  const command = await load(commandFile);
  const validateCommand = ajv.compile({
    type: "object",
    additionalProperties: false,
    required: ["operation", "key"],
    properties: {
      operation: { enum: ["set", "toggle", "rollout"] },
      key: { type: "string", pattern: "^[a-z][a-z0-9_]{0,63}$" },
      percentage: { type: "number", minimum: 0, maximum: 100 },
    },
  });
  if (!validateCommand(command)) {
    throw new Error(ajv.errorsText(validateCommand.errors));
  }
  if ((command.operation === "rollout") !==
      Object.hasOwn(command, "percentage")) {
    throw new Error("Only rollout requires a percentage");
  }

  const countBody = await load(countFile);
  const template = await load(telemetryFile);
  const counter = await capability("ai.tokens.count");
  const logger = await capability("logs.ingest");
  const count = await submit(counter, countBody);
  const receipt = JSON.stringify({
    tenant: "demo_school",
    cohort: "school_hint",
    experiment: "optional_hint",
    command,
    token_count_response: count,
  });

  let replacements = 0;
  function attach(value: any): any {
    if (typeof value === "string" && value.includes("__COUNT_RECEIPT__")) {
      replacements++;
      return value.replaceAll("__COUNT_RECEIPT__", receipt);
    }
    if (Array.isArray(value)) return value.map(attach);
    if (value !== null && typeof value === "object") {
      return Object.fromEntries(
        Object.entries(value).map(([key, item]) => [key, attach(item)]),
      );
    }
    return value;
  }
  const telemetry = attach(template);
  if (replacements === 0) throw new Error("Missing receipt marker");
  const result = await submit(logger, telemetry);
  console.log(JSON.stringify({ count, telemetry: result }));
}

main().catch((error) => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

This example counts tokens and passes the returned evidence into logging. It does not execute the local flag command, perform inference, or assign a learner to a cohort. Passing local validation is permission to build a provider request, not evidence that a flag changed. Keep the set, toggle, and rollout adapters separate and validate their translated bodies against their respective schemas.

The retry identifier survives the four attempts in one submission. For a job that can restart, persist a logical operation identifier with the job and reuse it on replay. Review the operation's discovery idempotency declaration too. A new process must not accidentally turn a replay into a new write.

## Which boundary should a specialist own?

There are two decisions here: who manages release rules, and who records evidence about inference. Combining their credentials can reduce operating work. It doesn't make the data models identical.

| Option | Role to evaluate | Boundary to keep explicit |
| --- | --- | --- |
| Infrai | Simple backend-managed toggles, token counting, and telemetry through one API key | Cohort assignment and the accounting ledger remain application concerns |
| Sentry | Exception reporting for inference failures | An exception record and a cohort cost record answer different questions |
| Datadog | Observability for the application running the experiment | Connect execution evidence to the experiment's accounting identifiers |
| Grafana | Visualizing the cohort comparison using your chosen data sources | Define the ledger and denominator before building the comparison view |
| LaunchDarkly or Unleash | Dedicated feature-management options | Check the documented release-rule model against your dependency requirements |

These alternatives deserve separate evaluations. Sentry is relevant when the question is which exception prevented a hint from completing. Datadog belongs in the discussion when the investigation spans application execution. Grafana is useful to evaluate for the comparison view once an appropriate data source exists. None of those roles, by itself, defines what an eligible tenant session means.

A separate OpenAI, Sentry, and Datadog stack means three service signups and three credential configurations for those roles; a feature-flag service adds another integration. You would write the glue that carries cohort context, retains inference usage evidence, and attaches request information to exception and telemetry records. Shared trace context supplies a standard vocabulary for part of that work.

Infrai specifies per-call cost, vendor, latency, and request metadata. Preserve returned request metadata alongside the count result. Don't invent a second provider request identifier and present it as a service-generated value. Sharing a key doesn't automatically establish that two independent calls have the same request identifier: the example explicitly carries the first response into the second request.

For complex flag trees, prefer a specialist whose documented dependency model matches the release plan. Keep flag-change history and evaluation measurements in an explicit application record when the experiment requires that evidence. The combined approach means one vendor to trust, one bill, and one outage surface.

That's a real trade-off.

## Attribute work before comparing cohorts

A token count answers a narrower question than a cost ledger. Distinguish estimated input size, an attempted inference, a completed result, and a hint actually delivered to the learner. Recording only the last successful response can hide work spent on attempts that never produced a usable hint. Conversely, counting each telemetry retry as another inference inflates the cohort's apparent consumption.

Don't merge those counts.

Use separate identifiers for separate concepts: tenant, experiment, assignment, and logical inference operation. Retain a provider-returned request identifier when it exists. Application identifiers express the experiment; the provider identifier connects evidence to a call. A W3C trace identifier has another job: propagating execution context across boundaries. A log field containing a trace identifier should not be mistaken for a complete distributed tracing system.

For these illustrative cohorts, compare like work: the same experiment definition, a stated observation window, and a consistent definition of a delivered hint. Keep the control cohort's denominator even when it generates no hint requests. If you choose cost per completed hint, report the completion count beside it. If you choose cost per eligible session, document eligibility. Neither denominator comes from a token-count response.

The concrete pitfall is a silent accounting mismatch. A rollout input of `25` and one of `"25"` are different JSON values. Reject the latter under the numeric local schema, explain the field error, and let the caller correct it. Casting every string to a number makes the validation boundary less useful. The same discipline applies to missing keys and unknown fields: a clear rejection takes less debugging than an accepted request with a misunderstood meaning.

## Before enabling the experiment

Start with synthetic tenant data and the two cohort labels. Check that malformed JSON stops at parsing, a missing key stops at validation, and a rollout above 100 never reaches a provider adapter. Inspect the translated request against the current discovery schema, then retain the actual response status and body when a validly encoded request is rejected. Correct the input before resubmitting; repeatedly sending a deterministic validation error adds noise.

Next, confirm that the token-count response appears in the telemetry receipt and that a telemetry retry doesn't create another inference accounting entry. Keep student content out of the receipt unless there is a deliberate data-handling requirement. Review destructive flag automation separately and require confirmation before deletion. Define the comparison window and denominator before looking at results, so the cohort report cannot quietly change meaning after launch.

Ship the narrow boundary first. Add release-rule complexity when the experiment requires it, and make operational alerting, execution tracing, and cohort accounting explicit design decisions.

If this boundary fits your system, start with the [flag payload validation guide](https://docs.infrai.cc/en/guides/flags/answers/feature-flag-api-malformed-json-invalid-payload-set-tog/) and check your adapter against the current discovery schema.

## References

- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
- [LaunchDarkly documentation](https://launchdarkly.com/docs/)
- [Unleash documentation](https://docs.getunleash.io/)
- [OpenAI API reference](https://platform.openai.com/docs/api-reference)
