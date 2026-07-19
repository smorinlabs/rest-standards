# Research Constraints

## Deliverable

A public, language-agnostic, normative standard for HTTP/REST API design. It must
serve API designers, implementers, and reviewers without depending on a specific
framework, programming language, cloud, or vendor.

## Evidence and currency

- Prefer current primary sources: IETF RFCs and drafts, IANA registries, OpenAPI
  Initiative specifications, JSON Schema specifications, and official security
  standards.
- Use official vendor API guidelines only as comparative evidence of established
  practice, never as protocol authority.
- Record direct URLs, document versions, publication or update dates, and access
  dates when available.
- Identify superseded documents and material disagreements between authoritative
  sources.
- Label inferences, recommendations, and project policy separately from facts.
- Do not treat search-result snippets, unsourced summaries, or popularity as proof.

## Research shape

- Default research budget: three independent landscape threads.
- Current threads: HTTP semantics, API contracts, and operational practice.
- Maximum decomposition depth: three levels below a landscape prompt.
- A follow-up thread must answer a narrower unresolved question and add new
  decision value; synthesis alone is not a separate research thread.
- Do not draft the normative standard during research execution.

## Scope boundaries

Include HTTP semantics, resource design, representations, API contracts, lifecycle,
security, and operational behavior. Discuss GraphQL, RPC, streaming, or messaging
only to clarify boundaries or interoperability. Do not prescribe server frameworks,
client libraries, deployment platforms, internal code architecture, or organization
structure.

## Security posture

Assume APIs may be public, accept untrusted input, cross network trust boundaries,
and handle sensitive data. Research must surface authentication, authorization,
data exposure, replay, injection, abuse, rate-limit, retry, webhook, and observability
risks relevant to API design. It must not invent regulatory requirements.

## Output expectations

Each research report must:

- separate protocol requirements from recommendations and policy choices;
- include an executive summary, evidence table, conflicts, anti-patterns, and
  candidate principles;
- report confidence and unresolved questions;
- provide enough source detail for a reviewer to reproduce the work; and
- avoid copying large passages when a precise paraphrase and citation will do.
