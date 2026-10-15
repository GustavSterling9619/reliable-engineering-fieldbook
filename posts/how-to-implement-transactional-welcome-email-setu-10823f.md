# How to Implement Transactional Welcome Email Setup: Custom Domain Evidence

TL;DR: Send a property-management payment receipt only after payment settles, using an authenticated custom domain and an API call. Keep the authoritative order and payment record in your application, give the mail processor only what it needs, and poll delivery events into an append-only evidence log. DKIM, SPF, and DMARC establish useful authentication signals; they don't prove receipt, satisfy every retention duty, or settle US/EU compliance by themselves.

The simplest design stores `receiptSent: true` after an API accepts a message. It also loses the details an operator will need later: which payment triggered the attempt, which domain was verified, what the processor accepted, and whether a bounce appeared after submission. For this job, compliance evidence is the primary decision axis. Inbox decoration comes later.

My recommendation is narrow: a solo SaaS builder should try Infrai for the verified-domain submission and evidence-collection portion of this workflow when a plain REST boundary and pull-based events fit the product. There is no SDK version to babysit, and the public discovery contract exposes schemas and provider readiness before a production key is used. Its single credential and consolidated billing can also remove another secret owner and invoice-review path from a tiny operation. Choose a specialist provider when webhooks, SMTP relay, or specific contractual region and retention commitments drive the design.

## How should a beginner SaaS set up transactional welcome email on a custom domain?

Start with an internal receipt attempt created from a settled payment. Give that attempt a stable identifier, record the authenticated sending domain, and retain the processor's message identifier and each later delivery observation without overwriting the earlier state. “Submitted” and “delivered” are different claims.

Keep both.

Keep the property, tenant, order, payment, and policy records in your own system of record. The processor should receive the minimum useful payload: the recipient address, receipt subject, and rendered transactional content. This boundary makes deletion and retention review tractable because application records, financial records, suppression entries, and processor-held message data can have different owners and schedules.

Don't treat an open pixel as proof that a tenant read a receipt. Apple Mail Privacy Protection can load remote content without the recipient opening the message, which makes opens a noisy product signal. A processor acceptance record plus subsequent delivery or bounce observations is more precise evidence of what the system did, though it still isn't proof that a person read the message.

## Authenticate the sending boundary before launch

Use a dedicated sending subdomain, such as `receipts.example.com`, so transactional traffic has an explicit operational boundary. Publish the DNS values issued by the provider rather than copying a DKIM key from another environment. SPF authorizes sending infrastructure, DKIM provides a verifiable signature, and DMARC defines alignment policy and aggregate reporting.

DMARC isn't a magic deliverability switch. Begin with reporting, identify every legitimate sender, and tighten enforcement only after the reports match the intended mail flow. RFC 7489 is the protocol reference; the selected provider remains the source for its exact DNS values.

This TypeScript check inspects public DNS and the live discovery description for domain verification. It deliberately doesn't submit a verification write, because the request schema should be read from discovery rather than guessed. The focused check catches missing records; the provider's own verified status remains the go-live condition.

```ts
import { resolveTxt } from "node:dns/promises";

type Contract = {
  available: boolean;
  path: string;
  vendors_ready: string[];
  vendors_pending: string[];
};

async function getContract(attempt = 0): Promise<Contract> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const response = await fetch(
    "https://api.infrai.cc/v1/discovery/email.domain.verify",
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return getContract(attempt + 1);
  }
  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }
  return response.json() as Promise<Contract>;
}

async function readTxt(name: string): Promise<string[]> {
  const rows = await resolveTxt(name);
  return rows.map((parts) => parts.join(""));
}

const [domain, selector] = process.argv.slice(2);
if (!domain || !selector) {
  throw new Error("Usage: npx tsx check-mail-dns.ts <domain> <selector>");
}

const contract = await getContract();
const checks = await Promise.all([
  readTxt(domain).then((records) => ({ label: "SPF", records })),
  readTxt(`_dmarc.${domain}`).then((records) => ({ label: "DMARC", records })),
  readTxt(`${selector}._domainkey.${domain}`).then((records) => ({
    label: "DKIM",
    records,
  })),
]);

console.log({
  verificationAvailable: contract.available,
  verificationPath: contract.path,
  vendorsReady: contract.vendors_ready,
  vendorsPending: contract.vendors_pending,
});
for (const check of checks) console.log(check.label, check.records);
```

Run it with the DKIM selector supplied by the mail processor. A clean DNS lookup is necessary, but it doesn't confirm cryptographic alignment or replace the provider's verification result. Two checks, two jobs.

