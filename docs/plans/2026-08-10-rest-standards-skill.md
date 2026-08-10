# `rest-standards` Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Author the `rest-standards` agent skill — five files that apply
`rest-api-standard.md` to an HTTP API in four modes (`plan`, `check`,
`review`, `audit`) — and land it through `skill-create` and `skill-quality`.

**Architecture:** The skill ships **procedure only**. Every normative
statement it needs (conformance tiers, applicability switches, the
conformance-note template, rule text) is read live out of
`rest-api-standard.md` at run time, never copied into the skill. Audit depth
scales by **evidence plane** — contract document, source, deployed runtime —
because §1.7 conformance tiers name an audience rather than a depth. The
runtime plane is gated: opt-in, non-production only, mutating probes behind a
second confirmation.

**Tech Stack:** Markdown with YAML frontmatter (the Claude Code skill format);
`@stoplight/spectral-cli` 6.16.3 via `npx` for the contract plane; `curl` for
the gated runtime plane; `skillsmith` / `skill-create` / `skill-quality` for
placement, wiring, and gating.

**Design spec:** [`docs/specs/2026-08-10-rest-standards-skill-design.md`](../specs/2026-08-10-rest-standards-skill-design.md).
Section references of the form "spec §3.6" point there. Section references of
the form "§1.7" point at `rest-api-standard.md`.

## Global Constraints

Every task's requirements implicitly include this section.

1. **No normative content in the skill.** If authoring reveals a missing rule,
   that is a Part II amendment proposal, not a skill addition (spec §4).
2. **Findings cite `R#.#` only.** `HS-*`, `AC-*`, and `OP-*` are frozen
   research provenance identifiers (R1.3), never rule citations.
3. **Every cited rule ID must exist.** 127 rule IDs are defined; the
   extraction command is in every verification step below. Both sides of that
   command use plain `sort -u`, never `sort -V`: BSD `comm` on macOS assumes
   lexicographic collation, so version-sorted input (where `R9.7` precedes
   `R10.1`) makes it report `R10.9`, `R11.2`, and `R11.7` as undefined when
   all three exist. Verified on this machine during Task 5.
4. **Read live, never from memory.** The standard is ~2,400 lines. No section
   index, no rule text, and no conformance-note template may be hardcoded into
   the skill (spec §3.3, §3.2).
5. **Version-pin every deliverable** to the standard's header version, read at
   run time. Current value: `1.0.0`.
6. **No `{{VARS}}` in shipped output.** Anything the skill renders into a
   target repository contains concrete values only.
7. **House voice matches `cli-standards`:** imperative, terse, tables over
   prose for anything with three or more parallel items, a `Red Flags` table of
   rationalization-versus-reality rows.
8. **Line budgets** (spec §3.2): `SKILL.md` ~100, `scoping.md` ~60,
   `planning.md` ~70, `review.md` ~55, `audit.md` ~90. Treat as review
   signals, not hard limits.
9. **Commit after every task**, conventional-commit format, on branch
   `worktree-docs+phase-7-skill`.

## File Structure

| File | Responsibility |
| --- | --- |
| `.claude/skills/rest-standards/SKILL.md` | Frontmatter and trigger description; the no-silent-deviation preamble; how to locate, version-pin, and navigate the standard; the mode table; the three dials; the six-step workflow; red flags; see-also |
| `.claude/skills/rest-standards/references/scoping.md` | Elicit the §1.7 tier, the §1.8 switch states, and the evidence plane. Defines nothing — reads and asks |
| `.claude/skills/rest-standards/references/planning.md` | `plan` mode: the ≤4-question interview, the OpenAPI skeleton, the seeded conformance note |
| `.claude/skills/rest-standards/references/review.md` | `review` mode: section-order sweep, cross-cutting traps, findings table |
| `.claude/skills/rest-standards/references/audit.md` | `audit` mode: three evidence planes, the Spectral invocation, the six-step probe gate, the report format |
| `docs/skills/rest-standards.md` | Documentation page named by the startup announcement; rendered by `skill-create` in Task 7 |

`check` mode needs no reference file: it is a targeted section read plus a
citation, fully specified in `SKILL.md`.

---

### Task 1: `SKILL.md`

**Files:**
- Create: `.claude/skills/rest-standards/SKILL.md`

**Interfaces:**
- Consumes: nothing.
- Produces: the four reference filenames (`references/scoping.md`,
  `references/planning.md`, `references/review.md`, `references/audit.md`),
  the mode names (`plan`, `check`, `review`, `audit`), and the three dial
  names (tier, switch, evidence plane). Tasks 2–5 must use these exact names.

- [ ] **Step 1: Write the frontmatter**

The `description` field is what makes the skill fire. It must claim REST/HTTP
API work and disclaim the four neighbors named in spec §3.1.

```markdown
---
name: rest-standards
description: Apply the org's REST API Design Standard (rest-api-standard.md in this repo) to any HTTP API work, scaled by conformance tier (internal / partner / public), applicability switches, and available evidence. Four modes — plan (greenfield interview → OpenAPI skeleton + seeded conformance note), check (mid-build lookups — "what status code / header / query parameter does the standard say"), review (design review of an API spec, OpenAPI document, or unshipped diff; findings cite rule IDs), audit (conformance sweep of an existing API across three evidence planes — contract document via Spectral, source, and gated live probes against a non-production deployment). Fires whenever an HTTP API is being created, designed, extended, reviewed, or audited — "new API", "design this endpoint", "review this OpenAPI spec", "is this API conformant", "audit this API", "REST standards". Not for CLIs (cli-standards), generic non-API architecture review (factor-architect), generic quality sweeps (factor-scan), or external REST-practice research (guided-research).
allowed-tools: Read, Grep, Glob, Bash, Write, Edit, AskUserQuestion
---
```

