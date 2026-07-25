# Deep Research Prompt — REST API Conventions Series (Part 5/7): Reliability — Idempotency, Concurrency, Async Operations & Bulk

> Part of a 7-part **descriptive** research series feeding a follow-up phase where the requester will define a prescriptive REST API design standard. Each part is self-contained and reports are merged later. This part covers ONLY the surface below — errors are Part 3, rate limiting Part 6, webhooks Part 7 — so do not expand scope.

## Scope line

**The exact question:** How do the eight references make mutations safe to retry, guard concurrent writes, model long-running work, and handle bulk operations — and on which reliability axes does the field genuinely split? **This is the highest-priority part of the series**; idempotency in particular is the design area the requester cares most about.

## Mandate

- **DESCRIPTIVE ONLY.** Document what each reference actually does today; no "the standard should…" statements. Flag conflicts and tradeoffs descriptively; decisions happen later.
- Treat **Stripe as a key reference to be checked against the field — not canonical.** Its idempotency design is famous; establish precisely what it does and how the rest of the field compares.
- **Decision-readiness is the quality bar:** capture each divergence precisely enough to decide from without re-research.

## Reference set (compare all eight on this surface)

1. **Stripe** — deep-dive target for idempotency 2. **GitHub REST** 3. **Google Cloud / AIP (aip.dev)** 4. **Microsoft (Azure REST guidelines + Graph)** 5. **Twilio** 6. **Shopify Admin REST** (flag currency caveats) 7. **Zalando RESTful API Guidelines** (guideline document) 8. **AWS** — light contrast notes (e.g., client tokens on some services).

## Out of scope (entire series)

OAuth/OIDC internals; GraphQL/gRPC; gateway/infra; SDK design; event streaming (webhook *retries* are Part 7; here cover only API-side reliability).

## Surface to research

1. **Stripe idempotency — full mechanics (deep-dive; verify each point against primary docs and Stripe's engineering writing):** the `Idempotency-Key` header; which methods it applies to (POST — and how Stripe's POST-for-updates interacts with it); key scope and uniqueness (per account? per endpoint?); **retention window (widely reported as 24 hours — verify current)**; replay semantics — what exactly is replayed (verify the widely-reported behavior that the *first* response is replayed **including errors**); behavior on concurrent duplicate requests in flight (verify the reported 409/`idempotency_error` behavior); behavior when the same key is reused with a *different* request body; key format recommendations (UUIDs); SDK auto-generation of keys.
2. **The IETF `httpapi` Idempotency-Key draft** — its specified mechanics vs Stripe's; current status (coordinate with Part 1; restate the status here for self-containment).
3. **Idempotency elsewhere** — Google **AIP-155 `request_id`** (field-in-body approach, server dedupe window); **Azure's repeatability headers (`Repeatability-Request-ID`/`Repeatability-First-Sent`, OASIS lineage)** per the Azure REST guidelines; Twilio's idempotency support; Shopify; whether GitHub offers anything; AWS client tokens on selected services.
4. **What APIs without keys do instead** — natural idempotency via PUT; client-generated IDs on create; conditional creates; documented retry guidance.
5. **Concurrency control** — ETag/`If-Match` conditional writes: **verify GitHub's use, Azure's guideline mandate, and Google AIP-154's `etag`-in-body pattern**; `If-Unmodified-Since`; optimistic version fields; which operations *require* preconditions; 412/precondition failure behavior; whether Stripe offers any optimistic concurrency at all.
6. **Async / long-running operations** — **Google AIP-151 Operations** (the `operations/{id}` resource, `done`, `response`/`error`); **Azure's LRO conventions** (202 + `Azure-AsyncOperation`/`Location` headers, `provisioningState`, final-state-via); Microsoft Graph long-running patterns; how Stripe handles long-running work (largely synchronous + webhook events — verify); polling contracts (intervals, `Retry-After` on 202); completion signaling; cancellation endpoints; operation retention/expiry.
7. **Bulk & batch** — **Microsoft Graph `$batch` JSON batching** (request shape, per-item responses, limits); Google's HTTP batch endpoints (verify current status — deprecations reported); Stripe's posture (no generic batch — verify); Zalando's guidance; per-item partial-failure reporting shapes; any 207 Multi-Status usage; atomicity guarantees (all-or-nothing vs independent items); bulk-import/export patterns (e.g., file-based) as a distinct model.

## Quality bar

- Primary sources first: docs.stripe.com and Stripe's engineering blog (their idempotency essay), aip.dev (AIP-151/154/155), the Azure REST API Guidelines repo, learn.microsoft.com, IETF datatracker; secondary only as corroboration.
- **Currency:** note retrieval dates; verify the IETF draft status and any recently changed retention/limit numbers.
- **Confidence** per non-obvious finding, with basis — especially on Stripe replay/concurrency behaviors, where docs and blog posts may differ in precision.
- **Surface disagreements** between sources rather than silently picking one.

## Specification-grade detail requirement

A finding on this surface is complete only when someone could implement or emulate the mechanism from this report **without opening the reference's own docs**. For every mechanism documented:

- **Exact names & formats** — header names with their value syntax, field/parameter names verbatim, media types, enum values.
- **Verbatim examples** — at least one real example per major mechanism: a request/response payload or header line, quoted (minimally elided for length).
- **Concrete numbers** — limits, windows, defaults, maximums — each with its source and retrieval date.

Summaries without these artifacts do not satisfy the deliverable.

## Required deliverable structure

1. **TL;DR** 2. **Key findings** (numbered)
3. **Baseline position** — RFC 9110 idempotent-method semantics, the IETF Idempotency-Key draft, and the guideline docs on THIS surface
4. **Side-by-side comparison tables** across all eight (idempotency mechanisms; concurrency; LRO patterns; batch)
5. **Per-reference notes** — a reliability character sketch of each, with the Stripe deep-dive as its own subsection
6. **Agreements vs divergences** — each divergence with tradeoffs, descriptive
7. **CONTESTED AXES REGISTER (scoped to this part)** — one row per contested axis (expect: idempotency mechanism header-vs-body-field-vs-client-ID, key retention window, replay-of-errors-or-not, same-key-different-body behavior, concurrency ETag-header-vs-body-etag-vs-none, precondition-required-or-optional, LRO pattern operation-resource-vs-header-polling, batch mechanism, atomicity). Columns: **Axis · Options observed · Who does what · Tradeoff in one line · How contested** (near-consensus / split / wide-open). Each row self-contained enough to become a decision item directly.
8. **EXAMPLES APPENDIX** — every verbatim payload, header line, and concrete number collected under the specification-grade requirement, grouped by reference
9. **Caveats**
