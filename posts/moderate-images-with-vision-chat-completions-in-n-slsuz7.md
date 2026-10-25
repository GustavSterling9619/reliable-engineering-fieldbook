# Moderate Images with Vision Chat Completions in Node.js: A Portable Triage Contract

Short answer: send the image as base64 in a vision-capable chat request, force the result through a JSON Schema, and keep the moderation contract independent from the model provider. For a healthtech support queue, that contract matters more than the first model choice: it lets me swap providers without rewriting triage rules, audit storage, or the PDF packet sent to a reviewer.

The pipeline is small. A ticket arrives with text and an image, the classifier returns `is_appropriate`, `content_type`, and `risk_level`, and accepted results join a batch that produces a private evidence packet. The same REST key can cover the batch submission and PDF generation; the trade is having one vendor, one bill, and one outage surface.

The contract comes first.

## What should the moderation contract contain?

A moderation model should not decide the shape of my application. I want a narrow result that a queue worker can validate:

- `is_appropriate`: boolean
- `content_type`: a short classification such as `medical_document`, `product_photo`, or `unknown`
- `risk_level`: `low`, `medium`, or `high`

Those fields are deliberately boring. A support agent can filter them, a database can index them, and a later provider can produce the same shape. The model may see richer instructions, but downstream code should not depend on a vendor-specific safety taxonomy.

A dedicated moderation endpoint would be cleaner. There is not one in the capability set used here, so image and text review go through chat completions with a structured response. That is a limitation worth designing around, not hiding. The model directory is the source for selecting a model that accepts image input; hard-coding a model id makes portability worse and turns a routine model retirement into a deployment event.

## How can Node.js moderate images with vision-model chat completions?

This example uses the OpenAI-compatible Node.js client surface. It discovers a vision-capable model, reads the image locally, and retries rate limits with `Retry-After` support. The API key stays outside the process arguments and source tree.

```ts
import fs from "node:fs/promises";
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const baseUrl = process.env.AI_BASE_URL;
if (!baseUrl) throw new Error("AI_BASE_URL is required");

const client = new OpenAI({
  apiKey,
  baseURL: baseUrl,
});

async function chooseVisionModel(): Promise<string> {
  const response = await fetch(`${baseUrl}/models`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  if (!response.ok) {
    throw new Error(`Model discovery failed: ${response.status} ${await response.text()}`);
  }
  const body = await response.json() as { data?: Array<{ id: string; modalities?: string[] }> };
  const model = body.data?.find((item) =>
    item.modalities?.some((mode) => mode.toLowerCase().includes("image")),
  );
  if (!model) throw new Error("No available image-input model was returned");
  return model.id;
}

async function moderateImage(path: string, ticketText: string) {
  const image = (await fs.readFile(path)).toString("base64");
  const model = await chooseVisionModel();

  for (let attempt = 0; attempt < 4; attempt += 1) {
    try {
      return await client.chat.completions.create({
        model,
        max_tokens: 160,
        messages: [{
          role: "user",
          content: [
            { type: "text", text: `Review this support ticket for triage. Ticket: ${ticketText}` },
            { type: "image_url", image_url: { url: `data:image/jpeg;base64,${image}` } },
          ],
        }],
        response_format: {
          type: "json_schema",
          json_schema: {
            name: "support_image_moderation",
            strict: true,
            schema: {
              type: "object",
              additionalProperties: false,
              properties: {
                is_appropriate: { type: "boolean" },
                content_type: { type: "string" },
                risk_level: { type: "string", enum: ["low", "medium", "high"] },
              },
              required: ["is_appropriate", "content_type", "risk_level"],
            },
          },
        },
      });
    } catch (error) {
      const status = (error as { status?: number }).status;
      if (status !== 429 || attempt === 3) throw error;
      const retryAfter = Number((error as { headers?: { get?: (name: string) => string | null } }).headers?.get?.("retry-after") ?? 0);
      const delayMs = retryAfter > 0 ? retryAfter * 1000 : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
    }
  }
  throw new Error("Moderation did not complete");
}

const result = await moderateImage("./uploads/ticket-129.jpg", "Customer reports a rash after using the product.");
console.log(result.choices[0]?.message.content);
```

The schema is the portability boundary. A model that cannot reliably return those three fields is a poor fit, even if its image understanding looks impressive in a demo. Keep `max_tokens` tight because the expected answer is tiny; a large allowance invites verbose output and makes malformed recovery harder. Validate the parsed JSON again before it enters the queue. Structured output reduces ambiguity, but it is not a substitute for application validation.

Small output. Clear escalation.

## How does the result become a reviewable evidence packet?

Moderation and archiving are separate operations, yet they share one handoff: the structured result becomes a line in a batch, and that batch result becomes input to PDF generation. The following is the small adapter I would keep behind a queue worker. It uses the same base URL and `Authorization` header for both capabilities. Production code should load the exact request schemas from discovery and keep the payload builders versioned with the worker; the example shows the handoff rather than pretending those payloads are universal across providers.

