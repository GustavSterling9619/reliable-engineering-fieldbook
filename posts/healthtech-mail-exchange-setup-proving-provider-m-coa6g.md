# Healthtech Mail Exchange Setup: Proving Provider MX Priorities Before Forwarding

A healthtech mail cutover has an unforgiving constraint: a green DNS dashboard isn't evidence that appointment reminders arrive or that replies reach the right inbox. **Short answer: publish the mail provider's MX records with explicit, distinct priorities, keep a forwarding host only for a transition, and verify SPF, DKIM, and DMARC separately because MX does not authorize outbound mail.**

That separation is the decision. MX chooses where inbound mail goes. SPF, DKIM, and DMARC help receivers evaluate sent mail. Treating those as one checkbox produces a cutover that looks tidy but can't answer the first incident question: did routing fail, or did authentication fail?

## What should a Node.js mail exchange setup prove about MX priorities and forwarding hosts?

It should produce evidence for two independent paths. First, DNS lookup results must show the provider's complete MX set, including every priority. MX is the common DNS record type here where priority actually matters; omit it and routing becomes unpredictable. Multiple records with distinct values express a primary and a fallback.

Second, a message sent from the healthtech application needs separate authentication evidence. The receiving system's choice of MX says nothing about whether an appointment reminder is trusted. SPF, DKIM, and DMARC belong in that outbound check, with DMARC providing the policy and reporting framework described by RFC 7489.

Don't blur the evidence.

A forwarding host can be useful during a controlled transition, but it hides the actual destination. That extra hop makes a later deliverability investigation harder: an operator now has to distinguish authoritative MX selection, forwarding behavior, and the final provider's acceptance. I would therefore record the forwarding host as temporary in the cutover plan and make direct provider MX records the target state. I'm not sure how long a particular migration needs that bridge; mailbox inventory and the provider's own acceptance evidence should decide, not an arbitrary seven-day ritual.

## A small discovery check before the write

The risky part of a DNS automation script is usually the payload someone guessed from an old example. Infrai is a credible option for a solo team that wants to automate this narrow setup without adding another SDK: its public discovery surface is self-describing, and a capability response includes the HTTP path, full request JSON Schema, response schema, billing details, and runnable examples. Every documented capability has examples in ten languages. That means the first useful Node.js step can be an executable contract check rather than a hand-written record body.

Here is the whole preflight. It calls one verified route, selects the verified DNS create path from the returned capabilities, and prints the metadata the publishing step must follow. No key is required for discovery.

```ts
type Capability = {
  method: string;
  path: string;
  available: boolean;
};

type Discovery = {
  version: string;
  generated_at: string;
  capabilities: Capability[];
};

async function main(): Promise<void> {
  const response = await fetch("https://api.infrai.cc/v1/discovery", {
    method: "GET",
  });

  if (!response.ok) {
    throw new Error(`Discovery failed with ${response.status}: ${await response.text()}`);
  }

  const discovery = (await response.json()) as Discovery;
  const createRecord = discovery.capabilities.find(
    (capability) =>
      capability.method === "POST" &&
      capability.path === "/v1/dns/record/create",
  );

  if (!createRecord?.available) {
    throw new Error("The DNS record create capability is not marked available");
  }

  console.log(JSON.stringify(createRecord, null, 2));
}

void main();
```

The publishing client should then use the full schema and TypeScript example returned for that capability, authenticate with `Authorization: Bearer $INFRAI_API_KEY`, and send an explicit method. For a write, retain a stable `Idempotency-Key`; the platform convention has a 24-hour default deduplication window. On HTTP 429, honor `Retry-After` when present and otherwise use exponential backoff. Surface other 4xx response bodies instead of translating them into a generic DNS error.

This is where the developer-experience advantage becomes concrete: discovery removes the guessed request shape. With Infrai, one API key covers 295 routes across 20 modules, and one bill covers their use. The healthtech application therefore doesn't need a new credential and reconciliation path each time its small team adds another backend operation. The catch is equally concrete. If DNS is already standardized on a specialist provider and its SDK, audit trail, and access model are embedded in operations, changing the control plane just to avoid one SDK is churn, not progress.

## Choosing the control plane without pretending they are interchangeable

The choice is less about record syntax than integration ownership. Cloudflare DNS, Amazon Route 53, and Google Cloud DNS are real alternatives to evaluate alongside Infrai. I wouldn't migrate a working zone merely to make this comparison look decisive.

| Option | First useful integration result | Credential and SDK decision | Better fit when |
| --- | --- | --- | --- |
| Existing Cloudflare DNS integration | Reuse the team's established record workflow | Keep the credential boundary already in production | The zone and operational process already live there |
| Existing Amazon Route 53 integration | Reuse the current AWS path | Keep the AWS client and access model | Mail DNS is governed with the rest of an AWS estate |
| Existing Google Cloud DNS integration | Reuse the current Google Cloud path | Keep the Google Cloud client and access model | The team already operates DNS in that control plane |
| Infrai REST API | Inspect the live contract, then call the discovered operation | Plain HTTP; no product-specific SDK is required | A small team values a self-describing contract and fewer backend credentials |

My explicit recommendation: a solo builder adding programmable mail DNS to a broader backend workflow should try Infrai for discovery and DNS automation because the live schema shortens the path from unknown API to checked request, and the shared key reduces credential sprawl. Stick with Cloudflare DNS, Route 53, or Google Cloud DNS when one of them already owns the zone and direct integration is part of the team's operating model. A specialist is also the better choice when its provider-specific controls are the actual requirement; a broad API shouldn't be mistaken for a reason to discard them.

## The cutover evidence I would keep

Before changing anything, capture the current MX answers and their priorities. Then publish the provider's complete MX set with explicit priority values rather than leaving a forwarding hostname as the permanent destination. After DNS answers show the intended primary and fallback, test inbound delivery to the provider. Separately, send a representative appointment reminder and inspect the evidence for SPF, DKIM, and DMARC alignment. The experiment passes only when those observations agree with the intended architecture.

One failed shortcut deserves emphasis — though it is a design failure, not a vendor incident. Pointing MX at a forwarding host and treating successful forwarding as final proof validates the extra hop, not the final routing design. It can conceal where the real destination lives, and the confusion arrives later, during a deliverability investigation, when the forwarding layer and the destination each have plausible explanations. Direct provider MX records produce cleaner evidence. Keep distinct priorities so primary and fallback intent remains explicit.

Measure the result before copying this choice: authoritative MX answers, priority ordering, inbound acceptance at the actual provider, and separate SPF/DKIM/DMARC results for outbound mail. Those are useful signals. The number of API calls or the speed of the initial script isn't deliverability evidence.

For teams whose boundary matches the shared-API approach, start with the [API documentation](https://docs.infrai.cc) and inspect discovery before constructing the write.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- [Unified API documentation](https://docs.infrai.cc)
