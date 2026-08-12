# Decisions — baseline-04 (streaming)

*Phase 6 ratification record, following the Gate C pattern. A `baseline`
report proposes; only this file ratifies. Classification per `PLAN.md`
Phase 2: protocol requirement · evidence-backed default · project policy ·
exception · unresolved question.*

**Walk completed 2026-08-10.** Six forks walked one at a time; the remaining
fifteen principles ratified en bloc. Evidence base:
[`survey-08-streaming.report.2026-08-10.md`](../reports/survey-08-streaming.report.2026-08-10.md)
(descriptive, 14 contested axes) and
[`baseline-04-streaming.report.2026-08-10.md`](../reports/baseline-04-streaming.report.2026-08-10.md)
(prescriptive, 20 proposed `ST-*` principles).

**Scope rulings carried in from the phase opening (2026-08-10).** Coverage is
SSE, long-polling, and streaming HTTP bodies; **WebSockets are an explicit
non-goal**, because after the `101` upgrade the exchange is no longer HTTP
request/response and none of this standard's status-code, media-type,
`Problem Details`, or applicability-switch machinery binds it.

---

## P6-D0 — Deliverable shape: compact normative §13 plus informative companion

**Decision (2026-08-10): RATIFIED.** Confirmed explicitly by the owner after
being flagged as an interpretation pending confirmation.

**Classification:** apparatus (project policy).

The `R13.x` rules live in **§13 of `rest-api-standard.md`**. A separate
**informative companion document** carries the explanatory body — mechanism
comparison, wire examples, vendor evidence, deployment and client guidance.
Progressive disclosure is the purpose: an API that does not stream declares
the `streaming` applicability switch off and never opens the companion.

**Consequence that made this decision load-bearing:** the conformance surface
stays single. Appendix A carries `R13.x` checklist rows, Part II carries their
provenance rows, and `conformance/spectral.yaml` remains the one ruleset. Every
`ST-*` principle below is classified `Normative §13` or `Companion` on this
basis; the rule-split reading would have invalidated that classification.

**Declined:** a separate extension document holding the rules themselves —
splits the conformance surface into two checklists and makes "is this API
conformant" ambiguous. Also declined earlier at the phase opening: distributing
streaming rules across §3, §9, and §10, which avoids amending §1.5's
section-count declaration but leaves no single place to read the posture.

---

## P6-D1 — Negotiation: `R4.10` composition wins over field compatibility

**Decision (2026-08-10): RATIFIED as proposed — option (a).** Ratifies
**`ST-002`** and **`ST-003`**.

**Classification:** project policy, derived from ratified `R4.10`/`R4.11`.

### The ratified rule

Where one endpoint serves both a streamed and a non-streamed representation,
the choice **MUST** be made by content negotiation on `Accept`, and the
response **MUST** carry `Vary: Accept`. A query parameter **MUST NOT** select
between them. An API **MAY** instead expose a distinct resource that streams
unconditionally, in which case no negotiation applies and `R4.10` does not
reach it.

`stream` is reserved as a request-modifier name. An endpoint that does not
implement streaming and receives `stream` — query parameter or body member,
any value — **MUST** reject with `400`, never silently answer with a
non-streamed response. **The guard binds per endpoint, not per API:** every
endpoint that does not implement streaming owes it, including endpoints of an
API whose `streaming` switch is on.

**Amendment ratified 2026-08-10 (Phase 6 review walk) — the conflicting-modifier
case.** As originally ratified this decision covered only an endpoint that
does *not* stream, leaving the streaming-capable case unruled. The internal
review rated that gap critical, and it is not an edge: a body-flag
`"stream": true` alongside `Accept` negotiation is the shape every surveyed
AI provider ships, so it arises on ordinary traffic. Three answers were each
defensible and mutually exclusive — honor the flag (forbidden, `R13.2` puts
selection on `Accept`), ignore it (the silent-ignore hazard this guard exists
to prevent), or reject it (unauthorized). **Ratified:** on an endpoint that
does implement streaming, `Accept` governs and `stream` selects nothing; a
`stream` modifier disagreeing with the negotiated representation — asking for
a stream where `Accept` selected the non-streamed form, or the reverse —
**MUST** be rejected with `400` rather than silently resolved in either
direction.

*Process note, recorded because the discipline matters more than the rule:*
this requirement was written into `R13.3` during the review fix pass before
being ratified here — drafting inventing policy, which the decision layer
exists to prevent. It was caught by the PR review reading the standard
against this record, and ratified on 2026-08-10 rather than quietly kept.
Declined at ratification: a weaker form requiring only that the API
*document* its behavior, which would mandate rejection in the easy case (the
endpoint does not stream at all) and merely ask for documentation in the hard
one.

### Justification

`R4.10` already governs media-type selection and already forbids the
query-parameter form; streaming changes the response media type, so it is
media-type selection. Choosing `Accept` inherits `R4.10`'s `406` guard and
`R4.11`'s `Vary` obligation for free. RFC 8895 §8.2 — Standards Track — sends
`Accept: text/event-stream`, so the mechanism is not merely theoretical.

