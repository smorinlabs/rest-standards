# REST API Design Standard

**Version 1.1.0.** Version 1.0.0 was released 2026-08-10 after Gates C, D,
and E all passed (decision layer ratified 2026-08-09; draft approved
2026-08-09 and systematically reviewed through 2026-08-10; review history in
[`docs/reviews/`](docs/reviews/)). Version 1.1.0 adds **§13, streaming
responses**, under the Part II amendment rule — a MINOR bump, since it adds
rules and scopes nine existing ones without strengthening or removing any.
This document, with
[`conformance/spectral.yaml`](conformance/spectral.yaml),
[`conformance/fixture-violations.yaml`](conformance/fixture-violations.yaml),
and the informative companion
[`streaming-profile.md`](streaming-profile.md), is the released standard;
changes follow the Part II amendment rule and are recorded in
[`CHANGELOG.md`](CHANGELOG.md).

**Provenance model:** this document transcribes decisions ratified in the
decision layer (recorded in [`research/decisions/`](research/decisions/));
it does not make policy. Gate C and its addendum ratified §1–§12; the
Phase 6 streaming walk (2026-08-10) ratified §13. Provisions with no
decision record — the document's own conformance apparatus — are marked
**Apparatus** and were ratified en bloc at Gate D (2026-08-09), with Phase 6
additions recorded in the same Part II apparatus register.

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
resource-oriented HTTP API must be settled. Streaming responses — Server-Sent
Events, long-polling, and streaming HTTP bodies — were out of scope for
version 1.0 and are governed by **§13** as of version 1.1.0.

**WebSockets are a stated non-goal**, not a deferral. After a `101 Switching
Protocols` upgrade the exchange is no longer HTTP request/response: there is
no status code, no response media type, and no request to which a conditional
header, a problem document, or an idempotency key could attach. None of this
standard's apparatus reaches it. An API offering a WebSocket surface is
neither conformant nor nonconformant on it; the surface is simply outside
what this document specifies.

> Provenance: `PLAN.md` scope; Gate A. The Gate D streaming deferral
> (2026-08-09) was discharged by the Phase 6 scope ruling (2026-08-10),
> `research/decisions/baseline-04-streaming.decision.md`, which also made
> WebSockets a stated non-goal · project policy.

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
`AC-001`–`AC-021`, `OP-001`–`OP-025`, and `ST-001`–`ST-020` are
**research-provenance keys** from the decision layer — the first three from
Gate C (2026-08-09), `ST-*` from the Phase 6 streaming walk (2026-08-10).
They are cited in provenance lines
throughout this document and remain the keys into
[`research/decisions/`](research/decisions/). No new identifier is ever
minted in those series: a drafted rule that had no proposed principle would
otherwise acquire fabricated research lineage. The full two-way mapping
between rule IDs and provenance IDs is maintained in Part II (Decision
Log). Where a provenance line cites the CLI-standards gap review, the
gap review's own rule numbers carry a `CLI-` prefix (for example
`CLI-R4.3`); those identifiers belong to the CLI Design Standard's
coverage table, never to this document's rule-ID namespace.

**Section namespace.** The `R<section>` prefix space is fixed at thirteen
normative sections, in this order: 1 purpose/conformance · 2 resources and
URI modeling · 3 methods, safety, idempotency, conditionals · 4 requests,
representations, negotiation, schemas · 5 status codes and errors ·
6 collections, pagination, filtering, sorting · 7 caching and concurrency ·
8 authentication, authorization, security · 9 lifecycle, versioning,
deprecation · 10 asynchronous work, bulk, webhooks · 11 rate limits,
retries, observability · 12 client obligations · 13 streaming responses.
Reserving the numbering here keeps section-prefixed rule IDs stable while
sections are drafted.

The space was declared at twelve in version 1.0.0 and extended to thirteen
in version 1.1.0 when §13 was added under the amendment rule. Extending it
is a MINOR change: no existing identifier moves, and `R1.2`'s guarantee that
an identifier is never renumbered or reused is unaffected.

> Provenance: Apparatus — ratified at Gate D 2026-08-09 — rule-ID mapping policy delegated to
> Phase 3 drafting by `PLAN.md` and the gap review (item B.13). Extended to
> thirteen sections by the Phase 6 deliverable-shape ruling `P6-D0`
> (2026-08-10), `research/decisions/baseline-04-streaming.decision.md`.

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

> Provenance: Apparatus — ratified at Gate D 2026-08-09 — gap review item B.8, adapted from the
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
`streaming`. Every switch controls at least one rule; the vocabulary grows
only when a new rule needs a switch, so a declaration never exists that
waives nothing. Capability facts with no rule attached — tenancy model, PII
handling, client audience — belong in the conformance note's free text, not
here.

**Where a rule's scope is stated.** A switch-scoped rule declares its scope
in its own provenance line, and the section that owns the capability
summarizes the scoping for its rules: §10 for `webhooks`,
`async-operations`, and `bulk-operations`; §13 for `streaming`. A rule may
live outside the section that summarizes its switch — R12.10 is scoped by
`streaming` but belongs in §12 with the other client obligations — so the
provenance line, not the section, is authoritative for scope.

**A rule may require two switches.** Where a rule governs the meeting of two
capabilities, it binds only when both are on and says so: R13.9 binds only
where `streaming` and `async-operations` are both on. An API with either
switch off takes it as not applicable, with the reason its own off-switch
declaration already gives.

**A switch never waives a guard.** A rule whose whole purpose is to define
what an API without the capability must do is not scoped by that
capability's switch — scoping it there would delete it for exactly the
endpoints it binds. Two such rules exist, and each says so in its own text:
`R1.9` (the `dry_run` rejection guard) and `R13.3` (the `stream` rejection
guard). Both bind **per endpoint**, not per API: an API that streams on some
endpoints still owes the guard on every endpoint that does not.


> Provenance: Apparatus — ratified at Gate D 2026-08-09 — gap review item B.8.

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
  bulk-operations=<on|off>, streaming=<on|off>
  (every switch declared off carries a one-line reason)
Context: <free text — tenancy model, PII handling, client audience,
  and other capability facts no rule attaches to>

Deviations:
- <rule ID> · <rule strength> · what differs · why · approver · date

