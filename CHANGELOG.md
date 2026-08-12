# Changelog

Versioning follows the Part II amendment rule of
[`rest-api-standard.md`](rest-api-standard.md): editorial changes bump
patch; added rules, appendices, or relaxations bump minor; strengthened,
removed, or re-meant rules bump major. Every rule change is atomic across
the rule text, its decision record, its Part II row, its checklist row,
and the worked example.

## 2.0.0 — 2026-08-12

Completes **Phase 8**, the streaming interactions §13.4 recorded as recognized
and not yet ruled. Six rules added, nine amended, one reserved member added.
The standard now carries **145 rules** (from 139); checklist 145/145.

### A MAJOR bump, and why

**Five amendments strengthen an existing rule.** The Part II amendment rule
classes a strengthening as MAJOR whatever its size, and the test that decides
it is adopter impact: **an API that renamed a documented open-enum value
inside a GA major was conformant at 1.1.3 and is not conformant here.**

| Rule | Obligation absent at 1.1.3 |
| --- | --- |
| `R9.4` | Documented open-enum values and their meanings are frozen; renaming one becomes breaking |
| `R4.9` | Adding a new **terminal** stream frame type is breaking, not a compatible enum addition |
| `R13.5` | Terminality marked; retired names not reused; terminal types retire only at a major version |
| `R8.10` | A client-visible JWT's revocation plan states its effect on in-flight streams |
| `R13.6` | A terminal frame ending delivery while the work continues carries `stream_end_reason` |

This was drafted as 1.2.0 claiming "nothing strengthened" and reclassified
before release, after a second-lens review of the drafted rule text. That
review also found that two of the five were **not in the draft at all**:
`R8.10`'s clause had been ratified and never enacted, and `R4.9`'s exception
had never been written — Phase 8 classified *renaming* a terminal frame as
breaking and missed *adding* one, the same defect one row over, which the
informative companion had been stating plainly since 1.1.0. Both are enacted
here. The reasoning is recorded as a dated correction in
`research/decisions/baseline-04-streaming.decision.md`.

**What an adopter must re-check:** whether any documented open enum has had a
value renamed, removed, or re-meant within the current major; whether any
stream frame vocabulary marks terminality; whether a new terminal frame type
was ever added within a major; whether a client-visible JWT's revocation plan
addresses in-flight streams; and whether terminal frames that end delivery
early carry a reason.

### Added

- **`R13.12`** — a streaming response carries explicit `Cache-Control` and
  defaults to tier 1. Tier 3 requires an immutable retained artifact, `Vary`
  coverage of every resumption input, and no authenticated data.
- **`R13.13`** — document a maximum stream duration, or declare the stream
  unbounded, which is available only where it genuinely has no normal end. A
  documented maximum is enforced and ends with a terminal frame, not a
  connection close.
- **`R13.14`** — a stream should not outlive its credential, should end on
  revocation, must publish its revocation posture as an upper bound, and when
  unbounded must state its exposure window.
- **`R13.15`** — a keyed repeat never re-executes; the server documents
  whether it attaches or rejects with `409`; omitted frames are made visible;
  an expired representation is stated as unavailable.
- **`R13.16`** — irreversible non-idempotent mutations should not stream;
  split execution from delivery, or name the work in the request target.
- **`R13.17`** — a held-open stream occupies a concurrency slot for its
  lifetime, and the counting rule is documented.
- **`stream_end_reason`** reserved in §1.10, with no standardized value set.

### Changed

- **`R9.4`** — documented open-enum values, their meanings, and frame-type
  terminality join the frozen surface. Removing, renaming, or re-meaning one
  is breaking. This fixes a class, not an instance: `operation_state` and
  `R6.7`'s sortable-field set had the same hole.
- **`R13.5`** — the vocabulary marks which types are terminal. Non-terminal
  types may retire through a documented dual-emit overlap; **terminal types
  only at a major version**, because a stream has exactly one ending.
- **`R12.10`** and **`R13.9`** — an `error` frame now ends the **delivery**,
  not the work, and a terminal frame may carry a non-terminal
  `operation_state` paired with `stream_end_reason`. Three rules end delivery
  while work continues, and the previous vocabulary could not express it.
- **`R3.9`** — split deliberately. The exception's premise is *clarified*, so
  that "naturally idempotent" covers a `PUT` storing a representation and not
  one starting work; and a *header exemption* is added where the request
  target names the work. Clarification plus relaxation — **not** one of the
  five strengthenings; it would have been MINOR standing alone.
