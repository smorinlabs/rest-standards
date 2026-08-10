# Phase 7 skill self-test — `rest-standards` audit mode against Appendix E

**Date:** 2026-08-10 · **Task:** Phase 7, Task 6 · **Gate:** F evidence record

Appendix E of `rest-api-standard.md` is a worked example — the Bloom Orders
API — already annotated with rule IDs and carrying its own conformance note.
That makes it a self-test with a known-correct answer: running the
`rest-standards` skill's `audit` mode against it should reproduce the
appendix's own reading, and any place it does not is either a skill defect or
a genuine standard finding.

**Result:** no rule-level contradiction. The skill reproduced every verdict the
appendix annotates. The run exposed **two skill defects** (both procedural gaps
in `references/audit.md`, both fixed and re-run) and **two standard findings**
about Appendix E's annotation apparatus, proposed here and not applied.

---

## 1. How the skill was executed

The skill was run as documented, not from memory of REST practice.

| Step | `SKILL.md` says | What was done |
|---|---|---|
| 1 | Locate and version-pin the standard | `grep -m1 -oE '\*\*Version [0-9]+\.[0-9]+\.[0-9]+'` → **v1.0.0** |
| 2 | Get the live section map; never read the standard whole | `grep -n '^## '` → 24 top-level sections |
| 3 | Identify the mode | `audit` — an existing, documented API assessed for conformance |
| 4 | Settle tier, switches, plane | `references/scoping.md` (§2 below) |
| 5 | Execute the mode procedure | `references/audit.md`, sweeping §1–§12 in section order per `references/review.md` |
| 6 | Cite `R#.#` only | Validated mechanically (§6 below) |

**Files loaded:** `SKILL.md`, `references/scoping.md`, `references/audit.md`,
and `references/review.md` (audit.md delegates its findings table and its
section-order sweep to review mode).

**Sections read live during the sweep:** §1 (lines 23–327), §2 (329–461),
§3 (463–597), §4 (599–760), §5 (762–942), §6 (944–1038), §7 (1040–1096),
§8 (1098–1209), §9 (1211–1302), §10 (1304–1437), §11 (1439–1512),
§12 (1514–1598). No rule text was quoted from memory.

---

