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

---

## Structural lock — Path depth

**Decision (2026-08-09): RATIFIED.** Nest a sub-resource only where the
child cannot exist outside the parent; **one sub-resource level is the
norm; hard ceiling of three resources per path**; beyond that, flatten and
relate with query filters (the Stripe `?customer=` pattern).

**Classification:** project policy, grounded in comparative practice.
**Justification:** `[COMPARATIVE]` practice is shallow — Stripe flat +
filters (max ~2 with action), Shopify one level with flat twins, Zalando
caps at 3, AIP discourages depth; GitHub's routine 4+ is the outlier.
**Declined:** strict-flat ceiling of 2 (forbids natural children like
`/orders/{id}/line-items`) · guidance-only (no enforceable line).
**Confidence: moderate-high.**
**Evidence:** `survey-02` finding 8, Table C.

---

## Structural lock — Trailing slash

**Decision (2026-08-09): RATIFIED.** Canonical URIs have **no trailing
slash**; a trailing slash MUST NOT carry semantics or identify a different
resource; a trailing-slash request SHOULD receive **308 Permanent
Redirect** to the canonical form (308 preserves method and body per the
already-settled redirect defaults in `baseline-01`).

**Classification:** project policy; near-unanimous convention.
**Justification:** `[COMPARATIVE]` Zalando: "MUST Avoid Trailing Slashes";
no surveyed vendor uses them. **Declined:** hard 404 (punishes a harmless
slip). **Confidence: high — near-false fork.**
**Evidence:** `survey-02` Zalando rules and Table (trailing-slash column).

---

## Structural lock — Custom-action syntax

**Decision (2026-08-09): RATIFIED.** Non-CRUD operations use the
**sub-path verb** form: `POST /{collection}/{id}/{action}` (e.g.
`POST /payment_intents/{id}/capture`), completing the ratified
POST-action-sub-resource default. **Reserved-word rule:** an action
segment is a documented verb and can never be used as a collection name
under the same parent.

**Classification:** project policy.
**Justification:** `[COMPARATIVE]` sub-path is the majority practice
(Stripe, GitHub, Shopify); `:verb` (AIP-136) is collision-proof but
carries colon-handling variance across routers/tooling, pairs with
camelCase verbs (clashing with the ratified casing), and is a
Google-ecosystem signature. Body-flag dispatch was already excluded by
`HS-007`. **Confidence: moderate.**
**Evidence:** `survey-02` orientation table (actions column) ·
`baseline-01` "Actions that resist CRUD" default.

---

## Structural lock — Path-segment casing (rider, added at ratification)

**Decision (2026-08-09): RATIFIED.** Path segments use **kebab-case**
(`/sales-order-items`), pattern `^[a-z][a-z\-0-9]*$`. Added to the pile as
a rider with owner consent — the body/query snake_case lock (`AC-007`) did
not cover paths.

**Classification:** project policy `[POLICY]` — `survey-02` finding 3:
"genuinely contested — four house styles", no cross-field winner.
**Justification:** kebab-case is the only style with a published
enforceable rule (Zalando), is visually distinct from snake_case
parameters, and produces shift-key-free URLs. **Declined:** snake_case
paths (coin-flip; underscores vanish under link underlines) · leaving it
undecided. **Confidence: moderate — a genuine coin-flip against
snake_case, decided for enforceability.**
**Evidence:** `survey-02` findings 3 and Zalando naming rules.

---

## DELETE response

**Decision (2026-08-09): RATIFIED.** A successful DELETE returns **204 No
Content** with an empty body. **Named exception:** an API that soft-deletes
(marks deleted but keeps the resource readable) returns **200 OK** with the
tombstoned representation, because a representation still exists.

