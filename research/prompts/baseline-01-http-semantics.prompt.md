Conduct a deep, current best-practice survey to determine the normative HTTP semantics and resource-modeling principles for a new public, implementation-neutral REST-over-HTTP design standard.

Exact scope: REST architectural constraints; resource identity and the resource/representation distinction; URI and resource modeling for individual resources, collections, subresources, relationships, and actions; method selection and method properties such as safety and idempotence; status-code selection and redirects; content negotiation and media types at the HTTP layer; conditional requests, validators, preconditions, optimistic concurrency, and range requests; and caching, freshness, validation, invalidation, and intermediary behavior.

Do not decide JSON field/envelope conventions, error object extensions, pagination or filter syntax, OpenAPI/schema lifecycle, bulk or asynchronous job shapes, authentication/authorization, rate-limit policy, retry algorithms, deprecation operations, observability, webhook delivery, or application-level Idempotency-Key contracts. Where an edge touches those subjects, state the dependency and stop at the HTTP-semantic boundary.

Context and constraints:

- The output will inform a durable public standards document, not a particular implementation.
- There is no prescribed language, framework, cloud, or HTTP transport version.
- Clients and services may evolve independently and may communicate through caches, gateways, proxies, and other intermediaries.
- Prefer interoperability, correctness, predictable client behavior, and faithful use of HTTP over organization-specific aesthetics.
- Distinguish REST architectural constraints, binding HTTP requirements, registered protocol elements, widely deployed conventions, and vendor-specific style choices.
- Discuss GraphQL, RPC, streaming, or messaging only where needed to clarify this standard's boundary or an HTTP interoperability point.

Research requirements:

1. Begin with current primary and authoritative sources. At minimum, verify the current status, updates, obsoletions, and applicable errata for Roy Fielding's REST dissertation, RFC 9110, RFC 9111, RFC 3986, RFC 8288, and relevant IANA registries. Expand to other current IETF/W3C/IANA specifications only where they directly bear on the question.
2. Use official vendor API guidelines only as comparative evidence of deployed practice, never as a substitute for a protocol specification. Use secondary commentary only when necessary to explain implementation experience, label it as secondary, and do not base a normative recommendation on an uncorroborated secondary source.
3. For every material claim, provide a direct URL to the supporting source and identify its authority and currency: published standard/current registry, active draft, obsolete document, official vendor convention, or secondary evidence. Include publication/update dates, version identifiers, and access dates where available. Label sourced facts, inferences, recommendations, and proposed project policy distinctly.
4. Explicitly surface source conflicts. When sources disagree, determine whether the disagreement is a true protocol conflict, a difference in scope, a historical change, or a policy choice. State which source should govern and why. Do not silently average or choose among incompatible recommendations.
5. Identify established conventions and canonical patterns, then identify common anti-patterns and their concrete failure modes. Include at least: RPC-shaped action proliferation, method tunneling, unsafe behavior behind safe methods, indiscriminate 200 responses, invented status semantics, identifier leakage into mutable paths, weak or missing validators, lost updates, cache disabling by default, incorrect `Vary` use, and assumptions that intermediaries are absent. Correct or reject any item that evidence does not support.
6. Address contested areas without forcing false consensus: nouns and pluralization, path depth, trailing slashes, actions that do not map cleanly to CRUD, PUT versus PATCH, DELETE response behavior, 404 versus 410, 409 versus 422, 202 semantics, redirects, link relations, and cache behavior for authenticated or mutable resources.
7. State all assumptions about API audience, trust boundaries, latency, intermediaries, client sophistication, and resource lifecycle. Explain which recommendations change when an assumption changes.
8. State a confidence level for each major finding and for the final recommendation, with its basis. Treat unresolved conflict, sparse evidence, or reliance on drafts as lower confidence.

Output:

- An executive recommendation naming the concrete set of HTTP/resource-modeling principles to design against.
- A source-and-currency matrix with direct URLs and authority classification.
- A conventions/patterns section and a separate anti-patterns section.
- A conflict-and-open-questions section that distinguishes research-resolvable questions from organization policy choices.
- A proposed normative-principles table. Give every principle a stable provisional ID, `MUST`/`SHOULD`/`MAY` strength, concise rule text, rationale, applicability or exceptions, evidence URLs, and confidence. The set must be concrete enough to seed a standards document rather than merely listing options.
- A short dependency handoff listing questions that must be resolved by the API-contracts or operational-practice research without attempting to answer them here.
- A final overall confidence statement and a list of assumptions that would invalidate or materially change the recommendation.

The research is complete only if it recommends an actionable, internally coherent normative baseline and explains why rejected alternatives or common conventions should not be adopted universally.