## 2. Scope record

    Standard: rest-api-standard v1.0.0
    Tier: public
    Switches: webhooks=on, async-operations=on, bulk-operations=off
      (off because "imports run through the async export/import operations;
       no synchronous bulk endpoint is offered")
    Evidence planes available: contract, source
    Evidence planes unavailable: runtime — no deployment exists; no HTTP
      request was issued

`references/scoping.md` calls for one `AskUserQuestion` to settle tier. There
is no user in this run, and none is needed: **the audited artifact declares
its own scope.** Appendix E.1 is Bloom's conformance note, rendered from the
§1.9 template, and it states the tier and all three switch states verbatim.
The scoping answers were taken from it rather than asked.

Tier does not change the applicable rule set here. §1.7 permits later sections
to scope rules by tier; grepping the live standard for a tier annotation on
any rule returns only R1.5's own hypothetical example, so **no rule in v1.0.0
carries a tier scope** and `public` neither adds nor relaxes anything.

### What each plane is, in this run

| Plane | Supplied by | Reaches |
|---|---|---|
| contract | The literal HTTP request/response exchanges in E.2–E.10 | Wire-observable shape: status codes, headers, media types, body structure |
| source | Appendix E's "reading" paragraphs, which state implementation and documentation behavior the exchanges cannot show | Idempotency replay semantics, documented default sort order, webhook consumer obligations |
| runtime | — not available — | Nothing |

**Spectral could not run.** `references/audit.md` names
`conformance/spectral.yaml` as the contract plane's checker, and Spectral
needs a machine-readable document. Appendix E publishes none. The contract
plane is still available — the exchanges *are* documented contract — so the
contract was read directly and the missing Spectral run is recorded as missing,
never as clean. That the worked example exhibits no OpenAPI document is itself
reported below under R4.1. **This gap in the skill's own procedure is skill
defect SD-1 (§5).**

### Evidence policy

Fixed before the sweep, so that the verdicts are reproducible:

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
   document, Bloom's implementation source — are `unverified`, naming the
   missing plane.

---

## 3. Conformance summary

    92 applicable MUSTs: 53 pass, 0 fail, 39 unverified

This matches the `<N> applicable MUSTs: <P> pass, <F> fail, <U> unverified`
format required by `references/audit.md` § Report.

**How `N` was derived** (per the definition added to `audit.md` as fix SD-2):

| Step | Count |
|---|---|
| Rules in Part I | 127 |
| — carrying a capitalized MUST / MUST NOT / REQUIRED / SHALL clause (R1.1) | 96 |
| — less rules binding the standard's own drafting, not the API: R1.1, R1.4 | −2 |
| — less rules scoped to a switch declared off: R10.4 (§10.2), R5.8 (bulk endpoints) | −2 |
| **N — applicable MUSTs** | **92** |

The 31 rules excluded as SHOULD-, MAY-, or keyword-free are assessed in the
sweep and appear in the table below where Appendix E annotates them; they are
not counted in `N`.

**Zero failures is the expected answer.** Appendix E is the standard's own
worked example and its conformance note records `Deviations: none`. A failure
would have meant the standard contradicts itself.

**`U = 39` is the honest reach of the evidence, not a defect in Bloom.**
Appendix E is a set of excerpts, not a deployed API with published
documentation. 39 applicable MUSTs have at least one clause that no supplied
plane reaches — 11 of them because the runtime plane is absent, the rest
because Bloom's OpenAPI document, implementation source, and operational
documentation are outside the appendix.

### N/A declarations (R1.6)

- `bulk-operations`: **off** — "imports run through the async export/import
  operations; no synchronous bulk endpoint is offered". Removes **R10.4**
  (§10.2) and **R5.8**, whose obligations bind bulk endpoints only.

Both carry the stated reason R1.6 demands. No other switch is off.

### Unverified rules needing the runtime plane

Each is reported `unverified` with the `curl` that would settle it, per
`audit.md` § Report. None was run — no deployment exists, and the gate in
`audit.md` §3 was never opened.

`audit.md` §3 ends with "Read Appendix G for the live probe table; do not work
from the list above." Appendix G (13 rows) was read and every row below was
checked against it. The **Source** column records whether the probe is
Appendix G's own definition or a construction beyond G's table. Appendix G's
remaining probes — empty collection (R6.2), existence masking (R5.10), quota
(R11.2), error negotiation (R5.12/R5.13), correlation (R11.7), 202 discovery
(R10.9), cache posture (R7.1–R7.3) — are not listed here because the contract
and source planes already settled those rules; a runtime probe would only
corroborate a verdict, not supply a missing one.

| Rule | What runtime would settle | Probe | Source |
|---|---|---|---|
| R2.6 | Trailing slash returns `308` to the canonical form | `curl -sSi https://api.example.com/v1/orders/` | G "Trailing slash" |
| R1.9 | `dry_run` on an endpoint not implementing it is rejected `400` | `curl -sSi -X POST 'https://api.example.com/v1/orders?dry_run=true' -H 'Authorization: Bearer $T' -d '{}'` | G "Rehearsal guard" |
| R3.7, R5.11 | Unsupported PATCH media type rejected `415`; `Accept-Patch` advertised | `curl -sSi -X PATCH https://api.example.com/v1/orders/ord_000example -H 'Content-Type: application/json' -H 'Authorization: Bearer $T' -d '{}'` | G "PATCH media type" |
| R5.11 | An unimplemented method on a real path returns `405` carrying `Allow` | `curl -sSi -X <unimplemented-method> https://api.example.com/v1/orders/ord_000example -H 'Authorization: Bearer $T'` — **gated**: `audit.md` §3.4 warns that a method assumed unimplemented and turning out to be implemented is a real mutation, so this needs the second confirmation and a disposable fixture | G "Unknown method" |
| R5.9 | `401` unauthenticated vs `403` authenticated-unauthorized | `curl -sSi https://api.example.com/v1/orders` (no credential) | G "Auth split" |
| R8.1 | TLS 1.2+ served, TLS 1.0/1.1 rejected | `curl -sSI --tlsv1.1 --tls-max 1.1 https://api.example.com/v1/orders` (must fail) | beyond G |
| R11.5 | `503` carries `Retry-After` | Appendix G defines **no** probe for the `503` clause, and none is constructible: `503` cannot be induced by a well-formed request. Settled by observation under real capacity overload, or by reading the error-path source | beyond G |
| R4.11 | `Vary` on negotiated responses | `curl -sSi https://api.example.com/v1/orders -H 'Accept: application/json' -H 'Authorization: Bearer $T'` | beyond G |
| R5.5 | `307`/`308` preserve method and body | `curl -sSi -X POST https://api.example.com/v1/orders/ -H 'Authorization: Bearer $T'` | beyond G |
| R7.4 clause 2 | No unfiltered collection-level DELETE is offered | `curl -sSi -X DELETE https://api.example.com/v1/orders -H 'Authorization: Bearer $T'` — **gated**: mutating and destructive precisely when the API fails the check, so it needs `audit.md` §3.4's second confirmation. G's "Destructive guard" row probes clause 1 only | beyond G |
| R11.4 | Any proprietary quota headers are documented | `curl -sSiD- https://api.example.com/v1/orders -H 'Authorization: Bearer $T'` | beyond G |

The remaining 28 unverified rules need Bloom's OpenAPI document (R4.1, R4.2,
R4.8, R4.9), its implementation source (R8.6–R8.9, R8.11), or its published
operational and client documentation (R3.1, R3.9, R5.16, R6.5, R10.2, R10.6,
R11.1, R11.6, R11.8, R12.1, R12.3, R12.4, R12.7, R12.8, and the conditional
clauses of R3.8, R3.12, R4.15, R8.4, R8.10).

