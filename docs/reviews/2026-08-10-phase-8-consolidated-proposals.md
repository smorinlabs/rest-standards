# Phase 8 — consolidated proposal set

**Date:** 2026-08-10 · **Status:** proposals only. Nothing here binds
`rest-api-standard.md` until ratified in a decision record.

## What this document is

Five research leaves ran in parallel under `baseline-04`, one per interaction
that §13.4 of the standard records as recognized and not yet ruled. Each leaf
proposed rules independently, so their identifiers collided and two of them
proposed overlapping rules. This document reconciles them into one set that a
ratification walk can take item by item.

It changes no leaf's reasoning. Where a merge alters a proposal, the change and
its reason are stated. The leaves remain authoritative for evidence; this
document is authoritative only for **which identifier each proposal carries**
and **which proposals merged**.

| Leaf | Topic | Report |
| --- | --- | --- |
| `baseline-04b` | Frame-vocabulary versioning | 1,151 lines, 53 sources |
| `baseline-04c` | Authorization over a stream's lifetime | 803 lines, 23 sources |
| `baseline-04d` | Caching posture for streams | 517 lines, 19 sources |
| `baseline-04e` | Idempotency-key replay of a streaming request | 788 lines, 22 sources |
| `baseline-04f` | Resource ceilings for streams | 821 lines, 27 sources |

## The collisions, and how each is resolved

**Rule identifiers.** Two leaves independently proposed `R13.12`
(`baseline-04c` and `baseline-04d`). Each correctly applied `R1.2` — "New rules
take the next unused sequence number in their section" — against the same
snapshot of §13, which ends at `R13.11`. `R1.2` has no concurrency story, and
parallel dispatch is what exposed it. A third leaf, `baseline-04b`, mentions
`R13.12` only as a **declined alternative**, so it is not a party to the
collision.

**Principle identifiers.** `baseline-04f` minted `ST-012` and `ST-013`, both
already ratified in version 1.1.0 — `ST-012` is the client-obligations
principle behind `R12.10`, and `ST-013` is the newline-delimited-JSON companion
guidance. A reader following either key from a Phase 8 rule would land on an
unrelated principle, which is the fabricated-lineage hazard `R1.3` exists to
prevent. `baseline-04c` claimed `ST-021`; `baseline-04e` anticipated the
collision and used leaf-scoped `ST-E01` and `ST-E02`, recording the conflict
rather than guessing.

`ST-001`–`ST-020` are frozen by `R1.3`. Phase 8 principles therefore begin at
**`ST-021`**, assigned below in leaf order.

**Overlapping proposals.** `baseline-04c` and `baseline-04f` each proposed a
maximum-stream-duration requirement, reaching it from opposite directions —
bounding exposure after credential expiry, and bounding resource commitment.
They are merged, and the merge is not a coin toss; see `ST-024`.

## Identifier assignment

| Principle | Rule target | Source leaf | Was |
| --- | --- | --- | --- |
| `ST-021` | Amendment to `R9.4` (no new rule) | `04b` | unnumbered |
| `ST-022` | Amendment to `R13.5` (no new rule) | `04b` | unnumbered |
| `ST-023` | New `R13.12` | `04d` | `R13.12`, unnumbered principle |
| `ST-024` | New `R13.13` | `04c` + `04f` merged | `ST-021` / `ST-012` |
| `ST-025` | New `R13.14` | `04c` | part of `ST-021` |
| `ST-026` | New `R13.15` | `04e` | `ST-E01` |
| `ST-027` | Companion guidance, no rule | `04e` | `ST-E02` |
| `ST-028` | New `R13.16` | `04f` | `ST-013` |

Ratifying any of these requires adding its identifier to `R1.3`'s frozen-series
list, exactly as `ST-001`–`ST-020` were added in version 1.1.0.

---

## `ST-021` — Frame-type names and terminality are frozen