- [ ] **Step 2: Write the no-silent-deviation preamble**

This is the skill's spine. It restates R1.7's obligation as an operating rule
without redefining it.

```markdown
# rest-standards

Apply the REST API Design Standard — this repo's `rest-api-standard.md` — to an
HTTP API being planned, built, reviewed, or audited, at the depth the available
evidence supports.

> **NO SILENT DEVIATION (R1.7).** Every applicable MUST the API skips is a
> blocker finding. Every SHOULD it waives gets a conformance-note entry with
> rule ID, rule strength, what differs, why, approver, and date. Judgment calls
> are *recorded*, never just made.
>
> No exceptions: not "it's an internal API" (that's a tier, and §1.7 applies
> every rule at every tier), not "we'll document it later", not "that header
> name is close enough" (R1.8 makes reserved names the contract).
>
> Violating the letter of this rule is violating the spirit of it.
```

- [ ] **Step 3: Write the locating section**

The skill's base directory may be a symbolic link, so resolve it before
walking up. Three levels up from the skill directory is the repo root
(`rest-standards/` → `skills/` → `.claude/` → repo root); this was verified
empirically while writing the plan, and Step 8 re-verifies it in place:

```markdown
## Locating the standard

This skill ships beside the canonical standard and reads it live — never from
memory, never from a bundled copy. Resolve the skill's base directory first
(it is a symlink under a global placement), then walk up to the repo root:

    STD="$(dirname "$(dirname "$(dirname "$(realpath "<skill-base-dir>")")")")/rest-api-standard.md"

Read the header and note the **Version** line:

    grep -m1 -oE '\*\*Version [0-9]+\.[0-9]+\.[0-9]+' "$STD"

Every deliverable — spec, review, audit, conformance note — pins the standard
version it was produced against. If the file is missing, stop and report; do
not proceed from memory.
```

- [ ] **Step 4: Write the navigation section**

```markdown
## Navigating the standard

~2,400 lines. Never read it whole; never answer from memory. Get the live
section map, then read only the sections in play:

    grep -n '^## ' "$STD"          # 24 top-level sections: Part I §1–§12, Part II, Appendices A–G
    grep -n '^### ' "$STD"         # subsections, when a section is large

The index is deliberately not written down here. The standard evolves by
Part II amendment — a hardcoded map would go stale silently, a grepped one
cannot.
```

- [ ] **Step 5: Write the modes table**

```markdown
## Modes

Pick by what the user is doing; ask only when genuinely ambiguous.

| Mode | Fires when | Load | Deliverable |
|---|---|---|---|
| **plan** | Greenfield API or new resource group: "new API", "design this endpoint" | `references/planning.md` | OpenAPI skeleton + seeded conformance note |
| **check** | Mid-build lookup: "what status code / header / parameter name…" | The relevant § of the standard | Cited answer with rule IDs; nothing written |
| **review** | A spec, OpenAPI document, or diff exists but hasn't shipped | `references/review.md` | Findings table with rule IDs |
| **audit** | An existing API: "is it conformant", pre-release gate | `references/audit.md` | Per-plane findings + conformance note + optional CI wiring |

Before plan, review, or audit: settle **tier, switches, and evidence plane**
via `references/scoping.md`. `check` skips scoping; it is a lookup, not a
judgment.
```

- [ ] **Step 6: Write the three-dials section**

```markdown
## Depth scaling

Three dials. The first two are the standard's and are read live; the third is
this skill's, and is procedural.

- **Tier** (§1.7) — `internal` / `partner` / `public`. Names the API's
  *audience*, not its quality bar. Every rule applies at every tier unless the
  rule itself carries a tier scope. Tier never waives a MUST.
- **Switches** (§1.8) — read the live switch vocabulary from §1.8; do not
  assume it. A switch that is off removes its rule group and MUST carry a
  stated reason (R1.6): `N/A — <switch>: off, <reason>`. "N/A" with no reason
  is a deviation, not an exemption.
- **Evidence plane** — contract / source / runtime. Decides which rules can be
  verified *at all*. Planes overlap; agreement between two is a stronger
  result than either alone, and disagreement is itself a finding.

A rule no available plane can reach is reported `unverified` with the reason —
never an inferred pass.
```

- [ ] **Step 7: Write the workflow, red flags, and see-also**

```markdown
## Workflow (every mode)

1. Locate and version-pin the standard (above).
2. Identify the mode; for plan/review/audit, settle tier + switches + plane.
3. Execute the mode's reference procedure.
4. Cite rule IDs (`R#.#`) on every finding and answer, with MUST/SHOULD/MAY
   severity — MUST violation = blocker, SHOULD deviation = fix or waiver,
   MAY = suggestion. Never cite `HS-*`/`AC-*`/`OP-*`: they are frozen research
   provenance (R1.3), not rules.
5. Deviations that stay: record in the target repo's conformance note,
   rendered from the template in §1.9 read live. No `{{VARS}}` may survive.