---

## 4. Comparison against the answer key

The answer key is the 50 rule IDs Appendix E annotates, extracted with
`grep -oE 'R[0-9]+\.[0-9]+' | sort -u`. Every one is a rule defined in the
standard (§6). Sub-rule references collapse under that grep: `R5.13.1` and
`R5.13.4` appear as R5.13, `R10.7.1` as R10.7.

**Outcome vocabulary.** `agree` — the skill's verdict matches what the
appendix demonstrates. `consistent (partial)` — the appendix names the rule
as exercised, the skill confirms the demonstrated clause, and the rule scores
`unverified` because *another* MUST clause of the same rule is unreachable on
the supplied planes. That is an evidence-plane limit both sides agree on, not
a contradiction, so it is not classified as a disagreement. `disagree` — the
skill's verdict contradicts the appendix.

The **Count** column marks whether the rule is one of the 92 applicable MUSTs
(`✔`) or is excluded from `N` as SHOULD / MAY / keyword-free (`—`).

| Rule | Count | Appendix E says | Skill reported | Outcome |
|---|---|---|---|---|
| R2.9 | ✔ | E.4: filters, sort, pagination travel as query parameters | pass — E.4's query string carries all four modifiers; no modifier in a path segment | agree |
| R2.11 | ✔ | E.6: `POST /v1/orders/{id}/cancel`, `200` with the mutated representation | pass — action sub-path form and synchronous response shape both as R2.11 specifies | agree |
| R2.13 | ✔ | E.6: "There is no `POST /v1/orders/cancel`" | pass — no collection-level action anywhere in the appendix | agree |
| R3.7 | ✔ | E.5: `application/merge-patch+json`; `null` deletes | **unverified** — media-type clause passes; the `Accept-Patch` advertisement and the `415` rejection are never exhibited | consistent (partial) |
| R3.8 | ✔ | E.5: Merge Patch is the one exception to null-equals-absent | **unverified** — null-equivalence asserted; "a `null` targeting a non-deletable field MUST return `400`" is never exhibited | consistent (partial) |
| R3.9 | ✔ | E.2: replay with same payload returns the stored response; different payload rejected | **unverified** — key and replay semantics documented; the MUST that "the stated retention window is at least 24 hours" is never stated | consistent (partial) |
| R3.10 | ✔ | E.2: strong `ETag` emitted | pass — `"v1-000example"` is strong (no `W/`) and is consumed by E.5's `If-Match` | agree |
| R3.11 | — | E.5: `If-Match` demanded, `428` when absent | pass (SHOULD) | agree |
| R4.4 | ✔ | E.2: the body is snake_case | pass — bodies and query parameters both snake_case across E.2, E.4, E.7, E.8 | agree |
| R4.5 | ✔ | E.2: `id` is a string | pass — every identifier is a JSON string | agree |
| R4.6 | ✔ | E.2: `created_at` carries an explicit offset; `deliver_on` is the date-only exception | pass | agree |
| R4.7 | ✔ | E.2: `amount` 4599 with `"usd"` is $45.99 in minor units | pass — integer minor units with the REQUIRED sibling `currency` | agree |
| R4.17 | ✔ | Preamble: every response carries `Access-Control-Expose-Headers` | pass — asserted in prose per evidence policy 1 | agree |
| R5.1 | ✔ | E.6, E.7: status matches the outcome | pass — `201`/`422`/`200`/`428`/`204`/`202`/`429` all match their registered semantics | agree |
| R5.3 | — | E.3: `422` for a well-formed but semantically invalid request | pass (SHOULD) | agree |
| R5.6 | ✔ | E.2: `201 Created` with `Location` | pass | agree |
| R5.7 | — | E.5: a matching `If-Match` delete returns `204 No Content` | pass (no RFC 2119 keyword — see SF-3) | agree |
| R5.11 | ✔ | E.5: `428 Precondition Required` where `If-Match` is demanded and absent | **unverified** — the `428` clause passes; the `405`+`Allow` and `415` clauses are never exhibited | consistent (partial) |
| R5.12 | ✔ | E.3: `application/problem+json` | pass — all three error responses are problem documents | agree |
| R5.13 | ✔ | E.3: `code` maps to `type` by the fixed template; the human link lives in `documentation` | pass — verified on all three: `validation_failed`, `precondition_required`, `rate_limit_exceeded`; `type` present, no `about:blank`, base domain provider-controlled. Points 2 (immutability) and 3 (clients must not depend on `type` resolving) are unexhibited: point 2 is a negative obligation with instances present and no counterexample among the three pairs (evidence policy 2), and point 3 binds clients through R12.7, which is separately scored `unverified` | agree |
| R5.15 | — | E.3: field failures ride `errors[]` with JSON Pointers | pass (SHOULD) — `pointer`, `code`, `detail` all present | agree |
| R6.1 | ✔ | E.4: the envelope carries `items` and `next_cursor` | pass | agree |
| R6.2 | — | E.4: an empty result is `200` with `"items": []` | pass (no RFC 2119 keyword — see SF-3) | agree |
| R6.3 | — | E.4: the cursor is opaque | pass (SHOULD) | agree |
| R6.5 | ✔ | E.4: the next page is `?cursor=…&limit=50` | **unverified** — the `cursor`/`limit` naming clause passes; the MUST that each collection documents its default and maximum `limit` is never exhibited | consistent (partial) |
| R6.6 | ✔ | E.4: documented default order is `-created_at` with `id` as tiebreak | pass — the soundness precondition R6.3 depends on | agree |
| R6.7 | — | E.4: `-created_at` descending | pass (MAY) | agree |
| R6.8 | — | E.4: equality plus a bracket range filter, AND-combined | pass (no RFC 2119 keyword — see SF-3) | agree |
| R7.1 | ✔ | E.2 header list; E.10 notes it is always on | pass — every exhibited response carries an explicit `Cache-Control` | agree |
| R7.2 | ✔ | E.2 **reading** (not its "Exercises" header): the response is explicitly non-cacheable-shared | pass — `private, no-cache` on authenticated data; `no-store` on errors | agree — but see **SF-1** |
| R7.3 | — | E.2: `private, no-cache` posture | pass (no RFC 2119 keyword — see SF-3) — tier 1 of the three-tier posture, `no-store` correctly reserved for errors rather than blanket-applied | agree |
| R7.4 | ✔ | E.5: DELETE without its precondition is refused `428` | **unverified** — clause 1 (DELETE MUST require `If-Match`) passes; clause 2 (no unfiltered collection-level DELETE) has no instance in evidence | consistent (partial) |
| R8.1 | ✔ | E.2: "the request authenticates with a bearer token over TLS" | **unverified** — runtime; the TLS version floor and the rejection of TLS 1.0/1.1 are not observable in an excerpt | consistent (partial) |
| R8.3 | ✔ | E.2: scoped API keys server-to-server, OAuth for third-party on a user's behalf | pass — matches the authority boundary R8.3 draws | agree |
| R9.5 | ✔ | E.10: `Deprecation` is a structured-field date, `Sunset` an HTTP-date | pass — `@1788220800` = 2026-09-01T00:00:00Z; `Wed, 01 Sep 2027` is a real Wednesday; sunset is not earlier than deprecation | agree |
| R9.6 | ✔ | E.10: `Link … rel="deprecation"` | pass — link relation present and the deprecation carries a sunset date | agree |
| R9.7 | — | E.10: the window honors the 12-month floor | pass (no RFC 2119 keyword — see SF-3) — 2026-09-01 → 2027-09-01 is exactly 12 months | agree |
| R10.1 | ✔ | E.7: addressable, documented terminal states, an expiry, a failure representation | pass — all four named | agree |
| R10.2 | ✔ | E.7: `Retry-After` paces the polling; cancel via the `cancel` action | **unverified** — the `Retry-After` SHOULD and the cancel clause pass; the MUST that "the operation's documentation states the expected polling cadence" is not exhibited | consistent (partial) |
| R10.5 | ✔ | E.8: `version` is the monotonic per-event marker | pass — at-least-once documented and `version: 3` present | agree |
| R10.7 | ✔ | E.8: Standard Webhooks envelope for the shared-secret topology | pass — HMAC-SHA256 over `id.timestamp.payload`, `whsec_` secret never on the wire, `webhook-*` headers per §1.10 | agree |
| R10.9 | ✔ | E.7: body `id` plus documented template satisfies the body clause; `Location` carries the same operation URI | pass — both present and agreeing, and `Location` denotes the operation, not the result | agree |
| R11.2 | ✔ | E.9: `429 Too Many Requests` with `Retry-After` | pass | agree |
| R11.3 | ✔ | E.9: the pinned draft-11 `RateLimit` fields | pass — named as a draft, never as standards-compliant, with the pinned revision cited | agree |
| R11.5 | ✔ | E.9 "Exercises" header | **unverified** — the `429` clause passes; the `503` clause is never exhibited. The rule is also absent from E.9's reading paragraph (**SF-1**) | consistent (partial) |
| R11.7 | ✔ | E.2: the response carries the correlation ID | pass — `request-id` on every exhibited response, including all three errors | agree |
| R12.2 | ✔ | E.9 "Exercises" header | pass — the provider surfaces the obligation in the problem `detail` ("Retry after the interval in `Retry-After`"), which is what §12's preamble requires | agree |
| R12.5 | ✔ | E.4: the cursor is opaque | pass — documented non-constructable | agree |
| R12.8 | ✔ | E.8: verify over the raw body before parsing, enforce the timestamp window, dedupe on `webhook-id`, compare in constant time | **unverified** — four of the five clauses are named and pass; the fifth, "MUST fail closed on a missing, empty, or default secret at configuration load," is a configuration-time obligation the appendix never exhibits | consistent (partial) |
| R12.9 | ✔ | E.8: acks before processing | pass — with at-least-once, unordered delivery acknowledged | agree |

