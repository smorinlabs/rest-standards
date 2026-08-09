# Decisions — baseline-01 (HTTP semantics)

*Gate C ratification record. One entry per decided principle; entries are
appended as the walkthrough proceeds. A `baseline` report proposes; only this
file ratifies. Classification per `PLAN.md` Phase 2: protocol requirement ·
evidence-backed default · project policy · exception · unresolved question.*

---

## Structural lock — Resource orientation

**Decision (2026-08-09): RATIFIED as MUST.** The standard mandates a
resource-oriented surface: noun resources, hierarchy expressed in path
segments, standard HTTP verbs carrying their RFC 9110 semantics. Operations
that resist CRUD are modeled as a **POST action sub-resource**; the concrete
syntax (`:verb` vs sub-path) is a separate Gate C item, not settled here.

**Classification:** evidence-backed default, reinforced by a protocol
requirement at its boundary (RFC 9205 §4.4 forbids overlaying generic
semantics on HTTP methods; ratified alongside the already-proposed `HS-007`
ban on method tunneling via `?_method=` or action-in-body dispatch).

**Justification:**

1. `[COMPARATIVE]` 7 of 8 surveyed references are resource-oriented (Stripe,
   GitHub, Google/AIP, Azure, Graph, Twilio, Shopify, Zalando guideline).
   The single split — AWS control-plane RPC (`POST /` + `Action=`) — is
   explained by a platform-scale constraint (uniform codegen and signing
   across hundreds of services) this standard's scope does not have, and
   AWS itself is resource-oriented where that constraint relaxes (S3).
2. `[FACT]` RFC 9205 §4.4 (BCP 56) forbids RPC-style method overlay;
   `baseline-01`'s threat table documents the failure mode (RPC-shaped
   proliferation defeats caching, method-based routing, and intermediary
   retry).

**Options declined:** RPC carve-out for designated service classes (no
in-scope consumer has the constraint; invites the proliferation `HS-007`
exists to stop) · no rule (forfeits the structural lock that pluralization,
path depth, and action syntax depend on).

**Confidence: high.** Fork honesty: essentially a false fork — vendor
consensus and IETF guidance point the same way; the genuine forks live
downstream (pluralization, casing, version placement, action syntax).

**Evidence:** `survey-02` (orientation table, AWS boundary analysis) ·
`baseline-01` §6 (threat table), HS-007 row, "Actions that resist CRUD"
default.

---

## Structural lock — Pluralization

**Decision (2026-08-09): RATIFIED as MUST.** Collection resources use
**plural nouns** (`/customers`, `/orders`); **exception:** singleton and
configuration resources that model exactly-one-per-context (the GitHub
`/user` / Microsoft Graph `me` pattern) use the singular and MUST be
documented as singletons. Irregular nouns: pick one form per resource and
keep it consistent everywhere the stem appears.

**Classification:** project policy, grounded in a near-universal
evidence-backed convention.

**Justification:** `[COMPARATIVE]` plural collections are near-universal
across the surveyed field — Google AIP mandates plural collection
identifiers; Zalando mandates "Pluralize Resource Names"; Azure, Graph,
GitHub, Stripe, Twilio, Shopify all ship plural. The only systematic
exceptions are singleton/config resources, which every mandating guideline
also carves out. The value of the rule is uniformity itself; divergence has
no documented upside.

**Options declined:** SHOULD (permits divergence on a pure convention) ·
MUST-singular (no surveyed vendor or guideline mandates it).

**Confidence: high.** Fork honesty: a convention lock, not a genuine fork.

**Evidence:** `survey-02` finding 2 and orientation table · `baseline-01`
§8.3 (listed as organization policy for Gate C).
