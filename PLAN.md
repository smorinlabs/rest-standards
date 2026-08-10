# REST Standards Project Plan

## Objective

Create a public, language-agnostic REST API design standard that is grounded in
current primary sources, records where judgment begins, and can be reviewed and
maintained with the same explicit decision discipline used for `cli-standards`.

The repository will distinguish three kinds of material:

1. externally supported facts and protocol requirements;
2. recommended defaults selected from legitimate alternatives; and
3. project policy that is intentionally stricter than the underlying standards.

## Research organization

All research lives flat under `research/`, split into `prompts/` and `reports/`,
with `decisions/` added in Phase 2. Files pair by a shared filename stem rather
than by directory. The full naming grammar and the current inventory are in
[`research/README.md`](research/README.md); the rules for running new research
are in [`research/CLAUDE.md`](research/CLAUDE.md).

Two research series exist. They differ in mode, and the distinction is
load-bearing for every gate below.

| Series | Mode | Question | Output | Status |
| --- | --- | --- | --- | --- |
| `survey` | Descriptive | What do eight reference APIs and the standards actually do? | Comparison plus a contested-axes register. No recommendations. | 8 prompts, 10 runs, complete |
| `baseline` | Prescriptive | What should this standard require? | Proposed normative-principles tables with `MUST`/`SHOULD`/`MAY`. | 3 prompts run 2026-07-25 + 9 follow-up leaves (4 Gate B, 5 Gate C); complete |

The original plan budgeted three descriptive threads. In practice the
descriptive work was done by the eight-part `survey` series, and the three
original prompts turned out to be prescriptive — each requires a proposed
normative-principles table and declares itself complete only on recommending an
actionable baseline. Both series are retained because they answer different
questions; the `survey` series supplies the evidence the `baseline` series
argues from.

**A `baseline` report proposes rules; it does not ratify them.** Its principles
tables are research output carrying provisional IDs and confidence levels.
Nothing becomes project policy until it passes Gate C and is recorded in
`research/decisions/`.

## Scope

The first release will cover:

- resource and URI design;
- HTTP methods, safety, idempotency, status codes, and conditional requests;
- representations, content negotiation, errors, and API description contracts;
- collection querying, filtering, sorting, and pagination;
- concurrency control, retry safety, bulk work, and asynchronous operations;
- caching;
- authentication, authorization, and common API security boundaries;
- versioning, compatibility, and deprecation;
- rate limits, observability, webhooks, and operational failure behavior; and
- conformance checks, exceptions, and worked examples.

The standard will not become a framework tutorial, an implementation-language
guide, or a general distributed-systems handbook. GraphQL, RPC, event streaming,
and messaging are considered only when defining REST boundaries or interoperability.

## Delivery phases and gates

### Phase 0: Reviewable bootstrap — complete

- Created this plan and the research lineage structure.
- Created the three `baseline` prompts with neighbouring `.framing.md` records.
- Published the repository publicly as `smorinlabs/rest-standards`.

**Gate A — passed.** The plan and the three prompt scopes were approved.

### Phase 1: Research execution — complete, awaiting Gate B

**`survey` series — complete.** Eight prompts run across ten executions,
covering foundations and spec mechanics, structure, representations and errors,
collections, reliability, lifecycle and operations, and webhooks. Two prompts
were run twice; both runs of each are retained because divergence between
independent runs is a confidence signal for the decision phase.

**`baseline` series — complete.** Three prescriptive threads run 2026-07-25
with primary-source status verified live, producing 66 proposed principles:
`HS-001`–`HS-020` (HTTP semantics), `AC-001`–`AC-021` (API contracts), and
`OP-001`–`OP-025` (operational practice).

The baseline series produced material currency corrections to the survey
evidence, tabulated in [`research/README.md`](research/README.md). The
load-bearing ones for Gate C: RFC 10008 (QUERY) exists and no survey report
covers it; OpenAPI 3.2.0 supersedes the 3.1 baseline the survey assumed; the
IETF Idempotency-Key draft **expired** on 2026-04-18, so no idempotency rule
can claim standards conformance; and the entire security and observability
standards layer — RFC 9421, RFC 9700, RFC 9325, W3C Trace Context, OWASP —
appears in no survey report.

**Gate B — passed 2026-07-25.** Four narrow leaves were identified and run,
each testing a specific weak link rather than re-synthesizing existing
evidence. All four changed something:

| Leaf | Effect |
| --- | --- |
| `baseline-01b-query-deployment` | `HS-009` confirmed at `MAY` — no CDN implements body-keyed caching for QUERY |
| `baseline-02b-openapi-tooling` | `AC-001` **revised** — the 3.2-only mandate was unsupportable; JSON Schema pin strengthened |
| `baseline-02c-problem-details-adoption` | `AC-003` **strengthened** — framework defaults support the historical-inertia argument |
| `baseline-03b-signatures-and-ratelimit` | `OP-016` raised; `OP-010` lowered with a dated expiry contingency |

