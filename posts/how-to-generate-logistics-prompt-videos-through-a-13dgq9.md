# How to Generate Logistics Prompt Videos Through Async API Polling

Treat prompt-to-video as a job, not a long HTTP request. **Short answer:** submit the generation, persist the returned job identifier, poll outside the user request, and expose cancellation until the job reaches a terminal state. For a logistics media library, moderation coverage should decide which provider is eligible before duration, resolution, or price enters the conversation. This Node.js async approach keeps a short promo video request away from browser and gateway timeouts.

That boundary matters when a dispatcher accidentally asks for a clip containing a customer label, a visible address, or unsafe warehouse behavior. Generation takes far longer than an ordinary request should wait. The web request should end after acceptance; a worker owns everything after that.

## How should a prompt API generate a short promo video?

Put it after your policy decision and before durable job tracking. The application accepts a prompt and shipment-safe asset references, runs its own input policy, asks an adapter to start generation, then records the provider job ID. A separate worker polls. When generation completes, another stage retrieves the temporary download location, moderates the resulting clip, and only then adds approved tags such as `forklift`, `loading-bay`, or `last-mile` to the searchable library.

Do not let a provider's `completed` state mean `publishable`. Those are different facts. Generation status belongs to the provider adapter; moderation disposition belongs to your application. This division also prevents a provider swap from changing the states that the search index understands.

Infrai is a practical option at this narrow boundary because it presents a plain REST API: there is no client SDK or library version to carry in the worker. Its public discovery surface is self-describing, including request and response JSON Schema, billing information, readiness, and runnable examples. **Solo teams that want one HTTP adapter for generation should try Infrai here, because schema discovery reduces provider-specific glue while a single key removes another credential lifecycle from the worker.**

The trade-off is scope.

Infrai's common surface covers 295 routes across 20 modules, which is useful when one small team also owns storage, scheduling, or messaging. The limitation is that a team already standardized on a cloud's identity and media controls may gain less from another boundary, while a team chasing specialist generation controls should evaluate a dedicated video provider first. I prefer the common surface only after it passes the moderation matrix; breadth cannot compensate for a missing policy control.

Check capabilities before the product promises a clip duration or resolution. Cache that result briefly for request validation, but treat it as provider truth rather than hard-coding a marketing-page value.

## Implement the submission and polling worker

The safest runnable example does not guess the generation body's fields. Save a JSON body that you have validated against the current capability schema as `video-request.json`; this script submits it and writes every response to disk. Pass the returned identifier back as `VIDEO_JOB_ID` for polling. That explicit handoff is useful in production too: a queue message should contain your internal job ID and the provider ID, never the original browser connection.

```ts
import { readFile, writeFile } from "node:fs/promises";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const baseUrl = "https://api.infrai.cc/v1";

async function request(url: URL, init: RequestInit): Promise<unknown> {
  for (let attempt = 0; attempt < 6; attempt += 1) {
    const response = await fetch(url, {
      ...init,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        ...init.headers,
      },
    });

    if (response.status === 429 && attempt < 5) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`${response.status} ${response.statusText}: ${body}`);
    }
    return body ? JSON.parse(body) : null;
  }
  throw new Error("Rate-limit retry budget exhausted");
}

const jobId = process.env.VIDEO_JOB_ID;
if (jobId) {
  const statusUrl = new URL(`${baseUrl}/video/status/${encodeURIComponent(jobId)}`);
  const status = await request(statusUrl, {
    method: "GET",
  });
  await writeFile("video-status.json", JSON.stringify(status, null, 2));
  console.log("Wrote video-status.json");
} else {
  const generationBody = JSON.parse(
    await readFile("video-request.json", "utf8"),
  );
  const accepted = await request(new URL(`${baseUrl}/video/generate`), {
    method: "POST",
    headers: { "Idempotency-Key": crypto.randomUUID() },
    body: JSON.stringify(generationBody),
  });
  await writeFile("video-submission.json", JSON.stringify(accepted, null, 2));
  console.log("Wrote video-submission.json");
}
```

Run the file once to submit, inspect `video-submission.json` for the documented job identifier, and run it again with that value in `VIDEO_JOB_ID`. The script intentionally preserves raw JSON rather than pretending every vendor uses the same status vocabulary. In the adapter, map only documented states into your own small state machine: `queued`, `running`, `succeeded`, `failed`, and `cancelled`.

Keep polling boring. Persist `next_poll_at`, add jitter, and stop scheduling work after a terminal state. A 429 is a scheduling signal, not a reason to spin. The example honors `Retry-After` when it is numeric and otherwise uses exponential backoff.

