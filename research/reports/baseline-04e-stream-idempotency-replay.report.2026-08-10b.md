# Baseline 04e — Idempotency-key replay of a streaming request (run **b**)

*Research leaf under `baseline-04` (streaming), answering the fourth row of
`rest-api-standard.md` §13.4 "Known unresolved interactions": **what an
`Idempotency-Key` replay of a streaming request delivers, and whether it may
re-execute the underlying work.** Series `baseline` = prescriptive: this report
**proposes** rule text; it does not ratify it. Ratification is a `decisions/`
record.*

*Second run, 2026-08-10 (suffix `b`). The first run of the same date is
retained unchanged at
`research/reports/baseline-04e-stream-idempotency-replay.report.2026-08-10.md`.
All web access dates are 2026-08-10 unless a row states otherwise. The ratified
record `research/decisions/baseline-04-streaming.decision.md` is treated as
settled and is not reopened.*

*Rules touched: `R3.9` (idempotency keys), `R12.10` (client obligations on a
truncated stream), `R13.2` (streamed/non-streamed negotiation on `Accept`),
`R13.5`/`R13.6` (frame typing and terminal frames), `R13.7` (post-commit error
frames), `R13.9` (stream ↔ operation-resource identity), `R13.10`
(resumption), `R10.9` (`202` operation discovery).*

---

## 0. This run supersedes the first run's central negative

**Read this section before relying on anything in the first run.**

The first run's stated "load-bearing claim" was a universal negative: that no
published API both accepts an idempotency key and streams, and that `R3.9` plus
§13 therefore "manufactures an intersection with zero published precedent."

**That negative is false, and this run withdraws it.** `[FACT]` Replicate's Cog
HTTP API documents both capabilities on one endpoint, in a capability table
whose row reads `PUT /predictions/<prediction_id>` with
`Accept: text/event-stream` → "Streaming, idempotent", and describes that
endpoint as "the idempotent version of the `POST /predictions` endpoint"
(https://raw.githubusercontent.com/replicate/cog/main/docs/http.md, pinned at
commit `966752e9f5f5c165fc5e9618642fd353f0db0e56`, file last revised
2026-07-06; accessed 2026-08-10).

**What follows from the withdrawal, item by item:**

| First-run claim | Status after this run |
| --- | --- |
| "No reference in the field ships the combination" (§1, §3.3 `Both?` column empty in fourteen rows) | **Withdrawn.** The column was empty because the candidate set excluded the software layer where the combination lives. |
| "Nobody has solved this because nobody has built it. That is the reason the standard must rule it rather than cite it." | **Withdrawn.** It has been built. The standard should now rule it *with* a citable implementation, which changes the rule's shape. |
| `ST-E01` clause 1 — a keyed repeat arriving while the original is still executing MUST return `409 Conflict` | **Falsified as a MUST.** The only shipped implementation of the intersection does the opposite: it attaches the caller to the in-flight stream. Revised in §7 to a documented choice among defined behaviors. |
| `ST-E01` clause 2 — a repeat after a terminal state MUST NOT re-execute and MUST deliver the recorded outcome | **Survives, and is now the rule's strongest clause** — but see §5.4: Cog *does* re-execute here, because it retains nothing after terminal, which is evidence about cost rather than about desirability. |
| `ST-E01` clause 3 — documented behavior for an abandoned execution | **Survives unchanged.** Still undocumented everywhere. |
| The `R3.9` "stored response" amendment | **Survives, and is corroborated** (§8). Cog's own behavior instantiates the outcome-not-frames reading. |
| The ≥24 h floor versus the field's minutes-long artifacts | **Survives and strengthens** (§8). Cog's buffer is not merely shorter — it is denominated in *events*, not time. |
| `ST-E02` / `ST-027` — prefer splitting execution from delivery | **Survives as one option, but is no longer the only structural answer** (§7.4). Cog demonstrates a second one. |

