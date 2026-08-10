# REST API Design Standard

**Status:** Phase 3 working draft — not approved. Gate D (approval for
systematic review) has not run. Part I (§1–§12) is drafted; Part II (the
Decision Log) and the appendices are pending per [`PLAN.md`](PLAN.md)
Phase 3.

**Provenance model:** this document transcribes decisions ratified at Gate C
and its addendum (recorded in [`research/decisions/`](research/decisions/));
it does not make policy. Where a provision has no Gate C record — the
document's own conformance apparatus — it is marked **Apparatus (Gate D)**
and becomes binding only when Gate D approves this document.

---

## Part I — Normative sections

## 1. Purpose, audience, terminology, and conformance

### 1.1 Purpose and audience

This standard defines how HTTP APIs under the adopting organization's control
are designed: their URI surface, method and status-code semantics,
representations, error contract, collection behavior, caching, security
posture, lifecycle, and operational behavior.

It is written for the people who design, review, and operate those APIs, and
for the tooling that checks them. It distinguishes three kinds of material,
and every rule declares which kind it is (§1.6):

1. externally supported facts and protocol requirements;
2. recommended defaults selected from legitimate alternatives; and
3. project policy that is intentionally stricter than the underlying
   standards.

> Provenance: `PLAN.md` objective; Gate A.

### 1.2 What this standard is, and is not

**Honest self-description.** This standard specifies **resource-oriented
HTTP APIs**. It does not require hypermedia controls (HATEOAS), does not
claim conformance to Fielding's REST architectural style, and notes that
divergence openly: every API surveyed for this standard declines hypermedia
controls, and a standard cannot both claim Fielding conformance and describe
the field. "REST" in this document's title is used in the industry's
prevailing sense — resource-oriented HTTP — not Fielding's.

> Provenance: Gate C walked decision "Standards posture — HATEOAS"
> (`research/decisions/baseline-01-http-semantics.decision.md`) ·
> project policy.

**Non-goals.** This standard is not a framework tutorial, an
implementation-language guide, or a general distributed-systems handbook.
GraphQL, RPC protocols, event streaming, and messaging systems are
considered only where a boundary or interoperability question with a
resource-oriented HTTP API must be settled.

> Provenance: `PLAN.md` scope; Gate A.

### 1.3 URI ownership — the BCP 190 scope reading

BCP 190 (RFC 8820, *URI Design and Ownership*) restrains interoperability
specifications from constraining **other parties'** URI spaces. This
standard constrains URI structure only for deployments the adopting
organization itself controls. A house standard governing its own URI space
is not the harm BCP 190 addresses. This reading is stated here, once,
because the standard's URI rules (path versioning, kebab-case segments,
plural collections, action sub-paths) depend on it.

> Provenance: Gate C walked decision "Standards posture — BCP 190 scope
> reading" (`baseline-01` decisions) · project policy — an interpretation,
> stated as such.

### 1.4 Requirement language

**R1.1** The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**,
**SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **NOT
RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be
interpreted as described in RFC 2119 and RFC 8174 when, and only when, they
appear in all capitals, as shown here.

> Provenance: `PLAN.md` Phase 3 (RFC 2119/8174 requirement language) ·
> protocol requirement (the interpretation rule is itself BCP 14).

### 1.5 Rule identifiers and provenance

**R1.2 — Stable rule IDs.** Every normative rule in this standard carries a
stable identifier of the form `R<section>.<sequence>` (for example, `R1.2`
is the second rule of §1). Once assigned, an identifier is never renumbered
and never reused. A rule that is later removed or superseded keeps its
identifier as a tombstone entry naming its replacement. New rules take the
next unused sequence number in their section regardless of where they sit
in the prose.

**R1.3 — The research series are frozen.** The identifiers `HS-001`–`HS-020`,
`AC-001`–`AC-021`, and `OP-001`–`OP-025` are **research-provenance keys**
from the Gate C decision layer. They are cited in provenance lines
throughout this document and remain the keys into
[`research/decisions/`](research/decisions/). No new identifier is ever
minted in those series: a drafted rule that had no proposed principle would
otherwise acquire fabricated research lineage. The full two-way mapping
between rule IDs and provenance IDs is maintained in Part II (Decision
Log).

**Section namespace.** The `R<section>` prefix space is fixed at twelve
normative sections, in this order: 1 purpose/conformance · 2 resources and
URI modeling · 3 methods, safety, idempotency, conditionals · 4 requests,
representations, negotiation, schemas · 5 status codes and errors ·
6 collections, pagination, filtering, sorting · 7 caching and concurrency ·
8 authentication, authorization, security · 9 lifecycle, versioning,
deprecation · 10 asynchronous work, bulk, webhooks · 11 rate limits,
retries, observability · 12 client obligations. Reserving the numbering
here keeps section-prefixed rule IDs stable while sections are drafted.

> Provenance: Apparatus (Gate D) — rule-ID mapping policy delegated to
> Phase 3 drafting by `PLAN.md` and the gap review (item B.13).

### 1.6 Classification and evidence labels

**R1.4** Every normative rule carries exactly one **classification**, and
material labeled project policy MUST never be presented as protocol law:

| Classification | Meaning |
| --- | --- |
| **Protocol requirement** | Required by a published standard (RFC, W3C Recommendation, BCP). The rule cites it. |
| **Evidence-backed default** | Chosen from legitimate alternatives because the evidence favors it. |
| **Project policy** | This organization's choice, stricter than or unaddressed by the underlying standards. Always labeled `[POLICY]`. |
| **Exception** | A named, bounded carve-out from another rule. |

The Phase 2 vocabulary's fifth class, *unresolved question*, does not
appear here by construction: an unresolved question cannot be a normative
rule. Open questions live in `PLAN.md` and the decision layer until
resolved.