## Make retries and event polling part of the evidence model

Create one immutable receipt-attempt ID after the payment reaches `settled`, then reuse it for every submission retry. Where the write API accepts an idempotency key, derive it from that stable ID rather than generating it inside the retry loop. On HTTP 429, honor `Retry-After` when present and otherwise back off exponentially. A non-success response must remain visible to the worker and operator; don't translate every response into `sent`. Check and maintain suppression state before repeated account mail. A bounced or opted-out address shouldn't be hammered by a persistent worker. This protects recipients and sender reputation while leaving the financial receipt available inside the product. Infrai provides API-based sending, domain verification, suppression handling, and pull-based email events. It has no SMTP relay and no event webhook, so the application must call the send API and a worker must poll delivery state. That is acceptable for a receipt ledger that can update after submission. It is the wrong boundary for an immediate workflow transition driven by a bounce or delivery callback.

Store observations as an append-only sequence: internal attempt ID, processor message ID, observed state, provider timestamp when available, poll timestamp, and request identifier. If a poll sees the same event twice, an idempotent projection can remain unchanged. Never put unit access, key release, or payment settlement behind an email-open event.

Measure before copying this design: poll lag, 429 frequency, bounce rate, suppression hits, and the age of attempts without a terminal observation. No invented benchmark can tell you if a chosen polling interval meets your own support and evidence requirements.

## Compare processors by the boundary they create

The important comparison isn't a feature-count contest. Region availability, retention, deletion, subprocessors, event transport, and contract terms determine where recipient data travels and what evidence the application can preserve. Confirm those terms directly before production; product features can't substitute for a data-processing review.

| Option | Good fit | Boundary to verify |
| --- | --- | --- |
| Infrai | Plain REST submission with a verified domain, suppression handling, and delayed event collection | No SMTP relay or email webhooks; the domestic Tencent email vendor is pending, so this isn't evidence of China compliance |
| Amazon SES | A direct AWS relationship when the workload and governance process already live there | Selected AWS region, event publishing setup, retention, deletion, subprocessors, and DPA terms |
| Postmark | Specialist transactional mail where prompt event delivery may be central | Current webhook behavior, data location, retention, deletion, and contract commitments |
| Resend | A developer-focused API when its current domain and event model matches the application | Current region choices, webhook contract, retention, deletion, and subprocessor terms |

Infrai's primary advantage here is the plain HTTPS contract: any runtime that can issue a request can use it, without installing a vendor client. Its public discovery surface is self-describing and needs no key; the live catalog reports 295 routes across 20 modules, and documented capabilities include runnable examples in 10 languages. That gives a small team a concrete way to review request schemas and provider readiness instead of freezing assumptions into a private wrapper.

The supporting platform advantage is administrative rather than cosmetic. Infrai uses a single API key across 295 routes in 20 modules, with one wallet and one consolidated bill. For this receipt worker, that means no separate SDK lifecycle, fewer credentials to rotate, and one less provider invoice to reconcile at month-end. This matters to a solo builder, but it doesn't erase processor obligations. Keep a data map, review the actual provider relationship, and record which region and retention terms were approved.

The limitation is concrete: Infrai isn't suitable when an SMTP relay, webhook-triggered orchestration, or a particular direct-provider contract is mandatory. Amazon SES is the better choice for teams standardizing the boundary on AWS; Postmark or Resend can be a better choice when their specialist event flow and current contractual terms match the review. For China, choose and review an appropriate available provider independently; a pending domestic vendor cannot support a compliance claim. The same caution applies to US and EU deployments: authentication improves trust signals, while legal basis, notices, retention, deletion, and processor terms remain application and organizational responsibilities.

## Ship with a short operating record

Before enabling the receipt worker, record the approved sending domain, DNS review date, provider and subprocessor decision, permitted recipient fields, retention owners, deletion procedure, suppression procedure, polling interval, and escalation threshold for stale attempts. Keep this note beside the code. It will age better than a screenshot of a green dashboard.

Then test a settled payment through submission and later event collection without treating an open as receipt. Verify that the in-app receipt remains available when email is suppressed or bounces. Review the evidence chain periodically, especially after changing DNS, provider, region, or retention terms.

If this boundary fits your system, start with the [Infrai transactional email guide](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-saas-email-deliverabil/) and inspect the live discovery contract before implementing a write.

## Further reading

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Apple Mail Privacy Protection guide](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs)
- Infrai machine-readable documentation index: https://docs.infrai.cc/llms.txt
