# review mode — design review of a spec, OpenAPI document, or unshipped diff

Review the *interface* before it hardens into shipped behavior. Input is
whatever exists: an OpenAPI document, a design doc, a Superpowers plan, a PR
diff, or README examples. This is not a code-quality review (that is
factor-scan) and not generic architecture review (factor-architect) — every
finding here traces to a rule ID.

## Procedure

1. Tier, switches, and evidence plane settled first (`references/scoping.md`).
2. Sweep **every** numbered normative section, **in the order the standard
   lists them**, skipping only groups whose applicability switch is off.
   Derive the list live rather than working from a remembered one — this
   command is the sweep checklist, and it grows on its own as the standard is
   amended:

       grep -n '^## [0-9]' "$STD"

   Walk its output top to bottom, reading each section from the live standard
   as you reach it — never from memory. Skipping a section requires an off
   switch and its reason (R1.6); nothing else licenses a skip, and a section
   you did not recognize is one you have not read yet, not one that does not
   apply.
3. Check the cross-cutting traps below.
4. Where the input is silent on an applicable area, that is a **gap finding**,
   not a pass.

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