- **`R4.9`** — adding a new **terminal** stream frame type is breaking. Every
  other enum addition is safe because R12.4 makes clients ignore what they do
  not recognize; a terminal type is the one addition that tolerance cannot
  absorb, because R12.10 then requires the client to report truncation on
  every success.
- **`R8.10`** — the token-format axis's revocation-propagation plan must state
  its effect on in-flight streams. Ratified with `ST-025` and, until this
  release, never enacted.
- **`R13.6`** — a terminal frame that ends delivery while the underlying work
  continues carries `stream_end_reason`. This binds **every** stream; the
  draft had scoped it to streams paired with an operation resource, which left
  a duration- or credential-ended stream with no reason signal at all.
- **`R6.5`** — states that it binds streamed collections, which it always did.

### Evidence

Six research leaves under `baseline-04` — frame versioning, authorization
lifetime, caching, idempotency replay (two runs), resource ceilings, and
delivery-ended signalling — roughly 6,700 lines and 230 sources. Reviewed by
one adversarial pass and one different-model-family pass, then walked through
sixteen owner rulings before en-bloc ratification. A second different-family
pass over the **drafted rule text** produced the version reclassification
above and eleven text corrections.

Two findings changed the answer rather than confirming it. The claim that no
published API both accepts an idempotency key and streams was **false** —
Replicate's Cog does both on one endpoint and *attaches* a repeat to the
running stream, which is the opposite of the `409` the first run had mandated.
And the obvious fix for delivery-ended signalling, a new terminal frame type,
is a design **A2A shipped and then removed** as redundant, so `R12.10` and
`R13.9` were scoped instead.

### Known weaknesses, ratified knowingly

`R13.14`'s expiry clause is `SHOULD` because no published incident exists and
a `MUST` would overrule Kubernetes' shipped design; a dated re-check trigger
is registered. `R13.17`'s premise depends on a distributive reading of
`R8.10`'s compressed wording. `R12.10`/`R13.9`'s strongest precedents —
Temporal and A2A — are not HTTP streaming APIs.

### Still unruled

§13.4 keeps six interactions, down from eleven: cancellation across the two
channels, an unsupported `stream_position`, a no-`Accept` request to an
always-streaming resource, CSRF under ambient browser credentials, where a
§13.4 resolution is recorded — and one **residue**. `R13.12` ruled the caching
posture and settled which streaming responses owe a strong `ETag`, but not
what revalidation *means* for a representation still arriving: whether a `304`
is coherent against a body that does not yet exist. The register keeps what is
left of that entry rather than closing it whole.

## 1.1.3 — 2026-08-10

Editorial. No rule text, strength, or obligation changes; 139 rules, checklist
139/139, fixtures 14/14.

Phase 8 research and two review waves found defects in released text. This
patch ships them ahead of the Phase 8 rule proposals, because none of them
depends on that work and one is actively misleading.

### Corrected — guidance that caused the failure it warned against

[`streaming-profile.md`](streaming-profile.md) advised *"Add frame types
freely; rename none."* Adding a new **terminal** frame type follows that
advice and produces the identical failure a rename does: deployed clients
ignore the type they do not recognize, see no terminal frame, and report
truncation on every successful stream. An implementer following the guidance
was worse off than one ignoring it. The advice now distinguishes terminal from
non-terminal types.

### Corrected — two §13.4 claims that overstated their own gaps

- The register said a stream **cannot supply** a strong `ETag`. It can:
  RFC 9110 §8.8.1 permits a validator based on a revision identifier assigned
  before the representation is made accessible. Nor does `R3.10` fail to reach
  streams — it binds a *resource* that supports conditional update, and
  `R13.2` lets one endpoint serve both a streamed and a non-streamed
  representation, so such a resource still owes a strong `ETag`. What is
  genuinely unresolved is what **revalidation means for an incrementally
  delivered representation**: whether a `304` is coherent against a body that
  does not yet exist. The row also now notes that `R7.3` carries no BCP 14
  keyword, so it states a default posture rather than an obligation.
- The register said streams have **no required concurrency ceiling**.
  `R8.10`'s rate-limit axis default already calls for a published posture
  including concurrency; what is missing is only a statement that a stream
  occupies its slot for its lifetime. The row now says so, and flags that the
  reading depends on `R8.10`'s compressed wording.

### Added — six interactions the register had not recorded

Cancellation across the two channels · an unsupported `stream_position` ·
a no-`Accept` request to an always-streaming resource · renaming or removing
an open enum value in general · CSRF under ambient browser credentials · and
where a §13.4 resolution is actually recorded, given that `R1.7`'s template
has no slot for one.

The register now lists eleven interactions rather than five. Recording them is
the point: §13.4 exists so that a reader who hits one knows the silence is a
decision rather than an oversight, and six of them were oversights.