**What is built on the falsified negative, and therefore needs revisiting.**
`ST-026` in `docs/reviews/2026-08-10-phase-8-consolidated-proposals.md` (target:
a new `R13.15`) carries the negative as its "Why this is not over-specification"
justification and the `409` as its highest-confidence clause. That entry is
already annotated **NOT RATIFIABLE AS WRITTEN** in the consolidated document.
This report supplies its replacement (§7). **No other file is edited by this
run.** Nothing in `rest-api-standard.md` depends on the negative, because §13.4
records this interaction as *unresolved* rather than resolved — the standard's
released text is unaffected either way.

**One important qualification, developed in §5.6.** The falsifying instance
sits inside a carve-out `R3.9` already grants. `R3.9`'s obligation is the
`Idempotency-Key` **header** on non-idempotent requests, and its stated
exception is "naturally idempotent operations (PUT with a client-supplied ID)"
— which is exactly Cog's design. So the negative is false **as the first run
stated it**, and the first run stated it about "an idempotency key," not about
the header. The narrower negative — that no API carries an `Idempotency-Key`
*header* on a streaming endpoint — was **not** falsified by this run and is
restated with its remaining scope in §6.3. Both facts matter, and the report
keeps them apart rather than collapsing them in either direction.

---

## 1. TL;DR and revised recommendation

**Placeholder — completed after the field sweep.**

---

## 2. What the first run got wrong, and why the search missed it

`[FACT]` The first run's own §8.6 conceded the bound: its negatives "rest on the
specific pages and machine-readable specifications listed in §2," and it
inherited `baseline-02g`'s caveat that absences come "from published
specifications and fetched pages rather than exhaustive site crawls." The
concession was accurate. The error was promoting a bounded negative to a
universal one, and then making it load-bearing.

**Four mechanisms produced the miss, in decreasing order of contribution.**

1. **The candidate set was a set of *businesses*, not a set of *software
   layers*.** `[INFERENCE]` Every one of the fourteen rows in the first run's
   §3.3 is a hosted commercial API or a corporate design guideline. Cog is
   neither: it is the open-source HTTP server that Replicate's model containers
   serve, shipped as a Docker image that anyone can run. An API-design survey
   scoped to vendors' *public product* documentation cannot see it, because the
   intersection lives in the serving layer underneath the product.

2. **The question was asked as "who has an `Idempotency-Key` header?" rather
   than "who deduplicates a client-supplied identifier?"** `[INFERENCE]` The
   first run searched for the Stripe-shaped mechanism it already knew. Cog's
   identifier is a path segment in a `PUT`, and the word "idempotent" appears in
   its docs attached to an endpoint, not to a header. A header-name search
   returns nothing on that page.

3. **Recency.** `[FACT]` The SSE capability landed in Cog on 2026-05-22 in pull
   request #3019, "feat: Expose prediction SSE streams"
   (https://github.com/replicate/cog/pull/3019, merged 2026-05-22T19:50:57Z,
   accessed 2026-08-10) — roughly eleven weeks before the first run. The
   idempotent `PUT` predates it; the *intersection* is new.

4. **Replicate was already inside this project's evidence base, on a different
   axis.** `[FACT]` `baseline-03d` (2026-08-09) cites
   `replicate.com/docs/topics/webhooks/verify-webhook` and lists Replicate among
   verified Standard-Webhooks implementers. `[INFERENCE]` The vendor was
   reachable and had already been read once, for webhooks. Nothing carried that
   familiarity across to the streaming question, because the two leaves searched
   by topic and never by vendor.

**The transferable lesson, stated as method rather than blame.** `[POLICY]` A
universal negative over "the field" is only as strong as the definition of the
field, and this project's default field — eight commercial references plus
three AI providers plus the standards bodies — systematically excludes
self-hosted and open-source infrastructure. Where a future leaf's conclusion
*depends* on a negative, the negative should be stated with its scope attached
("no *commercial hosted API in the reference set*…"), and at least one search
should be run against the mechanism rather than the mechanism's usual name.

