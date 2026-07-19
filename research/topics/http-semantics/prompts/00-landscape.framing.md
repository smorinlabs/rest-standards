# Framing: HTTP semantics and resource modeling

Date framed: 2026-07-18

## Trigger and question

This is a best-practice/domain-pattern survey for a new, public, implementation-neutral REST-over-HTTP design standard. The research question is: which current HTTP and REST principles should become normative guidance for resource identity and modeling, URI design, methods, status codes, conditional requests, content negotiation, and caching?

The deliverable is guidance for a standards document, not code for a particular runtime. There is therefore no application stack, framework, cloud, or language constraint. The target platform is interoperable HTTP APIs used by independently evolving clients and services.

## Light-search findings

- RFC 9110 is the current standards-track foundation for HTTP semantics across HTTP versions. It defines resources, representations, method properties, status codes, content negotiation, validators, and conditional requests. It obsoletes the older RFC 7231/7232/7233 semantics set, so research should not use those older RFCs as the current authority.
- RFC 9111 separately defines HTTP caching. Cacheability, freshness, validation, invalidation, authenticated responses, and intermediary behavior need to be treated as protocol behavior rather than an optional performance appendix.
- Roy Fielding's REST dissertation is the primary architectural source for REST constraints, but it is not a detailed HTTP API style guide. The research needs to distinguish REST architectural constraints from conventions that vendors later called "RESTful."
- RFC 3986 defines URI generic syntax; RFC 8288 defines Web Linking. Neither prescribes one universal noun/plural/path naming style. URI spelling conventions are policy choices unless stronger interoperability evidence exists.
- Official vendor guidelines commonly add organization-specific resource naming and action conventions. Those are useful evidence of deployed practice, but they must not override HTTP RFCs or be presented as universal protocol requirements.

## Chosen constraints

- Start from current published standards and their errata/status pages; explicitly identify obsolete specifications.
- Keep the result language-, framework-, transport-version-, and vendor-neutral.
- Separate protocol requirements from broadly useful conventions and organization-specific taste.
- Produce principles suitable for normative `MUST`, `SHOULD`, and `MAY` language, but do not manufacture a mandate where the standards or evidence support multiple valid choices.
- Treat safe/idempotent method semantics as distinct from application-level idempotency-key contracts. This thread owns the former only.
- Treat representations only as HTTP's representation model and content negotiation. JSON field shape, error object shape, schema contracts, and contract lifecycle belong to the API-contracts thread.

## Boundaries

In scope:

- REST constraints and the limits of the term "REST"
- resource identity, resource versus representation, URI/resource modeling, subresources, collections, relationships, and actions
- method selection and semantics, including safety and idempotence
- status-code selection and redirects
- conditional requests, validators, optimistic concurrency, ranges, and preconditions
- content negotiation and media types at the HTTP layer
- caching, freshness, validation, invalidation, and intermediary implications

Out of scope:

- JSON naming and envelopes, problem-detail extensions, pagination/filter syntax, OpenAPI, schema evolution, bulk and asynchronous job contracts
- authentication and authorization mechanisms, rate-limit policy, retry algorithms, deprecation operations, observability, and webhook delivery
- application-level `Idempotency-Key` formats and retention policy

## Source seeds

These sources were checked only to frame the field; the deep-research run must verify currency and expand the set.

- Roy Fielding, REST architectural style: https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm
- RFC 9110, HTTP Semantics: https://www.rfc-editor.org/rfc/rfc9110.html
- RFC 9111, HTTP Caching: https://www.rfc-editor.org/rfc/rfc9111.html
- RFC 3986, URI Generic Syntax: https://www.rfc-editor.org/rfc/rfc3986.html
- RFC 8288, Web Linking: https://www.rfc-editor.org/rfc/rfc8288.html
- IANA HTTP status-code registry: https://www.iana.org/assignments/http-status-codes/http-status-codes.xhtml

## Why this is one of three sufficient threads

The proposed standard has three distinct evidence domains: (1) protocol semantics and resource modeling, (2) application representation and contract design, and (3) security and operational lifecycle. This split follows authority boundaries and prevents one oversized survey from flattening protocol law, schema conventions, and production policy into a single confidence level. All proposed topics fit one primary owner, with explicit handoffs for unavoidable edges such as native method idempotence versus idempotency keys, and HTTP validators versus representation schemas. A fourth landscape thread is not justified until one of these surveys finds a specific unresolved question that needs a narrower evaluation pass.