6. **Feed back upstream.** A deviation that seems *right* and generalizable is
   a proposed amendment: draft a Part II Decision Log row, the rule edit, and a
   `CHANGELOG.md` entry against `rest-api-standard.md` in this repo, and offer
   it to the user. The standard is released at 1.0.0; it evolves by amendment
   under the Part II rule, not by drift.

## Red Flags

| Thought | Reality |
|---|---|
| "Internal API — the standard is overkill" | §1.7 tiers name an audience, not a depth. Every rule applies at every tier. |
| "I remember what the standard says" | 127 rules across 12 sections, amended under Part II. Read the section. |
| "`X-Correlation-Id` is close enough" | §1.10 reserves `request-id`; R1.8 makes the reserved name the contract, and RFC 6648 rules out new `X-` names. |
| "`AC-016` requires idempotency keys" | `AC-*` is research provenance (R1.3). Cite the `R#.#` rule. |
| "I can audit from the OpenAPI document alone" | The contract plane cannot reach runtime behavior. Probe (gated) or mark those rules unverified. |
| "This deviation is fine, moving on" | Fine → conformance-note entry. Generalizable → Part II amendment. Never just "moving on". |

## See also

- `cli-standards` — the sibling standard and skill, for command-line tools.
- `factor-architect` — generic (non-API) architecture review.
- `factor-scan` — generic code-quality sweeps; conformance is not quality.
- `guided-research` — researching external REST practice; this skill applies
  the settled internal standard.
```

- [ ] **Step 8: Verify the locating command resolves**

Run, from the repository root, substituting the real base directory:

```bash
BASE=".claude/skills/rest-standards"
STD="$(dirname "$(dirname "$(dirname "$(realpath "$BASE")")")")/rest-api-standard.md"
echo "$STD"; test -f "$STD" && echo "RESOLVES" || echo "BROKEN — adjust the dirname depth in Step 3"
```

Expected: prints an absolute path ending `/rest-api-standard.md`, then
`RESOLVES`. If it prints `BROKEN`, correct the number of `dirname` wrappers in
Step 3 and re-run until it resolves. Then repeat the check against the global
placement path `~/.agents/skills/rest-standards` after Task 7 creates it.

- [ ] **Step 9: Verify every cited rule ID exists**

```bash
grep -oE '^\| R[0-9]+\.[0-9]+ \|' rest-api-standard.md | tr -d '| ' | sort -u > /tmp/defined.txt
grep -rhoE 'R[0-9]+\.[0-9]+' .claude/skills/rest-standards/ | sort -u > /tmp/cited.txt
comm -13 /tmp/defined.txt /tmp/cited.txt
```

Expected: `/tmp/defined.txt` contains 127 IDs, and `comm` prints **nothing**.
Any line printed is a citation of a rule that does not exist — fix it.

- [ ] **Step 10: Commit**

```bash
git add .claude/skills/rest-standards/SKILL.md
git commit -m "feat(skill): rest-standards SKILL.md — modes, dials, workflow"
```

---

### Task 2: `references/scoping.md`

**Files:**
- Create: `.claude/skills/rest-standards/references/scoping.md`

**Interfaces:**
- Consumes: the dial names from Task 1 (tier, switches, evidence plane).
- Produces: a settled scope record that Tasks 3–5 consume, in this exact
  shape, which every mode's deliverable header reproduces:

```
Standard: rest-api-standard v<version>
Tier: <internal|partner|public>
Switches: <name>=<on|off> …  (each off switch carries a one-line reason)
Evidence planes available: <contract|source|runtime> …
```

- [ ] **Step 1: Write the elicitation procedure**

This file **defines nothing**. It reads §1.7 and §1.8 live and asks. Content:

```markdown
# Scoping — tier, switches, and evidence plane

Three dials, settled before plan, review, or audit. Two belong to the
standard and are read live; one belongs to this skill.

## 1. Tier (§1.7)

Read §1.7 for the current tier vocabulary — do not assume it. One
AskUserQuestion, recommendation first, derived from context: a public
developer portal or self-service signup ⇒ `public`; a partner integration
guide or contracted consumers ⇒ `partner`; consumed only by the providing
org's own services ⇒ `internal`.

Tier is an audience declaration. It does not reduce the rule set, and it never
waives a MUST. Honor notes ("internal, but it opens to partners next quarter"
⇒ scope at `internal`, flag the `partner` delta).

## 2. Switches (§1.8)

Read the live switch vocabulary from §1.8. Infer each switch's state from the
API's shape and confirm the inference inside the tier question's option
descriptions rather than asking one question per switch.

Every switch declared off MUST carry a one-line reason (R1.6). Record as
`N/A — <switch>: off, <reason>`. A switch wrongly marked off is itself a
finding.

Capability facts with no rule attached — tenancy model, PII handling, client
audience — go in the conformance note's free-text `Context` field, not here.

## 3. Evidence plane

Ask what exists. This is a factual question, not a judgment:

| Plane | Present when | Unlocks |
|---|---|---|
| contract | An OpenAPI or JSON Schema document exists | `conformance/spectral.yaml` |
| source | The implementation is readable | Rules with no contract expression |
| runtime | A deployment exists **and** the user opts in (`references/audit.md` § gate) | Appendix G probes |

