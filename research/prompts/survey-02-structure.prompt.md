# Deep Research Prompt — REST API Conventions Series (Part 2/7): Structure — Resources, URLs, Methods & Status Codes

> Part of a 7-part **descriptive** research series feeding a follow-up phase where the requester will define a prescriptive REST API design standard. Each part is self-contained and reports are merged later. This part covers ONLY the surface below — representations/errors, collections, reliability, lifecycle, and webhooks are covered in Parts 3–7 — so do not expand scope.

## Scope line

**The exact question:** How do the eight references structure their resource space — URL design, resource modeling, HTTP method usage, and status-code usage — and on which structural axes does the field genuinely split? (This is the structural core of the eventual standard, analogous to command ordering in CLI design.)

## Mandate

- **DESCRIPTIVE ONLY.** Document what each reference actually does today; no "the standard should…" statements. Flag conflicts and tradeoffs descriptively; decisions happen later.
- Treat **Stripe as a key reference to be checked against the field — not canonical.**
- **Decision-readiness is the quality bar:** capture each divergence precisely enough to decide from without re-research.

## Reference set (compare all eight on this surface)

1. **Stripe** 2. **GitHub REST** 3. **Google Cloud / AIP (aip.dev)** 4. **Microsoft (Azure REST guidelines + Graph)** 5. **Twilio** 6. **Shopify Admin REST** (flag its REST→GraphQL migration where it affects currency) 7. **Zalando RESTful API Guidelines** (a guideline document, not an API) 8. **AWS — this part is where AWS gets its FULL contrast treatment:** document precisely how its action-based/RPC-style service APIs (e.g., EC2 Query APIs, DynamoDB's JSON protocols, `Action=` parameters) diverge from resource-oriented REST, and what that divergence costs and buys. AWS exists in this series to show the boundary of the convention space, not to be emulated.

## Out of scope (entire series)

OAuth/OIDC protocol internals (auth surface is Part 6); GraphQL/gRPC/SOAP; gateway/infra; SDK design; event-streaming platforms (webhooks are Part 7).

## Surface to research

1. **Resource modeling** — resource-oriented vs action-based design; how each names resources; collection vs singleton resources; where business *actions* that don't map to CRUD live.
2. **Noun number & URL casing** — plural vs singular path segments; kebab vs snake vs camel in paths; casing of query-parameter names.
3. **Nesting & relationships** — subresource depth in practice (`/customers/{id}/subscriptions` vs flat `/subscriptions?customer=`); when each nests vs references; maximum observed depth.
4. **Custom / non-CRUD actions** — Google AIP custom methods (`:verb` suffix) vs POST-to-action-path (verify and document Stripe's pattern, e.g., `POST /v1/payment_intents/{id}/capture`) vs action-in-body; how Zalando's guidelines treat this.
5. **HTTP method usage** — which methods each API actually uses; **verify and document the widely-reported fact that Stripe uses POST (not PATCH/PUT) for updates, and its historical use of form-encoded request bodies** — this is a key data point; PUT-vs-PATCH semantics where both exist; PATCH body format in use (JSON Merge Patch vs JSON Patch vs proprietary); method-override headers; adherence to RFC 9110 safe/idempotent semantics.
6. **Status codes** — the working subset each API actually returns; 200 vs 201 (with Location?) vs 202 vs 204 for creates/updates/deletes; 400 vs 422 for validation failures; **404-vs-403 for existence-hiding** (verify GitHub's documented policy of 404 for unauthorized access); 409 usage; 405/406/415 usage; redirect usage in APIs.
7. **URL mechanics** — trailing-slash policy; path parameter styles; reserved/special segments; versioned path prefixes as *structure* (the versioning *scheme* itself is Part 6).

## Quality bar

- Primary sources first (official API references, aip.dev, the Azure guidelines repo, Zalando's GitHub guidelines, first-party engineering posts); secondary only as corroboration.
- **Currency:** note retrieval dates for anything volatile.
- **Confidence** per non-obvious finding, with basis.
- **Surface disagreements** between docs and widely-reported behavior rather than silently picking one.

## Specification-grade detail requirement

A finding on this surface is complete only when someone could implement or emulate the mechanism from this report **without opening the reference's own docs**. For every mechanism documented:

- **Exact names & formats** — header names with their value syntax, field/parameter names verbatim, media types, enum values.
- **Verbatim examples** — at least one real example per major mechanism: a request/response payload or header line, quoted (minimally elided for length).
- **Concrete numbers** — limits, windows, defaults, maximums — each with its source and retrieval date.

Summaries without these artifacts do not satisfy the deliverable.

## Required deliverable structure

1. **TL;DR** 2. **Key findings** (numbered)
3. **Baseline position** — what RFC 9110 and the guideline documents (AIP, Azure, Zalando) prescribe for THIS surface
4. **Side-by-side comparison tables** across all eight (one per dimension above, or sensibly grouped)
5. **Per-reference notes** — a structural character sketch of each, including the full AWS contrast analysis
6. **Agreements vs divergences** — de facto standards vs splits, each divergence with its tradeoffs stated descriptively
7. **CONTESTED AXES REGISTER (scoped to this part)** — one row per contested axis (expect: plural/singular, path casing, nesting policy, custom-action syntax, update method POST-vs-PUT-vs-PATCH, PATCH format, 400-vs-422, 404-vs-403, trailing slash, resource-vs-action orientation). Columns: **Axis · Options observed · Who does what · Tradeoff in one line · How contested** (near-consensus / split / wide-open). Each row self-contained enough to become a decision item directly.
8. **EXAMPLES APPENDIX** — every verbatim payload, header line, and concrete number collected under the specification-grade requirement, grouped by reference
9. **Caveats**
