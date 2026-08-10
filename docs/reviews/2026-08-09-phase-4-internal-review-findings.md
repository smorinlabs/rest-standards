# Phase 4 internal review — findings ledger (2026-08-09)

The first Phase 4 pass per `PLAN.md`: three independent review agents run
against the Gate D-approved draft at `5213abe` (merged to `main` via PR #5,
merge commit `175d687`) — missing/conflicting-rules pass (MC, opus),
ambiguity search (AM, sonnet), domain-gap audit (GA, sonnet). Two findings
were reported by two agents independently (marked "overlap"). Fixes land on
the `worktree-docs+phase-4-review` branch; the Codex adversarial pass (the
different-model-family second lens) runs AFTER these fixes, before Gate E.

## Major

1. MC-2: R5.13 dropped AC-004's ratified constraint — `type` must be an absolute
   URI **under a domain the provider controls** (decision B2:145-148; the
   httpstatuses.com repurposing evidence is the reason). Restore the clause.
2. MC-1: R12.8 cites "per the §8.3 replay axis" — but the five-axes decision
   says the replay axis "does not reopen the ratified webhook 5-min convention"
   (B3:367), and that firewall sentence is also missing from R8.10's replay row.
   Fix both: decouple R12.8's 300s from §8.3; add the carve-out to R8.10.
3. MC-3: E.7 emits `Location` on 202 but §1.10 registers `Location` only for
   201 creates and R1.8 forbids other uses. Broaden the register row (RFC 9110
   Location is general) and/or name the header in R10.1.
4. MC-4: A5's ratified action response shape (sync 200 + mutated representation;
   long-running 202 + AC-019 operation) exists in no rule. Add (R2.11 extension
   or §5.2 row) + Appendix C action row.
5. MC-5: R4.15's "a preference token MUST NOT carry safety semantics" has no
   ratified basis and is not Apparatus-marked (violates R1.3's own warning).
   Either demote to rationale prose or mark Apparatus with a Gate note.
6. AM-1: five of eight applicability switches (multi-tenant, public-internet,
   handles-pii, third-party-clients, file-upload) gate no rule. Either tie each
   to its rules (handles-pii→R2.10/R8.12 are unconditional — decide) or prune
   the vocabulary.
7. AM-2/MC-9 (overlap): `cancel` registered for "in-flight process" but E.6
   applies it to an order; `archive`/`restore` register row claims "the DELETE
   rule references" it — R5.7 doesn't. Broaden/clarify the cancel registration;
   fix or drop the parenthetical; consider R5.7 cross-ref.
8. AM-3: "destructive operation" undefined yet sets strengths in R3.12 (SHOULD
   dry-run) and R7.4 (MUST If-Match). Add §1.11 definition.
9. AM-4: provenance lines cite gap-review items as "R2.3/R9.7/R8.1/R7.12" —
   colliding with real rule IDs. Rename citations to "gap review item
   GAP-R2.3" style everywhere.
10. AM-5: R8.10's operative sentence lacks BCP 14 keywords ("adopts the
    defaults...records any flip"). Make it "MUST adopt ... MUST record".

## Minor

11. MC-6: R1.9 says "carrying `dry_run=true`" — ratified guard is "carrying it"
    (any value). Reword.
12. MC-7: Appendix C quick map inverts R5.10 (shows 403 primary, 404 alt).
13. MC-8/AM-6 (overlap): §1.10 Retry-After row says "Mandatory on 429; also
    used on 503/202" — R11.5 makes 503 a MUST, R10.2 makes 202 a SHOULD.
    Rewrite row: "MUST on 429 and 503; SHOULD on 202 (R10.2)".
14. AM-7: R2.5 three-resource cap has no counting rule (version segment?
    action segment?). Add boundary example.
15. AM-8: R3.10 "supports conditional update" vs R3.11/R7.4 "exposed to
    concurrent modification" — two undefined gating phrases. Unify via §1.11
    term.
16. AM-9: R2.3 "domain terms" MUST has no observable test. Split checkable
    (no abbreviations, one noun per concept) from judgment (SHOULD).
17. AM-10: archive/restore vs publish/unpublish visibility boundary unstated.
    One distinguishing clause.
18. AM-11: R8.3 "single trust relationship" undefined. Observable criterion.
19. AM-12: R6.3 exception "small or stable collections" unquantified. Proxy or
    ceiling.
20. GA-1: AC-003's third drafting consequence (Cloudflare application/json
    mirroring pattern) never adopted-or-declined. One sentence near R5.12.
21. Carried: path-parameter naming candidate (Part II register, open item) —
    decide snake_case rule or record deliberate silence.

## Disposition (2026-08-09, `worktree-docs+phase-4-review`)

Fixed in the fix-pass commit: MC-1 (R12.8 decoupled from the replay axis;
firewall sentence added to R8.10's row), MC-2 (provider-controlled-domain
constraint restored to R5.13.1), MC-3 (`Location` register row reframed to
bind 201 without restricting RFC-sanctioned uses; E.7 unchanged), MC-4 (A5
response shape added to R2.11 + checklist), MC-5 (R4.15's unratified MUST
NOT demoted to rationale prose), MC-6 (R1.9 covers any `dry_run` value),
MC-7 (Appendix C shows 404-default masking), MC-8/AM-6 (Retry-After row
states MUST 429+503, SHOULD 202), MC-9 partial (R5.7 now references
`archive`/`restore`), AM-3 (destructive operation defined in §1.11), AM-4
(gap-review citations prefixed `CLI-`), AM-5 (R8.10 keywords), AM-7 (R2.5
counting rule + boundary example), AM-8 (R3.10/R3.11 same-class entailment
stated), AM-9 (R2.3 split checkable MUST / judgment SHOULD), AM-10
(archive-vs-publish boundary stated), AM-11 (single-trust criterion),
AM-12 (R6.3 exception documentation requirement), GA-1 (mirroring pattern
explicitly permitted-not-required).

To the owner walk: AM-1 (five switches gate no rule — prune vs tie), AM-2
(`cancel` scope vs E.6's order usage), the path-parameter naming
candidate, and optionally ratifying `Location`-on-202 as a SHOULD.

## Process after fixes

- Re-run structural check suite (IDs, cross-refs, keywords, checklist count —
  checklist rows must track any rule-text changes; Part II rows likewise;
  worked example per amendment rule's five surfaces).
- Codex adversarial pass (different model family) on the corrected text.
- Owner walk for any item that changes a ratified decision's meaning (MC-1,
  MC-2 restorations are fidelity fixes, not decision changes; AM-1 switch
  pruning and cancel-scope broadening are owner calls).
- Then Gate E.
