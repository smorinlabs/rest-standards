# Deep Research Prompt — REST API Conventions Series (Part 1/7): Foundations & Standards Layer

> Part of a 7-part **descriptive** research series feeding a follow-up phase where the requester will define a prescriptive REST API design standard. Each part is self-contained and reports are merged later. This part covers ONLY the surface below — other parts cover resource structure, representations, collections, reliability, lifecycle, and webhooks — so do not expand scope.

## Scope line

**The exact question:** What does the foundational standards layer for HTTP/REST APIs actually specify, what is the current status of each document, and how much of it does major industry practice genuinely adopt versus ignore?

## Mandate

- **DESCRIPTIVE ONLY.** Document what the standards say and what adoption looks like today; no "the standard should…" statements. Decisions happen in a later phase.
- Treat **Stripe as a key reference to be checked against the field — not canonical.**
- **Decision-readiness is the quality bar:** capture each finding precisely enough to decide from later without re-research.

## Reference set (used here for adoption spot-checks, not full analysis)

Stripe · GitHub REST · Google Cloud/AIP (aip.dev) · Microsoft (Azure REST guidelines + Graph) · Twilio · Shopify Admin REST · Zalando RESTful API Guidelines · AWS (contrast). Full per-API analysis happens in Parts 2–7; here, use them only to answer "who actually adopts this baseline?"

## Out of scope (entire series)

OAuth/OIDC protocol internals (the auth *surface* is covered in Part 6); GraphQL/gRPC/SOAP; gateway/infra; SDK design; event-streaming platforms (webhooks are covered in Part 7).

## Surface to research

1. **RFC 9110 (HTTP Semantics)** — the method table (safe / idempotent / cacheable properties per method), status-code class semantics, conditional-request machinery, content negotiation basics. Note what changed vs the older RFC 7230–7235 family.
2. **RFC 9111 (HTTP Caching)** — the parts relevant to APIs: Cache-Control directives, validators, freshness.
3. **RFC 9457 Problem Details** (and predecessor RFC 7807) — the full format (`type`, `title`, `status`, `detail`, `instance`, extensions), the registry, what changed from 7807, and an honest **adoption survey**: which of the eight references use it, which use proprietary error shapes (details of those shapes belong to Part 3 — here just the adoption fact).
4. **RFC 3339 timestamps** — profile vs ISO 8601, offset rules, and who in the field deviates (flag any epoch-seconds users for Part 3 to detail).
5. **RFC 8288 Web Linking** — the `Link` header, standard relations (`next`, `prev`, etc.), and who actually uses it.
6. **RFC 7396 JSON Merge Patch vs RFC 6902 JSON Patch** — the semantics of each, especially Merge Patch's null-means-delete rule; adoption reality.
7. **RFC 8594 Sunset header** — semantics and adoption.
8. **IETF `httpapi` working group outputs — verify current status as of the run date:** the RateLimit header fields draft, the Idempotency-Key header draft, and the Deprecation header (which may have been published as an RFC recently). For each: what it specifies, its maturity, and known adopters.
9. **JSON:API 1.1** — what it standardizes (envelopes, `include`, `fields`, errors), and where it sits in industry adoption (widely cited, selectively adopted?).
10. **OpenAPI 3.1 + JSON Schema** — the 3.1/JSON Schema alignment, and which of the eight publish official OpenAPI documents (detail of docs practice belongs to Part 6 — here just the standards relationship).
11. **Fielding's REST dissertation + the Richardson Maturity Model** — the constraints, the levels, and an honest account of **how much "true REST"/HATEOAS major industry APIs actually implement** (which level the eight actually operate at). This adoption-reality finding is a first-class deliverable, not an aside.

## Quality bar

- Primary sources first: IETF datatracker/RFC editor, the specs themselves, official API docs for adoption checks; secondary sources only as corroboration.
- **Currency:** note the retrieval date; explicitly verify the current draft/RFC status of every `httpapi` item.
- **Confidence** per non-obvious finding, with basis.
- **Surface disagreements** between sources rather than silently picking one.

## Specification-grade detail requirement

A finding on this surface is complete only when someone could implement or emulate the mechanism from this report **without opening the reference's own docs**. For every mechanism documented:

- **Exact names & formats** — header names with their value syntax, field/parameter names verbatim, media types, enum values.
- **Verbatim examples** — at least one real example per major mechanism: a request/response payload or header line, quoted (minimally elided for length).
- **Concrete numbers** — limits, windows, defaults, maximums — each with its source and retrieval date.

Summaries without these artifacts do not satisfy the deliverable.

## Required deliverable structure

1. **TL;DR**
2. **Key findings** (numbered)
3. **Per-standard sections** — what it specifies, status, adoption
4. **Adoption matrix** — rows = the standards above, columns = the eight references, cells = adopts / partial / ignores (with a word of nuance)
5. **Where the standards layer is settled vs contested in practice**
6. **CONTESTED AXES REGISTER (scoped to this part)** — one row per axis where standard-vs-practice genuinely splits (e.g., Problem Details vs proprietary errors; Link header vs body pagination; HATEOAS vs level-2 pragmatism; Merge Patch vs JSON Patch). Columns: **Axis · Options observed · Who does what · Tradeoff in one line · How contested** (near-consensus / split / wide-open). Each row self-contained enough to become a decision item directly.
7. **EXAMPLES APPENDIX** — every verbatim payload, header line, and concrete number collected under the specification-grade requirement, grouped by standard
8. **Caveats** — anything unverifiable, drafts in flux
