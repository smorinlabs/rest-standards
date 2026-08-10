---
name: rest-standards
description: Apply the org's REST API Design Standard (rest-api-standard.md in this repo) to any HTTP API work, scaled by conformance tier (internal / partner / public), applicability switches, and available evidence. Four modes — plan (greenfield interview → OpenAPI skeleton + seeded conformance note), check (mid-build lookups — "what status code / header / query parameter does the standard say"), review (design review of an API spec, OpenAPI document, or unshipped diff; findings cite rule IDs), audit (conformance sweep of an existing API across three evidence planes — contract document via Spectral, source, and gated live probes against a non-production deployment). Fires whenever an HTTP API is being created, designed, extended, reviewed, or audited — "new API", "design this endpoint", "review this OpenAPI spec", "is this API conformant", "audit this API", "REST standards". Not for CLIs (cli-standards), generic non-API architecture review (factor-architect), generic quality sweeps (factor-scan), or external REST-practice research (guided-research).
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
memory, never from a bundled copy. Resolve the skill's base directory first
(it is a symlink under a global placement), then walk up to the repo root:

    STD="$(dirname "$(dirname "$(dirname "$(realpath "<skill-base-dir>")")")")/rest-api-standard.md"

Read the header and note the **Version** line:

    grep -m1 -oE '\*\*Version [0-9]+\.[0-9]+\.[0-9]+' "$STD"

Every deliverable — spec, review, audit, conformance note — pins the standard
version it was produced against. If the file is missing, stop and report; do
not proceed from memory.

## Navigating the standard

~2,400 lines. Never read it whole; never answer from memory. Get the live
section map, then read only the sections in play:

    grep -n '^## ' "$STD"          # 24 top-level sections: Part I §1–§12, Part II, Appendices A–G
    grep -n '^### ' "$STD"         # subsections, when a section is large

The index is deliberately not written down here. The standard evolves by
Part II amendment — a hardcoded map would go stale silently, a grepped one
cannot.

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
