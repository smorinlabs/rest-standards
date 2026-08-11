# Phase 7 skill self-test — `rest-standards` audit mode against Appendix E

**Date:** 2026-08-10 · **Task:** Phase 7 · **Gate:** F evidence record ·
**Standard version under test:** v1.1.2

> **This document supersedes the v1.0.0 run.** The skill was first self-tested
> against `rest-api-standard.md` at v1.0.0, which produced
> `92 applicable MUSTs: 52 pass, 0 fail, 40 unverified`. The standard has since
> moved to v1.1.2, adding **§13 (streaming)** — 11 new §13 rules plus `R12.10`,
> and streaming scope added to 9 existing rules — and the skill has been amended
> for it. That earlier result is no longer the record; this one is. What changed
> between the two runs, and why, is §7.

Appendix E of `rest-api-standard.md` is a worked example — the Bloom Orders
API — already annotated with rule IDs and carrying its own conformance note.
That makes it a self-test with a known-correct answer: running the
`rest-standards` skill's `audit` mode against it should reproduce the
appendix's own reading, and any place it does not is either a skill defect or
a genuine standard finding.

**Result:** no rule-level contradiction. The skill reproduced every verdict the
appendix annotates. The run exposed **one skill defect** (a procedural gap in
`references/audit.md`, fixed and re-run), **confirmed the two v1.0.0 defect
fixes hold under a real input**, and carries **four standard findings** about
the standard's annotation and drafting apparatus, proposed here and not
applied.

---

## 1. How the skill was executed

The skill was run as documented, not from memory of REST practice.

| Step | `SKILL.md` says | What was done |
|---|---|---|
| 1 | Locate and version-pin the standard | `grep -m1 -oE '\*\*Version [0-9]+\.[0-9]+\.[0-9]+'` → **v1.1.2** |
| 2 | Get the live section map; never read the standard whole | `grep -n '^## '` → 25 top-level headings, of which 13 are numbered normative sections |
| 3 | Identify the mode | `audit` — an existing, documented API assessed for conformance |
| 4 | Settle tier, switches, plane | `references/scoping.md` (§2 below) |
| 5 | Execute the mode procedure | `references/audit.md`, sweeping §1–§13 in section order per `references/review.md` |
| 6 | Cite `R#.#` only | Validated mechanically (§6 below) |

**Files loaded:** `SKILL.md`, `references/scoping.md`, `references/audit.md`,
and `references/review.md` (audit mode delegates its findings table and its
section-order sweep to review mode).

**Sections read live during the sweep**, in the order `grep -n '^## [0-9]'`
returned them: §1 (lines 32–411), §2 (413–550), §3 (552–686), §4 (688–849),
§5 (851–1066), §6 (1068–1189), §7 (1191–1247), §8 (1249–1360),
§9 (1362–1453), §10 (1455–1588), §11 (1590–1681), §12 (1683–1787),
§13 (1789–2091). Appendix G's live-probe table (2984–3033) was read as
`audit.md` §3 requires. No rule text was quoted from memory.

**The section list was derived live, not remembered.** The skill removed its
hardcoded snapshots of the standard's shape in commits `6fdc7a6`, `6189dad`,
and `c09cb72`; this run confirms the grepped sweep checklist reaches §13 with
no edit to the skill. §13 appeared in the sweep because the standard lists it,
not because the skill names it.

---