Planes overlap; more than one may be present. Runtime is never assumed
available — it requires the opt-in ladder, without exception.

## 4. Record it

Emit the scope record at the top of every deliverable, then proceed:

    Standard: rest-api-standard v<version>
    Tier: <internal|partner|public>
    Switches: <name>=<on|off> …  (each off switch carries a one-line reason)
    Evidence planes available: <contract|source|runtime> …
```

- [ ] **Step 2: Verify the section references resolve**

```bash
grep -n '^### 1\.7\|^### 1\.8\|^### 1\.9' rest-api-standard.md
```

Expected: three lines, for `1.7 Conformance tiers`, `1.8 Applicability
switches`, and `1.9 Deviations and the conformance note`. If any heading has
moved, update the citation in `scoping.md` to match the live heading.

- [ ] **Step 3: Verify the file defines nothing normative**

```bash
grep -nE 'internal \| partner \| public|webhooks|async-operations|bulk-operations' \
  .claude/skills/rest-standards/references/scoping.md
```

Expected: **no output**. Any hit means the file has copied the tier or switch
vocabulary instead of reading it live — delete the copy and cite the section.
The illustrative `<internal|partner|public>` placeholders in the scope record
use angle brackets and do not match this pattern.

- [ ] **Step 4: Verify rule-ID citations**

```bash
grep -oE '^\| R[0-9]+\.[0-9]+ \|' rest-api-standard.md | tr -d '| ' | sort -u > /tmp/defined.txt
grep -rhoE 'R[0-9]+\.[0-9]+' .claude/skills/rest-standards/ | sort -u > /tmp/cited.txt
comm -13 /tmp/defined.txt /tmp/cited.txt
```

Expected: no output.

- [ ] **Step 5: Commit**

```bash
git add .claude/skills/rest-standards/references/scoping.md
git commit -m "feat(skill): scoping reference — tier, switches, evidence plane"
```

---

### Task 3: `references/planning.md`

**Files:**
- Create: `.claude/skills/rest-standards/references/planning.md`

**Interfaces:**
- Consumes: the scope record from Task 2.
- Produces: two deliverable paths that Task 6's self-test asserts —
  `openapi.yaml` (or the target repo's existing contract path) and
  `CONFORMANCE.md` in the target repo root.

- [ ] **Step 1: Write the interview**

```markdown
# plan mode — greenfield API (or a new resource group)

Turn the standard into a contract before code exists. Process skills
(brainstorming, writing-plans) own the overall flow when active; this mode
supplies the API-domain content inside them.

## Interview — one question at a time, ≤4 total

Skip any question the conversation already answers; fold inferences into the
next question's options rather than asking extra rounds.

1. **Resources.** What does the API expose, and what are the resource types?
   Decides collection naming (plural, R2.2), nesting (containment only, ≤3
   resources per path, R2.5), and whether any operation needs the action form
   `POST /{collection}/{id}/{action}` (R2.11) — in which case the verb must
   come from the §1.10 registry or be justified as a new one.
2. **Tier** (§1.7, via `references/scoping.md`).
3. **Switches** (§1.8) — present as one menu with the inference pre-marked.
4. **Base URL and versioning shape** — host, base path, and the major-version
   segment; read §9 for the versioning rules before recommending.
```

- [ ] **Step 2: Write the OpenAPI skeleton deliverable**

```markdown
## Deliverable 1 — contract skeleton

Write `openapi.yaml` in the target repo (or extend its existing contract
document). Every section cites the rules it satisfies; do not restate a rule
the document already satisfies — cite it.

- **Identity**: title, major version in the base path, tier, standard version
  pinned.
- **Paths**: kebab-case segments (R2.4), plural collections (R2.2), no
  trailing slash (R2.6), no PII in any URI (R2.10), modifiers in query
  parameters rather than path segments (R2.9).
- **Operations**: method semantics and the status codes each returns; read §3
  and §5 before fixing the set.
- **Reserved names**: any of `sort`, `fields`, `cursor`, `limit`, `dry_run`,
  or the bracketed range filters that the API offers MUST use the §1.10
  registered meaning; read §1.10 for the live registry.
- **Errors**: `application/problem+json` responses with template-bound
  `type`/`code` (read §5).
- **Headers**: the response headers the API commits to; §1.10 registry only,
  and never a new `X-` name.

## Deliverable 2 — seeded conformance note

Render the template from §1.9 — read it live, do not reproduce it here — into
the target repo as `CONFORMANCE.md`: standard version, tier, every switch with
its state and a reason for each off switch, free-text context, an empty
deviations list, and the N/A declarations. Deviation tracking starts at day
zero, not at first audit.
```

- [ ] **Step 3: Write the self-check step**

```markdown
## Deliverable 3 — prove the skeleton on the contract plane

Lint the skeleton with the standard's own ruleset before handing it over:

    npx @stoplight/spectral-cli lint \
      --ruleset <standard-repo>/conformance/spectral.yaml openapi.yaml

Zero errors is the bar. Warnings are conservative heuristics — review each and
either fix it or record it, never silently ignore it.
```

- [ ] **Step 4: Prove the loop closes — write a throwaway skeleton and lint it**

Create a scratch file, lint it, confirm the ruleset engages, then delete it.

```bash
mkdir -p /tmp/rs-plan-check && cat > /tmp/rs-plan-check/openapi.yaml <<'YAML'
openapi: 3.1.0
info: { title: Widgets API, version: "1.0.0" }
paths:
  /v1/widgets:
    get:
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  created_at: { type: string, format: date-time }
YAML
npx @stoplight/spectral-cli lint \
  --ruleset conformance/spectral.yaml /tmp/rs-plan-check/openapi.yaml