Where individual claims inside a rationale need finer labeling, the decision
layer's evidence classes are used verbatim: `[FACT]` (primary-sourced),
`[COMPARATIVE]` (surveyed practice), `[INFERENCE]` (reasoned from evidence),
`[POLICY]` (this project's choice).

> Provenance: `PLAN.md` Phase 2 classification vocabulary and definition of
> done ("every project policy labeled as policy").

### 1.7 Conformance tiers

**R1.5** Every API declares one **conformance tier** in its conformance
note (§1.9). The tier names the API's audience, not its quality bar:

| Tier | Audience |
| --- | --- |
| `internal` | Consumed only by the providing organization's own teams. |
| `partner` | Consumed by a closed set of contracted third parties. |
| `public` | Open to self-service third-party developers. |

Every rule in this standard applies at every tier unless the rule itself
states a tier scope. Later sections MAY scope individual rules by tier
(for example, a rule that binds at `public` but relaxes to SHOULD at
`internal`); absence of a tier annotation means uniform applicability.

> Provenance: Apparatus (Gate D) — gap review item B.8, adapted from the
> CLI Design Standard's tier system.

### 1.8 Applicability switches

**R1.6** Rules that only make sense for APIs with a given capability are
scoped by a named **applicability switch**. An API's conformance note
declares the state of every switch. A rule scoped to a switch binds only
while that switch is on. A declaration that a switch is off — taking its
rules as not applicable — MUST carry a stated reason in the conformance
note (the N/A-with-reason discipline): "N/A" with no reason is a deviation,
not an exemption.

The switch vocabulary: `webhooks` · `async-operations` · `bulk-operations` ·
`multi-tenant` · `public-internet` · `handles-pii` · `third-party-clients` ·
`file-upload`.

> Provenance: Apparatus (Gate D) — gap review item B.8.

### 1.9 Deviations and the conformance note

**R1.7 — No silent deviation.** An API deviates from this standard only in
writing. Every deviation from a **SHOULD** MUST be recorded in the API's
conformance note with its reason. A deviation from a **MUST** makes the API
nonconformant unless it is recorded as an approved exception in the
conformance note; silent deviation from any rule strength is nonconformance
regardless of the deviation's merits. Each API maintains one conformance
note using this template:

```markdown
## Conformance note — <API name>

Standard: rest-api-standard v<version>
Tier: internal | partner | public
Switches: webhooks=<on|off>, async-operations=<on|off>,
  bulk-operations=<on|off>, multi-tenant=<on|off>,
  public-internet=<on|off>, handles-pii=<on|off>,
  third-party-clients=<on|off>, file-upload=<on|off>
  (every switch declared off carries a one-line reason)

Deviations:
- <rule ID> · <rule strength> · what differs · why · approver · date

N/A declarations:
- <rule ID or switch> · why it cannot apply
```

> Provenance: Apparatus (Gate D) — gap review item B.12 (no-silent-deviation
> clause and conformance-note template, structure carried from the CLI
> Design Standard).

### 1.10 Reserved names

This inventory is the single register of names this standard reserves
across every conforming API. Later sections define the full behavior behind
each name; this section fixes the names themselves so that no API assigns
them a different meaning and no two APIs name the same concept differently.
Requirement keywords inside the tables below restate obligations defined by
the cited decision records — and, once drafted, by their home-section
rules, which are authoritative; the reservation obligations of this section
itself are R1.8 and R1.9.

**R1.8 — Reservation discipline.** A conforming API MUST NOT use a reserved
name for any purpose other than the meaning registered here, and when it
offers the corresponding capability it MUST use the reserved name rather
than a synonym (same concept, same name — everywhere). Reserved names are
part of the frozen compatibility surface once shipped.

**R1.9 — `dry_run` rejection guard.** `dry_run` is reserved standard-wide:
a mutating request carrying `dry_run=true` to an endpoint that does not
implement dry-run MUST be rejected with `400`, never silently executed.
(Without this guard, the parameter would carry the same
silent-real-execution hazard that disqualified `Prefer: validate-only` at
ratification.)

> Provenance (R1.8): Gate C addendum A2/A4 namings + gap review item B.5
> (inventory as apparatus). Provenance (R1.9): Gate C addendum A4
> reservation guard (`baseline-02` decisions) · project policy `[POLICY]`.

#### Reserved query parameters

| Name | Meaning | Registered behavior | Provenance |
| --- | --- | --- | --- |
| `sort` | Sort order for a collection | Offering sorting is MAY; when offered: comma-separated snake_case field names, `-` prefix descending, bare name ascending, multi-key in listed order, restricted to a documented sortable-field set | Addendum A2.2 (`baseline-02` decisions) `[POLICY]` |
| `fields` | Sparse fieldsets (field selection) | Offering is MAY; when offered: comma-separated list of snake_case field names | Walked decision "Field selection" (`baseline-02` decisions) `[POLICY]` |
| `cursor` | Opaque pagination position | Request-side name for cursor pagination; cursors are opaque and non-constructable (`AC-013`) | Addendum A2.3, completing `AC-013`/`AC-014` `[POLICY]` |
| `limit` | Requested page size | Request-side name; each collection documents its default and maximum (`OP-009`) | Addendum A2.3 `[POLICY]` |
| `dry_run` | Rehearse a mutation without executing it | Support is MAY per endpoint, SHOULD for destructive and bulk operations; unsupported ⇒ `400` per R1.9; full output contract in R3.12 | Addendum A4 `[POLICY]` |
| `<field>[gte]`, `<field>[gt]`, `<field>[lte]`, `<field>[lt]` | Range filters on collection lists | The only permitted bracketed query-parameter forms; AND-combined; base name obeys the `AC-007` grammar | Walked decision "Filter grammar" (`baseline-02` decisions) `[POLICY]` |

#### Reserved headers

| Name | Direction | Meaning | Provenance |
| --- | --- | --- | --- |
| `Idempotency-Key` | Request | Idempotency key on non-idempotent state-changing requests; Stripe semantics — payload fingerprint, reuse with a different payload rejected; retained ≥ 24 h. `[POLICY]` — the IETF draft that standardized this shape expired 2026-04-18; never cite it as a standard | `AC-016`/`AC-017` (completed) |
| `request-id` | Response | Correlation ID, emitted on every response including errors. Lowercase name; RFC 6648 deprecates new `X-` prefixed fields, ruling out `X-Request-Id` | Addendum A2.4, completing `OP-018` `[POLICY]` |
| `ETag` / `If-Match` / `If-None-Match` | Response / request | Strong validators and conditional requests | `HS-014`/`HS-015` · protocol requirement (RFC 9110) |
| `Location` | Response | Target of every single-resource create (`201 Created`) | Addendum A3.1 · protocol requirement (RFC 9110) |
| `Allow` | Response | Mandatory on every `405 Method Not Allowed` | Addendum A3 · protocol requirement (RFC 9110) |
| `Retry-After` | Response | Mandatory on `429`; also used on `503` and on `202` polling guidance | `OP-010`/`OP-011` · protocol requirement (RFC 9110, RFC 6585) |
| `RateLimit` / `RateLimit-Policy` | Response | SHOULD advertise quota state **in the syntax of `draft-ietf-httpapi-ratelimit-headers-11`**. `[POLICY]` — an unpublished Internet-Draft; MUST NOT be described as standards-compliant; the pinned revision is cited wherever referenced | `OP-010` |
| `Deprecation` | Response | Deprecation signal (RFC 9745, Standards Track) | `OP-013` |
| `Sunset` | Response | Retirement date signal (RFC 8594 — Informational, not Standards Track; the pair use different date formats and are not equivalent) | `OP-014` |
| `Accept-Patch` | Response | Advertises every supported PATCH media type (RFC 5789) | Addendum A1 |
| `Prefer` | Request | RFC 7240 preferences; offering is MAY | `AC-020` |
| `traceparent` / `tracestate` | Both | W3C Trace Context propagation; `tracestate` capped | `OP-019` |
| `webhook-id` / `webhook-timestamp` / `webhook-signature` | Webhook delivery | Standard Webhooks delivery envelope: unique delivery ID, timestamp, signature (shared-secret topology); the ID and timestamp envelope is retained under the RFC 9421 cross-org branch | `OP-016` |
| `Content-Digest` | Webhook delivery | REQUIRED covered component when signing with RFC 9421 (RFC 9530) | `OP-016` invariant I11 |

**No new `X-` names.** Consistent with RFC 6648, this standard never
reserves or emits a new `X-` prefixed field name.

#### Reserved media types

| Media type | Meaning | Provenance |
| --- | --- | --- |
| `application/problem+json` | Every API MUST be capable of returning every application-generated error in this shape (RFC 9457; infrastructure carve-out applies) | `AC-003` |
| `application/merge-patch+json` | The MUST PATCH format (RFC 7396) | Addendum A1 |
| `application/json-patch+json` | The bounded MAY PATCH format (RFC 6902), for resources needing value-null-distinct-from-absent, per-element array edits, or test-conditioned updates | Addendum A1 |

#### Reserved action verbs (path segments)

Core registry for the `POST /{collection}/{id}/{action}` form. An API using
one of these verbs MUST mean exactly this; an action segment can never be
used as a collection name under the same parent. One verb per meaning,
API-wide; kebab-case for multi-word verbs.

| Verb | Registered meaning | Provenance |
| --- | --- | --- |
| `cancel` | Terminal, irreversible stop of an in-flight process | Addendum A5 `[POLICY]` |
| `archive` / `restore` | Reversible visibility pair (the soft-delete pair the DELETE rule references) | Addendum A5 `[POLICY]` |
| `approve` / `reject` | Review outcomes | Addendum A5 `[POLICY]` |
| `publish` / `unpublish` | Consumer-visibility pair | Addendum A5 `[POLICY]` |
| `duplicate` | Copy; returns `201` + `Location` | Addendum A5 `[POLICY]` |

### 1.11 Terminology

| Term | Meaning in this document |
| --- | --- |
| **Provider** | The organization operating the API. |
| **Consumer / client** | Any party issuing requests to the API, first- or third-party. |
| **Resource** | A named thing the API exposes at a URI. |
| **Collection** | A resource that contains other resources of one kind; named with a plural noun. |
| **Singleton** | A resource modeling exactly-one-per-context (the `/user`, `me` pattern); the documented exception to pluralization. |
| **Action** | A non-CRUD operation on a resource, expressed as `POST /{collection}/{id}/{action}`. |
| **Mutating request** | Any request whose success changes server state. |
| **Reserved name** | A query parameter, header, media type, or action verb registered in §1.10. |
| **Conformance note** | The per-API document required by R1.7. |
| **Tier / switch** | The declarations required by R1.5 and R1.6. |
| **Problem document** | An error body per RFC 9457 (`application/problem+json`). |

---

## 2. Resources and URI modeling

### 2.1 Resource orientation

**R2.1** An API MUST present a resource-oriented surface: noun resources,
hierarchy expressed in path segments, and the standard HTTP methods carrying
their RFC 9110 semantics. Operations that resist create/read/update/delete
modeling are expressed as action sub-resources (§2.4), never as an RPC-style
method overlay. There is no RPC carve-out for any service class.

> Provenance: walked decision "Structural lock — Resource orientation"
> (`baseline-01` decisions) · evidence-backed default, reinforced at its
> boundary by a protocol requirement (RFC 9205 §4.4 forbids overlaying
> generic semantics on HTTP methods) · confidence high.

### 2.2 Naming resources

**R2.2** Collection resources MUST use plural nouns (`/customers`,
`/orders`). **Exception:** a singleton or configuration resource modeling
exactly-one-per-context (the GitHub `/user` / Microsoft Graph `me` pattern)
uses the singular and MUST be documented as a singleton. For an irregular
noun, pick one form per resource and keep it consistent everywhere the stem
appears.

> Provenance: walked decision "Structural lock — Pluralization"
> (`baseline-01` decisions) · project policy on a near-universal
> convention · confidence high.

**R2.3** Resource names MUST be domain terms: the noun the business domain
itself uses, unabbreviated, with exactly one noun per concept API-wide.
The same concept MUST carry the same name wherever it appears — path,
query parameter, body field, header — differing only by the casing rules
of each surface (R2.4, R4.4).

> Provenance: Apparatus (Gate D) — gap review items R2.2/R3.8 (noun naming,
> same-concept-same-name parity), extending R1.8.

**R2.4** Path segments MUST use kebab-case (`/sales-order-items`), pattern
`^[a-z][a-z\-0-9]*$`.

> Provenance: walked decision "Structural lock — Path-segment casing"
> (`baseline-01` decisions) · project policy `[POLICY]` — a genuine
> coin-flip against snake_case, decided for enforceability · confidence
> moderate.

### 2.3 Path shape

**R2.5** Nest a sub-resource only where the child cannot exist outside its
parent. One sub-resource level is the norm; a path MUST NOT reference more
than three resources. Beyond that, flatten and relate with query filters
(`GET /orders?customer=…`).

> Provenance: walked decision "Structural lock — Path depth" (`baseline-01`
> decisions) · project policy · confidence moderate-high.

**R2.6** Canonical URIs have no trailing slash. A trailing slash MUST NOT
carry semantics or identify a different resource; a request with a
trailing slash SHOULD receive `308 Permanent Redirect` to the canonical
form (308 preserves method and body, per R5.5).

> Provenance: walked decision "Structural lock — Trailing slash"
> (`baseline-01` decisions) · project policy; near-unanimous convention ·
> confidence high.

**R2.7** Each resource MUST be identified by a URI that remains stable
across changes to its mutable attributes. **Exception:** deliberately
versioned or dated resources.

> Provenance: `HS-004` (batch, `baseline-01` §7) · protocol requirement
> (RFC 9110 §3.1; BCP 190) · confidence high.

**R2.8** This standard's URI rules are construction rules applied within
the adopting organization's own URI space (§1.3). Neither this standard
nor any conforming API's documentation may mandate a fixed URI path prefix
that constrains another party's deployment.

> Provenance: `HS-005` (batch, `baseline-01` §7) · protocol requirement
> (BCP 190 / RFC 8820) · confidence high.

**R2.9** Path segments identify resources; query parameters modify the
operation. Filtering, sorting, pagination, field selection, and rehearsal
(`dry_run`) MUST travel as query parameters, and an operation modifier
MUST NOT be encoded as a path segment.

> Provenance: Apparatus (Gate D) — gap review item R2.3, grounded in
> `HS-004` and the ratified filter grammar (§6).

**R2.10** Personally identifiable information MUST NOT appear in any URI —
path or query string. URIs land in access logs, browser history, referrer
headers, and URL-keyed caches by default. Identify people by opaque IDs.

> Provenance: Apparatus (Gate D) — gap review item R9.7, adjacent to
> `OP-002` (which bans tokens in query strings).

### 2.4 Actions — operations that resist CRUD

**R2.11** A non-CRUD operation MUST use the sub-path verb form
`POST /{collection}/{id}/{action}` (for example
`POST /payment-intents/{id}/capture`). An action segment is a documented
verb and MUST NOT be used as a collection name under the same parent. The
core verb registry, with fixed meanings, is in §1.10; a domain verb beyond
the core registry is permitted with a per-API registry entry, and an API
MUST use one verb per meaning API-wide.

> Provenance: walked decision "Structural lock — Custom-action syntax" +
> addendum A5.1/A5.2 (`baseline-01`/`baseline-02` decisions) · project
> policy · confidence moderate.

**R2.12** Before minting any action verb, a design SHOULD prefer a
PATCH-able status field or a sub-resource; an action is the escape hatch
for state semantics that genuinely resist CRUD.

> Provenance: addendum A5.2 (AIP-136 discipline, `baseline-02` decisions) ·
> project policy · confidence moderate.

**R2.13** Custom actions MUST NOT be defined at collection level. Batch
semantics belong to bulk endpoints (§10.2) with per-item outcomes; a
verb-based batch mechanism would fork them.

> Provenance: addendum A5.3 (`baseline-02` decisions) · project policy ·
> confidence moderate.

---

## 3. Methods, safety, idempotency, and conditional operations

### 3.1 Semantic foundation

**R3.1** Method, status-code, validator, and caching semantics MUST be
cited from RFC 9110 and RFC 9111. The obsoleted RFC 723x family MUST NOT
be cited.

> Provenance: `HS-001` (batch, `baseline-01` §7) · protocol requirement
> (STD 97/98) · confidence high.

**R3.2** An API MUST NOT redefine, refine, or overlay the semantics of any
registered method, status code, or header field.

> Provenance: `HS-002` (batch, `baseline-01` §7) · protocol requirement
> (RFC 9205 §4.4) · confidence high.

### 3.2 Method discipline

**R3.3** Methods MUST be used per their registered safety and idempotence.
A safe method MUST NOT perform requested state change. **Exception:**
logging and metrics side effects do not violate safety.

> Provenance: `HS-006` (batch, `baseline-01` §7) · protocol requirement
> (RFC 9110 §9.2.1) · confidence high.

**R3.4** A method MUST NOT be tunneled through another method — no
`?_method=` parameters, no action-in-body dispatch.

> Provenance: `HS-007` (batch, `baseline-01` §7) · protocol requirement
> (defeats RFC 9111 §4.4 cache invalidation) · confidence high.

**R3.5** PUT SHOULD be used for full replacement and PATCH for partial
modification.

> Provenance: `HS-008` (batch, `baseline-01` §7) · evidence-backed default
> (RFC 5789) · confidence high.

**R3.6** An API MAY use QUERY (RFC 10008) for safe, idempotent,
body-carrying reads instead of overloading POST.

> Provenance: `HS-009` (batch, `baseline-01` §7; confirmed MAY by
> `baseline-01b` — no CDN implements body-keyed caching) · evidence-backed
> default · confidence moderate.

### 3.3 PATCH body format

**R3.7** PATCH request bodies MUST be JSON Merge Patch (RFC 7396) sent
with `Content-Type: application/merge-patch+json`. Servers MUST reject
unsupported media types with `415 Unsupported Media Type` and MUST
advertise every supported format in `Accept-Patch` (RFC 5789). An API
whose resources genuinely require value-null distinct from absent,
per-element array edits, or test-conditioned updates MAY additionally
accept JSON Patch (RFC 6902) at `application/json-patch+json` on the same
resource and MUST document which format applies where. RFC 5789's
atomicity requirement and its conditional-request pairing (R3.10) apply
regardless of format.

**R3.8 — Null-equivalence companion rule.** Resource representations MUST
give `null` and an absent property the same meaning; Merge Patch delete
semantics are the sole exception, and a `null` targeting a non-deletable
field MUST return `400`. Consequence, accepted at ratification: tri-state
fields (unset / null / value) are forbidden across the standard; a
resource needing that distinction models it another way or uses the JSON
Patch path.

> Provenance: addendum A1 (`baseline-02` decisions, completing `HS-008`,
> companion to `AC-011`) · evidence-backed default (R3.7); project policy
> shared with Azure and Zalando (R3.8) · confidence moderate-high (format),
> high (companion rule).

### 3.4 Idempotency keys

**R3.9** An API MUST accept an idempotency key on non-idempotent
state-changing requests, carried in the `Idempotency-Key` request header
(Stripe semantics): the server fingerprints the request payload, replays
the stored response for a genuine retry, and rejects a reused key carrying
a different payload. **Exception:** naturally idempotent operations (PUT
with a client-supplied ID). The stated retention window MUST be at least
24 hours. `[POLICY]` This convention has no standards backing — the IETF
draft that standardized the shape expired 2026-04-18 — and MUST NOT be
cited as a standard. Deployments needing keys kept out of intermediary
logs MUST add explicit header-redaction and cache-key-exclusion
configuration.

> Provenance: `AC-016`/`AC-017` (completed; walked, `baseline-02`
> decisions + `baseline-02g`) · project policy `[POLICY]` · confidence
> high-moderate (placement), high (retention floor).

### 3.5 Conditional operations

**R3.10** A resource that supports conditional update MUST emit a strong
`ETag` (weak validators are sufficient for caching but not for
`If-Match`). **Exception:** resources never updated by clients.

> Provenance: `HS-014` (batch, `baseline-01` §7) · protocol requirement
> (RFC 9110 §8.8.3, §13.1.1) · confidence high.

**R3.11** Unsafe requests to resources exposed to concurrent modification
SHOULD require `If-Match`, returning `412 Precondition Failed` on
mismatch, and `428 Precondition Required` where the precondition is
demanded and absent (R5.11). Destructive-operation tightenings are in §7.3.
**Exception:** single-writer resources; append-only collections.

> Provenance: `HS-015` (batch, `baseline-01` §7) + addendum A3 drafting
> row (428) · protocol requirement (RFC 9110 §13.1.1; RFC 6585) ·
> confidence high.

### 3.6 Dry-run rehearsal

**R3.12** A mutating endpoint that offers a rehearsal accepts
`?dry_run=true` — support is MAY per endpoint and SHOULD for destructive
and bulk operations, with the standard-wide `400` rejection guard of
R1.9. The dry-run response contract:

1. MUST carry an explicit dry-run marker stating that no mutation
   occurred;
2. MUST return the outcome the real call would produce — validation
   errors in the ratified problem-document shape (R5.12), or the would-be
   representation with the real status semantics (R5.6 for creates);
3. MUST declare its validation depth (full pipeline versus schema-only),
   so a passing rehearsal is not over-trusted; and
4. MUST NOT consume an `Idempotency-Key` (R3.9).

> Provenance: addendum A4 (`baseline-02` decisions) · project policy
> `[POLICY]`; the transport-failure analysis behind declining
> `Prefer: validate-only` is evidence-backed (RFC 7240 advisory
> semantics) · confidence moderate-high (transport), moderate (contract
> details).

---

## 4. Requests, representations, negotiation, and schemas

### 4.1 The contract document

**R4.1** Every API MUST publish an OpenAPI document as the contract source
of truth — version 3.1, or 3.2 only where the team's full toolchain
(parser, linter, generator, docs renderer) is verified against it. Payload
structure MUST be validated with JSON Schema 2020-12. 3.1 is the
unconditional default; a verified-3.2 toolchain makes a 3.2 document fully
compliant, not an exception.

> Provenance: `AC-001` (completed; walked, `baseline-02` decisions +
> `baseline-02b` — Spectral silently ignores 3.2 constructs) ·
> evidence-backed default · confidence high (3.1 floor), moderate
> (conditional 3.2 clause).

**R4.2** The description document is authoritative: changes to it MUST be
gated on an automated compatibility check against the previous version.

> Provenance: `AC-002` (batch, `baseline-02` §7) · evidence-backed
> default · confidence moderate.

### 4.2 Representation rules

**R4.3** Every JSON response body MUST have an object at the top level —
never a bare array. **Exception:** non-JSON media types.

> Provenance: `AC-006` (batch, `baseline-02` §7) · evidence-backed
> default · confidence high.

**R4.4** Body properties and query parameters MUST use snake_case — one
convention across both surfaces — enforced by pattern `^[a-z_][a-z_0-9]*$`
for body properties and `^[a-z][_a-z0-9]*$` for query parameters. (Path
segments are kebab-case per R2.4; the two surfaces are deliberately
distinct.)

> Provenance: `AC-007` (completed; walked, `baseline-02` decisions) ·
> project policy `[POLICY]` — the consistency requirement is
> evidence-backed; the concrete pick is policy · confidence high (one
> convention required), policy (the pick).

**R4.5** Identifiers MUST be represented as JSON strings. **Exception:**
genuinely numeric domain quantities.

> Provenance: `AC-008` (batch, `baseline-02` §7) · evidence-backed default
> (avoids JS 2^53 truncation and volume leakage) · confidence high.

**R4.6** Timestamps MUST be RFC 3339 strings with an explicit offset.
**Exception:** date-only fields.

> Provenance: `AC-009` (batch, `baseline-02` §7) · evidence-backed default
> · confidence high.

**R4.7** Monetary amounts MUST NOT be floating-point numbers. Amounts are
encoded as minor-unit integers with a separate ISO 4217 `currency` field —
`"amount": 1099` with `"currency": "usd"` means $10.99, the amount in the
currency's smallest unit. The `currency` field is REQUIRED alongside every
amount: clients need the ISO 4217 exponent to render (JPY has exponent 0,
BHD has exponent 3), and API documentation SHOULD point at the exponent
table.

> Provenance: `AC-010` (batch, `baseline-02` §7; float ban, confidence
> high) + walked decision "Money representation" (`baseline-02` decisions;
> owner selection among float-safe encodings) · project policy `[POLICY]`
> for the encoding · confidence moderate (a genuine fork among safe
> encodings).

**R4.8** The contract MUST state explicitly how null and omission are
distinguished. Under this standard that rule is fixed by R3.8: null and
absent mean the same thing everywhere, with Merge Patch deletion as the
sole exception.

> Provenance: `AC-011` (batch, `baseline-02` §7) · evidence-backed default
> · confidence high.

**R4.9** Enum additions are non-breaking and MUST be documented as such;
the corresponding client obligation to tolerate unknown values is R12.4.
**Exception:** genuinely closed enumerations, documented as closed.

> Provenance: `AC-012` (batch, `baseline-02` §7) · evidence-backed
> default · confidence moderate.

### 4.3 Content negotiation

**R4.10** The default media type — served when a request carries no
`Accept` header — is `application/json`, encoded UTF-8. Media-type
selection MUST use HTTP content negotiation, never a `format` query
parameter. A request whose `Accept` header excludes every representation
the endpoint supports SHOULD receive `406 Not Acceptable` rather than a
silently substituted type.

> Provenance: Apparatus (Gate D) — gap review item R4.2
> (content-negotiation defaults).

**R4.11** Every response whose content was selected by a request header
MUST send `Vary` listing each header that influenced selection.
**Exception:** responses with no negotiation.

> Provenance: `HS-018` (batch, `baseline-01` §7) · protocol requirement
> (RFC 9110 §12.5.5) · confidence high.

### 4.4 Extension hygiene

**R4.12** New header fields SHOULD be defined using RFC 9651 structured
field types. **Exception:** fields that must match an existing deployed
non-structured syntax.

> Provenance: `HS-019` (batch, `baseline-01` §7) · evidence-backed
> default · confidence moderate.

**R4.13** Where a link relation is expressed and a registered relation
type exists for the semantic, the registered relation SHOULD be used.

> Provenance: `HS-020` (batch, `baseline-01` §7) · evidence-backed
> default · confidence moderate.

**R4.14** Where the contract must describe a family of URIs, RFC 6570 URI
Templates SHOULD be used rather than prose construction rules.

> Provenance: `AC-021` (batch, `baseline-02` §7) · evidence-backed
> default · confidence moderate.

**R4.15** An API MAY support RFC 7240 `Prefer`, including
`return=minimal` / `return=representation` (and `respond-async`, §10.1).
Preferences are advisory by design: a server MAY ignore them, and a
client MUST NOT depend on one being honored. For exactly that reason a
preference token MUST NOT carry safety semantics — the ground on which
`Prefer: validate-only` was declined in favor of `dry_run` (§1.10).

> Provenance: `AC-020` (batch, `baseline-02` §7) + addendum A4 rationale ·
> evidence-backed default · confidence moderate.

---

## 5. Status codes and errors

### 5.1 Status-code discipline

**R5.1** The status code MUST match the registered semantics of the
outcome. A failed operation MUST NOT return 2xx.

> Provenance: `HS-010` (batch, `baseline-01` §7) · protocol requirement
> (RFC 9205) · confidence high.

**R5.2** Unregistered status codes MUST NOT be defined or used.

> Provenance: `HS-003` (batch, `baseline-01` §7) · protocol requirement
> (RFC 9205 §4.6) · confidence high.

**R5.3** `422 Unprocessable Content` SHOULD be used for semantically
invalid but well-formed requests, `409 Conflict` for conflicts with
current resource state; `400 Bad Request` remains correct for malformed
syntax.

> Provenance: `HS-011` (batch, `baseline-01` §7) · protocol requirement
> (RFC 9110 §15.5.21) · confidence high.

**R5.4** `410 Gone` SHOULD NOT be returned unless permanence is actually
known and recorded. **Exception:** tombstoned resources with retained
deletion records.

> Provenance: `HS-012` (batch, `baseline-01` §7) · protocol requirement
> (RFC 9110 §15.5.11) · confidence high.

**R5.5** Where method and body must survive a redirect, `307` or `308`
MUST be used; `301`/`302` have a documented history of rewriting the
method to GET. **Exception:** `303 See Other` where a GET on a different
resource is genuinely intended.

> Provenance: `HS-013` (batch, `baseline-01` §7) · protocol requirement
> (RFC 9110 §15.4) · confidence high.

### 5.2 The operation-to-status map

**R5.6** Every single-resource create — via POST, or PUT with a
client-supplied ID — MUST return `201 Created` with a `Location` header
referencing the new resource. Updates return `200 OK` with the
representation. Bulk creation is governed by R5.8, not this rule.

> Provenance: addendum A3.1 (`baseline-01` decisions) · protocol
> requirement (RFC 9110) · confidence high.

**R5.7** A successful DELETE returns `204 No Content` with an empty body.
**Exception:** an API that soft-deletes (marks deleted but keeps the
resource readable) returns `200 OK` with the tombstoned representation,
because a representation still exists.

> Provenance: walked decision "DELETE response" (`baseline-01` decisions) ·
> evidence-backed default · confidence moderate-high.

**R5.8** A bulk endpoint that completes partially MUST return `200 OK`
with the per-item outcome envelope (§10.2); each per-item outcome carries
its own status and, for creates, the created resource's URI. `207
Multi-Status` MUST NOT be used — it carries WebDAV (RFC 4918) semantics,
and the envelope is the single source of truth.

> Provenance: addendum A3.2 (`baseline-01` decisions, completing `AC-018`)
> · project policy · confidence moderate-high.

**R5.9** `401 Unauthorized` means unauthenticated; `403 Forbidden` means
authenticated but unauthorized. The two MUST NOT be conflated.

> Provenance: addendum A3 drafting row (`baseline-01` decisions) ·
> protocol requirement (RFC 9110) · confidence high.

**R5.10** An object-level authorization denial defaults to
`404 Not Found`, so that resource existence does not leak across tenants.
`403` is permitted only where existence is already public, or the caller
is authenticated within the resource's tenant and merely lacks a role.

> Provenance: addendum A3.3 (`baseline-01` decisions; pairs with `OP-006`)
> · project policy · confidence moderate-high.

**R5.11** `405 Method Not Allowed` MUST carry an `Allow` header.
`415 Unsupported Media Type` is the response to an unsupported request
media type (including PATCH formats, R3.7). `428 Precondition Required`
is the response where `If-Match` is demanded and absent (R3.11).

> Provenance: addendum A3 drafting rows (`baseline-01` decisions) ·
> protocol requirements (RFC 9110 §15.3; RFC 6585) · confidence high.

### 5.3 Errors

**R5.12** Every API MUST be capable of returning every error response the
application itself generates as `application/problem+json` (RFC 9457)
when the client requests it. **Exception (named carve-out):** errors
emitted by infrastructure components outside application control —
reverse proxies, CDNs, WAFs, rate limiters, load balancers terminating
before application code — which MUST be documented as such. Nothing in
this standard is premised on the IANA HTTP Problem Types registry. The
matching client obligation — never *relying* on a problem document —
is R12.7.

> Provenance: `AC-003` (walked as amended, `baseline-02` decisions +
> `02d`/`02e`/`02f`) · evidence-backed default resting on a Standards
> Track RFC · confidence moderate (re-argued: no credible alternative
> exists).

**R5.13** Every problem document MUST carry `type`, `title`, `status`,
and a stable machine-readable `code` extension member, bound as follows:

1. `type` is the normative identifier; `code` is its short form. Each
   problem type has exactly one of each, bound by the fixed template
   `<https base>/<code, underscores to hyphens>` — for example
   `code: "out_of_credit"` gives
   `type: "https://problems.example.com/out-of-credit"`. The standard
   fixes the template shape; each API declares its base URI. The `code`
   grammar is `^[a-z][a-z0-9_]*$` (snake_case, hyphens excluded), so the
   underscore-to-hyphen mapping is injective.
2. Neither `type` nor `code` may change once published; a change of
   meaning is a new problem type with a new pair.
3. Dereferencing `type` is a courtesy, never a contract: a provider MAY
   redirect the `type` URI to documentation; clients MUST NOT depend on
   it resolving.
4. Human documentation lives in a separate `documentation` member, which
   MAY change over time and MAY vary by environment.
5. `type` MUST be present on every problem document, and `about:blank`
   MUST NOT be used (it is defined to carry no semantics beyond the
   status code, which contradicts the required discriminating `code`).
6. **Exception:** a provider operating an IANA-registered URN namespace
   identifier MAY use a URN `type` in that namespace; the 1:1 `code`
   binding and immutability rules apply identically.

`[POLICY]` Points 1, 5, and the resolvability declination are deliberate,
permitted deviations from RFC 9457's defaults, made in writing here.

> Provenance: `AC-004` (walked as amended, `baseline-02` decisions +
> `baseline-02f`) · evidence-backed default (member set) + project policy
> `[POLICY]` (binding design, `about:blank` ban) · confidence high
> (members), policy grounded in primary-sourced failure evidence (design).

**R5.14** RFC 7807 MUST NOT be cited — it is obsoleted by RFC 9457.
**Exception:** historical notes explicitly labeled as such.

> Provenance: `AC-005` (batch, `baseline-02` §7) · protocol requirement ·
> confidence high.

**R5.15** A validation failure covering one or more fields SHOULD carry a
field-level `errors[]` extension member on the problem document: an array
whose entries each carry a JSON Pointer to the offending input location
(`pointer`), a stable machine-readable `code`, and a human-readable
`detail`.

> Provenance: Apparatus (Gate D) — gap review item B.21 (Zalando/Belgif
> field-error shapes); extends R5.13.

**R5.16** An API MUST publish a catalog of every problem `type`/`code`
pair it can return.

> Provenance: Apparatus (Gate D) — gap review item R7.12; the pair
> stability it catalogs is ratified (R5.13.2).

**R5.17** No response body may expose stack traces, query fragments,
internal hostnames, or dependency detail. Secret and PII redaction rules
for problem `detail` are in §8.5.

> Provenance: `OP-020` (batch, `baseline-03` §7) · protocol-adjacent
> security requirement (OWASP API8:2023) · confidence high.

---

## 6. Collections, pagination, filtering, and sorting

### 6.1 The collection envelope

**R6.1** Every collection response MUST return a top-level object carrying
both the items and the continuation state — never a bare array — so that
metadata can be added without a breaking change.

> Provenance: `AC-014` (batch, `baseline-02` §7) · evidence-backed
> default · confidence high.

**R6.2** An empty collection returns `200 OK` with an empty items array —
never `404 Not Found`.

> Provenance: addendum A3 drafting row (`baseline-01` decisions) ·
> protocol requirement · confidence high.

### 6.2 Pagination

**R6.3** Pagination SHOULD use opaque, non-constructable cursors — offset
pagination is incorrect under concurrent mutation. **Exception:** small or
stable collections; UI requiring jump-to-page. The corresponding client
obligation not to construct or modify cursors is R12.5.

> Provenance: `AC-013` (batch, `baseline-02` §7) · evidence-backed
> default · confidence high.

**R6.4** Pagination state lives only in the body envelope (R6.1). RFC 8288
`Link` headers MUST NOT be emitted for pagination — dual emission creates
two places a cursor can live, which drift under maintenance.

> Provenance: walked decision "Pagination links" (`baseline-02` decisions)
> · project policy · confidence moderate-high.

**R6.5** The pagination request parameters are `cursor` and `limit`
(§1.10). Each collection MUST document its default and maximum `limit`
(the enforcement obligation is R11.1).

> Provenance: addendum A2.3 (`baseline-02` decisions, completing
> `AC-013`/`AC-014`) · project policy `[POLICY]` · confidence
> moderate-high.

### 6.3 Ordering and sorting

**R6.6** Every collection MUST document a total, stable default order,
with ties broken by an immutable key (`id`). A cursor over
nondeterministic order silently skips or duplicates rows, so this rule is
a soundness requirement for R6.3.

> Provenance: addendum A2.1 (`baseline-02` decisions) · project policy ·
> confidence high (soundness requirement).

**R6.7** Offering sorting is MAY. When offered, the syntax is fixed: the
`sort` query parameter takes a comma-separated list of snake_case field
names, `-` prefix for descending, bare name ascending, multi-key applied
in listed order, restricted to a documented enumerated sortable-field set
(bounded work, R11.1).

> Provenance: addendum A2.2 (`baseline-02` decisions) · project policy
> `[POLICY]` · confidence moderate-high.

### 6.4 Filtering

**R6.8** Collection list endpoints filter via per-field equality
parameters plus the bracket range operators `[gte]`, `[gt]`, `[lte]`,
`[lt]`, combined AND-only (§1.10). A structured query DSL is permitted
only as a separately documented search endpoint, never mixed into
collection listing.

> Provenance: walked decision "Filter grammar" (`baseline-02` decisions,
> companion to `AC-015`) · project policy · confidence moderate-high.

**R6.9** Filter parameters MUST NOT expose storage or query-engine
syntax — that couples the contract to the implementation and opens an
injection surface.

> Provenance: `AC-015` (batch, `baseline-02` §7) · evidence-backed
> default · confidence high.

### 6.5 Field selection

**R6.10** Offering sparse fieldsets is MAY — a fixed response shape is a
legitimate contract. When offered, the syntax is fixed: the `fields`
query parameter takes a comma-separated list of snake_case field names.
Expansion/embedding of related resources is a separate mechanism and is
deliberately not specified in this version of the standard.

> Provenance: walked decision "Field selection" (`baseline-02` decisions)
> · project policy · confidence moderate.

---

## 7. Caching and concurrency

### 7.1 Caching mechanism

**R7.1** Every response MUST carry explicit freshness information or an
explicit `no-store` — silence is not a decision; heuristic caching
(RFC 9111 §4.2.2) means an unlabeled response may still be cached by
intermediaries.

> Provenance: `HS-016` (batch, `baseline-01` §7) · protocol requirement
> (RFC 9111 §4.2.2) · confidence high.

**R7.2** Responses carrying user-specific or authenticated data MUST be
marked `private` or `no-store` — a shared cache may otherwise serve one
user's data to another. **Exception:** genuinely public responses.

> Provenance: `HS-017` (batch, `baseline-01` §7) · protocol requirement
> (RFC 9111 §3, §5.2.2.7 — a cross-user data-leak class) · confidence
> high.

### 7.2 Caching posture

**R7.3** Within the mechanism above, the default posture is three-tier:

1. Authenticated or mutable resources default to
   `Cache-Control: private, no-cache`, revalidating via the strong `ETag`
   machinery of R3.10 — cheap 304s, zero staleness.
2. `no-store` is reserved for genuinely sensitive payloads. Blanket
   `no-store` is a named anti-pattern: it discards the largest
   performance lever an API has.
3. `public` with `max-age` is permitted only for resources documented as
   immutable or deliberately stale-tolerant.

> Provenance: walked decision "Caching posture for mutable data"
> (`baseline-01` decisions) · protocol requirement (the leak mechanism) +
> project policy (the tier boundaries) · confidence moderate-high.

### 7.3 Concurrency control

**R7.4** Optimistic concurrency rides the conditional-request machinery of
R3.10/R3.11. This section tightens it for destructive operations:

1. DELETE of a resource exposed to concurrent modification MUST require
   `If-Match` (returning `428` when absent, `412` on mismatch) — for
   destructive operations the R3.11 SHOULD becomes MUST.
2. An unfiltered collection-level DELETE MUST NOT be offered; a bulk
   destructive operation MUST require an explicit filter or explicit
   item list.

> Provenance: Apparatus (Gate D) — gap review item R8.1 (destructive
> guards); a `[POLICY]` tightening of the ratified `HS-015` SHOULD.

---

## 8. Authentication, authorization, and security

### 8.1 Transport and credentials

**R8.1** APIs MUST be served only over TLS 1.2 or higher, preferring 1.3;
SSL 2/3 and TLS 1.0/1.1 MUST be rejected.

> Provenance: `OP-001` (batch, `baseline-03` §7) · protocol requirement
> (BCP 195 / RFC 9325) · confidence high.

**R8.2** Access tokens MUST NOT be accepted or emitted in URI query
parameters.

> Provenance: `OP-002` (batch, `baseline-03` §7) · protocol requirement
> (BCP 240 / RFC 9700) · confidence high.

**R8.3** The authentication mechanism follows the client class, and the
boundary is authority, not preference: OAuth/OIDC is REQUIRED wherever a
user delegates authority or a third party acts on a user's behalf (an API
key authenticates a caller; it cannot carry scoped, revocable, per-user
consent). API keys are acceptable for server-to-server traffic with a
single trust relationship.

> Provenance: walked decision "Auth mechanism per client class"
> (`baseline-03` decisions) · evidence-backed default (the OAuth rules)
> + project policy (the boundary) · confidence high.

**R8.4** Wherever OAuth is used: the resource owner password credentials
grant MUST NOT be used; public clients MUST use PKCE; redirect URIs MUST
be matched by exact string; the implicit grant MUST be avoided.

> Provenance: `OP-003`/`OP-004` (batch, `baseline-03` §7) · protocol
> requirement (BCP 240 / RFC 9700) · confidence high.

**R8.5** Credentials SHOULD be scoped and expiring; long-lived unscoped
tokens SHOULD be avoided.

> Provenance: `OP-025` (batch, `baseline-03` §7) · evidence-backed
> default (BCP 240) · confidence high.

### 8.2 Authorization

**R8.6** Every request MUST be authorized against the specific object,
not only the endpoint. The authorization decision is centralized — one
shared component invoked at every call site that resolves a
client-supplied ID — and enforced in the request handler, deny-by-default,
never gateway-enforced. Authorization tests block deployment.

> Provenance: `OP-005` (batch, `baseline-03` §7; OWASP API1:2023) + the
> five-axes object-level authorization default (walked, `baseline-03g`) ·
> protocol-adjacent security requirement + project policy `[POLICY]` (the
> enforcement placement) · confidence high.

**R8.7** Identifier unguessability MUST NOT be relied on as an access
control. (Its response-shape counterpart is existence masking, R5.10.)

> Provenance: `OP-006` (batch, `baseline-03` §7) · security requirement
> (OWASP API1:2023) · confidence high.

**R8.8** Readable and writable properties MUST be authorized per caller,
not per endpoint.

> Provenance: `OP-007` (batch, `baseline-03` §7) · security requirement
> (OWASP API3:2023) · confidence high.

**R8.9** Request bodies MUST be bound to an explicit allow-list of
writable fields.

> Provenance: `OP-008` (batch, `baseline-03` §7) · security requirement
> (OWASP API3:2023) · confidence high.

### 8.3 The deployment profile — five risk-based axes

**R8.10** Each axis below carries a ratified default and named
threat-model flip triggers. An API adopts the defaults unless a trigger
applies, and records any flip in its conformance note. The full trigger
tables and evidence live in the decision record (`baseline-03g`); the
normative skeleton:

| Axis | Default | Headline flip triggers |
| --- | --- | --- |
| Sender-constrained tokens | Bearer over TLS + short TTL + audience restriction + refresh-token rotation for public clients; validation SHOULD NOT hard-code the `Bearer` scheme | FAPI 2.0 / open banking → DPoP or mTLS; tokens visible to logging intermediaries; hostile-environment public clients → DPoP; existing PKI server-to-server → mTLS; per-operation value → RFC 9470 step-up |
| Token format | Opaque on the public wire; phantom-token pattern where a gateway exists; a client-visible JWT MUST be RFC 9068-conformant and paired with a revocation-propagation plan | Measured introspection bottleneck, AS-outage tolerance, or third-party resource servers → JWT; instant-revocation SLA or PII claims → stay opaque |
| Rate-limit aggressiveness | Multi-dimensional tiered posture, published: per-principal sustained + token-bucket burst (start ≈100 rps/account, 25 rps/endpoint, `[POLICY]` numbers); unauthenticated per-IP an order of magnitude lower; auth endpoints strictly stricter (start ≤5/min per IP+account); failed-auth budget; concurrency separate | Large per-request cost variance → cost/token accounting; metered third-party spend → spend caps; credential stuffing → lockout tier; multi-tenant → fair-share; free-tier abuse → spend/tenure gating |
| Replay window | 300 s past / 60 s future, asymmetric + mandatory dedup cache held at least the past window; NTP required; the window alone is never sufficient | Interactive high-value signing → 30–60 s; server-provided nonces remove skew; unmanaged clocks or store-and-forward → up to 15 min, never without dedup; signature omits body → add RFC 9530 binding |
| Object-level authorization | Centralized decision, in-handler enforcement (R8.6) | Relationship-derived permissions or cross-tenant sharing at scale → ReBAC; regulated audit or polyglot fleet → policy language; single service with an ownership column → stay embedded |

> Provenance: walked decision "Deployment profile — the five risk-based
> security axes" (`baseline-03` decisions + `baseline-03g`) · project
> policy `[POLICY]` throughout, grounded in BCP 240, RFC 9449/8705/9068,
> FAPI 2.0, and OWASP API Top 10 2023 · confidence per-axis in the
> decision record.

### 8.4 Server-side request forgery

**R8.11** Caller-supplied URLs (webhook targets, callbacks) MUST be
validated and restricted against internal address ranges.

> Provenance: `OP-023` (batch, `baseline-03` §7) · security requirement
> (OWASP API7:2023) · confidence high.

### 8.5 Redaction

**R8.12** Secrets, credentials, and PII MUST NOT appear in problem
`detail`, in echoed request input, or in any other response content.
(R5.17 bans internal implementation detail; this rule bans sensitive
caller data.)

> Provenance: Apparatus (Gate D) — gap review item R5.6.

---

## 9. Lifecycle, compatibility, versioning, and deprecation

### 9.1 Version marker

**R9.1** An API MUST use one uniform version-marker placement: the major
version in the path (`/v1`), never `/v1.0`; minor and patch versions
never appear in URIs.

> Provenance: `OP-015` (completed; walked, `baseline-03` decisions) ·
> project policy `[POLICY]` — the uniformity requirement is
> evidence-backed (both `survey-06` runs concur); the placement choice is
> policy · confidence moderate (a genuine fork; dated-header pinning is
> the strongest alternative).

**R9.2** Evolution within a major version is additive
(compatible-evolution-first); a major bump is a rare last resort.

> Provenance: `OP-015` (completed; AIP-181 posture) · project policy ·
> confidence moderate.

### 9.2 Stability tiers

**R9.3** Pre-GA stability is marked inside the version path segment,
AIP-style: `/v1alpha…` (experimental — no stability promise, MAY change
or vanish without notice, excluded from the deprecation policy, MAY be
access-gated), `/v1beta…` (preview — best-effort stability, breaking
changes with notice, outside the 12-month floor), `/v1` (GA — the full
deprecation policy applies). Graduation from a pre-GA tier to GA is an
explicit client migration, deliberately: the stability contract changed.

> Provenance: walked decision "Support tiers" (`baseline-03` decisions) ·
> project policy on AIP-185's channel convention · confidence
> moderate-high.

### 9.3 The frozen surface and the breaking-change taxonomy

**R9.4** Within a GA major version, the following surface is frozen —
compatible changes are permitted, breaking changes are not:

**Frozen (changing any of these is a breaking change):** paths and their
methods · request and response field names, types, and meanings · the
operation-to-status mapping (§5.2) · problem `type`/`code` pairs (R5.13.2)
· reserved-name semantics (§1.10) · documented default sort order (R6.6)
· header names and semantics · authentication requirements · documented
limits, in the tightening direction.

**Compatible (permitted within a major):** adding endpoints · adding
optional request fields or parameters · adding response fields · adding
enum values where the enumeration is documented open (R4.9) · adding new
problem types · relaxing a documented limit · adding an optional header.

**Breaking (a new major, or a pre-GA tier):** removing or renaming any
frozen element · changing a field's type or meaning · making an optional
input required · tightening validation on existing inputs · changing
defaults, including the default sort order · repurposing a status code ·
removing or narrowing an authentication mechanism.

> Provenance: Apparatus (Gate D) — gap review items R7.2/R9.3 (breaking
> change taxonomy + frozen-surface enumeration), anchored in ratified
> rules `AC-012`, A2.1, R5.13.2, and `OP-015`'s
> compatible-evolution-first posture.

### 9.4 Deprecation and sunset

**R9.5** Deprecation MUST be signaled with the `Deprecation` header
(RFC 9745, a structured-field Date) and a `Sunset` header (RFC 8594,
an HTTP-date) whose timestamp is not earlier than the deprecation date.
The two headers use deliberately different date formats and are not
interchangeable; RFC 8594 is Informational while RFC 9745 is Standards
Track.

> Provenance: `OP-013` (batch, `baseline-03` §7) · protocol requirement ·
> confidence high.

**R9.6** Every deprecation MUST carry a `deprecation` link relation to
human-readable migration documentation, and an element MUST NOT be
deprecated without a sunset date.

> Provenance: `OP-014` (batch, `baseline-03` §7) · protocol requirement
> (RFC 9745) · confidence high.

**R9.7** A deprecated GA major version remains fully supported for at
least 12 months after its successor ships, and the sunset date is
announced the day deprecation starts. The floor permits longer promises.

> Provenance: walked decision "Deprecation window and version overlap"
> (`baseline-03` decisions) · project policy · confidence moderate.

---

## 10. Asynchronous work, bulk operations, and webhooks

Rules in §10.1 are scoped by the `async-operations` switch, §10.2 by
`bulk-operations`, and §10.3–§10.4 by `webhooks` (§1.8).

### 10.1 Asynchronous operations

**R10.1** Every `202 Accepted` MUST return an addressable operation
resource with defined terminal states, an expiry, and a failure
representation — a 202 with no status resource strands the client.

> Provenance: `AC-019` (batch, `baseline-02` §7) · evidence-backed
> default (RFC 9110 §15.3.3) · confidence high.

**R10.2** A `202` response and the in-flight operation resource SHOULD
carry `Retry-After` as a polling hint, and the operation's documentation
MUST state the expected polling cadence. Where an in-flight operation can
be abandoned, cancellation is expressed as the `cancel` action (§1.10) on
the operation resource.

> Provenance: Apparatus (Gate D) — gap review items R7.4/R10.4
> (polling/cancellation guidance), riding `AC-019` and addendum A5.

**R10.3** An API MAY accept `Prefer: respond-async` (R4.15) to request
asynchronous processing; honoring it remains at the server's discretion.

> Provenance: `AC-020` (batch, `baseline-02` §7) · evidence-backed
> default · confidence moderate.

### 10.2 Bulk operations

**R10.4** Every bulk endpoint MUST state explicitly whether it is atomic
or partial, and a partial endpoint MUST represent per-item outcomes —
otherwise the client cannot know what to retry. Partial completion
returns the `200` envelope per R5.8; each per-item outcome carries its
own status and, for creates, the created resource's URI. Batch semantics
are never expressed as collection-level actions (R2.13).

> Provenance: `AC-018` (batch, `baseline-02` §7) + addendum A3.2 ·
> evidence-backed default · confidence high.

### 10.3 Webhook delivery

**R10.5** Delivery MUST be documented as at-least-once with no ordering
guarantee, and every event MUST carry a monotonic version or timestamp so
consumers can discard stale state.

> Provenance: `OP-021` (batch, `baseline-03` §7) · evidence-backed
> default · confidence high.

**R10.6** The provider MUST publish the acknowledgement timeout and the
retry schedule. Failed deliveries are retried with exponential backoff
for at least 72 hours; after retries exhaust, the delivery is held in a
dead-letter store for at least 30 days with a self-service redelivery
API.

> Provenance: `OP-022` (batch, `baseline-03` §7; the published schedule)
> + walked decision "Webhook retry and dead-letter policy" (`baseline-03`
> decisions) · project policy · confidence moderate.

### 10.4 Webhook signing

**R10.7** Every outbound webhook MUST be signed over a base that binds a
unique delivery ID, a timestamp, the raw body, and any metadata the
consumer is expected to act on. The scheme is selected by trust topology:

1. **Shared-secret topology** (the ordinary product webhook — the
   provider issues each consumer a secret): the Standard Webhooks
   scheme — HMAC-SHA256 over `id.timestamp.payload`, carried in the
   `webhook-id`, `webhook-timestamp`, and `webhook-signature` headers,
   with `whsec_`-prefixed secrets. `[POLICY]` Standard Webhooks is a
   vendor-TSC specification, not a standards-body product.
2. **Cross-organization topology** (consumers cannot hold a shared
   secret, or key custody / HSM / non-repudiation requirements apply):
   RFC 9421 HTTP Message Signatures with RFC 9530 `Content-Digest` as a
   covered component, keys published at a discoverable key set, and the
   `webhook-id`/`webhook-timestamp` delivery envelope retained.
3. Bespoke per-vendor HMAC schemes are not a sanctioned default.
4. SHA-1 is prohibited (NIST retires SHA-1 for all applications by
   2030-12-31 and SHA-256 costs nothing more — not because HMAC-SHA1 is
   broken; RFC 6194 §3.3: it is not).
5. Any asymmetric-signature requirement routes to the RFC 9421 branch:
   Standard Webhooks' `v1a`/ed25519 mode is safe only with a distinct
   key pair per endpoint (documented by the spec's own lead author,
   issue #34).

Honesty note, carried from ratification: no documented in-the-wild replay
incident was located; the signed-timestamp requirement rests on
defense-in-depth reasoning and unanimous vendor and specification
practice.

> Provenance: `OP-016` (walked as amended — topology split, `baseline-03`
> decisions + `03c`/`03d`) · evidence-backed default (the signing MUST
> and signed base) + project policy `[POLICY]` (naming the schemes) ·
> confidence high (signing MUST), moderate-high (Standard Webhooks
> branch), moderate (RFC 9421 branch).

**R10.8** Provider-side signing invariants: per-endpoint secrets of at
least 256 bits (`[POLICY]` — deliberately stricter than the Standard
Webhooks 192-bit floor; implementations under this standard reject
shorter secrets) · overlapping active signing secrets with a published
rotation procedure · HTTPS-only delivery · unknown and legacy schemes
rejected · verification made the default path (an SDK helper, published
test vectors including negative vectors) · the documentation states the
boundaries: signing is authentication of origin and integrity, not
authorization, and not a TLS substitute.

> Provenance: `OP-016` invariants I6–I13 + `OP-024` (batch, `baseline-03`
> §7) · project policy `[POLICY]` (secret floor) + evidence-backed
> defaults · confidence high.

Consumer-side verification obligations — raw-body verification, bounded
timestamp tolerance, dedup, constant-time comparison, fail-closed
configuration — are consolidated in §12.4. This placement resolves the
question the `OP-016` ratification left open for Phase 3 drafting.

---

## 11. Rate limits, retries, observability, and operational behavior

### 11.1 Capacity limits

**R11.1** An API MUST publish and enforce a maximum page size, expansion
depth, and bulk item count.

> Provenance: `OP-009` (batch, `baseline-03` §7) · security requirement
> (OWASP API4:2023) · confidence high.

### 11.2 Rate limiting

**R11.2** An API MUST apply rate limits and communicate exhaustion via
the published HTTP mechanism: `429 Too Many Requests` with `Retry-After`.
This MUST is a deliberate tightening of RFC 6585 §4's MAY — house policy
over a published standard.

**R11.3** An API SHOULD additionally advertise quota state using
`RateLimit` and `RateLimit-Policy` in the syntax of
`draft-ietf-httpapi-ratelimit-headers-11`. `[POLICY]` These fields are an
unpublished Internet-Draft, not a standard; they MUST NOT be described as
standards-compliant, and the pinned revision MUST be cited wherever they
are referenced.

**R11.4** Any proprietary quota headers MUST be documented explicitly,
including — for any reset field — whether the value is an absolute epoch
timestamp or a delta in seconds (the one documented ambiguity that breaks
real clients: GitHub and Zalando define the same header name with
opposite meanings). Proprietary and draft fields MAY coexist; whatever is
emitted is documented.

> Provenance (R11.2–R11.4): `OP-010` (walked as re-framed, `baseline-03`
> decisions + `03e`/`03f`) · protocol requirement (RFC 6585 + RFC 9110,
> deliberately tightened) + project policy `[POLICY]` (the SHOULD on
> draft fields) · confidence moderate-high (mandate), moderate (SHOULD
> clause). Re-check triggers are registered in `research/README.md`
> (semi-annual; next 2027-02-09).

**R11.5** `429` signals quota exhaustion; `503 Service Unavailable`
signals capacity overload; both MUST carry `Retry-After`.

> Provenance: `OP-011` (batch, `baseline-03` §7) · protocol requirement
> (RFC 9110 §15.6.4; RFC 6585) · confidence high.

The rate-limit *posture* — dimensions, starting numbers, stricter auth
endpoint tiers — is the third axis of the §8.3 deployment profile.

### 11.3 Retryability

**R11.6** An API MUST document which failure classes are retryable. The
matching client obligations (backoff, jitter, idempotency keys) are
R12.1.

> Provenance: `OP-012` (batch, `baseline-03` §7; provider half) ·
> evidence-backed default · confidence high.

### 11.4 Observability

**R11.7** Every response, including errors, MUST carry the `request-id`
correlation header (§1.10).

> Provenance: `OP-018` (batch, `baseline-03` §7) + addendum A2.4 (the
> name) · evidence-backed default + project policy `[POLICY]` (the
> name) · confidence high.

**R11.8** W3C Trace Context `traceparent` MUST be propagated;
`tracestate` is accepted subject to a published size and entry cap.

> Provenance: `OP-019` (batch, `baseline-03` §7) · protocol requirement
> (W3C Trace Context) · confidence high.

---

## 12. Client obligations

The preceding sections bind providers. This section consolidates the
obligations that bind consumers of a conforming API; a provider MUST
surface them in its documentation. Rules here restate no provider
obligation — each carries its own identifier and cites the shared
provenance.

### 12.1 Retries and pacing

**R12.1** Clients MUST retry only failures the API documents as
retryable, using exponential backoff with jitter, and MUST NOT retry a
non-idempotent request without an idempotency key (R3.9).

> Provenance: `OP-012` (batch, `baseline-03` §7; client half) ·
> evidence-backed default · confidence high.

**R12.2** Clients MUST honor `Retry-After` on `429` and `503` responses
rather than retrying on their own schedule.

> Provenance: Apparatus (Gate D) — client half of `OP-011`, named as new
> §12 content by the gap review (item B.9).

**R12.3** Clients MUST set explicit request timeouts, and MUST NOT
disable TLS certificate verification in any environment that reaches a
real API host.

> Provenance: Apparatus (Gate D) — gap review item B.9 (client timeouts,
> TLS verification).

### 12.2 Tolerant reading

**R12.4** Clients MUST tolerate unknown response fields and unknown enum
values (R4.9): ignore what you do not recognize, never fail on it. This
is what makes additive evolution (R9.4) non-breaking in practice.

> Provenance: `AC-012` (batch, `baseline-02` §7; client half) +
> Apparatus (Gate D) for the unknown-field generalization · confidence
> moderate.

**R12.5** Clients MUST treat cursors (R6.3) as opaque: never construct,
modify, or persist them beyond their documented lifetime.

> Provenance: `AC-013` (batch, `baseline-02` §7; client half) ·
> evidence-backed default · confidence high.

### 12.3 Error handling

**R12.6** Clients SHOULD send `Accept: application/problem+json`
explicitly when they want problem documents — many HTTP libraries do not
treat `application/problem+json` as a subset of `application/json`.

**R12.7** Clients MUST NOT rely on every error being a problem document:
infrastructure outside the application (R5.12's carve-out) may answer
first, with any shape. Robust error handling branches on the status code
first and parses the body opportunistically.

> Provenance: `AC-003` ratification consequences (client-robustness note
> and media-type interop hazard, `baseline-02` decisions) ·
> evidence-backed default · confidence moderate.

### 12.4 Webhook consumers

**R12.8** A webhook consumer MUST verify the signature over the raw
request body before parsing; MUST enforce a bounded, non-zero timestamp
tolerance (300 seconds is the convergent default, per the §8.3 replay
axis); MUST deduplicate on the signed delivery ID for at least the
tolerance window; MUST compare signatures in constant time; and MUST
fail closed on a missing, empty, or default secret at configuration
load.

> Provenance: `OP-017` (batch, `baseline-03` §7) + `OP-016` invariants
> I1–I3/I5/I7 (walked, `baseline-03` decisions) · evidence-backed
> default — every documented webhook failure is a receiver-side
> verification failure · confidence high.

**R12.9** A webhook consumer MUST acknowledge delivery within the
provider's published timeout before processing the event body
(ack-before-processing), and MUST be prepared for at-least-once,
unordered delivery (R10.5).

> Provenance: `OP-022` (batch, `baseline-03` §7; consumer half) ·
> evidence-backed default · confidence high.

---

## Part II — Decision Log

*Drafted later in Phase 3. One row per ratified decision, mapping rule IDs
(R-numbers) to research-provenance IDs (`HS-*`, `AC-*`, `OP-*`, walked
decisions, addenda A1–A5) and linking each to its record in
[`research/decisions/`](research/decisions/).*
