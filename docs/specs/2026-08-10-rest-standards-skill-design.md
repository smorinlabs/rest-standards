# Design — the `rest-standards` skill

*Written 2026-08-10 against `rest-api-standard.md` v1.0.0. Status: approved by
the owner 2026-08-10; implementation tracked as `PLAN.md` Phase 7. This
document specifies an apparatus artifact — an agent skill that applies the
standard. It adds no normative content to the standard and does not amend it.*

## 1. Purpose

Build an agent skill, `rest-standards`, that applies `rest-api-standard.md` to
an HTTP API being planned, built, reviewed, or audited — the REST counterpart
to the `cli-standards` skill that ships beside the CLI Design Standard in the
maintainer's `cli-standards` repository.

The skill is the unfinished half of an existing transfer. The gap review at
[`docs/reviews/2026-08-09-cli-standards-gap-review.md`](../reviews/2026-08-09-cli-standards-gap-review.md)
moved the CLI standard's **content** into this standard — its items B.8 and
B.12 are why §1.7 (tiers), §1.8 (applicability switches), and §1.9 (no silent
deviation, conformance-note template) exist. Nobody has moved the CLI
project's **operating apparatus**: the skill that turns a document into
applied practice.

## 2. What ports, and what does not

### 2.1 Ports unchanged

The four modes (`plan` / `check` / `review` / `audit`), the *no silent
deviation* preamble, "resolve the skill's base directory, then read the
standard live and version-pin it", rule-ID-citing findings tables ordered
blockers first, the MUST = blocker / SHOULD = waiver / MAY = suggestion
severity ladder, and the feed-back-upstream amendment step.

This standard supports all of it directly: stable `R#.#` identifiers (R1.2),
the Part II Decision Log, the Appendix A one-row-per-rule checklist, and the
Appendix B exception process.

### 2.2 Three things do not port

| # | `cli-standards` assumption | This standard's reality | Consequence |
| --- | --- | --- | --- |
| 1 | **Tier is ambition** (`minimal` / `standard` / `publishable`) and scales how deep an audit goes | **Tier is audience** (`internal` / `partner` / `public`), and §1.7 states every rule applies at every tier unless the rule itself carries a tier scope | The depth dial must be rebuilt. Depth comes from the **evidence plane** available, crossed with the §1.8 switches |
| 2 | The skill ships its own `references/tiers.md` and `templates/conformance-note.md` — the apparatus lives in the skill | §1.7–1.9 already are that apparatus, ratified en bloc at Gate D (2026-08-09) | The skill ships **procedure only**. A `tiers.md` would duplicate §1.7–1.8; a conformance-note template would drift from the one embedded in §1.9 |
| 3 | Live probes are cheap and sandboxable — point `XDG_*` at a temp directory and run the binary | Appendix G probes hit a **deployed API**. Three of them are destructive precisely when the API fails the check: `DELETE` without `If-Match` (R7.4), `dry_run` against a non-implementing endpoint (R1.9), and deliberate quota exhaustion (R11.2) | The audit gate must be sharper than the CLI's sandbox section — see §3.6 |

### 2.3 What this repository already has that the CLI project lacked

[`conformance/spectral.yaml`](../../conformance/spectral.yaml) — 92 lines,
execution-verified 2026-08-10 with `@stoplight/spectral-cli` 6.16.3 against
[`conformance/fixture-violations.yaml`](../../conformance/fixture-violations.yaml),
all twelve expected findings firing. The CLI audit mode hand-writes its
probes; this skill's audit mode can **run an existing checker**.

That also fixes `plan` mode's deliverable shape: an OpenAPI skeleton, which
`conformance/spectral.yaml` then lints directly — a loop the CLI skill cannot
close.

## 3. Design

### 3.1 Identity and placement

| | |
| --- | --- |
| Name | `rest-standards` |
| Source | `.claude/skills/rest-standards/SKILL.md` in this repository |
| Placement | `~/.agents/skills/rest-standards`, a symbolic link into this repository — parity with the `cli-standards` placement |
| `allowed-tools` | `Read, Grep, Glob, Bash, Write, Edit, AskUserQuestion` |
| Version pinning | Every deliverable records the standard version read from the `**Version …**` header line (v1.0.0 at time of writing) |