## 2. Scope record

    Standard: rest-api-standard v1.1.2
    Tier: public
    Switches: webhooks=on, async-operations=on, streaming=on,
      bulk-operations=off (off because "imports run through the async
      export/import operations; no synchronous bulk endpoint is offered")
    Evidence planes available: contract, source
    Evidence planes unavailable: runtime — no deployment exists; no HTTP
      request was issued

`references/scoping.md` calls for one `AskUserQuestion` to settle tier. There
is no user in this run, and none is needed: **the audited artifact declares
its own scope.** Appendix E.1 is Bloom's conformance note, rendered from the
§1.9 template, and it states the tier and all four switch states verbatim.
The scoping answers were taken from it rather than asked.

The switch vocabulary was read live from §1.8 — `webhooks` ·
`async-operations` · `bulk-operations` · `streaming` — as
`scoping.md` §2 requires, not assumed from the v1.0.0 vocabulary.

Tier does not change the applicable rule set here. §1.7 permits later sections
to scope rules by tier; grepping the live standard for a tier annotation on
any rule returns only R1.5's own hypothetical example, so **no rule in v1.1.2
carries a tier scope** and `public` neither adds nor relaxes anything.

### The `streaming` switch — settled `on`

**`streaming=on`**, declared in E.1 and demonstrated by E.11, which serves
`GET /v1/order-exports/{export_id}/events` as `200 OK` with
`Content-Type: text/event-stream`. No R1.6 reason is owed: R1.6 requires a
stated reason only for a switch declared **off**. Bulk operations remain the
only off switch, and its reason is quoted in the scope record above.

Because the switch is on, §13 was swept in full and its rules enter the
applicable set. `R13.9` binds as well: §1.8 makes it conditional on
`streaming` **and** `async-operations`, and Bloom declares both on.

### Guard rules bind regardless of the switch

§1.8 names two rules whose whole purpose is to define what an API *without* a
capability must do, and exempts them from that capability's switch: **`R1.9`**
(the `dry_run` rejection guard) and **`R13.3`** (the `stream` rejection guard).
Both bind **per endpoint**, not per API.

`streaming=on` does not retire `R13.3`. §13's switch-scope paragraph is
explicit that R13.3 "binds every endpoint that does not implement streaming,
whatever the API's `streaming` switch says … An API that streams on some
endpoints still owes the guard on the rest." Bloom streams on exactly one
exhibited path; `/v1/orders`, `/v1/order-exports`, `/v1/operations/{id}`, and
every action sub-path owe the guard. `R13.3` is therefore counted in `N` and
scored — it lands `unverified` (§3), never skipped.

This is the path `references/review.md`'s cross-cutting trap and `SKILL.md`'s
depth-scaling note were amended to cover, and it fired correctly. One
observation about the amendment's phrasing, which cost no verdict here, is
recorded as **SF-4** in §5.

### What each plane is, in this run

| Plane | Supplied by | Reaches |
|---|---|---|
| contract | The literal HTTP request/response exchanges in E.2–E.11 | Wire-observable shape: status codes, headers, media types, body structure, stream frames |
| source | Appendix E's "reading" paragraphs, which state implementation and documentation behavior the exchanges cannot show | Idempotency replay semantics, documented default sort order, webhook consumer obligations, the stream's retention window |
| runtime | — not available — | Nothing |

**Spectral could not run.** `references/audit.md` names
`conformance/spectral.yaml` as the contract plane's checker, and Spectral
needs a machine-readable document. Appendix E publishes none. The contract
plane is still available — the exchanges *are* documented contract — so the
contract was read directly and the missing Spectral run is recorded as missing,
never as clean. That the worked example exhibits no OpenAPI document is itself
reported below under R4.1. This is the corrected procedure from v1.0.0's skill
defect SD-1, and §5 records that it held.

### Evidence policy

Carried unchanged from the v1.0.0 run, so the two are comparable, and fixed
before the sweep so the verdicts are reproducible:

1. **Prose counts.** Assertions in Appendix E's "reading" paragraphs are
   documented behavior on the contract and source planes. The appendix says so
   explicitly for headers it omits: the preamble states that every response
   also carries `Access-Control-Expose-Headers` (R4.17), and E.10 states that
   the always-on R7.1 and R11.7 headers are excerpted away for brevity.
2. **Negative obligations** (MUST NOT, "never") are evaluated over the
   exhibited surface, and only where that surface actually contains instances
   of the governed class. No counterexample among exhibited instances ⇒ `pass`,
   scoped to what is exhibited. No instances at all ⇒ `unverified`.
3. **Positive obligations** are `pass` only where the appendix exhibits or
   asserts the required behavior. Silence is `unverified`, never an inferred
   pass.
4. **Rules needing an unsupplied plane** — runtime, Bloom's separate OpenAPI
   document, Bloom's implementation source, Bloom's published client and
   operational documentation — are `unverified`, naming the missing plane.

---

## 3. Conformance summary

    104 applicable MUSTs: 55 pass, 0 fail, 49 unverified

This matches the `<N> applicable MUSTs: <P> pass, <F> fail, <U> unverified`
format required by `references/audit.md` § Report.

**How `N` was derived.** `audit.md` § Report requires the subtraction to be
shown from the live rule count, not a remembered `N` restated. The old figure
of 92 was **not** adjusted; the count was rebuilt from scratch against v1.1.2.

| Step | Count |
|---|---|
| Rules defined in Part I (`grep -oE '^\*\*R[0-9]+\.[0-9]+' \| sort -u`) | 139 |
| — carrying a capitalized MUST / MUST NOT / REQUIRED / SHALL clause (R1.1) | 108 |
| — less rules binding the standard's own drafting, not the API: R1.1, R1.4 | −2 |
| — less rules scoped to a switch declared off: R10.4 (§10.2), R5.8 (bulk endpoints) | −2 |
| **N — applicable MUSTs** | **104** |

Three checks on that number:

- **The subtraction is the same shape as v1.0.0's**, with only the rule count
  moving: 139 − 31 − 2 − 2, against 127 − 31 − 2 − 2. The 31 rules excluded as
  SHOULD-, MAY-, or keyword-free is coincidentally identical in size to
  v1.0.0's, because **all 12 new rules carry a MUST-class keyword**. It is the
  same 31 rules, not merely the same count.
- **`104 = 92 + 12`** — the v1.0.0 applicable set recomputed from the v1.1.2
  file, with the 12 new rules removed, reproduces 92 exactly. The two runs
  therefore differ by the new rules and nothing else in the counting.
- **No new rule is removed by an off switch.** `bulk-operations` is the only
  off switch, and §10's scope statement assigns it §10.2 alone (R10.4); R5.8
  is bulk-scoped by its own text. No §13 rule is scoped to `bulk-operations`.

Two §13 rules deserve their counting stated explicitly, because both invite an
incorrect exclusion:

- **`R13.3` is in `N` although it is a guard rule.** `audit.md`'s test (c)
  excludes a rule "scoped to a switch declared off"; R13.3 is scoped to no
  switch at all, so nothing removes it. It would have stayed in `N` even had
  `streaming` been off.
- **`R13.11` (long-polling) is in `N` although Bloom exhibits no long-poll
  endpoint.** It is scoped by `streaming`, which is on. Absence of an instance
  is an evidence fact that makes it `unverified`; it is not an applicability
  fact that removes it from the denominator.

The 31 rules excluded as SHOULD-, MAY-, or keyword-free are assessed in the
sweep and appear in the table below where Appendix E annotates them; they are
not counted in `N`.

**Zero failures is the expected answer.** Appendix E is the standard's own
worked example and its conformance note records `Deviations: none`. A failure
would have meant the standard contradicts itself. §13 was the live risk here —
a new section can contradict an old one — and it did not: E.11's `200` on a
stream that later fails is explicitly reconciled with R5.1 by R5.1's own
streaming scope, and E.11's `Cache-Control: private, no-store` satisfies R7.1
and R7.2 while §13.4 records the general streaming cache posture as
deliberately unruled.

**`U = 49` is the honest reach of the evidence, not a defect in Bloom.**
Appendix E is a set of excerpts, not a deployed API with published
documentation. 49 applicable MUSTs have at least one clause that no supplied
plane reaches — 17 because the runtime plane is absent, the remaining 32
because Bloom's OpenAPI document, implementation source, and published
operational and client documentation are outside the appendix.

### N/A declarations (R1.6)

- `bulk-operations`: **off** — "imports run through the async export/import
  operations; no synchronous bulk endpoint is offered". Removes **R10.4**
  (§10.2) and **R5.8**, whose obligations bind bulk endpoints only.

That reason is the one R1.6 demands. `webhooks`, `async-operations`, and
`streaming` are all on and owe no reason.

### Unverified rules needing the runtime plane

Each is reported `unverified` with the `curl` that would settle it, per
`audit.md` § Report. **None was run** — no deployment exists, and the gate in
`audit.md` §3 was never opened.

`audit.md` §3 ends with "Read Appendix G for the live probe table; do not work
from the list above." Appendix G was read: it now carries **19 rows**, the 13
of v1.0.0 plus 6 streaming fixtures. Every row below was checked against it.
The **Source** column records whether the probe is Appendix G's own definition
or a construction beyond G's table, and the **Tier** column records the
step-3 classification, which `audit.md` §3.3 requires to be made per row from
what the request does.

| Rule | What runtime would settle | Tier | Probe | Source |
|---|---|---|---|---|
| R2.6 | Trailing slash returns `308` to the canonical form | read-only | `curl -sSi https://api.example.com/v1/orders/` | G "Trailing slash" |
| R1.9 | `dry_run` on an endpoint not implementing it is rejected `400` | mutating | `curl -sSi -X POST 'https://api.example.com/v1/orders?dry_run=true' -H 'Authorization: Bearer $T' -d '{}'` — **gated**: executes for real precisely when the guard is the thing that is broken; needs the §3.4 second confirmation naming a disposable fixture | G "Rehearsal guard" |
| R3.7, R5.11 | Unsupported PATCH media type rejected `415`; `Accept-Patch` advertised | mutating | `curl -sSi -X PATCH https://api.example.com/v1/orders/ord_000example -H 'Content-Type: application/json' -H 'Authorization: Bearer $T' -d '{}'` — **gated**: the patch applies for real if the `415` guard does not hold | G "PATCH media type" |
| R5.11 | An unimplemented method on a real path returns `405` carrying `Allow` | mutating | `curl -sSi -X <unimplemented-method> https://api.example.com/v1/orders/ord_000example -H 'Authorization: Bearer $T'` — **gated**: a method assumed unimplemented and turning out to be implemented is a real mutation | G "Unknown method" |
| R5.9 | `401` unauthenticated vs `403` authenticated-unauthorized | read-only | `curl -sSi https://api.example.com/v1/orders` (no credential) | G "Auth split" |
| R13.3 | `stream` on an endpoint without streaming is rejected `400`, never silently answered non-streamed | read-only | `curl -sSi 'https://api.example.com/v1/orders?stream=true' -H 'Authorization: Bearer $T'` — aimed at a safe method, so no state changes even when the guard fails; aimed at a mutating endpoint the same row is **mutating** and gated | G "Stream guard" |
| R13.2, R4.11 | Streamed/non-streamed choice made on `Accept`, response carries `Vary: Accept` | **unbounded** | `curl -N --max-time 10 -sSi https://api.example.com/v1/order-exports/exp_000example -H 'Accept: text/event-stream' -H 'Authorization: Bearer $T'` — **gated**: opens a stream, so it does not return on its own; only the response headers are needed, so 10 s is an ample bound and the cost ceiling is the frames delivered inside it | G "Stream negotiation" |
| R13.6 | A documented terminal frame arrives before the connection closes | **unbounded** | `curl -N --max-time 120 -sS https://api.example.com/v1/order-exports/exp_000example/events -H 'Accept: text/event-stream' -H 'Authorization: Bearer $T'` — **gated**: G says to *consume the stream to completion*, and precisely when R13.6 fails there is no completion to reach; a missing terminal frame is indistinguishable from a stream still in progress. Cut at the bound and reported unverified, never run longer | G "Stream termination" |
| R13.9 | Frames carry `operation_id`; the terminal state matches the operation resource | **unbounded** + mutating | `curl -N --max-time 120 -sS https://api.example.com/v1/order-exports/<disposable-export>/events -H 'Accept: text/event-stream' -H 'Authorization: Bearer $T'` — **gated twice**: it holds a connection open *and* starting the capability creates an export; needs the disposable fixture named and the bound agreed | G "Stream identity" |
| R13.11 | An expired long-poll hold returns `200` with an empty result and a `cursor`, never `204` | **unbounded** | Not constructible against Bloom: G's "Long-poll expiry" row says to *hold past the documented maximum*, and Bloom exhibits no long-polling endpoint and documents no maximum hold duration. The bound cannot even be chosen without the documented maximum the rule's own first clause requires | G "Long-poll expiry" |
| R8.1 | TLS 1.2+ served, TLS 1.0/1.1 rejected | read-only | `curl -sSI --tlsv1.1 --tls-max 1.1 https://api.example.com/v1/orders` (must fail) | beyond G |
| R11.5 | `503` carries `Retry-After` | — | Appendix G defines **no** probe for the `503` clause, and none is constructible: `503` cannot be induced by a well-formed request. Settled by observation under real capacity overload, or by reading the error-path source | beyond G |
| R5.5 | `307`/`308` preserve method and body | read-only | `curl -sSi -X POST https://api.example.com/v1/orders/ -H 'Authorization: Bearer $T'` | beyond G |
| R7.4 clause 2 | No unfiltered collection-level DELETE is offered | mutating | `curl -sSi -X DELETE https://api.example.com/v1/orders -H 'Authorization: Bearer $T'` — **gated**: destructive precisely when the API fails the check. G's "Destructive guard" row probes clause 1 only | beyond G |
| R11.4 | Any proprietary quota headers are documented | read-only | `curl -sSiD- https://api.example.com/v1/orders -H 'Authorization: Bearer $T'` | beyond G |
| R13.8 | A pre-commit error on a streaming request is served as `application/problem+json` | mutating | No G row exists. Requires inducing a failure *before* the status is committed — a malformed streaming request, or a fault injected in the source — which is a construction, not a probe of a healthy deployment | beyond G |

Every `curl` above carries its tier's precautions inline, per the corrected
`audit.md` §3.6 (skill defect **SD-3**, §5). Appendix G's remaining probes —
empty collection (R6.2), existence masking (R5.10), quota (R11.2), error
negotiation (R5.12/R5.13), correlation (R11.7), 202 discovery (R10.9), cache
posture (R7.1–R7.3), and resume window (R13.10) — are not listed here because
the contract and source planes already settled those rules; a runtime probe
would only corroborate a verdict, not supply a missing one.

### Unverified rules needing a document Appendix E does not contain

The remaining 32 need Bloom's OpenAPI document (R4.1, R4.2, R4.8, R4.9), its
implementation source (R8.6–R8.9, R8.11), or its published operational and
client documentation (R3.1, R3.9, R5.16, R6.5, R10.2, R10.6, R11.1, R11.6,
R11.8, R12.1, R12.2, R12.3, R12.4, R12.7, R12.8, R12.10, R13.4, R13.5, and the
conditional clauses of R3.8, R3.12, R4.15, R8.4, R8.10).

R13.4 and R13.5 are the §13 additions to this group, and both are documentation
obligations rather than wire facts:

- **R13.4** — Bloom adopts SSE, so it "MUST document that the media type has no
  IANA registration, and MUST NOT describe it as an IANA-registered media
  type." The prohibition passes over the exhibited surface: nothing in
  Appendix E describes `text/event-stream` as registered. The positive
  documentation obligation is never exhibited. §1.10 records the registration
  gap, but that is the *standard* documenting it, not Bloom.
- **R13.5** — every frame carries a documented type, which E.11 exhibits and
  asserts. Two further MUST clauses are unreached: the API "MUST document its
  full frame-type vocabulary **and state that the vocabulary may grow**", and
  it "MUST document whether it emits keep-alives and in what form". Neither
  statement appears anywhere in Appendix E. The growth statement is not
  decorative — R13.5 says it is what makes R12.10's unknown-type tolerance
  dischargeable.

---

## 4. Comparison against the answer key

The answer key is the **63** rule IDs Appendix E annotates, extracted with
`grep -oE 'R[0-9]+\.[0-9]+' | sort -u`. Every one is a rule defined in the
standard (§6). Sub-rule references collapse under that grep: `R5.13.1` and
`R5.13.4` appear as R5.13, `R10.7.1` as R10.7. Ranges written in `Exercises`
headers (`R4.4–R4.7`, `R6.5–R6.8`, `R13.4–R13.7`) were expanded by hand, since
the grep sees only a range's endpoints.

**Outcome vocabulary.** `agree` — the skill's verdict matches what the appendix
demonstrates. `consistent (partial)` — the appendix names the rule as
exercised, the skill confirms the demonstrated clause, and the rule scores
`unverified` because *another* MUST clause of the same rule is unreachable on
the supplied planes. That is an evidence-plane limit both sides agree on, not
a contradiction, so it is not classified as a disagreement. `disagree` — the
skill's verdict contradicts the appendix.

**Citation kinds.** E.11 introduced a citation shape the v1.0.0 key did not
contain: rules cited to explain what does *not* apply, or to name a hazard
avoided. `R4.10` is the clearest — E.11 cites it to say "R4.10 does not reach
it" of the `/events` resource. Such rows are marked **cross-ref** in the
"Appendix E says" column. A cross-ref row's verdict is the skill's own reading
of the rule against the whole appendix; it is not a claim that the citing block
demonstrates the rule.

The **Count** column marks whether the rule is one of the 104 applicable MUSTs
(`✔`) or is excluded from `N` as SHOULD / MAY / keyword-free (`—`).

| Rule | Count | Appendix E says | Skill reported | Outcome |
|---|---|---|---|---|
| R1.8 | ✔ | E.11 **cross-ref**: `part_url` is deliberately not named `url`, which R10.9 reserves in an operation body | pass — every reserved name in the appendix carries its registered meaning; `operation_id`, `operation_state`, `stream_position`, and the `error` frame type are all used as §1.10 registers them, and no new `X-` name appears | agree |
| R2.3 | ✔ | E.11 **cross-ref**: reusing `url` for a part file would be the same-name-different-concept hazard R2.3 exists to prevent | pass — one noun per concept across paths, parameters, and body fields; `order`, `customer`, `operation`, `export` each carry one name throughout | agree |
| R2.9 | ✔ | E.4: filters, sort, pagination travel as query parameters | pass — E.4's query string carries all four modifiers; no modifier in a path segment | agree |
| R2.11 | ✔ | E.6: `POST /v1/orders/{id}/cancel`, `200` with the mutated representation | pass — action sub-path form and synchronous response shape both as R2.11 specifies. The clause added in v1.1.0 (a streaming long-running action returns `200`, not `202`) carries no capitalized keyword and so gates no verdict; E.11 conforms to it regardless | agree |
| R2.13 | ✔ | E.6: "There is no `POST /v1/orders/cancel`" | pass — no collection-level action anywhere in the appendix | agree |
| R3.7 | ✔ | E.5: `application/merge-patch+json`; `null` deletes | **unverified** — media-type clause passes; the `Accept-Patch` advertisement and the `415` rejection are never exhibited | consistent (partial) |
| R3.8 | ✔ | E.5: Merge Patch is the one exception to null-equals-absent | **unverified** — null-equivalence asserted; "a `null` targeting a non-deletable field MUST return `400`" is never exhibited | consistent (partial) |
| R3.9 | ✔ | E.2: replay with same payload returns the stored response; different payload rejected | **unverified** — key and replay semantics documented; the MUST that "the stated retention window is at least 24 hours" is never stated | consistent (partial) |
| R3.10 | ✔ | E.2: strong `ETag` emitted | pass — `"v1-000example"` is strong (no `W/`) and is consumed by E.5's `If-Match` | agree |
| R3.11 | — | E.5: `If-Match` demanded, `428` when absent | pass (SHOULD) | agree |
| R4.4 | ✔ | E.2: the body is snake_case | pass — bodies and query parameters snake_case across E.2, E.4, E.7, E.8, and now E.11's frame payloads (`operation_id`, `stream_position`, `total_estimate`, `rows_total`, `part_url`, `next_cursor`), which R13.5 binds to §4 unchanged | agree |
| R4.5 | ✔ | E.2: `id` is a string | pass — every identifier is a JSON string, including E.11's `operation_id` | agree |
| R4.6 | ✔ | E.2: `created_at` carries an explicit offset; `deliver_on` is the date-only exception | pass | agree |
| R4.7 | ✔ | E.2: `amount` 4599 with `"usd"` is $45.99 in minor units | pass — integer minor units with the REQUIRED sibling `currency` | agree |
| R4.10 | ✔ | E.11 **cross-ref**: `/events` performs no selection, so R4.10 does not reach it | pass — E.4's GET carries no `Accept` and receives `application/json`; no `format` query parameter appears anywhere. The cross-ref itself is correct: R13.2's second limb is a distinct resource, not a negotiation | agree |
| R4.17 | ✔ | Preamble: every response carries `Access-Control-Expose-Headers` | pass — asserted in prose per evidence policy 1 | agree |
| R5.1 | ✔ | E.6, E.7: status matches the outcome. E.11: "R5.1 is satisfied, not violated" for the `200` on a stream that later fails | pass — `201`/`422`/`200`/`428`/`204`/`202`/`429` all match their registered semantics, and the streamed `200` matches the outcome known when it was generated, which is what R5.1's v1.1.0 streaming scope states | agree |
| R5.3 | — | E.3: `422` for a well-formed but semantically invalid request | pass (SHOULD) | agree |
| R5.6 | ✔ | E.2: `201 Created` with `Location` | pass | agree |
| R5.7 | — | E.5: a matching `If-Match` delete returns `204 No Content` | pass (no RFC 2119 keyword — see SF-3) | agree |
| R5.11 | ✔ | E.5: `428 Precondition Required` where `If-Match` is demanded and absent | **unverified** — the `428` clause passes; the `405`+`Allow` and `415` clauses are never exhibited | consistent (partial) |
| R5.12 | ✔ | E.3: `application/problem+json`. E.11 **cross-ref**: the error frame is the second carve-out | pass — all three error responses are problem documents, and the stream error frame takes the carve-out the rule itself names rather than violating it | agree |
| R5.13 | ✔ | E.3: `code` maps to `type` by the fixed template; the human link lives in `documentation`. E.11: the frame's problem object carries the same members except `status` | pass — verified on four pairs now: `validation_failed`, `precondition_required`, `rate_limit_exceeded`, and E.11's `export_source_unavailable` → `…/export-source-unavailable`; `type` present on all, no `about:blank`, base domain provider-controlled. Points 2 (immutability) and 3 (clients must not depend on `type` resolving) are unexhibited: point 2 is a negative obligation with instances present and no counterexample, and point 3 binds clients through R12.7, separately scored `unverified` | agree |
| R5.15 | — | E.3: field failures ride `errors[]` with JSON Pointers | pass (SHOULD) — `pointer`, `code`, `detail` all present | agree |
| R5.16 | ✔ | E.11 **cross-ref**: the error frame's `code` is "listed in Bloom's R5.16 catalog" | **unverified** — the assertion establishes that a catalog exists and contains this pair; the MUST is that it catalogs **every** `type`/`code` pair the API can return, and completeness is never asserted or exhibited | consistent (partial) |
| R6.1 | ✔ | E.4: the envelope carries `items` and `next_cursor` | pass — E.4 exhibits the envelope, and E.11's terminal frame carries `next_cursor`, which is the streamed form the rule's v1.1.0 scope names | agree |
| R6.2 | — | E.4: an empty result is `200` with `"items": []` | pass (no RFC 2119 keyword — see SF-3) | agree |
| R6.3 | — | E.4: the cursor is opaque | pass (SHOULD) | agree |
| R6.5 | ✔ | E.4: the next page is `?cursor=…&limit=50` | **unverified** — the `cursor`/`limit` naming clause passes; the MUST that each collection documents its default and maximum `limit` is never exhibited | consistent (partial) |
| R6.6 | ✔ | E.4: documented default order is `-created_at` with `id` as tiebreak | pass — the soundness precondition R6.3 depends on, and which R6.1's streaming scope says a streamed collection owes more strictly still | agree |
| R6.7 | — | E.4: `-created_at` descending | pass (MAY) | agree |
| R6.8 | — | E.4: equality plus a bracket range filter, AND-combined | pass (no RFC 2119 keyword — see SF-3) | agree |
| R7.1 | ✔ | E.2 header list; E.10 notes it is always on | pass — every exhibited response carries an explicit `Cache-Control`, including E.11's stream (`private, no-store`) | agree |
| R7.2 | ✔ | E.2 **reading**: the response is explicitly non-cacheable-shared | pass — `private, no-cache` on authenticated data; `no-store` on errors and on the stream — but see **SF-1** | agree |
| R7.3 | — | E.2: `private, no-cache` posture | pass (no RFC 2119 keyword — see SF-3) — tier 1 of the three-tier posture, `no-store` reserved rather than blanket-applied. §13.4 records the streaming posture as deliberately unruled, so E.11's `no-store` is neither conformance nor deviation | agree |
| R7.4 | ✔ | E.5: DELETE without its precondition is refused `428` | **unverified** — clause 1 (DELETE MUST require `If-Match`) passes; clause 2 (no unfiltered collection-level DELETE) has no instance in evidence | consistent (partial) |
| R8.1 | ✔ | E.2: "the request authenticates with a bearer token over TLS" | **unverified** — runtime; the TLS version floor and the rejection of TLS 1.0/1.1 are not observable in an excerpt | consistent (partial) |
| R8.3 | ✔ | E.2: scoped API keys server-to-server, OAuth for third-party on a user's behalf | pass — matches the authority boundary R8.3 draws | agree |
| R9.5 | ✔ | E.10: `Deprecation` is a structured-field date, `Sunset` an HTTP-date | pass — `@1788220800` = 2026-09-01T00:00:00Z; `Wed, 01 Sep 2027` is a real Wednesday; sunset is not earlier than deprecation | agree |
| R9.6 | ✔ | E.10: `Link … rel="deprecation"` | pass — link relation present and the deprecation carries a sunset date | agree |
| R9.7 | — | E.10: the window honors the 12-month floor | pass (no RFC 2119 keyword — see SF-3) — 2026-09-01 → 2027-09-01 is exactly 12 months | agree |
| R10.1 | ✔ | E.7: addressable, documented terminal states, an expiry, a failure representation. E.11 **cross-ref**: `operation_state` is drawn from that documented vocabulary | pass — all four named, and E.11's `succeeded`/`failed` values come from E.7's stated vocabulary | agree |
| R10.2 | ✔ | E.7: `Retry-After` paces the polling; cancel via the `cancel` action | **unverified** — the `Retry-After` SHOULD and the cancel clause pass; the MUST that "the operation's documentation states the expected polling cadence" is not exhibited | consistent (partial) |
| R10.5 | ✔ | E.8: `version` is the monotonic per-event marker | pass — at-least-once documented and `version: 3` present | agree |
| R10.7 | ✔ | E.8: Standard Webhooks envelope for the shared-secret topology | pass — HMAC-SHA256 over `id.timestamp.payload`, `whsec_` secret never on the wire, `webhook-*` headers per §1.10 | agree |
| R10.9 | ✔ | E.7: body `id` plus documented template satisfies the body clause; `Location` carries the same operation URI. E.11 **cross-ref**: Bloom uses the `id` form, so the stream carries `operation_id` | pass — both present and agreeing, `Location` denotes the operation not the result, and the form choice is carried consistently into the stream | agree |
| R11.2 | ✔ | E.9: `429 Too Many Requests` with `Retry-After` | pass — the MUST clause is settled by E.9. The streaming scope added in v1.1.0 states the mid-stream reporting behavior in the indicative, with no capitalized keyword, so under R1.1 it is not a MUST clause and gates no verdict — see **SF-4** | agree |
| R11.3 | ✔ | E.9: the pinned draft-11 `RateLimit` fields | pass — named as a draft, never as standards-compliant, with the pinned revision cited | agree |
| R11.5 | ✔ | E.9 "Exercises" header | **unverified** — the `429` clause passes; the `503` clause is never exhibited. The rule is also absent from E.9's reading paragraph (**SF-1**) | consistent (partial) |
| R11.7 | ✔ | E.2: the response carries the correlation ID | pass — `request-id` on every exhibited response, including all three errors and E.11's stream | agree |
| R12.2 | ✔ | E.9 "Exercises" header | **unverified** — two independent grounds. (a) The only candidate evidence is a problem `detail` string ("Retry after the interval in `Retry-After`"), which is a response body, not the documentation §12's preamble requires the provider to surface the obligation in; R12.1, R12.3, R12.4 and R12.7 are `unverified` for exactly that missing plane, and E.9 has no reading paragraph, so evidence policy 1 never engages. (b) Even granting the string, it reaches only the `429` half — no `503` appears anywhere in Appendix E | consistent (partial) |
| R12.5 | ✔ | E.4: the cursor is opaque. E.11 **cross-ref**: `stream_position` is not a cursor, so R12.5 does not reach it | pass — documented non-constructable, and the cross-ref is correct: R13.10 requires visible ordering, which is why the two names exist | agree |
| R12.8 | ✔ | E.8: verify over the raw body before parsing, enforce the timestamp window, dedupe on `webhook-id`, compare in constant time | **unverified** — four of the five clauses are named and pass; the fifth, "MUST fail closed on a missing, empty, or default secret at configuration load," is a configuration-time obligation the appendix never exhibits | consistent (partial) |
| R12.9 | ✔ | E.8: acks before processing | pass — with at-least-once, unordered delivery acknowledged | agree |
| R12.10 | ✔ | E.11: a terminal frame written without its blank line "is a frame the client never receives — and under R12.10 the client would then correctly report truncation on a successful export" | **unverified** — the passage explains the rule's *consequence* for a hypothetical malformed stream; it does not show Bloom surfacing the obligation in client documentation, which §12's preamble requires. Six further MUST clauses (sentinel tolerance, unknown-type tolerance, branching on `code`/`type`, keep-alive independence, the non-idempotent replay prohibition) are never exhibited | consistent (partial) |
| R13.1 | ✔ | E.11: the response is `200` with `text/event-stream`, an accurate self-delimiting media type, and it is not a `202` | pass — all three clauses settled over the one exhibited streaming response: `200` ✓, self-delimiting type per §1.11's definition ✓, no streaming `202` and no concatenated-JSON body labeled `application/json` ✓ | agree |
| R13.2 | ✔ | E.11: `/events` is R13.2's **second limb** — a distinct resource that streams unconditionally | **unverified** — the second limb is a MAY and Bloom takes it lawfully. R13.2's MUST clauses all sit under the first limb's antecedent ("Where one endpoint serves both a streamed and a non-streamed representation"), and Bloom exhibits no such endpoint, so `Accept`-based selection, `Vary: Accept`, and the query-parameter prohibition have no instance | consistent (partial) |
| R13.4 | ✔ | E.11: the response is `text/event-stream` for incrementally generated content | **unverified** — the SHOULD on SSE framing is followed and the MUST NOT (never described as IANA-registered) passes over the exhibited surface; the MUST that an API adopting SSE **documents** the missing registration is never exhibited. §1.10 documents it, but §1.10 is the standard, not Bloom | consistent (partial) |
| R13.5 | ✔ | E.11: "Every frame carries a type from Bloom's documented vocabulary" | **unverified** — the per-frame typing MUST passes, exhibited on four frame types. Two MUST clauses are unreached: documenting the **full** vocabulary *and stating that it may grow*, and documenting whether keep-alives are emitted and in what form | consistent (partial) |
| R13.6 | ✔ | E.11: `export.completed` is the terminal frame, so a connection dropping before it is truncation | **unverified** — the terminal-frame MUST passes on both outcomes (`export.completed` and the stream-ending `error` frame, which R13.6 makes terminal in its own right). Unreached: "An API MUST NOT carry outcome information in a sentinel" has no instance — Bloom emits no sentinel in the excerpt, and absence in an excerpt is not proof of absence in the API — and the client-side "MUST tolerate one" needs the documentation plane | consistent (partial) |
| R13.7 | ✔ | E.11: the error frame is the second carve-out from R5.12 — a problem object with `type`, `title`, `code`, omitting `status`, never described as an `application/problem+json` response | pass — every MUST clause settled: reserved `error` frame type ✓, `type`/`title`/`code` present ✓, `status` omitted ✓, template binding `export_source_unavailable` → `…/export-source-unavailable` ✓, R5.16 catalog membership asserted ✓, not described as a problem response ✓, and the one exhibited `error` frame ends the stream rather than being used for a survivable failure ✓ | agree |
| R13.9 | ✔ | E.11: `operation_id` is what makes the stream and the operation resource one capability with one identity; both terminal frames carry `operation_state` | **unverified** — identity carriage passes (`operation_id: op_000example` matches E.7's `id`, and the member matches R10.9's `id` form), the reserved-member and vocabulary clauses pass, and the retrievable full problem document is asserted. Unreached: "both channels MUST report the same terminal state" is never exhibited as two matching values — E.7 shows the operation resource as `running`, and no exchange shows it reporting `failed` alongside the error frame | consistent (partial) |
| R13.10 | ✔ | E.11: `stream_position` increases monotonically and is what a client echoes to resume; Bloom documents a 30-minute retention window; a resume outside it fails with a defined error rather than silently restarting | pass — all three MUST clauses reached. Strictly increasing positions exhibited (1, 2, 3), retention window asserted, out-of-window rejection asserted. The SHOULD on offering resumption is followed | agree |

**Totals:** 63 answer-key rules — **45 agree**, **18 consistent (partial
evidence)**, **0 disagree**.

Three §13 rules — **R13.3**, **R13.8**, and **R13.11** — are applicable MUSTs
that Appendix E annotates nowhere, so they have no answer-key row. All three
are scored in §3 (`unverified`). Their absence from the annotations is part of
standard finding **SF-2**.

---

## 5. Disagreements and findings

No rule verdict contradicts Appendix E. The items below came out of executing
the procedure: one is a defect in the skill, fixed here and re-run; four are
observations about the standard, proposed and **not applied**.

### The two v1.0.0 skill defects — fixes confirmed under a real input

Both were re-tested against the evidence this run produced, not by re-reading
the amended files.

**SD-1 (the contract plane defined by evidence, not by file format) held.**
Appendix E publishes no OpenAPI or JSON Schema document, so the pre-fix
procedure would have declared the contract plane unavailable and audited on
the source plane alone. Under the fixed text the plane was declared
**available**, the missing Spectral run was recorded as missing rather than
clean, and R4.1 — the rule requiring the OpenAPI document Bloom does not
exhibit — was reported `unverified` rather than silently skipped. The fix's
reach grew with §13: E.11's stream is contract evidence of exactly the kind
no schema language can express, and Appendix G says so directly — "OpenAPI 3.1
has no construct for 'the body is a sequence of items, each matching this
schema,' so `R13.5`'s frame-type vocabulary cannot be expressed in the contract
document." A file-format definition of the contract plane would have discarded
every §13 verdict this run produced.

**SD-2 (a definition behind the summary line) held, and did visible work.**
`N` was rebuilt by subtraction from the live count rather than adjusted from
92, and the one-verdict-per-rule rule changed five verdicts that would
otherwise have been inferred passes. Each of **R13.2, R13.4, R13.5, R13.6, and
R13.9** is annotated by E.11 as exercised, exhibits its headline clause
correctly, and carries at least one further MUST clause the appendix never
reaches. Scoring the demonstrated clause as `pass` is the natural move and is
precisely what the fix forbids. R13.5 is the sharpest case: the frames plainly
carry types, and the two unreached clauses — the vocabulary-may-grow statement
and the keep-alive disclosure — are the ones R13.5 itself says make R12.10
dischargeable.

That the same failure mode recurred five times on a section the fix predates is
the strongest evidence available that the fix was not over-fitted to the
v1.0.0 input.

### Skill defect (fixed, re-run)

#### SD-3 — an unbounded probe's hand-off `curl` carried none of the bound that made it safe

**What happened.** Commit `c09cb72` added a fourth gate tier to
`references/audit.md` §3.3, `unbounded`, for probes that hold a connection open
for as long as the API chooses. It requires "a stated wall-clock bound and cost
ceiling agreed before running; at the bound the probe is cut and reported
unverified with the exact `curl` (step 6), never run longer." That is correct
for a probe the auditor **runs**.

This run runs none of them. Every probe reaches the executor through step 6
instead: "Anything not run is reported unverified with the exact `curl` for the
user to run by hand." Step 6 knew nothing about tiers. Followed literally, this
audit would have handed the user
`curl -N https://api.example.com/v1/order-exports/exp_000example/events` to
settle R13.6 — the bare, open-ended request whose hazard the new tier exists to
control, stripped of the bound, the cost ceiling, and the warning, and handed
to someone running it *outside* the gate that would have supplied them.

The exposure is not hypothetical for R13.6 specifically. Appendix G's row says
to consume the stream to completion, and `audit.md` §3.3 already records why
that can never terminate when the rule fails: "a missing terminal frame
(R13.6) is indistinguishable from a stream still in progress."

**Why it is a defect and not a standard finding.** Appendix G is doing its job
— it defines the probe and expects the *skill* to classify and gate it, which
`audit.md` §3.3 says explicitly ("Appendix G does not mark the tiers; that
judgment is this skill's"). The gap is entirely inside the skill: §3.3 attaches
the precautions to running a probe, and §3.6 hands probes off without them.
A gate that a hand-off silently drops is not a gate.

**Fix applied** — `references/audit.md` §3, step 6, extended so the tier's
precautions travel with the command:

> **Anything not run is reported unverified with the exact `curl`** for the
> user to run by hand, **labeled with the tier from step 3**. … The tier's
> precautions travel with the command, because a handed-off probe is run
> without the gate that would otherwise have applied them: an unbounded
> probe's `curl` carries its wall-clock bound inline (`--max-time <seconds>`,
> plus `-N` so the bound is not defeated by buffering) and states the cost
> ceiling; a mutating or disruptive probe's `curl` names the disposable
> fixture it may touch and the side effect to expect. Handing over a bare
> unbounded `curl` reissues exactly the open-ended request the gate exists to
> prevent.

**Re-run.** The runtime-probe table in §3 was regenerated under the corrected
step 6. Every row now carries a **Tier** column and its precautions inline: the
four unbounded rows (R13.2/R4.11, R13.6, R13.9, R13.11) carry `-N --max-time`
with a bound chosen from what the probe actually needs — 10 s where only
response headers are wanted, 120 s where frames must be consumed — and the five
mutating rows name the fixture and the side effect. **No verdict changed**: the
defect was in the artifact handed to the user, not in the scoring, so `104
applicable MUSTs: 55 pass, 0 fail, 49 unverified` is the figure both before and
after. That is the correct shape for this defect — the pre-fix table was
unsafe, not wrong.

The fix also exposed the one probe the corrected procedure cannot express, which
is itself a result: **R13.11's** bound is not choosable, because G's row says to
hold past the documented maximum and Bloom documents no maximum. The row is
reported as not constructible rather than given an invented bound.

### Standard findings (proposed, not applied)

Offered to the owner as Part II amendments. `rest-api-standard.md`,
`conformance/`, and Part II content were **not modified**.

#### SF-1 — Appendix E's "Exercises" headers and "reading" paragraphs are two disagreeing rule lists

Each block carries an `Exercises …` header naming rule IDs, and a following
"The reading:" paragraph that also cites rule IDs. They do not agree, and
neither is marked authoritative. Recomputed for v1.1.2, with ranges expanded:

| Block | In the header, absent from the reading | In the reading, absent from the header |
|---|---|---|
| E.2 | R5.6, R7.1, R3.10 | **R7.2** |
| E.3 | R5.12, R5.3 | — |
| E.4 | R2.9, R6.5 | **R6.2, R12.5** |
| E.5 | R3.10, R3.11, R7.4, R5.11 | **R5.7** |
| E.6 | R2.11, R5.1 | — |
| E.7 | R5.1 | **R13.9** |
| E.8 | — | — |
| E.9 | R11.2, R11.5, R12.2 | **R11.3** |
| E.10 | R9.5, R9.6 | **R7.1, R11.7** |
| E.11 | — | **R1.8, R2.3, R4.10, R5.12, R5.13, R5.16, R10.1, R10.9, R12.5** |

E.8 is the only block whose two lists agree exactly. E.9 remains the clearest
failure: its header and its prose still share no rule at all.

**v1.1.2 changes the shape of the problem, and the v1.0.0 proposal no longer
fits.** That proposal was to make the header authoritative and require the
reading's citations to be a subset of it. **E.11 inverts the relation**: its
header is a strict subset of its reading, and the nine extra citations are
overwhelmingly the *cross-references* — R4.10 cited to say it does not reach
`/events`, R2.3 and R1.8 cited as hazards avoided, R10.1 and R10.9 cited for
the vocabulary and identity form the stream inherits. Those are genuinely not
coverage claims, and forcing them into the header would misreport E.11 as
exercising R4.10.

**Revised proposal.** Keep the per-block `Exercises` header as the
authoritative coverage list, and add a line to Appendix E's preamble
distinguishing the two citation kinds: a reading may cite a rule either
because the block demonstrates it (in which case the header must carry it too)
or because the block explains what the rule does *not* reach or which hazard it
prevents (in which case it must not). Under that rule the E.2/E.4/E.5/E.7/E.9/
E.10 reading-only entries are header omissions to be fixed, and E.11's nine are
correct as they stand. Editorial; no normative rule changes.

The answer key for this self-test had to be built by grepping both lists, and
would have to be built the same way after the amendment — which is fine, but it
means the amendment improves the human reading rather than the tooling.

#### SF-2 — the preamble reads as an exhaustiveness claim that the annotations do not deliver

Appendix E's preamble states: "Every block is annotated with the rules it
exercises." Read naturally, that claims the annotations are complete. They are
not, though the gap narrowed in v1.1.2.

This audit scored **55 applicable MUSTs as `pass`** on the appendix's own
evidence. **20 of those 55 are never annotated anywhere in Appendix E**,
despite being plainly demonstrated by it:

| Section | Demonstrated but unannotated |
|---|---|
| §1 | R1.6, R1.7 — E.1 *is* the conformance note that satisfies them |
| §2 | R2.1, R2.2, R2.4, R2.5, R2.7, R2.10 |
| §3 | R3.2, R3.3, R3.4 |
| §4 | R4.3, R4.16 |
| §5 | R5.2, R5.14 |
| §6 | R6.4, R6.9 |
| §8 | R8.2, R8.12 |
| §9 | **R9.1** — every URI in the appendix is `/v1`, the path-versioning MUST |

Three rules left this list in v1.1.2 — R1.8, R2.3, and R4.10 all gained
citations through E.11's reading — which is progress attributable to §13's
worked example being unusually well annotated.

**The §13 side of the gap runs the other way.** Three applicable §13 MUSTs are
annotated **nowhere**: **R13.3**, **R13.8**, and **R13.11**. Unlike the twenty
above, these are not demonstrated-but-uncited — Appendix E exhibits no evidence
for them at all. That is a different and arguably more interesting hole: E.11
demonstrates the streaming happy path and the post-commit error, and never the
guard, the pre-commit error, or long-polling. R13.3 is the notable absence,
since it is the one §13 rule that binds every conforming API whatever its
switch says.

**Proposed amendment.** Two parts. (a) Soften the preamble to say each block is
annotated with the *principal* rules it exercises — the accurate description of
what is there — rather than extending the annotations to all 55, since a
complete coverage map is a different artifact from a worked example. (b)
Separately, consider a short E.12 exhibiting the `stream` rejection guard
(R13.3) and a pre-commit streaming error (R13.8): both are small exchanges, both
are rules a reader is likely to get wrong, and R13.3 in particular is the rule
whose whole point is that it applies to APIs the rest of §13 does not.
Recommendation: **(a) now**, **(b) as a Part II candidate**.

#### SF-3 — 13 Part I rules carry no RFC 2119 keyword, so their strength is indeterminate

Carried forward from the v1.0.0 run and **recomputed against v1.1.2**, because
a section that added 12 rules could have changed the set. It did not.

R1.1 binds the RFC 2119 keywords "when, and only when, they appear in all
capitals." Thirteen Part I rules contain no capitalized keyword at all and are
stated in the indicative: **R1.2, R1.3, R2.8, R5.7, R5.10, R5.17, R6.2, R6.8,
R7.3, R9.2, R9.4, R9.7, R10.8** — the identical set to v1.0.0. **§13 added
none**: all 11 §13 rules and R12.10 carry capitalized keywords, so the section
was drafted clear of this problem.

Several of the thirteen remain load-bearing, and five are annotated by
Appendix E: R5.7 (DELETE returns `204`), R6.2 (empty collection returns `200`),
R6.8 (the filter grammar), R7.3 (the three-tier caching posture), R9.7 (the
12-month deprecation floor). R5.17 uses a lowercase "may" for what reads as a
prohibition on leaking stack traces.

**Why it matters operationally.** R1.7 makes the consequence of a deviation
depend on rule strength — a SHOULD deviation needs a conformance-note entry, a
MUST deviation makes the API nonconformant. For these thirteen rules that
question has no textual answer. It also makes `N` a judgment call: this audit
had to rule them out of the MUST count to get a defensible number, and a
different auditor could defensibly rule several of them in.

R6.2 acquired a second reason to care in v1.1.2: its streamed form ("A
**streamed** empty collection returns `200 OK` with zero item frames followed
by the terminal frame R13.6 requires") is now the streaming counterpart of a
rule whose strength is undetermined, so the strength question propagates into
§13 without §13 having introduced it.

**Not proposed as an amendment here.** Assigning a strength to thirteen
normative rules is a ratification decision, not an editorial one, and it sits
outside this task's mandate. Raised for the owner to route.

#### SF-4 — two v1.1.0 streaming scopes state obligations in the indicative, inside MUST-class rules

**New in this run.** SF-3 is about whole rules with no keyword; this is the same
problem one level down, in clauses added to rules that do carry MUSTs — and
unlike SF-3 it changed how a rule was scored.

**R11.2**'s streaming scope reads: "Quota exhausted *during* a committed stream
**is reported** in-band under R13.7, with `code` naming the exhaustion and a
`retry_after` extension member (§1.10) carrying the pacing hint." **R11.5**'s
reads: "the condition **is reported** under R13.7 and **carries** `retry_after`
in place of the header." Both describe provider behavior an auditor would want
to check. Neither carries a capitalized keyword.

**The audit consequence is concrete.** `audit.md`'s one-verdict rule scores a
rule `pass` only when the evidence settles *every* MUST clause it carries.
Under R1.1, these clauses are not MUST clauses, so they do not gate the verdict
— which is why **R11.2 is scored `pass`** here on E.9's `429` alone, even
though Appendix E never exhibits a mid-stream quota exhaustion and never shows
a `retry_after` member on any frame. Read the clauses as obligations instead
and R11.2 becomes `unverified`. One sentence of drafting decides a published
conformance figure, and the standard does not say which way.

The neighbouring drafting shows the alternative was available and used
elsewhere: R13.7's parallel obligation is written "MUST be delivered in-band",
and R5.13's streaming parenthetical is bound to R13.7's explicit "MUST omit".
The two rate-limit scopes are the outliers, not the pattern.

**Proposed amendment.** Restate both scopes with explicit keywords — R11.2:
"Quota exhausted during a committed stream MUST be reported in-band under
R13.7, carrying a `code` naming the exhaustion and a `retry_after` extension
member (§1.10)"; R11.5 correspondingly. This changes no obligation anyone
would dispute; it makes an already-intended MUST checkable, and removes a
clause-level instance of SF-3 from a rule that is otherwise clean. Small and
editorial in effect, but it touches normative text, so it is proposed rather
than applied.

Recommendation: **take SF-4 with SF-3**, since the owner routing one is already
deciding the same question.

---

## 6. Verification

| Check | Command | Result |
|---|---|---|
| Standard version pinned | `grep -m1 -oE '\*\*Version [0-9]+\.[0-9]+\.[0-9]+' rest-api-standard.md` | `**Version 1.1.2` |
| Section map derived live | `grep -n '^## [0-9]'` | 13 numbered normative sections, §1–§13 |
| Answer key non-empty | `sed -n '2544,2965p'` → `wc -l` | 422 lines |
| Answer key size | `grep -oE 'R[0-9]+\.[0-9]+' \| sort -u \| wc -l` | 63 rule IDs |
| Every answer-key rule is defined | `comm -23 expected-rules.txt defined-rules.txt` | empty |
| Every rule this audit cites is defined | every `R#.#` in **this document**, `sort -u` → 125 IDs, `comm -23` against the defined set | empty |
| No frozen research ID cited as a rule (R1.3) | `grep -nE '(HS\|AC\|OP\|ST)-[0-9]+' appendix-e.md` | no matches |
| Part I rule count | `grep -oE '^\*\*R[0-9]+\.[0-9]+' \| sort -u \| wc -l` | 139 |
| MUST-class rule count | per-rule blocks matched against `(MUST\|REQUIRED\|SHALL)` | 108 |
| `N` reproduces from the live standard | re-derivation, subtraction shown in §3 | 104 |
| `N` reconciles with the superseded run | applicable set less the 12 new rules | 92, matching v1.0.0 exactly |
| Verdict list covers exactly the applicable-MUST set | explicit 55-rule pass list and 49-rule unverified list built, unioned, and `comm`-diffed against the applicable set in both directions | empty both ways; union is 104 |
| No rule scored twice | `comm -12` of the pass and unverified lists | empty |
| Verdict arithmetic | 55 + 0 + 49 | 104 |
| Runtime probes checked against Appendix G's live-probe table | read Appendix G (lines 2984–3033) per audit.md §3 | 19 rows, up from 13; the report's 16 probe rows cover 17 rules, 10 rows from G's own table and 6 constructed beyond it |
| Every unrun probe carries its tier's precautions | corrected `audit.md` §3.6 | 4 rows classified unbounded — 3 carry `-N --max-time` inline, and R13.11 is reported not constructible rather than given an invented bound; 5 mutating rows name the fixture and the side effect |
| No HTTP request issued | — | runtime gate never opened |

Both `comm` inputs were produced with plain `sort -u`. `sort -V` is **not**
used: BSD `comm` on macOS mishandles version-sorted input and falsely reports
R10.9, R11.2, and R11.7 as undefined.

Appendix E's concrete values were checked rather than assumed:
`Deprecation: @1788220800` resolves to 2026-09-01T00:00:00Z, matching the
stated announcement date; `Sunset: Wed, 01 Sep 2027 00:00:00 GMT` names the
correct weekday; `webhook-timestamp: 1786295445` resolves to
2026-08-09T17:10:45Z, matching the event body's `created_at`. E.11's additions
check out too: `operation_id: "op_000example"` matches E.7's operation `id`
exactly; `stream_position` runs 1, 2, 3 strictly increasing across the success
stream; and the error frame's `code: "export_source_unavailable"` maps to
`type: "https://problems.example.com/export-source-unavailable"` under R5.13's
underscore-to-hyphen template, with `status` correctly absent.

---

## 7. What changed against the superseded v1.0.0 run

| | v1.0.0 run | v1.1.2 run |
|---|---|---|
| Standard version | 1.0.0 | 1.1.2 |
| Part I rules | 127 | 139 |
| MUST-class rules | 96 | 108 |
| `N` (applicable MUSTs) | 92 | **104** |
| Summary | `92: 52 pass, 0 fail, 40 unverified` | **`104: 55 pass, 0 fail, 49 unverified`** |
| Switches | 3 (`bulk-operations` off) | 4 — `streaming` added, declared **on** |
| Answer-key rules | 50 | **63** |
| Agreement | 39 agree / 11 partial / 0 disagree | **45 agree / 18 partial / 0 disagree** |
| Appendix G probe rows | 13 | **19** |
| Skill defects found | 2 (SD-1, SD-2) | **1 (SD-3)**; SD-1 and SD-2 re-confirmed |
| Standard findings | 3 (SF-1, SF-2, SF-3) | **4** — SF-1 and SF-2 revised, SF-3 unchanged, **SF-4 new** |

**Every one of the 92 v1.0.0 verdicts is unchanged.** The delta is entirely the
12 new rules: 3 pass (R13.1, R13.7, R13.10) and 9 unverified (R12.10, R13.2,
R13.3, R13.4, R13.5, R13.6, R13.8, R13.9, R13.11). That the older verdicts held
is a result, not an assumption — each was re-derived against the amended text
and the new E.11 evidence, and two were re-examined closely:

- **R11.2** was the one rule whose verdict was genuinely in play. Its v1.1.0
  streaming scope adds behavior Appendix E never exhibits, and whether that
  makes it `unverified` turns on whether an indicative clause is a MUST clause.
  Under R1.1 it is not, so `pass` stands — and the fact that the answer hinged
  on drafting rather than evidence is standard finding **SF-4**.
- **R5.16** kept its `unverified` verdict but changed its *reason*. In v1.0.0
  nothing in the appendix mentioned a problem catalog at all. E.11 now asserts
  one exists and contains the stream error's pair, so the unreached clause is
  no longer "is there a catalog" but "does it list **every** pair the API can
  return."

**The unverified share rose, from 43% to 47%**, and the cause is worth naming
because it is not a regression. §13's rules lean heavily on *documentation*
obligations — the frame-type vocabulary and its growth statement (R13.5), the
keep-alive disclosure (R13.5), the SSE registration gap (R13.4), the retention
window (R13.10) — and documentation is precisely the plane an excerpt-shaped
worked example cannot supply. R13.10 is the exception that proves the point:
it scores `pass` only because E.11 explicitly asserts all three of its
documentation obligations in prose. Where the appendix says the documentation
exists, the rule passes; where it merely shows conforming wire bytes, the rule
cannot.

**The most useful thing the version bump produced** is five fresh instances of
the inferred-pass failure mode (§5, SD-2), on a section written after the fix
that catches it. In v1.0.0 the fix caught one instance mechanically and a second
needed a human reader. Here it caught five, unaided, on first pass.

---

## 8. Verdict

The skill reproduced Appendix E's reading with no rule-level contradiction
across a section it had never seen. Its plane discipline forced 49 applicable
MUSTs to `unverified` where a less disciplined sweep would have inferred passes
from an excerpt, and the live-derivation discipline mattered concretely: the
skill carries no hardcoded map of the standard's shape, so §13 entered the
sweep, the `streaming` switch entered the scope record, and `N` grew from 92 to
104 with no edit to the skill at all.

The guard-rule path introduced in commit `c09cb72` did real work in the
direction that is easy to get wrong: `streaming` is **on** for Bloom, and
R13.3 still binds — per endpoint, on every path that does not stream — so it
is counted in `N` and reported `unverified` rather than waved through as
covered by an on switch.

The one defect found is the kind only a real input finds: a gate whose
precautions were attached to running a probe and were silently dropped when the
probe was handed to the user instead. It is fixed in
`.claude/skills/rest-standards/references/audit.md`, and the corrected
procedure reproduces this document's results with a safer artifact and an
unchanged score.

**Gate F evidence:** `104 applicable MUSTs: 55 pass, 0 fail, 49 unverified`
against `rest-api-standard.md` v1.1.2, on the contract and source planes, with
`streaming=on` and `bulk-operations=off`.