**Target:** amend `R9.4`, the compatibility taxonomy in §9.3. No new rule.
**Class:** `[COMPARATIVE]` on the name and meaning clauses; `[POLICY]` on
terminality. **Confidence:** moderate-high, except terminality at moderate.

Add to **Frozen**: documented stream frame-type names, their meanings, and
which of them are terminal.

Add to **Compatible**: adding stream frame types, where the API has documented
its frame-type vocabulary as growable.

Add to **Breaking**: changing whether a documented frame type is terminal, or
changing what a documented frame type means, with or without a change of name.

**Why the compatible entry is conditional.** Kubernetes holds that adding an
enumerated value is not compatible unless growth was declared when the
vocabulary was first published. The leaf surfaced that dissent rather than
averaging it, and the condition reconciles it with the seven vendors who permit
free addition.

**Why terminality is separately frozen.** Demoting a type from terminal without
renaming it produces the same client-visible failure as renaming it: under
`R12.10` the client no longer recognizes a terminal frame and reports
truncation on success. A name freeze alone would not catch it. No source states
this; it is this standard's own construction, labeled `[POLICY]`.

**Field confirmation.** Google Gemini renamed six Interactions event types
including the terminal `interaction.complete`, published it as a breaking
change behind a dated version header with a sunset date. Every announced rename
found rode a version mechanism. One unannounced counterexample exists on
OpenAI's generally-available surface and was not licensed by OpenAI's own
published compatibility list.

## `ST-022` — Vocabulary documentation marks terminality; retirement has a path

**Target:** amend `R13.5`. No new rule. **Class:** `[POLICY]` on the marking
duty, `[COMPARATIVE]` on the retirement path. **Confidence:** moderate-high.

The vocabulary documentation marks, per frame type, whether it is terminal. A
retired frame-type name is never reused for a different meaning. Retirement
goes either to a new major version, or through a documented dual-emit overlap
period stated in the vocabulary documentation.

**Why the overlap window must be documented rather than signaled.** `R9.5`'s
`Deprecation` and `Sunset` headers are response headers. A stream's headers are
sent once, before any frame, so they cannot deprecate one frame type inside a
stream. The header channel is unavailable, which is a protocol fact; the
documentation remedy is policy.

## `ST-023` — Caching posture for a stream

**Target:** new `R13.12`. **Class:** `[POLICY]` on the tier choice, resting on
protocol facts per clause. **Confidence:** moderate-high.

A streaming response carries an explicit `Cache-Control`; `R7.1` is not relaxed
by incremental delivery, because the header section precedes the body and the
directive describes policy rather than bytes. Within `R7.3`'s posture a stream
takes tier 1, `private, no-cache`, by default, and **tier 1's strong-`ETag`
revalidation clause does not apply**, because `R3.10` binds conditionally
updatable resources and a stream is not one. Tier 2, `no-store`, applies on the
same condition as for any other response — a genuinely sensitive payload — and
is not adopted merely because the response is a stream. Tier 3 is permitted
only where the stream views an immutable retained artifact **and** every input
selecting a resumption point is in the request URI or named in `Vary`; an
unbounded stream never uses tier 3.

**The hazard behind the `Vary` condition, which no prior review found.** SSE
resumption travels in the `Last-Event-ID` **request header**. A cache key does
not include it unless `Vary` names it, so a cache could answer a
resume-from-position-N request with a stored stream beginning at position 1.

**Two premises this leaf disproved, both in released text.** `R7.3` contains no
BCP 14 keyword at all, so Appendix E's `private, no-store` was never a
conflict — only an unpicked default. And §13.4's claim that a stream cannot
supply a strong validator is too strong: RFC 9110 §8.8.1 permits a
revision-identifier validator computed before transmission. What actually
blocks tier-1 revalidation is `R3.10`'s scope, not validator availability.

## `ST-024` — Stream duration bound (merged)