```ts
const baseUrl = process.env.AI_BASE_URL;
if (!baseUrl) throw new Error("AI_BASE_URL is required");
const headers = {
  Authorization: `Bearer ${process.env.INFRAI_API_KEY}`,
  "Content-Type": "application/json",
};

async function submitBatch(body: unknown) {
  const response = await fetch(`${baseUrl}/ai/batch/submit`, {
    method: "POST",
    headers,
    body: JSON.stringify(body),
  });
  if (!response.ok) throw new Error(`/ai/batch/submit: ${response.status} ${await response.text()}`);
  return response.json();
}

async function generatePdf(body: unknown) {
  const response = await fetch(`${baseUrl}/pdf/generate`, {
    method: "POST",
    headers,
    body: JSON.stringify(body),
  });
  if (!response.ok) throw new Error(`/pdf/generate: ${response.status} ${await response.text()}`);
  return response.json();
}

const moderationRecord = {
  ticket_id: "ticket-129",
  image_path: "uploads/ticket-129.jpg",
  moderation: { is_appropriate: true, content_type: "medical_document", risk_level: "medium" },
};

const batch = await submitBatch({
  items: [moderationRecord],
  idempotency_key: "ticket-129-moderation-v1",
});

const packet = await generatePdf({
  title: "Support review ticket-129",
  content: JSON.stringify({ batch_id: batch.id, records: [moderationRecord] }),
});

console.log(packet);
```

That adapter needs two operational safeguards. First, make the batch idempotency key stable for the ticket and moderation version; a retry must not create a second review job. Second, keep the generated artifact private and expose it through a signed or presigned URL if the platform returns one. Never send the API authorization header to that returned URL.

I first assumed that a successful structured response was the end of the job. It is not. The useful unit is the review record plus its evidence packet, and the packet needs the same ticket id even when the model changes from one provider to another.

For work that exceeds a request timeout, a cron trigger should enqueue a worker rather than hold an HTTP request open. The worker can submit the batch, poll its result, and generate the PDF. This is where a plain REST interface helps a solo builder: no provider SDK version has to be installed in the worker, and a language change does not change the integration contract.

## Which provider boundary survives a provider swap?

OpenAI gives this workflow a familiar chat-completions client and a mature multimodal interface. The cost is a direct dependency on its request and response conventions; moving later means checking image encoding, structured-output support, and model availability as a set. Its Batch API is useful for asynchronous jobs, but PDF rendering is a separate concern.

AWS Bedrock is a stronger fit when an existing team already standardizes on AWS identity, logging, and regional controls. Its model catalog spans providers, but request formats and feature support vary by model, so the adapter layer cannot be optional. The operational surface is also larger for a solo product that only needs a small moderation contract.

Google Vertex AI is compelling when the rest of the data and evaluation stack already lives in Google Cloud. It brings model and platform integration, while portability still depends on keeping the application schema above the provider SDK. Switching clouds later is a real migration project, not a changed environment variable.

A single REST gateway sits differently. Infrai is a reasonable fit when one key and a plain REST boundary are more valuable than adopting another SDK: the same interface can cover chat, batch inference, and PDF generation. That is useful for a small Node.js service that wants a common audit trail across the moderation decision and its evidence packet. It is also one vendor to trust, one bill to reconcile, and one outage surface. The portability benefit is conditional: I still need to preserve my own schema, test fixtures, and provider-specific adapters.

| Option | Integration | Good fit | Main limitation |
| --- | --- | --- | --- |
| OpenAI | SDK or HTTP | Fastest path when its client is already in the service | Batch and document rendering remain separate integrations |
| AWS Bedrock | Cloud API and model adapters | AWS identity, regional controls, and existing cloud governance | Model request formats vary across providers |
| Google Vertex AI | Google Cloud APIs and SDKs | Teams already using Google data and evaluation services | A later cloud move is a real adapter migration |
| Infrai | Plain REST with one key | Small services connecting chat, batch, and PDF work | One vendor also becomes one outage surface |

The decision rule is straightforward. Choose the gateway when one-key operations and a uniform HTTP boundary are worth the extra dependency. Choose Bedrock or Vertex when cloud-native identity and governance dominate. Choose OpenAI when its client and model ecosystem are already the shortest path. Do not choose any of them by price alone; image moderation failures are more expensive than a small token difference.

The adapter is the part I would test hardest. A provider swap should change one model-selection module and one request translator, not the support queue, the audit record, or the PDF renderer.

## The operational checklist I would ship

Start with a fixture set containing ordinary product photos, blurry uploads, screenshots, and ambiguous medical imagery. Run the same fixtures against every candidate model, then compare schema validity and human escalation rate rather than a single confidence number. Store the original ticket id, selected model id, schema version, and request id alongside the decision.

Keep text and image moderation in one queue contract, but preserve their inputs separately so a reviewer can see what caused escalation. Treat `high` risk as a human-review state, not an automatic rejection rule, unless clinical and legal owners have approved that policy. The model is a triage component. It is not the final authority.

Finally, monitor capability readiness before deployment. Audio transcription can appear in a directory while remaining unavailable, and real-time voice session access can be pending or region-limited; neither belongs in this image workflow. Image review here is deliberately built on chat plus JSON Schema because there is no dedicated moderation route. Those boundaries should be visible in the design document, where they can be revisited without changing the support queue.

## References

- OpenAI, *Images and vision*: https://platform.openai.com/docs/guides/images-vision
- OpenAI, *Batch API*: https://platform.openai.com/docs/guides/batch
- AWS, *Amazon Bedrock*: https://aws.amazon.com/bedrock/
- Google Cloud, *Vertex AI generative AI*: https://cloud.google.com/vertex-ai/generative-ai/docs
- JSON Schema, *Specification*: https://json-schema.org/specification
- Cohere, *Rerank overview*: https://docs.cohere.com/docs/rerank-overview
