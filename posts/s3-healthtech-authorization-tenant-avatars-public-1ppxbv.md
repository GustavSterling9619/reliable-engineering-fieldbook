# S3 Healthtech Authorization: Tenant Avatars, Public URLs, and Signed Download Expiry

Short answer: an avatar's public URL won't make a private object readable, while a signed download URL is temporary authorization and expires by design. For a tenant-isolated healthtech system, persist an internal file ID and lifecycle metadata, authorize the tenant on each resolution, and mint a new short-lived grant only after that check.

The evaluation constraint is tenant isolation. Imagine one storage account holding replaceable staff avatars alongside signed patient documents that have explicit deletion deadlines. The quick implementation stores the URL returned after upload in a profile row. It renders during a smoke test, so the URL looks like durable file identity. It isn't.

There are actually three different things hiding behind that one string: the identity of the object, permission to read it, and an address the browser can request. A private origin URL supplies an address but no permission. A signed URL supplies an address plus time-bounded permission. Neither is a sound permanent identifier for a database record.

No magic here.

The experiment should therefore compare access decisions, not ask whether an image rendered once. The useful result is that same-tenant access survives grant renewal, cross-tenant access remains denied, and a document becomes inaccessible at its deletion deadline even when cleanup runs separately.

## Two clocks explain the failure better than one URL

Signed download expiry and object deletion are independent clocks. The first clock limits one authorization grant. When it runs out, the underlying bytes may still exist and an authorized caller may request another grant. The second clock belongs to retention policy. When `deleteAt` arrives, the application must stop creating grants, and storage cleanup removes the bytes according to that policy.

This distinction matters in a healthtech upload flow because avatars and signed documents often have different lifetimes even when they use the same bucket abstraction. An avatar might be replaced while its prior grant is still cached. A signed document might reach its deletion deadline while an old application page still holds its file ID. Treating URL expiry as deletion would leave retention ambiguous; treating deletion as merely another expired link could allow a new grant after the policy deadline.

The simple approach fails in both directions. Saving a signed URL as `avatarUrl` makes an authorization artifact look permanent, so a later page load sees a broken image after the signature expires. Saving a bare private-origin URL avoids signature expiry but still doesn't authorize the browser. Making every object public fixes display by removing the access boundary, which is the wrong result when tenant isolation is the deciding constraint.

Short links aren't deletion controls.

A better record has durable identity and explicit policy: `fileId`, `tenantId`, `objectKey`, `purpose`, `version`, and `deleteAt`. The object key locates bytes inside the storage layer. The file ID is what application code exposes. The version separates a current avatar from one it replaced. None of those fields grants a download by itself.

## How should an avatar image public URL work with private object storage after a signed download URL expires?

It should work as a stable application action, not as a permanent storage grant. The browser asks the application for the current avatar using an internal profile or file identifier. The application authenticates the caller, resolves the trusted file record, checks that the caller and file belong to the same tenant, verifies that lifecycle policy still permits access, and then returns or redirects to a newly signed download URL.

That extra resolution step is deliberate. A client must not send an arbitrary object key and ask the server to sign it, because storage prefixes are organization, not authorization. The server derives the key from trusted metadata after the tenant decision. A missing record and a tenant mismatch can share the same outward `File not found` response so the endpoint doesn't confirm that another tenant's file exists.

CORS sits on another layer. It controls whether browser code may access a cross-origin response; it doesn't turn a private object into an authorized one. An `<img>` request and a scripted cross-origin `fetch` also aren't interchangeable browser tests. First establish that the request has valid object authorization. Then investigate CORS if authorized JavaScript access is blocked. Changing CORS while using an unsigned private URL only moves attention away from the missing permission.

The delivery choices have concrete trade-offs:

| Delivery shape | Where access is decided | Durable application value | Suitable when | Limitation |
| --- | --- | --- | --- | --- |
| Public object | When the object is published | Object key or public address | The avatar is intentionally world-readable | No per-request tenant authorization |
| Application proxy | On every download request | Internal file ID | Policy must control every response or transform bytes | Application carries file traffic |
| Signed redirect | Before each temporary grant | Internal file ID and object key | Bytes stay private but storage serves the download | Clients must renew expired grants |

The catch is real: private signed delivery adds an application lookup and a signing operation. It is not suitable when the actual requirement is a permanently reusable, broadly cacheable public image and the data classification allows public access. In that case, keep the avatar public. Keep signed clinical documents on the private path because presentation data and retained records should not inherit one another's policy by convenience.

## A focused TypeScript boundary keeps durable state separate from grants