**Target:** new `R13.13`. **Merges** `baseline-04f`'s duration proposal with
`baseline-04c`'s first clause. **Class:** `[POLICY]` on the requirement,
`[COMPARATIVE]` on publish-your-number practice. **Confidence:** moderate-high.

For each streaming endpoint, an API documents either the maximum duration it
will hold the response open, or that the stream is unbounded by design — the
unbounded declaration being the one `R13.6` already requires, which discharges
this rule. Where a maximum is documented, the server enforces it and ends the
stream at the maximum with `R13.6`'s terminal frame rather than by closing the
connection, because a close at a published limit is a normal end and a client
must not be made to report it as truncation. A documented maximum SHOULD be
randomized per connection within a published range.

**How the merge resolved, and why it is not arbitrary.** `baseline-04c`
proposed a flat "MUST publish a maximum stream duration." `baseline-04f`
proposed "document a maximum **or** declare the stream unbounded." The second
formulation governs, because the first would make every watch and event-tail
API nonconformant — and `R13.6` already provides for unbounded streams
explicitly. Taking the stricter text would have contradicted a rule ratified in
version 1.1.0.

**No number is set.** The surveyed spread runs four orders of magnitude, which
is the evidence that no number is standardizable. The jitter clause is the
weakest of the three and rests on two implementations; it can be dropped
without harming the rule.

## `ST-025` — Authorization over a stream's lifetime

**Target:** new `R13.14`. **Class:** `[POLICY]` throughout, grounded in
RFC 9068 §4 for the expiry clause. **Confidence:** moderate-high on the
disclosure clauses, moderate on the expiry clause.

A stream does not continue past the expiry of the credential that authorized
it: where the credential carries or implies an expiry, the server ends the
stream at or before it with an `error` frame stating that the credential
expired, rather than relying on the maximum duration to bound it. An API SHOULD
end an in-flight stream when authorization is revoked, and MUST document its
revocation posture as an upper bound on how long a stream may continue after
revocation — never as a claim that revocation is immediate. An API that does
not terminate on revocation says so, and its maximum duration is then the
exposure window it is publishing. Resumption is a new request and is authorized
as one; a resumption position is never evidence of authorization.

**Why disclosure alone was judged insufficient.** The standard did accept a
disclosure-only duty for keep-alive frames. The leaf distinguishes this case:
an authority for the underlying obligation already exists in `R8.6`'s
deny-by-default posture and RFC 9068 §4's expiry check, whereas no authority
existed for a keep-alive interval.

**The honest weakness.** No published incident of post-revocation delivery over
an HTTP stream was found. The threat is argued from mechanism. The expiry
clause also overrules the field's clearest precedent, since a Kubernetes watch
plausibly outlives its bound token — which is why that clause carries the
lowest confidence in the set.

**Consequential amendment.** `R8.10`'s token-format axis requires a
revocation-propagation plan; it would gain a clause requiring that plan to
state its effect on in-flight streams.

## `ST-026` — A keyed repeat of a streaming request never re-executes

**Target:** new `R13.15`. **Class:** `[POLICY]`. **Confidence:** moderate-high
on the non-re-execution clause; moderate on the response shape.

A repeat of a streaming request carrying the same idempotency key does not
re-execute the work. The server answers by the original execution's state: a
defined `409` conflict while the original is still executing; the recorded
outcome once a terminal state exists; and, for the case where execution began
and was abandoned without reaching a terminal state, a documented behavior. For
a stream, "the stored response" means its terminal state plus a documented
replayable representation, not the original frame sequence.

**Why this is not over-specification.** Across fourteen published sources, no
API both accepts an idempotency key and streams. Every keyed API does not
stream; every streaming API has no key. `R3.9` combined with §13 manufactures
an intersection with zero field precedent, which is exactly why two conforming
servers can differ on it today — one re-executing, one replaying.

**Where the three-source agreement is.** The `409`-while-executing answer is
supported by the expired IETF Idempotency-Key draft §2.6, Stripe's
`idempotency_key_in_use`, and Shopify's `IDEMPOTENCY_CONCURRENT_REQUEST`.