`Bash` is requested for exactly two jobs: running `npx @stoplight/spectral-cli`
on the contract plane, and issuing the gated `curl` probes of §3.6. No other
tool in the set can do either.

The trigger description must not collide with `cli-standards`,
`factor-architect`, `guided-research`, or `code-review`. It fires on an
HTTP/REST API being designed, extended, reviewed, or audited — "new API",
"design this endpoint", "review this OpenAPI spec", "is this API conformant",
"REST standards" — and explicitly disclaims CLIs (`cli-standards`), generic
architecture review (`factor-architect`), and external-practice research
(`guided-research`). Final wording is settled by `skill-create`'s collision
analysis (§5).

### 3.2 File layout — five files, no `templates/`

```
.claude/skills/rest-standards/
├── SKILL.md              ~100 lines  modes, locating/navigating, workflow, red flags
└── references/
    ├── scoping.md         ~60 lines  elicit tier + switches + evidence plane
    ├── planning.md        ~70 lines  greenfield → OpenAPI skeleton + conformance note
    ├── review.md          ~55 lines  section-order sweep of an unshipped design
    └── audit.md           ~90 lines  three planes, Spectral, gated live probes
```

Two deliberate absences, both following from §2.2 item 2:

- **No `templates/conformance-note.md`.** The note renders from the fenced
  template inside §1.9, read live at render time. One source of truth; a
  future amendment to §1.9 propagates without a skill edit.
- **No `references/tiers.md`.** `scoping.md` replaces it and **defines
  nothing** — it reads §1.7 and §1.8 live and elicits the declarations.

### 3.3 Navigating a 2,400-line standard

`rest-api-standard.md` is roughly 114 KB. The skill never reads it whole and
never answers from memory. It navigates by running `grep -n '^## '` (and
`'^### '` within a section) against the live file, then reads only the
sections in play.

The section index is deliberately **not** hardcoded in the skill: Phase 6
(streaming) is in flight and will add a section under the Part II amendment
rule. A grepped index cannot go stale; a shipped one silently would.

### 3.4 Depth scaling — three dials

| Dial | Source | Effect |
| --- | --- | --- |
| **Tier** — `internal` / `partner` / `public` | §1.7, read live | An audience declaration. Does **not** reduce the rule set; only rules carrying an explicit tier annotation vary |
| **Switches** — `webhooks`, `async-operations`, `bulk-operations` | §1.8, read live | Each switch that is off removes its rule group, reported as `N/A — <switch>: off, <reason>`, per R1.6's requirement that an off switch carry a stated reason |
| **Evidence plane** — contract / source / runtime | Skill-owned procedure (this document) | Decides which rules can be verified at all, and by what |

The evidence plane is the only dial the skill itself owns, and it is
procedural rather than normative: it describes what evidence an auditor has
in hand, not what any API must do.

### 3.5 Evidence planes

| Plane | Input | Checker | Rules it can prove |
| --- | --- | --- | --- |
| **contract** | An OpenAPI or JSON Schema document | `conformance/spectral.yaml` via `npx @stoplight/spectral-cli` | Path and property naming and casing (R2.4, R4.4), declared error media type, declared headers, reserved-name misuse (§1.10) |
| **source** | Route tables, handlers, middleware | `Read` / `Grep` / `ast-grep` | Rules with no contract expression: idempotency-key retention, existence masking (R5.10), redaction, documented default sort order |
| **runtime** | A deployed base URL | The Appendix G live-probe table, gated per §3.6 | R2.6 trailing-slash 308, R1.9 `dry_run` 400 guard, R7.4 `If-Match` 428, R11.2 429 + `Retry-After`, R11.7 `request-id` |