The following interface makes the ordering visible without binding application records to a particular storage product. The example lifetimes are policy inputs, not universal recommendations; the right values depend on request duration, client retry behavior, and the exposure cost of a copied link. I'm not sure one lifetime should cover both mobile avatar rendering and document download, and production measurements should decide that rather than a tidy constant shared everywhere.

```ts
type FilePurpose = "avatar" | "signed-document";

type StoredFile = {
  id: string;
  tenantId: string;
  objectKey: string;
  purpose: FilePurpose;
  version: number;
  deleteAt: Date | null;
};

type Actor = {
  tenantId: string;
  userId: string;
};

type DownloadGrant = {
  url: string;
  expiresAt: string;
};

interface FileRepository {
  findById(fileId: string): Promise<StoredFile | null>;
}

interface ObjectSigner {
  signDownload(objectKey: string, expiresInSeconds: number): Promise<string>;
}

async function createDownloadGrant(
  actor: Actor,
  fileId: string,
  files: FileRepository,
  signer: ObjectSigner,
  now: Date,
): Promise<DownloadGrant> {
  const file = await files.findById(fileId);

  if (!file || file.tenantId !== actor.tenantId) {
    throw new Error("File not found");
  }

  if (file.deleteAt && file.deleteAt.getTime() <= now.getTime()) {
    throw new Error("File is past its deletion deadline");
  }

  const expiresInSeconds = file.purpose === "avatar" ? 900 : 300;
  const url = await signer.signDownload(file.objectKey, expiresInSeconds);

  return {
    url,
    expiresAt: new Date(now.getTime() + expiresInSeconds * 1000).toISOString(),
  };
}
```

The order carries more weight than the interface names. Tenant comparison happens before the object key reaches the signer. The deletion deadline is checked before a fresh grant is created. The caller receives `expiresAt`, so it can distinguish a renewable authorization timeout from a missing avatar instead of persisting the temporary URL again.

This also keeps the client recovery path small. On expiry, request another grant through the stable application identifier. On avatar replacement, resolve the newest version rather than guessing whether a cached storage address still points at current bytes. Don't log signed URLs: they remain credentials until their expiry, and ordinary request logs are a poor place to copy them.

## Deletion should be tested as a state transition

An explicit deadline needs an application state change, not just a cleanup schedule. Once `deleteAt` has passed, reads stop immediately at the authorization boundary. An idempotent worker can remove the underlying object and mark the record deleted afterward. That separation prevents worker timing from extending application access, while still allowing deletion work to be retried without turning a temporary operational delay into a new authorization policy.

Model at least active, deletion-due, and deleted states. The denial matrix should cover a same-tenant actor reading an active file, a cross-tenant actor receiving the same outward result as a nonexistent record, and every actor being denied a fresh grant once the deadline is reached. For avatar replacement, verify that the stable application lookup resolves the latest version and that an old signed grant is never mistaken for the current database value.

One failure deserves a focused regression test: set `now` exactly equal to `deleteAt`. The comparison must deny access at the boundary, not one millisecond later. That concrete edge catches the easy `>` versus `>=` mistake without pretending that a made-up production incident proved it.

Browser tests should split rendering from scripted access. Load the image through its intended element, then separately exercise any cross-origin JavaScript request that the UI genuinely needs. If the signed request is authorized but the scripted response is unavailable to JavaScript, inspect the origin's CORS policy. If the signature has expired, refresh the grant. Those diagnoses call for different fixes.

## Measure the boundary before copying this design

Measure grant creation latency separately from byte-download latency, along with grant age when a download starts, refresh frequency, denials by tenant policy, stale-avatar resolutions after replacement, and deletion lag. A single “image failed” counter collapses authorization, CORS, expiry, cache state, and missing objects into noise. Cost belongs in the evaluation too: count application lookups, signing operations, storage reads, and outbound bytes under realistic cache behavior rather than assuming that direct delivery is free or that proxying is automatically unaffordable.

For a solo team, concentrating policy in one TypeScript boundary reduces the number of places that must understand storage credentials. Your mileage may vary with file size, client retries, regional placement, and the amount of public caching the privacy model permits.

The application hop still costs time.

Ship only after the denial matrix passes. Then watch expiry refreshes, cross-tenant denials, deletion lag, and grant latency with production-shaped traffic. A signed URL proves that one temporary grant exists; it does not prove that tenant isolation or retention is correct.

## References

- [MDN: Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS)
- [Cloudflare R2 documentation](https://developers.cloudflare.com/r2/)

## Further reading

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
- https://developers.cloudflare.com/r2/
