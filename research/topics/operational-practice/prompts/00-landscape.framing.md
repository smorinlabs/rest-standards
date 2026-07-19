# Framing: Security and operational practice

Date framed: 2026-07-18

## Trigger and question

This is a best-practice/domain-pattern survey for a new, public, implementation-neutral REST-over-HTTP design standard. The research question is: which current security, resilience, lifecycle, observability, and webhook principles should become normative guidance for production APIs, and which official ecosystem conventions are mature enough to standardize?

The deliverable is guidance for a standards document, not a product deployment. There is no application stack or cloud constraint. Security is a binding domain constraint because the standard will cover public or cross-trust-boundary APIs handling untrusted requests, credentials, identifiers, and potentially sensitive data.

## Light-search findings

- RFC 9700 is the current Best Current Practice for OAuth 2.0 security and updates or deprecates several older deployment patterns. Merely citing RFC 6749 is not enough for current security advice.
- OWASP API Security Top 10 2023 is current awareness guidance, but its own methodology says the update did not obtain data sufficient for statistical analysis. It is useful as a threat checklist, not as a protocol authority or prevalence ranking.
- W3C Trace Context standardizes `traceparent` and `tracestate` propagation. OpenTelemetry provides a broader official telemetry specification, but observability policy still needs choices about correlation, privacy, metrics, logs, and externally exposed identifiers.
- RFC 9745 now standardizes the `Deprecation` response header and explicitly composes with RFC 8594 `Sunset`. Older custom-header advice should be tested against these current standards.
- The IETF RateLimit header work is an active Internet-Draft (`draft-ietf-httpapi-ratelimit-headers-11` at framing time), not a published RFC. Vendor APIs still use divergent rate-limit headers, so the research must distinguish future-facing draft alignment from current interoperable guarantees.
- Official GitHub and Stripe webhook guidance converges on secrets/signatures, replay protection, quick acknowledgement, asynchronous processing, unique delivery IDs, retries, and deduplication, but header formats and delivery guarantees remain vendor-specific. RFC 9421 HTTP Message Signatures may provide a general building block and needs evaluation rather than automatic adoption.
- Google, Microsoft/Azure, and GitHub publish official API guidance, but their architecture and product constraints differ. Their operational recommendations are comparative evidence, not a single consensus standard.

## Chosen constraints

- Base security recommendations on current BCPs, standards, and threat guidance, with an explicit currency check.
- Keep identity-provider, cloud, language, framework, gateway, and observability-vendor choices out of the normative baseline.
- Separate externally visible API contract requirements from internal implementation controls while covering both when public behavior depends on them.
- Label all drafts and proprietary conventions. A common vendor header is not automatically a standard.
- Treat client retry/backoff and service throttling here; application-level idempotency-key semantics belong to the API-contracts thread.
- Treat lifecycle communication and enforcement here; compatibility/versioning taxonomy belongs to the API-contracts thread.
- Treat webhook delivery and security here; asynchronous job resource shapes belong to the API-contracts thread.

## Boundaries

In scope:

- TLS and transport security posture; authentication and authorization patterns; OAuth/OIDC and token handling; API keys where appropriate
- tenant/object/function authorization, least privilege, abuse resistance, resource limits, input/output exposure, privacy, and security errors
- rate limiting, quota communication, overload behavior, retries, backoff, jitter, timeouts, circuit-breaking implications, and retry safety dependencies
- version rollout operations, deprecation, sunset, migration communication, and support windows
- observability and diagnostics exposed at the boundary: request/correlation IDs, trace context, metrics, logs, audit events, privacy, and supportability
- webhook/event delivery, signing, verification, replay protection, ordering, duplication, retries, acknowledgement, and versioning operations
- comparative operational/security evidence from official Google, Microsoft/Azure, GitHub, Stripe, and similarly authoritative vendor guidance

Out of scope:

- general URI/resource modeling, method and status-code semantics, validators, and caching policy
- JSON representation conventions, error schema extensions, pagination/filter syntax, OpenAPI source-of-truth policy, compatibility taxonomy, idempotency-key retention/response semantics, bulk contracts, and asynchronous job resource shapes
- selection of a specific identity provider, API gateway, tracing vendor, or deployment platform

## Source seeds

These sources were checked only to frame the field; the deep-research run must verify currency and expand the set.

- RFC 9700, Best Current Practice for OAuth 2.0 Security: https://www.rfc-editor.org/rfc/rfc9700.html
- OAuth 2.0 Authorization Framework: https://www.rfc-editor.org/rfc/rfc6749.html
- OpenID Connect Core 1.0: https://openid.net/specs/openid-connect-core-1_0.html
- OWASP API Security Top 10 2023: https://owasp.org/API-Security/editions/2023/en/0x00-header/
- OWASP methodology and data caveat: https://owasp.org/API-Security/editions/2023/en/0xd0-about-data/
- RFC 9745, Deprecation header: https://www.rfc-editor.org/rfc/rfc9745.html
- RFC 8594, Sunset header: https://www.rfc-editor.org/rfc/rfc8594.html
- Current IETF RateLimit draft status: https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/
- W3C Trace Context: https://www.w3.org/TR/trace-context/
- OpenTelemetry specifications: https://opentelemetry.io/docs/specs/otel/
- RFC 9421, HTTP Message Signatures: https://www.rfc-editor.org/rfc/rfc9421.html
- GitHub webhook best practices: https://docs.github.com/en/webhooks/using-webhooks/best-practices-for-using-webhooks
- Stripe webhook behavior and security: https://docs.stripe.com/webhooks
- Google Cloud API Design Guide: https://cloud.google.com/apis/design
- Microsoft REST API Guidelines: https://github.com/microsoft/api-guidelines
- GitHub REST API best practices: https://docs.github.com/en/rest/using-the-rest-api/best-practices-for-using-the-rest-api

## Why this is one of three sufficient threads

The proposed standard has three distinct evidence domains: (1) protocol semantics and resource modeling, (2) application representation and contract design, and (3) security and operational lifecycle. Security, resilience, rollout, observability, and event delivery interact strongly in production and draw on BCPs, threat models, and vendor operations rather than only interface schemas, so they belong in one coordinated survey. The explicit boundary rules assign idempotency contracts, compatibility taxonomy, and asynchronous resource shapes elsewhere while keeping retry, deprecation operations, and webhook transport here. This covers the planned standard without a fourth overlapping landscape thread; any genuinely unresolved specialist question should be a later focused narrowing pass.