**Planes overlap, and that is a feature.** Some rules are reachable two ways —
existence masking (R5.10) is both a source-plane read of the authorization
path and an Appendix G runtime probe. The table above names each rule's
*cheapest* plane, not its only one. Where two planes both reach a rule, the
skill reports both, and agreement between them is a stronger result than
either alone; disagreement between them is itself a finding, because it means
the deployed behavior and the source have diverged.

Every finding names the plane that produced it. A rule that no available plane
can reach is reported `unverified — no <plane> available`, carrying the
CLI audit rule across verbatim: **never an inferred pass**.

### 3.6 The live-probe gate

Owner ruling, 2026-08-10: **opt-in, non-production only**.

1. **Default: the skill issues no HTTP requests.** Contract and source planes
   only.
2. **The user must ask for live probes.** The skill then requires a base URL,
   an explicit statement that the deployment is non-production (local,
   sandbox, or staging), and the identity of the resources that are
   disposable.
3. **Read-only probes run first**: trailing slash (R2.6), credential split
   (R5.9), existence masking (R5.10), empty collection (R6.2), `request-id`
   presence (R11.7), cache posture (R7.1–R7.3), and error negotiation
   (R5.12–R5.13) against a naturally failing request.
4. **Mutating or disruptive probes require a second confirmation** that names
   the disposable fixture resource: `DELETE` without `If-Match` (R7.4),
   the `dry_run` guard (R1.9), PATCH media-type rejection (R3.7, R5.11),
   the unknown-method probe (R5.11 — an "unimplemented" method that turns out
   to be implemented is a real mutation), and 202 discovery (R10.9), which
   starts real work. The quota probe (R11.2) additionally warns, before
   running, that it degrades service for every other client of that
   deployment.
5. **Never** against production, against an API the user does not own, or with
   credentials the user has not deliberately provided for this purpose.
6. Anything not run is reported unverified **with the exact `curl` invocation**
   for the user to run by hand. This is also the retreat path when a probe
   turns out to be riskier than it looked.

### 3.7 Modes

| Mode | Fires when | Loads | Deliverable |
| --- | --- | --- | --- |
| **plan** | Greenfield API, or a new resource group on an existing one | `scoping.md` + `planning.md` | An OpenAPI skeleton that is clean on the contract plane, plus a seeded conformance note rendered from §1.9 |
| **check** | Mid-build lookup — "what status code, header name, parameter name does the standard say" | The relevant section only | A cited answer with rule IDs; nothing written |
| **review** | A spec, OpenAPI document, or unshipped diff exists | `scoping.md` + `review.md` | A findings table, blockers first |
| **audit** | An existing API, or a pre-release conformance gate | `scoping.md` + `audit.md` | Per-plane findings, a conformance note, and optional CI wiring of `conformance/spectral.yaml` |

`plan`, `review`, and `audit` settle tier and switches before executing.
`check` skips that gate: it is a lookup, not a judgment.

`plan` mode's interview is capped at four questions, asked one at a time,
skipping any the conversation already answers: resource inventory and
shape; tier (§1.7); switch states (inference pre-marked, presented as one
menu rather than one question per switch); and base-URL and versioning shape.

### 3.8 Cross-cutting traps for `review` mode

The REST analogue of the CLI review mode's Decision Log traps — mistakes
common in otherwise-clean designs, each traceable to a ratified decision:

- A reserved name from §1.10 used with a different meaning, or a synonym used
  where a reserved name exists (R1.8).
- Any newly minted `X-` prefixed header (RFC 6648; §1.10 states the standard
  never reserves one).
- `?format=` used for content negotiation.
- PII in a URI (R2.10).
- Collection-level custom actions (R2.13).
- Opaque cursors (`AC-013` lineage) with no documented default sort order.
- `207` versus `200` on partial bulk outcomes.
- `HS-*` / `AC-*` / `OP-*` identifiers cited as if they were rule IDs. They
  are frozen research provenance (R1.3); findings cite `R#.#` only.

### 3.9 The amendment loop

