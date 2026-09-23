# True Redaction vs Drawing Black Boxes: Safer E-Commerce PDF Forms

TL;DR: A black rectangle changes what a legal PDF looks like; true redaction removes the sensitive content and related recoverable traces before release. For an e-commerce team filling and flattening merchant disclosure forms, the practical rule is to redact first, rebuild or sanitize the document as needed, flatten only to lock the intended presentation, and then test both appearance and extraction. Flattening is not redaction.

This distinction matters whenever a merchant response packet contains customer names, order notes, addresses, signatures, or internal case commentary. The cheapest render that looks right in one viewer can still ship the original text underneath an opaque shape. A defensible workflow spends render budget where it buys evidence: on the final artifact, across multiple inspection methods, before the file leaves the controlled boundary.

## Why doesn't a black box remove the text?

PDF is a page-description format. A page can contain text-drawing operators and then paint an opaque rectangle over the same coordinates. The later object may hide the earlier one on screen, but it does not necessarily alter or delete the earlier object's bytes or logical content. Copy and paste, text extraction, object inspection, search, accessibility tooling, or removing the covering object can expose what the page still contains. ISO 32000-2 defines the PDF format and its object and content-stream model; visual overlap is not a deletion guarantee.

Flattening addresses a different problem. It commonly merges interactive form values or annotations into page content so the document presents consistently and fields are no longer casually editable. That is useful after an e-commerce service fills a return authorization, seller affidavit, or legal-response form. It does not prove that covered text, annotation contents, embedded files, prior revisions, metadata, or alternate representations have disappeared.

One trap is especially easy to miss: a form value can exist in the field dictionary while an appearance stream displays the value on the page. Painting over the appearance, or flattening that appearance, is not evidence that the field value was removed. The release check must inspect the final saved file, not the in-memory page preview.

Black boxes fail.

## Make removal a document transformation

Treat redaction as a boundary in the document pipeline, not as a drawing command. Inputs stay restricted. A policy layer identifies exact regions and data classes to remove; a PDF processor applies redactions or reconstructs affected pages; a sanitizer handles non-page containers; and a release validator examines the newly serialized output. Only that output may move to signing, delivery, or archival storage.

Order matters. Fill the required merchant-facing values, resolve which content is allowed, apply true redaction, remove prohibited document-level material, and then flatten the approved form presentation. If filling happens after redaction, a template or mapping bug can reintroduce sensitive values. If signing happens before mutation, the later redaction invalidates the signature's coverage. Digital signatures protect a byte range, so the artifact intended for release should be finalized before it is signed.

The transformation should use a fresh output file rather than an incremental update. PDF supports incremental changes that append new objects while retaining earlier bytes. Retaining those bytes is useful for revision history and signatures, but it is the wrong property for a sanitized disclosure copy. The NSA and CISA guidance on redacting documents explicitly warns that obscuring information is insufficient and recommends verification of the released result.

Consider one merchant affidavit with a customer address in a text field, a scan of an order note behind it, and an internal comment attached to the page. A rectangle can hide the field's appearance while leaving its value available to a parser; flattening can burn that same appearance into the page while the attachment or comment survives elsewhere; rasterizing only the visible page can remove the selectable text but also preserve the address as pixels for OCR to recover. The correct transformation therefore depends on where each sensitive value lives. The release gate has to inspect the field tree, annotations, attachments, content streams, rendered pixels, and saved revision structure rather than declaring success because one screenshot looks clean. This is the concrete reason a single visual check has poor coverage.

**The release unit is the serialized PDF, not the canvas screenshot.**

## A focused release gate in TypeScript

No single extraction library proves absence. Still, a small gate can enforce useful invariants around a standards-based PDF transformation service: it keeps forbidden source strings out of logs, hashes the exact artifact reviewed, rejects interactive remnants, and requires independent content checks before release. The processor behind this interface may be local or remote; the contract is the important part.

```ts
type Region = { page: number; x: number; y: number; width: number; height: number };

type Inspection = {
  extractedText: string;
  formFieldCount: number;
  embeddedFileCount: number;
  hasIncrementalUpdates: boolean;
};

interface PdfPipeline {
  fill(input: Uint8Array, values: Record<string, string>): Promise<Uint8Array>;
  redact(input: Uint8Array, regions: Region[]): Promise<Uint8Array>;
  sanitize(input: Uint8Array): Promise<Uint8Array>;
  flatten(input: Uint8Array): Promise<Uint8Array>;
}

interface IndependentInspector {
  inspect(input: Uint8Array): Promise<Inspection>;
}

export async function buildReleaseCopy(
  source: Uint8Array,
  values: Record<string, string>,
  regions: Region[],
  forbiddenTerms: string[],
  pdf: PdfPipeline,
  inspector: IndependentInspector,
): Promise<Uint8Array> {
  const filled = await pdf.fill(source, values);
  const redacted = await pdf.redact(filled, regions);
  const sanitized = await pdf.sanitize(redacted);
  const output = await pdf.flatten(sanitized);
  const report = await inspector.inspect(output);

  const normalized = report.extractedText.normalize("NFKC").toLocaleLowerCase("en-US");
  const leaked = forbiddenTerms.filter((term) =>
    normalized.includes(term.normalize("NFKC").toLocaleLowerCase("en-US")),
  );

  if (leaked.length > 0) throw new Error("Release copy contains prohibited text");
  if (report.formFieldCount !== 0) throw new Error("Release copy retains form fields");
  if (report.embeddedFileCount !== 0) throw new Error("Release copy retains embedded files");
  if (report.hasIncrementalUpdates) throw new Error("Release copy retains prior revisions");

  return output;
}
```

