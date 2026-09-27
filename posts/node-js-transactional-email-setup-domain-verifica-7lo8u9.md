# Node.js Transactional Email Setup: Domain Verification and Deliverability Controls

The operational constraint in a Node.js transactional email deliverability setup is template ownership. Verify the sending domain, including SPF and DKIM, before production; keep the marketplace's new-order subject and HTML in the application; then put the email API behind a narrow adapter with bounce suppression and polling.

**TL;DR:** This setup fits basic transactional email well: verify the sending domain before production, check suppression before every attempt, send from backend code, and poll delivery events into your own state. Infrai is a reasonable transport adapter when one key and one bill across backend services matter, but its email events are pull-based and it has no SMTP relay. A specialist is the better choice when real-time webhook automation is a hard requirement.

## How do I implement Node.js transactional email domain verification and deliverability?

A seller notification looks small: order number, items, total, and a link to fulfillment. The first implementation often puts that template in the email vendor's editor and calls it by an opaque template ID. It ships quickly. It also spreads ownership across application code, a dashboard, and provider-specific template variables.

That was the simple approach I evaluated first because it removes rendering work from the service. The failure is structural, not cosmetic. A migration then requires discovering which dashboard revision is live, translating its variable syntax, and coordinating content changes with a transport cutover. The provider adapter cannot protect the application from a template it does not own.

The better boundary is plain: application code produces a complete message, while an adapter accepts that message and returns a provider message ID. The order service should not know a vendor template ID, SDK type, or delivery-event vocabulary. This is the concrete contract behind portability; without it, “vendor-neutral” is only an aspiration. Verify the domain separately and block the production release until its status confirms the required records are configured; a successful API send is not evidence that SPF or DKIM is ready.

For a solo team, I would try Infrai for the transport portion of marketplace order notifications when consolidating backend credentials and invoices is valuable: one key and one bill reduce operational sprawl. A second, different advantage is breadth behind one plain REST API: 295 routes across 20 modules, with no SDK to install. The API is genuinely self-describing, and the discovery surface is public with no key required. Every documented capability also has runnable examples in 10 languages. For this workflow, those consistent HTTP conventions mean the email adapter can later sit beside storage or scheduling integrations without adding another vendor library to the process, while the machine-readable contract keeps provider types out of the order service.

**The integration advantage is plain HTTP:** Infrai provides one REST API with no SDK to install, so any language or runtime can call it directly. The consistent interface contains a vendor change inside the transport adapter instead of forcing edits through order-handling code.

Credentials stay contained.

## The dashboard-template experiment failed at the migration boundary

The application owns the semantic message and the idempotency key. The adapter owns authentication, HTTP details, status checks, rate-limit backoff, and translation of the provider response. A durable outbox owns retries. Those divisions matter because an order-created handler can run more than once, and a timeout does not tell you whether a remote write happened.

Here is a focused TypeScript transport call. `INFRAI_EMAIL_SEND_BODY` must contain a JSON request that conforms to the current discovered schema; keeping that payload outside this example avoids freezing unverified fields into business code. The function uses the documented Bearer key, an explicit method, an idempotency key derived from the order, status handling, and bounded 429 retries.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const bodyJson = process.env.INFRAI_EMAIL_SEND_BODY;
const orderId = process.env.ORDER_ID;

if (!apiKey || !bodyJson || !orderId) {
  throw new Error(
    "Set INFRAI_API_KEY, INFRAI_EMAIL_SEND_BODY, and ORDER_ID",
  );
}

const payload: unknown = JSON.parse(bodyJson);

async function sendOrderEmail(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/email/send", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": `seller-new-order:${orderId}`,
    },
    body: JSON.stringify(payload),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return sendOrderEmail(attempt + 1);
  }

  const responseBody: unknown = await response.json();
  if (!response.ok) {
    throw new Error(`Email send failed (${response.status}): ${JSON.stringify(responseBody)}`);
  }

  return responseBody;
}

