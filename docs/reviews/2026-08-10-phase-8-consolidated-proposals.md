# Phase 8 — proposal set for ratification

**Date:** 2026-08-10 · **Revision 3 — integrated** · **Status:** proposals
only. Nothing here binds `rest-api-standard.md` until ratified in a decision
record.

## How to read this document

Each proposal below states its **final normative text after all owner
rulings**. That text is what a ratification walk rules on. Decision history,
declined alternatives, and evidence notes follow each proposal as supporting
material — they explain the text but are not the text.

Revisions 1 and 2 were layered: base proposal text with decision annotations
stacked on top. A different-family review found the result self-contradictory —
one line said all questions were decided while another said two remained open,
and `ST-026` carried a decision adopting one shape above rule text stating
another. Revision 3 integrates instead of annotating, because annotation does
not survive twelve rulings.

**Authority.** The research leaves in `research/reports/` remain authoritative
for evidence. This document is authoritative for the **post-ruling rule text**
and for identifier assignment.

## Evidence base

| Leaf | Topic | Size |
| --- | --- | --- |
| `baseline-04b` | Frame-vocabulary versioning | 1,151 lines, 53 sources |
| `baseline-04c` | Authorization over a stream's lifetime | 803 lines, 23 sources |
| `baseline-04d` | Caching posture for streams | 517 lines, 19 sources |
| `baseline-04e` run a | Idempotency replay — **superseded** | 788 lines, 22 sources |
| `baseline-04e` run b | Idempotency replay — **controlling** | 1,479 lines, 49 sources |
| `baseline-04f` | Resource ceilings for streams | 821 lines, 27 sources |
| `baseline-04g` | Delivery-ended signalling | 1,144 lines, 46 sources |

Run b supersedes run a's central claim. Both are retained: divergence between
independent runs of one prompt is a confidence signal.

## Identifier assignment

`R1.3` freezes `ST-001`–`ST-020`, so Phase 8 principles begin at `ST-021`.
Rule identifiers continue from `R13.11`, the current end of §13.

| Principle | Rule target | Source |
| --- | --- | --- |
| `ST-021` | Amends `R9.4`; retains a terminality clause | `04b` |
| `ST-022` | Amends `R13.5` | `04b` |
| `ST-023` | New `R13.12` | `04d` |
| `ST-024` | New `R13.13` | `04c` + `04f` merged |
| `ST-025` | New `R13.14` | `04c` |
| `ST-026` | New `R13.15`; amends `R3.9` | `04e` run b |
| `ST-027` | New `R13.16`, at `SHOULD` | `04e` run b |
| `ST-028` | New `R13.17` | `04f` |
| `ST-029` | Scoping note on `R6.5`; no rule text change | `04f` |
| `ST-030` | Amends `R12.10` and `R13.9`; reserves `stream_end_reason` in §1.10 | `04g` |

Ratifying any of these adds its identifier to `R1.3`'s frozen-series list, as
`ST-001`–`ST-020` were added in version 1.1.0.

---

# The proposals

## `ST-021` — Open-enum values are frozen

**Target:** amend `R9.4`, the compatibility taxonomy in §9.3.
**Class:** `[COMPARATIVE]` on rename and meaning; `[POLICY]` on terminality.
**Confidence:** moderate-high; terminality moderate.

Add to **Frozen**: documented open-enum values, their meanings, and — for
stream frame types — which of them are terminal.

Add to **Compatible**: adding a value to an enumeration documented as
growable.

Add to **Breaking**: removing or renaming a documented open-enum value, and
changing what a documented value means with or without a change of name.

### Why the class rather than the instance

`R9.4` classifies *adding* an enum value and classifies *renaming a frozen
element*, but an enum value is not a field name and is not on the frozen list,
so removing or renaming one was unclassified. Three enumerations in the
standard have that hole: stream frame types (`R13.5`), `operation_state`
(`R10.1`, made cross-channel by `R13.9`), and `R6.7`'s sortable-field set.

The precedent is inside `R9.4` itself: its frozen list already contains
"problem `type`/`code` pairs (R5.13.2)", and a problem `code` is an enum-like
value rather than a field name. Freezing enum values is established here and
was never generalized.

### The two clauses that are not generic