**Totals:** 50 answer-key rules — **40 agree**, **10 consistent (partial
evidence)**, **0 disagree**.

---

## 5. Disagreements and findings

No rule verdict contradicts Appendix E. The four items below came out of
executing the procedure: two are defects in the skill, fixed here; two are
observations about the standard, proposed and **not applied**.

### Skill defects (fixed, re-run)

#### SD-1 — the contract plane was defined by file format, not by evidence

**What happened.** `references/scoping.md` and `references/audit.md` both
define the contract plane as "an OpenAPI or JSON Schema document exists," with
`conformance/spectral.yaml` as its checker. Appendix E has neither. Followed
literally, the skill would have declared the contract plane **unavailable** and
audited on the source plane alone — discarding the HTTP request/response
exchanges, which are the single richest piece of contract evidence in the
input, and silently converting roughly half the table to `unverified`.

**Why it is a defect and not a standard finding.** The skill's own premise is
that planes are about *what evidence can be reached*, not about file
extensions. `SKILL.md` says so: "Evidence plane — contract / source / runtime.
Decides which rules can be verified *at all*." Defining the plane by artifact
type contradicts that premise, and the first real input exposed it.

**Fix applied** — `references/scoping.md`, plane table:

> `| contract | A documented interface contract exists — an OpenAPI or JSON Schema document, or, failing that, reference documentation or worked request/response exchanges | conformance/spectral.yaml on a machine-readable document; otherwise read the contract directly |`