---

## 3. Standards-and-currency matrix

**Placeholder — completed after the field sweep.**

---

## 4. The intersection implementations found

**Placeholder — completed after the field sweep.**

---

## 5. Cog design study

Cog is the closest thing the field has to a reference implementation of the
intersection, so this section reads it in detail. **Two independent artifact
classes are used and cited separately:** the published documentation, and the
implementation source plus its integration tests. Where they diverge, §5.4 says
so explicitly.

**Sources for this section.**

| Artifact | URL | Version pinned | Accessed |
| --- | --- | --- | --- |
| `docs/http.md` (documentation) | https://raw.githubusercontent.com/replicate/cog/main/docs/http.md | commit `966752e9f5f5c165fc5e9618642fd353f0db0e56`, file last revised 2026-07-06 | 2026-08-10 |
| `crates/coglet/src/transport/http/routes.rs` (HTTP routing and SSE assembly) | https://github.com/replicate/cog/blob/main/crates/coglet/src/transport/http/routes.rs | repository HEAD `e456e1924c6051d22008761f4457a008a5f5d7b7`, 2026-08-10 | 2026-08-10 |
| `crates/coglet/src/prediction.rs` (replay buffer) | https://github.com/replicate/cog/blob/main/crates/coglet/src/prediction.rs | same HEAD | 2026-08-10 |
| `crates/coglet/src/service.rs` (prediction registry and lifetime) | https://github.com/replicate/cog/blob/main/crates/coglet/src/service.rs | same HEAD | 2026-08-10 |
| `integration-tests/tests/sse_stream_history_capacity.txtar`, `…_disabled.txtar` | https://github.com/replicate/cog/tree/main/integration-tests/tests | same HEAD | 2026-08-10 |
| Pull request #3019, "feat: Expose prediction SSE streams" | https://github.com/replicate/cog/pull/3019 | merged 2026-05-22T19:50:57Z | 2026-08-10 |

Authority class for all six: **E** — shipped implementation documentation and
source. Source code is used here as evidence of *behavior*, never as protocol
authority.

### 5.1 How the prediction is identified: a URI path segment, not a header

`[FACT]` The identifier is the last path segment of the request target:
`PUT /predictions/<prediction_id>`. The docs place the responsibility on the
caller — "Clients are responsible for providing unique prediction IDs" — and
recommend "generating a UUIDv4 or UUIDv7, base32-encoding that value, and
removing padding characters," yielding "a random identifier that is 26 ASCII
characters long."

`[FACT]` The server advertises the shape in its root discovery document as
`"predictions_idempotent_url": "/predictions/{prediction_id}"`, alongside
`predictions_cancel_url`; trainings expose the parallel
`trainings_idempotent_url`.

`[FACT]` The identifier may also appear in the request body, and the **only**
body check performed is agreement with the URI. `create_prediction_idempotent`
in `routes.rs` returns `422 Unprocessable Entity` with the message "prediction
ID must match the ID supplied in the URL" when `body.id` is present and differs
from the path segment. There is no other comparison against the body.

### 5.2 What a repeat does while the original is running

`[FACT]` Documentation: "if a client calls this endpoint more than once with the
same ID (for example, due to a network interruption) while the prediction is
still running, no new prediction is created. Instead, the client receives the
response type requested by the retry: JSON for regular requests or a server-sent
event stream for streaming requests." And, for the streaming case specifically:
"If the prediction is still running, the server returns a stream for the
existing prediction instead of creating a duplicate prediction."

`[FACT]` Source, `create_prediction_idempotent`: the handler looks the ID up in
the live prediction registry. On a hit it branches on the *repeat's* negotiated
response mode — a streaming repeat is answered by subscribing to the existing
prediction's event stream; a non-streaming repeat is answered `202 Accepted`
carrying the prediction's current state snapshot. On a miss it falls through to
ordinary creation.