**Terminality** cannot be expressed by any generic enum rule, because no other
enumeration has a notion of "this value ends the response." Demoting a frame
type from terminal without renaming it produces the same client-visible
failure as renaming it — under `R12.10` the client no longer recognizes a
terminal frame and reports truncation on success. No source states this; it is
this standard's own construction, labeled `[POLICY]`.

**Meaning changes** are retained explicitly from `baseline-04b`, which
proposed "changing what a documented frame type means, with or without a
change of name." Generalizing to values must not lose it, or the guarantee
disappears when the frame-specific entry is absorbed.

### Field evidence, with its limits stated

Google Gemini renamed six Interactions event types including the terminal
`interaction.complete`, published as a breaking change behind a dated version
header with a June 2026 sunset. **The leaf classifies that rename as pre-GA**,
which `R9.4` already permits, since `R9.4` opens "Within a GA major version."
So it is not a GA-freeze precedent, and revision 1 presented it as one by
omitting the classification.

Two further limits: the mechanism Gemini used is a dated *request header*,
which is not a version mechanism this standard has — `R9.1` requires the major
version in the path. And the OpenAI counterexample, an unannounced rename on a
generally-available surface, dates to an issue opened 2025-05-23 and has since
been resolved by the replacement event shipping.

The gap the proposal closes is nonetheless real: `R9.4` reaches frame types
only through "reserved-name semantics (§1.10)", and §1.10 registers exactly
one frame type while stating that an API's own vocabulary "is otherwise its
own."

### Declined

A frame-type-only entry, which patches one instance and leaves
`operation_state` and the sortable-field set unclassified. Recorded cost of
the class fix: it generalizes past `baseline-04b`'s evidence, mitigated
because it classifies a change `R9.4`'s own logic already implies rather than
creating a new obligation.

---

## `ST-022` — Vocabulary documentation and retirement

**Target:** amend `R13.5`. **Class:** `[POLICY]` on the marking duty,
`[COMPARATIVE]` on the retirement path. **Confidence:** moderate-high.

The vocabulary documentation marks, per frame type, whether it is terminal. A
retired frame-type name is never reused for a different meaning.

**Non-terminal** frame types may be retired either at a new major version or
through a documented dual-emit overlap period stated in the vocabulary
documentation, retiring the old name only at a major version.

**Terminal** frame types may be retired only at a new major version. Dual-emit
is unavailable for them.

### Why terminal types are excluded from dual-emit

`R13.6` requires a stream to end with *the* terminal frame, and §1.11 defines
that as "the frame that ends a stream." Two concurrent typed, payload-carrying
terminal frames are not expressible: the first would not have ended the
stream, and a `R12.10` client recognizing the old name stops reading at it and
never sees the replacement. Where `R13.9` also binds, nothing would say which
of the two carries `operation_state`.

Every dual-emit precedent the leaf cites — GitHub, CloudEvents, Standard
Webhooks — is a webhook system, where two deliveries are two independent
messages and nothing declares the conversation over. A stream has exactly one
ending. The one real terminal rename in the field used a version gate, so the
exclusion matches observed practice while the alternative would license
something nobody has shipped.

**Recorded cost:** terminal-frame renames require a major version. Accepted as
correct — a terminal frame is the one frame every client must recognize.

### Why the overlap window is documented rather than signalled

`R9.5`'s `Deprecation` and `Sunset` headers are response headers. A stream's
headers are sent once, before any frame, so they cannot deprecate one frame
type within a stream. The header channel is unavailable, which is a protocol
fact; the documentation remedy is policy.

### Declined

Amending `R13.6` to define an overlap ending. It would weaken the
complete-versus-truncated guarantee that `R12.10` depends on, making a client
that stops at the old terminal frame sometimes right and sometimes early.

---

## `ST-023` — Caching posture for a stream

**Target:** new `R13.12`. **Class:** `[POLICY]` on the tier choice, resting on
protocol facts per clause. **Confidence:** moderate-high.

A streaming response carries an explicit `Cache-Control`. `R7.1` is not
relaxed by incremental delivery: the header section is transmitted before the
body exists, and the directive describes policy rather than bytes.

Within `R7.3`'s posture a stream takes **tier 1, `private, no-cache`**, by
default, and **tier 1's strong-`ETag` revalidation clause does not apply to
it** — `R3.10` binds resources supporting conditional update, and a stream is
not one.