**Fix applied** — `references/audit.md`, plane table row plus a new paragraph
in §1 Contract plane stating that Spectral's absence is recorded as a missing
run rather than a clean one, that the executor must not downgrade to
source-only, and that a contract R4.1 requires to be an OpenAPI document and
isn't is itself an R4.1 finding.

**Re-run:** the audit was re-executed with the corrected plane definition. The
contract plane is available, Spectral is recorded as inapplicable, R4.1 is
reported `unverified` rather than silently skipped, and the verdict table is
unchanged — the fixed procedure reproduces the run this document reports.

#### SD-2 — the mandated summary line had no definition of "applicable MUST"

**What happened.** `references/audit.md` § Report requires the deliverable to
end in `<N> applicable MUSTs: <P> pass, <F> fail, <U> unverified` and never
says how to compute `N`, nor how to score a rule whose clauses split. Both had
to be invented to produce the required line, and two auditors inventing
differently would emit non-comparable numbers for the same API — from a line
whose entire purpose is comparability.

The clause-splitting half is the sharper gap. Most interesting rules are
multi-clause: R3.7 carries a media-type MUST, an `Accept-Patch` MUST, and a
`415` MUST; R11.5 carries one clause for `429` and one for `503`. Appendix E
exhibits some and not others. With no stated rule, scoring the demonstrated
clause as `pass` is the natural move — and it is precisely the inferred pass
that `SKILL.md`'s plane discipline forbids.

