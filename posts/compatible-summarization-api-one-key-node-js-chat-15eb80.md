# Compatible Summarization API — One-Key Node.js Chat Completions Model Switching

Short answer: put one narrow TypeScript adapter between the application and any chat-completions-compatible runtime, then accept a candidate score only after local schema validation and rubric checks. Use one credential at that boundary if the runtime supports it, but treat model switching as a controlled deployment. The request may stay fixed while output correctness, latency, and token usage change.

For an edtech hiring workflow, a compact summary is useful only as evidence for a score, never as a substitute for the job rubric. The least complex reliable flow is to normalize the submitted text, request a structured evidence summary and scores, validate the JSON locally, recompute the weighted total, and store the model identifier plus usage metadata beside the decision.

## Can one compatible summarization API handle chat completions and model switching?

Transport compatibility gets a request across the wire. It does not prove that a returned object contains every criterion, that a score is within range, or that a quotation exists in the candidate's material. Those are application invariants.

Keep the contract small. Each criterion needs an identifier, an integer score, and evidence copied from the submitted text. The response also needs a short summary and declared maximum. Reject duplicate criteria and extra fields so the model cannot quietly invent another scoring scheme.

That is the boundary.

There is a separate boundary around sensitive data. Candidate records can contain identifiers, employment history, or health information. If a workflow handles electronic protected health information for a covered entity or business associate, 45 CFR Part 164 includes privacy and security requirements; an API-compatible transport does not settle compliance. Data classification, access controls, retention, agreements, and audit evidence remain deployment decisions.

## Put the executable contract before the router

This runnable core has no provider-specific SDK. The endpoint and model are configuration; the scoring rule stays fixed.

```ts
const rubric = [
  { id: "analysis", weight: 2, max: 4 },
  { id: "communication", weight: 1, max: 4 },
] as const;

type CriterionId = (typeof rubric)[number]["id"];
type Scorecard = {
  summary: string;
  criteria: Array<{ id: CriterionId; score: number; evidence: string }>;
  weightedTotal: number;
  weightedMaximum: number;
};

function isScorecard(value: unknown): value is Scorecard {
  if (typeof value !== "object" || value === null) return false;
  const record = value as Record<string, unknown>;
  if (typeof record.summary !== "string" || !Array.isArray(record.criteria)) return false;
  if (typeof record.weightedTotal !== "number") return false;
  if (typeof record.weightedMaximum !== "number") return false;

  const remaining = new Set(rubric.map((item) => item.id));
  return record.criteria.length === rubric.length && record.criteria.every((item) => {
    if (typeof item !== "object" || item === null) return false;
    const result = item as Record<string, unknown>;
    const definition = rubric.find((entry) => entry.id === result.id);
    return Boolean(
      definition &&
      remaining.delete(definition.id) &&
      Number.isInteger(result.score) &&
      (result.score as number) >= 0 &&
      (result.score as number) <= definition.max &&
      typeof result.evidence === "string" &&
      result.evidence.length > 0,
    );
  }) && remaining.size === 0;
}
```

The duplicate-ID check matters. Two valid `analysis` entries could otherwise pass a shallow shape check while omitting `communication`. That syntactically clean failure is easy to miss when monitoring stops at HTTP status codes.

Now make the network boundary boring.

```ts
type RuntimeConfig = { baseUrl: string; apiKey: string; model: string };
type ChatResponse = {
  model?: string;
  choices?: Array<{ message?: { content?: string } }>;
  usage?: { prompt_tokens?: number; completion_tokens?: number };
};

async function scoreCandidate(config: RuntimeConfig, candidateText: string) {
  const response = await fetch(`${config.baseUrl}/chat/completions`, {
    method: "POST",
    headers: {
      authorization: `Bearer ${config.apiKey}`,
      "content-type": "application/json",
    },
    body: JSON.stringify({
      model: config.model,
      temperature: 0,
      messages: [
        {
          role: "system",
          content: "Return JSON only. Score every criterion. Evidence must be an exact quote from the candidate text.",
        },
        { role: "user", content: JSON.stringify({ rubric, candidateText }) },
      ],
    }),
  });

  if (!response.ok) throw new Error(`Runtime failed with status ${response.status}`);
  const envelope = (await response.json()) as ChatResponse;
  const content = envelope.choices?.[0]?.message?.content;
  if (!content) throw new Error("Runtime returned no message content");

  const parsed: unknown = JSON.parse(content);
  if (!isScorecard(parsed)) throw new Error("Scorecard failed schema validation");

  const total = parsed.criteria.reduce((sum, result) => {
    const definition = rubric.find((item) => item.id === result.id)!;
    return sum + result.score * definition.weight;
  }, 0);
  const maximum = rubric.reduce((sum, item) => sum + item.max * item.weight, 0);
  if (parsed.weightedTotal !== total || parsed.weightedMaximum !== maximum) {
    throw new Error("Scorecard arithmetic is inconsistent");
  }
  for (const result of parsed.criteria) {
    if (!candidateText.includes(result.evidence)) {
      throw new Error(`Evidence for ${result.id} is absent from the submission`);
    }
  }

  return { scorecard: parsed, model: envelope.model ?? config.model, usage: envelope.usage };
}
```