**Tier 2, `no-store`,** applies on the same condition as for any other
response, a genuinely sensitive payload, and MUST NOT be adopted merely
because the response is a stream.

**Tier 3, `public` with `max-age`,** MAY be used only where **all three** hold:
the stream is a view over an immutable retained artifact; every input
selecting a resumption point is in the request URI or named in `Vary`; and
`R7.2` is satisfied, meaning the response carries no user-specific or
authenticated data. A stream unbounded by design MUST NOT use tier 3.

### The `R7.2` condition, and why it is explicit

Tier 3 means `Cache-Control: public`, which is precisely the directive that
re-enables shared-cache storage for a request carrying `Authorization` under
RFC 9111 §3. Subsequent requests would then be answered without the origin
running `R8.6`'s per-object authorization check, and `ST-025`'s revocation
posture would bound nothing, because the cache does not know about the
revocation. `R7.2` independently forbids this; naming it here stops a drafter
working from two gates and omitting the third.

### The `Vary` hazard

SSE resumption travels in the `Last-Event-ID` **request header**, which a
cache key does not include unless `Vary` names it. A cache could otherwise
answer a resume-from-position-N request with a stored stream beginning at
position 1. RFC 9111 §3.3 confines this to **completed** stored responses,
which is exactly tier 3's population, so the gate sits in the right place.
Drafting caution: RFC 9111 never mentions `Last-Event-ID`; the applicable text
is the general secondary-cache-key rule in §4.1.

### Two premises this leaf disproved, both in released text