**Classification:** evidence-backed default.
**Justification:** `[COMPARATIVE]` AIP, Azure, Zalando, and GitHub all
return 204; Stripe's `200` + `{deleted: true, id}` receipt is the one
documented outlier, imitated because Stripe is imitated, endorsed by no
guideline. **Declined:** the Stripe receipt pattern (requires a per-resource
confirmation schema; diverges from every surveyed guideline).
**Confidence: moderate-high.**
**Evidence:** `survey-02` finding 5 · `baseline-01` §8.3.

---

## Standards posture — BCP 190 scope reading

**Decision (2026-08-09): RATIFIED.** The standard **states explicitly** the
scope reading of BCP 190 (RFC 8820, URI Design and Ownership): BCP 190
restrains interoperability standards from constraining *other parties'* URI
spaces; a house standard constraining **its own organization's
deployments** is not the harm the BCP addresses. This position licenses the
ratified URI rules (path versioning, kebab-case segments, plural
collections, action sub-paths).

**Classification:** project policy — an interpretation, stated as such.
**Declined:** silence (invites the objection ad hoc from every reviewer who
knows the BCP). **Confidence: high.**
**Evidence:** `baseline-01` §8.2 (which instructs stating the reading
explicitly).

---

## Standards posture — HATEOAS

**Decision (2026-08-09): RATIFIED.** The standard **describes itself
honestly**: resource-oriented HTTP, not Fielding-complete REST. It does not
require hypermedia controls, does not claim Fielding conformance, and
notes the divergence openly.

**Classification:** project policy.
**Justification:** `[COMPARATIVE]` every surveyed API declines HATEOAS;
`baseline-01` §8.2: "The standard cannot both claim Fielding conformance
and describe the field." **Declined:** mandating HATEOAS (no field
practice, no client tooling) · claiming REST-in-Fielding's-sense anyway
(knowingly falsifiable). **Confidence: high.**
**Evidence:** `baseline-01` §8.2.

---

## Caching posture for mutable data

**Decision (2026-08-09): RATIFIED — three-tier explicit default.** Every
response carries explicit `Cache-Control` (silence is forbidden — it
delegates to heuristic caching). Authenticated or mutable resources
default to `private, no-cache`, revalidating via the strong `ETag`
machinery `HS-014` mandates (cheap 304s, zero staleness). `no-store` is
reserved for genuinely sensitive payloads. `public` with `max-age` is
permitted only for resources documented as immutable or deliberately
stale-tolerant.

**Classification:** the leak mechanism is a protocol requirement
(RFC 9111 §3, §5.2.2.7 — authenticated responses in shared caches are a
cross-user leak); the tier boundaries are project policy.
**Declined:** blanket `no-store` (named anti-pattern — discards the
largest performance lever) · mechanism-only with no posture (leaves every
API to re-derive the tiers). **Confidence: moderate-high.**
**Evidence:** `baseline-01` threat table (cache rows), §6 "Caching
authenticated / mutable resources" (SETTLED mechanism / POLICY
aggression), HS-014.

---

## Batch ratification — HS-001 through HS-020

**Decision (2026-08-09): RATIFIED en bloc, as proposed** in
`baseline-01` §7, per the Gate C batch-confirm procedure the owner
selected at intake (one batch per report for uncontested principles) and
confirmed for this report. No principle was pulled out.

Covers: protocol hygiene (HS-001–003) · URI discipline (HS-004–005) ·
method semantics (HS-006–009; `HS-009` QUERY stays MAY with the Spring
7.1 / Nov 2026 promotion trigger) · status codes (HS-010–013) ·
conditional requests (HS-014–015) · caching mechanism (HS-016–018) ·
extension hygiene (HS-019–020).

**Classification:** per each row's evidence class in `baseline-01` §7 —
predominantly protocol requirements grounded in RFC 9110/9111, with
HS-009's MAY and HS-019/020's SHOULDs as evidence-backed defaults.

**Notes:** four principles were already load-bearing inside individually
ratified decisions before this batch (HS-007 in action syntax, HS-013 in
trailing slash, HS-014 in caching posture, HS-016/017 as caching tier
zero). Nothing in the twenty conflicts with any walked decision.

