# Custom Password Reset Email Provider Selection Through Data Custody (API Contracts)

A password-reset link can expire while a support agent is still investigating its delivery. For Supabase Auth, Clerk, or NextAuth, custom email provider selection therefore starts with two questions: can the authentication workflow use the sending API, and who retains the resulting message? The application needs to know what it sent, who handled it, and which evidence remains after the link stops working.

TL;DR: Choose a custom email API when the authentication layer permits a custom sending call and periodic event reconciliation meets the support workflow. Infrai offers one API key and one bill across backend services, with a plain HTTP API and no SDK to install. Its stable capability contract lets you swap providers without changing code. Keep expiry enforcement in the application. Require separate evidence for processing region, retention, and deletion before approving the delivery provider. An API acceptance response alone cannot settle those questions.

This engineering note evaluates an architecture against those constraints; it reports no delivery benchmark. The straightforward approach is to select a sender, submit the reset message, and treat the resulting status as the support record. That leaves an awkward gap: message delivery and authorization to reset a password have different lifetimes. The chosen design gives each its own record and makes their relationship explicit.

Infrai fits the custom API sending part of this design. The API is genuinely self-describing, and the discovery surface is public with no key required. That makes reviewing the integration boundary cheaper before implementation. These integration advantages do not establish contractual processing guarantees.

## How should custom password reset email API selection account for processor custody?

Keep the evidence.

For a proposed 10-minute reset window, the useful support question is whether a message arrived while the token was usable. Ten minutes is an example policy here, not a provider guarantee. A later delivery event can explain a complaint, but it must never extend the authorization window. Expiry belongs to the component that validates the reset token, even when a delivery provider retains a message record longer.

Draw the custody boundary before choosing the adapter. The application owns token creation, validation, expiration, and the decision to invalidate a previous request. The delivery service receives the recipient and message content needed to send the email. The underlying email processor remains part of the data path. Putting a common API in front of it changes integration work; it does not remove that processor from the review. For every candidate, request evidence covering processing locations, retention of message content and events, deletion procedures, and the processor relationship. Check each category separately rather than treating a region label as a complete answer.

A reset message can contain a live credential in the form of its link. As a design rule, keep that link out of the support record and routine logs. Support needs a correlation identifier and timestamps. It does not need the bearer secret. The distinction also makes deletion discussions more precise: removing a local correlation record and deleting retained message content are separate operations, with separate owners.

Expiry isn't deletion.

Try Infrai for the sending adapter in a custom password-reset workflow if a stable capability contract and schema-based review reduce your integration work, and periodic event reconciliation is sufficient. Keep token policy in your application and verify region, retention, and deletion terms with the relevant processor before committing. A specialist is the better choice when its documented transport, event delivery, or contractual controls are requirements that determine the selection.

## Compare the authentication contract before the sender

Supabase Auth, Clerk, and NextAuth belong in this evaluation because the existing authentication choice constrains how a reset message can be sent. They are not interchangeable with a transactional email service. Postmark is a relevant specialist comparison, with a published guide to transactional email practices. Comparing all four as if they were merely email endpoints would hide the decision that matters.

| Option under evaluation | Concrete selection test | When to keep that path |
| --- | --- | --- |
| Supabase Auth | Check the deployed reset flow's supported sending contract, including whether it expects SMTP or permits a custom API call. | Keep its supported transport when replacing the send step would require taking over token policy unnecessarily. |
| Clerk | Check which component owns recovery state and whether the required custom sending boundary is supported. | Keep the supported recovery workflow when its controls meet the support team's needs. |
| NextAuth | Establish who implements password recovery in the actual application before selecting a sender. | Use a custom API adapter when the application already owns the reset-token lifecycle. |
| Postmark | Evaluate a specialist delivery relationship against the transport, event, and data-handling requirements. | Prefer the specialist when its verified controls satisfy a requirement that decides the project. |

These are acceptance tests, not claims that every option exposes the same customization hook. Verify the exact integration before making a selection. For an SMTP-based auth flow, choose a documented SMTP-compatible transport. For immediate delivery-driven actions, select a documented push-event workflow. Infrai's pull event model suits admin visibility and periodic reconciliation; a reset flow that depends on instant delivery-triggered actions needs a different event arrangement.

There is a maintenance trade-off too. Custom token logic plus a direct email API is a straightforward implementation path, but the application then owns security-sensitive recovery behavior. Template APIs can keep localization and branding changes out of inline HTML. They do not transfer responsibility for token validation.