`R7.3` contains **no BCP 14 keyword at all**, so Appendix E's `private,
no-store` was never a conflict — only an unpicked default. And §13.4's claim
that a stream cannot supply a strong validator is too strong: RFC 9110 §8.8.1
permits a validator based on "a unique node name and revision identifier being
assigned before the representation is made accessible to GET." `R3.10`'s scope
is the real blocker.

### Declined

Mandating `no-store` on streams. Zero of nine surveyed streaming
implementations send `no-store` alone; all five that emit a header include
`no-cache`.

---

## `ST-024` — Stream duration bound

**Target:** new `R13.13`. **Merges** `04f`'s duration proposal with `04c`'s
first clause. **Class:** `[POLICY]` on the requirement, `[COMPARATIVE]` on
publish-your-number practice. **Confidence:** moderate-high.

For each streaming endpoint, an API documents either the maximum duration it
will hold the response open, or that the stream is **unbounded by design**.
The unbounded declaration is the one `R13.6` already requires, and an API that
has made it owes nothing further under this rule.

An API may declare a stream unbounded only where it meets `R13.6`'s test —
the stream **has no normal end**, as a watch or an event tail does. A
generative completion has a normal end and cannot declare itself unbounded.

Where a maximum is documented, the server enforces it and ends the stream at
the maximum with `R13.6`'s terminal frame rather than by closing the
connection, because a close at a published limit is a normal end and a client
must not be made to report it as truncation.

**Resolved by `ST-030`.** That terminal frame carries the operation's current
state — permitted once `R13.9` is scoped — plus a `stream_end_reason` naming
why delivery ended.

### How the merge resolved

`04c` proposed a flat "MUST publish a maximum stream duration"; `04f` proposed
"document a maximum **or** declare the stream unbounded." The second governs,
because the first would make every watch and event-tail API nonconformant, and
`R13.6` already provides for unbounded streams explicitly. Taking the stricter
text would have contradicted a rule ratified in version 1.1.0.

The gating clause restores what the merge otherwise lost: `04c`'s flat form
made a bound unavoidable, and the unbounded escape hatch could otherwise be
claimed by a stream that plainly has a normal end.

**No number is set.** The surveyed spread runs four orders of magnitude, which
is the evidence that no number is standardizable.

### Moved to the companion

The randomize-the-deadline advice — that a documented maximum should be
jittered per connection so streams opened together do not expire together —
becomes informative guidance in `streaming-profile.md` rather than rule text.
The finding is real, and the two implementations behind it (Kubernetes'
`[1.0, 2.0)` factor, Consul's `wait / 16`) are the most-studied long-poll
systems in the research, but two implementations is below the bar this phase
applied to every other `SHOULD`, and the leaf volunteered as much. Dropping it
entirely was declined: a reader who never meets the idea builds the fixed
deadline and finds the herd in production.

---

## `ST-025` — Authorization over a stream's lifetime

**Target:** new `R13.14`. **Class:** `[POLICY]` throughout, grounded in
RFC 9068 §4 for the expiry clause. **Confidence:** moderate-high on the
disclosure clauses, moderate on expiry.

A stream **SHOULD NOT** continue past the expiry of the credential that
authorized it. Where the credential carries or implies an expiry, the server
should end the stream at or before it rather than relying on the maximum
duration to bound it.

An API **SHOULD** end an in-flight stream when the authorization that
permitted it is revoked, and **MUST** document its revocation posture as an
upper bound on how long a stream may continue after revocation — never as an
assertion that revocation is immediate. An API that does not terminate on
revocation at all MUST say so.

An API whose stream is **unbounded by design** MUST state its exposure window
explicitly, including when that statement is "indefinite." **This binds
whatever the credential does** — an unbounded stream that terminates on
credential expiry states that it does.

Resumption (`R13.10`) is a new request and is authorized as one (`R8.6`); a
resumption position is never evidence of authorization.

### Why expiry is `SHOULD` and not `MUST`

Ruled by the owner. The clause overrules the field's clearest precedent — a
Kubernetes watch authenticates once and plausibly outlives its bound token —
on a threat argued from mechanism with **no recorded incident**. A `MUST`
there is the thin-evidence shape the Phase 6 decision record declined Tier B
items to avoid.

**Research trigger, registered with the dated re-check register when this is
ratified.** Any of the following fires a leaf reconsidering `MUST`, each
chosen because it removes one of the specific weaknesses behind the demotion:

| Trigger | What it changes |
| --- | --- |
| A published incident of data delivered over a stream after revocation | Converts the threat from argued-from-mechanism to observed |
| Any implementation shipping mid-stream re-evaluation or credential-bound termination | Removes the overrules-the-precedent objection |
| An IETF or OAuth work item on authorization for a request already in progress | Supplies authority RFC 9700, RFC 9068, and RFC 7009 currently lack |
| Any of OpenAI, Anthropic, or Google Gemini publishing a maximum duration or in-flight revocation posture | Breaks the three-for-three vendor silence |

### The verified negatives behind this proposal

RFC 9700, RFC 9068, and RFC 7009 contain no occurrence of "streaming" or
"long-lived" and say nothing about a request already in progress. All three
deep-dive providers publish no maximum stream duration and no in-flight
revocation behavior. Where the field converged, it converged on *bounded
lifetime, re-authorize on reconnect* — Kubernetes at 1800–3600 s, AWS Kinesis
at a hard 5 minutes, Microsoft Entra coauthoring at 1 hour.

**A supporting citation revision 2 missed:** RFC 7009 §2.1 is not silent on
revocation timing — "In practice, there could be a propagation delay…
Implementations should minimize that window." That supports the
upper-bound-not-immediacy clause directly.

### Consequential amendment

`R8.10`'s token-format axis requires a revocation-propagation plan; it gains a
clause requiring that plan to state its effect on in-flight streams.

**Why the statement binds on the unbounded declaration rather than on
credential type.** It originally covered only non-expiring credentials,
because expiry was then a `MUST` and the credential genuinely supplied the
bound. Once expiry became a `SHOULD`, an unbounded stream on an *expiring*
credential reached the same exposure by a second route: `ST-024` is discharged
by the unbounded declaration, and both termination clauses are `SHOULD`s, so
nothing required the server to stop and nothing required it to say so.

The trigger that matters is therefore **unbounded**, not what kind of
credential is presented. An unbounded stream is the one shape where nothing
else in the set supplies a bound.

Two narrower forms were declined. Binding the statement to the `R1.7`
deviation record — required only of an API that explicitly declines the expiry
`SHOULD` — is well-aimed but under-inclusive, catching the API that considered
expiry and missing the one that never did. Doing both adds a second obligation
the first already covers.

**Recorded cost:** an unbounded stream that *does* terminate on credential
expiry must still say so. That is one sentence, and it is the sentence a
client needs in order to plan.

---

## `ST-026` — A keyed repeat never re-executes

**Target:** new `R13.15`, plus an amendment to `R3.9`. **Class:** `[POLICY]`.
**Confidence:** moderate-high on non-re-execution; moderate on response shape.

A repeat of a streaming request carrying the same idempotency key **MUST NOT
re-execute** the work.

The server answers by the original execution's state, and **documents which of
the two permitted shapes it implements**:

- **serve the in-flight work** — attach the caller to the running stream; or
- **reject with `409`** while the original is still executing.

Once the original reaches a terminal state, the server delivers the recorded
outcome. Where the replayable representation has expired but the terminal
state is retained, delivering the terminal state discharges this rule,
**provided the response states that the representation is no longer
available** — carried in `stream_end_reason` (§1.10) where the reply is itself
a stream, and in a documented member where it is not. The API documents its
**representation-retention window separately** from `R3.9`'s key-retention
floor.

Where a delivered stream omits frames the client would otherwise have
received, the server **MUST** make the omission visible rather than delivering
a terminal frame as though the stream were complete. Run b permits three
mechanisms: delivering the retained frames, ending with a defined `error`
frame, or carrying a position on every frame whose numbering makes an omission
visible.

**A client MUST NOT infer, from the presence of a terminal frame alone, that
it received every frame of the original stream.**

### `R3.9` amendment

"The stored response" for a stream means its terminal state together with
whatever representation of the result the API documents as replayable — not
the original frame sequence.

`R3.9`'s exception is amended in **two parts**, deliberately separated so each
carries its own version class.

**Part one — clarification, editorial.** "Naturally idempotent operations (PUT
with a client-supplied ID)" covers a `PUT` that **stores a representation**,
not one that **starts work**. A `PUT` that starts work was never naturally
idempotent, because a second one runs the work again. Such an operation was
therefore never inside the exception, so stating it strengthens nothing — the
drafting misled, the rule did not change.

**Part two — a narrow header exemption, a relaxation, MINOR.** Where the
**request target itself names the work**, the `Idempotency-Key` header is not
required, because the URI supplies the deduplication key. Every guarantee still
applies: the operation MUST NOT re-execute, and MUST reject a repeat whose
payload differs.

### Why the change is split

Applying part one alone would bind a work-starting `PUT` to all of `R3.9`,
including "MUST accept an idempotency key… carried in the `Idempotency-Key`
request header." For the one shipped implementation — Cog's
`PUT /predictions/<prediction_id>` — that means **two deduplication keys for
one purpose**, the same-concept-different-name divergence §1.10 exists to
prevent. Cog carries no such header and needs none.

The split also resolves the version question rather than arguing it. A
different-family review contended that the unsplit form cannot be editorial,
because adding the header obligation to previously-exempt operations is a
strengthening and the amendment rule puts strengthening at MAJOR. Part one is
editorial because nothing was ever exempt; part two is a relaxation, which the
amendment rule puts at MINOR. Editorial plus relaxation resolves to MINOR
overall.

The design argument governs independently of the version one: requiring a
second key from an API that already has one in its URI is a defect whatever it
costs to ship.

### Why the mandated `409` did not survive

Run a claimed a universal negative — that no published API both accepts an
idempotency key and streams — and called it the load-bearing claim. It is
false. **Replicate's Cog** documents both on one endpoint: a capability table
row reads `PUT /predictions/<prediction_id>` with `Accept: text/event-stream`
→ "Streaming, idempotent", and on a repeat while running the server "returns a
stream for the existing prediction instead of creating a duplicate
prediction." Pinned source: `replicate/cog` commit
`966752e9f5f5c165fc5e9618642fd353f0db0e56`, `docs/http.md`; official page at
`https://cog.run/http/`.

