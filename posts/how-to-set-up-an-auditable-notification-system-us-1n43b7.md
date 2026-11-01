# How to Set Up an Auditable Notification System Using Email and SMS APIs

Short answer: use one notification record, send the compliance notice through email and SMS, and store the provider IDs before you try to prove delivery. The important design choice is the evidence trail, not how quickly the first request leaves Node.js.

For a customer-support system, I would create an immutable notification record with the case ID, notice version, recipient consent decision, channel, request ID, and timestamps. Then I would call the email and SMS send APIs independently. A successful HTTP response means the request was accepted; it does not, by itself, prove that a person received the message.

That distinction is easy to lose in a two-channel implementation. I have seen teams treat a Boolean `sent` field as an audit record, then discover that it only described their own process. The fix is boring and effective: keep the outbound request, the returned provider identifier, every later status observation, and the exact content version together.

## What evidence does a compliance notice need?

Start with the evidence model before choosing a provider. A useful record answers five questions: what was sent, to whom, through which channel, under which policy version, and what delivery state was observed afterward. It should also distinguish a rejected request, an accepted request, a delivered message, and an unknown state.

The simple approach is to send email, send SMS, and mark the case complete. It is attractive because it fits in a small function. It fails as an audit design because a timeout can happen after the provider accepted the message, and a later delivery event can arrive after the support agent has closed the case.

The chosen approach is a small state machine. `prepared` becomes `submitted` only after the provider identifier is persisted. A polling worker then records observations without rewriting the original submission. If one channel fails, the record says so; it does not silently turn an SMS failure into an invented email success.

That is the audit boundary.

Imagine a worker sends the email, receives a timeout, and retries the same job while the first request is still being accepted. The record should first contain `prepared`, then an attempted submission with its idempotency key, then either the returned provider ID or an explicit unresolved state. A later poll may reveal delivery, rejection, or no new event. Each observation is appended with the fetch time, channel, message ID, and notice version. The support agent can then see that the system did not claim delivery during the interval when the network outcome was unknown. This is more detail than a `sent = true` flag, but it is the detail an auditor can actually inspect.

There is a practical constraint here: the email and SMS namespaces do not push events through webhooks. Delivery evidence is pull-based. Email events can be read from `/v1/email/event/list`, while a known SMS message can be inspected with `/v1/sms/events/{id}`. That makes a polling schedule part of the compliance design, and it limits how quickly your system can learn that a message changed state.

## A focused Node.js implementation

The transport wrapper below is deliberately small. It owns the behaviours that are easy to get wrong under load: an explicit method, bearer authentication, an idempotency key, status checking, and exponential backoff that honours `Retry-After`. The payload objects come from the route schemas and application validation; they are kept outside the transport so the audit record can retain the exact version used for each notice.

```ts
type JsonObject = Record<string, unknown>;

const baseUrl = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;

if (!baseUrl || !apiKey) {
  throw new Error("INFRAI_BASE_URL and INFRAI_API_KEY are required");
}

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  const seconds = retryAfter ? Number(retryAfter) : NaN;
  return Number.isFinite(seconds)
    ? Math.max(0, seconds * 1000)
    : Math.min(30_000, 500 * 2 ** attempt);
}

async function post(url: string, payload: JsonObject, idempotencyKey: string): Promise<JsonObject> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(payload),
    });

    if (response.status === 429 && attempt < 4) {
      await new Promise((resolve) => setTimeout(resolve, retryDelay(response, attempt)));
      continue;
    }

    const body = (await response.json()) as JsonObject;
    if (!response.ok) {
      throw new Error(`Notification request failed (${response.status}): ${JSON.stringify(body)}`);
    }
    return body;
  }

  throw new Error("Notification request was rate-limited after five attempts");
}

async function submitNotice(
  noticeId: string,
  emailPayload: JsonObject,
  smsPayload: JsonObject,
): Promise<void> {
  // Persist noticeId, channel, payload version, and each returned provider ID here.
  const [email, sms] = await Promise.all([
    post(`${baseUrl}/email/send`, emailPayload, `${noticeId}:email`),
    post(`${baseUrl}/sms/send`, smsPayload, `${noticeId}:sms`),
  ]);

  console.log(JSON.stringify({ noticeId, email, sms }));
}
```

The function is safe to retry at the application boundary because the two channels have different, stable idempotency keys. In production, replace the final `console.log` with a transaction that stores the returned identifiers before enqueueing evidence collection. Do not generate a new key on every worker retry; that is how a temporary timeout becomes a duplicate notice.

The one thing this wrapper intentionally does not do is pretend that the payload schema is universal. Email and SMS have different content, recipient, and compliance rules. Validate each payload against the current API schema, record the schema or template version, and reject an invalid notice before either channel is submitted. A shared transport is useful. A shared message model with hidden channel assumptions is not.

