# baseline-04 — Streaming posture (prompt)

*Phase 6, prescriptive leaf. Dispatch only after
`survey-08-streaming`'s report is filed and renamed — this prompt cites it.
Shaping decisions in [`baseline-04-streaming.framing.md`](baseline-04-streaming.framing.md);
shared Phase 6 scope in [`survey-08-streaming.framing.md`](survey-08-streaming.framing.md).
Run by an opus research subagent with the repository evidence rules embedded.*

## Question

An existing, released standard (`rest-api-standard.md` v1.0.0, 127 rules)
excludes streaming. Given the field evidence in the `survey-08-streaming`
report, **what should this standard require of an API that delivers a response
incrementally over a single HTTP request** — Server-Sent Events, long-polling,
or a streaming HTTP body (chunked NDJSON/JSON Lines)?

WebSockets are an explicit non-goal by owner ruling: after the `101` upgrade
the exchange is no longer HTTP request/response, and none of this standard's
apparatus binds it. Propose the boundary sentence, not WebSocket rules.

## Required inputs — read before searching

1. **`research/reports/survey-08-streaming.report.<date>.md`** — the field
   evidence and its Contested Axes Register. Every proposed principle cites
   rows from it. Independent verification is welcome; replacing its evidence
   with your own search is not.
2. **`rest-api-standard.md`** — in particular §1.5 (rule IDs and the frozen
   research series), §1.6 (classification), §1.8 (applicability switches),
   §1.10 (reserved names), §4.3 (`R4.17`, CORS header exposure), §5 (`R5.12`,
   `R5.13`, the `problem+json` obligation), §10.1 (`R10.9`, `202` operation
   discovery), and §12 (client obligations).
3. **`research/decisions/baseline-02-api-contracts.decision.md`** — the
   `baseline-02i` operation-discovery record, for the reasoning that declined
   `Operation-Location` on IANA-registry grounds.

## Required coverage

Produce a position, with evidence, on each of these. Where the survey found the
field genuinely split, say so and recommend conditionally rather than averaging.

1. **Negotiation.** Which mechanism should the standard require — a request-body
   flag, `Accept: text/event-stream`, a query parameter, or a distinct
   endpoint — and what must happen when a client requests streaming from an
   endpoint that does not offer it. Test the last question against the `R1.9`
   `dry_run` rejection guard, which exists because silently ignoring an
   unimplemented request modifier is a documented hazard in this standard.
2. **Framing and media type.** What the standard should require on the wire,
   and whether it can require anything version-neutral given that chunked
   transfer coding is HTTP/1.1-only. Apply the IANA-registry test to every
   media type considered, `application/x-ndjson` included.
3. **Event typing and termination.** Whether named event types and an explicit
   terminal signal should be required, and specifically whether a client must
   be able to distinguish a normal end from a truncated connection.
4. **Errors after the status is committed.** The hardest interaction. `R5.12`
   requires every application-generated error to be servable as
   `application/problem+json`; a `200` already on the wire cannot become a
   `4xx`. Propose what discharges that obligation mid-stream — an in-band
   problem document in a typed error frame, trailer fields, or a documented
   carve-out — and say plainly whether your proposal amends `R5.12`, carves out
   from it, or composes with it unchanged. Also address what the client must do
   on truncation, connecting to §12.
5. **Resumption.** Whether resumption is required, recommended, or permitted,
   and on what evidence. If the survey shows most references do not implement
   it, a `MUST` needs an argument stronger than the mechanism's existence.
6. **Relationship to `R10.9` and §10 asynchronous operations.** When both
   channels exist for one capability, what must be true. This must **compose
   with** `R10.9`, not fork it: propose how the operation identity that `R10.9`
   binds into the `202` body appears in the streaming channel, and whether the
   two channels must agree on terminal state.
7. **Browser clients and CORS.** Whether streaming adds standard-bound headers
   that `R4.17` must list, and what the standard should say about `EventSource`
   being unable to send `Authorization`. If the field's answer is
   query-parameter tokens, state the security consequence rather than blessing
   it silently — §8 already governs credential handling.
8. **Keep-alive, timeouts, and buffering.** What is genuinely required of the
   server versus what is deployment advice belonging in the companion.
9. **Applicability switch and reserved names.** Propose the `streaming` switch
   scope (which rules it gates) per §1.8, and any reserved name the phase needs
   registered in §1.10.
10. **Long-polling.** Whether it warrants its own rules or is adequately covered
    by the general rules plus §10; if separate, what an expired hold returns.
11. **The §1.2 boundary sentence.** Propose replacement text for the current
    streaming deferral, stating the WebSockets non-goal and its reason.

## Constraints

- **Propose; do not ratify, and do not draft.** The output is a
  proposed-principles table. Ratification is the owner walk; §13 prose is a
  later step. Per `research/CONSTRAINTS.md`, do not draft normative standard
  prose during research execution.
- **Provisional IDs in the new `ST-*` series only.** `R1.3` freezes `HS-*`,
  `AC-*`, and `OP-*` as 1.0 provenance keys; minting there would fabricate
  lineage.
- **Do not re-litigate any 1.0 decision.** Where a ratified rule constrains
  your answer, say so and work within it. If evidence genuinely contradicts a
  ratified rule, do not quietly propose around it — raise it as a flagged
  conflict for the owner.
- **Normative weight is scarce.** The deliverable is a compact §13 plus an
  informative companion, so every principle must declare which it belongs in.

## Evidence rules (from `research/CONSTRAINTS.md` / `research/CLAUDE.md`)

Direct URL, authority class, and publication or access date on every material
claim; primary sources first; two-source minimum on load-bearing claims; an
Internet-Draft or vendor convention never presented as a published standard,
with draft number and expiry recorded; a WHATWG living standard named as such;
conflicts surfaced and adjudicated with a stated reason, never averaged;
`[FACT]` / `[COMPARATIVE]` / `[INFERENCE]` / `[POLICY]` labels applied
distinctly. This repository is public with push protection enabled — placeholder
credentials only (`Bearer <access-token>`).

## Output

1. **Executive recommendation** — the streaming posture in a paragraph, plus
   the single hardest call and why.
2. **Standards-and-currency matrix** — every source with URL, authority class,
   version or date, and access date.
3. **Composition analysis** — one subsection per ratified rule this touches
   (`R5.12`/`R5.13`, `R10.9`, `R4.17`, `R1.8`/`R1.9`, §12), stating whether the
   proposal composes, amends, or carves out, and what the amendment rule would
   have to change on each of its five surfaces.
4. **Proposed normative-principles table** — every principle with: `ST-###`
   provisional ID · `MUST`/`SHOULD`/`MAY` strength · concise proposed rule
   sentence · rationale · classification (`[FACT]`/`[COMPARATIVE]`/`[POLICY]`)
   · applicability (switch scope, exceptions) · evidence URLs and the
   `survey-08` register rows relied on · confidence · **and a
   `Normative §13` / `Companion` column** marking where the material belongs.
5. **Declined options** — each with the reason it was declined, in the shape
   the decision layer can cite directly.
6. **Proposed §1.2 boundary text** and the proposed `streaming` switch
   definition.
7. **Conflicts and open questions** — separating what more research could
   settle from what is an owner policy decision, since only the latter goes to
   the ratification walk.
8. **Overall confidence** and the conditions that would invalidate the
   recommendation, including any dated re-check trigger worth adding to the
   register in `research/README.md`.

The research is complete only if it hands the ratification gate a coherent,
decidable streaming baseline — including an answer to the post-commit error
problem and to the streaming-versus-operation-resource relationship — rather
than a catalog of what vendors do.
