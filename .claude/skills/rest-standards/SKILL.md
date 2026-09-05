---
name: rest-standards
description: Apply the org's REST API Design Standard to HTTP API work in plan, check, review, and audit modes, scaled by conformance tier. Use when an HTTP API is being created, extended, reviewed, or audited, such as "design this endpoint", "review this OpenAPI spec", or "is this API conformant". Not for CLIs (cli-standards) or generic architecture review (factor-architect).
allowed-tools: Read, Grep, Glob, Bash, Write, Edit, AskUserQuestion
---

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

## Locating the standard

This skill ships beside the canonical standard and reads it live — never from
memory, never from a bundled copy. Substitute `<skill-base-dir>` below with the
directory holding this `SKILL.md` — the harness names it when it loads the
skill, and it is often a symlink into the repo. `realpath` resolves the link;
the three `..` steps then climb out of `rest-standards`, `skills`, and
`.claude` to reach the repo root:

    STD="$(realpath "<skill-base-dir>")/../../../rest-api-standard.md"
    REPO="$(realpath "<skill-base-dir>")/../../.."

`$STD` and `$REPO` below are shorthand for the paths these two commands print,
not live shell variables — shell state does not survive between commands in
most harnesses. Run each command on its own, record its output as text, and
paste the literal path into every later command. **Wherever `$STD` or `$REPO`
appears in a command in this skill, substitute that recorded path**; the
variable form is written for brevity and will be empty if pasted as-is.

Read the header and note the **Version** line:

    grep -m1 -oE '\*\*Version [0-9]+\.[0-9]+\.[0-9]+' "<the path STD printed>"

Keep each of these as a short, standalone command. Some harnesses refuse a shell command
they cannot statically verify, and nesting or bundling these with other
commands is what trips that check. **A refusal to run the command is not a
missing standard** — it is a shell problem, so retry rather than stop: run
`realpath "<skill-base-dir>"` on its own, then use its literal output in place
of the substitution in a second plain command.

Every deliverable — spec, review, audit, conformance note — pins the standard
version it was produced against. Stop and report only when the file itself is
absent (`test -f "$STD"` fails); never proceed from memory.

## Navigating the standard

Thousands of lines. Never read it whole; never answer from memory. Get the
live section map, then read only the sections in play:

    grep -n '^## ' "$STD"          # every top-level heading — parts, sections, appendices, and headings embedded in templates
    grep -n '^## [0-9]' "$STD"     # just the numbered normative sections, in order
    grep -n '^### ' "$STD"         # subsections, when a section is large

Neither the index nor its size is written down here, and no count of sections
or rules appears anywhere in this skill. The standard evolves by Part II
amendment — a hardcoded map would go stale silently, a grepped one cannot.

## Modes

Pick by what the user is doing; ask only when genuinely ambiguous.

| Mode | Fires when | Load | Deliverable |
|---|---|---|---|
| **plan** | Greenfield API or new resource group: "new API", "design this endpoint" | `references/planning.md` | OpenAPI skeleton + seeded conformance note + Spectral lint pass |
| **check** | Mid-build lookup: "what status code / header / parameter name…" | The relevant § of the standard | Cited answer with rule IDs; nothing written |
| **review** | A spec, OpenAPI document, or diff exists but hasn't shipped | `references/review.md` | Findings table with rule IDs |
| **audit** | An existing API: "is it conformant", pre-release gate | `references/audit.md` | Per-plane findings + conformance summary line + conformance note + optional fix plan, CI wiring, and amendments |

Before plan, review, or audit: settle **tier and switches** via
`references/scoping.md`, then the **evidence plane** under that file's §3,
which owns the per-mode rule. Review and audit always survey. Plan mode skips
the survey only for a genuinely greenfield API — there `planning.md`'s
Deliverable 1 *creates* the contract plane and Deliverable 3 checks it — and
surveys when the plan extends an existing API's contract. `check` skips
scoping; it is a lookup, not a judgment.

## Depth scaling

Three dials. The first two are the standard's and are read live; the third is
this skill's, and is procedural.

- **Tier** (§1.7) — `internal` / `partner` / `public`. Names the API's
  *audience*, not its quality bar. Every rule applies at every tier unless the
  rule itself carries a tier scope. Tier never waives a MUST.
- **Switches** (§1.8) — read the live switch vocabulary from §1.8; do not
  assume it. A switch that is off removes the rules *scoped to it* and MUST
  carry a stated reason (R1.6): `N/A — <switch>: off, <reason>`. "N/A" with no
  reason is a deviation, not an exemption. **Off is not empty:** §1.8 exempts
  guard rules — those defining what an API *without* the capability must do —
  from their own switch, and they bind per endpoint, so an off switch still
  leaves binding rules in that section. A rule's own provenance line, not the
  section it sits in, is authoritative for its scope.
- **Evidence plane** — contract / source / runtime. Decides which rules can be
  verified *at all*. Planes overlap; agreement between two is a stronger
  result than either alone, and disagreement is itself a finding.

A rule no available plane can reach is reported `unverified` with the reason —
never an inferred pass.

## Workflow (every mode)

1. Locate and version-pin the standard (above).
2. Identify the mode; for plan/review/audit, settle tier + switches, then the
   evidence plane on `references/scoping.md` §3's per-mode rule.
3. Execute the mode's reference procedure.
4. Cite rule IDs (`R#.#`) on every finding and answer, with MUST/SHOULD/MAY
   severity — MUST violation = blocker, SHOULD deviation = fix or waiver,
   MAY = suggestion. **Only `R#.#` is a rule.** Every other identifier series
   in the document is frozen research provenance (R1.3) and is never citable
   as a requirement; R1.3 names the series live, and amendments add more.
   Even an ID shaped like a rule can belong elsewhere — R1.3 assigns
   `CLI-`-prefixed IDs to the CLI Design Standard, not to this one.
5. Deviations that stay: record in the target repo's conformance note,
   rendered from the template in §1.9 read live. No placeholder may survive:
   §1.9's template marks them with angle brackets (`<API name>`, `<version>`,
   `<rule ID>`, `<free text …>`). Every one is replaced, or the note is not
   rendered.
6. **Feed back upstream.** A deviation that seems *right* and generalizable is
   a proposed amendment: draft a Part II Decision Log row, the rule edit, and a
   `CHANGELOG.md` entry against `rest-api-standard.md` in this repo, and offer
   it to the user. The standard is released — the version pinned in step 1 is
   the live one — and it evolves by amendment under the Part II rule, not by
   drift.

## Red Flags

| Thought | Reality |
|---|---|
| "Internal API — the standard is overkill" | §1.7 tiers name an audience, not a depth. Every rule applies at every tier. |
| "I remember what the standard says" | The rule set is larger than memory holds and grows by Part II amendment. Read the section. |
| "`X-Correlation-Id` is close enough" | §1.10 reserves `request-id`; R1.8 makes the reserved name the contract, and RFC 6648 rules out new `X-` names. |
| "`AC-016` requires idempotency keys" | Not an `R#.#`, so not a rule — it is research provenance (R1.3). Find and cite the rule. |
| "I can audit from the OpenAPI document alone" | The contract plane cannot reach runtime behavior. Probe (gated) or mark those rules unverified. |
| "This deviation is fine, moving on" | Fine → conformance-note entry. Generalizable → Part II amendment. Never just "moving on". |

## See also

- `cli-standards` — the sibling standard and skill, for command-line tools.
- `factor-architect` — generic (non-API) architecture review.
- `factor-scan` — generic code-quality sweeps; conformance is not quality.
- `guided-research` — researching external REST practice; this skill applies
  the settled internal standard.
