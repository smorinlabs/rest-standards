Conduct a deep, current best-practice survey to determine the normative representation and contract-design principles for a new public, implementation-neutral REST-over-HTTP design standard.

Exact scope: JSON representation structure and naming; presence, omission, nullability, dates/times, numbers, booleans, identifiers, enums, links, metadata, and extensibility; machine-readable contracts and the contract source of truth; OpenAPI and JSON Schema versions, workflows, linting, and compatibility checks; Problem Details and validation-error extensions; collection representations; pagination, filtering, sorting, search, field selection, and query encoding; additive and breaking changes, compatibility, and API versioning; application-level idempotency keys and replay/deduplication behavior; bulk request and partial-result contracts; and asynchronous operation resources, polling, cancellation, completion, expiration, results, and failures.

Do not re-decide the protocol meaning of HTTP resources, methods, status codes, validators, conditional requests, or caching. Do not decide authentication/authorization, sensitive-data security controls, rate-limit policy, client retry/backoff algorithms, deprecation/sunset communication, observability, or webhook delivery and signing. Treat native HTTP method idempotence as an input; this thread owns only application-level idempotency contracts. Treat compatibility/versioning here, but hand operational rollout and deprecation communication to the operational-practice research.

Context and constraints:

- The output will inform a durable public standards document, not a particular implementation.
- There is no prescribed language, framework, ORM, cloud, or persistence model.
- Contracts should support independent client/server evolution, hand-written and generated clients, human documentation, linting, conformance tests, and gateways.
- Recommendations should be practical for public JSON-over-HTTP APIs while clearly marking cases where another media type or established domain format should govern.
- Prefer interoperable standards and explicit lifecycle behavior over proprietary envelopes and undocumented conventions.
- Discuss GraphQL, RPC, streaming, or messaging only where needed to clarify this standard's boundary or an interoperability point.

Research requirements:

1. Begin with current primary and authoritative sources. At minimum, verify the current status and versions of OpenAPI, its JSON Schema relationship, JSON Schema itself, RFC 9457, RFC 6570, RFC 7240, RFC 8288 where links affect representation contracts, and the IETF Idempotency-Key work. Verify whether every IETF item is a published standard, active draft, expired draft, or obsolete document at research time.
2. Compare current official Google, Microsoft/Azure, and GitHub API guidance, plus other official vendor or standards-body guidance only when it provides material evidence. Compare formats such as JSON:API only in the dimensions they actually standardize. Do not treat vendor prevalence or a complete media-type specification as proof that every convention should become universal.
3. For every material claim, provide a direct URL and identify source authority, version, publication/update date, access date where available, and currency. Separate binding standards, official specification choices, official vendor conventions, and secondary implementation evidence. Label sourced facts, inferences, recommendations, and proposed project policy distinctly.
4. Explicitly surface source conflicts. Determine whether each conflict results from different API styles, compatibility goals, tool constraints, historical versions, or organization policy. Recommend which rule should govern this general-purpose standard, make the recommendation conditional when appropriate, and never silently merge incompatible schemes.
5. Identify canonical patterns and anti-patterns with concrete consequences. Include at least: inconsistent envelopes, ambiguous null versus omission, floating-point money, local-time timestamps, unstable enum handling, duplicated error formats, using RFC 7807 as if current, offset pagination on mutable high-volume data without caveats, leaking storage/query syntax, arbitrary version placement, permanent parallel versions, undocumented breaking changes, reused idempotency keys across payloads, unbounded deduplication, non-atomic bulk ambiguity, and async 202 responses with no status resource. Correct or reject any item that evidence does not support.
6. Address contested choices explicitly: camelCase versus snake_case, top-level envelopes, string identifiers, date/time precision, unknown enum values, link placement, cursor versus offset pagination, filter grammars, field masks, version in path/header/media type/host or no version marker, OpenAPI 3.2 versus 3.1/3.0 baselines, design-first versus code-first versus generated single-source workflows, Problem Details extension shapes, atomic versus partial bulk behavior, and operation-resource lifecycle.
7. Evaluate whether one universal convention is defensible for each choice. Where it is not, define a default plus documented exceptions or a conditional recommendation with decision criteria.
8. State assumptions about API audience, data volume and mutation rate, client generation, backward-compatibility horizon, offline/retry behavior, transactionality, and duration of asynchronous work. Explain which recommendations change with those assumptions.
9. State confidence for every major finding and the final recommendation, including the evidence basis. Lower confidence for expired or unsettled drafts, thin tooling evidence, or irreconcilable vendor divergence.

Output:

- An executive recommendation naming the concrete representation and contract principles to design against.
- A source-and-currency matrix with direct URLs, versions/dates, and authority classification.
- A domain map showing which layer is governed by HTTP, media-type specifications, JSON Schema, OpenAPI, shared conventions, or organization policy.
- A conventions/patterns section and a separate anti-patterns section.
- A comparison table for the contested choices, including the recommendation, alternatives, tradeoffs, exceptions, evidence, and confidence.
- A proposed normative-principles table. Give every principle a stable provisional ID, `MUST`/`SHOULD`/`MAY` strength, concise rule text, rationale, applicability or exceptions, evidence URLs, and confidence. The rules must form a concrete, internally coherent baseline that can seed prose, OpenAPI examples, lint rules, and a conformance checklist.
- A conflict-and-open-questions section separating research-resolvable matters from organization policy decisions.
- A short dependency handoff to the HTTP-semantics and operational-practice research without answering those threads' questions.
- A final overall confidence statement and assumptions that would invalidate or materially change the recommendation.

The research is complete only if it recommends an actionable contract baseline, including a version-aware OpenAPI policy and complete error, collection, compatibility, idempotency, bulk, and asynchronous-operation patterns, rather than merely cataloging alternatives.