This example deliberately does not log the matching terms, coordinates, or extracted text. Those values are sensitive too. In production, keep the policy decision and artifact hash in an audit record, with access controls and retention rules appropriate to the legal matter. Do not put raw customer data into traces just to make the document job observable.

The check is necessary but incomplete. Text may be encoded in ways one extractor misses; sensitive content may be an image; optical character recognition can rediscover rasterized text; and annotations or attachments can carry information outside the primary page stream. Run at least 2 independent inspection paths: structural parsing plus rendering and extraction. For scanned pages, add OCR against the released render.

## Fidelity and render cost belong in the same test

True removal can change layout. Removing a line from a dense seller declaration may leave an obvious gap; reconstructing or rasterizing a page can alter fonts, vector detail, accessibility, file size, or signature behavior. Those are release-quality concerns, but none justifies keeping secret content in the file. Security is the constraint. Fidelity is optimized inside it.

A useful fixture set is small and hostile rather than huge and comfortable. Start with 8 fixtures: a native-text form, a scanned form with an OCR text layer, rotated pages, repeated customer values, multiline fields, annotations, an attachment, and a file saved incrementally. Seed each fixture with unique canary values that never occur elsewhere. Then compare the released artifact against the intended page image and search for every canary through structural extraction, copy and paste, OCR, and byte-level inspection where applicable.

Measure three cost buckets separately: transformation CPU time, rendered pixels, and inspection work. Page count alone is misleading because a vector form and a high-resolution scanned exhibit can have very different render costs. Cache only unrestricted source assets and policy-independent intermediates; caching a partially redacted document creates another sensitive artifact with an ambiguous release status.

A sensible tiered path preserves vectors and text when the processor can remove the selected objects and sanitize surrounding structures. Rebuild a page when its object graph or font encoding makes removal hard to verify. Rasterize only when the legal and accessibility requirements permit it, because rasterization can reduce searchability and enlarge files while still requiring OCR-based leak testing.

This approach has real limitations. Object-level removal preserves the best fidelity, but unusual encodings and shared resources make verification harder. Page reconstruction gives a cleaner boundary at the cost of more layout drift and engineering work. Full-page rasterization is easier to reason about structurally, yet it is a poor fit when searchable text, accessibility, compact files, or crisp vector output are requirements. The trade-off is explicit: use the least destructive transformation whose result can pass independent leak tests, and pay the extra render cost only for pages that cannot be verified otherwise.

Fast matters. Wrong matters more.

## What must pass before disclosure?

The final gate should answer concrete questions about the exact bytes being released. Can two independent extractors recover a canary? Does OCR find it in any rendered page? Are form fields, comments, attachments, hidden layers, metadata, or prior revisions present? Does the visual diff show clipped required text, missing signatures, shifted checkboxes, substituted fonts, or unexpected blank areas? Is the output hash the same hash recorded in the approval and delivery events?

A pass should also be reproducible. Pin the processor and renderer versions, keep immutable test fixtures, record policy identifiers rather than secret values, and rerun the suite when the PDF engine, fonts, OCR component, or form template changes. A renderer upgrade can affect fidelity without changing policy code; a template revision can move a field outside an old coordinate-based redaction region.

Coordinate-only policies are brittle for exactly that reason. Prefer semantic anchors when the source format exposes stable fields, then resolve them to page regions and verify the result visually. For scanned evidence, human review remains appropriate when OCR confidence or page geometry makes the target uncertain. Automation should route ambiguity, not silently guess.

**Approve the artifact only after removal checks and fidelity checks pass on the same file.** Drawing a black box may be acceptable as a visual annotation in an internal working copy. It is not a redaction control for a legal disclosure.

## Further reading

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
- NSA and CISA, Redacting Documents Safely: https://media.defense.gov/2023/Dec/14/2003351919/-1/-1/0/CSI-REDACting-DOCUMENTS-SAFELY.PDF
- NIST, Guidelines for Media Sanitization (SP 800-88 Rev. 1): https://csrc.nist.gov/pubs/sp/800/88/r1/final
- W3C, PDF Techniques for WCAG 2.0: https://www.w3.org/TR/WCAG20-TECHS/pdf.html