### Corrected — an Appendix A row that asserted a non-obligation

The checklist row for `R7.3` read as a requirement. `R7.3` contains no BCP 14
keyword, so the row now describes what a reviewer should look for rather than
a rule to enforce.

## 1.1.2 — 2026-08-10

Editorial. No rule text, strength, or obligation changes; 139 rules,
checklist 139/139, fixtures 14/14.

Two review findings that arrived on PR #9 after it merged, fixed here rather
than left on `main`:

- The 1.1.1 entry's lead-in read "Phase 7 renumbered to Phase 8", which
  became ambiguous the moment Phase 7 came to mean the skill-apparatus
  phase — a reader could take it as saying *that* phase was renumbered. It
  now names the streaming deferred-work phase explicitly and says the skill
  phase is unaffected.
- A missing preposition in the Part II map row for the Tier B deferral
  ("renumbered v1.1.1" → "renumbered in v1.1.1").

*Process note:* PR #9 was merged on CodeRabbit's green check before Copilot
had posted, and Copilot then found both items. The lesson is recorded rather
than the finding alone — a clean check from one reviewer is not the same as
the review wave having settled.

## 1.1.1 — 2026-08-10

Editorial. No rule text, strength, or obligation changes; the standard stands
at 139 rules, checklist 139/139, fixtures 14/14.

**The streaming deferred-work phase is renumbered from 7 to 8.** Phase 7 now
means the skill-apparatus phase, which is a different body of work and is
unaffected by this release. Version 1.1.0 opened a phase for
streaming's five unresolved interactions and numbered it 7. That number was
already claimed: a skill-apparatus phase had been declared as Phase 7
twenty-nine minutes earlier, on a branch that had not yet merged and was not
visible to the streaming work. The streaming phase merged first and so
reached released text first, but it was declared second — and merge order is
a poor way to settle a claim in a repository whose discipline is that the
record decides. The streaming phase therefore yields the number.

Renumbered here rather than on the unmerged branch because the alternative
would have rewarded merge order over declaration order, and because the
skill-apparatus phase carries a defined acceptance gate (Gate F) while this
one is a register of deferred work with no gate.

Corrected in `PLAN.md`, §13.4 and the Part II map of
[`rest-api-standard.md`](rest-api-standard.md), and
[`streaming-profile.md`](streaming-profile.md). The 1.1.0 entry below carries
its pointer corrected in place with a note, rather than being silently
rewritten — two sections briefly held the same phase number in released text,
and that is worth being able to see.

## 1.1.0 — 2026-08-10

Adds **§13, streaming responses** — Server-Sent Events, long-polling, and
streaming HTTP bodies. A MINOR bump: rules are added and nine existing rules
are scoped, none strengthened, removed, or re-meant. The standard now carries
**139 rules** (from 127); checklist 139/139; conformance fixtures 14/14.

**WebSockets are a stated non-goal**, not a further deferral: after a `101`
upgrade the exchange is no longer HTTP request/response, so none of this
standard's status-code, media-type, `Problem Details`, or applicability-switch
machinery reaches it. §1.2 now says so with its reason.

### Added

- **§13** (`R13.1`–`R13.11`) — response shape and negotiation, frame typing
  and termination, the post-commit error contract, composition with the
  `R10.9` operation resource, resumption, and long-polling.
- **`R12.10`** — the client half, placed in §12 with the other client
  obligations rather than in §13.
- **`streaming` applicability switch** (§1.8), plus three scoping principles
  Phase 6 forced into the open: where a rule's scope is stated, that a rule
  may require two switches, and that a switch never waives a guard.
- **§1.10 reserved names** — `stream` and `stream_position` request
  modifiers; `text/event-stream` as the table's first unregistered media type,
  with the registration gap disclosed in the row; a new **reserved stream
  members** category (`operation_id`, `operation_url`, `operation_state`,
  `retry_after`); and a
  new **reserved stream frame types** category, first entry `error`.
- **[`streaming-profile.md`](streaming-profile.md)** — an informative
  companion carrying the explanatory body, so §13 itself stays short for the
  majority of APIs that do not stream.
- Appendix D OpenAPI rows, Appendix E worked example **E.11**, six Appendix G
  live-probe rows, and two Spectral rules (`rs-r13-1-no-streaming-202`,
  `rs-r13-2-stream-negotiation-vary`).

### Changed — existing rules scoped for streaming

Each of these was a ratified `MUST` that a conforming streaming API could not
satisfy, because a committed `200` leaves no status code available:

