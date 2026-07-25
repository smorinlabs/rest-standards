# Deep Research Prompt — REST API Conventions Series (Part 4/7): Collections — Pagination, Filtering, Sorting, Selection & Expansion

> Part of a 7-part **descriptive** research series feeding a follow-up phase where the requester will define a prescriptive REST API design standard. Each part is self-contained and reports are merged later. This part covers ONLY the surface below — general body conventions are Part 3, reliability Part 5, lifecycle Part 6 — so do not expand scope.

## Scope line

**The exact question:** How do the eight references let clients query collections — pagination mechanics, filtering syntax, sorting, field selection, and related-resource expansion — and on which of these axes does the field genuinely split?

## Mandate

- **DESCRIPTIVE ONLY.** Document what each reference actually does today; no "the standard should…" statements. Flag conflicts and tradeoffs descriptively; decisions happen later.
- Treat **Stripe as a key reference to be checked against the field — not canonical.**
- **Decision-readiness is the quality bar:** capture each divergence precisely enough to decide from without re-research.

## Reference set (compare all eight on this surface)

1. **Stripe** 2. **GitHub REST** 3. **Google Cloud / AIP (aip.dev)** 4. **Microsoft (Azure REST guidelines + Graph)** 5. **Twilio** 6. **Shopify Admin REST** — a key reference for THIS part (cursor pagination at scale); flag currency caveats from its GraphQL migration 7. **Zalando RESTful API Guidelines** (guideline document) 8. **AWS** — light treatment: `NextToken` pattern and filter conventions as one-line contrast notes.

## Out of scope (entire series)

OAuth/OIDC internals; GraphQL/gRPC; gateway/infra; SDK design; event streaming. Rate limiting belongs to Part 6 even though it interacts with pagination.

## Surface to research

1. **Pagination style** — cursor vs offset vs page-number per reference: **verify and document** Stripe's `starting_after`/`ending_before` + `limit` + `has_more`; GitHub's `Link` header with `page`/`per_page` (and its newer cursor-based endpoints); Google AIP-158 `page_size`/`page_token`/`next_page_token`; Microsoft Graph `@odata.nextLink`/`$skiptoken`; **Shopify's `Link` header + `page_info` cursors (and the deprecation of page-based access)**; Twilio's `Page`/`PageSize` + URI links; Zalando's guidance; AWS `NextToken`.
2. **Pagination response shape** — where the "next" signal lives: body fields (`has_more`, `next_page_token`, `@odata.nextLink`) vs `Link` header (RFC 8288); first/prev/last support; cursor opacity and stability guarantees.
3. **Totals & counts** — whether a total count is available (`$count`, `total_count`, none-by-design); documented reasons for omitting totals at scale.
4. **Page-size policy** — defaults and maximums per reference; behavior when limits are exceeded.
5. **Filtering syntax** — the syntax families: **Microsoft OData `$filter` expressions; Google AIP-160 filter language; Stripe's minimal per-field params + range syntax (`created[gte]=…`) and — verify — its separate Search API query language (`query=`); GitHub's `q=` search syntax on search endpoints; Shopify's per-field params**; Zalando's guidance; expressiveness vs simplicity positioning of each.
6. **Sorting** — `$orderby` vs `order_by` vs `sort`+`direction` params; multi-key sorting; default orders and whether they're documented/stable.
7. **Field selection (sparse fieldsets)** — `$select`, AIP field masks (`read_mask`), JSON:API `fields[type]`; who offers none (verify: Stripe relies on `expand[]` instead of selection).
8. **Expansion / embedding** — **Stripe `expand[]` (including nested `expand[]=data.customer` and depth limits)**; Graph `$expand`; JSON:API `include`; AIP resource views / `view` enums; default-embedded vs reference-by-ID postures; N+1 tradeoff positioning.
9. **Collection-adjacent conventions** — search-as-GET-vs-POST where bodies get large; list-endpoint consistency guarantees (eventual vs strong read-after-write, where documented).

## Quality bar

- Primary sources first (official API references, aip.dev AIP-132/158/160, the Azure/Graph docs, shopify.dev, Zalando's repo); secondary only as corroboration.
- **Currency:** note retrieval dates; Shopify's REST status especially.
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
3. **Baseline position** — RFC 8288, JSON:API, and the guideline docs on THIS surface
4. **Side-by-side comparison tables** across all eight (pagination mechanics; filter/sort syntax; selection/expansion)
5. **Per-reference notes** — a collections character sketch of each
6. **Agreements vs divergences** — each divergence with tradeoffs, descriptive
7. **CONTESTED AXES REGISTER (scoped to this part)** — one row per contested axis (expect: cursor-vs-offset, next-signal-in-body-vs-Link-header, totals exposure, page-size defaults, filter-syntax family, sort syntax, selection mechanism, expansion mechanism, search GET-vs-POST). Columns: **Axis · Options observed · Who does what · Tradeoff in one line · How contested** (near-consensus / split / wide-open). Each row self-contained enough to become a decision item directly.
8. **EXAMPLES APPENDIX** — every verbatim payload, header line, and concrete number collected under the specification-grade requirement, grouped by reference
9. **Caveats**