```

Expected: the command runs and reports **no errors** for this conformant
skeleton (kebab-case path, snake_case property). Then prove the ruleset is
actually engaged rather than silently passing everything:

```bash
sed -i '' 's|/v1/widgets|/v1/myWidgets|' /tmp/rs-plan-check/openapi.yaml
npx @stoplight/spectral-cli lint \
  --ruleset conformance/spectral.yaml /tmp/rs-plan-check/openapi.yaml
```

Expected: an error citing `rs-r2-4-path-segments-kebab-case`. If the second
run also passes, the ruleset is not engaging — stop and diagnose before
continuing. Clean up: `rm -rf /tmp/rs-plan-check`.

- [ ] **Step 5: Verify rule-ID citations**

```bash
grep -oE '^\| R[0-9]+\.[0-9]+ \|' rest-api-standard.md | tr -d '| ' | sort -u > /tmp/defined.txt
grep -rhoE 'R[0-9]+\.[0-9]+' .claude/skills/rest-standards/ | sort -u > /tmp/cited.txt
comm -13 /tmp/defined.txt /tmp/cited.txt
```

Expected: no output.

- [ ] **Step 6: Commit**

```bash
git add .claude/skills/rest-standards/references/planning.md
git commit -m "feat(skill): planning reference — interview, contract skeleton, conformance note"
```

---

### Task 4: `references/review.md`

**Files:**
- Create: `.claude/skills/rest-standards/references/review.md`

**Interfaces:**
- Consumes: the scope record from Task 2.
- Produces: the findings-table column set — `Rule | Level | Where | Finding |
  Fix` — which Task 5's audit report reuses unchanged.

- [ ] **Step 1: Write the procedure**

```markdown
# review mode — design review of a spec, OpenAPI document, or unshipped diff

Review the *interface* before it hardens into shipped behavior. Input is
whatever exists: an OpenAPI document, a design doc, a Superpowers plan, a PR
diff, or README examples. This is not a code-quality review (that is
factor-scan) and not generic architecture review (factor-architect) — every
finding here traces to a rule ID.

## Procedure

1. Tier, switches, and evidence plane settled first (`references/scoping.md`).
2. Sweep the input against the standard **in section order**, skipping
   switched-off groups: §1 conformance apparatus and reserved names → §2 URIs
   → §3 methods → §4 representations → §5 status codes and errors → §6
   collections → §7 caching and concurrency → §8 security → §9 lifecycle →
   §10 async, bulk, webhooks → §11 rate limits and observability → §12 client
   obligations. Read each section from the live standard as you sweep — never
   from memory.
3. Check the cross-cutting traps below.
4. Where the input is silent on an applicable area, that is a **gap finding**,
   not a pass.
```

- [ ] **Step 2: Write the cross-cutting traps**

These are the mistakes common in otherwise-clean designs. Each traces to a
ratified decision; verify each rule ID in Step 4 before committing.

```markdown
## Cross-cutting traps

Common in otherwise-clean designs — check each explicitly:

- A §1.10 reserved name used with a different meaning, or a synonym used where
  a reserved name exists (R1.8). Read the live registry; it grows by amendment.
- Any newly minted `X-` prefixed header. §1.10 states the standard never
  reserves one (RFC 6648).
- `?format=` used for content negotiation instead of `Accept`.
- PII in a URI (R2.10).
- A custom action on a collection rather than an instance (R2.13).
- Opaque cursors with no documented default sort order — cursor pagination
  over an unordered set silently skips and duplicates rows.
- Partial bulk outcomes: confirm the status code against §10 rather than
  assuming `207`.
- `HS-*`/`AC-*`/`OP-*` cited as if they were rules (R1.3).
- `dry_run` accepted on an endpoint that does not implement it (R1.9) — the
  guard is standard-wide, not per-endpoint-optional.
```

- [ ] **Step 3: Write the findings format**

```markdown
## Findings format

One table, blockers first:

| Rule | Level | Where | Finding | Fix |
|---|---|---|---|---|
| R1.8 | MUST | `X-Request-Id` response header | Reserved concept emitted under a non-registered name | Emit `request-id`; §1.10 registers it and RFC 6648 rules out new `X-` names |

- **MUST violation** → blocker; the design does not conform until fixed.
- **SHOULD deviation** → fix, or a conformance-note entry with rule strength,
  what differs, why, approver, and date. Offer to write the entry.
- **MAY / style** → suggestion; no tracking obligation.

Close with: the N/A list (switches off, with reasons), the evidence plane the
review covered, and the standard version reviewed against.

## After the table

1. Offer to apply agreed fixes to the spec or contract document directly.
2. Waivers accepted → update the conformance note (template from §1.9, live).
3. A deviation that is arguably *better than the rule* → propose a Part II
   amendment (SKILL.md workflow step 6) instead of a waiver.
