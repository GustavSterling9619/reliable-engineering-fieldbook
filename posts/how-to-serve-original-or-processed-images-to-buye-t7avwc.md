# How to Serve Original or Processed Images to Buyers (API Trade-Offs)

TL;DR: Keep one private, immutable original for each shipment photo. Before purchase, return a short-lived URL for a processed rendition sized to the buyer's current surface; after purchase, return a separate short-lived URL for the original only when the authenticated buyer is entitled to that asset. Generate common crops once, name them by content and transformation, and let shared caches absorb repeat reads. Generate rare sizes on demand, but put a hard ceiling on the accepted dimensions.

That rule fits a logistics marketplace where a creator supplies loading-bay, pallet, or proof-of-condition photos and buyers view them in square search tiles, 4:3 detail pages, and wide shipment summaries. The API is deciding which asset a buyer may receive. The image worker is deciding how a permitted rendition is produced. Combining those decisions makes authorization hard to audit and turns every new aspect ratio into a storage surprise.

The useful unit is not "an image URL." It is an entitlement plus a deterministic rendition request. Purchase state can change; pixels do not need to.

## Should the API serve buyers an original or processed image?

Return metadata and an expiring delivery URL, not image bytes embedded in the purchase response. The response should identify whether the delivered asset is an original or a rendition, include its dimensions and media type, and avoid exposing a permanent storage key. A buyer who has not purchased the photo gets a presentation crop. A buyer with an active entitlement may explicitly request the original.

Here is the whole decision path in TypeScript. It has no vendor SDK and can run with `npx tsx delivery.ts`. The sample signer is deliberately local: production code should replace it with the signing operation supported by the chosen object store or delivery tier.

```ts
type Asset = {
  id: string;
  originalKey: string;
  contentHash: string;
  width: number;
  height: number;
  mediaType: "image/jpeg" | "image/png" | "image/webp";
};

type DeliveryRequest = {
  buyerId: string;
  assetId: string;
  purchased: boolean;
  mode: "preview" | "original";
  crop?: { width: number; height: number; focalX: number; focalY: number };
};

type Delivery = {
  kind: "rendition" | "original";
  url: string;
  width: number;
  height: number;
  mediaType: Asset["mediaType"];
  expiresAt: string;
};

const allowedSizes = new Set(["320x320", "800x600", "1200x675"]);

function clamp01(value: number): number {
  return Math.min(1, Math.max(0, value));
}

function signPath(path: string, expiresAt: number): string {
  const token = Buffer.from(`${path}:${expiresAt}`).toString("base64url");
  return `https://media.example.invalid${path}?expires=${expiresAt}&token=${token}`;
}

function renditionKey(asset: Asset, request: NonNullable<DeliveryRequest["crop"]>): string {
  const size = `${request.width}x${request.height}`;
  if (!allowedSizes.has(size)) throw new Error("unsupported rendition size");

  const focalX = clamp01(request.focalX).toFixed(3);
  const focalY = clamp01(request.focalY).toFixed(3);
  return `renditions/${asset.contentHash}/${size}/fx-${focalX}-fy-${focalY}.webp`;
}

function createDelivery(asset: Asset, request: DeliveryRequest, now = Date.now()): Delivery {
  const expiresAt = now + 5 * 60 * 1000;

  if (request.mode === "original") {
    if (!request.purchased) throw new Error("original requires an active entitlement");
    return {
      kind: "original",
      url: signPath(`/private/${asset.originalKey}`, expiresAt),
      width: asset.width,
      height: asset.height,
      mediaType: asset.mediaType,
      expiresAt: new Date(expiresAt).toISOString(),
    };
  }

  const crop = request.crop ?? { width: 800, height: 600, focalX: 0.5, focalY: 0.5 };
  return {
    kind: "rendition",
    url: signPath(`/${renditionKey(asset, crop)}`, expiresAt),
    width: crop.width,
    height: crop.height,
    mediaType: "image/webp",
    expiresAt: new Date(expiresAt).toISOString(),
  };
}

const photo: Asset = {
  id: "shipment-photo-1842",
  originalKey: "originals/shipment-photo-1842.jpg",
  contentHash: "sha256-example-7d4a",
  width: 4032,
  height: 3024,
  mediaType: "image/jpeg",
};

