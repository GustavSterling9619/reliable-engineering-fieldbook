# Tenant-Safe App Data: Cheap Compatible Storage, Daily Retention, Automated Lifecycle Rules

Short answer: use a private bucket boundary per tenant or isolation class, record an immutable deletion deadline with every signed document, and treat 30- and 90-day lifecycle rules as cleanup backstops for daily app backups, not as the source of retention truth.

The deciding constraint is tenant isolation. A low storage rate doesn't rescue a design that can restore one merchant's signed return authorization into another merchant's account, or delete the live agreement while leaving copies in daily backups with no auditable expiry. For a solo team, the practical target is boring: one policy compiler, one key convention, and a restore test that tries the wrong tenant as aggressively as the right one.

This is an experiment note, not a provider ranking. The simple design was one shared prefix tree plus a 30-day expiration rule. It failed on paper before deployment because it collapsed three separate questions: when the signed document must be deleted, how long a backup containing it may remain, and which tenant is allowed to restore it. The replacement keeps those decisions explicit.

## How should automated daily app backups enforce 30- and 90-day retention across US and EU tenants?

Start with a deletion deadline on the document record. Then derive storage placement and backup expiry from policy. A signed fulfillment agreement might belong to an EU tenant, carry its own `deleteAt` timestamp, appear in a daily backup class retained for 30 days, and have a narrow audit export retained for 90 days. Those are related facts, but they aren't interchangeable. The document deadline answers a business requirement. The lifecycle rule removes objects from a storage namespace. The backup window limits how long an old database or manifest can reintroduce data after the live object is gone.

The key layout should expose the isolation boundary before it exposes the date:

```ts
type Region = "us" | "eu";
type RetentionClass = "daily-30" | "audit-90";

type SignedDocument = {
  tenantId: string;
  documentId: string;
  region: Region;
  deleteAt: string;
  retentionClass: RetentionClass;
};

function documentKey(document: SignedDocument): string {
  return [
    "tenants",
    encodeURIComponent(document.tenantId),
    document.region,
    "signed-documents",
    `${encodeURIComponent(document.documentId)}.pdf`,
  ].join("/");
}

function backupKey(
  tenantId: string,
  region: Region,
  retentionClass: RetentionClass,
  createdAt: Date,
): string {
  const day = createdAt.toISOString().slice(0, 10);
  return [
    "tenants",
    encodeURIComponent(tenantId),
    region,
    "backups",
    retentionClass,
    `${day}.jsonl.gz`,
  ].join("/");
}
```

The function is intentionally dull. It makes tenant and region reviewable in code, and it gives a lifecycle policy a narrow prefix such as `tenants/acme/eu/backups/daily-30/`. Don't accept arbitrary object keys from a browser and prepend a tenant later; construct the entire key after authentication. A presigned URL can grant time-limited access to a specific object, but the signing service still has to authorize the tenant, object, operation, and expiry before it creates that URL. The URL is a delivery mechanism, not an access-control decision.

Region belongs in the placement decision too. “US/EU supported” is too vague for a retention design. The application should know where each tenant's object and backup are written, reject a restore into a conflicting region, and log the policy version that allowed the operation. I'm not sure a provider's region label alone satisfies any particular contract; the contract, data-flow inventory, and provider terms are what resolve that question.

## Compile deadlines instead of scattering delete jobs

The dangerous shortcut is a daily script that lists everything older than 30 days and deletes it. It looks ship-first. It also embeds a global retention assumption into an operational loop, so a new 90-day class, a paused legal deletion, or a tenant migration can turn a tiny conditional into policy archaeology. One missed run is visible, but a subtly wrong filter can execute successfully.

Use a small policy compiler instead. It should validate that the requested class exists, calculate the expected backup expiration date, and emit both a storage-rule specification and a test fixture. Keep live-document deletion separate because `deleteAt` can be more precise than a backup window and can vary by document. The application worker deletes due live documents; bucket lifecycle handles expired backup objects. A reconciliation job compares intent with inventory and raises a signal when an object remains past its deadline.

```ts
const retentionDays = {
  "daily-30": 30,
  "audit-90": 90,
} as const;

type RetentionClass = keyof typeof retentionDays;

type ExpiryIntent = {
  prefix: string;
  expireAfterDays: number;
  policyVersion: string;
};

function compileExpiryIntent(
  tenantId: string,
  region: "us" | "eu",
  retentionClass: RetentionClass,
): ExpiryIntent {
  if (!tenantId.trim()) throw new Error("tenantId is required");

  return {
    prefix: `tenants/${encodeURIComponent(tenantId)}/${region}/backups/${retentionClass}/`,
    expireAfterDays: retentionDays[retentionClass],
    policyVersion: "signed-documents-v1",
  };
}
```

