Step back from `OP-016`'s mechanism question and validate its foundation:
what is webhook signing FOR, and how do the surveyed vendors' actual schemes
fare against that purpose?

Raised at Gate C by the project owner before ratifying `OP-016`: "even if
the eight don't use RFC 9421, what do the eight do for webhooks, and what is
the purpose of this? I believe it's security, but please validate."
`survey-07` documents the mechanics; this leaf validates the purpose and
evaluates the schemes.

Questions:

1. The purpose, from primary sources: what do provider security docs
   themselves state webhook signing is for? Purely security
   (authenticity/integrity/replay) or also operational? Confirm or refute
   "the purpose is security."
2. The threat model, enumerated: for a consumer exposing an HTTPS endpoint —
   forgery, tampering, replay, cross-endpoint replay, unsigned metadata,
   timing attacks, algorithm weakness, secret leakage, SSRF, verification
   skipped due to friction. Which threats a body-HMAC stops, which need a
   signed timestamp, which need other measures. Documented incidents and
   CVEs preferred.
3. Scheme-vs-threat matrix: Stripe, GitHub, Shopify, Twilio, Microsoft
   Graph, AWS SNS, Google Pub/Sub, Standard Webhooks — protected / partial /
   unprotected per threat, with the specific gap.
4. What security reviewers and practitioners criticize about each scheme.
5. Does RFC 9421 actually address the threat model better than a
   well-designed HMAC with signed timestamp — what it adds, what it costs.

Evidence rules as in `CONSTRAINTS.md`: 2+ sources per material claim,
primary preferred; label claims; date findings; verbatim quotes for
load-bearing text; report absences explicitly (including "no documented
incident found").

Output: purpose validation with quotes; the threat model; the matrix;
criticisms; the RFC 9421 delta; and "Net input for OP-016" — the invariants
any signing scheme must satisfy, independent of which concrete scheme
carries them.