`[FACT]` Pull request #3019 states the intent in the same terms: "Replay
retained events for clients that reconnect to an in-flight prediction with
`PUT /predictions/{id}` and `Accept: text/event-stream`."

**Three consequences worth naming.** `[INFERENCE]`

- **The idempotency unit is the *work*, not the *response*.** The repeat may ask
  for a different representation than the original request did, and gets it.
  The integration tests exercise exactly this: the original is issued with
  `Prefer: respond-async` (a `202` JSON response) and the repeat with
  `Accept: text/event-stream` (a stream over the same running prediction).
- **This composes with content negotiation rather than fighting it.** `R13.2`
  already puts the streamed/non-streamed choice on `Accept`; Cog applies the
  same rule to a repeat. A standard that mandated "a replay delivers what the
  original delivered" would contradict its own negotiation rule.
- **There is no `409` for a keyed repeat.** Cog does return `409 Conflict`, but
  for an unrelated condition — see §5.5.

### 5.3 The replay buffer: what it retains, and for how long

`[FACT]` Retention is **denominated in events, not in time.** `prediction.rs`
defines `DEFAULT_STREAM_HISTORY_CAPACITY: usize = 1024` and reads the override
from the environment variable `COG_STREAM_HISTORY_CAPACITY`. The documentation
states it plainly: "The replay buffer keeps the most recent 1024 events by
default. Set `COG_STREAM_HISTORY_CAPACITY` to change this limit, or set it to
`0` to disable replay history while keeping live streaming enabled."

`[FACT]` The buffer is a per-prediction ring: on each emitted event, if the
history is at capacity the oldest event is dropped and a `skipped` counter is
incremented, then the new event is appended and also broadcast live. A
subscriber receives a snapshot of the retained history, the current `skipped`
count, and a live receiver.

`[FACT]` **The buffer's lifetime is the prediction's lifetime, and nothing
longer.** The prediction registry is documented in `service.rs` as the "Active
predictions — single source of truth for prediction state," and it is the only
store; `remove_prediction` deletes the entry, and `routes.rs` calls it from the
spawned task immediately after the prediction function returns. A comment in the
stream guard records the resulting race directly: "Prediction cleanup may remove
the service entry before the SSE response finishes draining."

`[FACT]` Live subscribers are separately capped: `MAX_STREAM_SUBSCRIBERS` equals
the channel capacity of 1024, and exceeding it yields `429 Too Many Requests`
with `{"error": "Too many stream subscribers"}`.

`[FACT]` Keep-alives are SSE comment lines on a 15-second interval, emitted by
the framework's `KeepAlive` with the text `keep-alive` — transport filler, not
typed frames, which is the shape `R13.5` already contemplates.

### 5.4 What happens after the buffer is exhausted — and the silent-loss case

This is the most decision-relevant part of the design, and the documentation
tells only half of it.

`[FACT]` **Buffer overflowed (capacity > 0, events dropped).** The stream emits
an `error` event and closes. Documentation: "If earlier events have been dropped
from the replay buffer, the stream emits an `error` event and closes." Source:
when the retained `skipped` count is non-zero the stream yields a single event
named `error` with payload
`{"error": "SSE stream replay truncated; events were dropped", "skipped": N}`
and terminates — the retained prefix is **not** delivered first. The integration
test `sse_stream_history_capacity.txtar` pins this behavior at
`COG_STREAM_HISTORY_CAPACITY=2`, asserting `event: error`, the message text, and
`"skipped":3`.

`[FACT]` **A separate live-lag path** produces the same shape when a connected
subscriber falls behind the broadcast channel: `error` with "SSE stream lagged;
events were dropped" and a `skipped` count, then close. A source comment records
that the fix is known and unbuilt: "In the future, this could become
backpressure or cursor-based replay."

