# Phase 8 — consolidated proposal set

**Date:** 2026-08-10 · **Revision 2, same day** · **Status:** proposals only.
Nothing here binds `rest-api-standard.md` until ratified in a decision record.

## Revision 2 — what changed and why

An adversarial review of revision 1 found that this document had **altered
four leaf proposals while stating that it altered none**. Revision 1's opening
claim — "It changes no leaf's reasoning" — was false, and the alterations were
not marked. That is a defect in the consolidation, not in the research. All
four are restored below and each restoration is marked.

| Proposal | What revision 1 did | Effect |
| --- | --- | --- |
| `ST-022` | Dropped "retiring the old name only at a major version" | Converted dual-emit from a preparation for a major bump into a substitute for one, making `ST-022` contradict `ST-021` |
| `ST-027` | Demoted a SHOULD rule with two MUST clauses to non-normative companion guidance, then described that as the leaf's proposal | Removed the conformance-note obligation the leaf attached |
| `ST-029` | Omitted `baseline-04f`'s third proposal entirely | A scoping note the leaf judged high-confidence was lost |
| `ST-026` | Retargeted `baseline-04e`'s `R3.9` amendment onto a new §13 rule | Changed the switch scope of a §3 definition |

The same review falsified `ST-026`'s load-bearing evidence. That is recorded
in its section below and the leaf is being re-run; `ST-026` is not ratifiable
in its current form.

Revision 2 also corrects three overstatements revision 1 introduced when
summarizing leaf findings, each marked in place, and adds the review's own
findings as a closing section.

## Decision walk — 2026-08-10, ten rulings

Walked one at a time with the owner before the ratification walk, so the walk
inherits settled ground rather than re-deciding it. Each ruling is recorded in
full at its proposal; this table is the index.

| # | Question | Ruling |
| --- | --- | --- |
| 1 | Ship the editorial corrections now or hold for Phase 8? | **Ship as `v1.1.3`** — six defects sit in released text, including guidance that causes the failure it warns of |
| 2 | Fix the open-enum hole at the class or the instance? | **The class.** `R9.4` gains a general rule; `ST-021` shrinks to its terminality clause |
| 3 | Can dual-emit retire a terminal frame type? | **No.** Terminal types go via major version only; `R13.6` untouched |
| 4 | `ST-026`'s shape after its evidence was falsified | **Non-re-execution at MUST; response shape a documented choice** between attaching and `409` |
| 5 | Must an attached stream reveal missed frames? | **Yes — server obligation.** A silent false completion is the failure class `R1.9` and `R13.3` both guard against |
| 6 | `R3.9`'s "naturally idempotent" exception | **Clarify the premise** — it covers a `PUT` that stores a representation, not one that starts work |
| 7 | A stream cut at its cap has no terminal state | **Exempt and signal.** The terminal frame is exempt from `operation_state`; the client continues via the operation resource |
| 8 | The unbounded-plus-non-expiring escape hatch | **Gate the claim, then require the statement.** Unbounded requires `R13.6`'s no-normal-end test; unbounded plus a non-expiring credential must state its exposure window |
| 9 | A request carrying both a key and a position | **They compose.** Key governs execution, position governs delivery |
| 10 | `ST-024`'s jitter clause | **Move to the companion** — real finding, evidence below the bar for a `SHOULD` |
| 11 | A different-family review before the walk? | **Run it.** The last one found the consolidation layer was the weak link, and this document has been revised ten times since |

Two earlier rulings stand unchanged: `ST-025`'s expiry clause ships at
`SHOULD` with a research trigger, and cancellation is registered now and
researched later.

**Still open, not yet consented into the pile:** the retention-mismatch case at
D4 — a keyed repeat arriving while the terminal state is retained but the
replayable representation has been collected, which `R3.9`'s 24-hour floor
against 5-to-10-minute artifacts makes the common case rather than the rare
one.

## What this document is