Cost calculation is intentionally absent. Token counts are operational data, but multiplying them by a mutable price table inside the decision path mixes two jobs. Record usage, then calculate spend in reporting with a dated rate table and the billing unit returned by the selected service. Lowest-price claims age badly when models, caching rules, and accounting units change.

## Switch models with a shadow decision

A single key and stable envelope reduce integration work. They do not remove migration work. OpenAI, Anthropic's Claude, and Google's Gemini expose different native concepts, and a compatibility layer chooses how to translate or omit them. Even when the same chat-shaped fields are accepted, test structured-output behavior against the exact model and intermediary configuration intended for production.

Build a frozen evaluation set from consented, de-identified candidate submissions. Include strong answers, weak answers, missing criteria, instruction-like text inside a resume, Unicode, long input, and evidence that appears twice. Keep the human-approved result separate from the prompt. Fifty varied cases are a useful engineering starting point, not a universal statistical threshold. Run a candidate model in shadow mode and compare schema acceptance, criterion coverage, evidence grounding, weighted-score agreement, latency distributions, and input/output usage. Averages hide tails, so inspect p50 and p95 latency on the same workload; compare cost using observed usage under a dated rate snapshot, not a headline unit price. Promote only against a written gate. One defensible gate requires 100% parse success and zero missing criteria on the frozen set, because an invalid scorecard cannot be repaired by averaging, while the acceptable score disagreement rate remains a policy choice that needs domain review.

Malformed JSON is obvious. Plausible but unsupported evidence is worse. A retry may fix syntax, yet repeated retries can turn one submission into several judgments while raising latency and usage. Cap attempts, classify the reason, and do not retry a deterministic contract failure with the same prompt and model.

A production record should retain the rubric version, prompt version, requested and reported model identifiers, validation outcome, correlation ID, timestamps, token counts when supplied, and a hash of normalized input. Store raw candidate text only under its retention and access policy. Logs should not become a second candidate database.

Human review needs a clear state. Route absent evidence, inconsistent arithmetic, exhausted retries, and out-of-policy disagreement to `needs_review`; do not coerce them into zeroes. If the runtime is unavailable, queueing may fit a workflow that tolerates delay. Otherwise return a typed failure.

Fail closed.

The central trade-off is speed versus auditability. Requesting a summary in one call and a score in another can make each prompt easier to inspect, but it adds latency and repeats input tokens. I would start with one call, provided every score carries verbatim evidence and the application checks both evidence and arithmetic. Split the calls only when evaluation shows summary contamination or separate retention rules require it.

## Operate the contract, not the brand name

Before launch, pin rubric and prompt versions, set input limits, redact fields the scorer does not need, and define the review state. Exercise timeouts, non-JSON bodies, duplicate criteria, missing usage, out-of-range scores, and quotations absent from the source. Run the frozen set against the exact production route.

After launch, watch validation failures and review referrals by model version. Sample accepted scorecards for evidence quality under an approved review process, because a schema proves shape, not fairness or relevance. Track latency and usage separately from correctness. A fast invalid score is still invalid.

Model portability is earned through a narrow adapter, strict local validation, versioned evaluation data, and an auditable promotion rule. Those controls let one credential and model switching simplify operations without pretending different runtimes make identical decisions. The rubric remains the authority.

## Further reading

- LangChain ChatOpenAI integration documentation: https://python.langchain.com/docs/integrations/chat/openai/
- 45 CFR Part 164, Security and Privacy: https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164