console.log(createDelivery(photo, {
  buyerId: "buyer-91",
  assetId: photo.id,
  purchased: false,
  mode: "preview",
  crop: { width: 320, height: 320, focalX: 0.62, focalY: 0.44 },
}));
```

This example makes three decisions visible. Originals are private. Preview dimensions come from a small allowlist. The rendition key includes the source content hash, output geometry, and normalized focal point, so two identical requests converge on one object instead of accumulating timestamped duplicates.

One source. Few shapes.

The `purchased` Boolean is only compact sample data, not a production authorization model. In a real handler, derive entitlement from server-side purchase records using the authenticated buyer ID and asset ID. Never accept purchase state, an original storage key, or a focal point asserted by an untrusted client without validation.

## Build the crop pipeline as a bounded cache

Smart cropping should produce coordinates, not grant access. Store the chosen focal point or crop rectangle as versioned metadata beside the asset record. The worker then uses that metadata to generate the three approved shapes. If the crop logic changes, increment a transformation version in the rendition key; do not overwrite old bytes under an unchanged key.

The storage calculation is plain enough to do before writing a worker. Suppose the catalog has 40,000 source photos, three standard shapes, and an illustrative average of 140 KB per processed file. Eagerly materializing every shape creates 120,000 derivative objects and about 16.8 GB of derivative payload before replicas or metadata overhead. Those are planning assumptions, not a benchmark. Replace them with a sample from the actual shipment catalog.

Do the same arithmetic for reads. A square search thumbnail may be requested thousands of times and is a good eager candidate. A one-off export size is a poor candidate: generate it only after a miss, cache it with a retention policy, and record whether it is ever reused. **Popularity, not the number of possible dimensions, should decide what remains materialized.**

Use immutable cache keys for processed content and a short-lived signed delivery URL for authorization. Those lifetimes solve different problems. The cached object can remain stable while the URL expires quickly; changing the token does not force the crop worker to make identical pixels again.

One trap is allowing arbitrary width and height query parameters. A bot, a typo, or an overenthusiastic responsive-image component can create thousands of nearly identical objects: `799x600`, `800x600`, `801x600`, and onward. Picture one shipment photo moving through three surfaces. Search asks for `320x320`; the detail page asks for `800x600`; the dispatch summary asks for `1200x675`. Those three requests settle into three reusable objects. If each frontend instead rounds its container width independently, the same photo can collect dozens of derivatives that differ by a pixel and look identical to a buyer. That is the storage failure to design out. The allowlist in the example stops the cardinality leak, gives frontend code a shared vocabulary, and makes cache behavior legible. It does trade away arbitrary layout freedom, so changing a card shape becomes a coordinated schema change rather than a casual query-string edit.

Keep it boring.

## Separate authorization failure from image failure

The delivery service should distinguish four outcomes internally: no entitlement, unknown asset, invalid transformation, and rendition unavailable. Public responses may intentionally reveal less, but logs and metrics need the precise category. Otherwise, a burst of expired purchase access looks identical to a broken crop worker.

Do not fall back from a missing rendition to the private original. That shortcut converts a processing miss into a disclosure. A safer path is to queue or perform the permitted transformation, return a bounded retry response if the system cannot finish in time, and keep the original behind the entitlement check. If a preview request fails, a prebuilt neutral placeholder is safer than a source-photo fallback.

Content negotiation deserves restraint. Image formats differ in browser support, features, compression behavior, and use cases; MDN's image format guide is a useful compatibility reference. Keep the source media type in metadata, select output formats using tested client capabilities, and include the representation choice in the cache key. Do not label transformed bytes with the source's media type.

Observability should follow the same boundary. Record the transformation key, result dimensions, output byte count, cache status, processing duration, and an opaque asset identifier. Record authorization decisions separately with buyer and purchase identifiers appropriate to the retention policy. Signed URLs and tokens do not belong in logs.

This approach has limits. It is a poor fit when buyers need arbitrary editorial crops, lossless scientific inspection, or immediate access to every source format; a controlled rendition vocabulary deliberately sacrifices that flexibility. On the other side, prebuilding every approved crop wastes storage when most catalog images are rarely viewed. The practical trade-off is to prebuild only the shapes used on high-traffic surfaces and create the long tail after a cache miss.

No fallback means no leak.

## How do you know the design is ready to ship?

Start with contract tests around the decision table: an unpurchased buyer cannot request an original; a purchased buyer can; both may request an allowed preview; unsupported geometry is rejected. Add tests showing that focal points below zero or above one normalize deterministically and that repeated requests produce the same rendition key. Then test the uncomfortable path: delete a derived object and verify that the system regenerates only that allowed rendition without exposing the source.

Deployment should be staged by cache behavior, not just error rate. Watch the ratio of generated renditions to cache hits, derivative bytes per source asset, unique transformation keys, and original-download authorizations. A sudden rise in unique keys often means a caller bypassed the size vocabulary. A rise in generation with flat traffic suggests unstable keys or missing cache writes.

The operational checklist is short enough to keep in prose. Confirm that originals live in a private namespace, entitlement is read server-side, delivery links expire, and processed objects use immutable deterministic names. Confirm that every dimension is bounded, crop metadata is versioned, format and dimensions match the returned bytes, and logs omit credentials. Finally, restore one original and one rendition from backup, revoke a purchase entitlement, and verify that a previously issued URL stops working at its declared expiry.

The final choice is not original versus processed for every request. It is processed for presentation and original for an entitled post-purchase download, with separate controls for each. That division keeps the buyer experience predictable while making storage growth and cache reuse measurable.

## References

- MDN Web Docs, “Image file type and format guide”: https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
