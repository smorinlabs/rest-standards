Companion leaf to `baseline-03e` (field survey): establish the IETF
RateLimit draft's real trajectory, implementation ecosystem, and whether a
credible alternative deserves `OP-010`'s mandate instead.

Context: `draft-ietf-httpapi-ratelimit-headers` is at draft-11, expires
2026-11-24, no Last Call in 7 years; the HTTPDIR early review said "Not
Ready" on editorial grounds while judging the technical design sound. The
proposed rule mandates the fields with an expiry contingency.

Questions:

1. Draft mechanics, current revision: exact field syntax; what changed
   across revisions — pin precisely which revision renamed the fields from
   the `RateLimit-{Limit,Remaining,Reset}` trio to the combined structured
   pair, and show old vs new wire format side by side.
2. Process trajectory: datatracker history, WG milestones, WGLC status,
   2026 mailing-list activity, editor statements, the response to the
   HTTPDIR review. Is there any dated path to RFC? What happens procedurally
   at expiry — revival mechanics, and how often this draft has revived.
3. Implementation ecosystem, precisely versioned: who implements WHICH
   revision — Kong, Envoy, Traefik, nginx, Tyk, express-rate-limit's
   `standardHeaders` drafts, Fastify, Flask-Limiter, Laravel, Spring,
   ASP.NET, Go/Rust limiters. The old-vs-new split is the key measurement.
4. Alternative conventions with standardization energy: the `X-RateLimit-*`
   trio and its documented ambiguities; `Retry-After` alone (what RFC 9110
   and RFC 6585 actually make normative for 429); OpenAI-style
   multi-dimension patterns and whether the draft's `qu` parameter covers
   them; anything else with a standards-body work item.
5. The contingency design: evaluate rule shapes — (a) MUST IETF fields now
   with expiry contingency; (b) MUST `Retry-After` on 429 (published
   standards) + SHOULD the pinned IETF fields, upgrade on advancement; (c)
   documented proprietary. Note that per project rules an unpublished draft
   must be labeled [POLICY] and never presented as a published standard.

Evidence rules as in `CONSTRAINTS.md`: 2+ sources per material claim,
primary preferred (datatracker, raw draft/RFC text, plugin source, IANA
registries); label and date claims; verbatim wire-format examples; report
absences explicitly; re-verify any summarizer-mediated quote against raw
text.

Output: findings per question; an implementations-by-revision table; old vs
new wire formats side by side; "Recommendation for OP-010" with proposed
wording, confidence, and exactly how the contingency should be worded given
the revival mechanics.