**The cost is accepted knowingly.** Every deep-dive provider (OpenAI,
Anthropic, Gemini) negotiates with a request-body flag, and Gemini's `alt=sse`
is the forbidden query-parameter shape outright. An API that literally wraps
those semantics is nonconformant on this rule and must record it in its
conformance note under `R1.7`.

**Precedent relied on:** `R8.2` already overrules Gemini's `key=` query
parameter on BCP 240 grounds, and nobody proposed relaxing it for field
compatibility.

**Declined:** (b) a body flag as the primary mechanism with a named deviation —
puts a permanent recorded deviation in every streaming API's conformance note;
(c) scoping `R4.10` to exempt streaming — weakens a ratified rule for one case.

**Confidence: moderate.** The composition is firm; the departure from unanimous
practice is a policy call, and it is recorded as one.

---

## P6-D2 — The post-commit error: carve out from `R5.12`, scope `R5.13`

**Decision (2026-08-10): RATIFIED as proposed — option (a).** Ratifies
**`ST-007`**. Enacted as a **MINOR amendment** to `R5.12` and `R5.13`.

**Classification:** project policy on the carve-out and the omission; the two
constraints it obeys are protocol requirements.

### The ratified rule

An error raised after the response status is committed **MUST** be delivered
in-band, in a frame of the reserved `error` type, whose payload is a problem
details object per RFC 9457 §3 carrying `R5.13`'s required members **other than
`status`** — `type`, `title`, and `code` — bound by `R5.13`'s `type`/`code`
template and listed in the `R5.16` catalog. `detail` and extension members are
permitted exactly as on any other problem document. The object **MUST omit the
`status` member**, and the frame **MUST NOT** be described as an
`application/problem+json` response.

Where an operation resource exists, the full problem document — carrying
`status`, as a real `application/problem+json` response — **MUST** be
retrievable from it (`ST-009`).

### Justification

Two protocol facts, independently verified against raw RFC text for this walk,
fix the shape and leave only the resolution to this project:

1. **RFC 9457 §3.1**, verbatim: *"The `status` member, if present, is only
   advisory; it conveys the HTTP status code used for the convenience of the
   consumer. Generators MUST use the same status code in the actual HTTP
   response, to assure that generic HTTP software that does not understand this
   format still behaves correctly."* A problem document carrying `status: 503`
   inside a committed `200` therefore violates a Standards Track MUST. The same
   sentence's "if present" makes omission explicitly permitted.
2. **RFC 9110 §6.5.1** excludes trailer fields as the alternative channel: *"in
   most cases, the trailers are simply discarded,"* and a server *"SHOULD NOT
   generate trailer fields that it believes are necessary for the user agent to
   receive."*

RFC 9457 Appendix C anticipates embedding the problem-details model in another
format, which is what the `error` frame does.

This preserves everything `R5.12` exists to buy — stable machine identity, a
published catalog, one client error handler — and surrenders only the media-type
label and the advisory number, both of which are structurally unavailable once
the status is committed. It is strictly better than the field's unanimous
answer: **0 of 7** surveyed references emit `problem+json` in-band and **0 of 7**
use trailers, every one using a private schema with no registered identity.

**Declined:** (b) carrying the would-be status in a reserved extension member —
preserves AWS's demonstrated retryability signal but mints a new reserved name
and a member whose relationship to RFC 9457's `status` must be explained to
every reader; the retryability signal is already in the `R5.16` catalog.
(c) permitting a private in-band schema — matches all vendors, abandons generic
interpretability, leaves `R5.12` simply unsatisfied.

**Confidence: moderate-high** — the constraints are certain; the resolution
among them is this project's choice and is labeled as such.

---

## P6-D3 — `R5.1` gets an explicit streaming scope

**Decision (2026-08-10): RATIFIED — option (b).** A **MINOR amendment** to
`R5.1`, atomic across all five surfaces.

**Classification:** apparatus correction to a ratified protocol-requirement
rule; the amendment is project policy, the underlying rule is unchanged
elsewhere.

### The conflict

`R5.1` reads: *"The status code MUST match the registered semantics of the
outcome. A failed operation MUST NOT return 2xx."* (Provenance `HS-010`,
protocol requirement, RFC 9205, confidence high.) A stream that fails after
commit **has** returned `200` for a failed operation. The collision is head-on,
applies to the most common streaming error case, and **the descriptive survey
never surfaced it** — it was found by reading this standard's own ratified rules
against the field evidence.

### The ratified resolution

`R5.1` is amended to scope its prohibition explicitly: the status code binds to
the outcome **as known when the status is generated**, and a streaming response
whose failure occurs after the status is committed discharges its error
obligation through `ST-007` instead. The scope lives in `R5.1`'s own rule text.

**Justification.** `R1.7` is "No silent deviation." A reading load-bearing
enough that an entire section depends on it belongs in rule text, visible to
someone reading §5 alone, not in a §13 preamble sentence they may never open.

