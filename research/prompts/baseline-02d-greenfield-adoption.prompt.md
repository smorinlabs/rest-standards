Extend leaf `baseline-02c` before ratifying `AC-003` at Gate C: widen the
evidence base from framework defaults to the broader field of newly built
(roughly 2023+) APIs, and stress-test the standard itself.

`baseline-02c` raised `AC-003`'s confidence to high-moderate partly on a
`[FACT — absence]`: no documented case of a greenfield API evaluating RFC 9457
and rejecting it on the merits. That absence claim is only as strong as the
search behind it. Older APIs cannot cheaply change error shapes, so **new**
APIs are the population that matters; and a mandate should not issue without
weighing the standard's known criticisms and any credible alternative.

Dimensions to research, via web search:

1. What new (2023+) APIs and API standardization efforts are doing for error
   responses — commercial, open-source, and government (US/UK/EU/AU
   guidelines).
2. Who has adopted RFC 9457 / RFC 7807 — companies, frameworks beyond
   ASP.NET Core and Spring (FastAPI, Django, Rails, Go, Node, Laravel),
   gateways, and industry or national API guidelines. Distinguish
   **mandates** from **recommends** from **supports**.
3. Practitioner opinion since ~2023 on adopting it.
4. Criticisms of the standard — concrete design complaints — and whether a
   credible alternative format is gaining traction (JSON:API errors,
   `google.rpc.Status`, bespoke envelopes).
5. Practical adoption signals: client-library parsing, OpenAPI tooling,
   gateway and observability integration.

Evidence rules as in `CONSTRAINTS.md`: two or more sources per material
claim, primary preferred; label `[FACT]` / `[COMPARATIVE]` / `[INFERENCE]` /
`[OPINION]`; date findings; report absence honestly. Actively re-test
`baseline-02c`'s absence claim.

Output: findings per dimension with sources; a net assessment of whether the
evidence strengthens, weakens, or leaves unchanged the case for mandating
RFC 9457 in a greenfield standard; and whether any alternative deserves
consideration.