Run a also **escalated the expired IETF Idempotency-Key draft's `409` from
`SHOULD` to `MUST`** — an error independent of the falsified negative. Two
defects produced one wrong rule.

Run b found **one** shipped implementation of the intersection and one
specification-level near-miss (A2A Protocol v1.0.0, deduplication at `MAY`)
across roughly forty candidates in three parallel sweeps. It recorded a false
falsifier as a methodology warning: Bedrock AgentCore's `idempotencyToken` is
an SDK auto-fill marker, not deduplication.

### Why a preference between the two shapes was declined

Attach has two sources (Cog, Temporal `USE_EXISTING`); reject has three
(Stripe, Shopify, the draft at `SHOULD`). A `SHOULD` either way would be this
standard's opinion presented as evidence. Leaving the shape undocumented was
also declined: a client would then have to handle any behavior at all.

### Why the retention mismatch resolves to "terminal state suffices"

`R3.9` requires the key remembered at least 24 hours; the representations run
b measured last far less — Cog's buffer is **1024 events**, a count rather
than a duration and configurable to zero; OpenAI's about ten minutes;
Kubernetes' about five. So for roughly 23 of the 24 hours the key is known,
the terminal state is known, and the frames are gone.

This deliberately does **not** mirror `R13.10`. A resumption request asks for
frames and without them is unanswerable, so `R13.10`'s defined error is
honest. A keyed repeat asks whether the work already ran, which the terminal
state answers — the frames were never the point, not re-executing was.
Copying `R13.10`'s error would return a failure when the server knows the
answer, pushing the client toward re-running work that already succeeded.