```

- [ ] **Step 4: Verify rule-ID citations**

```bash
grep -oE '^\| R[0-9]+\.[0-9]+ \|' rest-api-standard.md | tr -d '| ' | sort -u > /tmp/defined.txt
grep -rhoE 'R[0-9]+\.[0-9]+' .claude/skills/rest-standards/ | sort -u > /tmp/cited.txt
comm -13 /tmp/defined.txt /tmp/cited.txt
```

Expected: no output. In particular this catches a wrong ID for the
collection-action ban or the PII rule — verify each against Appendix A rather
than trusting the draft.

- [ ] **Step 5: Commit**

```bash
git add .claude/skills/rest-standards/references/review.md
git commit -m "feat(skill): review reference — section sweep, traps, findings table"
```

---

### Task 5: `references/audit.md`

**Files:**
- Create: `.claude/skills/rest-standards/references/audit.md`

**Interfaces:**
- Consumes: the scope record from Task 2 and the findings-table columns from
  Task 4.
- Produces: the conformance summary line format that Task 6 asserts:
  `<N> applicable MUSTs: <P> pass, <F> fail, <U> unverified`.

- [ ] **Step 1: Write the three-plane opening**

```markdown
# audit mode — conformance sweep of an existing API

Gap-analyze a real API against the standard: brownfield APIs never built to
it, and pre-release conformance gates. Depth scales with the evidence
available, not with tier — §1.7 tiers name an audience, and every rule applies
at every tier.

Findings use review mode's table: `Rule | Level | Where | Finding | Fix`.

## The three evidence planes

| Plane | Input | Checker |
|---|---|---|
| **contract** | OpenAPI or JSON Schema document | `conformance/spectral.yaml` |
| **source** | Routes, handlers, middleware | Read / Grep / ast-grep |
| **runtime** | A deployed base URL | Appendix G probes — gated, see below |

Every finding names the plane that produced it. Planes overlap: where two
reach the same rule, report both. Agreement is a stronger result than either
alone; **disagreement is itself a finding** — the deployment and the source
have diverged.
```

- [ ] **Step 2: Write the contract and source plane procedures**

```markdown
## 1. Contract plane

    npx @stoplight/spectral-cli lint \
      --ruleset <standard-repo>/conformance/spectral.yaml <contract-document>

The ruleset's rules are conservative heuristics; each description states its
known false-positive and false-negative limits. Warn-severity findings exist
to be reviewed, not blindly enforced. Read Appendix G for what the ruleset
does and does not traverse before treating a clean run as conformance.

## 2. Source plane

Read the routes, handlers, and middleware for what the contract cannot
express: authorization checks and existence masking, idempotency-key storage
and retention, redaction in error paths, the documented default sort order
behind cursor pagination, and rate-limit enforcement points. Grep for reserved
names (§1.10) used with a non-registered meaning.
```

- [ ] **Step 3: Write the runtime gate — the six-step ladder**

This is the section with the most consequence. It implements the owner ruling
of 2026-08-10 (spec §3.6) and must not be softened.

```markdown
## 3. Runtime plane — gated

Appendix G probes hit a real deployment. Three of them are destructive
*precisely when the API fails the check*: an unguarded DELETE succeeds, an
unimplemented method turns out to be implemented, a `dry_run` parameter
executes for real. The gate is not optional.

1. **Default: issue no HTTP requests.** Contract and source planes only.
2. **The user must ask.** Then require: a base URL, an explicit statement that
   the deployment is non-production (local, sandbox, or staging), and which
   resources are disposable.
3. **Read-only probes first** — trailing slash (R2.6), credential split
   (R5.9), existence masking (R5.10), empty collection (R6.2), correlation ID
   (R11.7), cache posture (§7), error negotiation (§5) against a naturally
   failing request.
4. **Mutating or disruptive probes need a second confirmation** naming the
   disposable fixture: DELETE without `If-Match` (R7.4), the `dry_run` guard
   (R1.9), PATCH media-type rejection (R3.7), the unknown-method probe
   (R5.11 — an "unimplemented" method that turns out to be implemented is a
   real mutation), and 202 discovery (R10.9 — it starts real work). The quota
   probe (R11.2) additionally warns, before running, that it degrades service
   for every other client of that deployment.
5. **Never** against production, against an API the user does not own, or with
   credentials the user has not deliberately provided for this purpose.
   Reviewing a third party's *published contract* on the contract plane
   remains available and needs no gate.
6. **Anything not run is reported unverified with the exact `curl`** for the
   user to run by hand. This is also the retreat path when a probe turns out
   to be riskier than it looked.

Read Appendix G for the live probe table; do not work from the list above,
which names the gate tiers rather than the probes.
```

- [ ] **Step 4: Write the report and artifacts sections**

```markdown
## 4. Report

Findings table (blockers first), then:

- N/A switches with their reasons (R1.6).
- Unverified rules, each with the plane that was missing and, for runtime
  rules, the `curl` that would settle it.
- The evidence planes actually used.
- Standard version.
- A conformance summary line: `<N> applicable MUSTs: <P> pass, <F> fail,
  <U> unverified`.

## 5. Artifacts (offer, don't assume)

1. **Conformance note** — render the §1.9 template (read live) or update the
   existing one: tier, switches with reasons, deviations, N/A declarations.
2. **CI wiring** — add the Spectral ruleset to the target repo's CI so the
   contract plane is checked on every change, not once.
3. **Fix plan** — hand ordered blockers to the normal planning flow
   (writing-plans) if the user wants them fixed now.