**Why frame-for-frame replay is permitted but never required.** `R3.9`'s
retention floor is at least 24 hours. The field's retained artifacts run about
10 minutes (OpenAI) and about 5 minutes (Kubernetes). Requiring frame-for-frame
replay would require retention an order of magnitude beyond what anyone ships.

**Replay and resumption are not substitutes.** They have different
preconditions, different carriers, and different failure modes. They converge
only when execution completed over a retained artifact, where replay is
resumption from position zero.

## `ST-027` — Prefer splitting execution from delivery

**Target:** companion guidance in `streaming-profile.md`. No rule.
**Class:** `[COMPARATIVE]`. **Confidence:** moderate.

An irreversible non-idempotent mutation should not stream its own result.
Prefer splitting execution from delivery: the mutation returns an operation
resource, and the stream is a safe `GET` over it, resumable under `R13.10`.
This is OpenAI's shipped `background: true` design.

**Why not a prohibition.** A flat ban was assessed against AIP-151 and
declined: AIP-151 binds long-running-operation methods only, and a ban would
forbid the pattern §13 exists to serve.

## `ST-028` — A held-open stream occupies a concurrency slot

**Target:** new `R13.16`. **Class:** `[POLICY]`, clarifying an already-ratified
axis. **Confidence:** moderate on the strength, high that the clarification is
needed.

A held-open stream occupies one unit of the concurrency dimension of `R8.10`'s
rate-limit axis for the whole time the server holds it open, not only at
arrival. An API documents how streams are counted against its published
concurrency posture. Where a request is rejected because the concurrency
posture rather than the rate posture is exhausted, the `429` required by
`R11.2` SHOULD distinguish the two conditions in its problem `code`.

**This is a clarification, not a new ceiling.** `R8.10`'s rate-limit axis
default already requires a published posture including concurrency, and
`R8.10` makes adopting the axis defaults a MUST. The residual gap is only that
nothing says a stream holds the slot for its lifetime.

---

## What the research removed from the register

Two of §13.4's five entries overstate their own gaps, and the ratification walk
should correct the register regardless of which proposals it adopts.

| Register claim | What the research established |
| --- | --- |
| Streams have no required concurrency ceiling | `R8.10` already requires a published concurrency posture. Only the lifetime-occupancy clarification is missing. |
| Streamed collections have no published maximum | `R6.5` is unqualified and was never scoped away from streamed collections, so they already owe one. |
| A stream cannot supply a strong `ETag` | RFC 9110 §8.8.1 permits a pre-transmission revision validator. `R3.10`'s scope is the actual blocker. |
| `R7.3` conflicts with streaming | `R7.3` carries no BCP 14 keyword, so it never bound anything. There was no conflict. |

## Defects found outside Phase 8's scope

Both predate streaming and are independent of every proposal above.

1. **Appendix A's checklist row for `R7.3` states an obligation that `R7.3`
   does not impose.** `R7.3` has no BCP 14 keyword; its checklist row reads as
   a requirement.
2. **`R9.4` does not classify removing or renaming an open enum value in
   general.** Frame types are one instance. `operation_state` (`R10.1`'s
   vocabulary, made cross-channel load-bearing by `R13.9`) and `R6.7`'s
   sortable-field set have the same hole. `ST-021` patches the streaming
   instance; the class remains open.

## What the ratification walk must decide

Each of `ST-021` through `ST-028` is a separate ruling. Beyond adopting or
declining each, three questions cut across them:

1. **Does `ST-025`'s expiry clause survive?** It carries the lowest confidence
   in the set and overrules the field's clearest precedent.
2. **Is `ST-027` guidance or a rule?** It is proposed as companion guidance;
   the ratification walk may judge that an irreversible-mutation preference
   belongs in normative text.
3. **Are the register corrections and the two out-of-scope defects handled in
   the same release, or separately?** They are editorial relative to the rule
   proposals.