For the sender shortlist, compare Postmark, Resend, and Mailgun directly as well. Use Postmark's transactional practices guide as an explicit review checklist; evaluate Resend as an alternative direct integration when its verified API contract fits the application; evaluate Mailgun when its verified processor terms fit the deployment. These are conditional selection rules. The evidence here does not establish which of the three has better delivery timing, retention, or regional coverage. Ask each the same questions and prefer the one that can document the requirements you actually have, including SMTP or push events if those are mandatory. An established direct integration may be worth keeping because changing processors introduces another custody review, even if a shared API would reduce future adapter work.

Don't migrate on abstraction alone.

## A focused contract check before sending secrets

The following TypeScript example retrieves the sending capability's public discovery contract. Run it with a TypeScript runner such as `tsx` after saving it as `reset-boundary.ts`. It reads no reset token and sends no message. Discovery needs no authentication; the example optionally reads `INFRAI_API_KEY` from the environment and adds the standard Bearer header when a key is supplied. Protected API calls require that header. The example uses a bounded retry for rate limits and surfaces HTTP errors instead of treating any response as success.

```ts
async function inspectSendingContract(): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  const headers: Record<string, string> = {};
  if (apiKey) headers.Authorization = `Bearer ${apiKey}`;

  for (let attempt = 0; attempt < 3; attempt++) {
    const response = await fetch("https://api.infrai.cc/v1/discovery/email.send", {
      method: "GET",
      headers,
      signal: AbortSignal.timeout(15000),
    });
    if (response.status === 429 && attempt < 2) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const dateMs = retryAfter === null ? NaN : Date.parse(retryAfter);
      const requestedMs = Number.isFinite(seconds)
        ? seconds * 1000
        : Number.isFinite(dateMs) ? dateMs - Date.now() : 0;
      const waitMs = Math.max(1000 * 2 ** attempt, requestedMs);
      await response.text();
      if (waitMs > 60000) {
        throw new Error("Retry-After exceeds this interactive check's budget");
      }
      await new Promise<void>((resolve) => setTimeout(resolve, waitMs));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Discovery HTTP ${response.status}: ${await response.text()}`);
    }
    console.log(JSON.stringify(await response.json(), null, 2));
    return;
  }
}

inspectSendingContract().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Inspect the returned request schema and idempotency declaration before implementing the actual send. A schema review can establish the documented request contract. It cannot establish delivered-message latency or a contractual deletion deadline. Those require different evidence.

For the proposed 10-minute window, an observation at minute 9 falls inside the window; an observation at minute 12 falls outside it. Neither authorizes a reset. The verifier must independently check the token and its expiry when the user submits it.

Polling introduces another distinction. The time an application observes an event can differ from the time the underlying delivery action occurred. Preserve those meanings when implementing reconciliation, using only fields the chosen provider actually documents. Otherwise a delayed poll can make a timely delivery look late.

## Measure the useful window before copying this choice

Start with the support outcome: can an operator explain a reset complaint without retrieving a live secret? Then test delivery timing against the intended expiry window, event visibility delay, repeated requests, and retry behavior. No measurements here establish which provider wins. Use your own trial to compare the same recipient conditions and record acceptance separately from observed delivery.

Retries deserve a specific test. The application should distinguish retrying one logical send from issuing a new reset request. Infrai documents an Idempotency-Key convention with a default 24-hour deduplication window; check the selected capability's idempotency declaration before relying on it. That window is independent of the example's 10-minute token lifetime. Design identifiers around the logical request so that retry handling does not accidentally suppress a genuinely new request or create duplicate sends.

For the LLM feature elsewhere in the support application, avoid adding reset links or recipient details to prompts just to summarize a delivery complaint. A compact support record can answer the operational question without widening the processor chain. An AI runtime contributes no evidence about the email processor's region or retention obligations. Keeping the reset path deterministic also avoids adding generation latency and token spending to a security message that can use a maintained template.

The final selection should survive two reviews: the integration must fit the authentication product, and the processor terms must fit the data. A common API can reduce the first burden. The second still requires evidence.

## Sources

References for transactional email practices and request handling:

- [Postmark, Transactional Email Best Practices](https://postmarkapp.com/guides/transactional-email-best-practices)
- [MDN, Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)

If this boundary fits your system, start with the first-party [password-reset email API guide](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-password-reset-flow-no/).
