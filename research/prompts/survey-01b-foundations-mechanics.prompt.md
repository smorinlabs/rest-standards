# Deep Research Prompt — REST API Conventions Series (Part 1b): Foundations Supplement — Specification Mechanics

> Supplement to Part 1, which established the *status and adoption* of the foundational standards. This part captures the **implementation-grade mechanics** of those same standards — exact syntax, formats, and verbatim examples — so the eventual prescriptive standard can specify conforming usage without transcription errors. Descriptive only; same series rules as Part 1. Do not re-survey adoption (done) and do not analyze the eight reference APIs (Parts 2–7).

## Scope line

**The exact question:** For each foundational standard identified in Part 1, what are the precise mechanics — exact grammars, header/value formats, member semantics, and algorithms — captured with verbatim examples at implementation grade?

## Mandate

- **DESCRIPTIVE ONLY**; quote the specs, don't editorialize.
- **Decision-readiness bar:** every mechanism must be reproducible from this report alone, without opening the spec.

## Out of scope

Adoption surveys (Part 1 did this); per-API behavior (Parts 2–7); OAuth internals; non-REST paradigms.

## Surface to research — mechanics only, with verbatim examples for each

1. **RFC 9457 Problem Details** — precise semantics and type constraints of each member (`type`, `title`, `status`, `detail`, `instance`); extension-member rules and namespacing; `type` URI guidance including `about:blank`; how *multiple problems* are represented under 9457; media types (`application/problem+json`/`+xml`); interplay with the HTTP status code. Deliver **two verbatim example payloads**: one minimal, one with extensions.
2. **RFC 9110 conditional requests** — precondition evaluation order; strong vs weak ETag comparison; `If-Match`, `If-None-Match`, `If-Modified-Since`, `If-Unmodified-Since` usage for APIs; when the outcome is `304` vs `412`. Deliver **two verbatim header exchanges**: a cached-read flow ending in 304, and a guarded-write flow ending in 412.
3. **RFC 9111 essentials for APIs** — the exact `Cache-Control` directive strings for common API postures (e.g., `private, no-store`; `max-age=0, must-revalidate` + ETag); the freshness calculation in brief. Deliver example header lines.
4. **RFC 8288 Link header** — the serialization grammar (`<URI-Reference>; param=value`), multiple links in one field line, quoting rules; the standard relations relevant to APIs (`next`, `prev`, `first`, `last`, `self`, `describedby`, `deprecation`, `sunset`). Deliver **verbatim examples**: a pagination Link line and a deprecation/sunset Link line.
5. **RFC 7396 JSON Merge Patch** — the merge algorithm stated precisely; null-means-delete; whole-array replacement; the explicit-null limitation. Deliver a **verbatim before / patch / after triple**. **RFC 6902 JSON Patch** — all six operations with JSON Pointer (RFC 6901) syntax; atomic all-or-nothing application; the `test` operation; error behavior. Deliver a **verbatim patch document** applied to a sample target.
6. **RFC 3339** — the `date-time` grammar essentials; offset rules (`Z` vs numeric offsets, the `-00:00` meaning, including RFC 9557's refinement); fractional seconds. Deliver a **valid/invalid examples table** (include ISO-8601-valid-but-3339-invalid forms).
7. **RFC 8594 Sunset + RFC 9745 Deprecation** — exact value formats (`Sunset:` HTTP-date vs `Deprecation:` structured-field Date `@<unix>`); the associated link relations; how the two combine. Deliver a **verbatim combined example** (deprecated as of a past date, sunset at a future date).
8. **RateLimit header fields (draft-ietf-httpapi-ratelimit-headers, current draft)** — the exact field names and structured-field syntax as currently drafted (`RateLimit`, `RateLimit-Policy`, partition keys, windows); deliver verbatim example header lines; **mark clearly as draft/unstable** with the draft revision cited.
9. **Idempotency-Key header (draft-ietf-httpapi-idempotency-key-header, current/expired draft)** — exact header syntax (structured-field String); the server behaviors as drafted (fingerprinting, replay, conflict responses and status codes); deliver a **verbatim request/replay exchange**; **mark clearly as expired-draft/unstable** with the revision cited.
10. **OpenAPI 3.1 mechanics that matter for a standard** — `jsonSchemaDialect`, type arrays replacing `nullable`, the `webhooks` top-level object; deliver **small verbatim snippets** of each.
11. **JSON:API 1.1 shapes (as comparison material)** — a **verbatim minimal document** (top-level envelope, one resource object, one `included` entry) and a **verbatim error object** example.

## Quality bar

- Primary spec texts only (RFC Editor / IETF Datatracker / spec sites); quote exact syntax; note the retrieval date and, for drafts, the revision number.
- Flag any point where the spec is ambiguous or where errata exist.
- If any finding here *corrects* something in the Part 1 report, call it out explicitly in a "Corrections to Part 1" note.

## Specification-grade detail requirement

A finding on this surface is complete only when someone could implement the mechanism from this report **without opening the spec**. Exact names & formats; verbatim examples per mechanism; concrete numbers with sources. Summaries without these artifacts do not satisfy the deliverable.

## Required deliverable structure

1. **TL;DR**
2. **Per-standard mechanics sections** (items 1–11 above), each ending in its verbatim example(s)
3. **EXAMPLES APPENDIX** — all verbatim payloads/header lines in one place, grouped by standard
4. **Corrections to Part 1** (if any; otherwise state "none")
5. **Caveats** — draft instability, errata, ambiguities

*No Contested Axes Register is required for this supplement — Part 1 already produced it; this part is mechanics capture only.*
