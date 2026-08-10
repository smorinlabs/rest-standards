# Changelog

Versioning follows the Part II amendment rule of
[`rest-api-standard.md`](rest-api-standard.md): editorial changes bump
patch; added rules, appendices, or relaxations bump minor; strengthened,
removed, or re-meant rules bump major. Every rule change is atomic across
the rule text, its decision record, its Part II row, its checklist row,
and the worked example.

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
stream resource ceilings. `PLAN.md` **Phase 7** is open to rule them with the
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
family on the corrected text — then released via PR #8, with `v1.1.0` tagged
on its merge commit.

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