> **Open — response shape for the expired-representation case.** A terminal
> state such as `succeeded` does not necessarily carry the application
> *result*. The rule needs an exact shape — for example, state-only replay
> with an explicit result-expired indication — before "the recorded outcome"
> is accurate. See "Known open items."

### Composition with resumption

A request may carry both an `Idempotency-Key` and a `stream_position`. They
compose: the **key governs execution** — the work is not run again — and the
**position governs delivery** — the server resumes from it, or returns
`R13.10`'s defined error where the position is outside the retention window.

This resolves the apparent conflict by construction. Replaying from the first
frame while a position is present is exactly the silent restart `R13.10`
forbids, so that option does not arise.

---

## `ST-027` — Prefer separating execution from delivery

**Target:** new `R13.16`, at `SHOULD`. **Class:** `[COMPARATIVE]`.
**Confidence:** moderate-high.

An API **SHOULD NOT** stream the response to a non-idempotent mutation whose
repeated execution has an external effect that cannot be reversed — a payment,
a disbursement, a message send, a metered charge.

Where such a capability needs incremental delivery, the API SHOULD use one of
two structures:

1. **Split execution from delivery.** The mutation is a non-streaming request
   returning an operation resource (`R10.9`, `R13.9`), and the incremental
   delivery is a **safe** request over that resource, resumable under
   `R13.10`. This is OpenAI's shipped `background: true` design.
2. **Name the work in the request target.** The client supplies the work's
   identifier in the URI, so the request target itself is the deduplication
   key. This is Replicate Cog's `PUT /predictions/<prediction_id>` design.

An API that does stream such a mutation MUST comply with `ST-026` and MUST
state in its conformance note (`R1.7`) which of `ST-026`'s permitted shapes it
implements.

### Note on the second structure

Revision 2 omitted it. It is consequential because `ST-026`'s `R3.9`
clarification is about exactly that shape — a `PUT` naming client-supplied
work — so the proposal and the clarification must agree about whether it is
recommended.

### Declined

A flat prohibition, assessed against AIP-151 and declined: AIP-151 binds
long-running-operation methods only, and a ban would forbid the pattern §13
exists to serve.

---

## `ST-028` — A held-open stream occupies a concurrency slot

**Target:** new `R13.17`. **Class:** `[POLICY]`, clarifying an already-ratified
axis. **Confidence:** moderate on the strength, high that the clarification is
needed.

A held-open stream occupies one unit of the concurrency dimension of
`R8.10`'s rate-limit axis for the whole time the server holds it open, not
only at the moment the request arrives. An API **MUST** document how streams
are counted against its published concurrency posture. Where a request is
rejected because the concurrency posture rather than the rate posture is
exhausted, the `429` required by `R11.2` SHOULD distinguish the two conditions
in its problem `code`.

### This is a clarification, not a new ceiling

`R8.10`'s rate-limit axis default already requires a published posture
including concurrency, and `R8.10` makes adopting the axis defaults a MUST.
The residual gap is only that nothing says a stream holds the slot for its
lifetime.

**Stated limit on that reading:** the leaf labeled it `[INFERENCE]`, because
the obligation depends on reading `R8.10`'s "published: … concurrency
separate" as distributing across the whole list, and the sentence is
compressed enough that a reviewer could read "published" as governing only the
first clause. The leaf called that ambiguity itself a finding. **The
ratification walk should disambiguate `R8.10`'s cell in the same release**
rather than rely on the reading.

---

## `ST-029` — Streamed collections owe their documented maximum

**Target:** a scoping note on `R6.5`, or one sentence in §13. No rule-text
change. **Class:** `[POLICY]`. **Confidence:** high.