**Fix applied** — `references/audit.md` § Report, two paragraphs after the
summary line: a three-part test for an applicable MUST (carries a capitalized
MUST-class keyword per R1.1; binds the audited API rather than the standard's
own drafting; is not scoped to a switch declared off), a requirement to show
the subtraction from the live rule count rather than restate a remembered `N`,
and the one-verdict-per-rule rule — `pass` only when the evidence settles
*every* MUST clause, `unverified` otherwise, naming the unreached clause.

**Re-run:** `N` was re-derived mechanically from the fixed definition against
the live standard — 127 Part I rules, 96 carrying a MUST-class keyword, less
R1.1 and R1.4 (which bind the standard's drafting) and R10.4 and R5.8 (bulk,
switched off) — giving **N = 92**, matching the first pass, and the verdict
list was asserted to cover exactly that set with no rule missing and none
extra.

Re-scoring every rule under the new one-verdict rule also **changed one
verdict**, which is the clearest evidence the fix was needed. R12.8 had been
scored `pass` on the strength of the four webhook-consumer clauses E.8 names,
with its fifth — "MUST fail closed on a missing, empty, or default secret at
configuration load" — waved through as covered by the same reading. That is
exactly the inferred pass the fix forbids, and applying the fix caught it.
R12.8 is now `unverified`, and the summary line moved from
`54 pass, 0 fail, 38 unverified` to **`53 pass, 0 fail, 39 unverified`**;
this document reports the corrected figures throughout. The error was in the
audit, not in Bloom — nothing suggests Bloom fails the clause, only that
Appendix E never exhibits it.