- **`R5.1`** — the status binds to the outcome *as known when it is
  generated*; a post-commit stream failure reports under `R13.7`.
- **`R5.12`** — a second named carve-out, for post-commit stream errors.
- **`R5.13`** — the required member set is scoped to response-carried problem
  documents, so an in-band object omits `status`. RFC 9457 §3.1 permits either
  omitting `status` or setting it to the status actually sent, and forbids
  only a `status` that disagrees; omission is this standard's policy choice,
  because `status: 200` on a document describing a failure is accurate about
  the response and misleading about the outcome.
- **`R11.2`, `R11.5`** — the `429` + `Retry-After` obligation binds while a
  status can still be generated; mid-stream quota exhaustion reports under
  `R13.7` with a `retry_after` member.
- **`R2.11`** — a long-running action returns `202` where it does not stream,
  and `200` plus a stream media type where it does.
- **`R6.1`** — a streamed collection carries continuation state on its
  terminal frame in place of the envelope.
- **`R6.2`** — an empty streamed collection is zero item frames followed by
  the terminal frame, since there is no items array to be empty.
- **`R6.4`** — pagination state lives in the body *representation* — the
  envelope, or the terminal frame when streamed. The `Link`-header
  prohibition binds streamed collections identically.

Also: §1.5's namespace extended from twelve normative sections to thirteen;
`R1.3`'s frozen-series list gains `ST-001`–`ST-020`; §1.9's conformance-note
template gains the `streaming` switch; §1.11 defines *stream*, *frame*,
*terminal frame*, *self-delimiting stream media type*, and *status committed*.

### Known gaps, recorded rather than left to be found

**§13.4** registers five interactions between streaming and the rest of the
standard that are recognized and not yet ruled — frame-vocabulary versioning,
stream authorization lifetime, caching posture, idempotency-key replay, and
stream resource ceilings. `PLAN.md` **Phase 8** — numbered 7 when this
release shipped and renumbered in 1.1.1, see below — is open to rule them with the
same evidence discipline that produced §13. The versioning item carries the
sharpest known failure and an interim posture: treat frame-type names and
which types are terminal as frozen surface, because `R12.10`'s
ignore-unknown-types tolerance would otherwise turn a rename into truncation
reported on every successful stream.

### Evidence

Two research leaves under the two-series discipline —
`survey-08-streaming` (descriptive; 14 contested axes) and
`baseline-04-streaming` (prescriptive; 20 `ST-*` principles) — ratified
through a six-fork owner walk plus an en-bloc batch, recorded in
[`research/decisions/baseline-04-streaming.decision.md`](research/decisions/baseline-04-streaming.decision.md).

One finding is load-bearing enough to restate: **`text/event-stream` has no
IANA registration.** It is absent from the `text/*` subregistry and its
per-type URL returns `404` against a `text/html` control returning `200`.
`R13.4` blesses it anyway — the only registered alternative,
`application/json-seq`, has no HTTP adoption and no browser parser — but
requires every adopting API to disclose the gap. A dated re-check is
registered for **2027-02-10**.

Reviewed in four waves — three internal lenses (consistency, ambiguity,
missing-and-conflicting) and a Codex second lens from a different model
family on the corrected text — then through the PR bot cycle. Released via
PR #8; `v1.1.0` is tagged on that PR's merge commit.

## 1.0.0 — 2026-08-10

First release. 127 rules across twelve normative sections, each carrying
provenance (decision-layer key), classification, and confidence; Part II
Decision Log (provenance map + apparatus register); Appendices A–G;
executable conformance fixtures (`conformance/spectral.yaml`,
execution-verified against `conformance/fixture-violations.yaml`).

Produced by the phased process in [`PLAN.md`](PLAN.md):

- **Gate C (2026-08-09):** all 66 researched principles plus the
  structural locks and five addenda ratified into
  [`research/decisions/`](research/decisions/) (PRs #3, #4).
- **Gate D (2026-08-09):** draft approved for systematic review (PR #5);
  SDK-configuration conventions excluded, SSE/streaming deferred to a
  post-1.0 phase.
- **Phase 4 + Gate E (2026-08-10):** three review waves (internal
  three-agent, Codex second lens, raw-RFC classification sweep), four
  owner rulings (switch pruning, `cancel` scope, R4.16 path
  placeholders, R10.9 operation discovery via research leaf
  `baseline-02i`), and the Gate E CORS ruling (R4.17); approved as the
  1.0 candidate and landed via PR #6.
- **Phase 5 release (2026-08-10):** version flipped to 1.0.0, this
  changelog created, and `v1.0.0` tagged on the merge commit of release
  PR #7.
