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

**Complete 2026-08-10** (started 2026-08-09). The internal three-agent
pass (consistency / ambiguity / domain-gap) ran first — 22 findings,
recorded in
[`docs/reviews/2026-08-09-phase-4-internal-review-findings.md`](docs/reviews/2026-08-09-phase-4-internal-review-findings.md);
fixes landed on the Phase 4 branch. The Codex second lens ran 2026-08-10
against the corrected text (10 findings, all dispositioned — same review
doc), followed by a raw-RFC classification sweep of every
protocol-requirement label. The owner walk ruled all four
decision-touching items 2026-08-10 (switches pruned; `cancel` broadened;
R4.16 path placeholders; R10.9 operation discovery, backed by the
`baseline-02i` research leaf and a filed decision record), and the
closing structural pass ran clean. **Phase 4 completed 2026-08-10 and
Gate E passed the same day (recorded below); the standard stands at 127
rules, checklist 127/127, fixture 12/12.**

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

**Gate E — passed 2026-08-10.** The owner approved
`rest-api-standard.md` (with the `conformance/` fixtures) as the
version 1.0 candidate. Ruled at the gate: the CORS header-exposure item
from `baseline-02i`, option (a) — new rule R4.17 mandates
`Access-Control-Expose-Headers` coverage for cross-origin browser
clients (rule count 127). The four Phase 4 apparatus rulings (switch
pruning, `cancel` scope, R4.16, R10.9) become fully binding with this
approval.

### Phase 5: Publish and maintain

**Release executed 2026-08-10:** version flipped to 1.0.0,
`CHANGELOG.md` created, `v1.0.0` tagged on the release merge. Ongoing
maintenance mode:

- Review through pull requests — the standing practice (every gate landed
  via a bot-reviewed PR: #3–#6 and release PR #7).
- Decision log: Part II of the standard + `research/decisions/`;
  changelog: `CHANGELOG.md` under the Part II amendment rule.
- Re-run focused research when underlying specifications or dominant
  operational practice materially change — the dated re-check register
  lives in `research/README.md` (next scheduled: `OP-010` semi-annual,
  2027-02-09).

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

**Opened 2026-08-10 — in progress.** Two owner rulings scope the phase:

1. **Coverage:** SSE, long-polling, and streaming HTTP bodies (chunked
   NDJSON/JSON Lines). **WebSockets are an explicit non-goal** — after the
   `101` upgrade the exchange leaves HTTP, so no status-code, media-type,
   `Problem Details`, or applicability-switch machinery in this standard
   binds it. §1.2's open deferral is replaced by a stated boundary.
2. **Deliverable shape:** §13-in-document with the body held in a separate
   referenced document, for progressive disclosure. The reading recorded
   here — **confirmed by the owner 2026-08-10 as `P6-D0`** — is
   the detail split: a **compact normative §13** in
   `rest-api-standard.md` carrying the `R13.x` rules, paired with a
   separate **informative companion document** holding the explanatory
   body (mechanism comparison, wire examples, vendor evidence, client
   guidance). Progressive disclosure — an API that does not stream
   declares the `streaming` switch off and never opens the companion —
   while the conformance surface stays single (one Appendix A checklist,
   one Part II provenance map, one Spectral ruleset). The exact
   normative/informative line is set at drafting, once the rule count is
   known. Additions are a MINOR bump under the amendment rule: **v1.1.0**.

Step sequence and status:

| Step | What | Status |
| --- | --- | --- |
| 0 | Scope framing + research prompts (`survey-08-streaming`, `baseline-04-streaming`) | **Done 2026-08-10** |
| 1a | Descriptive leaf `survey-08-streaming` | **Done 2026-08-10** — 1,747-line report, 44 sources, 14 contested axes; arrival check passed; RFC 8895 correction annotated at §3.0a |
| 1b | Prescriptive leaf `baseline-04-streaming` | **Done 2026-08-10** — 898-line report, 20 proposed `ST-*` principles, six conflicts with ratified 1.0 rules raised |
| 2 | Ratification walk (Gate C pattern) → `research/decisions/baseline-04-streaming.decision.md` | **Done 2026-08-10** — six forks walked (`P6-D0`–`P6-D5`), fifteen principles ratified en bloc |
| 3 | Draft §13 + companion, atomic across the amendment rule's five surfaces | **Done 2026-08-10** — 12 new rules (`R13.1`–`R13.11`, `R12.10`); standard at **139 rules, checklist 139/139, fixture 14/14** |
| 4 | Review waves (internal lenses → Codex second lens → raw-source verification → structural suite) | **Done 2026-08-10** — three internal lenses (consistency, ambiguity, missing-and-conflicting) dispositioned; Tier A/B/C ruled by the owner; Codex second lens on the corrected text found a release-blocking §6 gap plus five factual corrections, all fixed; raw-source verification of every new normative citation; fixtures extended to 14/14 |
| 5 | Release v1.1.0 | Release PR **#8** — changelog written, bot cycle worked (Copilot, Greptile, CodeRabbit). `v1.1.0` is tagged on this PR's merge commit, matching the `v1.0.0`/PR #7 pattern; the tag is applied at merge, so a reader of this branch before merge sees the intent and a reader after sees the fact |

Drafting in Step 3 must also amend §1.5's "the `R<section>` prefix space is
fixed at twelve normative sections" declaration, rewrite the §1.2 streaming
deferral, add the `streaming` applicability switch to §1.8, register any
reserved names in §1.10, and name the `ST-*` series in `R1.3`'s freeze list.

### Phase 7: Skill apparatus (post-1.0)

The standard is a document; this phase gives it an applier. `rest-standards`
is an agent skill that reads `rest-api-standard.md` live and applies it in
four modes — `plan`, `check`, `review`, `audit` — the REST counterpart to the
`cli-standards` skill that ships beside the CLI Design Standard.

This closes the second half of the transfer begun at Gate C. The gap review
(`docs/reviews/2026-08-09-cli-standards-gap-review.md`) moved the CLI
standard's *content* here — items B.8 and B.12 are why §1.7–1.9 exist. This
phase moves its *operating apparatus*. Design:
[`docs/specs/2026-08-10-rest-standards-skill-design.md`](docs/specs/2026-08-10-rest-standards-skill-design.md).

Scope:

- author `.claude/skills/rest-standards/` — `SKILL.md` plus four procedure
  references (`scoping`, `planning`, `review`, `audit`);
- ship **no normative content in the skill**: §1.7–1.9 are already ratified,
  so the skill reads tiers, switches, and the conformance-note template live
  rather than restating them, and navigates by grepping the standard's
  headings rather than shipping a section index that a Part II amendment
  would silently stale;
- scale audit depth by **evidence plane** — contract document
  (`conformance/spectral.yaml`), source, deployed runtime (Appendix G probes)
  — because §1.7 tiers name an audience, not a depth;
- gate the runtime plane: opt-in, non-production only, read-only probes
  first, mutating and quota probes behind a second confirmation (owner
  ruling 2026-08-10); and
- add the repository's first `.claude/settings.json` startup announcement,
  which names the skill and points at `docs/skills/rest-standards.md` — both
  therefore Gate F conditions, not optional follow-ups.

Placement is deliberately **not** part of this phase. The skill exists only
on this branch until it merges, so a `~/.agents/skills/rest-standards`
symlink created here would dangle when the worktree is removed. Placement
follows the merge and the fast-forward pull; the quality gate runs against
the worktree instead.

**The v1.1.0 rebase — why this phase absorbed §13 mid-flight.** Phase 6
merged while this work was in progress, and the note left at this heading
asked for the self-test to be re-run against the extended standard. It was:
the branch took v1.1.2, and the re-run is the phase's acceptance evidence
rather than a follow-up. The event also proved the design's central bet in
both directions. The five skill files read the standard live, so §13 reached
them with no edit — that half held. But three *hardcoded snapshots* the
implementation plan had mandated went stale exactly as predicted: a section
count, a rule count, and a twelve-section sweep list that silently omitted
§13. They were removed rather than corrected, so the next amendment cannot
repeat the failure.

**Gate F — skill accepted.** The skill's `audit` mode, run against the
Appendix E worked example (Bloom Orders API) at the current standard version,
reproduces that appendix's rule-ID annotations and conformance note;
`skillsmith verify` passes on both `claude-code` and `codex`; the Spectral
ruleset still fires its full fixture count when invoked through the skill's
own path; and `git diff origin/main...HEAD` shows no change to
`rest-api-standard.md` or `conformance/`. No rule text changes in this phase
— anything the skill exposes as a missing rule is an amendment under Part II,
not a skill addition.

### Phase 8: Streaming's unresolved interactions (post-1.1.0)

Opened 2026-08-10 by the Phase 6 review walk. **Numbered 7 at creation and
renumbered to 8 in version 1.1.1**: the skill-apparatus phase had claimed
Phase 7 twenty-nine minutes earlier on a branch not yet merged, so this
phase — which merged first but was declared second — yields the number. The
renumber is recorded rather than silently applied because two sections
briefly carried the same phase number in released text. The three internal review
lenses over the drafted §13 found that streaming disturbs more of the 1.0
standard than the research leaves had been pointed at. Collisions where a
ratified MUST became unsatisfiable were fixed inside Phase 6 (Tier A:
`R11.2`, `R11.5`, `R2.11`, `R6.1`). **Five interactions where the standard is
merely silent were deliberately deferred here** rather than ruled without
evidence — the same reasoning that declined the single-lighter-leaf option
at the phase opening.

The register lives in **§13.4** of `rest-api-standard.md`, so a reader who
hits one of these knows the silence is a decision. Scope when opened:

| Item | The unresolved question |
| --- | --- |
| Frame-vocabulary versioning (§9.3) | `R9.4` does not classify frame-type names. Renaming a terminal frame reads as compatible, while `R12.10` makes every deployed client ignore it and report truncation on every success. **Sharpest known failure; interim posture recorded in §13.4.** |
| Stream authorization lifetime (§8) | `R8.6` authorizes a request; a stream is one request that can outlive the credential that opened it. |
| Caching posture for streams (§7) | `R7.3` tier 1 revalidates via strong `ETag`, which a stream cannot supply. |
| Idempotency-key replay of a streaming request (§3) | `R3.9`'s "stored response" is undefined for a stream, and overlaps `R13.10` resumption. |
| Resource ceilings for streams (§11) | `R11.1`'s published maxima cover page size, expansion depth, and bulk count — not stream duration or per-principal concurrency. |

Method when opened: a research leaf per cluster under the two-series
discipline, a ratification walk, then amendments under the Part II amendment
rule. Some items may prove to need no rule; that verdict is itself recorded.

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
| `research/reports/survey-*.report.*.md` | Descriptive evidence — ten runs in Phase 1; `survey-08` adds an eleventh in Phase 6 | Phase 1 | Done |
| `research/reports/baseline-*.report.*.md` | Proposed normative baselines — 66 principles in Phase 1 (`HS-*`, `AC-*`, `OP-*`); `baseline-04` adds `ST-001`–`ST-020` in Phase 6, for 86 | Phase 1 | Done |
| `research/decisions/*.decision.md` | Ratified conclusions and consequences | Phase 2 | **Complete 2026-08-09** — Gate C ratified all 66 principles (walked decisions + three per-report batches), plus the Gate C addendum (same day): PATCH format, sorting cluster, status-code rows, dry-run, action verbs, from the CLI-standards gap review. Index in `research/README.md`. The Phase 2 "master register" is realized as that decision index plus the per-stem decision files, rather than a separate merged-register artifact — each contested axis was resolved decision-by-decision instead. |
| Normative standard | Stable rules and rationale | Phase 3 (extended in Phase 6) | **Released 1.0.0 2026-08-10** (Gate E-approved; drafted and Gate D-approved 2026-08-09; Phase 4 review complete) — §1–§12, 127 rules. **Extended to 1.1.0 in Phase 6** with §13 streaming: `rest-api-standard.md` now carries **139 rules**, each with provenance, classification, and confidence, plus Part II Decision Log (provenance map + apparatus register), Appendices A–G, the Spectral ruleset (`conformance/spectral.yaml`, re-verified 2026-08-10 against `conformance/fixture-violations.yaml`, 14/14), and the informative companion `streaming-profile.md` |
| Checklist and worked example | Conformance and integration proof | Phase 3–4 | **Drafted 2026-08-09** (Appendices A and E of `rest-api-standard.md`); refined during Phase 4 review |
| `.claude/skills/rest-standards/` + `docs/skills/rest-standards.md` | Agent skill applying the standard in four modes, and its documentation page | Phase 7 | Designed 2026-08-10 (`docs/specs/2026-08-10-rest-standards-skill-design.md`), authored and self-tested the same day. Five files: `SKILL.md` plus four procedure references. Acceptance evidence in `docs/reviews/2026-08-10-phase-7-skill-self-test.md` |
| `research/prompts/survey-08-streaming.*` · `baseline-04-streaming.*` | Phase 6 scope framing and the two research leaves | Phase 6 | **Done 2026-08-10** — both leaves framed, prompted, and run |
| `research/reports/survey-08-streaming.report.*` · `baseline-04-streaming.report.*` | Streaming field evidence, then proposed `ST-*` principles | Phase 6 | **Done 2026-08-10** — descriptive run 1,747 lines / 44 sources / 14 contested axes (carries a dated §3.0a correction: RFC 8895); prescriptive run 898 lines / 20 `ST-*` principles |
| `research/decisions/baseline-04-streaming.decision.md` | Ratified streaming posture (Step 2 owner walk) | Phase 6 | **Done 2026-08-10** — six forks walked (`P6-D0`–`P6-D5`), fifteen principles ratified en bloc; indexed in `research/README.md` |
| §13 of `rest-api-standard.md` + `streaming-profile.md` | The v1.1.0 extension: compact `R13.x` rules plus the informative explanatory body | Phase 6 | **Done 2026-08-10** — `R13.1`–`R13.11` + `R12.10`; **nine** existing rules scoped for streaming (`R5.1`, `R5.12`, `R5.13`, `R11.2`, `R11.5`, `R2.11`, `R6.1`, `R6.2`, `R6.4`); §1.2, §1.5, §1.8, §1.9, §1.10, §1.11 amended; Appendix A, C, D, E, and G plus Part II rows landed atomically. Released via PR **#8** |

## Definition of done for version 1.0

- Every normative rule has a stable identifier and an explicit strength.
- Every external claim has traceable source lineage.
- Every project policy is labeled as policy rather than presented as protocol law.
- Conflicts and exceptions have recorded resolutions.
- The checklist and worked example cover every normative section.
- Structural verification passes with no duplicate IDs, broken internal links,
  undefined normative terms, or untracked decision changes.