Five research leaves ran in parallel under `baseline-04`, one per interaction
that §13.4 of the standard records as recognized and not yet ruled. Each leaf
proposed rules independently, so their identifiers collided and two of them
proposed overlapping rules. This document reconciles them into one set that a
ratification walk can take item by item.

The leaves remain authoritative for evidence and for rule text. This document
is authoritative only for **which identifier each proposal carries** and
**which proposals merged**. Where it alters a leaf's proposal it says so at
that proposal; revision 1 failed this test in four places and revision 2
restores them.

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
| `ST-027` | New `R13.17`, at SHOULD | `04e` | `ST-E02` |
| `ST-028` | New `R13.16` | `04f` | `ST-013` |
| `ST-029` | Scoping note on `R6.5`, no rule text change | `04f` | `ST-014` |

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

**Field confirmation — with the qualifier revision 1 dropped.** Google Gemini
renamed six Interactions event types including the terminal
`interaction.complete`, published it as a breaking change behind a dated
version header with a sunset date. **The leaf classifies that rename as
pre-GA, which `R9.4` already permits** — `R9.4` opens "Within a GA major
version." So the headline field evidence is not a GA-freeze precedent, and
revision 1 presented it as one by omitting the classification.

Two further limits: the mechanism Gemini used is a dated *request header*,
which is not a version mechanism this standard has — `R9.1` requires the major
version in the path and `R9.2` requires additive evolution within a major. And
the OpenAI counterexample was stated in the present tense; it is dated to an
issue opened 2025-05-23 and has since been resolved by the replacement event
shipping.

**None of this sinks the proposal** — the gap it closes is real, because
`R9.4` reaches frame types only through "reserved-name semantics (§1.10)", and
§1.10 registers exactly one frame type while stating that an API's own
vocabulary "is otherwise its own." It sinks the paragraph, not the rule.

## `ST-022` — Vocabulary documentation marks terminality; retirement has a path

**Target:** amend `R13.5`. No new rule. **Class:** `[POLICY]` on the marking
duty, `[COMPARATIVE]` on the retirement path. **Confidence:** moderate-high.

The vocabulary documentation marks, per frame type, whether it is terminal. A
retired frame-type name is never reused for a different meaning. Retirement
goes either to a new major version, or through a documented dual-emit overlap
period stated in the vocabulary documentation, **retiring the old name only at
a major version**.

**[Restored in revision 2.]** The final clause is the leaf's and was dropped in
revision 1. Without it, dual-emit reads as a *substitute* for the major bump
rather than a *preparation* for one, which would let an API remove a frozen
frame type inside a GA major — contradicting `ST-021`, which adds frame-type
names to `R9.4`'s Frozen list, and `R9.4`'s Breaking entry "removing or
renaming any frozen element."

**Open defect the walk must resolve: dual-emit cannot express a terminal
frame.** `ST-021`'s motivating case is a terminal rename, but `R13.6` requires
a stream to end with *the* terminal frame, singular, and §1.11 defines that as
"the frame that ends a stream." Two concurrent typed, payload-carrying
terminal frames are not expressible: the first would not have ended the
stream, and a `R12.10` client recognizing the old name stops reading at it.
Where `R13.9` also binds, neither the leaf nor this document says which of two
dual-emitted terminal frames carries `operation_state`. Every dual-emit
precedent the leaf cites — GitHub, CloudEvents, Standard Webhooks — is a
webhook system, where two deliveries are two independent messages; a stream has
exactly one ending. **Either terminal types are excluded from the dual-emit
path, leaving only the major-version route for them, or `R13.6` needs an
overlap ending defined.** Found by the revision-1 adversarial review; not
noticed by the leaf.

**Decision (2026-08-10): exclude terminal types from the dual-emit path.**
Dual-emit remains available for non-terminal frame types; a terminal-frame
rename goes only via a new major version. `R13.6` is not amended.