A fifth candidate — strong-validator generation cost — was dropped as
implementation-dependent and not resolvable by research.

Research is complete. Proceed to Gate C.

### Phase 2: Convert research into explicit decisions

- Merge the `survey` contested-axes registers into one master register.
- Reconcile it against the `baseline` proposed-principles tables, which argue
  from the same evidence toward specific rules.
- Record the outcome in `research/decisions/`, one file per stem, following the
  same naming convention as prompts and reports.
- Classify every candidate rule as protocol requirement, evidence-backed
  default, project policy, exception, or unresolved question.
- Record disagreements between sources rather than silently averaging them.
- Compare both runs of any twice-run prompt before ratifying a rule that
  depends on it. For `survey-06-lifecycle-operations` this was checked on
  2026-07-25: the two runs agree on versioning transport and differ only in
  emphasis, so agreement across independent runs raises confidence there
  rather than blocking.

**Gate C:** Review and approve the policy decisions before drafting normative text.
This is the point at which a proposed rule becomes project policy.

### Phase 3: Draft the standard

Draft a normative document using RFC 2119/8174 requirement language and stable
rule identifiers. The working outline is:

1. purpose, audience, terminology, and conformance;
2. resources and URI modeling;
3. methods, safety, idempotency, and conditional operations;
4. requests, representations, negotiation, and schemas;
5. success status codes and standardized error responses;
6. collections, pagination, filtering, and sorting;
7. caching and concurrency;
8. authentication, authorization, and security;
9. lifecycle, compatibility, versioning, and deprecation;
10. asynchronous work, bulk operations, and webhooks; and
11. rate limits, retries, observability, and operational behavior.

Appendices will include a conformance checklist, exception process, compact cheat
sheet, OpenAPI mapping, and a worked fictional API.

**Drafting items folded in from the CLI-standards gap review
(`docs/reviews/2026-08-09-cli-standards-gap-review.md`), 2026-08-09** — the
review's Tier 1 decisions were ratified in the Gate C addendum; these are the
drafting-plane remainders, mapped onto the outline above:

- §1 (conformance): tier system (`internal`/`partner`/`public`) +
  applicability switches with N/A-with-reason discipline; the
  no-silent-deviation clause + per-API conformance-note template; rule-ID
  mapping policy (provenance IDs ↔ section IDs) settled before section 1 is
  written.
