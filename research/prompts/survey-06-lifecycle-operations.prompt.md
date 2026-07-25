# Deep Research Prompt — REST API Conventions Series (Part 6/7): Lifecycle & Operations — Versioning, Rate Limits, Caching, Auth Surface & Docs Practice

> Part of a 7-part **descriptive** research series feeding a follow-up phase where the requester will define a prescriptive REST API design standard. Each part is self-contained and reports are merged later. This part covers ONLY the surface below — reliability is Part 5, webhooks Part 7 — so do not expand scope.

## Scope line

**The exact question:** How do the eight references manage the API contract over time and under load — versioning and deprecation, rate limiting, caching/conditional reads, the authentication surface, and spec/docs publication — and on which of these axes does the field genuinely split?

## Mandate

- **DESCRIPTIVE ONLY.** Document what each reference actually does today; no "the standard should…" statements. Flag conflicts and tradeoffs descriptively; decisions happen later.
- Treat **Stripe as a key reference to be checked against the field — not canonical.**
- **Decision-readiness is the quality bar:** capture each divergence precisely enough to decide from without re-research.

## Reference set (compare all eight on this surface)

1. **Stripe** 2. **GitHub REST** 3. **Google Cloud / AIP (aip.dev)** 4. **Microsoft (Azure REST guidelines + Graph)** 5. **Twilio** 6. **Shopify Admin REST** (flag currency caveats) 7. **Zalando RESTful API Guidelines** (guideline document) 8. **AWS** — contrast notes here on **SigV4 request signing** as the auth-surface outlier, plus its `Version=` parameter versioning.

## Out of scope (entire series)

OAuth/OIDC protocol internals — **the auth *surface* IS in scope here** (key formats, bearer conventions, 401-vs-403, rotation surface), the protocol flows are not. Also out: GraphQL/gRPC; gateway/infra; SDK design; event streaming.

## Surface to research

1. **Versioning schemes** — the full landscape, verified per reference: **Stripe's account-pinned dated versions + `Stripe-Version` header** (how pinning, upgrades, and per-request overrides work); **GitHub's `X-GitHub-Api-Version` date header**; **Google's `/v1`, `/v1beta` path versions + stability channels**; **Microsoft Graph's `v1.0`/`beta` paths**; **Twilio's dated path (`/2010-04-01/`) and what it means in practice**; **Shopify's quarterly dated paths (`/admin/api/2024-01/`) and its support window**; **Zalando's position (verify the reported stance: prefer compatible evolution, avoid versioning, URL versioning discouraged)**; AWS `Version=`.
2. **Breaking-change definitions** — each reference's published backwards-compatibility policy: what counts as breaking vs additive (**Stripe's list is famous — capture it in full**); additive-change conventions (new fields, new enum values — links to Part 3's open-enum question).
3. **Deprecation & sunset signaling** — use of the `Deprecation` and `Sunset` headers (coordinate with Part 1's status findings); docs-level deprecation; timelines and support windows per reference; how each communicates migrations.
4. **Rate limiting** — header conventions per reference: **GitHub's `X-RateLimit-*` family (and its secondary limits)**; **Shopify's leaky-bucket `X-Shopify-Shop-Api-Call-Limit` + `Retry-After`**; Microsoft Graph throttling (429 + `Retry-After`, per-app/per-tenant nuance); **Stripe's posture (verify: limited public headers, 429-centric guidance)**; Google quota model; adoption of the standardized IETF `RateLimit-*` fields; 429 semantics; documented client backoff guidance; quota scopes (global vs per-endpoint vs per-resource).
5. **Caching & conditional reads** — ETag/Last-Modified issuance per reference; **verify GitHub's notable behavior: conditional requests (304s) not counting against the rate limit**; Cache-Control usage in APIs; where caching is explicitly disabled.
6. **Auth surface** — bearer-token conventions; **API-key formats and prefixing: Stripe `sk_live_`/`sk_test_`/`pk_…` and test/live separation; verify GitHub's prefixed tokens (`ghp_`, `gho_`, fine-grained tokens); Shopify `shpat_…`**; Twilio's Account SID + Auth Token basic auth; key rotation/management endpoints; 401-vs-403 usage in practice; **AWS SigV4** as the signed-request contrast and what it costs/buys.
7. **Spec & docs practice** — official OpenAPI publication (verify: Stripe's and GitHub's OpenAPI repos; Azure's azure-rest-api-specs; Google's discovery docs/proto sources); changelog practice; versioned reference docs; docs-generation pipelines where publicly known.

## Quality bar

- Primary sources first (official docs, guideline repos, first-party engineering posts — e.g., Stripe's API-versioning essay); secondary only as corroboration.
- **Currency:** note retrieval dates; version-window and rate-limit numbers change often — date them.
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
3. **Baseline position** — RFC 9111, `Deprecation`/`Sunset`/`RateLimit-*` status, and the guideline docs on THIS surface
4. **Side-by-side comparison tables** across all eight (versioning; deprecation; rate limits; caching; auth surface; docs)
5. **Per-reference notes** — a lifecycle character sketch of each
6. **Agreements vs divergences** — each divergence with tradeoffs, descriptive
7. **CONTESTED AXES REGISTER (scoped to this part)** — one row per contested axis (expect: versioning scheme path-vs-header-vs-account-pinned-dated, breaking-change strictness, new-enum-values-breaking-or-not, sunset signaling, rate-limit header family, quota model, conditional-request support, key-prefix conventions, 401/403 line, signed-requests-vs-bearer). Columns: **Axis · Options observed · Who does what · Tradeoff in one line · How contested** (near-consensus / split / wide-open). Each row self-contained enough to become a decision item directly.
8. **EXAMPLES APPENDIX** — every verbatim payload, header line, and concrete number collected under the specification-grade requirement, grouped by reference
9. **Caveats**