The reasoning: a stream having exactly one ending is not an obstacle to work
around — it is the guarantee `R13.6` sells and `R12.10` consumes, and an
overlap ending would make a client stopping at the old terminal frame
sometimes right and sometimes early. The field evidence points the same way:
the one real terminal rename, Gemini's `interaction.complete`, used a version
gate rather than dual-emit, so this matches observed practice while the
alternative would license something nobody has shipped. Every dual-emit
precedent the leaf cites is a webhook system, where two deliveries are two
independent messages and nothing declares the conversation over.

**Recorded cost:** terminal-frame renames now require a major version. That is
accepted as correct — a terminal frame is the one frame every client must
recognize, so renaming it is the most breaking change a stream vocabulary can
make.

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
is the evidence that no number is standardizable.

**Decision (2026-08-10) on the jitter clause: move it to the companion.** The
randomize-the-deadline advice leaves `ST-024`'s rule text and becomes
informative guidance in `streaming-profile.md`.

The finding is real and worth keeping — streams opened against a fixed
deadline expire together and reconnect together — and the two implementations
behind it, Kubernetes' `[1.0, 2.0)` factor and Consul's `wait / 16`, are the
two most-studied long-poll systems in the research. But two implementations is
below the bar this phase has applied to every other `SHOULD`, and the leaf
volunteered as much: "the weakest of the three… could be dropped without
harming the rule." It is also operational tuning rather than an
interoperability requirement, which is where `ST-019` already puts comparable
material — proxy buffering, compression, idle timeouts.

Dropping it entirely was declined: a reader who never meets the idea builds the
fixed deadline and finds the herd in production.

**Decision (2026-08-10) on the `R13.9` collision (D2): exempt and signal.**
Where a stream ends at its documented maximum with the operation still
running, the terminal frame indicates that and is **exempt from carrying
`operation_state`**. The client continues through the operation resource,
which `R13.9` already makes reachable by requiring `operation_id` or
`operation_url` on every frame.

The conflation being corrected: `R13.9` treats the stream ending and the
operation ending as one event, because they usually coincide. A duration cap
is exactly where they diverge — the stream is over, the work is not, and the
operation is therefore in none of `R10.1`'s terminal states. Routing it
through an `error` frame does not help, since `R13.9` binds error terminal
frames identically and calling a published, expected limit an error
contradicts `ST-024`'s own reasoning that reaching the cap is a normal end.

Two alternatives were declined. Relaxing `R13.9` to carry the operation's
*current* state would weaken the cross-channel comparison in every case to
accommodate one, and that comparison being between **terminal** states is the
whole point of it. Forbidding a cap while the operation still runs would rule
out the long-export case that motivates having a cap at all.

**Version consequence:** scoping `R13.9` in a named case is a relaxation,
which the amendment rule puts at MINOR — the bump Phase 8 already carries.

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

> **NOT RATIFIABLE AS WRITTEN — evidence falsified 2026-08-10.** The leaf's
> load-bearing claim was a universal negative: that no published API both
> accepts an idempotency key and streams. It is false. Replicate's Cog HTTP API
> documents both on one endpoint — a capability table row reads
> `PUT /predictions/<prediction_id>` with `Accept: text/event-stream` →
> "Streaming, idempotent", and the endpoint is described as "the idempotent
> version of the `POST /predictions` endpoint." Verified independently against
> the raw source file, not the leaf.
>
> Worse for the proposal: on a keyed repeat while the original is running, Cog
> "returns a stream for the existing prediction instead of creating a duplicate
> prediction" — it **attaches the caller to the in-flight stream**. The clause
> below requiring a `409` conflict in exactly that case is the one the leaf
> rated highest-confidence, and the only shipped implementation of the
> intersection does the opposite. Its behavior is also better for the client,
> who receives the output rather than an error. The three sources supporting
> the `409` answer — the expired IETF draft §2.6, Stripe, Shopify — are all
> non-streaming APIs.
>
> The leaf was re-run against this evidence
> (`baseline-04e-...report.2026-08-10b.md`, 1,479 lines, 49 sources). The text
> below is retained as the first run's proposal, superseded by the decision
> that follows.