4. **Amendments** — deviations that beat the rule go upstream as Part II
   Decision Log proposals (SKILL.md workflow step 6).
```

- [ ] **Step 5: Verify every Appendix G probe is classified exactly once**

List the live probe table, then confirm each row appears in either the
read-only set (ladder step 3) or the gated set (ladder step 4), and in only
one of them.

Appendix G is the last appendix, so scan to end of file; exclude the header and
separator rows, and do not filter on a leading capital — one probe name starts
with a digit.

```bash
sed -n '/^## Appendix G/,$p' rest-api-standard.md \
  | grep -E '^\| ' | grep -vE '^\| (Probe|---)' | cut -d'|' -f2
```

Expected: exactly these 13 probes —

    Trailing slash · Rehearsal guard · PATCH media type · Destructive guard
    Unknown method · Empty collection · Auth split · Existence masking
    Quota · Error negotiation · Correlation · 202 discovery · Cache posture

Read-only set (ladder step 3): Trailing slash, Auth split, Existence masking,
Empty collection, Correlation, Cache posture, Error negotiation — 7 probes.
Gated set (ladder step 4): Destructive guard, Rehearsal guard, PATCH media
type, Unknown method, 202 discovery, Quota — 6 probes. 7 + 6 = 13. Any probe
in neither list, or in both, is a defect — fix `audit.md` before committing.

- [ ] **Step 6: Verify rule-ID citations**

```bash
grep -oE '^\| R[0-9]+\.[0-9]+ \|' rest-api-standard.md | tr -d '| ' | sort -u > /tmp/defined.txt
grep -rhoE 'R[0-9]+\.[0-9]+' .claude/skills/rest-standards/ | sort -u > /tmp/cited.txt
comm -13 /tmp/defined.txt /tmp/cited.txt
```

Expected: no output.

- [ ] **Step 7: Verify the gate has not been softened**

```bash
grep -c 'non-production\|second confirmation\|unverified' \
  .claude/skills/rest-standards/references/audit.md
```

Expected: at least 4. The ladder's three load-bearing constraints —
non-production requirement, second confirmation, unverified-not-inferred-pass
— must each be present in the text.

- [ ] **Step 8: Commit**

```bash
git add .claude/skills/rest-standards/references/audit.md
git commit -m "feat(skill): audit reference — three planes, Spectral, gated probe ladder"
```

---

### Task 6: Self-test against the Appendix E worked example

**Files:**
- Create: `docs/reviews/2026-08-10-phase-7-skill-self-test.md`

**Interfaces:**
- Consumes: all five skill files.
- Produces: the Gate F evidence record that Task 8 cites.

This is the plan's real test. Appendix E is a worked example — the Bloom
Orders API — already annotated with rule IDs and carrying its own conformance
note at `rest-api-standard.md` line 2007. Running the skill against it has a
known-correct answer, so a disagreement is either a skill defect or a genuine
standard finding.

- [ ] **Step 1: Extract the expected answer**

```bash
sed -n '/^## Appendix E/,/^## Appendix F/p' rest-api-standard.md > /tmp/appendix-e.md
wc -l /tmp/appendix-e.md
grep -oE 'R[0-9]+\.[0-9]+' /tmp/appendix-e.md | sort -u > /tmp/expected-rules.txt
wc -l < /tmp/expected-rules.txt
```

Expected: a non-empty extract, and a list of the rule IDs the appendix
annotates. This list is the answer key.

- [ ] **Step 2: Run the skill's audit mode against it**

Invoke the skill on `/tmp/appendix-e.md` as a contract-plane-and-source-plane
input, with no runtime plane available. Record: the scope record it produced,
its findings table, its N/A list, and its conformance summary line.

- [ ] **Step 3: Compare against the answer key**

Write `docs/reviews/2026-08-10-phase-7-skill-self-test.md` containing:

- The scope record the skill produced.
- A table with one row per rule ID in `/tmp/expected-rules.txt`: the rule, what
  Appendix E says about it, what the skill reported, and agree/disagree.
- Every disagreement, classified as **skill defect** (fix the skill and re-run)
  or **standard finding** (a Part II amendment proposal, offered to the owner —
  do not amend unilaterally).
- The conformance summary line, and confirmation it matches the
  `<N> applicable MUSTs: <P> pass, <F> fail, <U> unverified` format.

- [ ] **Step 4: Fix any skill defects and re-run**

For each disagreement classified as a skill defect, fix the reference file and
repeat Steps 2–3. The task is not complete while any unresolved skill defect
remains. Do not reclassify a defect as a standard finding to close the task.

- [ ] **Step 5: Commit**

```bash
git add docs/reviews/2026-08-10-phase-7-skill-self-test.md .claude/skills/rest-standards/
git commit -m "test(skill): self-test against Appendix E worked example"
```

---

### Task 7: Placement, wiring, and the quality gate

**Files:**
- Create: `docs/skills/rest-standards.md`
- Create (outside the repo): `~/.agents/skills/rest-standards` symbolic link

**Interfaces:**
- Consumes: the five skill files.
- Produces: the documentation page the startup announcement in
  `.claude/settings.json` already names.

Spec §5 assigns this task to `skill-create`: it owns trigger-description
collision analysis against the installed fleet, placement, harness and
documentation wiring, and the `skill-quality` gate. Do not hand-roll these
steps.

- [ ] **Step 1: Run `skill-create` in update mode against the authored skill**

Point it at `.claude/skills/rest-standards/`. It performs the collision
analysis, places the skill, and renders the documentation page.

- [ ] **Step 2: Verify the placement matches the `cli-standards` pattern**

```bash
ls -l ~/.agents/skills/rest-standards
ls -l ~/.agents/skills/cli-standards
```

Expected: both are symbolic links into their respective repositories. If the
new one is a copy rather than a link, the skill would go stale against the
repo — fix before continuing.

- [ ] **Step 3: Re-verify the locating command from the global placement**

```bash
STD="$(dirname "$(dirname "$(dirname "$(realpath ~/.agents/skills/rest-standards)")")")/rest-api-standard.md"
echo "$STD"; test -f "$STD" && echo "RESOLVES" || echo "BROKEN"
```

Expected: `RESOLVES`. This is the path that matters — the skill runs from the
global placement, not from the repo path used in Task 1.

- [ ] **Step 4: Verify the documentation page the announcement names exists**

```bash
test -f docs/skills/rest-standards.md && echo "PRESENT" || echo "MISSING — Gate F blocker"
grep -o 'docs/skills/rest-standards.md' .claude/settings.json
```

Expected: `PRESENT`, and the announcement's path matches the real file.

- [ ] **Step 5: Run the `skill-quality` gate against the worktree**

Point it at the worktree path, not the global placement — the placement still
resolves to pre-merge bytes until the branch lands.

Expected: pass. Address every finding before continuing.

- [ ] **Step 6: Commit**

```bash
git add docs/skills/rest-standards.md .claude/skills/rest-standards/
git commit -m "docs(skill): rest-standards documentation page and wiring"
```

---

### Task 8: Gate F and the pull request

**Files:**
- Modify: `PLAN.md` — flip Phase 7 status and the planned-artifacts row

**Interfaces:**
- Consumes: Tasks 1–7.
- Produces: the merged phase.

- [ ] **Step 1: Verify every Gate F condition**

```bash
test -f .claude/skills/rest-standards/SKILL.md && echo "skill: PRESENT"
ls .claude/skills/rest-standards/references/ | wc -l          # expect 4
test -f docs/skills/rest-standards.md && echo "docs: PRESENT"
test -f docs/reviews/2026-08-10-phase-7-skill-self-test.md && echo "self-test: PRESENT"
npx @stoplight/spectral-cli lint --ruleset conformance/spectral.yaml \
  conformance/fixture-violations.yaml 2>&1 | tail -3