A streamed collection documents and enforces its maximum `limit` exactly as an
unstreamed one does.

`R6.5` already binds it — the rule is unqualified, and `R6.4`'s version 1.1.0
widening to "the terminal frame where the collection is streamed" is the
precedent that §6 reaches streamed collections. The standard has never said
so, which is the whole content of this proposal: it closes a silence, not a
gap.

---

## `ST-030` — Delivery and work are reported separately

**Target:** amends `R12.10`, amends `R13.9`, and adds a reserved member in
§1.10. **Class:** `[COMPARATIVE]` on the shape, `[POLICY]` on the reason
member. **Confidence:** moderate-high on the shape; moderate on the
HTTP-nativeness of the precedent. **Version class:** MINOR — the same
relaxation class `ST-024` already accepted.

An **`error` frame determines the fate of the delivery, not of the work.**
`R12.10` is scoped accordingly: a client receiving an `error` frame treats the
*stream* as ended in failure, and consults the operation resource — already
authoritative and already reachable via `operation_id` or `operation_url`
(`R13.9`) — for the fate of the operation.

A **terminal frame may carry the operation's current state**, including a
non-terminal one, rather than only a terminal-state value. `R13.9` is scoped
accordingly.

A terminal frame that ends delivery while the work continues **MUST** carry a
documented **`stream_end_reason`** (§1.10) naming why delivery ended. The
standard reserves the member name and **standardizes no value set**; each API
documents its own vocabulary.

### Why this shape, and not a new terminal frame type

`baseline-04g` built a six-pattern taxonomy of how the field signals
stream-end-without-work-end. The decisive evidence is that **the one protocol
that shipped a standalone end-of-stream marker removed it.** A2A carried
`final: true` on `TaskStatusUpdateEvent` in v0.3.0 and deleted it in v1.0.1.
The issue, verified directly at `a2aproject/A2A` #1308, is titled "Remove
redundant `final` field from `TaskStatusUpdateEvent`" and reasons:

> Terminal states (COMPLETED, FAILED, CANCELLED, REJECTED) already indicate
> task completion, making the final field redundant. This creates consistency
> across streaming, polling, and push notification communication patterns.

Adding a frame type would therefore re-adopt a design a peer protocol
reversed, for the reason that the state vocabulary already carries the
information.

Where the field converged instead is the shape proposed here: Temporal returns
the most advanced stage reached plus a continuation handle **on a success
response**, and A2A v1.0.1 closes the stream when the task reaches "a terminal
**or interrupted** state" — one vocabulary spanning both.

### Why not reuse the error channel with a distinguishing reason

That pattern is shipped — Kubernetes watch signals `410 Expired` inside an
`ERROR` event, and `client-go` logs "Watch closed" for `Expired` against
"Failed to watch" otherwise — and it is the most **HTTP-native** precedent
available. It was declined because every client would then have to read inside
an error to avoid treating a normal end as a failure, while `R12.10`'s plain
text still says the operation failed. Scoping `R12.10` fixes the text rather
than working around it.

### This reverses the earlier repair for the duration-cap collision

An earlier ruling exempted the terminal frame from carrying `operation_state`
in the duration-cap case. `baseline-04g` recommends reversing that, and the
reasoning holds: the objection to carrying the current state was that it
"weakens the comparison in every case," and that does not reach a relaxation
**conditioned on the divergence case**. Exempting removes information;
carrying the current state adds it. `ST-030` supersedes that repair.

### What it unblocks

| Was blocked | Now |
| --- | --- |
| `ST-024`'s enforcement clause named no frame type, member, or value | The terminal frame carries the current state plus `stream_end_reason` |
| `ST-025` ends an expired-credential stream with an `error` frame that `R12.10` forced clients to read as operation failure | The error frame ends the *delivery*; the operation resource reports the work |
| `ST-026`'s gap-visibility clause permits three mechanisms, only one an error | All three remain available, and none is forced to imply the work failed |

### The honest weakness, recorded

The two strongest precedents are **not HTTP streaming APIs**: Temporal is an
RPC framework and A2A is a protocol specification. Gemini Live's advance-warning
pattern is WebSocket, which §1.2 places outside this standard. The HTTP-native
evidence is thinner — Kubernetes' error-with-reason and Kinesis's silent close
plus documented reconnect. Whether an RPC-shaped precedent should drive an HTTP
rule is a fair objection and was weighed at the ruling.

