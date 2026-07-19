# REST Standards Project Plan

## Objective

Create a public, language-agnostic REST API design standard that is grounded in
current primary sources, records where judgment begins, and can be reviewed and
maintained with the same explicit decision discipline used for `cli-standards`.

The repository will distinguish three kinds of material:

1. externally supported facts and protocol requirements;
2. recommended defaults selected from legitimate alternatives; and
3. project policy that is intentionally stricter than the underlying standards.

## Why three research prompts

Three prompts are the smallest set that gives each major evidence domain a
coherent question without creating a fourth synthesis thread that merely repeats
the work:

1. HTTP semantics and resource modeling;
2. representations and API contracts; and
3. security and operational practice.

Cross-cutting conflicts will be reconciled after all three reports exist. Each
prompt has a neighboring `.framing.md` file that preserves its source lineage,
constraints, and expected decision value. Those framing files are not extra
research runs.

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

### Phase 0: Reviewable bootstrap — current phase

- Create this plan and the research lineage structure.
- Create three standalone research prompts and three framing records.
- Publish the repository publicly as `smorinlabs/rest-standards`.
- Stop for review before running research.

**Gate A:** Approve or revise this plan and the three prompt scopes.

### Phase 1: Execute the three research threads

- Run each approved prompt as an independent deep-research task.
- Save its report beside the prompt as `00-landscape.md`.
- Preserve source URLs, publication dates, conflicts, assumptions, and confidence.
- Reject claims that cannot be traced to an authoritative or clearly labeled
  comparative source.

**Gate B:** Review the reports for coverage and decide whether any narrow follow-up
thread adds genuinely new evidence. The default budget remains three threads.

### Phase 2: Convert research into explicit decisions

- Break broad reports into terminal question leaves where necessary.
- Add a `DECISION.md` for each research topic.
- Classify every candidate rule as protocol requirement, evidence-backed default,
  project policy, exception, or unresolved question.
- Record disagreements between sources rather than silently averaging them.
- Update `research/CLAUDE.md` so later work starts from existing evidence.

**Gate C:** Review and approve the policy decisions before drafting normative text.

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

**Gate D:** Approve the draft for systematic review.

### Phase 4: Decision-focused review

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

## Planned artifacts

| Artifact | Purpose | Created in |
| --- | --- | --- |
| `PLAN.md` | Scope, sequence, gates, and definition of done | Phase 0 |
| `research/CONSTRAINTS.md` | Shared research boundaries | Phase 0 |
| Three `.framing.md` files | Prompt provenance and shaping record | Phase 0 |
| Three `.prompt.md` files | Standalone runnable research tasks | Phase 0 |
| Three `00-landscape.md` reports | Raw research evidence | Phase 1 |
| Topic `DECISION.md` files | Accepted conclusions and consequences | Phase 2 |
| Normative standard | Stable rules and rationale | Phase 3 |
| Checklist and worked example | Conformance and integration proof | Phase 3–4 |

## Definition of done for version 1.0

- Every normative rule has a stable identifier and an explicit strength.
- Every external claim has traceable source lineage.
- Every project policy is labeled as policy rather than presented as protocol law.
- Conflicts and exceptions have recorded resolutions.
- The checklist and worked example cover every normative section.
- Structural verification passes with no duplicate IDs, broken internal links,
  undefined normative terms, or untracked decision changes.
