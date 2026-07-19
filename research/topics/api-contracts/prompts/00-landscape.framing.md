# Framing: API representations and contracts

Date framed: 2026-07-18

## Trigger and question

This is a best-practice/domain-pattern survey for a new, public, implementation-neutral REST-over-HTTP design standard. The research question is: which current representation and contract conventions should become normative guidance for JSON APIs, machine-readable contracts, error details, collections and queries, compatibility/versioning, idempotency keys, bulk operations, and asynchronous workflows?

The deliverable is guidance for a standards document, not code for a particular runtime. There is no application stack or framework constraint. The target is externally consumable HTTP APIs whose contracts should be readable by people, tooling, generated clients, and conformance tests.

## Light-search findings

- The official OpenAPI site now lists 3.2.0 as a published specification and also maintains 3.1 and 3.0 lines. "Use the latest" is not automatically a sufficient recommendation because tooling support and JSON Schema alignment may differ. Deep research must recommend a baseline or conditional policy rather than assume a version.
- RFC 9457 is the current standards-track Problem Details specification and obsoletes RFC 7807. Existing guidance that still names RFC 7807 needs explicit migration/currency handling.
- RFC 6570 and RFC 8288 provide standard building blocks for URI templates and links, but collection pagination, filtering, sorting, sparse fields, and search syntax remain areas where major official API guidelines differ.
- RFC 7240 supplies the `Prefer` framework, including asynchronous handling building blocks, while HTTP 202 supplies only noncommittal acceptance semantics. A complete asynchronous-operation contract requires additional resource and lifecycle decisions.
- The IETF Idempotency-Key work reached draft-07 but is currently expired/archived as of this framing date. Its patterns are deployed by vendors, but it must be labeled as work in progress rather than cited as a current RFC.
- OpenAPI, JSON Schema, Problem Details, vendor API design guides, and formats such as JSON:API answer different layers of the contract problem. The research must not imply that adopting one eliminates decisions at the other layers.

## Chosen constraints

- Prefer current published specifications and official registries; label active or expired Internet-Drafts and assess them separately from standards.
- Keep recommendations language-, framework-, ORM-, and vendor-neutral.
- Treat OpenAPI as a contract-description candidate whose version and source-of-truth workflow require an explicit recommendation.
- Produce rules that can be represented in documentation and, where practical, linted or tested.
- Preserve the distinction between HTTP method idempotence and application-level retry deduplication. This thread owns the latter, including idempotency keys.
- Treat API compatibility/versioning policy here; operational communication of deprecation and sunset dates belongs to the operational-practice thread.
- Treat the body and lifecycle contract for async work here; webhook delivery mechanics and security belong to the operational-practice thread.

## Boundaries

In scope:

- JSON representation structure and naming, presence/nullability, dates/times, numbers, identifiers, enums, links, metadata, and extensibility
- contract-first/code-first/source-of-truth choices, OpenAPI and JSON Schema version policy, linting and compatibility checks
- Problem Details and validation/error extensions
- collection representations, pagination, filtering, sorting, searching, field selection, and query encoding
- compatibility, additive versus breaking change, versioning, and evolution
- application-level idempotency keys and replay/deduplication contracts
- bulk requests and partial outcomes
- asynchronous operations, operation resources, polling, cancellation, and result/error lifecycle

Out of scope:

- the protocol meaning of resources, methods, status codes, conditional requests, and caching
- authentication/authorization mechanisms and sensitive-data security policy
- rate limiting, client retry/backoff algorithms, deprecation/sunset communication, telemetry, SLIs/SLOs, and webhook signing/delivery
- implementation framework or language bindings

## Source seeds

These sources were checked only to frame the field; the deep-research run must verify currency and expand the set.

- OpenAPI specification index and current versions: https://spec.openapis.org/oas/
- OpenAPI latest published version: https://spec.openapis.org/oas/latest.html
- JSON Schema 2020-12: https://json-schema.org/draft/2020-12
- RFC 9457, Problem Details for HTTP APIs: https://www.rfc-editor.org/rfc/rfc9457.html
- RFC 6570, URI Template: https://www.rfc-editor.org/rfc/rfc6570.html
- RFC 7240, Prefer Header for HTTP: https://www.rfc-editor.org/rfc/rfc7240.html
- IETF Idempotency-Key draft status: https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/
- Google Cloud API Design Guide: https://cloud.google.com/apis/design
- Microsoft REST API Guidelines: https://github.com/microsoft/api-guidelines

## Why this is one of three sufficient threads

The proposed standard has three distinct evidence domains: (1) protocol semantics and resource modeling, (2) application representation and contract design, and (3) security and operational lifecycle. This thread contains the choices that must remain internally consistent across schemas, examples, generated clients, and compatibility checks. Giving them one owner avoids contradictory local choices for pagination, errors, versioning, idempotency, bulk, and asynchronous work. All remaining subjects have a clear owner in one of the other two threads, so a fourth landscape thread would add overlap rather than coverage unless this survey identifies a focused unresolved decision.