### Standard findings (proposed, not applied)

Offered to the owner as Part II amendments. `rest-api-standard.md`,
`conformance/`, and Part II content were **not modified**.

#### SF-1 — Appendix E's "Exercises" headers and "reading" paragraphs are two disagreeing rule lists

Each block carries an `Exercises …` header naming rule IDs, and a following
"The reading:" paragraph that also cites rule IDs. They do not agree, and
neither is marked authoritative.

| Block | In the header, absent from the reading | In the reading, absent from the header |
|---|---|---|
| E.2 | R7.1 | **R7.2** |
| E.4 | R2.9 | **R6.2, R12.5** |
| E.5 | R3.10, R3.11, R7.4, R5.11 | **R5.7** |
| E.7 | R5.1 | — |
| E.9 | R11.2, R11.5, R12.2 | **R11.3** |

E.9 is the clearest case: its header and its prose share no rule at all. E.2 is
the most consequential: the header cites R7.1 and R7.3, the reading cites R7.2
and R7.3, and the block genuinely exercises all three — but no single list in
the appendix says so.

Nothing here is *wrong*: every cited rule is genuinely exercised. The problem
is that a reader, or a tool extracting coverage, cannot tell which list is the
claim. The answer key for this very self-test had to be built by grepping both.

**Proposed amendment.** Make the per-block `Exercises` header the authoritative
coverage list, require the reading's citations to be a subset of it, and add a
line to Appendix E's preamble saying so. Editorial; no normative rule changes.

#### SF-2 — the preamble reads as an exhaustiveness claim that the annotations do not deliver

Appendix E's preamble states: "Every block is annotated with the rules it
exercises." Read naturally, that claims the annotations are complete.

They are not, and the gap is large. This audit scored **53 applicable MUSTs as
`pass`** on the appendix's own evidence. **23 of those 53 are never annotated
anywhere in Appendix E**, despite being plainly demonstrated by it:

| Section | Demonstrated but unannotated |
|---|---|
| §1 | R1.6, R1.7, R1.8 — E.1 *is* the conformance note that satisfies them |
| §2 | R2.1, R2.2, R2.3, R2.4, R2.5, R2.7, R2.10 |
| §3 | R3.2, R3.3, R3.4 |
| §4 | R4.3, R4.10, R4.16 |
| §5 | R5.2, R5.14 |
| §6 | R6.4, R6.9 |
| §8 | R8.2, R8.12 |
| §9 | **R9.1** — every URI in the appendix is `/v1`, the path-versioning MUST |

R9.1 and the §1 trio are the striking ones: the appendix demonstrates path
versioning in literally every exhibited URI and never cites the rule, and E.1
satisfies the entire conformance apparatus without citing R1.6, R1.7, or R1.8.

**Proposed amendment.** Either soften the preamble to say each block is
annotated with the *principal* rules it exercises — the accurate description of
what is there — or extend the annotations to the full set. The first is a
one-sentence editorial change; the second is larger and would make Appendix E a
genuine coverage fixture. Recommendation: **soften the preamble now**, and
record extending the annotations as a separate candidate, since a complete
coverage map is a different artifact from a worked example.

#### SF-3 — observation: 13 Part I rules carry no RFC 2119 keyword, so their strength is indeterminate

Surfaced while computing `N`, and reported as an observation rather than a
proposal because it touches normative drafting rather than an appendix.