**Decision (2026-08-10): adopt run b's shape — non-re-execution at `MUST`, the
response shape a documented choice.**

The **non-re-execution invariant survives and is strengthened**: a keyed repeat
must not run the work again. It now rests on seven sources. That is the safety
property, and it is what prevents a payment executing twice.

The **mandated `409` does not survive**, for two independent reasons. Run b
found the only shipped implementation at the intersection — Replicate's Cog —
*attaches* the caller to the in-flight stream rather than rejecting. And run a
had escalated the expired IETF draft's `409` from `SHOULD` to `MUST`, an error
unrelated to the falsified negative. The rule now names two permitted
behaviors, and the API documents which it implements:

| Behavior | Shipped by | Trade-off |
| --- | --- | --- |
| Attach to the in-flight stream | Replicate Cog; Temporal `USE_EXISTING` | Better for the client — receives the output rather than an error |
| Reject with `409` | Stripe, Shopify; IETF draft at `SHOULD` | Better for the server — no reader fan-out on one execution |

A preference between them was considered and declined: the sources run two
against three, so a `SHOULD` either way would be this standard's opinion
presented as evidence. Leaving the shape entirely undocumented was also
declined, because a client would then have to handle any behavior at all —
naming both is what bounds the space it must handle.

Run b found exactly one shipped implementation and one specification-level
near-miss (A2A Protocol v1.0.0, deduplication at `MAY`) across roughly forty
candidates in three parallel sweeps. It also recorded a **false falsifier** as
a methodology warning: Bedrock AgentCore's `idempotencyToken` is an SDK
auto-fill marker, not deduplication.

**Decision (2026-08-10), carried by permitting attach: an attached or resumed
stream must not silently hide a gap.** Where the server cannot deliver frames
the client missed, it signals that rather than delivering a terminal frame as
though the stream were complete.

The hole this closes: `R12.10`'s completeness test is **suffix-shaped** — it
asks only whether the documented terminal frame arrived, never whether the
middle arrived. Run b found Cog's replay buffer is denominated in **events
(1024), not time**, and its capacity is configurable; at zero capacity an
attaching client receives the terminal frame with every intermediate frame
missing and no error. That client, following `R12.10` correctly, reports
success on a stream that lost its entire contents.

Cog already implements the remedy — if earlier events have been dropped from
the replay buffer, the stream emits an error event and closes — so the rule
describes shipped behavior rather than inventing one. No client-side clause is
needed: an `error` frame is already terminal under `R13.6` and `R13.7`, so a
client receiving one already reads failure rather than silent success.

A documentation-only duty was declined. Documentation does not prevent the
client being told the wrong thing, and a silent false completion is the same
failure class that `R1.9`'s `dry_run` guard and `R13.3`'s `stream` guard both
exist to prevent.

**Decision (2026-08-10) on `R3.9`'s exception: clarify what "naturally
idempotent" means, rather than narrowing the exception or routing around it.**

`R3.9` ends "**Exception:** naturally idempotent operations (PUT with a
client-supplied ID)." Cog's endpoint is exactly that shape —
`PUT /predictions/<prediction_id>` — so as written it is exempt from the whole
rule, including the non-re-execution guarantee and the reject-on-different-
payload guarantee. The client-supplied identifier *is* the deduplication key;
it is simply not carried in a header.

The premise is what fails. "Naturally idempotent" holds for a `PUT` that
stores a representation — send the same body twice, get the same state. It
does not hold for a `PUT` that **starts work**, because a second one runs the
work again. The drafting did not anticipate work-starting `PUT`s.

The fix therefore states that the exception covers a `PUT` that stores a
representation, not one that starts work. This fixes the whole class rather
than the streaming instance, and it does so by explaining a term the rule
already uses.

