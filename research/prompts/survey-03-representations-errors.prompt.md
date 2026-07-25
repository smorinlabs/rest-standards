# Deep Research Prompt — REST API Conventions Series (Part 3/7): Representations & Errors

> Part of a 7-part **descriptive** research series feeding a follow-up phase where the requester will define a prescriptive REST API design standard. Each part is self-contained and reports are merged later. This part covers ONLY the surface below — structure is Part 2, collections Part 4, reliability Part 5, lifecycle Part 6, webhooks Part 7 — so do not expand scope.

## Scope line

**The exact question:** What do request/response bodies actually look like across the eight references — field conventions, data-type representations, and error shapes — and on which representation axes does the field genuinely split? ("What the JSON looks like," for both success and failure.)

## Mandate

- **DESCRIPTIVE ONLY.** Document what each reference actually does today; no "the standard should…" statements. Flag conflicts and tradeoffs descriptively; decisions happen later.
- Treat **Stripe as a key reference to be checked against the field — not canonical.**
- **Decision-readiness is the quality bar:** capture each divergence precisely enough to decide from without re-research.

## Reference set (compare all eight on this surface)

1. **Stripe** 2. **GitHub REST** 3. **Google Cloud / AIP (aip.dev)** 4. **Microsoft (Azure REST guidelines + Graph)** 5. **Twilio** 6. **Shopify Admin REST** (flag currency caveats from its GraphQL migration) 7. **Zalando RESTful API Guidelines** (guideline document) 8. **AWS** — light treatment here: one-line notes on its body/error conventions as contrast (its full analysis lives in Part 2).

## Out of scope (entire series)

OAuth/OIDC internals (auth surface is Part 6); GraphQL/gRPC/SOAP; gateway/infra; SDK design; event streaming (webhooks are Part 7). Pagination/filter/expansion *response shapes* belong to Part 4 — here cover only the general body conventions.

## Surface to research

1. **Field casing** — snake_case vs camelCase per reference; **verify the split:** Stripe/GitHub snake_case vs Google (proto3 JSON mapping → camelCase in JSON) vs Microsoft Graph camelCase; what Zalando mandates; consistency of query-param casing with body casing.
2. **Envelopes** — bare objects/arrays vs typed envelopes; **verify and document Stripe's `{"object": "list", "data": [...]}` pattern** and its `object` type field on every resource; GitHub's bare arrays; JSON:API's envelope as the formalized extreme.
3. **Null vs absent** — whether fields are omitted when empty, always present, or null; PATCH implications (null-means-delete under Merge Patch); documented policies per reference.
4. **Timestamps** — **verify and document the major divergence: Stripe uses integer Unix epoch seconds, while GitHub/Google/Microsoft use RFC 3339/ISO 8601 strings**; timezone/offset conventions; field naming (`created` vs `created_at` vs `createTime`).
5. **Money & amounts** — minor-units integers + currency code (Stripe) vs decimal strings vs floats (who, if anyone, uses floats); currency representation.
6. **Identifiers** — **prefixed opaque IDs (Stripe `cus_…`, and verify GitHub's prefixed tokens/node IDs) vs UUIDs vs numeric IDs vs Google's full resource names (`projects/p/locations/l/…`)**; opacity guarantees; ID stability commitments.
7. **Standard/common fields** — `id`, `object`, `created`, `updated`, `metadata` (Stripe's user-metadata pattern), `livemode`; Google's `name`/`create_time`/`update_time`/`etag`; which common fields recur across the field.
8. **Enums & booleans** — enum value casing (lower_snake vs SCREAMING_SNAKE vs camel); open vs closed enum policies (unknown-value handling); boolean naming conventions.
9. **Error bodies** — the full shape per reference: **Stripe's `error{type, code, message, param, doc_url}`; GitHub's `message` + `errors[]` + `documentation_url`; Google's `google.rpc.Status` (`code`, `message`, `details[]`); Microsoft Graph's `error{code, message, innerError}`; verify that Zalando mandates RFC 9457 `application/problem+json`**; RFC 9457 adoption vs proprietary shapes (adoption fact-level from Part 1 — full shapes here).
10. **Validation errors & error taxonomies** — per-field error detail shapes; machine-readable code style (string codes vs numeric); documented error-code catalogs; stability guarantees on codes.
11. **Request/correlation IDs** — response headers (Stripe `Request-Id`, GitHub `X-GitHub-Request-Id`, Azure `x-ms-request-id`); whether IDs appear in error bodies; support-workflow integration.

## Quality bar

- Primary sources first (official API references and guideline repos, first-party engineering posts); secondary only as corroboration.
- **Currency:** note retrieval dates for anything volatile.
- **Confidence** per non-obvious finding, with basis.
- **Surface disagreements** between sources rather than silently picking one.

## Specification-grade detail requirement

A finding on this surface is complete only when someone could implement or emulate the mechanism from this report **without opening the reference's own docs**. For every mechanism documented:

- **Exact names & formats** — header names with their value syntax, field/parameter names verbatim, media types, enum values.
- **Verbatim examples** — at least one real example per major mechanism: a request/response payload or header line, quoted (minimally elided for length).
- **Concrete numbers** — limits, windows, defaults, maximums — each with its source and retrieval date.

Summaries without these artifacts do not satisfy the deliverable.

## Required deliverable structure

1. **TL;DR** 2. **Key findings** (numbered)
3. **Baseline position** — what RFC 9457, RFC 3339, JSON:API, and the guideline docs prescribe for THIS surface
4. **Side-by-side comparison tables** across all eight (grouped sensibly: casing/envelope/null; scalars — time, money, IDs; errors)
5. **Per-reference notes** — a representation character sketch of each
6. **Agreements vs divergences** — each divergence with tradeoffs, descriptive
7. **CONTESTED AXES REGISTER (scoped to this part)** — one row per contested axis (expect: casing, envelope-or-bare, null policy, timestamp format epoch-vs-3339, money representation, ID format, error shape RFC-9457-vs-proprietary, validation-error shape, enum casing, metadata pattern). Columns: **Axis · Options observed · Who does what · Tradeoff in one line · How contested** (near-consensus / split / wide-open). Each row self-contained enough to become a decision item directly.
8. **EXAMPLES APPENDIX** — every verbatim payload, header line, and concrete number collected under the specification-grade requirement, grouped by reference
9. **Caveats**