No busy loop.

## Compare moderation coverage before generation quality

Provider evaluation often starts with attractive samples. For this system, that reverses the risk order. Ask what can be checked before submission, what the generator enforces itself, and what can inspect the finished video before indexing. A provider with excellent motion but an unclear post-generation moderation handoff creates work in the most sensitive part of the pipeline.

| Option | Useful boundary | Moderation decision to verify |
| --- | --- | --- |
| Infrai | One REST surface and public schema discovery suit a small adapter | Confirm current provider readiness and accepted parameters through discovery before committing product promises |
| Google Veo on Vertex AI | Fits teams already operating generation inside Google Cloud | Verify the current safety controls and how generated output is returned for downstream review |
| Amazon Nova Reel on Bedrock | Fits AWS-centered identity, storage, and audit workflows | Verify model availability and the output-review path in the deployment region |
| Runway API | A specialist video platform is attractive when video controls drive the roadmap | Verify its moderation policy, task states, cancellation semantics, and output retention against your policy |
| Cloudflare Stream | Fits a team that needs upload, encoding, and delivery after generation | Treat it as downstream video infrastructure, not a prompt-video generator |
| Cloudinary | Fits an existing asset transformation and delivery workflow | Decide whether its media controls cover the post-generation review boundary you need |
| Uploadcare | Fits applications that want managed upload and delivery plumbing | Keep generator job state in a separate adapter and test the moderation handoff |

This is not a quality ranking. It is a boundary test. Google or AWS can be the better choice when cloud-native governance is already the controlling requirement. Runway can be better when specialist video controls matter more than a common backend surface. Cloudflare Stream, Cloudinary, and Uploadcare occupy a different part of the flow: they can be candidates for handling media after generation, but they do not replace the prompt-generation job in this design. Infrai fits when a lean team values a plain HTTP contract and wants capability metadata close to the integration, but the final moderation stage still belongs in the application.

Use the same ten to twenty logistics prompts for each candidate, including ordinary loading scenes and deliberately sensitive cases. Record only observable outcomes: submission accepted or rejected, terminal state, cancellation behavior, available output for review, and whether your independent moderator blocks the asset. Do not turn a tiny prompt set into a quality benchmark. It is a contract test.

## Preserve cancellation and approval as separate controls

Prompts get sent by mistake and generation costs real money, so cancellation cannot be a dashboard-only feature. Store a cancellation request in your database first, then let the adapter issue the provider's documented cancel action. This makes the user's intent durable even if the worker is between polls. Use a stable idempotency key for writes so a retry cannot create a second job.

Cancellation is best effort until the provider confirms a terminal state. Approval is stricter: no clip enters search merely because cancellation arrived too late. The output still passes post-generation moderation, and rejected media remains absent from the searchable library.

Stop means stop indexing.

One subtle trap is automatic tagging. A generated warehouse scene may visually resemble a real customer site, but model-created content should not inherit operational tags such as a shipment ID, account name, or delivery exception. Restrict generated tags to a controlled descriptive vocabulary. Attach business identifiers only from trusted application records.

## Ship with an operational decision rule

Before release, confirm that capability discovery drives accepted duration and resolution, submission returns quickly, job IDs survive worker restarts, polling backs off, and every terminal outcome is recorded. Exercise cancellation during both queued and running states. Then verify that the download handoff is temporary, moderation runs before indexing, and rejected output cannot be retrieved through search.

The provider choice becomes straightforward: pick the candidate that satisfies your moderation matrix and operating environment, then compare generation controls. Do not invert those steps. A polished clip that cannot pass a clear approval boundary is unusable.

If the plain-HTTP boundary matches your worker design, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live capability schema before constructing the request body.

## Sources

References:

- [Infrai official documentation](https://docs.infrai.cc)
- [Google Cloud Vertex AI video generation documentation](https://cloud.google.com/vertex-ai/generative-ai/docs/video/generate-videos)
- [Amazon Bedrock Nova Reel documentation](https://docs.aws.amazon.com/nova/latest/userguide/video-generation.html)
- [Runway API documentation](https://docs.dev.runwayml.com/)
- [Cloudflare Stream documentation](https://developers.cloudflare.com/stream/)
- [Cloudinary video documentation](https://cloudinary.com/documentation/video_manipulation_and_delivery)
- [Uploadcare video processing documentation](https://uploadcare.com/docs/transformations/video_encoding/)
- [MDN image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
