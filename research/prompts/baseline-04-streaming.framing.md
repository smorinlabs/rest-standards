# Framing: Streaming posture — prescriptive leaf (Phase 6)

Date framed: 2026-08-10

Shared Phase 6 context — the owner's coverage and deliverable rulings, the
reference set and its divergence from the series charter, the prior-work check,
the scope boundaries, and the source seeds — lives in
[`survey-08-streaming.framing.md`](survey-08-streaming.framing.md) and is not
repeated here. This document records only what is specific to the prescriptive
leaf.

## Dependency: this leaf runs second

`baseline-04-streaming` **must not be dispatched until
`survey-08-streaming`'s report has passed its arrival check and been
committed** — not merely written to disk. The arrival check is a mandate scan
(a survey report that recommends a rule is sent back), a citation check
(authority class and date on every material claim), and a credential scan. A
subagent reads whatever is on disk, so an uncommitted or unchecked report would
propagate its defects into the principles table. The prescriptive leaf's
proposed-principles table has to cite the descriptive run's evidence rows; run
in parallel it would derive rules from its own search, duplicating the survey's
work at lower quality and destroying the evidence chain that
`research/CLAUDE.md` exists to protect. This is the ordering used for every
prior baseline leaf.

## Provisional identifier series: `ST-*`

Proposed principles take identifiers `ST-001`, `ST-002`, … in a **new series**.

`R1.3` of `rest-api-standard.md` freezes `HS-001`–`HS-020`,
`AC-001`–`AC-021`, and `OP-001`–`OP-025` as research-provenance keys into the
Gate C decision layer. Minting a streaming principle in any of them would give
a Phase 6 rule fabricated 1.0 lineage. `ST-*` is the streaming series, matching
the existing pattern where `baseline-01` produced `HS-*`, `baseline-02`
produced `AC-*`, and `baseline-03` produced `OP-*`.

Drafting `R1.3` in Phase 6 will need to name `ST-*` once the series is
ratified, so the freeze list stays complete.

## Connect to ratified rules; do not fork them

Four 1.0 decisions already govern territory this leaf touches. Each is
**ratified and out of scope for re-litigation**; the leaf's job is to state how
streaming composes with it, not to re-derive it.

| Ratified rule | What it fixes | The streaming question it raises |
| --- | --- | --- |
| `R5.12`, `R5.13` (riding `AC-003`, `AC-004`) | Every application-generated error servable as `application/problem+json`; `type` a stable https URI 1:1 with `code` | What satisfies this obligation for an error raised after `200` is committed and no status code remains available? |
| `R10.9` (riding `AC-019` + `baseline-02i`) | A `202` body MUST identify its operation resource; `Location` SHOULD denote the operation, never the result | When streaming and a `202` operation resource both exist for one capability, how do they compose without forking this contract? |
| `R4.17` (Gate E ruling) | Cross-origin browser clients: standard-bound headers listed in `Access-Control-Expose-Headers` | Does a streaming response add standard-bound headers, and does `EventSource`'s inability to set request headers create an authentication gap this standard must address? |
| `R1.8`, `R1.9`; §1.10 reserved names | Reservation discipline; the `dry_run` rejection guard as its model | Does streaming need a reserved name (`stream`?), and does an unimplemented streaming request need a rejection guard on the `dry_run` model? |

## Two tests any proposed principle must pass

Both come from precedents already set in this repository, and a proposal that
fails either will not survive the ratification gate.

1. **The IANA-registry test.** `Operation-Location` was declined in
   `baseline-02i` on IANA-registry grounds. Any header name or media type this
   leaf would bless — including `application/x-ndjson` — must be checked in the
   registry, and an unregistered name must be proposed as such, openly, with
   the registration gap stated.
2. **The classification test.** Phase 4 relabeled twelve rules because
   `[POLICY]` had been presented as protocol law. Every proposed principle
   carries its classification (`[FACT]` protocol requirement / `[COMPARATIVE]`
   selected default / `[POLICY]` project choice) and its confidence. A
   principle resting on a WHATWG living standard, an expired draft, or a
   convention with no standards body is not a protocol requirement, however
   universal its adoption.

## Deliverable-shape consequence for this leaf

The owner ruled a compact normative §13 paired with an informative companion
document. That makes **normative weight a scarce resource**, so this leaf must
do something the earlier baseline leaves did not: for every proposed principle,
state whether it needs normative force in §13 or belongs in the companion as
guidance. A leaf that proposes thirty rules of equal weight would hand the
drafting step an unsorted pile and defeat the progressive-disclosure ruling.

## Boundaries specific to this leaf

This leaf proposes; it does not ratify, and it does not draft. Ratification is
the Step 2 owner walk landing in
`research/decisions/baseline-04-streaming.decision.md`. Drafting is Step 3. Per
`research/CONSTRAINTS.md`, do not draft normative standard prose during
research execution — proposed rule sentences in a principles table are the
expected output, a §13 draft is not.
