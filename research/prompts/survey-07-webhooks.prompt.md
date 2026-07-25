# Deep Research Prompt — REST API Conventions Series (Part 7/7): Webhooks

> Part of a 7-part **descriptive** research series feeding a follow-up phase where the requester will define a prescriptive REST API design standard. Each part is self-contained and reports are merged later. This part covers ONLY the webhook surface — API-side reliability is Part 5, rate limits and auth Part 6 — so do not expand scope.

## Scope line

**The exact question:** How do the eight references (and the emerging Standard Webhooks specification) design outbound webhooks — event envelopes, signatures, delivery semantics, verification, and payload philosophy — and on which webhook axes does the field genuinely split?

## Mandate

- **DESCRIPTIVE ONLY.** Document what each reference actually does today; no "the standard should…" statements. Flag conflicts and tradeoffs descriptively; decisions happen later.
- Treat **Stripe as a key reference to be checked against the field — not canonical.** Its webhook design is a major reason it was chosen; establish precisely what it does and how the field compares.
- **Decision-readiness is the quality bar:** capture each divergence precisely enough to decide from without re-research.

## Reference set (compare on this surface)

1. **Stripe** — deep-dive target 2. **GitHub webhooks** 3. **Google** — mostly a **contrast** here: Pub/Sub push as its eventing model, plus any product-level webhooks 4. **Microsoft Graph change notifications** — the subscription-based model 5. **Twilio** — its callback-webhook model (verify the notable divergence: form-encoded callback bodies) 6. **Shopify webhooks** 7. **Zalando** — its event guidelines within the API guidelines, if applicable 8. **AWS** — **contrast**: SNS/EventBridge instead of classic webhooks; document what that alternative model implies.

**Additional baseline for this part:** the **Standard Webhooks specification** (standardwebhooks.com, Svix-initiated) — treat it as the emerging clig.dev-analog for webhooks: document what it specifies (signature scheme, headers, envelope guidance) and its adoption level.

## Out of scope (entire series)

OAuth/OIDC internals; GraphQL/gRPC; gateway/infra; SDK design; event-streaming platforms as such (Pub/Sub, SNS, EventBridge appear only as contrast models to classic webhooks).

## Surface to research

1. **Event envelope** — full shape per reference: **Stripe's event object (`id` `evt_…`, `type`, `data.object`, `data.previous_attributes`, `created`, `livemode`, `api_version`, `pending_webhooks`, `request`)**; GitHub's payload + `X-GitHub-Event`/`X-GitHub-Delivery` headers and `action` field; Shopify's `X-Shopify-Topic` + payload; Graph's notification envelope (`subscriptionId`, `changeType`, `resource`, `clientState`); Twilio's callback parameters.
2. **Event naming** — `resource.verb` dot-notation (Stripe `invoice.paid`, past tense) vs header-event + `action` (GitHub) vs topic strings (Shopify `orders/create`) vs changeType enums (Graph); tense and casing conventions.
3. **Signatures & authenticity** — scheme per reference, in mechanism detail: **Stripe `Stripe-Signature` (t= timestamp + v1= HMAC-SHA256, tolerance window, replay protection)**; **GitHub `X-Hub-Signature-256` (HMAC-SHA256, and the legacy SHA-1 header)**; **Shopify's base64 HMAC (`X-Shopify-Hmac-Sha256`)**; **Twilio's `X-Twilio-Signature` scheme**; Graph's `clientState`+`validationToken` approach; **the Standard Webhooks signature scheme (`webhook-id`/`webhook-timestamp`/`webhook-signature`)**; secret rotation support; mTLS or IP-allowlist options where offered.
4. **Delivery semantics** — at-least-once posture; **retry policies with numbers (verify: Stripe's exponential backoff up to ~3 days; Shopify's reported 19 attempts over 48h; GitHub's redelivery model)**; ordering guarantees (verify the common position: none); duplicate delivery and consumer-side idempotency guidance (consumers dedupe by event `id`); timeout expectations on the receiving endpoint; what response codes count as success; manual redelivery tooling.
5. **Endpoint verification & subscription lifecycle** — handshakes: **Graph's `validationToken` echo requirement**; challenge-response patterns elsewhere; subscription creation/expiry/renewal (Graph's expiring subscriptions); endpoint management APIs (Stripe webhook endpoints API); event-type filtering at subscription time.
6. **Thin vs fat payloads** — full-object payloads (Stripe classic `data.object`) vs notification-only requiring a fetch (Graph's default, and **verify Stripe's newer "thin events" offering**); the security/consistency tradeoffs each cites; fetch-on-receipt patterns.
7. **Event versioning & evolution** — how event schemas version (Stripe's `api_version` on events; pinning per endpoint); event catalogs/documentation; typed event catalogs (OpenAPI/AsyncAPI usage for events, if any).
8. **Operational conventions** — dead-lettering or failure notification to the developer; event retention/replay windows (Stripe's events API as a reconciliation source — verify retention); recommended consumer architecture each documents (queue-first, respond-200-fast).

## Quality bar

- Primary sources first (official webhook docs, standardwebhooks.com spec, first-party engineering posts); secondary only as corroboration.
- **Currency:** note retrieval dates; retry numbers and thin-event availability change — date them.
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
3. **Baseline position** — the Standard Webhooks spec, plus anything relevant from HTTP semantics, on THIS surface
4. **Side-by-side comparison tables** (envelope; naming; signatures; delivery/retry; verification; payload philosophy)
5. **Per-reference notes** — a webhook character sketch of each, with the Stripe deep-dive as its own subsection and Google/AWS explicitly framed as alternative-model contrasts
6. **Agreements vs divergences** — each divergence with tradeoffs, descriptive
7. **CONTESTED AXES REGISTER (scoped to this part)** — one row per contested axis (expect: envelope shape, naming convention, signature scheme, timestamp/replay protection, retry policy shape, ordering posture, thin-vs-fat payloads, verification handshake, event versioning, retention/replay). Columns: **Axis · Options observed · Who does what · Tradeoff in one line · How contested** (near-consensus / split / wide-open). Each row self-contained enough to become a decision item directly.
8. **EXAMPLES APPENDIX** — every verbatim payload, header line, and concrete number collected under the specification-grade requirement, grouped by reference
9. **Caveats**
