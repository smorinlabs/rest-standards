Research the five risk-based security axes `baseline-03` §8.2 classified
as "changing with threat model rather than resolvable by evidence", so
Gate C can ratify a RECOMMENDED DEFAULT for each plus the named
threat-model triggers that change it — a deployment-profile structure,
decided now per the owner's direction.

The five axes, for each: the real options; what published standards say
(BCP 240 / RFC 9700, RFC 9449 DPoP, RFC 8705 mTLS, RFC 9068 JWT access
tokens, OWASP API Security Top 10 2023, FAPI 2.0, anything newer); what
major vendors ship — including OpenAI, Anthropic, and Google Gemini
wherever discoverable; a recommended default for a typical public SaaS
API; and the specific triggers that flip it:

1. Sender-constrained tokens — bearer vs DPoP vs mTLS-bound.
2. Opaque vs self-contained (JWT) access tokens — introspection vs
   revocation latency; phantom-token pattern; gateway support.
3. Rate-limit aggressiveness — posture, not header format: per-class
   defaults, burst vs sustained, auth-endpoint tiers, published vendor
   numbers.
4. Replay-window length — beyond the ratified webhook 5-minute
   convention: signed-request windows, clock-skew handling, dedup.
5. Centralized object-level authorization — OWASP API1:2023 BOLA;
   gateway vs middleware vs policy engine (OPA, Cedar, Zanzibar lineage);
   what the engines' own docs admit about staleness and dependencies.

Evidence rules as in `CONSTRAINTS.md`, plus: re-verify every load-bearing
normative quote against raw text (summarizer-mediated quotes are
untrusted); report absences and retrieval gaps explicitly.

Output: per axis — options, standards positions with verbatim quotes,
vendor practice table including the three AI platforms, recommended
default with confidence, and explicit flip triggers; closing with a
compact deployment-profile skeleton table (axis | default | flip
triggers) ready for a decision record.