Two alternatives were declined: narrowing the exception to exempt only the
header, which adds obligations to currently-exempt operations and is a
strengthening the amendment rule puts at MAJOR; and binding the guarantees
inside `ST-026` alone, which avoids touching §3 but leaves the defect live for
every non-streaming work-starting `PUT`, the larger population.

> **Open for the ratification walk — version classification.** Whether this
> clarification is *editorial* (patch) or *re-means* `R3.9` (major) is a
> genuine judgment call under the amendment rule. The walk should rule on the
> classification explicitly rather than let a commit message settle it.

**Target:** new `R13.15`. **Class:** `[POLICY]`. **Confidence:** moderate-high
on the non-re-execution clause; moderate on the response shape.

**[Restored in revision 2 — amendment surface.]** The leaf proposes the
definition of "the stored response" as a clause riding `R3.9` itself, in §3.
Revision 1 folded it into a new §13 rule. Those are different changes: `R3.9`
is not scoped by the `streaming` switch and a new `R13.x` would be, so folding
it changed which APIs the definition binds. The definition of a §3 term belongs
on the §3 rule. This part is independent of the falsified negative and survives
it.

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

**Target:** new `R13.17`, at SHOULD. **Class:** `[COMPARATIVE]` on the split.
**Confidence:** moderate-high.

**[Restored in revision 2.]** Revision 1 demoted this to non-normative
companion guidance and then described that as the leaf's proposal. The leaf
proposes it as a rule, and its final sentence carries two MUST clauses that
non-normative text cannot hold — §13's own preamble says of the companion that
"nothing there is normative."

An API SHOULD NOT stream the response to a non-idempotent mutation whose
repeated execution has an external effect that cannot be reversed — a payment,
a disbursement, a message send, a metered charge. Where such a capability needs
incremental delivery, the API SHOULD split it: the mutation is a non-streaming
request returning an operation resource (`R10.9`, `R13.9`), and the incremental
delivery is a **safe** request over that resource, resumable under `R13.10`. An
API that does stream such a mutation MUST comply with `ST-026` and MUST state
in its conformance note (`R1.7`) which of `ST-026`'s cases it implements and
how.

**Why not a prohibition.** A flat ban was assessed against AIP-151 and
declined: AIP-151 binds long-running-operation methods only, and a ban would
forbid the pattern §13 exists to serve.

**Dependency.** The final sentence references `ST-026`, which is not currently
ratifiable (see its section). If `ST-026` changes shape, this sentence follows.

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

## `ST-029` — Streamed collections owe their documented maximum

**Target:** a scoping note on `R6.5`, or one sentence in §13. No rule-text
change. **Class:** `[POLICY]`. **Confidence:** high.

**[Restored in revision 2.]** Revision 1 omitted this proposal entirely,
converting it into a register-correction row and dropping its remedy.

A streamed collection documents and enforces its maximum `limit` exactly as an
unstreamed one does. `R6.5` already binds it — the rule is unqualified, and
`R6.4`'s version 1.1.0 widening to "the terminal frame where the collection is
streamed" is the precedent that §6 reaches streamed collections. **The standard
has never said so**, which is the whole content of this proposal: it closes a
silence, not a gap.

---

## What the research removed from the register

Two of §13.4's five entries overstate their own gaps, and the ratification walk
should correct the register regardless of which proposals it adopts.

| Register claim | What the research established |
| --- | --- |
| Streams have no required concurrency ceiling | **Probably already required, on a reading the leaf flagged as an inference.** `R8.10`'s rate-limit axis says "published: … concurrency separate", and the obligation depends on reading the colon as distributing across the whole list. The leaf labeled this `[INFERENCE]` and called the ambiguity itself a finding; revision 1 promoted it to a flat assertion. The walk should disambiguate `R8.10`'s cell in the same release rather than rely on the reading. |
| *(revision 1 listed a row here about streamed collections lacking a published maximum)* | **Withdrawn as a strawman.** §13.4 makes no such claim; its ceilings entry says only that a held-open stream is in none of `R11.1`'s dimensions. The substantive point survives as `ST-029`. |
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

