Gate C addendum leaf: which PATCH document format should the standard
mandate — RFC 7396 JSON Merge Patch, RFC 6902 JSON Patch, or a plain
partial-JSON body? The CLI-standards gap review identified this as a
dropped handoff: `baseline-01` §9 handed "patch document format" to
`baseline-02`, which answered only the null-vs-omission half (`AC-011`);
neither RFC appears in the decision records while `HS-008` mandates PATCH
for partial modification.

Questions:

1. The formats' semantics from raw RFC text: RFC 5789's negotiation
   surface (`Accept-Patch`, 415, atomicity, conditional-request guidance);
   RFC 7396's null-means-remove pseudocode and array limitation; RFC
   6902's op/path/value model, `test` op, atomicity, JSON Pointer
   escaping.
2. What the field ships, primary-verified: Azure guidelines, Microsoft
   Graph, Google AIP-134 (`update_mask`), Kubernetes (four formats by
   Content-Type), GitHub, Stripe (no PATCH), Shopify, and the three AI
   platforms where discoverable. Which `Content-Type` each accepts.
3. The `AC-011` collision, precisely: under Merge Patch `null` is the
   delete verb, so a semantically-meaningful stored null cannot be set —
   how do real APIs resolve this, and does Merge Patch + AC-011 force a
   specific resolution?
4. Tooling reality: OpenAPI 3.1 expressiveness for dual-format bodies;
   the all-fields-optional validation problem; library support.
5. Recommendation across (a) Merge Patch + null rule, (b) JSON Patch
   required, (c) both by Content-Type, (d) plain partial JSON,
   (e) AIP-134 field masks — with proposed wording, confidence, and what
   flips it.

Evidence rules as in `CONSTRAINTS.md`: raw-text verification for every
normative quote; 2+ sources; label and date claims; report absences and
source conflicts explicitly.

Output: findings per question; a vendor practice table (vendor | PATCH? |
format | Content-Type | null handling); the recommendation with rule
wording and confidence.