```

Expected: all four `PRESENT` markers, `4` reference files, and the Spectral run
reporting **12 problems (7 errors, 5 warnings)** — the count recorded in
Appendix G. A different count means either the ruleset or the fixture changed;
stop and reconcile before claiming the gate.

- [ ] **Step 2: Verify no rule text changed in this phase**

```bash
git diff origin/main --stat -- rest-api-standard.md conformance/
```

Expected: **empty**. Phase 7 adds an applier; it does not amend the standard.
Any diff here means an amendment leaked into the phase — extract it into its
own change with a Part II Decision Log row and a `CHANGELOG.md` entry.

- [ ] **Step 3: Update `PLAN.md`**

Flip the Phase 7 planned-artifacts row from `Designed 2026-08-10 …; authoring
pending` to the completed state with the self-test evidence path, and record
Gate F as passed with its date.

- [ ] **Step 4: Commit and open the pull request**

```bash
git add PLAN.md
git commit -m "docs: Phase 7 complete — Gate F passed, rest-standards skill shipped"
git push -u origin worktree-docs+phase-7-skill
gh pr create --title "docs: Phase 7 — the rest-standards skill apparatus" --body "..."
```

The pull request body states: what the skill does, the three deliberate
divergences from `cli-standards` (evidence planes, procedure-only, the gated
runtime plane), the self-test result, and that no rule text changed.

- [ ] **Step 5: Drive the review to merge**

Use the `pr-merge-flow` skill. This repository's convention is that every gate
lands via a bot-reviewed pull request (#3–#7), and the merge strategy is a
merge commit.

---

## Self-Review

**Spec coverage.** Every spec section maps to a task: §3.1 identity →
Task 1 Step 1; §3.2 layout → the File Structure table; §3.3 navigation →
Task 1 Step 4; §3.4 dials → Task 1 Step 6 and Task 2; §3.5 planes → Task 5
Step 1; §3.6 probe gate → Task 5 Step 3; §3.7 modes → Task 1 Step 5 and
Tasks 3–5; §3.8 traps → Task 4 Step 2; §3.9 amendments → Task 1 Step 7;
§5 ownership split → Task 7; §6 verification → Tasks 6, 7, and 8 Step 1.

**Known gap, deliberately left open.** Spec §8 notes that a Phase 6 streaming
section could introduce an evidence plane that is neither a document nor a
request/response pair. No task covers that, because Phase 6 has not landed.
When it does, §3.5 and `audit.md` need revisiting — this is recorded here so
it is not lost.

**Type consistency.** The names used across tasks are fixed: the four mode
names, the four reference filenames, the three plane names
(`contract`/`source`/`runtime`), the scope-record shape (Task 2 Interfaces),
the findings-table columns (Task 4 Interfaces), and the conformance summary
line (Task 5 Interfaces). Later tasks reuse these verbatim.