R1.1 binds the RFC 2119 keywords "when, and only when, they appear in all
capitals." Thirteen Part I rules contain no capitalized keyword at all and are
stated in the indicative: **R1.2, R1.3, R2.8, R5.7, R5.10, R5.17, R6.2, R6.8,
R7.3, R9.2, R9.4, R9.7, R10.8**.

Several are load-bearing, and five are annotated by Appendix E: R5.7 (DELETE
returns `204`), R6.2 (empty collection returns `200`), R6.8 (the filter
grammar), R7.3 (the three-tier caching posture), R9.7 (the 12-month
deprecation floor). R5.17 uses a lowercase "may" for what reads as a
prohibition on leaking stack traces.

**Why it matters operationally.** R1.7 makes the consequence of a deviation
depend on rule strength — a SHOULD deviation needs a conformance-note entry, a
MUST deviation makes the API nonconformant. For these thirteen rules that
question has no textual answer. It also makes `N` a judgment call: this audit
had to rule them out of the MUST count to get a defensible number, and a
different auditor could defensibly rule several of them in.

**Not proposed as an amendment here.** Assigning a strength to thirteen
normative rules is a ratification decision, not an editorial one, and it sits
outside this task's mandate. Raised for the owner to route.

---

## 6. Verification

| Check | Command | Result |
|---|---|---|
| Standard version pinned | `grep -m1 -oE '\*\*Version [0-9]+\.[0-9]+\.[0-9]+' rest-api-standard.md` | `**Version 1.0.0` |
| Answer key non-empty | `sed -n '/^## Appendix E/,/^## Appendix F/p'` → `wc -l` | 328 lines |
| Answer key size | `grep -oE 'R[0-9]+\.[0-9]+' \| sort -u \| wc -l` | 50 rule IDs |
| Every answer-key rule is defined | `comm -23 expected-rules.txt defined-rules.txt` | empty |
| Every rule this audit cites is defined | `comm -23 audited-rules.txt defined-rules.txt` | empty |
| No frozen research ID cited as a rule (R1.3) | `grep -nE '(HS\|AC\|OP)-[0-9]+' appendix-e.md` | no matches |
| `N` reproduces from the live standard | re-derivation script | 92, matching |
| Verdict list covers exactly the applicable-MUST set | set difference, both directions | empty both ways |
| Runtime probes checked against Appendix G's live-probe table | read Appendix G (lines 2339–2373) per audit.md §3 | 5 of 11 rows are G's own; 6 recorded as constructed beyond G |
| No HTTP request issued | — | runtime gate never opened |

Both `comm` inputs were produced with plain `sort -u`. `sort -V` is **not**
used: BSD `comm` on macOS mishandles version-sorted input and falsely reports
R10.9, R11.2, and R11.7 as undefined.

Appendix E's concrete values were checked rather than assumed:
`Deprecation: @1788220800` resolves to 2026-09-01T00:00:00Z, matching the
stated announcement date; `Sunset: Wed, 01 Sep 2027 00:00:00 GMT` names the
correct weekday; `webhook-timestamp: 1786295445` resolves to
2026-08-09T17:10:45Z, matching the event body's `created_at`. All three are
correct.

---

## 7. Verdict

The skill reproduced Appendix E's reading with no rule-level contradiction, and
its plane discipline did real work: it forced 39 applicable MUSTs to
`unverified` where a less disciplined sweep would have inferred passes from an
excerpt, and `references/review.md`'s cross-cutting traps caught the
standard-wide R1.9 `dry_run` guard and the R6.6 default-sort-order soundness
precondition, neither of which the input volunteers.

The two defects it exposed are the kind only a real input finds: a plane
defined by file format instead of by evidence, and a mandated summary
statistic with no definition behind it. Both are fixed in
`.claude/skills/rest-standards/references/`, and the corrected procedure
reproduces this document's results.

**Gate F evidence:** `92 applicable MUSTs: 53 pass, 0 fail, 39 unverified`
against `rest-api-standard.md` v1.0.0, on the contract and source planes.