N/A declarations:
- <rule ID or switch> · why it cannot apply
```

> Provenance: Apparatus — ratified at Gate D 2026-08-09 — gap review item B.12 (no-silent-deviation
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
a mutating request carrying the `dry_run` parameter — with any value — to
an endpoint that does not implement dry-run MUST be rejected with `400`,
never silently executed.
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
| `stream` | Request incremental delivery of this response | Reserved as a request modifier in **either** carriage — query parameter or request-body member. It never selects the response media type (R13.2 puts that on `Accept`); an endpoint that does not stream and receives it ⇒ `400` per R13.3 | `P6-D1` (`ST-003`) `[POLICY]` |
| `stream_position` | A stream's strictly increasing position, for resumption | Reserved in **either** carriage when resuming — query parameter or request-body member, matching how the stream itself is requested — and the same name carries the position on every frame. Distinct from `cursor`: a stream position has visible ordering, where a cursor is opaque and non-constructable (R12.5). Clients echo it and never compute one | `P6-D5` (`ST-010`) `[POLICY]` |
| `<field>[gte]`, `<field>[gt]`, `<field>[lte]`, `<field>[lt]` | Range filters on collection lists | The only permitted bracketed query-parameter forms; AND-combined; base name obeys the `AC-007` grammar | Walked decision "Filter grammar" (`baseline-02` decisions) `[POLICY]` |

#### Reserved headers

| Name | Direction | Meaning | Provenance |
| --- | --- | --- | --- |
| `Idempotency-Key` | Request | Idempotency key on non-idempotent state-changing requests; Stripe semantics — payload fingerprint, reuse with a different payload rejected; retained ≥ 24 h. `[POLICY]` — the IETF draft that standardized this shape expired 2026-04-18; never cite it as a standard | `AC-016`/`AC-017` (completed) |
| `request-id` | Response | Correlation ID, emitted on every response including errors. Lowercase name; RFC 6648 deprecates new `X-` prefixed fields, ruling out `X-Request-Id` | Addendum A2.4, completing `OP-018` `[POLICY]` |
| `ETag` / `If-Match` / `If-None-Match` | Response / request | Strong validators and conditional requests | `HS-014`/`HS-015` · protocol requirement (RFC 9110) |
| `Location` | Response | Bound on every single-resource create (`201 Created`, R5.6) and, at SHOULD strength, on `202` operation responses — denoting the operation, never the result (R10.9); its RFC 9110 §10.2.2 redirect meaning on 3xx is untouched | Addendum A3.1 · `baseline-02i` + Phase 4 owner walk |
| `Allow` | Response | Mandatory on every `405 Method Not Allowed` | Addendum A3 · protocol requirement (RFC 9110) |
| `Retry-After` | Response | Mandatory on `429` and on `503` (R11.5); recommended polling hint on `202` (R10.2) | `OP-010`/`OP-011` · protocol requirement (RFC 9110, RFC 6585) |
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
| `text/event-stream` | Server-Sent Events framing; the default for incrementally generated content (R13.4). `[POLICY]` — **this media type has no IANA registration**: it is absent from the `text/*` subregistry and its per-type registry URL returns `404` (probed 2026-08-10). The WHATWG text carries a registration template that has never been submitted. It MUST NOT be described as IANA-registered — it **is** standardized by the WHATWG HTML Living Standard, which is a separate claim | `P6-D4a` (`ST-004`) |

#### Reserved action verbs (path segments)

Core registry for the `POST /{collection}/{id}/{action}` form. An API using
one of these verbs MUST mean exactly this; an action segment can never be
used as a collection name under the same parent. One verb per meaning,
API-wide; kebab-case for multi-word verbs.

| Verb | Registered meaning | Provenance |
| --- | --- | --- |
| `cancel` | Terminal, irreversible stop of any in-flight state — a pending order, a running operation | Addendum A5 `[POLICY]`; scope confirmed at the Phase 4 owner walk (2026-08-10) |
| `archive` / `restore` | Reversible visibility pair — removes a resource from default listings for every audience (the soft-delete modeling R5.7 references) | Addendum A5 `[POLICY]` |
| `approve` / `reject` | Review outcomes | Addendum A5 `[POLICY]` |
| `publish` / `unpublish` | Consumer-visibility pair — controls whether an otherwise-existing resource is visible to external consumers | Addendum A5 `[POLICY]` |
| `duplicate` | Copy; returns `201` + `Location` | Addendum A5 `[POLICY]` |

#### Reserved stream members

Member names a stream's frames carry, registered so that generic tooling and
a client reading two APIs find the same concept under the same name.

| Name | Where | Registered meaning | Provenance |
| --- | --- | --- | --- |
| `operation_id` | Any frame of a stream that has an operation resource | The identifier R10.9 binds into the `202` body, where the API uses R10.9's `id` form. Carrying it is what makes R13.9's one-identity obligation checkable | `P6-D0` batch (`ST-009`) `[POLICY]` |
| `operation_url` | Same | The absolute URI of the operation resource, where the API uses R10.9's `url` form instead of `id`. Exactly one of the two is carried, matching whichever form the `202` body uses | `P6-D0` batch (`ST-009`) `[POLICY]` |
| `operation_state` | A terminal frame | The operation's terminal-state value, drawn from the vocabulary R10.1 requires the operation resource to document (`succeeded`, `failed`, `canceled`, …). This is the member R13.9 compares across the two channels. Deliberately **not** named `status`: R13.7 forbids a `status` member on an `error` frame's problem object, and one name for the terminal state across both success and failure frames is what makes the comparison mechanical | Phase 6 review walk `[POLICY]` |
| `retry_after` | An `error` frame | Seconds a client should wait before retrying, carrying the pacing hint that the `Retry-After` header would have carried had a status still been available (R11.2 streaming scope). Same semantics and units as `Retry-After`'s delay-seconds form | Phase 6 review walk `[POLICY]` |

#### Reserved stream frame types

Frame-type names carry the same "same concept, same name" obligation as the
categories above: a client — or generic tooling that never read the API's
documentation — must be able to recognize these frames by name alone. An API
that streams MUST use the registered name for the registered meaning, and
MUST NOT give it another. The API's own frame-type vocabulary is otherwise
its own (R13.5).

| Frame type | Registered meaning | Provenance |
| --- | --- | --- |
| `error` | The frame carrying an error raised after the response status was committed. Its payload is a problem details object with `status` omitted, per R13.7 | `P6-D4b` (`ST-007`) `[POLICY]` |

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
| **Destructive operation** | A mutating request that removes data or irreversibly ends a process: DELETE, and any action verb registered as irreversible (`cancel`). Reversible-visibility actions (`archive`) are not destructive. |
| **Reserved name** | A query parameter, header, media type, action verb, stream member, or stream frame type registered in §1.10. |
| **Stream** | A response delivered incrementally over a single HTTP request, as a sequence of frames (§13). |
| **Frame** | One self-delimited unit of a stream, carrying a type (R13.5) and a payload. |
| **Terminal frame** | The frame that ends a stream and carries its final outcome (R13.6); its absence means the stream was truncated. |
| **Self-delimiting stream media type** | A media type whose own specification lets a parser find each frame's boundaries from the body alone, without relying on `Content-Length` or on the connection closing. `text/event-stream` and `application/x-ndjson` qualify; `application/json` over concatenated documents does not. |
| **Status committed** | The moment the application has issued the response status line and headers to the HTTP layer. Commitment is judged at that point regardless of downstream buffering, so a proxy holding bytes does not move the boundary between R13.8 and R13.7. |
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

**R2.3** Resource names MUST be unabbreviated, and each concept MUST have
exactly one noun API-wide (both checkable against the contract document).
Names SHOULD be the noun the business domain itself uses — a judgment
call, reviewed rather than machine-checked. The same concept MUST carry
the same name wherever it appears — path, query parameter, body field,
header — differing only by the casing rules of each surface (R2.4, R4.4).

> Provenance: Apparatus — ratified at Gate D 2026-08-09 — gap review items CLI-R2.2/CLI-R3.8 (noun naming,
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
(`GET /orders?customer=…`). Counting rule: resources are the noun segments
(collections and singletons) with their identifiers; the version segment
(R9.1) and a trailing action segment (§2.4) do not count.
`/v1/orders/{order_id}/line-items/{line_item_id}/adjustments/{adjustment_id}`
sits exactly at the ceiling.

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

> Provenance: `HS-004` (batch, `baseline-01` §7) · evidence-backed
> default, protocol-grounded (RFC 9110 §3.1 resource identity; BCP 190) —
> no RFC mandates identity stability; the MUST is this standard's ·
> confidence high.

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

> Provenance: Apparatus — ratified at Gate D 2026-08-09 — gap review item CLI-R2.3, grounded in
> `HS-004` and the ratified filter grammar (§6).

**R2.10** Personally identifiable information MUST NOT appear in any URI —
path or query string. URIs land in access logs, browser history, referrer
headers, and URL-keyed caches by default. Identify people by opaque IDs.

> Provenance: Apparatus — ratified at Gate D 2026-08-09 — gap review item CLI-R9.7, adjacent to
> `OP-002` (which bans tokens in query strings).

### 2.4 Actions — operations that resist CRUD

**R2.11** A non-CRUD operation MUST use the sub-path verb form
`POST /{collection}/{id}/{action}` (for example
`POST /payment-intents/{id}/capture`). An action segment is a documented
verb and MUST NOT be used as a collection name under the same parent. The
core verb registry, with fixed meanings, is in §1.10; a domain verb beyond
the core registry is permitted with a per-API registry entry, and an API
MUST use one verb per meaning API-wide. Response shape: a synchronous
action returns `200 OK` with the mutated representation; a long-running
action that does not stream returns `202 Accepted` with the R10.1 operation
resource; a long-running action that streams its progress returns `200` with
a stream media type per R13.1, and R13.9 binds the two channels where both
are offered; `duplicate` returns `201` with `Location` per R5.6.

> Provenance: walked decision "Structural lock — Custom-action syntax" +
> addendum A5.1/A5.2 (`baseline-01`/`baseline-02` decisions) · project
> policy · confidence moderate. Response-shape clause scoped for streaming
> in version 1.1.0 by the Phase 6 review walk (2026-08-10),
> `research/decisions/baseline-04-streaming.decision.md`, resolving its
> collision with R13.1's prohibition on a streaming `202`.

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
A resource within this rule's scope by definition supports conditional
update, so R3.10's strong-`ETag` obligation applies to it — the two rules
gate the same resource class. **Exception:** single-writer resources;
append-only collections.

> Provenance: `HS-015` (batch, `baseline-01` §7) + addendum A3 drafting
> row (428) · protocol-grounded machinery (RFC 9110 §13.1.1; RFC 6585 §3
> defines 428); requiring the precondition is an evidence-backed
> default · confidence high.

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

**R4.16** Path placeholders — URI Template variable names and OpenAPI
`in: path` parameter names, such as `{order_id}` — MUST use snake_case,
the body-property grammar `^[a-z_][a-z_0-9]*$`. Placeholders never
appear on the wire, but they appear throughout documentation and the
contract document, and the value of the rule is uniformity itself.
(Numbered out of prose order per R1.2: rules take the next unused
number in their section.)

> Provenance: Apparatus — ruled by the owner at the Phase 4 walk
> (2026-08-10), closing the Part II register candidate; enters the
> Gate E approval · project policy `[POLICY]` · confidence high (the
> same uniformity logic as `AC-007`).

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

> Provenance: Apparatus — ratified at Gate D 2026-08-09 — gap review item CLI-R4.2
> (content-negotiation defaults).

**R4.11** Every response whose content was selected by a request header
MUST send `Vary` listing each header that influenced selection.
**Exception:** responses with no negotiation.

> Provenance: `HS-018` (batch, `baseline-01` §7) · protocol-grounded
> (RFC 9110 §12.5.5, where `Vary` on cacheable responses is a SHOULD);
> the unconditional MUST is a `[POLICY]` tightening · confidence high.

**R4.17** Where an API serves cross-origin browser clients, every
response MUST list the standard-bound response headers it emits — among
`request-id`, `ETag`, `Location`, `Retry-After`, `RateLimit`,
`RateLimit-Policy`, `Deprecation`, `Sunset`, and any other §1.10
response header in use — in `Access-Control-Expose-Headers`. None of
those fields is CORS-safelisted, so without this header a cross-origin
browser client cannot read them at all. (Numbered out of prose order
per R1.2.)

> Provenance: `baseline-02i` CORS surfacing (WHATWG Fetch + MDN,
> primary-sourced); ruled option (a) by the owner at Gate E
> (2026-08-10) · project policy `[POLICY]` — the invisibility mechanism
> is fact; the exposure mandate is this standard's · confidence high.

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
client MUST NOT depend on one being honored. That advisory nature is the
ground on which `Prefer: validate-only` was declined in favor of
`dry_run` (§1.10): a token a server may ignore cannot carry safety
semantics.

> Provenance: `AC-020` (batch, `baseline-02` §7) + addendum A4 rationale ·
> evidence-backed default · confidence moderate.

---

## 5. Status codes and errors

### 5.1 Status-code discipline

**R5.1** The status code MUST match the registered semantics of the
outcome **as that outcome is known when the status is generated**. A failed
operation MUST NOT return 2xx.

**Streaming scope.** In a streaming response the status is committed before
the outcome exists, so a stream that begins successfully and later fails has
already sent `200`. That is not a violation of this rule: the status was
correct for the outcome known when it was generated. The failure is reported
in-band under R13.7, and the full problem document — carrying `status` — is
retrievable from the operation resource under R13.9 where one exists. This
scope is stated here, rather than only in §13, because a reader of §5 alone
would otherwise conclude that §13 contradicts this rule.

> Provenance: `HS-010` (batch, `baseline-01` §7) · protocol requirement
> (RFC 9205) · confidence high. Streaming scope added in version 1.1.0 by
> `P6-D3`, `research/decisions/baseline-04-streaming.decision.md` ·
> project policy `[POLICY]` · confidence high.

**R5.2** Unregistered status codes MUST NOT be defined or used.

> Provenance: `HS-003` (batch, `baseline-01` §7) · protocol requirement
> (RFC 9205 §4.6) · confidence high.

**R5.3** `422 Unprocessable Content` SHOULD be used for semantically
invalid but well-formed requests, `409 Conflict` for conflicts with
current resource state; `400 Bad Request` remains correct for malformed
syntax.

> Provenance: `HS-011` (batch, `baseline-01` §7) · evidence-backed
> default over protocol-defined codes (RFC 9110 §15.5.21 defines 422;
> the usage rule is this standard's) · confidence high.

**R5.4** `410 Gone` SHOULD NOT be returned unless permanence is actually
known and recorded. **Exception:** tombstoned resources with retained
deletion records.

> Provenance: `HS-012` (batch, `baseline-01` §7) · evidence-backed
> default grounded in RFC 9110 §15.5.11's permanence semantics ·
> confidence high.

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

> Provenance: addendum A3.1 (`baseline-01` decisions) · protocol-grounded
> (RFC 9110 §15.3.2/§9.3.3, where `Location` on `201` is a SHOULD); the
> unconditional MUST is a `[POLICY]` tightening · confidence high.

**R5.7** A successful DELETE returns `204 No Content` with an empty body.
**Exception:** an API that soft-deletes (marks deleted but keeps the
resource readable) returns `200 OK` with the tombstoned representation,
because a representation still exists. Reversible visibility modeled as
an explicit action pair uses the registered `archive`/`restore` verbs
(§1.10) rather than overloading DELETE.

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
> protocol requirements (RFC 9110 §15.5.6 — `Allow` on `405` is the
> RFC's own MUST — and §15.5.16; RFC 6585 §3) · confidence high.

### 5.3 Errors

**R5.12** Every API MUST be capable of returning every error response the
application itself generates as `application/problem+json` (RFC 9457)
when the client requests it. **Exception (named carve-out):** errors
emitted by infrastructure components outside application control —
reverse proxies, CDNs, WAFs, rate limiters, load balancers terminating
before application code — which MUST be documented as such.
**Second exception (named carve-out):** an error raised after a streaming
response's status was committed, which cannot be served as a response at all
and is delivered in-band under R13.7 instead. That carve-out preserves
everything this rule exists to guarantee — `type`, `title`, `code`, and the
R5.16 catalog entry all still apply — and surrenders exactly two things, the
media-type label and the advisory `status` member, both structurally
unavailable once the status is on the wire. Nothing in
this standard is premised on the IANA HTTP Problem Types registry. A
provider MAY additionally serve the identical problem body under
`application/json` when the client's `Accept` asks for it (the Cloudflare
mirroring pattern) as a compatibility measure; this standard neither
requires nor forbids it. The
matching client obligation — never *relying* on a problem document —
is R12.7.

> Provenance: `AC-003` (walked as amended, `baseline-02` decisions +
> `02d`/`02e`/`02f`) · evidence-backed default resting on a Standards
> Track RFC · confidence moderate (re-argued: no credible alternative
> exists). Second carve-out (post-commit stream errors) added in version
> 1.1.0 by `P6-D2`, `research/decisions/baseline-04-streaming.decision.md`
> · project policy `[POLICY]` · confidence moderate-high.

**R5.13** Every problem document **carried in an HTTP response** MUST carry
`type`, `title`, `status`, and a stable machine-readable `code` extension
member, bound as follows. (A problem object delivered in-band inside a
stream frame carries the same members **except `status`**, which R13.7
requires to be omitted; RFC 9457 §3.1 makes a `status` that disagrees with
the actual response status a violation, and the actual status there is
`200`. Every other binding below applies unchanged.)

1. `type` is the normative identifier; `code` is its short form. `type`
   is a stable absolute `https` URI **under a domain the provider
   controls** (the URN exception in point 6 is the sole alternative) —
   identity on an uncontrolled domain is the failure the
   `httpstatuses.com` repurposing evidenced. Each
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
> Member set scoped to response-carried documents in version 1.1.0 by
> `P6-D2`, `research/decisions/baseline-04-streaming.decision.md` · project
> policy `[POLICY]` · confidence high. RFC 9457 §3.1 permits either omitting
> `status` or setting it to the status actually sent; what it forbids is a
> `status` that disagrees. This standard chooses omission, because a `status`
> of `200` on a document describing a failure is accurate about the response
> and misleading about the outcome.

**R5.14** RFC 7807 MUST NOT be cited — it is obsoleted by RFC 9457.
**Exception:** historical notes explicitly labeled as such.

> Provenance: `AC-005` (batch, `baseline-02` §7) · the obsoletion is
> protocol fact; the citation ban is project policy `[POLICY]` ·
> confidence high.

**R5.15** A validation failure covering one or more fields SHOULD carry a
field-level `errors[]` extension member on the problem document: an array
whose entries each carry a JSON Pointer to the offending input location
(`pointer`), a stable machine-readable `code`, and a human-readable
`detail`.

> Provenance: Apparatus — ratified at Gate D 2026-08-09 — gap review item B.21 (Zalando/Belgif
> field-error shapes); extends R5.13.

**R5.16** An API MUST publish a catalog of every problem `type`/`code`
pair it can return.

> Provenance: Apparatus — ratified at Gate D 2026-08-09 — gap review item CLI-R7.12; the pair
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

**Streaming scope.** A streamed collection has no top-level object by
construction: its frames are the items. It satisfies this rule by carrying
the continuation state on the terminal frame (R13.6) instead, which
preserves what the envelope exists to protect — a place to add metadata
without a breaking change. R6.2 and R6.4 carry the matching streamed forms
in their own text — an empty streamed collection is zero item frames plus
the terminal frame, and pagination state stays in the body, never a header.
Every other §6 rule binds unchanged; in particular a streamed collection
still owes R6.6's documented stable total order, and owes it more strictly,
since R13.10's resumption depends on it.

> Provenance: `AC-014` (batch, `baseline-02` §7) · evidence-backed
> default · confidence high. Streaming scope added in version 1.1.0 by the
> Phase 6 review walk (2026-08-10),
> `research/decisions/baseline-04-streaming.decision.md` · project policy
> `[POLICY]`.

**R6.2** An empty collection returns `200 OK` with an empty items array —
never `404 Not Found`. A **streamed** empty collection returns `200 OK` with
zero item frames followed by the terminal frame R13.6 requires: the empty
result is signaled by the terminal frame arriving with nothing before it,
which is the streamed form of the same guarantee — a client can tell "no
results" from "something went wrong."

> Provenance: addendum A3 drafting row (`baseline-01` decisions) ·
> evidence-backed default — the registered semantics of 200 and 404
> decide it; no RFC states the collection-specific rule · confidence
> high. Streamed form added in version 1.1.0 by the Phase 6 review walk
> (2026-08-10), `research/decisions/baseline-04-streaming.decision.md` ·
> project policy `[POLICY]`.

### 6.2 Pagination

**R6.3** Pagination SHOULD use opaque, non-constructable cursors — offset
pagination is incorrect under concurrent mutation. **Exception:** small or
stable collections; UI requiring jump-to-page. The exception applies only
where the contract documents the property claimed — a bounded size or an
append-only/immutable mutation pattern. The corresponding client
obligation not to construct or modify cursors is R12.5.

> Provenance: `AC-013` (batch, `baseline-02` §7) · evidence-backed
> default · confidence high.

**R6.4** Pagination state lives only in the body representation — the
envelope (R6.1), or the terminal frame where the collection is streamed.
RFC 8288 `Link` headers MUST NOT be emitted for pagination — dual emission
creates two places a cursor can live, which drift under maintenance. The
prohibition is what this rule is for, and it binds streamed collections
identically: one place, in the body, never a header.

> Provenance: walked decision "Pagination links" (`baseline-02` decisions)
> · project policy · confidence moderate-high. Widened from "body envelope"
> to "body representation" in version 1.1.0 by the Phase 6 review walk
> (2026-08-10), `research/decisions/baseline-04-streaming.decision.md`, so
> a streamed collection has a conforming place to carry it; the
> `Link`-header prohibition is unchanged.

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

**R7.1** Every response MUST carry an explicit `Cache-Control` header —
stating freshness, `no-cache`, or `no-store` per the R7.3 posture —
because silence is not a decision; heuristic caching (RFC 9111 §4.2.2)
means an unlabeled response may still be cached by intermediaries, and
the ratified posture names `Cache-Control` as the vehicle (an `Expires`
header alone does not satisfy this rule).

> Provenance: `HS-016` (batch, `baseline-01` §7) · protocol-grounded
> (RFC 9111 §4.2.2 permits heuristic caching — the hazard); the
> always-emit MUST is a `[POLICY]` tightening per the walked caching
> decision · confidence high.

**R7.2** Responses carrying user-specific or authenticated data MUST be
marked `private` or `no-store` — a shared cache may otherwise serve one
user's data to another. **Exception:** genuinely public responses.

> Provenance: `HS-017` (batch, `baseline-01` §7) · protocol-grounded
> (RFC 9111 §3, §5.2.2.7 — the cross-user leak class binds caches); the
> origin-side marking MUST is a `[POLICY]` tightening · confidence high.

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

> Provenance: Apparatus — ratified at Gate D 2026-08-09 — gap review item CLI-R8.1 (destructive
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
single trust relationship — a key issued to exactly one calling system;
a key shared across distinct callers exceeds this boundary.

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
threat-model flip triggers. An API MUST adopt each axis default unless a
named trigger applies, and MUST record any flip in its conformance note. The full trigger
tables and evidence live in the decision record (`baseline-03g`); the
normative skeleton:

| Axis | Default | Headline flip triggers |
| --- | --- | --- |
| Sender-constrained tokens | Bearer over TLS + short TTL + audience restriction + refresh-token rotation for public clients; validation SHOULD NOT hard-code the `Bearer` scheme | FAPI 2.0 / open banking → DPoP or mTLS; tokens visible to logging intermediaries; hostile-environment public clients → DPoP; existing PKI server-to-server → mTLS; per-operation value → RFC 9470 step-up |
| Token format | Opaque on the public wire; phantom-token pattern where a gateway exists; a client-visible JWT MUST be RFC 9068-conformant and paired with a revocation-propagation plan | Measured introspection bottleneck, AS-outage tolerance, or third-party resource servers → JWT; instant-revocation SLA or PII claims → stay opaque |
| Rate-limit aggressiveness | Multi-dimensional tiered posture, published: per-principal sustained + token-bucket burst (start ≈100 rps/account, 25 rps/endpoint, `[POLICY]` numbers); unauthenticated per-IP an order of magnitude lower; auth endpoints strictly stricter (start ≤5/min per IP+account); failed-auth budget; concurrency separate | Large per-request cost variance → cost/token accounting; metered third-party spend → spend caps; credential stuffing → lockout tier; multi-tenant → fair-share; free-tier abuse → spend/tenure gating |
| Replay window | 300 s past / 60 s future, asymmetric + mandatory dedup cache held at least the past window; NTP required; the window alone is never sufficient. Does not reopen the ratified webhook tolerance convention (R12.8) | Interactive high-value signing → 30–60 s; server-provided nonces remove skew; unmanaged clocks or store-and-forward → up to 15 min, never without dedup; signature omits body → add RFC 9530 binding |
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

> Provenance: Apparatus — ratified at Gate D 2026-08-09 — gap review item CLI-R5.6.

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

> Provenance: Apparatus — ratified at Gate D 2026-08-09 — gap review items CLI-R7.2/CLI-R9.3 (breaking
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

> Provenance: `OP-013` (batch, `baseline-03` §7) · protocol-grounded
> (RFC 9745/RFC 8594 define the headers; emitting them is this
> standard's `[POLICY]` MUST) · confidence high.

**R9.6** Every deprecation MUST carry a `deprecation` link relation to
human-readable migration documentation, and an element MUST NOT be
deprecated without a sunset date.

> Provenance: `OP-014` (batch, `baseline-03` §7) · protocol-grounded
> (RFC 9745; the link relation and RFC 8594 `Sunset` are optional
> mechanisms there); the MUSTs are `[POLICY]` tightenings · confidence
> high.

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

> Provenance: Apparatus — ratified at Gate D 2026-08-09 — gap review items CLI-R7.4/CLI-R10.4
> (polling/cancellation guidance), riding `AC-019` and addendum A5.

**R10.9** A `202 Accepted` MUST identify its operation resource in the
response body — either the operation's `id`, where the operation
resource's URI template is documented in the description document
(R4.1), or an absolute `url` member. The `202` SHOULD additionally
carry a `Location` header whose value is the absolute URI of the
operation resource — never of the eventual result — and where both are
present they MUST denote the same resource. A `202` carrying neither
body identity nor header strands the client and violates R10.1.
(Numbered out of prose order per R1.2.)

> Provenance: research leaf `baseline-02i` (2026-08-10), riding
> `AC-019`; ruled by the owner at the Phase 4 walk · body clause
> protocol-grounded (RFC 9110 §15.3.3 — the representation "ought to …
> point to (or embed) a status monitor"); the `Location` SHOULD is
> `[POLICY]` (RFC 9110 defines no `Location` semantics for `202`) ·
> confidence moderate-high (body), moderate (header).

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

**Streaming scope.** The `429` obligation binds wherever a status code is
still available to be generated, which includes every request for a stream
up to the moment its status is committed. Quota exhausted *during* a
committed stream is reported in-band under R13.7, with `code` naming the
exhaustion and a `retry_after` extension member (§1.10) carrying the pacing
hint that `Retry-After` would otherwise have carried. The full problem
document, with `status` and a real `Retry-After` header, remains available
from the operation resource where one exists (R13.9).

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
> (semi-annual; next 2027-02-09). R11.2's streaming scope added in version
> 1.1.0 by the Phase 6 review walk (2026-08-10),
> `research/decisions/baseline-04-streaming.decision.md` · project policy
> `[POLICY]`.

**R11.5** `429` signals quota exhaustion; `503 Service Unavailable`
signals capacity overload; both MUST carry `Retry-After`. Where the
condition arises after a streaming response's status is committed, neither
status nor header is available: the condition is reported under R13.7 and
carries `retry_after` in place of the header, per R11.2's streaming scope.

> Provenance: `OP-011` (batch, `baseline-03` §7) · protocol-grounded
> (RFC 9110 §15.6.4 and RFC 6585 §4 both make `Retry-After` optional);
> the MUST is a `[POLICY]` tightening, as the `OP-010` record states ·
> confidence high. Streaming scope added in version 1.1.0 by the Phase 6
> review walk (2026-08-10),
> `research/decisions/baseline-04-streaming.decision.md` · project policy
> `[POLICY]`.

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

> Provenance: Apparatus — ratified at Gate D 2026-08-09 — client half of `OP-011`, named as new
> §12 content by the gap review (item B.9).

**R12.3** Clients MUST set explicit request timeouts, and MUST NOT
disable TLS certificate verification in any environment that reaches a
real API host.

> Provenance: Apparatus — ratified at Gate D 2026-08-09 — gap review item B.9 (client timeouts,
> TLS verification).

### 12.2 Tolerant reading

**R12.4** Clients MUST tolerate unknown response fields and unknown enum
values (R4.9): ignore what you do not recognize, never fail on it. This
is what makes additive evolution (R9.4) non-breaking in practice.

> Provenance: `AC-012` (batch, `baseline-02` §7; client half) +
> Apparatus — ratified at Gate D 2026-08-09 for the unknown-field generalization · confidence
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
infrastructure outside the application (R5.12's first carve-out) may answer
first, with any shape. Robust error handling branches on the status code
first and parses the body opportunistically.

> Provenance: `AC-003` ratification consequences (client-robustness note
> and media-type interop hazard, `baseline-02` decisions) ·
> evidence-backed default · confidence moderate.

### 12.4 Webhook consumers

**R12.8** A webhook consumer MUST verify the signature over the raw
request body before parsing; MUST enforce a bounded, non-zero timestamp
tolerance (300 seconds is the ratified webhook convention — a fixed
default, deliberately not coupled to the §8.3 replay axis or its flip
triggers); MUST deduplicate on the signed delivery ID for at least the
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

**R12.10** A client consuming a streaming response MUST treat a connection
that closes without the documented terminal frame (R13.6) as **truncated**,
and MUST NOT treat partial content as a complete result — noting that a
stream-ending `error` frame *is* a terminal frame, so a reported failure is
a complete result, not a truncation. It MUST tolerate a trailing sentinel
frame after the terminal frame (R13.6). It MUST ignore
frame types it does not recognize (the streaming case of R12.4). On an
`error` frame it MUST treat the operation as failed and MUST branch on the
frame's `code` and `type` rather than on the response status, which is
`200`; where an operation resource exists it SHOULD fetch the full problem
document from it (R13.9). It MUST NOT depend on keep-alive frames arriving
on any schedule. It MUST NOT recover from truncation by replaying a
non-idempotent request without an idempotency key (R12.1, R3.9).

> Provenance: `ST-012` · `P6-D0` batch (Phase 6 walk, 2026-08-10),
> `research/decisions/baseline-04-streaming.decision.md` · evidence-backed
> default, with the replay clause `[POLICY]` · confidence high. Scoped by
> the `streaming` switch, client side.

---

## 13. Streaming responses

This section governs responses delivered incrementally over a single HTTP
request: Server-Sent Events, long-polling, and streaming HTTP bodies. It is
deliberately short. The reasoning behind these rules, the wire examples, the
vendor evidence, and the deployment and client guidance live in the
informative companion,
[`streaming-profile.md`](streaming-profile.md); nothing there is normative,
and where it restates a rule the rule governs.

**WebSockets are outside this section and outside this standard** (§1.2).

**Switch scope.** Rules R13.1, R13.2, and R13.4–R13.11 are scoped by the
`streaming` applicability switch (§1.8), as is R12.10. **R13.3 is not** — it
binds **every endpoint that does not implement streaming, whatever the API's
`streaming` switch says**, because scoping it to the switch would delete it
for exactly the endpoints it protects. An API that streams on some endpoints
still owes the guard on the rest. R13.9 binds only where `streaming` **and**
`async-operations` are both on.

### 13.1 Shape and negotiation

**R13.1** A streaming response MUST be a `200 OK` whose `Content-Type` names
a self-delimiting stream media type. `202 Accepted` MUST NOT be used for a
streaming response — R10.1 binds `202` to an operation resource, and a
streaming `202` forks that contract. A body of concatenated JSON documents
MUST NOT be labeled `application/json`: a conforming JSON parser fed the
whole body fails, and the label is therefore a false statement about the
body.

> Provenance: `ST-001` · `P6-D0` batch (Phase 6 walk, 2026-08-10) ·
> evidence-backed default on the `200` and the declaration, `[POLICY]` on
> the two prohibitions · confidence high.

**R13.2** Where one endpoint serves both a streamed and a non-streamed
representation, the choice MUST be made by content negotiation on `Accept`,
and the response MUST carry `Vary: Accept` (R4.11). A query parameter MUST
NOT select between them. An API MAY instead expose a distinct resource that
streams unconditionally; that resource performs no selection, so R4.10 does
not reach it.

This is R4.10 applied to streaming: streaming changes the response media
type, so choosing it is media-type selection, and R4.10 already forbids the
query-parameter form and already supplies the `406 Not Acceptable` guard.

> Provenance: `ST-002` · `P6-D1` (Phase 6 walk, 2026-08-10) · project
> policy `[POLICY]`, derived from ratified R4.10/R4.11 · confidence
> moderate — the composition is firm; departing from unanimous vendor
> practice is a policy choice, recorded as one in the decision record.

**R13.3** `stream` is a reserved request-modifier name (§1.10) meaning
"deliver this response incrementally." An endpoint that does not implement
streaming and receives `stream` — as a query parameter or a request-body
member, with any value — MUST reject the request with `400`, and MUST NOT
silently answer with a non-streamed response.

**This rule is not scoped by the `streaming` switch,** and it binds per
endpoint rather than per API: every endpoint that does not implement
streaming owes the guard, including endpoints of an API whose `streaming`
switch is on. That is exactly the set a switch-scoped rule would exempt. It
is the streaming counterpart of R1.9's `dry_run` guard and exists for the
identical hazard: silently ignoring an unimplemented request modifier leaves
a client believing it received what it asked for.

On an endpoint that **does** implement streaming, `Accept` governs the
choice (R13.2) and `stream` selects nothing. A `stream` modifier that
disagrees with the negotiated representation — asking for a stream where
`Accept` selected the non-streamed form, or the reverse — MUST be rejected
with `400` rather than silently resolved in either direction.

> Provenance: `ST-003` · `P6-D1` (Phase 6 walk, 2026-08-10) · project
> policy `[POLICY]`, modeled on R1.9 · confidence high.

**R13.4** Server-Sent Events framing, served as `text/event-stream`, SHOULD
be the framing chosen for incrementally generated content — that is, content
a client can act on before it is complete. (This is a different sense of
"default" from R4.10's, which fixes the media type served when a request
carries no `Accept`; SSE is never served without being asked for.) A large
finite record set, where a partial answer has no value, is the
newline-delimited case instead. An API adopting SSE MUST
document that the media type has no IANA registration, and MUST NOT describe
it as an IANA-registered media type. (It **is** standardized — the WHATWG
HTML Living Standard normatively defines the format and names the media
type. What it lacks is a registry entry, which is a different claim; do not
conflate them in either direction.)

The registration gap is disclosed rather than worked around: `text/event-stream`
is absent from the IANA `text/*` subregistry and its per-type registry URL
returns `404` (probed 2026-08-10). The specification that defines it is the
WHATWG HTML Living Standard, not an RFC, and the W3C Recommendation that
IETF documents cite for it was **Retired** by W3C on 2021-01-28. The one
registered
alternative, `application/json-seq` (RFC 7464), has no HTTP adoption among
surveyed APIs and no browser parser — which is why this rule blesses the
unregistered type while §1.10 records what it is.

> Provenance: `ST-004` · `P6-D4a` (Phase 6 walk, 2026-08-10) ·
> evidence-backed default `[COMPARATIVE]` — explicitly **not** a protocol
> requirement, since a living-standard section plus a missing registry row
> cannot make one · confidence high on the practice, moderate on the
> default; standing weakness recorded in the decision record.

### 13.2 Frames, termination, and errors

**R13.5** Every frame in a stream MUST carry a documented type identifying
what the frame is, and the API MUST document its full frame-type vocabulary
and state that the vocabulary may grow. Without a per-frame type a client
must parse every payload to learn what it received, and can neither route,
ignore, nor count frames. The growth statement is what makes R12.10's
unknown-type tolerance dischargeable.

**Frame payloads are representations.** §4's representation rules bind them
unchanged — R4.4's snake_case grammar, R4.8's null-versus-absent discipline
(fixed by R3.8), R4.6's timestamps, R4.7's money encoding. A frame is not
exempt from the
body conventions merely because the enclosing response is not
`application/json`.

**Transport filler is not a frame.** Bytes the framing parser discards
without dispatching — an SSE comment line used as a keep-alive — carry no
type and are not subject to the typing obligation. Where an API sends a
keep-alive as a *typed* frame instead, it is a frame and the typing
obligation binds it.

**Keep-alive disclosure.** An API MUST document whether it emits keep-alives
and in what form — typed frame, comment line, or none. No interval is
required, and a client MUST NOT depend on one (R12.10); but a client cannot
discharge that obligation against a mechanism it was never told about, and
an undocumented keep-alive frame is indistinguishable from an unrecognized
one.

> Provenance: `ST-005` · `P6-D0` batch (Phase 6 walk, 2026-08-10) ·
> evidence-backed default · confidence high.

**R13.6** A stream MUST end with a documented terminal frame carrying the
final outcome, so that a client can distinguish normal completion from a
truncated connection. **A stream-ending `error` frame (R13.7) is a terminal
frame** and satisfies this rule on its own; no further frame is required
after it, and a client MUST NOT treat a stream ended that way as truncated.
A trailing sentinel frame MAY follow the terminal frame, and clients MUST
tolerate one. A stream that is unbounded by design — a watch, an event
tail — has no normal end; such an API MUST document that the stream is
unbounded, and the terminal-frame obligation then applies to the
server-initiated close case only.

> Provenance: `ST-006` · `P6-D0` batch (Phase 6 walk, 2026-08-10) ·
> evidence-backed default on the requirement, `[POLICY]` on preferring a
> typed terminal frame over a bare sentinel · confidence high.

**R13.7** An error raised after the response status is committed MUST be
delivered in-band, in a frame of the reserved `error` type (§1.10), whose
payload is a problem details object per RFC 9457 §3 carrying R5.13's
required members **other than `status`** — `type`, `title`, and `code` —
bound by R5.13's `type`/`code` template and listed in the R5.16 catalog,
with `detail` and extension members permitted exactly as on any other
problem document. The object MUST omit the `status` member. The frame MUST
NOT be described as an `application/problem+json` response.

**Scope: this rule governs errors that end the stream.** A per-item failure
inside a stream that continues — one bad row in an export, one rejected
item in a batch — is ordinary payload, not an `error` frame, and is carried
in whatever frame type the API documents for per-item results. An `error`
frame ends the stream and is its terminal frame (R13.6). An API MUST NOT use
the reserved `error` type for a failure the stream survives.

Two protocol facts fix this shape. RFC 9457 §3.1: the `status` member "if
present, is only advisory," and "Generators MUST use the same status code in
the actual HTTP response" — so a `status` naming the error would contradict
the `200` actually sent, while omission is permitted by the same sentence.
RFC 9110 §6.5.1 excludes trailer fields as the alternative channel: "in most
cases, the trailers are simply discarded," and a server "SHOULD NOT generate
trailer fields that it believes are necessary for the user agent to
receive." What remains is in-band delivery, which this rule specifies.

This is the second named carve-out from R5.12.

> Provenance: `ST-007` · `P6-D2` (Phase 6 walk, 2026-08-10) · project
> policy `[POLICY]` on the carve-out and the omission; the two constraints
> it obeys are protocol requirements (RFC 9457 §3.1, RFC 9110 §6.5.1) ·
> confidence moderate-high — the constraints are certain, the resolution
> among them is this standard's choice.

**R13.8** An error detected **before** the response status is committed MUST
follow R5.12 unchanged and be servable as an `application/problem+json`
error response, whatever the request's streaming `Accept` or `stream`
modifier asked for. A streaming request modifier governs the success
representation only; nothing in this section licenses answering a request
that never began succeeding with `200` plus an error frame.

> Provenance: `ST-008` · `P6-D0` batch (Phase 6 walk, 2026-08-10) ·
> project policy `[POLICY]` — a scope clarification of R5.12 · confidence
> high.

### 13.3 Relationship to operation resources, and resumption

**R13.9** Where one capability is exposed both as a stream and as an
operation resource (R10.9), the two MUST be one capability with one
identity: the stream MUST carry the operation's identity in the reserved
member matching the form R10.9's `202` body uses — `operation_id` for the
`id` form, `operation_url` for the `url` form (§1.10); both channels MUST
report the same terminal state, **the terminal frame carrying that value in
the reserved `operation_state` member (§1.10), drawn from the vocabulary
R10.1 requires the operation resource to document**, so that the two are
comparable — this applies to an `error` terminal frame exactly as to a
success one, which is why the member is not named `status` (R13.7 forbids
`status` on the problem object) — with the operation resource
authoritative; and
the full problem document for a failed operation — carrying `status`, as
`application/problem+json` — MUST be retrievable from the operation
resource.

**When two endpoints are "one capability."** The test is documentary and
independent of compliance: a stream and an operation resource are one
capability when the API documents them as producing the same result from the
same inputs, such that a client would choose between them rather than use
both. An API MUST NOT escape this rule by omitting the shared identifier —
omission is the violation this rule names, not an exit from its scope.

This composes with R10.9 rather than forking it, and it is the out-of-band
half of R13.7's carve-out: the operation resource is the one place where a
status code is still available to be generated.

> Provenance: `ST-009` · `P6-D0` batch (Phase 6 walk, 2026-08-10) ·
> evidence-backed default on the unified shape, `[POLICY]` on making the
> operation resource authoritative · confidence moderate-high — one shipped
> exemplar against one contrary guideline, adjudicated in the decision
> record. Binds only where `streaming` and `async-operations` are both on.

**R13.10** Where a stream is a view over a retained artifact, the API SHOULD
offer resumption. An API that offers resumption MUST carry a **strictly
increasing** `stream_position` (§1.10) on every frame — no two frames of one
stream share a position — MUST document the retention window, and MUST
reject a resumption request whose position lies outside that window with a
defined error rather than silently restarting the stream.

**"A view over a retained artifact"** means the data the stream reads
survives independently of any open connection: a stored export, a change log,
a persisted operation result. Content generated fresh per request, with no
independently addressable backing store, is not — which is why an API whose
stream is a live generation owes nothing here, while one streaming a stored
artifact does. Where an API is unsure, R10.9's operation resource is the
signal: if the work has an addressable operation resource that outlives the
connection, the stream is a view over it.

`stream_position` is deliberately not `cursor`: R12.5 requires a client to
treat a cursor as opaque and never construct or modify one, while this rule
requires a position with visible ordering. Two names keep both obligations
literally true. A client echoes a `stream_position` and never computes one.

> Provenance: `ST-010` · `P6-D5` (Phase 6 walk, 2026-08-10) ·
> evidence-backed default · confidence moderate — two implementations
> across the surveyed field, resuming two different kinds of thing, which
> is why this is SHOULD and conditional rather than MUST.

### 13.4 Known unresolved interactions

Five interactions between streaming and the rest of this standard are
**recognized and not yet ruled**. They are recorded here rather than left to
be discovered, because a reader who hits one should know the standard is
silent by decision rather than by oversight. None of them makes a rule in
this section unsatisfiable; each is scheduled for the phase named in
`PLAN.md` Phase 7, to be ruled with the same evidence discipline as §13
itself. Until then an API resolves them as it sees fit and records its
choice in its conformance note (R1.7).

| Interaction | What is unresolved |
| --- | --- |
| **Frame-vocabulary versioning** (§9.3) | R9.4's breaking-change taxonomy does not classify frame-type names. Renaming a terminal frame looks compatible, yet every deployed client would ignore the unrecognized frame under R12.10, see no terminal frame, and report truncation on every success. Until ruled, treat documented frame-type names and which types are terminal as part of the frozen surface. |
| **Authorization over a stream's lifetime** (§8) | R8.6 authorizes a request; a stream is one request that may outlive the credential that opened it (R8.5 wants expiring credentials). No rule says whether a server must re-evaluate authorization mid-stream or bound a stream's lifetime by its credential's. |
| **Caching posture for a stream** (§7) | R7.1 requires an explicit `Cache-Control` on every response, and R7.3's tier-1 default revalidates via strong `ETag` — machinery a stream cannot supply, since the body does not exist when headers are sent. The worked example uses `no-store`; the general rule is unruled. |
| **Idempotency-key replay of a streaming request** (§3) | R3.9 replays "the stored response" for a genuine retry. For a stream that is undefined: replay from the first frame, serve a non-streamed representation, or resume — which is R13.10, a different mechanism with different preconditions. |
| **Resource ceilings for streams** (§11) | R11.1 requires published maxima for page size, expansion depth, and bulk item count. A held-open stream is the largest unbounded commitment the API makes and is in none of those dimensions; no maximum duration or per-principal concurrency ceiling is required. |

### 13.5 Long-polling

**R13.11** A long-polling endpoint MUST document its maximum hold duration.
An expired hold MUST return `200` with a well-formed empty-result
representation carrying the `cursor` for the next poll. `204 No Content`
MUST NOT be used for an expired hold.

A long poll returns a representation, not a stream, so its continuation
token is the ordinary opaque `cursor` of §1.10 and R6.3, with R12.5's
opacity obligation applying unchanged. `stream_position` belongs to
resumable streams (R13.10) and is not used here.

The `204` prohibition has two independent grounds: R5.7 already binds `204`
to a successful DELETE, and the WHATWG HTML Living Standard gives `204` a
conflicting reserved meaning on this exact surface — a client "can be told
to stop reconnecting using the HTTP 204 No Content response code" — so an
API offering both mechanisms on one path would have the two meanings
collide.

> Provenance: `ST-011` · `P6-D0` batch (Phase 6 walk, 2026-08-10) ·
> evidence-backed default on the `200`-with-empty-result shape, `[POLICY]`
> on the `204` prohibition · confidence high.

---

## Part II — Decision Log

This log is the two-way mapping between the decision layer and the drafted
rules. The decision records in [`research/decisions/`](research/decisions/)
remain authoritative for rationale, evidence, declined options, and
confidence; each Part I rule's provenance line carries its decision-layer
key, and this log maps each key back to its rules. Record files are
abbreviated:
**B1** = `research/decisions/baseline-01-http-semantics.decision.md` ·
**B2** = `research/decisions/baseline-02-api-contracts.decision.md` ·
**B3** = `research/decisions/baseline-03-operational-practice.decision.md` ·
**B4** = `research/decisions/baseline-04-streaming.decision.md` (Phase 6).

**Amendment rule (Apparatus — ratified at Gate D 2026-08-09).** This document is versioned with
semantic versioning once approved: editorial changes bump patch; added
rules, appendices, or relaxations bump minor; strengthened, removed, or
re-meant rules bump major. A change to any rule is atomic across five
surfaces — the rule text, its decision record (as a dated annotation,
never a silent edit), its row here, its Appendix A checklist row, and the
Appendix E worked example where it appears.

### II.1 Provenance map — decision layer → rules

| Key | Maps to | Record |
| --- | --- | --- |
| `HS-001` | R3.1 | B1 |
| `HS-002` | R3.2 | B1 |
| `HS-003` | R5.2 | B1 |
| `HS-004` | R2.7 | B1 |
| `HS-005` | R2.8 | B1 |
| `HS-006` | R3.3 | B1 |
| `HS-007` | R3.4 | B1 |
| `HS-008` | R3.5, R3.7 | B1 |
| `HS-009` | R3.6 | B1 |
| `HS-010` | R5.1 | B1 |
| `HS-011` | R5.3 | B1 |
| `HS-012` | R5.4 | B1 |
| `HS-013` | R5.5 | B1 |
| `HS-014` | R3.10 | B1 |
| `HS-015` | R3.11, R7.4 | B1 |
| `HS-016` | R7.1 | B1 |
| `HS-017` | R7.2 | B1 |
| `HS-018` | R4.11 | B1 |
| `HS-019` | R4.12 | B1 |
| `HS-020` | R4.13 | B1 |
| `AC-001` (completed) | R4.1 | B2 |
| `AC-002` | R4.2 | B2 |
| `AC-003` (amended) | R5.12, R12.6, R12.7 | B2 |
| `AC-004` (amended) | R5.13 | B2 |
| `AC-005` | R5.14 | B2 |
| `AC-006` | R4.3 | B2 |
| `AC-007` (completed) | R4.4 | B2 |
| `AC-008` | R4.5 | B2 |
| `AC-009` | R4.6 | B2 |
| `AC-010` | R4.7 | B2 |
| `AC-011` | R4.8, R3.8 | B2 |
| `AC-012` | R4.9, R12.4 | B2 |
| `AC-013` | R6.3, R12.5 | B2 |
| `AC-014` | R6.1 | B2 |
| `AC-015` | R6.9 | B2 |
| `AC-016` (completed) | R3.9 | B2 |
| `AC-017` (completed) | R3.9 | B2 |
| `AC-018` | R10.4 | B2 |
| `AC-019` | R10.1 | B2 |
| `AC-020` | R4.15, R10.3 | B2 |
| `AC-021` | R4.14 | B2 |
| `OP-001` | R8.1 | B3 |
| `OP-002` | R8.2 | B3 |
| `OP-003` | R8.4 | B3 |
| `OP-004` | R8.4 | B3 |
| `OP-005` | R8.6 | B3 |
| `OP-006` | R8.7 | B3 |
| `OP-007` | R8.8 | B3 |
| `OP-008` | R8.9 | B3 |
| `OP-009` | R11.1 | B3 |
| `OP-010` (re-framed) | R11.2, R11.3, R11.4 | B3 |
| `OP-011` | R11.5, R12.2 | B3 |
| `OP-012` | R11.6, R12.1 | B3 |
| `OP-013` | R9.5 | B3 |
| `OP-014` | R9.6 | B3 |
| `OP-015` (completed) | R9.1, R9.2 | B3 |
| `OP-016` (amended) | R10.7, R10.8, R12.8 | B3 |
| `OP-017` | R12.8 | B3 |
| `OP-018` | R11.7 | B3 |
| `OP-019` | R11.8 | B3 |
| `OP-020` | R5.17 | B3 |
| `OP-021` | R10.5 | B3 |
| `OP-022` | R10.6, R12.9 | B3 |
| `OP-023` | R8.11 | B3 |
| `OP-024` | R10.8 | B3 |
| `OP-025` | R8.5 | B3 |
| Resource orientation (walked) | R2.1 | B1 |
| Pluralization (walked) | R2.2 | B1 |
| Path depth (walked) | R2.5 | B1 |
| Trailing slash (walked) | R2.6 | B1 |
| Custom-action syntax (walked) | R2.11 | B1 |
| Path-segment casing (walked) | R2.4 | B1 |
| DELETE response (walked) | R5.7 | B1 |
| BCP 190 scope reading (walked) | position, §1.3; R2.8 | B1 |
| HATEOAS posture (walked) | position, §1.2 | B1 |
| Caching posture (walked) | R7.3 | B1 |
| Money representation (walked) | R4.7 | B2 |
| Pagination links (walked) | R6.4 | B2 |
| Field selection (walked) | R6.10 | B2 |
| Filter grammar (walked) | R6.8 | B2 |
| Deprecation window (walked) | R9.7 | B3 |
| Webhook retry/dead-letter (walked) | R10.6 | B3 |
| Support tiers (walked) | R9.3 | B3 |
| Auth per client class (walked) | R8.3 | B3 |
| Five-axes deployment profile (walked) | R8.10, R8.6, R12.8 | B3 |
| A1 · PATCH format | R3.7, R3.8 | B2 |
| A2 · Sorting cluster | R6.5, R6.6, R6.7, R11.7 | B2 |
| A3 · Status-code rows | R5.6, R5.8, R5.9, R5.10, R5.11, R6.2, R3.11 | B1 |
| A4 · Dry-run | R1.9, R3.12 | B2 |
| A5 · Action verbs | R2.11, R2.12, R2.13; §1.10 verb registry | B2 |
| Phase 4 owner walk (2026-08-10) | R4.16; R10.9; §1.8 switch pruning; §1.10 `cancel` scope | `docs/reviews/2026-08-09-phase-4-internal-review-findings.md` |
| `baseline-02i` · Operation discovery on 202 (Phase 4) | R10.9 | B2 |
| Gate E ruling (2026-08-10) | R4.17 | `docs/reviews/2026-08-09-phase-4-internal-review-findings.md` |
| `ST-001` | R13.1 | B4 |
| `ST-002` | R13.2 | B4 |
| `ST-003` | R13.3; §1.10 `stream` | B4 |
| `ST-004` | R13.4; §1.10 `text/event-stream` | B4 |
| `ST-005` | R13.5 | B4 |
| `ST-006` | R13.6 | B4 |
| `ST-007` | R13.7; §1.10 `error` frame type; R5.12 second carve-out; R5.13 scoping | B4 |
| `ST-008` | R13.8 | B4 |
| `ST-009` | R13.9 | B4 |
| `ST-010` | R13.10; §1.10 `stream_position` | B4 |
| `ST-011` | R13.11 | B4 |
| `ST-012` | R12.10 | B4 |
| `ST-013`–`ST-020` | none — informative, `streaming-profile.md` | B4 |
| `P6-D0` · Deliverable shape (Phase 6 walk) | §13 as a compact normative section; §1.5 namespace extended to thirteen | B4 |
| `P6-D1` · Streaming negotiation (walked) | R13.2, R13.3 | B4 |
| `P6-D2` · Post-commit stream errors (walked) | R13.7; R5.12, R5.13 (amended) | B4 |
| `P6-D3` · `R5.1` streaming scope (walked) | R5.1 (amended) | B4 |
| `P6-D4a`/`P6-D4b` · §1.10 additions (walked) | §1.10 media-type row; §1.10 stream frame types | B4 |
| `P6-D5` · Resumption position name (walked) | R13.10; §1.10 `stream_position` | B4 |
| Phase 6 review walk · Tier A collisions (2026-08-10) | R11.2, R11.5, R2.11, R6.1 (each scoped for streaming) | B4 |
| Phase 6 review walk · Tier C completions (2026-08-10) | R13.9 (identity member + terminal-state vocabulary); R13.5 (keep-alive disclosure); §1.10 reserved stream members | B4 |
| Codex second lens (2026-08-10) | R6.2, R6.4 (completing the R6.1 streaming scope); §1.10 `operation_state`; R5.13 and R13.4 provenance corrections | B4 |
| Phase 6 review walk · Tier B deferral (2026-08-10) | §13.4 known-unresolved register; `PLAN.md` Phase 7 | B4 |

### II.2 Apparatus register — provisions ratified at Gate D

Every provision marked **Apparatus** in this document, in one place.
These have no Gate C decision record; they were ratified en bloc at
Gate D (2026-08-09) when the owner approved this draft, and this
register is their ratification record. Provisions ruled at the Phase 4
owner walk (2026-08-10) are marked as such in their rows; they enter
the Gate E approval.

| Item | Where | Origin |
| --- | --- | --- |
| Rule-ID scheme and frozen provenance series | §1.5 (R1.2, R1.3) | Gap review B.13 |
| Conformance tier system | §1.7 (R1.5) | Gap review B.8 |
| Applicability switches + N/A-with-reason | §1.8 (R1.6) | Gap review B.8 |
| No-silent-deviation + conformance-note template | §1.9 (R1.7) | Gap review B.12 |
| Reserved-name inventory as a register | §1.10 (R1.8) | Gap review B.5 |
| Noun naming, same-concept-same-name | §2.2 (R2.3) | Gap review CLI-R2.2/CLI-R3.8 |
| Path = identity, query = modifiers | §2.3 (R2.9) | Gap review CLI-R2.3 |
| PII never in URIs | §2.3 (R2.10) | Gap review CLI-R9.7 |
| Content-negotiation defaults | §4.3 (R4.10) | Gap review CLI-R4.2 |
| Field-level `errors[]` shape | §5.3 (R5.15) | Gap review B.21 |
| Problem-type catalog obligation | §5.3 (R5.16) | Gap review CLI-R7.12 |
| Destructive-operation guards | §7.3 (R7.4) | Gap review CLI-R8.1 |
| Secret/PII redaction in responses | §8.5 (R8.12) | Gap review CLI-R5.6 |
| Breaking-change taxonomy + frozen surface | §9.3 (R9.4) | Gap review CLI-R7.2/CLI-R9.3 |
| Polling and cancellation guidance | §10.1 (R10.2) | Gap review CLI-R7.4/CLI-R10.4 |
| Client obligations: `Retry-After`, timeouts, TLS, unknown fields | §12 (R12.2, R12.3, part of R12.4) | Gap review B.9 |
| Amendment rule (SemVer + atomic five-surface updates) | Part II preamble | Gap review B.11 |
| Exception process | Appendix B | Gap review B.12 |
| Path-placeholder naming rule (raised during drafting as an open candidate; ruled snake_case at the Phase 4 owner walk 2026-08-10) | §4.2 (R4.16) | Appendix E drafting → Phase 4 owner walk |
| Switch vocabulary pruned to the three rule-gating switches (was eight) | §1.8 (R1.6) | Phase 4 owner walk (2026-08-10) |
| `202` operation-discovery rule | §10.1 (R10.9) | Research leaf `baseline-02i` + Phase 4 owner walk (2026-08-10) |
| CORS header exposure | §4.3 (R4.17) | `baseline-02i` surfacing + Gate E ruling (2026-08-10) |
| `streaming` applicability switch, and the rule that a switch never waives a guard | §1.8 (R1.6) | Phase 6 drafting, from the ratified applicability of `ST-003` (B4) |
| Section namespace extended from twelve to thirteen | §1.5 (R1.2, R1.3) | Phase 6 deliverable-shape ruling `P6-D0` (B4) |
| Fifth reserved-name category — stream frame types | §1.10 (R1.8) | Phase 6 ruling `P6-D4b` (B4) |
| Informative companion document as a normative-section relief valve | §13 preamble; `streaming-profile.md` | Phase 6 deliverable-shape ruling `P6-D0` (B4) |

---

## Part III — Appendices

The appendices are informative. The rules in Part I are authoritative;
where an appendix restates or illustrates one, the rule governs, and no
appendix defines a new rule identifier.

## Appendix A — Conformance checklist

One row per rule. Rows marked *(standard-internal)* bind this document's
own maintenance rather than a conforming API.

| Rule | Check |
| --- | --- |
| R1.1 | BCP 14 keywords normative only in all capitals *(standard-internal)* |
| R1.2 | Rule IDs stable — never renumbered or reused; tombstones on removal *(standard-internal)* |
| R1.3 | No new `HS-`/`AC-`/`OP-` identifiers minted *(standard-internal)* |
| R1.4 | Every rule carries one classification; policy labeled `[POLICY]` *(standard-internal)* |
| R1.5 | Conformance note declares exactly one tier |
| R1.6 | Every switch declared; every off switch carries a reason |
| R1.7 | Conformance note exists; every deviation recorded in it |
| R1.8 | No reserved name repurposed; reserved name used when the capability is offered |
| R1.9 | `dry_run` on a non-implementing endpoint rejected with 400 |
| R2.1 | Surface is resource-oriented; no RPC-style overlay |
| R2.2 | Collections plural; singletons documented as singletons |
| R2.3 | Resource names unabbreviated; one noun per concept; domain vocabulary reviewed |
| R2.4 | Path segments kebab-case |
| R2.5 | Nesting justified by containment; three resources per path at most |
| R2.6 | No canonical trailing slash; slash requests redirected 308 |
| R2.7 | URIs stable across mutable-attribute changes |
| R2.8 | No fixed-prefix constraint on another party's URI space |
| R2.9 | Operation modifiers in query parameters, never path segments |
| R2.10 | No PII in any URI |
| R2.11 | Actions use `POST /{collection}/{id}/{action}`; verbs registered; response shape per the A5 map (200 sync, 202 long-running and non-streaming, 200 + stream media type where it streams, 201 duplicate) |
| R2.12 | Status field or sub-resource considered before any new verb |
| R2.13 | No collection-level custom actions |
| R3.1 | Semantics cited from RFC 9110/9111, never RFC 723x |
| R3.2 | No redefined or overlaid method, status, or header semantics |
| R3.3 | Methods used per registered safety and idempotence |
| R3.4 | No method tunneling |
| R3.5 | PUT for full replacement; PATCH for partial modification |
| R3.6 | QUERY, where used, only for safe idempotent body-carrying reads |
| R3.7 | PATCH accepts `merge-patch+json` (plus `json-patch+json` where the MAY is exercised); unsupported types 415; `Accept-Patch` advertised |
| R3.8 | Null and absent equivalent everywhere; Merge Patch deletion the sole exception |
| R3.9 | `Idempotency-Key` accepted on non-idempotent mutations; payload fingerprinted; retention window at least 24 h and stated |
| R3.10 | Strong `ETag` on conditionally updatable resources |
| R3.11 | `If-Match` expected where concurrent modification is exposed; 412 and 428 used correctly |
| R3.12 | Dry-run responses carry the marker, the real outcome shape, and validation depth; no key consumed |
| R4.1 | OpenAPI 3.1 (or verified 3.2) published; JSON Schema 2020-12 dialect |
| R4.2 | Contract changes gated on an automated compatibility check |
| R4.3 | Top-level JSON object on every response |
| R4.4 | snake_case bodies and query parameters; patterns pass |
| R4.5 | Identifiers are strings |
| R4.6 | Timestamps RFC 3339 with explicit offset |
| R4.7 | Money as minor-unit integer with required ISO 4217 `currency` |
| R4.8 | Null-versus-omission rule stated in the contract |
| R4.9 | Open enums documented; additions non-breaking |
| R4.10 | Default media type `application/json` UTF-8; no `format` parameter; 406 for unsatisfiable `Accept` |
| R4.11 | `Vary` lists every selecting header |
| R4.12 | New header fields use RFC 9651 structured types |
| R4.13 | Registered link relations used where one fits |
| R4.14 | URI Templates describe URI families |
| R4.15 | Preferences advisory only; never relied on for safety-relevant behavior |
| R4.16 | Path placeholders snake_case (body-property grammar) |
| R4.17 | Cross-origin browser clients: standard-bound headers listed in `Access-Control-Expose-Headers` |
| R5.1 | Status matches the outcome as known when it is generated; no 2xx failures; streaming scope stated (a post-commit stream failure reports under R13.7) |
| R5.2 | Registered status codes only |
| R5.3 | 422, 409, and 400 used per their distinctions |
| R5.4 | 410 only with recorded permanence |
| R5.5 | 307/308 where method and body must survive redirects |
| R5.6 | Single-resource creates return 201 with `Location` |
| R5.7 | DELETE returns 204; soft delete returns 200 with tombstone |
| R5.8 | Partial bulk returns 200 with the per-item envelope; never 207 |
| R5.9 | 401 for unauthenticated, 403 for unauthorized, never conflated |
| R5.10 | Cross-tenant existence masked with 404 |
| R5.11 | 405 carries `Allow`; 415 for unsupported media; 428 where a precondition is demanded |
| R5.12 | Every application error servable as `problem+json`; both carve-outs documented — infrastructure, and post-commit stream errors (R13.7) |
| R5.13 | Response-carried problem documents carry `type`/`title`/`status`/`code`; in-band stream frames carry the same members without `status`; `type` under a provider-controlled domain; template binding; pairs immutable; no `about:blank` |
| R5.14 | RFC 7807 never cited |
| R5.15 | Validation failures carry `errors[]` with JSON Pointers |
| R5.16 | Problem `type`/`code` catalog published |
| R5.17 | No internal implementation detail in any response body |
| R6.1 | Collection responses use the items-plus-continuation envelope; a streamed collection carries continuation state on its terminal frame instead |
| R6.2 | Empty collections return 200 with an empty array; streamed form is zero item frames followed by the terminal frame |
| R6.3 | Cursor pagination (recommendation-strength, with documented bounded/append-only and jump-to-page exceptions); cursors opaque and non-constructable |
| R6.4 | No `Link` headers for pagination; state in the body representation — envelope, or terminal frame when streamed |
| R6.5 | `cursor` and `limit` names; default and maximum documented |
| R6.6 | Total stable default order documented; `id` tiebreak |
| R6.7 | Sort syntax fixed when offered; sortable set enumerated |
| R6.8 | Filters are per-field equality plus bracket ranges, AND-only; DSL only on a search endpoint |
| R6.9 | No storage syntax in filters |
| R6.10 | `fields` comma-list when field selection is offered |
| R7.1 | Explicit `Cache-Control` on every response |
| R7.2 | Authenticated data `private` or `no-store` |
| R7.3 | Three-tier posture applied; no blanket `no-store` |
| R7.4 | Destructive operations demand `If-Match`; no unfiltered collection DELETE |
| R8.1 | TLS 1.2+ only, 1.3 preferred |
| R8.2 | No tokens in query strings |
| R8.3 | OAuth where authority is delegated; keys only server-to-server single-trust |
| R8.4 | No password grant; PKCE; exact redirect matching; no implicit grant |
| R8.5 | Credentials scoped and expiring |
| R8.6 | Object-level authorization on every request; centralized decision, in-handler enforcement |
| R8.7 | Unguessability never an access control |
| R8.8 | Property-level authorization per caller |
| R8.9 | Writable-field allow-lists |
| R8.10 | Five-axes defaults adopted, or flips recorded in the conformance note |
| R8.11 | Caller-supplied URLs validated against internal ranges |
| R8.12 | No secrets or PII in problem `detail` or echoed input |
| R9.1 | `/v1` path major only; no minor versions in URIs |
| R9.2 | Evolution additive; majors rare |
| R9.3 | Pre-GA tiers in the path; graduation an explicit migration |
| R9.4 | Frozen surface respected; every change classified per the taxonomy |
| R9.5 | `Deprecation` and `Sunset` headers, correct formats, sunset not earlier |
| R9.6 | Deprecation link relation to migration docs; sunset date always present |
| R9.7 | Deprecated GA majors supported at least 12 months |
| R10.1 | 202 returns an operation resource with terminal states, expiry, failure shape |
| R10.2 | `Retry-After` polling hints; `cancel` action where abandonable |
| R10.3 | `respond-async` honored at server discretion only |
| R10.4 | Bulk endpoints declare atomic or partial; per-item outcomes |
| R10.5 | Delivery documented at-least-once, unordered; monotonic version per event |
| R10.6 | Ack timeout and retry schedule published; retries at least 72 h; dead-letter at least 30 d with redelivery |
| R10.7 | Webhooks signed per topology; SHA-1 banned |
| R10.8 | Secrets at least 256 bits; overlapping rotation; HTTPS-only; verification tooling shipped |
| R10.9 | `202` body identifies the operation (`id` + documented template, or `url`); `Location` (the operation's absolute URI, never the result) recommended; header and body agree |
| R11.1 | Page size, expansion depth, and bulk count maxima published and enforced |
| R11.2 | 429 with `Retry-After` on exhaustion; exhaustion during a committed stream reported under R13.7 with a `retry_after` member |
| R11.3 | Draft-11 fields, when emitted, pinned and never called standard |
| R11.4 | Proprietary quota headers documented, including epoch-versus-delta |
| R11.5 | 429 for quota, 503 for overload; `Retry-After` on both, or `retry_after` in-band once a stream's status is committed |
| R11.6 | Retryable failure classes documented |
| R11.7 | `request-id` on every response |
| R11.8 | `traceparent` propagated; `tracestate` caps published |
| R12.1 | Clients retry only documented-retryable failures, with backoff and jitter; keys on non-idempotent retries |
| R12.2 | Clients honor `Retry-After` |
| R12.3 | Clients set timeouts; TLS verification never disabled |
| R12.4 | Clients tolerate unknown fields and enum values |
| R12.5 | Clients treat cursors as opaque |
| R12.6 | Clients send `Accept: application/problem+json` when they want problem documents |
| R12.7 | Client error handling never assumes a problem document |
| R12.8 | Consumers verify raw-body-first; window enforced; dedupe; constant-time compare; fail closed |
| R12.9 | Consumers ack before processing; tolerate at-least-once, unordered delivery |
| R12.10 | Stream clients treat a missing terminal frame as truncation (an `error` frame is terminal, not truncation); tolerate a trailing sentinel; ignore unknown frame types; never depend on keep-alive timing; never replay non-idempotent requests without a key |
| R13.1 | Streaming response is `200` with a self-delimiting stream media type; no `202` streaming; concatenated JSON never labeled `application/json` |
| R13.2 | Streamed-versus-non-streamed chosen by `Accept` with `Vary: Accept`; never by a query parameter; or a distinct always-streaming resource |
| R13.3 | `stream` on a non-streaming endpoint rejected with `400`, never silently ignored; on a streaming endpoint, a `stream` modifier disagreeing with the negotiated representation rejected with `400` *(binds per endpoint, whatever the `streaming` switch says)* |
| R13.4 | SSE as `text/event-stream` the default for generated content; its lack of IANA registration documented and never described as registered |
| R13.5 | Every frame carries a documented type; full frame-type vocabulary documented and stated as growable; frame payloads obey §4's representation rules; keep-alive emission documented (or its absence stated) |
| R13.6 | Documented terminal frame carrying the outcome; a stream-ending `error` frame counts as terminal; trailing sentinel tolerated; unbounded streams documented as unbounded |
| R13.7 | Stream-ending errors after commit delivered in an `error` frame carrying a problem object with `status` omitted; never called an `application/problem+json` response; `error` never used for a failure the stream survives |
| R13.8 | Pre-commit errors follow R5.12 unchanged, whatever the request asked to stream |
| R13.9 | Stream carries `operation_id` or `operation_url` matching R10.9's form; terminal frame carries `operation_state` from R10.1's vocabulary, on error frames as well as success ones; operation resource authoritative; full problem document retrievable there |
| R13.10 | Resumption offered where the stream views a retained artifact (data outliving the connection); strictly increasing `stream_position` on every frame; retention window documented; out-of-window resume fails with a defined error |
| R13.11 | Long-poll maximum hold documented; expired hold returns `200` plus an empty result carrying the next `cursor`; never `204` |

## Appendix B — Exception process

*Apparatus, ratified at Gate D 2026-08-09. This appendix describes
process; the normative anchor is R1.7 (no silent deviation).*

An API that cannot meet a rule follows these steps, in order:

1. **Attempt conformance first.** The process exists for genuine
   constraints, not preferences; the request records what was tried.
2. **Write the rationale.** Rule ID, rule strength, what differs, why,
   the evidence, and the blast radius (who is affected and how).
3. **Obtain approval.** The API's governance owner approves deviations
   from recommendation-strength rules; deviations from
   requirement-strength rules go to the standard's owner, because they
   render the API nonconformant unless recorded (R1.7).
4. **Record it.** The approved exception lands in the API's conformance
   note (§1.9 template — this appendix does not restate it): rule ID,
   strength, difference, reason, approver, date.
5. **Bound it.** Every exception carries a review date or expiry;
   open-ended exceptions are not granted.
6. **Revisit.** Exceptions are re-reviewed at every major version of the
   API and at least annually; an expired exception is a deviation again.

## Appendix C — Cheat sheet

Informative quick reference; every entry restates a Part I rule.

**Shape.** `https://api.example.com/v1/{collection}/{id}[/{action}]` —
plural kebab-case collections, at most three resources deep, no trailing
slash, PII never in URIs.

**Casing.** Paths kebab-case · bodies and query parameters snake_case ·
problem `code` snake_case · multi-word action verbs kebab-case.

**Always emit.** `Cache-Control` on every response · `request-id` on
every response · strong `ETag` on updatable resources · `Location` on
201.

**Reserved names.** §1.10 is the register: `sort`, `fields`, `cursor`,
`limit`, `dry_run`, `stream`, `stream_position` and the bracket range
filters; `Idempotency-Key`, `request-id`, the webhook envelope headers;
the reserved media types; the action verbs; the stream members
`operation_id`, `operation_url`, `operation_state`, and `retry_after`; and
the `error` stream frame type.

**Status quick map.**

| Outcome | Status |
| --- | --- |
| Create (single resource) | 201 + `Location` |
| Update | 200 + representation |
| Delete | 204 (soft delete: 200 + tombstone) |
| Bulk, partial completion | 200 + per-item envelope |
| Accepted for async | 202 + operation resource |
| Empty collection | 200 + empty array |
| Malformed syntax | 400 |
| Unauthenticated | 401 |
| Unauthorized | 404 by default (existence masking, R5.10); 403 only where existence is public or the caller is intra-tenant |
| State conflict | 409 |
| Precondition failed / missing | 412 / 428 |
| Unsupported media type | 415 |
| Semantically invalid | 422 |
| Quota exhausted | 429 + `Retry-After` |
| Capacity overload | 503 + `Retry-After` |

**Errors.** `application/problem+json`; `type` and `code` bound by the
fixed template; pairs immutable; catalog published; field failures in
`errors[]`.

**Collections.** `cursor` + `limit` in; `items` + continuation out;
documented stable default order with `id` tiebreak; filters per-field
plus `[gte]`/`[gt]`/`[lte]`/`[lt]`.

**Lifecycle.** `/v1` majors only; additive within a major; `Deprecation`
+ `Sunset` + migration link; deprecated majors live at least 12 months.

**Webhooks.** Sign per topology (Standard Webhooks shared-secret;
RFC 9421 + RFC 9530 cross-org); retry at least 72 h; dead-letter at
least 30 d with redelivery.

**Streaming.** `200` + a self-delimiting media type, never `202`;
`Accept` selects it, never a query parameter; SSE as `text/event-stream`
is the default and is unregistered — say so; typed frames with a
documented terminal frame; errors after commit arrive in an `error` frame
carrying a problem object with `status` omitted; errors before commit are
ordinary problem responses; where an operation resource also exists, the
two share one identity and the operation resource is authoritative.
Details in [`streaming-profile.md`](streaming-profile.md).

## Appendix D — OpenAPI mapping

How Part I lands in an OpenAPI 3.1 document (R4.1). Informative.

| Rule(s) | OpenAPI 3.1 expression |
| --- | --- |
| R4.1 | The document itself: `openapi: 3.1.x`, `jsonSchemaDialect` pinned to JSON Schema 2020-12 |
| R9.1, R9.3 | `servers` entries carry the versioned base (`https://api.example.com/v1`, `/v1beta…`) |
| R2.4, R2.5 | Path keys: kebab-case segments, at most three resources |
| R4.4 | Schema property keys and parameter names satisfy the pinned patterns (lintable — Appendix G) |
| R5.6 | `responses` for create operations declare `201` with a `Location` header object |
| R5.12, R5.13 | A shared problem schema under `components.schemas`; 4xx/5xx responses declare `application/problem+json` content. **Two variants are needed:** the response-carried schema requires `status`, and a second variant for in-band stream frames omits it (R13.7), so the frame payload cannot reference the response schema |
| R3.7 | PATCH `requestBody.content` keyed by `application/merge-patch+json` (and `application/json-patch+json` only where R3.7's JSON Patch option is exercised) |
| R3.9 | A shared `components.parameters` header parameter for `Idempotency-Key`, referenced by every non-idempotent mutation |
| R6.1, R6.5 | A shared collection envelope schema (`items` + continuation member); shared `cursor`/`limit` query parameters with documented default and maximum |
| R6.7, R6.10 | `sort` and `fields` parameters with enumerated permitted values |
| R9.5, R9.6 | `deprecated: true` on sunsetting operations; description carries the sunset date and migration link |
| R8.3, R8.4 | `components.securitySchemes` for OAuth flows and server-to-server keys; per-operation `security` |
| R10.7 | The OpenAPI 3.1 `webhooks` object documents deliveries, envelope headers, and event schemas |
| R11.2, R11.5 | Shared `429`/`503` responses declaring the `Retry-After` header |
| R13.1, R13.4 | Stream operations declare `responses.200.content` keyed by the stream media type (`text/event-stream`, or the NDJSON type where R13.4's record-set case applies); never `application/json` for a concatenated-document body; never a stream media type under `202` |
| R13.2 | One operation declaring both `application/json` and the stream media type under `200`, plus a declared `Vary` response header — or, for R13.2's second limb, a distinct always-streaming path with the stream media type alone and no `Vary` |
| R13.5 | **Not expressible in OpenAPI 3.1** — the specification has no construct for "the body is a sequence of items each matching this schema", so the frame-type vocabulary and its growth statement live in `description` prose. OpenAPI 3.2's item schema covers it where R4.1's verified-toolchain clause is met |
| R13.7 | The `status`-omitting problem variant described in the R5.12/R5.13 row, referenced from the frame-payload documentation — never the shared 4xx/5xx response schema |
| R13.9 | The `operation_id` or `operation_url` member documented on stream frames, referencing the same `components.schemas` entry the R10.9 `202` body uses; `operation_state` on the terminal frame declared with the same enumeration as the operation resource's state member |
| R13.10 | `stream_position` needs two expressions, because an OpenAPI Parameter Object permits only `query`, `header`, `path`, and `cookie` and so cannot describe a body or frame member: a `components.parameters` entry for the query carriage, plus a shared `components.schemas` scalar that the request-body property and the frame-payload property both reference. Retention window in the description |

## Appendix E — Worked example

A fictional flower-delivery platform, "Bloom," at
`https://api.example.com/v1`. Every block is annotated with the rules it
exercises; all identifiers, keys, and signatures are placeholders.
Bloom serves cross-origin browser clients, so every response also
carries `Access-Control-Expose-Headers` listing the standard-bound
headers it emits (R4.17); like the R7.1/R11.7 always-on headers in
E.10, it is omitted from the excerpts below for brevity.

### E.1 Conformance note (§1.9 template, filled in)

```markdown
## Conformance note — Bloom Orders API

Standard: rest-api-standard v1.1.0
Tier: public
Switches: webhooks=on, async-operations=on, streaming=on,
  bulk-operations=off (imports run through the async export/import
  operations; no synchronous bulk endpoint is offered)
Context: single-tenant product; handles PII (delivery addresses);
  public-internet API with third-party clients; no binary payloads
  in v1

Deviations: none.

N/A declarations: none beyond the switch reasons above.
```

### E.2 Create an order — idempotent create, 201 + Location

Exercises R5.6, R3.9, R4.4–R4.7, R7.1/R7.3, R8.1/R8.3, R11.7, R3.10.

```http
POST /v1/orders HTTP/1.1
Host: api.example.com
Authorization: Bearer <access-token>
Content-Type: application/json
Accept: application/json
Idempotency-Key: 8f1c6e0a-4b2d-4c19-9c3a-000000example

{
  "customer_id": "cus_000example",
  "deliver_on": "2026-08-12",
  "amount": 4599,
  "currency": "usd",
  "note": "Birthday bouquet"
}
```

```http
HTTP/1.1 201 Created
Location: https://api.example.com/v1/orders/ord_000example
Content-Type: application/json
Cache-Control: private, no-cache
ETag: "v1-000example"
request-id: req_000example

{
  "id": "ord_000example",
  "customer_id": "cus_000example",
  "status": "pending",
  "deliver_on": "2026-08-12",
  "amount": 4599,
  "currency": "usd",
  "note": "Birthday bouquet",
  "created_at": "2026-08-09T17:04:05-07:00"
}
```

The reading: `id` is a string (R4.5); `amount` 4599 with `"usd"` is
$45.99 in minor units (R4.7); `deliver_on` is a date-only field (R4.6
exception); `created_at` carries an explicit offset (R4.6); the body is
snake_case (R4.4); the response is explicitly non-cacheable-shared
(R7.2/R7.3) and carries the correlation ID (R11.7). The request
authenticates with a bearer token over TLS — Bloom's server-to-server
integrations use scoped API keys, while third-party access on a user's
behalf goes through OAuth (R8.1, R8.3). Replaying the same
`Idempotency-Key` with this payload returns this stored response;
replaying it with a different payload is rejected (R3.9).

### E.3 Validation failure — problem document with field errors

Exercises R5.12, R5.13, R5.15, R5.3.

```http
HTTP/1.1 422 Unprocessable Content
Content-Type: application/problem+json
Cache-Control: no-store
request-id: req_001example

{
  "type": "https://problems.example.com/validation-failed",
  "title": "Validation failed",
  "status": 422,
  "code": "validation_failed",
  "detail": "One or more fields are invalid.",
  "documentation": "https://developers.example.com/errors#validation_failed",
  "errors": [
    {
      "pointer": "/amount",
      "code": "below_minimum",
      "detail": "amount must be at least 500."
    }
  ]
}
```

The reading: `code` `validation_failed` maps to `type`
`…/validation-failed` by the fixed template — underscores to hyphens
(R5.13.1); the human link lives in `documentation`, not in `type`
(R5.13.4); field-level failures ride `errors[]` with JSON Pointers
(R5.15).

### E.4 List orders — envelope, filters, sort, pagination

Exercises R6.1, R6.3, R6.5–R6.8, R2.9.

```http
GET /v1/orders?status=pending&created_at[gte]=2026-08-01T00:00:00Z&sort=-created_at&limit=50 HTTP/1.1
Host: api.example.com
Authorization: Bearer <access-token>
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: private, no-cache
request-id: req_002example

{
  "items": [
    {
      "id": "ord_000example",
      "customer_id": "cus_000example",
      "status": "pending",
      "deliver_on": "2026-08-12",
      "amount": 4599,
      "currency": "usd",
      "created_at": "2026-08-09T17:04:05-07:00"
    }
  ],
  "next_cursor": "b3BhcXVlLW5vdC1jb25zdHJ1Y3RhYmxl"
}
```

The reading: equality plus a bracket range filter, AND-combined (R6.8);
`-created_at` descending (R6.7); the envelope carries `items` and
`next_cursor` (R6.1); the documented default order is `-created_at`
with `id` as tiebreak (R6.6); the next page is
`GET /v1/orders?cursor=…&limit=50` and the cursor is opaque (R6.3,
R12.5). An empty result is `200` with `"items": []` (R6.2).

### E.5 Partial update and the destructive guard

Exercises R3.7, R3.8, R3.10, R3.11, R7.4, R5.11.

```http
PATCH /v1/orders/ord_000example HTTP/1.1
Host: api.example.com
Authorization: Bearer <access-token>
Content-Type: application/merge-patch+json
If-Match: "v1-000example"

{
  "note": null
}
```

A `200` response returns the representation without `note` — under Merge
Patch, `null` deletes (R3.7), the one exception to null-equals-absent
(R3.8). A DELETE without its precondition is refused:

```http
DELETE /v1/orders/ord_000example HTTP/1.1
Host: api.example.com
Authorization: Bearer <access-token>
```

```http
HTTP/1.1 428 Precondition Required
Content-Type: application/problem+json
Cache-Control: no-store
request-id: req_003example

{
  "type": "https://problems.example.com/precondition-required",
  "title": "Precondition required",
  "status": 428,
  "code": "precondition_required",
  "detail": "DELETE on this resource requires If-Match."
}
```

With `If-Match` supplied and matching, the delete returns
`204 No Content` (R5.7).

### E.6 An action — cancel

Exercises R2.11, R2.13, §1.10 verb registry, R5.1.

```http
POST /v1/orders/ord_000example/cancel HTTP/1.1
Host: api.example.com
Authorization: Bearer <access-token>
```

`200 OK` returns the representation with `"status": "canceled"` —
`cancel` is the registered terminal, irreversible stop. There is no
`POST /v1/orders/cancel`: collection-level actions do not exist (R2.13).

### E.7 Asynchronous work — export as an operation resource

Exercises R10.1, R10.2, R10.9, R5.1.

```http
POST /v1/order-exports HTTP/1.1
Host: api.example.com
Authorization: Bearer <access-token>
Content-Type: application/json
Idempotency-Key: 2c7d1f4b-8a3e-4b6f-b1d0-000001example

{
  "created_after": "2026-01-01T00:00:00Z"
}
```

```http
HTTP/1.1 202 Accepted
Location: https://api.example.com/v1/operations/op_000example
Retry-After: 5
Content-Type: application/json
Cache-Control: private, no-cache
request-id: req_004example

{
  "id": "op_000example",
  "status": "running",
  "export_id": "exp_000example",
  "created_at": "2026-08-09T17:08:00-07:00",
  "expires_at": "2026-08-16T17:08:00-07:00"
}
```

The reading: the operation is addressable, has documented terminal
states (`succeeded`, `failed`, `canceled`), an expiry, and a failure
representation (R10.1); the body `id` plus Bloom's documented
`/v1/operations/{operation_id}` template satisfies R10.9's body clause,
and `Location` carries the same operation URI — the two are required to
agree (R10.9); `Retry-After` paces the polling (R10.2); an abandonable
run is stopped with `POST /v1/operations/op_000example/cancel` (R10.2,
§1.10). The same export is also reachable as a stream — E.11 shows that
channel and the R13.9 obligation that binds the two together.

### E.8 A webhook delivery

Exercises R10.5, R10.7, R12.8, R12.9.

```http
POST /webhooks/bloom HTTP/1.1
Host: consumer.example.net
Content-Type: application/json
webhook-id: msg_000example
webhook-timestamp: 1786295445
webhook-signature: v1,<signature-example-redacted>

{
  "type": "order.canceled",
  "created_at": "2026-08-09T17:10:45Z",
  "version": 3,
  "data": {
    "id": "ord_000example",
    "status": "canceled"
  }
}
```

The reading: the Standard Webhooks envelope for the shared-secret
topology (R10.7.1) — the signature covers `id.timestamp.payload`, the
secret is `whsec_`-prefixed and never appears on the wire. `version` is
the monotonic per-event marker (R10.5). The consumer verifies over the
raw body before parsing, enforces the timestamp window, dedupes on
`webhook-id`, compares in constant time, and acks before processing
(R12.8, R12.9).

### E.9 Rate limiting

Exercises R11.2, R11.5, R12.2.

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
Content-Type: application/problem+json
Cache-Control: no-store
request-id: req_005example

{
  "type": "https://problems.example.com/rate-limit-exceeded",
  "title": "Rate limit exceeded",
  "status": 429,
  "code": "rate_limit_exceeded",
  "detail": "Retry after the interval in Retry-After."
}
```

Bloom also advertises quota state with the pinned draft-11 `RateLimit`
fields (R11.3); their concrete syntax is deliberately not reproduced
here — the pinned draft is the reference, and this appendix avoids
restating a moving wire format.

### E.10 Deprecation signals

Exercises R9.5, R9.6, R9.7.

A v1 response after the v2 launch (deprecation announced 2026-09-01,
sunset one year later; only the deprecation-relevant headers are
excerpted — the full response also carries the always-on headers of
R7.1 and R11.7):

```http
Deprecation: @1788220800
Sunset: Wed, 01 Sep 2027 00:00:00 GMT
Link: <https://developers.example.com/migrate-v2>; rel="deprecation"
```

The reading: `Deprecation` is a structured-field date (RFC 9745);
`Sunset` is an HTTP-date (RFC 8594) — deliberately different formats —
and the window honors the 12-month floor (R9.7).

### E.11 Streaming the export as it is produced

Exercises R13.1, R13.2, R13.4–R13.7, R13.9, R13.10, R5.1, R12.10. This is
the same export capability as E.7, reached through the streaming channel —
the pair together is what R13.9 governs.

`/events` is R13.2's **second limb** — a distinct resource that streams
unconditionally. It performs no selection, so it emits no `Vary` and R4.10
does not reach it. (The first limb, one endpoint negotiating both shapes on
`Accept`, is what `GET /v1/order-exports/{export_id}` itself would use if
Bloom offered a non-streamed representation of the same events.) The export
identifier comes from E.7's `202` body, which carries `export_id` alongside
the operation `id`:

```http
GET /v1/order-exports/exp_000example/events HTTP/1.1
Host: api.example.com
Authorization: Bearer <access-token>
Accept: text/event-stream
```

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: private, no-store
request-id: req_010example

event: export.started
data: {"operation_id":"op_000example","stream_position":1,"total_estimate":4820}

event: export.chunk
data: {"stream_position":2,"rows":500,"part_url":"https://files.example.com/exp_000example/part-1.csv"}

event: export.completed
data: {"stream_position":3,"operation_state":"succeeded","rows_total":4820,"next_cursor":null}
```

The failure case, on a stream that had already committed `200`:

```http
event: error
data: {"type":"https://problems.example.com/export-source-unavailable",
data:  "title":"The order source became unavailable mid-export",
data:  "code":"export_source_unavailable",
data:  "operation_id":"op_000example",
data:  "operation_state":"failed",
data:  "stream_position":47}
```

The reading. The response is `200` with `text/event-stream`, an accurate
self-delimiting media type, and it is not a `202` (R13.1, R13.4). Every
frame carries a type from Bloom's documented vocabulary (R13.5), and
`export.completed` is the terminal frame, so a connection that drops before
it is truncation rather than a short result (R13.6, R12.10). `part_url` is
deliberately not named `url`: R10.9 gives `url` a registered meaning in an
operation body, and reusing it for a part file would be the
same-name-different-concept hazard R2.3 and R1.8 exist to prevent.

The error frame is the second carve-out from R5.12 in action (R13.7): the
payload is a problem object carrying `type`, `title`, and `code` bound by
R5.13's template and listed in Bloom's R5.16 catalog — and it **omits
`status`**, because the response status is `200` and RFC 9457 §3.1 forbids
an advisory `status` that disagrees with it. The frame is not an
`application/problem+json` response and is never described as one. R5.1 is
satisfied, not violated: the `200` was correct for the outcome known when it
was generated.

`operation_id` is the reserved member (§1.10) carrying the identifier R10.9
binds into E.7's `202` body — Bloom uses R10.9's `id` form, so it carries
`operation_id` rather than `operation_url`. That is what makes the stream and
the operation resource one capability with one identity (R13.9). Both terminal
frames carry `operation_state` — `succeeded` on the completion frame, `failed`
on the `error` frame — drawn from the operation resource's own vocabulary
(R10.1), so the two channels are comparable rather than merely both present.
The member is not called `status`, because R13.7 forbids `status` on the
problem object and one name across both outcomes is what makes the comparison
mechanical. `GET /v1/operations/op_000example` remains authoritative and
serves the full problem document — with `status` — as a real
`application/problem+json` response.

`stream_position` increases monotonically and is what a client echoes to
resume (R13.10); Bloom documents a 30-minute retention window, and a resume
outside it fails with a defined error rather than silently restarting. It is
not a `cursor`, and R12.5's opacity obligation does not reach it.

## Appendix F — Framework and gateway mapping

Informative. Each row records a verified finding from the research layer
with its report; findings carry the date they were verified and may age.

| Surface | Recorded finding | Bearing on this standard | Source |
| --- | --- | --- | --- |
| ASP.NET Core | Emits RFC 9457 Problem Details by default; the .NET 7→8 transition changed a problem `type` identity | R5.12 is near-free on this stack; the identity break is the failure R5.13.2 (immutability) exists to prevent | `baseline-02c` (2026-07-25), `baseline-02f` (2026-08-09) |
| Spring | RFC 9457 support enabled via a single property | R5.12 adoption cost is one configuration line | `baseline-02c` (2026-07-25) |
| Spectral | Silently ignores OpenAPI 3.2 constructs — a lint pass that validates nothing | Why R4.1 gates 3.2 on a verified toolchain | `baseline-02b` (2026-07-25) |
| Redocly CLI | Full OpenAPI 3.2 support | The one verified all-3.2 tool in the surveyed chain | `baseline-02b` (2026-07-25) |
| swagger-parser / openapi-generator | Open, unaddressed 3.2 issues (#2248, #22728) | The registered re-check triggers for flipping R4.1's default | `baseline-02b` (2026-07-25) |
| Cloudflare | Network-wide `problem+json` rollout 2026-03-11 with a measured 55–64× payload reduction for agent consumers; yet its own edge once served `text/plain` for an enforced rate limit despite `Accept: application/problem+json` | Live proof of both R5.12's value and its infrastructure carve-out — hence R12.7 | `baseline-02d`, `baseline-02e` (2026-08-09, live-verified) |
| Kong | The only surveyed gateway documenting the phantom-token pattern natively | The token-format axis (R8.10) names phantom-token as integration work everywhere else | `baseline-03g` (2026-08-09) |
| AWS Cognito | Documents that revoked JWTs still verify | Why the token-format axis pairs any client-visible JWT with a revocation-propagation plan | `baseline-03g` (2026-08-09) |
| Microsoft Entra | Ships neither DPoP nor RFC 8705 today; CAE revocation lag documented at up to 15 minutes; mTLS PoP announced as the future direction | Why bearer-over-TLS is the ratified default and Entra mTLS PoP is the highest-value watch item | `baseline-03g` (2026-08-09) |
| MCP | The specification mandates plain `Bearer` | A whole 2025–2026 protocol ecosystem on the ratified default | `baseline-03g` (2026-08-09) |

## Appendix G — Executable conformance fixtures

Two fixtures: a Spectral ruleset over the contract document, and a
live-probe table over a running API. The ruleset,
[`conformance/spectral.yaml`](conformance/spectral.yaml), is drafted
from the pinned patterns in Part I and cites rule IDs in each rule
description. It is execution-verified: run 2026-08-10 with
`@stoplight/spectral-cli` 6.16.3 against
[`conformance/fixture-violations.yaml`](conformance/fixture-violations.yaml)
(a deliberately violating OpenAPI document covering each rule, both
header directions, POST and PUT creates, and a `$ref`-only envelope
schema the ruleset deliberately does not traverse), all fourteen expected
findings fired — the twelve of version 1.0.0 plus the two §13 rules added
in 1.1.0. The rules are conservative heuristics: each description
states its known false-positive and false-negative limits, and
warn-severity rules exist to be reviewed, not blindly enforced.

**What §13 can and cannot be checked for statically.** `R13.1`'s
`202`-never-streams prohibition and `R13.2`'s negotiation-implies-`Vary`
obligation are contract-level and are in the ruleset. The rest are not, and
the reason is worth stating: OpenAPI 3.1 has no construct for "the body is a
sequence of items, each matching this schema," so `R13.5`'s frame-type
vocabulary cannot be expressed in the contract document that R4.1 names as
the source of truth — it lives in prose, and only a live probe can check it.
`R13.7`'s frame payload is likewise a body-level fact no contract asserts.

Live probes — each row is one request against a deployed API and the
response that conformance predicts:

| Probe | Request | Expected | Rules |
| --- | --- | --- | --- |
| Trailing slash | `GET /v1/orders/` | 308 to `/v1/orders` | R2.6 |
| Rehearsal guard | `POST …?dry_run=true` to an endpoint without dry-run | 400 | R1.9 |
| PATCH media type | PATCH with `Content-Type: application/json` | 415 + `Accept-Patch` | R3.7, R5.11 |
| Destructive guard | DELETE a guarded resource without `If-Match` | 428 | R7.4, R5.11 |
| Unknown method | An unimplemented method on a real path | 405 + `Allow` | R5.11 |
| Empty collection | List with a filter matching nothing | 200 + empty `items` | R6.2 |
| Auth split | No credentials | 401 | R5.9 |
| Existence masking | Another tenant's resource ID | 404 | R5.10 |
| Quota | Exceed the published limit | 429 + `Retry-After` | R11.2 |
| Error negotiation | Force an error with `Accept: application/problem+json` | Problem document with template-bound `type`/`code` | R5.12, R5.13 |
| Correlation | Any request | `request-id` present on the response | R11.7 |
| 202 discovery | Start an async operation | Body carries `id` or `url`; any `Location` is the operation's absolute URI and agrees with the body | R10.9 |
| Cache posture | Authenticated GET | `Cache-Control: private, no-cache` (or stricter) | R7.1–R7.3 |
| Stream guard | `stream=true` to an endpoint without streaming | 400, never a silently non-streamed 200 | R13.3 |
| Stream negotiation | `Accept: text/event-stream` on an endpoint offering both shapes | `200` + `text/event-stream` + `Vary: Accept` | R13.2, R4.11 |
| Stream termination | Consume a stream to completion | A documented terminal frame arrives before close | R13.6 |
| Stream identity | Stream a capability that also has an operation resource | Frames carry `operation_id` or `operation_url`; terminal state matches the operation resource | R13.9 |
| Resume window | Resume with a `stream_position` older than the documented window | A defined error, never a silent restart from the beginning | R13.10 |
| Long-poll expiry | Hold past the documented maximum | `200` + empty result + next `cursor`; never `204` | R13.11 |