---

# Known open items

| Item | What is unresolved |
| --- | --- |
| **Cancellation** | Registered for §13.4 by owner ruling; not researched. Nothing says what happens to an open stream when its operation is cancelled through the other channel, nor whether disconnecting cancels the operation |

---

# Register corrections

Four claims in the released §13.4 register overstate their own gaps. The walk
should correct them regardless of which proposals it adopts.

| Register claim | What the research established |
| --- | --- |
| A stream cannot supply a strong `ETag` | RFC 9110 §8.8.1 permits a pre-transmission revision validator. `R3.10`'s scope is the actual blocker |
| `R7.3` conflicts with streaming | `R7.3` carries no BCP 14 keyword, so it never bound anything |
| Streams have no required concurrency ceiling | Probably already required by `R8.10`, on a reading the leaf flagged as an inference. Disambiguate `R8.10` rather than rely on it |
| *(revision 1 also listed a row about streamed collections lacking a maximum)* | Withdrawn as a strawman — §13.4 makes no such claim. The substantive point survives as `ST-029` |

# Defects outside Phase 8's scope

Both predate streaming and are independent of every proposal.

1. **Appendix A's checklist row for `R7.3` states an obligation the rule does
   not impose.** `R7.3` has no BCP 14 keyword; its row reads as a requirement.
2. **`R9.4` did not classify open-enum removal or renaming at all.** `ST-021`
   now fixes this class-wide, so this defect is absorbed rather than left open.

# Withdrawn from the earlier defect list

**Missing `R5.16` catalog entries were listed as a ratification blocker. They
are not.** `R5.16` says "An API MUST publish a catalog of every problem
`type`/`code` pair **it** can return" — a per-API obligation. A proposal may
require a defined problem condition whose API-specific pair appears in that
API's catalog. Standard-owned universal pairs would be a separate design
choice, not a prerequisite for satisfiability.

---

# Decision index

Twelve rulings, taken one at a time before the ratification walk so the walk
inherits settled ground.

| # | Question | Ruling |
| --- | --- | --- |
| 1 | Ship the editorial corrections now or hold? | Ship as `v1.1.3` |
| 2 | Fix the open-enum hole at the class or the instance? | The class |
| 3 | Can dual-emit retire a terminal frame type? | No — major version only |
| 4 | `ST-026`'s shape after its evidence was falsified | Non-re-execution `MUST`; response shape a documented choice |
| 5 | Must an attached stream reveal missed frames? | Yes — server obligation, with the client clause restored |
| 6 | `R3.9`'s "naturally idempotent" exception | Clarify the premise — form settled by ruling 14 |
| 7 | A stream cut at its cap has no terminal state | Exempt and signal — **superseded by ruling 13** |
| 8 | The unbounded-plus-non-expiring escape hatch | Gate the claim, require the statement |
| 9 | A request carrying both a key and a position | They compose |
| 10 | `ST-024`'s jitter clause | Move to the companion |
| 11 | A different-family review before the walk? | Run it |
| 12 | The retention mismatch | Terminal state suffices; exact shape remains open |
| 13 | The delivery-ended gap | Scope `R12.10` and `R13.9`; the terminal frame carries the operation's current state plus `stream_end_reason` (`ST-030`). Supersedes ruling 7 |
| 14 | `R3.9` exception form and version class | Split it — clarification is editorial, the header exemption for URI-named work is a relaxation. Resolves to MINOR |
| 15 | `ST-025`'s exposure statement scope | Binds on the unbounded declaration, whatever the credential does |
| 16 | `ST-026`'s expired-representation shape | Terminal state **plus** an explicit statement that the representation is unavailable — `stream_end_reason` where the reply is a stream, a documented member where it is not |

Two further rulings stand: `ST-025`'s expiry clause at `SHOULD` with a
research trigger, and cancellation registered now and researched later.

# What the ratification walk decides

Each of `ST-021` through `ST-029` is a separate ruling. Beyond adopting or
declining each:

1. Whether the delivery-ended gap is resolved in this release or deferred,
   once `baseline-04g` reports.
2. `R3.9`'s exception form and its version classification.
3. Whether `R8.10`'s concurrency cell is disambiguated in the same release.
4. Whether the register corrections and out-of-scope defects ship together
   with the rule proposals or separately.