**Declined:** (a) recording the reading in the §13 preamble — cheapest, but
leaves a core rule's text unqualified. (c) leaving it — not viable; a reviewer
finds the collision.

---

## P6-D4a — `text/event-stream` admitted to the §1.10 reserved media types

**Decision (2026-08-10): RATIFIED — option (a).** Supports **`ST-004`**.

**Classification:** project policy.

`text/event-stream` is added to §1.10's reserved media-type table **with its
unregistered status disclosed in the row**. It is the table's first
unregistered entry; the three existing entries (`application/problem+json`,
`application/merge-patch+json`, `application/json-patch+json`) are all
RFC-registered.

**Justification.** The disclosure pattern is already established in the same
section, on two rows of the headers table: `Idempotency-Key` ("the IETF draft
that standardized this shape expired 2026-04-18; never cite it as a standard")
and `RateLimit` ("an unpublished Internet-Draft; MUST NOT be described as
standards-compliant"). Neither was excluded for lacking standing; each was
admitted with its standing disclosed. The register's purpose — same concept,
same name, everywhere — applies to an unregistered name with equal force.

**The registration gap, recorded.** Verified independently 2026-08-10:
`text/event-stream` is absent from the 105-entry IANA `text/*` subregistry, and
its per-type URL returns **404** against a `text/html` control returning
**200**. The WHATWG text carries a registration *template* prefaced "will be
submitted to the IESG for review, approval, and registration with IANA"; the
submission has not happened.

**Why this passes the test that declined `Operation-Location`.** In
`baseline-02i` a registered alternative existed and worked. Here the only
registered option, `application/json-seq` (RFC 7464), has **zero** HTTP
adoption among the references and no browser parser. The test is applied, not
waived: the gap is disclosed in the rule itself, and `ST-004` is classified
`[COMPARATIVE]`, never `[FACT]`.

**Declined:** (b) keeping the table registered-only and carrying the media type
in §13 rule text — preserves a property of the table at the cost of splitting
where a reader looks a reserved name up.

---

## P6-D4b — §1.10 opens a fifth reserved-name category

**Decision (2026-08-10): RATIFIED — option (a).**

**Classification:** apparatus (project policy).

§1.10 gains a fifth category, **reserved stream frame types**, alongside
reserved query parameters, headers, media types, and action verbs. Its first
entry is `error`, the frame type that `P6-D2` makes the sole carrier of the
mid-stream problem object.

**Justification.** Three independent vendors converged on a named `error`
event, and `P6-D2` makes the name load-bearing for interoperability rather than
cosmetic: generic tooling must recognize it to find the problem document. That
is exactly the "same concept, same name, everywhere" ground on which the other
four categories rest. This is a structural addition to §1.10, not a row in an
existing table, which is why it was walked rather than assumed at drafting.

**Declined:** (b) binding the name in §13 rule text only — leaves the one frame
name generic tooling must recognize outside the register that exists to hold
exactly such names.

---

## P6-D5 — The resumption position gets its own reserved name

**Decision (2026-08-10): RATIFIED — option (a), the new reserved name.**
Completes **`ST-010`**.

**Classification:** project policy on the §1.10 registry's shape.

The stream-resumption position is reserved under a **new name**, not by reusing
the existing `cursor`. The exact name is a drafting choice; the decision here is
that it is distinct from `cursor`.

**The genuine fork, recorded because it was close.** `R12.5` reads: *"Clients
MUST treat cursors (R6.3) as opaque: never construct, modify, or persist them
beyond their documented lifetime,"* and §1.10 reserves `cursor` as an opaque,
non-constructable pagination position (`AC-013`). `ST-010` requires a
**monotonically ordered** position. OpenAI's mechanism is a readable integer the
client echoes from `sequence_number` — echoing is not constructing, but a
monotonic integer invites the arithmetic `R12.5` forbids. Kubernetes'
`resourceVersion` is a version token that behaves like a cursor.

**Why the new name won.** The contradiction is in the rule text, not in anyone's
judgment: `R12.5` says never construct or modify, and `ST-010` requires visible
ordering — precisely the property that makes construction tempting. A distinct
name lets both disciplines stay literally true, where reuse forces one of them
to be qualified.

**The counterargument is recorded and is not weak:** two names for two nearly
identical client obligations is the exact divergence §1.10 exists to prevent,
and a resumption position is opaque in every respect that matters, since the
client only ever echoes it. The owner ruled for the new name with this
counterargument in view.

---

## Batch — the remaining fifteen principles, ratified en bloc

**Decision (2026-08-10): RATIFIED en bloc as proposed.** Following the Gate C
precedent, where the `HS`, `AC`, and `OP` batches were each ratified en bloc
after the contested forks were walked. None of the fifteen was flagged as
contested by the research, and each was presented in full before confirmation.

### Normative §13 (seven)

| ID | Strength | Ratified obligation |
| --- | --- | --- |
| `ST-001` | MUST | A streaming response is `200 OK` with a `Content-Type` naming a self-delimiting stream media type. `202 Accepted` MUST NOT be used for a streaming response; a stream of concatenated JSON documents MUST NOT be labeled `application/json`. |
| `ST-005` | MUST | Every frame carries a documented type; the API documents its full frame-type vocabulary and states that the vocabulary may grow. |
| `ST-006` | MUST | A stream ends with a documented terminal frame carrying the final outcome, so normal completion is distinguishable from truncation. A trailing sentinel frame MAY follow, and clients MUST tolerate one. Unbounded streams document that they are unbounded. |
| `ST-008` | MUST | An error detected **before** the status is committed follows `R5.12` unchanged and is servable as `application/problem+json`, whatever the request asked for. A streaming request modifier governs the success representation only. |
| `ST-009` | MUST | Where one capability is exposed both as a stream and as an `R10.9` operation resource, they are one capability with one identity: the stream carries the `R10.9` operation identifier, both channels report the same terminal state with the operation resource authoritative, and the full problem document is retrievable from it. |
| `ST-011` | MUST | A long-polling endpoint documents its maximum hold duration; an expired hold returns `200` with a well-formed empty-result representation carrying the next cursor. `204 No Content` MUST NOT be used for an expired hold. |
| `ST-012` | MUST | Client obligations: treat a close without the terminal frame as truncation and never treat partial content as complete; ignore unrecognized frame types; never depend on keep-alive frames arriving on a schedule; never recover from truncation by replaying a non-idempotent request without an idempotency key. |

**`ST-011`'s `204` prohibition rests on two independent grounds**, both
recorded: `R5.7` already binds `204` to a successful DELETE, and the WHATWG HTML
Living Standard §9.2 gives `204` a conflicting reserved meaning on this exact
surface — a client "can be told to stop reconnecting using the HTTP 204 No
Content response code" — so a server offering both mechanisms on one path would
have the two meanings collide.

**`ST-009` rests on a single shipped exemplar** (OpenAI's `background: true`
unified design) against a contrary guideline (AIP-151's "The response must not
be a streaming response"), which is declined with reasons. A second exemplar
would materially strengthen it; this is recorded as the batch's weakest link.

### Informative companion (eight)

| ID | Strength | Ratified guidance |
| --- | --- | --- |
| `ST-013` | MAY | Newline-delimited JSON for record streams and bulk result sets, as `application/x-ndjson`, with its unregistered status disclosed and RFC 6648's discouragement of new `x-` names noted. `application/json-seq` (RFC 7464) where a registered media type is required, at the cost of no client ecosystem. |
| `ST-014` | SHOULD | An SSE stream carries each frame's type in both the `event:` field and a payload `type` member. |
| `ST-015` | SHOULD | Document the streaming contract in media-type terms, not as a `Transfer-Encoding: chunked` requirement — chunked coding exists only in HTTP/1.1. See the citation-precision note below. |
| `ST-016` | SHOULD | Document whether keep-alive frames are emitted and in what form; a keep-alive frame carries no application state. **No interval is mandated** — none is citable. |
| `ST-017` | SHOULD | Per-stream metadata travels in the stream body rather than in response headers, which keeps `R4.17`'s exposed-header list unchanged by Phase 6. |
| `ST-018` | SHOULD | Browser clients consume streams with a `fetch`-based reader or a first-party relay holding the credential server-side. |
| `ST-019` | SHOULD | Deployment guidance on intermediary buffering, compression, and idle timeouts. |
| `ST-020` | MAY | Frame padding where frame sizes could reveal sensitive information to an on-path observer, with the behavior and any opt-out documented. |

**Citation precision — `ST-015`, 2026-08-10 (same-day, pre-release).** The
supporting citation was filed as RFC 9112 §7.1. Verified against raw RFC
text: **§7.1 defines** chunked transfer coding, while **§6.1 scopes** it —
"Transfer-Encoding was added in HTTP/1.1," and a server "MUST NOT send a
response containing Transfer-Encoding unless the corresponding request
indicates HTTP/1.1 (or later minor revisions)." Additionally, **RFC 9113**
makes the header worse than unused over HTTP/2: it is connection-specific,
and "Any message containing connection-specific header fields MUST be
treated as malformed." The ratified guidance is unchanged and strengthened;
the companion carries the corrected citations.

**`ST-018` records a capability consequence rather than creating a rule.**
*(The factual premise below was corrected by the Codex second lens — see item
5 of the dated annotation at the end of this record. `EventSource` can send
ambient credentials, including TLS client certificates and HTTP-authentication
entries, not cookies alone; what it cannot carry is a caller-supplied header.
The guidance is unchanged.)*
`EventSource` cannot set request headers, so it cannot send `Authorization`; its
only native credential is a cookie. The field's one workaround is a
query-parameter key, which ratified `R8.2` already forbids on BCP 240 grounds.
The consequence — **a browser-direct `EventSource` connection cannot be
authenticated under this standard** — is stated openly, not engineered around.

**`ST-019` describes `X-Accel-Buffering` without reserving it.** The header is
nginx-documented and widely used for SSE, and is **absent from the IANA HTTP
Field Name Registry** (0 matches across 259 rows, verified 2026-08-10). It fails
the same registry test applied to `Operation-Location`, so it may be described
as infrastructure practice and must never be reserved or mandated.

### Axes deliberately carrying no §13 rule

| Axis | Why no rule |
| --- | --- |
| **S9** Keep-alive | The only available authority is an authoring note in a Living Standard, and the field shows three mechanisms across three references. There is nothing to make normative. Companion guidance (`ST-016`) plus `ST-012`'s client clause. |
| **S10** Browser authentication | Ratified `R8.2` already decides it. `ST-018` states the consequence. |
| **S11** CORS exposure | Ratified `R4.17` already binds any header a provider chooses to emit. `ST-017` steers metadata into the body so the list stays unchanged. |
| **S12** Final-chunk metadata shape | Four shapes across six references, no dominant pattern, and no interoperability consequence once frames are typed (`ST-005`) and the terminal frame is defined (`ST-006`). |

---

## Consequential amendments to version 1.0 — the drafting contract

Every item below is a change to released 1.0 text and is atomic across the
amendment rule's five surfaces (rule text · decision record · Part II
provenance row · Appendix A checklist row · Appendix E worked example).

| Surface | Change | Source |
| --- | --- | --- |
| `R5.1` | Explicit streaming scope: status binds to the outcome as known when the status is generated | `P6-D3` |
| `R5.12` | Named carve-out for post-commit stream errors — the second named exception, alongside the infrastructure carve-out | `P6-D2` |
| `R5.13` | Required member set scoped to response-carried problem documents, so `status` is not required in-band | `P6-D2` |
| `R1.3` | `ST-*` named in the frozen-series list once ratified, so the freeze list stays complete | Phase 6 framing |
| §1.2 | The streaming deferral is replaced by a pointer to §13 plus the stated WebSockets non-goal and its reason | Phase 6 scope ruling |
| §1.5 | The "`R<section>` prefix space is fixed at twelve normative sections" declaration is amended | `P6-D0` |
| §1.8 | New `streaming` applicability switch, gating the `ST-*` rules as marked; plus three scoping principles Phase 6 forced into the open — where a rule's scope is stated, that a rule may require two switches, and that a switch never waives a guard | `ST-001`–`ST-012` |
| §1.9 | The conformance-note template gains a `streaming` switch line in the same on-or-off form as the other three, without which an API following the template verbatim cannot satisfy `R1.6` | `P6-D0` (found in the Phase 6 review wave) |
| §1.11 | The **Reserved name** definition gains the fifth category; new entries for *stream*, *frame*, *terminal frame*, *self-delimiting stream media type*, and *status committed* | `P6-D4b` and the Phase 6 review wave |
| §1.10 | `stream` reserved query-parameter/body-modifier row; `text/event-stream` media-type row with disclosure; new fifth category for reserved stream frame types, first entry `error`; new reserved name for the resumption position | `P6-D1`, `P6-D4a`, `P6-D4b`, `P6-D5` |
| §12 | `ST-012` drafts as `R12.x` client obligations alongside `R12.1`–`R12.9`, not as `R13.x` | Drafting call, recorded |

**Version consequence:** additions plus these scoped amendments are a **MINOR**
bump under the Part II amendment rule — **v1.1.0**.

## Phase 6 review walk — 2026-08-10

Three internal review lenses over the drafted §13 found collisions the
research leaves had not: the descriptive survey documented the field, and the
prescriptive leaf composed against the rules it was pointed at, but neither
read **every** ratified rule against streaming. `P6-D3` turned out to be an
instance of a pattern — *a rule that assumes a status is still available
breaks once the status is committed* — and the pattern had more instances.
The owner ruled all three tiers on 2026-08-10.

### Tier A — RATIFIED: scope all three (ratified MUSTs a conforming streaming API could not satisfy)

| Rule | Collision | Ratified resolution |
| --- | --- | --- |
| `R11.2`, `R11.5` | `429` + `Retry-After` is required on quota exhaustion and is a deliberate tightening of RFC 6585's MAY — but neither status nor header exists mid-stream, and `R13.7`'s frame had no carrier for a pacing hint | Scope both: the `429` obligation binds while a status can still be generated; mid-stream exhaustion reports under `R13.7` carrying a reserved `retry_after` member. This does **not** reopen `P6-D2`, which declined a reserved member for `status` specifically — a pacing hint is not a status code, and RFC 9457 §3.1 constrains only `status` |
| `R2.11` | "A long-running action returns `202 Accepted`" is unscoped, while `R13.1` forbids `202` for a streaming response — both bind the same request | Scope the response-shape clause: `202` where the action does not stream; `200` + stream media type where it does, with `R13.9` binding the channels |
| `R6.1` | "Every collection response MUST return a top-level object… never a bare array" has no non-JSON exception, so a streamed NDJSON collection violates it — the shape the companion recommends | Scope it: a streamed collection carries continuation state on its terminal frame, which preserves what the envelope protects (room for metadata without a breaking change). All other §6 rules bind unchanged |

### Tier B — RATIFIED: document as known gaps, open Phase 7

Five interactions are recognized, unruled, and recorded in a new **§13.4**
register rather than silently omitted, with Phase 7 opened in `PLAN.md` to
rule them under the same evidence discipline: frame-vocabulary versioning
(§9.3) · authorization over a stream's lifetime (§8) · caching posture (§7) ·
idempotency-key replay of a streaming request (§3) · resource ceilings for
streams (§11).

The rejected alternative was ruling all five now. Declined for the reason
Option B was declined at the phase opening: five normative decisions with no
research leaf behind them is exactly the thin-evidence ruling that invites
re-litigation. **The versioning item carries the sharpest known failure** and
is called out in the register: `R9.4` does not classify frame-type names, so
renaming a terminal frame reads as compatible while `R12.10` makes every
deployed client ignore it, see no terminal frame, and report truncation on
every success. The register states the interim posture — treat frame-type
names and terminality as frozen surface.

### Tier C — RATIFIED: absorb all three completions

| Gap | Ratified completion |
| --- | --- |
| `R13.9`'s cross-channel identifier was unnamed, and under `R10.9`'s permitted `url` form there was no identifier at all — so the rule had no referent and two APIs would name it differently | Reserve `operation_id` and `operation_url` in §1.10 as a new **reserved stream members** category; the stream carries whichever matches the `202` body's form |
| `R13.9` required both channels to report "the same terminal state", but `R10.1` has a documented vocabulary and `R13.6` had none — the equality was unverifiable by construction | The terminal frame carries the value from the operation resource's `R10.1` vocabulary wherever `R13.9` binds |
| No rule required a provider to document keep-alive emission, yet `R12.10` binds clients not to depend on keep-alive timing — undischargeable against an undocumented mechanism | `R13.5` gains a disclosure duty: document whether keep-alives are emitted and in what form. **No interval is mandated**, so `ST-016`'s ratified reasoning is untouched |

No new rule identifiers were minted for any of the above: every change scopes
an existing rule or adds a clause to one, so the standard stands at 139 rules.

### Codex second-lens corrections — dated annotation, 2026-08-10

A different-model-family review of the corrected text found defects the three
internal lenses did not. Recorded here rather than silently absorbed, because
two of them correct claims this record itself asserted.

1. **The Tier A `R6.1` resolution was incomplete, and this record said it was
   complete.** The scoping above states that "all other §6 rules bind
   unchanged." Two of them cannot: `R6.2` requires an empty collection to
   carry "an empty items array," which a streamed collection has no place to
   put, and `R6.4` said pagination state "lives only in the body envelope,"
   which the terminal-frame carriage contradicts. **Both are now scoped in
   their own text** — an empty streamed collection is zero item frames plus
   the terminal frame; `R6.4` reads "body representation — the envelope, or
   the terminal frame where the collection is streamed," with its `Link`-header
   prohibition binding streamed collections identically. This was a
   release-blocking defect: without it the standard was unsatisfiable for
   ordinary empty and paginated streamed collections.
2. **The Tier C terminal-state completion created an obligation the `error`
   frame could not discharge.** `R13.9` required the terminal frame to carry
   the operation's terminal-state value, while `R13.7` forbids a `status`
   member on that frame's problem object — so a failure terminal frame had no
   conforming way to carry `failed`. Resolved by reserving **`operation_state`**
   in §1.10 and using it on both success and error terminal frames, so one
   member name carries the comparison in both directions.
3. **`R5.13`'s provenance over-attributed to RFC 9457.** It said an in-band
   object omits `status` "as RFC 9457 §3.1 requires." RFC 9457 permits either
   omitting `status` or setting it to the status actually sent; what it forbids
   is a `status` that disagrees. Omission is **this standard's policy choice**,
   made because `status: 200` on a document describing a failure is accurate
   about the response and misleading about the outcome. `R13.7` had labeled it
   `[POLICY]` correctly; the `R5.13` provenance now does too.
4. **"Registered" and "standardized" were conflated.** `R13.4` forbade
   describing `text/event-stream` as "a registered or standardized media type."
   It is not IANA-registered — that finding stands and was re-verified — but it
   **is** standardized, by the WHATWG HTML Living Standard, which normatively
   defines the format and names the media type. The prohibition is now scoped
   to the IANA claim.
5. **`ST-018`'s browser-authentication conclusion rested on a false premise.**
   This record and the companion stated that `EventSource`'s "only native
   credential is a cookie" and that a browser-direct connection "cannot be
   authenticated under this standard." The Fetch Standard defines credentials
   as cookies, TLS client certificates, **and** HTTP-authentication entries,
   and `withCredentials` sends them. The accurate statement, now carried in the
   companion: an `EventSource` connection cannot use **caller-supplied header**
   credentials under this standard, and must rely on ambient credentials where
   the deployment permits them. `ST-018`'s guidance is unchanged; its
   justification is corrected.
6. **Smaller corrections:** the W3C Recommendation's status is **Retired**
   (2021-01-28), W3C's own term, not "obsolete" · absence of `Content-Length`
   is common for streams but not universal, since RFC 9110 §8.6 has a server
   send it when the length is known in advance · `R13.5` cited `R4.5`
   (identifiers as strings) for the null-versus-absent discipline, which is
   `R4.8` · the release preamble said three rules were scoped when seven were ·
   §1.11's **Reserved name** definition omitted the new stream-member category ·
   Appendix D described `stream_position` as a `components.parameters` entry,
   which cannot express a body or frame member.

Codex confirmed the remaining quotations from RFC 9110, RFC 9112, RFC 9113,
RFC 8895, and the WHATWG SSE text, and all four IANA registration findings,
as accurate.

---

## Re-check trigger — added to the register in `research/README.md`

> **2027-02-10 — streaming standards-footing re-check.** Re-probe the IANA
> media-type registry for `text/event-stream` (per-type URL, against a
> `text/html` control) and re-enumerate the IETF `httpapi` working-group docket
> for any HTTP response-streaming or stream-media-type work item. Either
> change fires a §13 review. `ST-004` is the only ratified principle whose
> classification would change outright, and both probes are cheap and
> unambiguous. **Baseline recorded 2026-08-10:** registry probe returns 404
> against a 200 control; the `httpapi` docket carries three active
> working-group drafts, none on streaming.

## Conditions that would reopen a ratified decision

1. **The `text/event-stream` IANA registration completes** — `ST-004`'s
   disclosure duty and `P6-D4a` both dissolve, and the `Operation-Location`
   parallel dissolves with them.
2. **Any reference ships `application/json-seq` over HTTP** — converts the
   registered option from theoretical to demonstrated and reopens the
   media-type call.
3. **Any reference emits `id:` and honors `Last-Event-ID`** — reopens
   `ST-010`'s mechanism choice.
4. **An IETF working-group item on HTTP response streaming is adopted** —
   changes the authority class available to §13 and could move several
   `[COMPARATIVE]` principles toward `[FACT]`.
5. **A vendor emits `application/problem+json` content inside a stream frame** —
   does not change `P6-D2` (the `status` constraint is a protocol fact, not a
   practice observation) but moves the axis from a unanimous negative to a
   two-sided split and strengthens the evidence base.

**Non-invalidating:** further AI-provider examples of SSE framing, or further
private in-band error schemas. Both findings are already unanimous; additional
instances change no ratified principle.

---

# Phase 8 ratification — 2026-08-12

**All ten `ST-021`–`ST-030` principles ratified en bloc**, following the Gate C
precedent where the `HS`, `AC`, and `OP` batches were each ratified en bloc
after the contested forks had been walked. Sixteen owner rulings shaped the set
first; nothing remained contested at ratification.

**Evidence base.** Six research leaves under `baseline-04` — `04b` frame
versioning, `04c` authorization lifetime, `04d` caching, `04e` idempotency
replay (two runs, `b` controlling), `04f` resource ceilings, `04g`
delivery-ended signalling — totalling roughly 6,700 lines and 230 sources.
Reviewed by one same-family adversarial pass and one Codex second lens, both
dispositioned in
[`docs/reviews/2026-08-10-phase-8-consolidated-proposals.md`](../../docs/reviews/2026-08-10-phase-8-consolidated-proposals.md),
which carries the full post-ruling text and is the drafting source.

**Version class: MINOR — v1.2.0.** Six new rules, six amended, four new
reserved names. Nothing is strengthened: `R3.9`'s change is deliberately split
into an editorial clarification plus a relaxation so that it stays MINOR, and
every other amendment either scopes or relaxes.

> **Correction (2026-08-12) — the version class above is wrong, and so is the
> reserved-name count.** Both were found by a Codex second-lens review of the
> *drafted rule text*, before release. The record is corrected here rather
> than edited, per this layer's no-silent-edit rule.
>
> **The release is MAJOR — v2.0.0.** Five amendments strengthen an existing
> rule, and the Part II amendment rule classes a strengthening as MAJOR
> whatever its size:
>
> | Rule | Obligation absent at v1.1.3 |
> | --- | --- |
> | `R9.4` | Documented open-enum values and their meanings are frozen; renaming one inside a GA major becomes breaking |
> | `R4.9` | Adding a new **terminal** stream frame type is breaking, not a compatible enum addition |
> | `R13.5` | Terminality marked; retired names not reused; terminal types retire only at a major version |
> | `R8.10` | A client-visible JWT's revocation plan states its effect on in-flight streams |
> | `R13.6` | A terminal frame ending delivery while work continues carries `stream_end_reason` |
>
> The decisive test is adopter impact: an API that renamed an open-enum value
> inside a GA major was conformant at v1.1.3 and is not at this release. That
> is the condition the MAJOR class exists to signal. `R3.9`'s split remains
> correct and remains a clarification plus a relaxation; it is **not** one of
> the five.
>
> Two of the five were not in the drafted text at all when this was written.
> **`R8.10`'s clause was ratified here and never enacted** — `ST-025`'s
> "consequential amendment" reached no drafted surface. **`R4.9`'s terminal-frame
> exception did not exist**: Phase 8 classified *renaming* a terminal frame as
> breaking and missed *adding* one, though the informative companion had
> stated the failure plainly. Both are now enacted.
>
> **One new reserved name, not four.** Only `stream_end_reason` was ratified
> and only `stream_end_reason` exists. The figure "four" is an unexplained
> error in the sentence above; no candidate for the other three is recorded
> anywhere in this record, the proposal set, or the leaves, and the integrated
> proposal says "a reserved member" singular.

## The ratified set

| ID | Obligation | Surfaces |
| --- | --- | --- |
| `ST-021` | Documented open-enum values, their meanings, and — for frame types — which are terminal, are frozen within a GA major | Amends `R9.4` |
| `ST-022` | Vocabulary marks terminality; non-terminal types may retire via dual-emit, terminal types only at a major version | Amends `R13.5` |
| `ST-023` | Explicit `Cache-Control`; tier 1 default; tier 3 gated on immutability, `Vary`, and `R7.2` | New `R13.12` |
| `ST-024` | Document a maximum duration or declare the stream unbounded, the latter only where it has no normal end; enforce with a terminal frame | New `R13.13` |
| `ST-025` | A stream should not outlive its credential; should end on revocation; must publish its revocation posture and, when unbounded, its exposure window | New `R13.14` |
| `ST-026` | A keyed repeat never re-executes; answer by attaching or `409`, documented; gaps and expired representations stated | New `R13.15`, amends `R3.9` |
| `ST-027` | Irreversible non-idempotent mutations should not stream; split execution from delivery, or name the work in the request target | New `R13.16` |
| `ST-028` | A held-open stream occupies a concurrency slot for its lifetime; the counting rule is documented | New `R13.17` |
| `ST-029` | A streamed collection documents and enforces its maximum `limit` | Note on `R6.5` |
| `ST-030` | An `error` frame determines the fate of the delivery, not the work; a terminal frame may carry a non-terminal state plus `stream_end_reason` | Amends `R12.10`, `R13.9`, §1.10 |

## Weaknesses ratified with the set, recorded rather than resolved

Three principles carry known weakness. Ratifying them accepts these, and each
is stated in the standard's own provenance lines.

| Principle | Weakness |
| --- | --- |
| `ST-025` | The expiry clause is `SHOULD` because no published incident exists and it overrules the field's clearest precedent — a Kubernetes watch authenticates once and plausibly outlives its bound token. A research trigger is registered below |
| `ST-028` | Its premise depends on reading `R8.10`'s compressed "published: … concurrency separate" as distributive. `baseline-04f` labelled that `[INFERENCE]` and called the ambiguity itself a finding |
| `ST-030` | Its two strongest precedents are not HTTP streaming APIs: Temporal is an RPC framework and A2A a protocol specification. The HTTP-native evidence — Kubernetes' error-with-reason, Kinesis's documented reconnect — is thinner |

## Research trigger, added to the register in `research/README.md`

> **`ST-025` expiry-clause re-check.** Any one of the following fires a leaf
> reconsidering `MUST`: a published incident of data delivered over a stream
> after the principal's authorization was revoked; any implementation shipping
> mid-stream re-evaluation or credential-bound termination; an IETF or OAuth
> work item addressing authorization for a request already in progress; or any
> of OpenAI, Anthropic, or Google Gemini publishing a maximum stream duration
> or an in-flight revocation posture. Baseline recorded 2026-08-12: none
> present, and `baseline-04g` re-confirmed the Anthropic negative.

## Left unruled, by ruling

**Cancellation across the two channels.** `R10.2` expresses cancellation as the
`cancel` action on the operation resource and `R13.9` makes stream and
operation one identity, but nothing says what happens to an open stream when
its operation is cancelled through the other channel, nor whether a client
disconnecting cancels the operation. Registered in §13.4 as of v1.1.3;
deliberately not researched, because no proposal in this set depends on
resolving it.

## Corrections this phase made to released text

Recorded because two of them corrected claims the decision layer itself had
asserted, and one corrected a correction.

- §13.4 said a stream **cannot supply** a strong `ETag`. RFC 9110 §8.8.1
  permits a validator from a revision identifier assigned before the
  representation is accessible. The replacement claim — that `R3.10`'s scope
  excludes streams — was **also wrong**, because `R3.10` binds a *resource*
  and `R13.2` lets that resource serve a streamed representation. Shipped
  corrected in v1.1.3 after two review passes.
- §13.4 said streams have no required concurrency ceiling; `R8.10` already
  requires a published posture including concurrency.
- Appendix A's `R7.3` row asserted an obligation the keyword-free rule does not
  impose.
- The companion advised "add frame types freely", which detonates the same
  false-truncation cascade when the added type is terminal.
- Six interactions were missing from §13.4 entirely; the register went from
  five entries to eleven in v1.1.3.