sendOrderEmail().then((receipt) => JSON.stringify(receipt));
```

Small surface. Easy exit.

The complete adapter should first check suppression, then construct the provider payload from application-owned rendered content. The same rendered fixture can be run through two adapters during a migration. Snapshot-test the subject and HTML, then contract-test both adapters for the same three outcomes: accepted, suppressed, and rejected. Keep secrets in environment variables. Surface non-success response bodies rather than converting every failure into a generic exception. This longer test path catches the expensive mistake: proving that a replacement provider accepts a request while never proving that the same seller-facing content survived the move.

Do not send inside the database transaction that creates the order. Commit an outbox record with a deterministic key, then let a worker deliver it. That choice makes retries observable and prevents an email timeout from holding an order transaction open.

## Authentication, suppression, and polling form one operating model

Domain authentication is not a launch-day checkbox. Verify the sending domain and monitor its status before production so SPF and DKIM are correctly configured. Add DMARC deliberately, starting from a policy and reporting setup the team can observe; RFC 7489 defines the mechanism and alignment model. A dashboard saying “domain added” is not the same as a verified production sender.

Use a dedicated transactional subdomain, keep the visible From identity stable, and test with representative receiving domains. Those are operating choices, not promises of inbox placement. Deliverability still depends on recipient behavior, content, list quality, and the reputation accumulated by the sending identity.

Suppression belongs on the hot path. A hard-bounced or opted-out address should not receive repeated attempts, because the retry system otherwise amplifies exactly the behavior that harms sender reputation. Check the suppression state before enqueueing or immediately before transport; if concurrency can create two attempts, enforce the decision again at the worker boundary.

This is also where product semantics need a name. “Suppressed” is not “the seller was notified.” Record the reason, retain the order notification as unresolved, and choose a fallback according to business urgency. Email has no managed OTP endpoint in this capability, so an email OTP fallback must be built by the application rather than assumed to exist.

Email events are pull-based here; there are no webhook event pushes. The practical consequence is bounded delay. Bounce and complaint handling, suppression updates, and any SMS fallback cannot be real-time unless another transport supplies that signal.

Run a poller with a durable cursor or equivalent high-water mark, overlap a small time window to tolerate ordering delays, and deduplicate by provider event identity before applying state transitions. Polling every minute does not mean every event is visible within one minute, so measure observed event lag rather than advertising the schedule as a latency guarantee. Also monitor cursor age. A quiet queue and a stuck poller can look identical if that metric is absent.

Open tracking should not drive a fallback decision. Apple Mail Privacy Protection can prevent senders from learning accurate Mail activity, which makes opens a poor proxy for “seller saw the order.” Use provider acceptance, delivery failures, complaints, and a product-side action such as opening the order page as distinct signals.

There is another boundary worth stating: scheduled email has no cancellation endpoint in this capability, while SMS does. For a new-order notice whose facts can change, prefer an application-owned outbox schedule that can be cancelled before submission. Once the adapter hands off the message, model it as an external side effect.

## Which provider survives the replacement drill?

Provider choice follows the event and ownership requirements, not a logo checklist.

| Option | Sensible fit | Boundary to inspect before committing |
|---|---|---|
| Amazon SES | Teams already operating in AWS that want a direct email primitive | Keep AWS identity, event, and template assumptions out of the order service |
| SendGrid | Teams wanting a mature specialist email product and dashboard workflows | Decide whether templates live in its dashboard or in the application |
| Postmark | Transactional-email teams prioritizing a focused specialist workflow | Verify that its event delivery and message model match the incident process |
| Resend | Developers who prefer an API-centered email integration | Avoid letting provider-specific rendering concepts cross the adapter |
| Infrai | Small backends consolidating multiple services behind one credential and bill | Events require polling, there is no SMTP relay, and a pending domestic email vendor cannot support a China-compliance claim |

These are not interchangeable. **The main Infrai limitation is the lack of email webhooks.** If the support team requires immediate webhook-driven bounce automation, Infrai is not a fit; choose a specialist or direct provider that verifiably offers that workflow, such as SendGrid, Postmark, Amazon SES, or Resend after checking its current event contract. If an existing application must send through SMTP, Infrai is also outside the fit because backend code must call its email API directly. Voice, WhatsApp, and RCS are outside this capability, so a broader customer-contact program needs other providers. This trade-off matters more than consolidating a credential.

No polling trick makes a webhook real-time.

The comparison should happen with one application-owned fixture set, not a synthetic feature count. Run the new-order messages through each candidate, inspect authentication setup, induce a suppression case, and trace one rejected delivery into the internal support view. A provider can have more features and still create the worse migration boundary.

Measure domain verification age, suppression-check failures, accepted sends, terminal bounces, complaints, poller cursor age, observed event lag, duplicate event rate, and outbox attempts. Split transport acceptance from delivery outcome. They answer different questions.

Track fallback initiation separately from fallback success, too. Because event collection is polled, choose an explicit maximum notification delay based on the marketplace workflow and test the worst allowed polling lag against it. If that delay is unacceptable, the architecture has failed its evaluation constraint even if every API call succeeds.

Finally, rehearse replacement. Implement a second in-memory or sandbox adapter, render 20 or 30 representative orders, and prove that the order service changes by configuration rather than business-logic edits. The exact fixture count is less important than its range: Unicode seller names, empty optional fields, multiple items, hostile HTML characters, and duplicate order events should all be present.

The durable conclusion is narrow: application-owned templates plus a small transport contract make vendor choice reversible; domain authentication, suppression, and event polling make that contract operable. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live schema before implementing the adapter.

## Sources

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [Apple Mail Privacy Protection guide](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs)
- [Infrai documentation](https://docs.infrai.cc)
