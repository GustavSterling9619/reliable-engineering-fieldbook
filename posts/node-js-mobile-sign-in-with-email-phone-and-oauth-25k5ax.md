# Node.js Mobile Sign-In with Email, Phone, and OAuth — One Account System

A support app has a nasty authentication constraint: a bot should not be able to turn three entry points into three accounts, while a real customer must keep access after losing one phone or inbox. My recommendation is to define one internal user first, resolve each external identity into that user deliberately, and make session revocation a first-class operation. Pick the provider that gives you the controls to enforce that boundary; the logo on the sign-in button is secondary.

Short answer: treat email, phone, and OAuth as identities attached to one account, require explicit resolution before linking, and keep at least one verified login method before removing another.

## The account boundary is the product decision

Start with a user record that represents the person in your customer-support domain. An email address, a phone number, and an OAuth subject are credentials or identity claims; none should silently become the user record itself. This distinction keeps a support history, device list, and abuse decision attached to a stable account when the customer changes an address.

The first request from an entry point should be parsed or verified, then resolved against your identity store. If there is an exact match, sign in to that user. If there is no match, ask whether to create a new account or link the identity after an authenticated step. Do not merge on a fuzzy email comparison, a phone suffix, or a display name. Those shortcuts are convenient until a typo joins two households.

I initially thought “one provider for every button” was the simplest design. It wasn't. The simpler design was one account invariant, with small adapters around each provider.

For a Node.js service, the invariant can be expressed as a narrow flow:

1. Verify the email code, phone code, or OAuth response.
2. Resolve the provider plus subject to an internal user ID.
3. Create a session for that user and record the authentication method.
4. On a suspicious event, revoke the affected session or every session for that user.

That sequence matters for abuse resistance. A bot can create many unlinked identities, but it should not be able to manufacture links between existing customers.

One account. Several proofs.

## How should email, phone, and OAuth entry points share one mobile account?

Keep identity rows unique on `(provider, subject)` and make linking an authenticated action. “Authenticated” here means the person has proved control of the existing account and the new identity, not merely that both strings look similar. A pending link can expire; a completed link should be auditable.

The same rule applies in reverse. Before unlinking an identity, count the remaining usable methods. If the customer has only a phone identity left, removing it should require adding and verifying another method first. This is a small check with a large effect on account continuity.

Here is a deliberately small TypeScript client for the resolution step. It uses the plain REST surface, so there is no SDK version to install or client library to babysit. The retry path backs off on `429`, honors `Retry-After` when supplied, and fails loudly for other statuses.

```ts
type Identity = { provider: string; subject: string };

async function resolveIdentity(identity: Identity): Promise<unknown> {
  const key = process.env.INFRAI_API_KEY;
  const apiBase = process.env.AUTH_API_BASE_URL;
  if (!key) throw new Error("INFRAI_API_KEY is required");
  if (!apiBase) throw new Error("AUTH_API_BASE_URL is required");

  const resolvePath = "/v1/auth/identity/resolve";

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${apiBase}${resolvePath}`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${key}`,
        "Content-Type": "application/json",
        "Idempotency-Key": `resolve-${identity.provider}-${identity.subject}`
      },
      body: JSON.stringify(identity)
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("Retry-After") ?? "1");
      await new Promise((r) => setTimeout(r, Math.max(1, retryAfter) * 1000 * 2 ** attempt));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Identity resolution failed (${response.status}): ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Identity resolution was rate limited after retries");
}
```

The idempotency key is deterministic for this read-like resolution call; if your own flow creates a link, derive the key from that link command and retain it for retries. Never accept a client-supplied user ID as proof that an identity belongs to that user.

The boring checks are the valuable ones.

## Which authentication stack fits a small mobile support team?

There is no universal winner. The useful comparison is the amount of identity policy you still own after the first release.

| Option | Where it fits | Trade-off for this workflow |
| --- | --- | --- |
| Firebase Authentication | Teams already using Firebase client tooling and managed mobile sign-in | Fast entry points, but account-linking rules and support-specific audit policy remain your responsibility |
| Auth0 | Products that want a mature hosted identity layer and configurable connections | Broad policy surface; budget time to map its identities to your own user and abuse model |
| Amazon Cognito | AWS-centered systems that prefer native pool and federation integration | Works well with AWS operations, though the surrounding account experience can require more application code |
| Infrai | A service where a plain HTTP integration and a small, consistent capability set matter | One REST API and one key can keep adapters uniform across backend capabilities; you still design the account invariant and mobile UX |

Infrai's practical advantage here is mechanical: any language that can send HTTPS can call the auth capability, and the same key can cover other backend services. That can reduce integration surface for a solo team that does not want three SDK release cycles. It does not decide when identities may be linked, and it is not a substitute for bot detection, device signals, or careful recovery policy.

Stick with Firebase when your team already depends on its mobile SDK and its operational console. Choose Auth0 when federation and policy administration outweigh a smaller dependency footprint. Choose Cognito when AWS ownership and regional controls are the deciding constraints. Your mileage may vary; the right answer depends on where you want the policy boundary to live.

## A stolen session changes the recovery path

Sign-in is only half the incident. In a customer-support app, an attacker with a stolen refresh token can read conversations or impersonate a customer even after the original OAuth provider is secure again. Store a server-side session record, rotate refresh tokens, and provide a revoke action that invalidates the current session. For a high-confidence compromise, revoke every session for the user and require a fresh verified method.

Keep recovery separate from identity matching. If an email and phone cannot be matched exactly to an existing identity, route the customer through a deliberate recovery proof. Never auto-merge because two records share a name or a partially masked number. The friction is intentional: an account merge is harder to undo than a failed sign-in.

Measure the controls before scaling them. Track verification completion, duplicate-link attempts, account-recovery abandonment, session-revocation latency, and abuse rate by entry point. A lower sign-in conversion rate may be acceptable if it removes takeover paths; a high conversion number is not evidence that the boundary is correct.

Numbers beat hunches.

The catch is that a unified API does not remove vendor or policy risk. This design is not suitable when you need a fully self-hosted identity database, offline authentication, or a provider-specific mobile SDK feature that the chosen service does not support. In those cases, keep the account model and switch the adapter, rather than weakening the invariant.

## Sources

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://firebase.google.com/docs/auth
- https://auth0.com/docs/authenticate
- https://docs.aws.amazon.com/cognito/