That object is not a pretend vendor request. An adapter must translate it into the chosen service's documented lifecycle schema, and a read-back check must compare the installed rule with the compiled intent. This boundary matters for S3-compatible storage because “compatible” should be verified against the exact operations the app uses rather than treated as a blanket promise. For this workload, the compatibility test suite needs object upload, authenticated download, listing by the chosen prefix, deletion, lifecycle configuration, and presigned access. If the adapter cannot read back lifecycle state, don't silently call configuration successful.

Be strict here.

## Tenant isolation is tested during restore, not upload

Uploads mostly prove credentials and network paths. Restore drills prove whether the system preserved ownership. For every release of the backup worker or authorization layer, create two synthetic tenants in each deployed region, write distinct signed-document manifests, and attempt four paths: same-tenant restore, cross-tenant restore, same-region recovery, and a region-conflicting recovery. The two disallowed paths must fail before any object bytes reach the caller. Log a stable request identifier, tenant identifier, region, object-key hash, policy version, and authorization result. Don't log the signed document or a reusable presigned URL.

The longest test should cover a realistic failure sequence. Imagine tenant A and tenant B both have `agreement-1042.pdf`. A support operator requests tenant A's restore, the job retries after a timeout, and the queue delivers the request twice. The restore worker must derive the object key from the authenticated tenant each time, write into a tenant-scoped destination, and make the operation idempotent. Then the test swaps only the requested tenant while keeping the document ID. If lookup is keyed by document ID alone, the suite has found the isolation bug before a customer does. This case is more valuable than another happy-path upload test because retries, repeated identifiers, and support tooling are where a clean namespace often gets bypassed.

For deletion, inject a due document and an adjacent document whose deadline is still in the future. Run the worker twice. The due object should remain absent, the future object should remain readable to its own tenant, and neither should be readable by the other synthetic tenant. For backup expiry, inspect lifecycle configuration and inventory around the 30- and 90-day boundaries. The exact timing semantics must come from the selected service's current documentation and should be encoded in adapter tests; don't infer them from the S3 label.

A monthly restore drill is the minimum operational habit I would budget for a small app, but the cadence is a design choice, not a universal standard. Increase it when authorization, backup format, encryption keys, or region placement changes. Your mileage may vary with document volume and recovery objectives. What matters is that the test produces evidence: which policy version was exercised, which tenant boundary was challenged, how much data was restored, and whether the restored artifact was usable.

## Cost follows the restore and isolation model

“Cheap storage” is an incomplete requirement. Model stored bytes over both retention windows, daily writes, lifecycle operations, inventory or listing work, presigned access, restore downloads, and the duplicate copies required by the chosen tenant and region boundaries. A shared bucket may reduce bucket administration while demanding stricter prefix authorization. Per-tenant buckets can make blast radius and policy ownership easier to see, but may create more configuration objects and more reconciliation work. Neither layout wins without the expected tenant count and the service's documented limits.

Use a spreadsheet or a tiny deterministic model fed by the provider's current pricing page. Price the normal month and a restore month separately. The restore month matters because signed documents that are never tested are only assumed recoverable, and download or request charges can move differently from storage charges. Don't publish a savings percentage from a sample workload; it won't survive a change in object size, tenant count, or recovery frequency.

The operational comparison is compact:

| Decision | Lower-complexity choice | Catch | Evidence before launch |
| --- | --- | --- | --- |
| Isolation | Shared bucket with tenant-first prefixes | Every list, sign, delete, and restore path must enforce the prefix | Cross-tenant negative tests |
| Policy | Separate 30-day and 90-day backup prefixes | More lifecycle rules to reconcile | Installed-rule read-back |
| Access | Short-lived presigned access | A leaked URL remains usable until its expiry | Authorization and expiry tests |
| Region | Explicit US or EU placement per tenant | Moving data becomes a governed operation | Region-conflict restore test |
| Cost | Model bytes, operations, and restore traffic | Headline storage rate omits workflow costs | Normal-month and drill-month estimates |

Price comes last. First prove that the design can deny the wrong tenant and erase the right copy on schedule.

## Where does this design stop fitting?

The catch is that lifecycle expiry is cleanup automation, not proof of immutable retention or legal disposition. This design is not suitable when signed documents require write-once enforcement, a formal legal hold, independently certified destruction, or recovery that spans a regional outage. Choose storage and governance controls explicitly documented for those requirements, and have the retention owner review the evidence rather than stretching a 90-day rule into a compliance claim.

It also stops fitting when each document needs a unique deadline but the only deletion mechanism is a coarse prefix rule. Keep the per-document deletion worker and reconciliation evidence, or select a storage design whose documented expiry controls match that precision. Likewise, stick with a provider-specific client when the application depends on semantics outside the tested compatibility subset. Portability is useful only when the adapter contract stays smaller than the application.

Measure before copying this choice: cross-tenant denial rate in automated tests, overdue-object count, lifecycle-policy drift, restore completion time, restored-document validity, stored bytes by class and region, and the full cost of a scheduled drill. Those numbers reveal whether 30/90-day daily backup retention is working. A logo comparison won't.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- https://www.backblaze.com/cloud-storage/pricing