`[FACT]` **Buffer disabled (`COG_STREAM_HISTORY_CAPACITY=0`) — loss is
silent.** With capacity zero the history block is skipped entirely, so the
`skipped` counter is never incremented and the error path never fires. The
integration test `sse_stream_history_disabled.txtar` asserts precisely this: a
repeat that attaches mid-prediction receives `event: completed` with
`"status":"succeeded"`, and the test asserts the **absence** of both
`event: output` and the truncation message. The attaching client is handed a
well-formed terminal frame and never learns that every intermediate frame was
lost.

`[INFERENCE]` **This is a gap `R12.10` cannot currently detect, and it is a
finding this leaf did not previously have.** `R12.10` gives a client exactly one
integrity test: a connection that closes *without* the documented terminal frame
is truncated. An attached stream fails the opposite way — the terminal frame
arrives, so the client concludes "complete result," while an arbitrary number of
intermediate frames were never delivered. Truncation is a missing *suffix*;
attachment produces a missing *prefix or middle*. A client that acted on frames
incrementally — the case `R13.4` says SSE exists to serve — has silently
processed a gapped stream.

`[FACT]` Cog partially mitigates this by accident of payload design rather than
by rule: `output` events carry an `index` member, and the terminal `completed`
event carries the full accumulated `output` array, so a client that compares can
notice. `[FACT]` `log` and `metric` events carry no position of any kind
(`{source, data}` and `{name, value, mode}` respectively), so gaps in those are
undetectable. `[FACT]` And **no** Cog event carries an SSE `id:` field — the
source constructs every event with a name and JSON data only — so
`Last-Event-ID` resumption is not available and replay is keyed entirely by the
request's own identity. `[INFERENCE]` This leaves `survey-08`'s verified
negative on `id:`/`Last-Event-ID` intact and gives no reason to reopen the
ratified `R13.10`.

### 5.5 The `409` Cog does return is not an idempotency conflict

`[FACT]` Cog's documented `409 Conflict` is concurrency-slot exhaustion: "By
default, the server runs one prediction at a time, but this can be increased
with `@cog.concurrent(max=N)`. When all prediction slots are in use, the server
returns `409 Conflict`." The source confirms the semantics — the only `409` in
the routing layer is raised on a `CreatePredictionError::AtCapacity` with the
body `{"error": "At capacity - all prediction slots busy", "status": "failed"}`.

`[POLICY]` **This must not be counted as field support for the Stripe-shaped
`409`.** It is a capacity signal on a *new* prediction, not a conflict signal on
a *repeated* one; a keyed repeat of a running prediction never reaches that
branch, because the registry hit returns first. The first run drew the same
distinction for Twilio's `409`, and it is preserved here.

### 5.6 How Cog measures against `R3.9`, clause by clause

This is the comparison that determines what the falsification actually costs the
standard.

| `R3.9` obligation | Cog | Verdict |
| --- | --- | --- |
| Key carried in the `Idempotency-Key` request header | Client-supplied ID as a URI path segment on a `PUT`; no `Idempotency-Key` header anywhere in the docs or source | **Different mechanism** |
| Server fingerprints the request payload | No payload comparison; only `body.id` versus URI id, else `422` | **Not implemented** |
| Rejects a reused key carrying a different payload | A repeat with the same ID and a *different* `input` attaches to the original and the new `input` is ignored | **Contradicts** |
| Replays the stored response for a genuine retry | Only while the work is in flight | **Partial** |
| Stated retention window at least 24 hours | Retention ends when the prediction reaches a terminal state; nothing is kept | **Contradicts** |
| Exception for "naturally idempotent operations (PUT with a client-supplied ID)" | Exactly this | **Cog is the exception, not the rule** |

`[INFERENCE]` **Cog is therefore not a conforming implementation of `R3.9`; it
is an implementation of the carve-out `R3.9` already grants.** Two things follow,
and they pull in opposite directions, so both are stated.