**Evidence:** `baseline-01` §7 (the principles table, per-row citations).

> **Annotation (2026-08-10, Phase 4 classification sweep):** the batch
> note "predominantly protocol requirements" is refined per raw-RFC
> verification. `HS-004` (URI stability), `HS-011`/`HS-012` (status-code
> usage guidance), and `HS-015` (requiring `If-Match`) are
> evidence-backed defaults grounded in protocol-defined machinery, not
> protocol law; `HS-016`/`HS-017` (explicit cache marking) and `HS-018`
> (`Vary`, a SHOULD in RFC 9110 §12.5.5) are `[POLICY]` tightenings of
> protocol-grounded rules. No ratified strength changes; the drafted
> standard's provenance lines now label these accordingly.

---

## Addendum A3 — Status-code contested rows (+ drafting rows en bloc)

**Decision (2026-08-09, Gate C addendum): RATIFIED, three contested rows
plus five drafting rows.** Raised by the gap review's closed
status-code-table finding; the full "which codes, when" table is Phase 3
drafting, anchored by these rulings.

1. **Create responses:** every **single-resource** create returns
   **`201 Created` with a `Location` header** to the new resource — via
   POST or PUT-with-client-supplied-ID alike (RFC 9110 semantics); updates
   return `200` with the representation. Stripe's flat-200 declined as the
   documented outlier. **Bulk creation is governed by row 2, not this
   row:** an `AC-018` bulk endpoint returns the `200` envelope, and each
   per-item outcome carries its own created-resource URI (the
   `Location`-equivalent) and per-item status. Confidence high.
2. **Partial-bulk status (completes `AC-018`):** **`200` with the
   per-item outcome envelope**; `207 Multi-Status` declined — WebDAV
   (RFC 4918) baggage, poor generator support, Zalando's
   only-with-WebDAV-body rule, and the envelope is authoritative (the
   same one-source-of-truth logic as the pagination-links decision).
   Confidence moderate-high.
3. **Existence masking:** object-level authorization denials default to
   **`404 Not Found`** so existence does not leak across tenants (pairs
   with `OP-006` — unguessability is not access control — and the
   in-handler object-authorization decision; the GitHub private-repo
   model). `403 Forbidden` is permitted only where existence is already
   public, or the caller is authenticated within the resource's tenant
   and merely lacks a role. Confidence moderate-high.

**Drafting rows ratified en bloc:** `405` with a mandatory `Allow`
header · `415` for unsupported media types (load-bearing in A1's PATCH
rule) · `428 Precondition Required` where `HS-015` demands `If-Match`
and it is absent · `401`-vs-`403` discipline (401 = unauthenticated,
403 = unauthorized) · empty collection = `200` with an empty array,
never `404`.

**Classification:** protocol requirements where published HTTP semantics
decide — RFC 9110 for 201, 405+`Allow`, 415, and the 401/403 distinction;
RFC 6585 for 428 — with project policy for the existence-masking default
and the 207 declination.

**Evidence:** `survey-02` finding 5 (Stripe 200-on-create outlier) ·
RFC 9110 §15.3 · RFC 4918 (207's WebDAV definition) · Zalando 207 rule ·
`OP-006` and the five-axes authorization decision · the gap review.

> **Annotation (2026-08-10, Phase 4 second-lens review):** classification
> nuance on row 1 — RFC 9110 §15.3.2/§9.3.3 make `Location` on `201` a
> SHOULD, not a MUST, so the unconditional MUST ratified here is a
> `[POLICY]` tightening of a protocol-grounded rule, not protocol law
> itself. The ruling is unchanged; the drafted standard (R5.6) now labels
> the strength accordingly. The 405+`Allow` row is unaffected — that MUST
> is RFC 9110 §15.5.6's own.
