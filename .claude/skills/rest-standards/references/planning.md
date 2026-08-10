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

## Deliverable 3 — prove the skeleton on the contract plane

Lint the skeleton with the standard's own ruleset before handing it over:

    npx @stoplight/spectral-cli lint \
      --ruleset <standard-repo>/conformance/spectral.yaml openapi.yaml

Zero errors is the bar. Warnings are conservative heuristics — review each and
either fix it or record it, never silently ignore it.
