# REST API Design Standard

**Status:** Phase 3 working draft — not approved. Gate D (approval for
systematic review) has not run. Part I currently contains §1; the remaining
sections are drafted incrementally per [`PLAN.md`](PLAN.md) Phase 3.

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
| `dry_run` | Rehearse a mutation without executing it | Support is MAY per endpoint, SHOULD for destructive and bulk operations; unsupported ⇒ `400` per R1.9; response carries an explicit dry-run marker and MUST NOT consume an `Idempotency-Key` | Addendum A4 `[POLICY]` |
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

## Part II — Decision Log

*Drafted later in Phase 3. One row per ratified decision, mapping rule IDs
(R-numbers) to research-provenance IDs (`HS-*`, `AC-*`, `OP-*`, walked
decisions, addenda A1–A5) and linking each to its record in
[`research/decisions/`](research/decisions/).*