The standard is released at 1.0.0, so a deviation that is arguably better than
the rule does not become a waiver. It becomes a proposed **Part II Decision
Log row, a rule edit, and a `CHANGELOG.md` entry** under the Part II amendment
rule — drafted and offered to the owner, never applied unilaterally.

## 4. Non-goals

- **No normative content in the skill.** If applying the standard reveals a
  missing rule, that is an amendment (§3.9), not a skill addition.
- **No new executable tooling.** A probe runner, CI workflow templates, and a
  conformance `justfile` target were considered and deferred: they are
  repository infrastructure belonging under `conformance/`, and they compete
  with Phase 6 for attention. The audit mode can call them if they are later
  built.
- **No re-litigation of any ratified decision.** The skill applies §1–§12 as
  written.

## 5. Ownership split — brainstorming designs, `skill-create` ships

Naming this split keeps two pipelines from each running in full:

| Owner | Responsibility |
| --- | --- |
| This design (brainstorming → writing-plans) | Mode semantics, file layout, evidence planes, the probe gate, findings formats |
| `skill-create` | Trigger-description collision analysis against the installed fleet, placement and the `~/.agents/skills/rest-standards` symbolic link, harness and documentation wiring, and the `skill-quality` gate |

## 6. Verification

1. **`skill-quality` gate** — frontmatter validity, trigger-description
   collisions, least-privilege `allowed-tools`, documentation completeness.
2. **Self-test** — run the finished skill's `audit` mode against Appendix E,
   the Bloom Orders API worked example, which already carries rule-ID
   annotations and its own conformance note, and confirm the skill reproduces
   them.
3. **Checker still fires** — confirm that, invoked through the skill's own
   path, `npx @stoplight/spectral-cli lint --ruleset conformance/spectral.yaml
   conformance/fixture-violations.yaml` still produces all twelve findings
   (7 errors, 5 warnings).

## 7. Execution sequence

1. Worktree `.claude/worktrees/docs+phase-7-skill` on branch
   `worktree-docs+phase-7-skill`, based on `origin/main` — created before any
   file was written. **Done.**
2. `PLAN.md` gains Phase 7 and its gate; the planned-artifacts table gains the
   skill. **Done in this commit.**
3. This design document, committed. **Done in this commit.**
4. Author the five skill files.
5. Hand placement, wiring, and the quality gate to `skill-create`.
6. Self-test per §6, then open the pull request.

## 8. Decisions recorded

| Decision | Ruling | Date |
| --- | --- | --- |
| May audit mode issue live HTTP requests? | Opt-in, non-production only — the §3.6 ladder | 2026-08-10, owner |
| Where is this work tracked? | `PLAN.md` Phase 7, matching every other unit of work in this repository | 2026-08-10, owner |
| Ship a conformance-note template with the skill? | No — render from §1.9 live | 2026-08-10, design |
| Ship a section index with the skill? | No — grep the live file (§3.3) | 2026-08-10, design |
| Build now, or wait for Phase 6 streaming? | Build now; the skill reads the standard live, so a new section needs no skill edit. Caveat: a streaming section could introduce an evidence plane that is neither document nor request/response pair, which would require a §3.5 edit | 2026-08-10, design |
| May the runtime plane probe an API the user does not own? | No — hard ban. Probing a third party's deployment raises an authorization question the skill cannot resolve. Reviewing a vendor's *published contract* on the contract plane remains available | 2026-08-10, reviewed and upheld |
| Conformance notes carry no audit-history section, unlike the CLI template | Accepted. §1.9 defines the note's shape; git records history. The skill will not emit structure the standard does not define (§4) | 2026-08-10, reviewed and upheld |
| Announcement wording — interim string, or the final one naming the skill? | The final one. `.claude/settings.json` is written up front and is forward-looking until this phase completes; both the skill and `docs/skills/rest-standards.md` become Gate F conditions | 2026-08-10, owner |

## 9. Deferred, and not part of Phase 7

This repository has no `.claude/settings.json` `companyAnnouncements` entry,
which the maintainer's global instructions ask for on owned repositories. It
is unrelated to the skill and is listed here only so it is not lost.
