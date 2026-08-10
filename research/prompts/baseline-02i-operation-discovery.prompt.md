# baseline-02i — Operation-resource discovery on 202 (prompt)

*Dispatched 2026-08-10 during the Phase 4 owner walk, at the owner's
direction ("research first before recommending"), as a narrow follow-up
leaf under `baseline-02` / `AC-019`. Run by an opus research subagent
with the repository evidence rules embedded.*

## Question

When a REST API accepts long-running work with `202 Accepted` and an
operation resource (rule `R10.1`, riding `AC-019`), how should the
client learn the operation resource's URI — and should the standard
mandate, recommend, or merely permit the `Location` header for it?

## Required coverage

1. RFC 9110 §10.2.2 (`Location` semantics) — the operative sentences
   quoted from raw RFC text; what is defined for `202`.
2. Vendor async patterns, primary-doc-verified: Azure REST guidelines
   (`Operation-Location` vs `Location`), Microsoft REST guidelines,
   Google AIP-151, Microsoft Graph, GitHub, Stripe, Zalando rule 253,
   Kubernetes conventions, others as checkable.
3. The three modern AI providers (mandatory per project rule): OpenAI,
   Anthropic, Google Gemini — long-running job discovery mechanics.
4. Any standards-track or guideline text recommending `Location` on
   `202`; draft status and expiry recorded for any Internet-Draft.
5. Trade-off analysis: header vs body discovery, including whether the
   ratified body-envelope-over-`Link`-headers pagination posture cuts
   the same way here.

## Evidence rules (from `research/CONSTRAINTS.md` / `research/CLAUDE.md`)

Direct URL + authority class + publication/access date on every material
claim; primary sources only; drafts never presented as standards;
`[FACT]`/`[COMPARATIVE]`/`[INFERENCE]`/`[POLICY]` labels; conflicts
surfaced, never averaged; two-source minimum on load-bearing claims.

## Output

A report with: TL;DR + recommendation with proposed rule sentence ·
standards layer · vendor survey table · AI-provider rows · conflict and
trade-off analysis · confidence and invalidating conditions.