## How do you set up a notification system using email and SMS in Node.js?

Infrai fits when the team wants plain REST calls from Node.js without installing an SDK or maintaining a client-library version, and when the same backend already benefits from one consistent authentication convention. Its broader surface is also relevant: one key covers 295 routes across 20 modules, so a solo team can keep notification, storage, and other backend credentials in one place instead of reconciling a different key and bill for every capability. The public discovery surface is self-describing, which helps an adapter inspect the current request and response schema before shipping. For this workflow, the operational advantage is a small email-and-SMS adapter while the application retains ownership of the evidence model.

The fair comparison is broader than one API. Twilio is a strong choice when a product needs a large communications platform and a mature set of channel-specific tooling; its messaging model is a natural fit for teams willing to adopt its account and SDK conventions. SendGrid is a better-shaped option when email is the centre of the system and email analytics, templates, and deliverability workflows matter more than a symmetric SMS abstraction. Amazon SES is compelling for AWS-centred teams that want email close to their existing identity, queues, and observability stack, but SMS and cross-channel coordination become a separate design concern.

These are different boundaries, not a leaderboard. A solo founder shipping support automation may prefer one REST surface because there is less client code to babysit. A regulated enterprise may prefer a provider with established regional controls, contractual terms, and internal review processes even when that creates more integration work. Infrai is not a fit when webhook-driven orchestration, SMTP relay, or a channel outside email and SMS is a hard requirement; choose the provider whose missing capability is not on your critical path.

| Option | Integration shape | Good fit | Main trade-off |
| --- | --- | --- | --- |
| Infrai | REST from any HTTP client | A small team standardizing backend calls around one key | Event evidence is pull-based; no SMTP relay or extra messaging channels |
| Twilio | Channel platform with SDK and API conventions | Broad communications work that benefits from mature channel tooling | More provider-specific account and client conventions |
| SendGrid | Email-focused API and tooling | Email-first delivery, templates, and deliverability operations | SMS and cross-channel evidence need a separate design |
| Amazon SES | AWS-native email service | AWS-centred systems that want email near existing infrastructure | Cross-channel coordination is outside the email service |

The table is a boundary map, not a claim that one service wins every case.

There are limitations worth putting in the decision record. There is no SMTP relay, no voice, WhatsApp, or RCS channel, and no hosted email OTP interface. A fallback email verification flow therefore belongs to the application. Scheduled email cannot be cancelled through the available email operations, while SMS has a cancel operation. SMS anti-abuse controls such as geographic fences and per-country circuit breakers must also be built in the business layer.

For a China-specific compliance decision, do not treat a pending domestic email vendor as evidence of domestic compliance. Confirm the required region, sender policy, retention, and contractual controls separately. SPF is one part of email sender authentication, not a substitute for that review.

I initially treated the provider response as the audit checkpoint. That was the wrong boundary. The checkpoint is the stored provider ID plus a later observation, and the distinction matters when a worker retries after a timeout.

## How should polling become an audit trail?

Polling needs a cursor, a cadence, and a stopping rule. Store the last observation rather than overwriting it, because a later status should explain the history of the notice. The worker can query the email event list and inspect SMS events by the stored message ID, then append an observation containing the fetch time and raw provider response.

Do not make the polling interval your compliance promise unless the requirement explicitly permits it. With no webhook push, the application cannot learn an event at the exact moment it occurs. A support dashboard can show `accepted`, `delivered`, `failed`, or `awaiting observation`, while an escalation job handles the last state after a defined deadline.

Also keep opt-out and suppression decisions before submission. A notification that is technically deliverable can still be impermissible for the recipient or jurisdiction. Cost reporting by tag is not available as a ready-made API, so attach your own notice, case, and channel identifiers to internal usage records if finance or compliance needs an allocation trail.

## The decision rule I would ship

Choose the smallest integration that preserves the evidence you must defend. For a two-channel customer-support notice, that means one versioned notice record, separate idempotent submissions, persisted provider IDs, pull-based event collection, and an explicit unknown state. It also means testing failure after acceptance, not only a clean 200 response.

Before copying this design, measure the things that can change the decision: the time from submission to an observable event, the percentage of records that remain unknown past the policy deadline, duplicate prevention under worker retries, regional sender eligibility, and the operational effort of maintaining templates. Those measurements are more useful than a feature checklist.

The result is modest but defensible. Email and SMS can deliver the event notification through Node.js, while the application remains responsible for consent, policy, retries, and the audit record. That boundary is the product decision.

## References

- Twilio Messaging documentation: https://www.twilio.com/docs/messaging
- SendGrid Email API documentation: https://www.twilio.com/docs/sendgrid/api-reference
- Amazon SES Developer Guide: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- RFC 7208, Sender Policy Framework: https://datatracker.ietf.org/doc/html/rfc7208