## Defects the adversarial review found that no leaf caught

These arise from **interactions between proposals** researched independently,
which is the defect class parallel dispatch is most likely to produce and least
likely to catch. Each must be repaired before the proposal it affects ships.

**D1 — `ST-024`'s unbounded declaration voids `ST-025`'s bound.** Take an API
authenticated by a non-expiring server-to-server key, which `R8.5` permits
because it says only that credentials *SHOULD* be scoped and expiring. It
declares its stream unbounded. Then `ST-024` is discharged by the declaration
and requires nothing; `ST-025`'s expiry clause is scoped to credentials that
carry or imply an expiry and does not fire; `ST-025`'s revocation clause is a
SHOULD; and `ST-025`'s fallback — "its maximum duration is then the exposure
window it is publishing" — **has no referent**, because no maximum exists. A
conforming API can hold a stream open indefinitely for a revoked principal
with every clause satisfied. `baseline-04c`'s flat form had a backstop here
("where the credential carries no expiry, the maximum stream duration is the
bound") that the merge dropped. *Repair:* tie the unbounded declaration to
`R13.6`'s own test — "has no normal end" — so a generative completion cannot
claim it, and make "unbounded **and** non-expiring credential" the combination
that must carry an explicit exposure statement.

**Decision (2026-08-10): adopt the repair — gate the claim, then require the
statement.** Both halves ship together, because the hole has two halves.

The **gate**: an API may declare a stream unbounded only where it meets
`R13.6`'s existing test — the stream *has no normal end*, as a watch or an
event tail does. A generative completion plainly has a normal end and
therefore cannot declare itself unbounded to escape `ST-024`'s duration
requirement. This costs no new concept; `R13.6` already carries the test.

The **statement**: an API that is unbounded **and** authenticates with a
non-expiring credential must state its exposure window explicitly, including
when that statement is "indefinite." This forbids nothing — a Kubernetes-shaped
watch on a long-lived key stays conformant — and forces the true exposure into
the open, which is exactly the disclosure `ST-025`'s strongest clause already
requires in the bounded case.

Note the interaction with **Ruling 1**: demoting `ST-025`'s expiry clause to
`SHOULD` widened this hole slightly, because even a credential that *does*
carry an expiry now bounds the stream only at `SHOULD` strength. The statement
requirement is what keeps the exposure visible in both cases.

Restoring `04c`'s flat backstop — no unbounded declaration on a non-expiring
credential — was declined: it re-imports the over-strictness the `ST-024` merge
was made to remove, and would make a shipped and legitimate shape
nonconformant. Fixing only the dangling fallback clause was also declined,
because it leaves the mechanism that produced the hole intact.

**D2 — `ST-024`'s enforcement clause collides with `R13.9`.** Where `streaming`
and `async-operations` are both on, `R13.9` requires the terminal frame to
carry `operation_state` from the operation's documented terminal-state
vocabulary. A stream cut at a duration ceiling while its operation is still
running has no terminal state to report: it is not succeeded, failed, or
canceled. Routing it through an `error` frame does not help, because `R13.9`
binds error terminal frames identically and calling a published, expected limit
an error contradicts `ST-024`'s own reasoning that such a close is a normal
end. *Repair:* define a "stream ended at limit, work still running" ending, or
carve `ST-024` out of `R13.9`.

**D3 — `ST-023`'s tier-3 gate omits authorization.** Tier 3 means
`Cache-Control: public`, which is precisely the directive that re-enables
shared-cache storage for a request carrying `Authorization` under RFC 9111 §3.
Subsequent requests are then answered without the origin running `R8.6`'s
per-object check, so `ST-025`'s revocation posture bounds nothing — the cache
does not know. Ratified `R7.2` independently blocks this, which is why the
proposal is not wrong; but `ST-023` is the text a drafter works from, and
listing two gates while omitting the third invites the misconfiguration.
*Repair:* add `R7.2` as an explicit condition.

**D4 — `ST-026` has an unenumerated fourth case.** `R3.9`'s key-retention floor
is at least 24 hours; the field's retained artifacts run about 5 to 10 minutes.
So for roughly 23 of those 24 hours a keyed repeat arrives when the terminal
state is retained but the replayable representation is gone. `ST-026` requires
delivering a representation that no longer exists and defines no behavior.
`R13.10` has the matching rule for resumption — reject out-of-window with a
defined error — and `ST-026` needs its analogue.

**Decision (2026-08-10) on precedence: the two mechanisms compose.** A request
may carry both an `Idempotency-Key` and a `stream_position`, and no proposal
stated which governs. They answer orthogonal questions, so both do:

- the **key governs execution** — the work is not run again;
- the **position governs delivery** — the server resumes from it, or returns
  `R13.10`'s defined error where the position lies outside the retention
  window.

This matches the only realistic case, which is one coherent request: a client
whose keyed streaming mutation delivered frames 1 through 47 before the
connection dropped wants both *do not re-run* and *continue from 48*.

It also resolves the apparent conflict by construction rather than by a
separate prohibition. `ST-026` otherwise permits replaying from the first
frame; where a position is present, doing so is exactly the silent restart
`R13.10` forbids, so the option simply does not arise.

Rejecting the combination with `400` was declined as punishing the most
natural recovery request a streaming client can make. A document-your-behavior
duty was declined for the reason this walk has declined it elsewhere: it leaves
the client unable to rely on anything.

**Still open at D4:** the retention-mismatch case — a keyed repeat arriving
while the terminal state is retained but the replayable representation has
been collected.

**D5 — new error conditions have no `R5.16` catalog entries.** `R13.7` requires
an in-band error's `type` and `code` to be listed in the `R5.16` catalog. Three
proposals mandate new conditions without supplying entries: `ST-025`'s
credential-expired frame, `ST-028`'s concurrency-versus-rate `code`, and
`ST-026`'s conflict. Without them the rules are unsatisfiable on first
drafting.

**D6 — a sixth unregistered interaction: cancellation.** `R10.2` expresses
cancellation as the `cancel` action on the operation resource, and `R13.9`
makes the stream and the operation one capability with one identity, operation
authoritative. Nothing says what happens to an open stream when its operation
is canceled through the other channel, nor whether a client disconnecting from
the stream cancels the operation. §13.4 does not register it and no proposal
touches it. Answering the five register questions makes it *more* visible, not
less, because `ST-024`, `ST-025`, and `ST-026` each introduce a new way for a
stream to end that `R13.9` must be able to report.

**D7 — `ST-021`'s Compatible entry is vacuous as written.** It permits adding
frame types "where the API has documented its frame-type vocabulary as
growable", but ratified `R13.5` already makes that declaration mandatory, so
every conforming API satisfies the condition by construction. It reconciles the
Kubernetes dissent only for APIs that are already nonconformant. Harmless;
keep it only if the walk wants the reasoning visible.

**A correction in the proposals' favor.** RFC 7009 §2.1 is *not* silent on
revocation timing: "In practice, there could be a propagation delay…
Implementations should minimize that window." That is a missed supporting
citation for `ST-025`'s strongest clause, which requires the revocation posture
to be stated as an upper bound rather than as a claim of immediacy. RFC 9700
and RFC 9068 are genuinely silent on in-progress requests, as the leaf claimed.

## What the ratification walk must decide

Each of `ST-021` through `ST-029` is a separate ruling. The adversarial review's
assessment of how each survives:

| Proposal | Standing after review |
| --- | --- |
| `ST-023` | Ship as written, plus the `R7.2` gate (D3) |
| `ST-028` | Sound; needs a catalog entry (D5) and `R8.10` disambiguation |
| `ST-021` | Substance sound; the field-confirmation paragraph is rewritten above |
| `ST-025` | Disclosure clauses sound; expiry clause is the walk's real judgment call; D1 must be closed |
| `ST-024` | Merge reasoning sound; D1 and D2 must be fixed; jitter clause droppable |
| `ST-027` | Sound; restored to its proposed form in revision 2 |
| `ST-022` | Sound; restored in revision 2, but the terminal-frame defect must be resolved |
| `ST-029` | Sound; smallest change in the set |
| `ST-026` | **Not ratifiable** — evidence falsified, leaf re-running |

## Owner rulings made in advance of the walk (2026-08-10)

Two of the four cross-cutting questions were ruled before the walk, so the walk
inherits them rather than reopening them.

**Ruling 1 — `ST-025`'s expiry clause ships at `SHOULD`, with a research
trigger.** The disclosure clauses ship at `MUST` and are not affected. The
expiry clause is demoted because it overrules the field's clearest precedent —
a Kubernetes watch authenticates once and plausibly outlives its bound token —
on a threat argued from mechanism with no recorded incident, which is the
thin-evidence shape the Phase 6 decision record declined Tier B items to avoid.

**The ruling carries a condition: research this again if the rule becomes
applicable.** "Applicable" means any of the following, and any one of them
fires a leaf reconsidering `MUST`:

| Trigger | Why it changes the answer |
| --- | --- |
| A published incident of data delivered over a stream after the principal's authorization was revoked | Converts the threat from argued-from-mechanism to observed, which is the specific weakness behind the demotion |
| Any reference implementation shipping mid-stream re-evaluation or credential-bound stream termination | Removes the "overrules the field's clearest precedent" objection |
| An IETF or OAuth work item addressing authorization for a request already in progress | Supplies the standards authority that RFC 9700, RFC 9068, and RFC 7009 currently do not |
| Any of OpenAI, Anthropic, or Google Gemini publishing a maximum stream duration or an in-flight revocation posture | Breaks the three-for-three vendor silence the demotion partly rests on |

This trigger is registered with the dated re-check register in
`research/README.md` when `ST-025` is ratified.

**Ruling 2 — cancellation is registered now and researched later.** D6 is added
to §13.4 as a sixth recognized-and-not-yet-ruled interaction, so a reader
hitting it learns the silence is deliberate. No leaf is dispatched for it now,
because no current proposal depends on resolving it and a sixth leaf would
delay a walk already waiting on the `ST-026` re-run.

Two questions remain open for the walk:

1. ~~**Where should the general open-enum hole be fixed?**~~ **Decided
   2026-08-10: fix the class.** `R9.4` gains a general classification —
   removing or renaming a documented open-enum value is breaking — rather than
   `ST-021` patching the frame-type instance alone. Precedent inside `R9.4`
   itself: its frozen list already contains "problem `type`/`code` pairs
   (R5.13.2)", and a problem `code` is an enum-like value rather than a field
   name, so freezing enum values is established here and was simply never
   generalized. The same hole leaves `operation_state` (`R10.1`, made
   cross-channel by `R13.9`) and `R6.7`'s sortable-field set unclassified, and
   the failure mode is identical in all three: a client dispatches on a value
   that moved.

   **Consequence for `ST-021`:** it shrinks to its **terminality** clause,
   which is the genuinely frame-specific part — no generic enum rule can
   express "this value ends the response." The frozen-list entry for
   frame-type names is absorbed by the general rule.

   **Recorded cost:** this generalizes past `baseline-04b`'s evidence, which
   covered frame types only. Mitigated because the general rule classifies a
   change `R9.4`'s own logic already implies rather than creating a new
   obligation, and because it is weaker per instance than `ST-021` was.
2. **Are the register corrections and out-of-scope defects handled in the same
   release, or separately?** They are editorial relative to the rule proposals,
   and several — including Ruling 2's cancellation entry — do not depend on the
   walk at all.