- §2/§4: reserved query-parameter and header inventory table (the cheat
  sheet's core); content-negotiation defaults; noun-naming rules; PII MUST
  NOT appear in URIs.
- §5: field-level validation error shape (`errors[]` extension with JSON
  Pointer targets); problem-type/code catalog obligation.
- §6: breaking-change taxonomy + frozen-surface enumeration (with §9);
  empty-collection-is-200 row.
- §7/§8: destructive-operation guards (mandatory `If-Match` on
  concurrent DELETE; unfiltered collection-DELETE ban); secret/PII
  redaction in problem `detail`.
- §10: polling/cancellation guidance for operation resources
  (`Retry-After` on 202).
- §11 + new section: consolidated client-obligations section (OP-012,
  OP-017, AC-012, AC-013 + client timeouts, honor `Retry-After`, TLS
  verification, unknown-field tolerance).
- Appendices: framework/gateway mapping (evidence in baseline-02b/02c/02e/
  03g); executable conformance fixtures (Spectral ruleset from the ratified
  regexes + live-probe table); in-document Decision Log (Part II) linking to
  `research/decisions/`; worked example annotated with rule IDs.
- Open scope questions carried to Gate D: client/SDK configuration
  conventions appendix; SSE/streaming posture (reaffirm the boundary in
  non-goals or open a decision item).

**Gate D — passed 2026-08-09.** The owner approved the draft (Parts
I–III of `rest-api-standard.md` plus `conformance/spectral.yaml`) for
Phase 4 systematic review. Rulings recorded at the gate:

- The 18 apparatus provisions (Part II register) are **ratified en
  bloc**; the register is their ratification record. The path-parameter
  naming candidate raised during drafting was not ratified and stays
  open for Phase 4.
- **SDK/client configuration conventions stay excluded** — the
  `CONSTRAINTS.md` boundary is upheld for 1.0; revisit after release.
- **SSE/streaming is a named feature goal, deliberately sequenced after
  this body of work completes** — added as Phase 6 below; the draft's
  §1.2 non-goals state the deferral.

### Phase 4: Decision-focused review

**In progress 2026-08-09.** The internal three-agent pass (consistency /
ambiguity / domain-gap) is complete — 22 findings, recorded in
[`docs/reviews/2026-08-09-phase-4-internal-review-findings.md`](docs/reviews/2026-08-09-phase-4-internal-review-findings.md);
fixes landed on the Phase 4 branch. The Codex second lens ran 2026-08-10
against the corrected text (10 findings, all dispositioned — same review
doc), followed by a raw-RFC classification sweep of every
protocol-requirement label. Remaining before Gate E: the owner walk for
the four decision-touching items and the closing structural pass.

Run the recovered `cli-standards` review pattern:

1. internal consistency and cross-reference pass;
2. independent second pass for missing and conflicting rules;
3. ambiguity search;
4. classification of research questions versus policy choices;
5. one-decision-at-a-time review;
6. domain gap audit; and
7. atomic updates to the rule, rationale or decision log, checklist, and worked
   example whenever a decision changes.

Finish with structural checks for duplicate IDs, missing anchors, stale references,
undefined terms, and RFC-keyword consistency.

**Gate E:** Approve the version 1.0 candidate.

### Phase 5: Publish and maintain

- Review through pull requests.
- Tag the accepted first release.
- Maintain an explicit decision log and changelog.
- Re-run focused research when underlying specifications or dominant operational
  practice materially change.

### Phase 6: Streaming extension (post-1.0)

Owner ruling at Gate D (2026-08-09): SSE and streaming responses are a
**feature goal**, deliberately sequenced after the v1.0 body of research
and publication completes. The original scope exclusion predates streaming
becoming the default AI-API shape; this phase closes that gap without
destabilizing 1.0. Scope when opened:

- research the streaming posture (SSE, long-poll, streaming HTTP
  responses) with the same evidence discipline as the baseline series;
- ratify a decision through the standard's decision layer; and
- extend the standard — a new section or an extension document — under
  the Part II amendment rule (semantic versioning, atomic five-surface
  updates).

## Planned artifacts

| Artifact | Purpose | Created in | Status |
| --- | --- | --- | --- |
| `PLAN.md` | Scope, sequence, gates, and definition of done | Phase 0 | Done |
| `research/README.md` | Naming convention and research inventory | Phase 0 | Done |
| `research/CLAUDE.md` | Rules for running and filing research | Phase 0 | Done |
| `research/CONSTRAINTS.md` | Shared research boundaries | Phase 0 | Done |
| `research/prompts/baseline-*.framing.md` | Prompt provenance and shaping record | Phase 0 | Done |
| `research/prompts/baseline-*.prompt.md` | Prescriptive research tasks | Phase 0 | Done |
| `research/prompts/survey-*.prompt.md` | Descriptive research tasks | Phase 1 | Done |
| `research/reports/survey-*.report.*.md` | Descriptive evidence, ten runs | Phase 1 | Done |
| `research/reports/baseline-*.report.*.md` | Proposed normative baselines, 66 principles | Phase 1 | Done |
| `research/decisions/*.decision.md` | Ratified conclusions and consequences | Phase 2 | **Complete 2026-08-09** — Gate C ratified all 66 principles (walked decisions + three per-report batches), plus the Gate C addendum (same day): PATCH format, sorting cluster, status-code rows, dry-run, action verbs, from the CLI-standards gap review. Index in `research/README.md`. The Phase 2 "master register" is realized as that decision index plus the per-stem decision files, rather than a separate merged-register artifact — each contested axis was resolved decision-by-decision instead. |
| Normative standard | Stable rules and rationale | Phase 3 | **Complete 2026-08-09 — Gate D passed same day** — `rest-api-standard.md`: §1–§12 (125 rules after the Phase 4 owner walk, each with provenance, classification, confidence), Part II Decision Log (provenance map + apparatus register; the 18 apparatus provisions ratified en bloc at Gate D, marked in place), Appendices A–G, Spectral ruleset (`conformance/spectral.yaml`, execution-verified 2026-08-09 against `conformance/fixture-violations.yaml`) |
| Checklist and worked example | Conformance and integration proof | Phase 3–4 | **Drafted 2026-08-09** (Appendices A and E of `rest-api-standard.md`); refined during Phase 4 review |

## Definition of done for version 1.0

- Every normative rule has a stable identifier and an explicit strength.
- Every external claim has traceable source lineage.
- Every project policy is labeled as policy rather than presented as protocol law.
- Conflicts and exceptions have recorded resolutions.
- The checklist and worked example cover every normative section.
- Structural verification passes with no duplicate IDs, broken internal links,
  undefined normative terms, or untracked decision changes.