- *Against the first run:* its negative was asserted about "an idempotency key,"
  and Cog is an idempotency key by any ordinary reading — the docs call the
  endpoint idempotent, the discovery document calls the URL
  `predictions_idempotent_url`, and its purpose is stated as safe retry. The
  negative as written is false and the first run's reasoning from it collapses.
- *For the first run's conclusion, partially:* the narrower claim that no
  surveyed API carries an `Idempotency-Key` **header** on a streaming endpoint
  survives this run's search. `R3.9`'s specific mandate still has no shipped
  streaming exemplar.

`[POLICY]` The honest synthesis is that the standard's gap is real but its shape
was wrong. The field has not declined to solve the problem; it has solved it by
**moving the identifier into the request target and making the method
idempotent** — which is the one route `R3.9` explicitly blesses and then stops
describing. §7 builds the revised rule on that observation.

### 5.7 Relationship to `Prefer: respond-async`, the `202` path, and `R10.9`

`[FACT]` Cog exposes three response modes on both create endpoints, selected by
request headers: no header → synchronous JSON; `Prefer: respond-async` → `202
Accepted` with a prediction object in status `starting`;
`Accept: text/event-stream` → `200 OK` held open as an SSE stream.

`[FACT]` **The `202` path has no pollable operation resource.** The
documentation states it as a limitation: "For JSON responses, the only supported
way to receive updates on the status of predictions started asynchronously is
using webhooks. Polling for prediction status is not currently supported." There
is no `GET /predictions/<id>`.

`[INFERENCE]` **This inverts the usual relationship, and it is why Cog reached
attach-to-stream rather than the split the first run recommended.** In OpenAI's
`background: true` design the operation resource is authoritative and the stream
is a view over it, so recovery is a safe `GET`. Cog has no such resource, so the
*only* way back to a running prediction is to re-issue the idempotent `PUT` —
attachment is not a stylistic preference but the sole available channel. `[FACT]`
Cancellation is the one exception: `POST /predictions/<id>/cancel` addresses the
prediction by ID and answers `200` or `404`, which is `R10.9`'s `cancel` action
without the operation resource around it.

`[INFERENCE]` Measured against `R13.9`, Cog is out of scope rather than
non-conforming: `R13.9` binds only where a capability is exposed *both* as a
stream and as an operation resource, and Cog exposes no operation resource. It
is, however, a clean demonstration that the two channels the standard assumes
are not always both present.

### 5.8 Does Cog re-execute in any case? Yes — and the case matters

`[FACT]` After a prediction reaches a terminal state — `succeeded`, `failed`, or
`canceled` — its registry entry is removed immediately, and the registry is the
only store. A subsequent `PUT /predictions/<same-id>` therefore finds nothing,
falls through the registry check, and **creates and executes a new prediction
under the same ID.**

`[INFERENCE]` So Cog's guarantee is narrower than "idempotent" suggests, and the
documentation's own wording is careful about it in a way a reader may miss: the
promise is scoped "**while the prediction is still running**." Cog deduplicates
concurrent retries; it does not deduplicate a retry that arrives after the work
finished. `R3.9`'s ≥24-hour floor asks for the second guarantee, and Cog
provides zero seconds of it.

`[POLICY]` **This is evidence about cost, not about desirability**, and the
report declines to read it as either an endorsement of re-execution or a refutation of
`R3.9`'s floor. What it shows is that the only implementation to reach the
intersection stopped at the boundary where retention becomes an obligation
outliving the request — the same boundary §8 identifies from the other side.

---

## 6. Evidence for and against each candidate rule shape

**Placeholder — completed after the field sweep.**

---

## 7. Revised proposed rule text

**Placeholder — completed after the field sweep.**

---

## 8. What survived from the first run

**Placeholder — completed after the field sweep.**

---

## 9. Declined alternatives

**Placeholder — completed after the field sweep.**

---

## 10. What could not be verified

**Placeholder — completed after the field sweep.**

---

## 11. Sources

**Placeholder — completed after the field sweep.**
