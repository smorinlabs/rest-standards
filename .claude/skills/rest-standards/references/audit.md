# audit mode — conformance sweep of an existing API

Gap-analyze a real API against the standard: brownfield APIs never built to
it, and pre-release conformance gates. Depth scales with the evidence
available, not with tier — §1.7 tiers name an audience, and every rule applies
at every tier.

Findings use review mode's table: `Rule | Level | Where | Finding | Fix`.

## The three evidence planes

| Plane | Input | Checker |
|---|---|---|
| **contract** | Any documented interface contract — OpenAPI or JSON Schema, else reference docs or worked request/response exchanges | `conformance/spectral.yaml`, or direct reading (below) |
| **source** | Routes, handlers, middleware | Read / Grep / ast-grep |
| **runtime** | A deployed base URL | Appendix G probes — gated, see below |

Every finding names the plane that produced it. Planes overlap: where two
reach the same rule, report both. Agreement is a stronger result than either
alone; **disagreement is itself a finding** — the deployment and the source
have diverged.

## 1. Contract plane

    npx @stoplight/spectral-cli lint \
      --ruleset <standard-repo>/conformance/spectral.yaml <contract-document>

The ruleset's rules are conservative heuristics; each description states its
known false-positive and false-negative limits. Warn-severity findings exist
to be reviewed, not blindly enforced. Read Appendix G for what the ruleset
does and does not traverse before treating a clean run as conformance.

**When there is no machine-readable document.** Reference documentation and
worked request/response exchanges are still contract evidence — a contract is
what the provider has documented, not the file format it is documented in.
Spectral cannot run, so say so in the scope record and read the contract
directly instead; the plane is available, only its automated checker is not.
Never downgrade to source-only on this basis, and never treat the missing
Spectral run as a clean one. A published contract that R4.1 requires to be an
OpenAPI document, and isn't, is itself an R4.1 finding.

## 2. Source plane

Read the routes, handlers, and middleware for what the contract cannot
express: authorization checks and existence masking, idempotency-key storage
and retention, redaction in error paths, the documented default sort order
behind cursor pagination, and rate-limit enforcement points. Grep for reserved
names (§1.10) used with a non-registered meaning.

## 3. Runtime plane — gated

Appendix G probes hit a real deployment. Some are destructive *precisely when
the API fails the check* — an unguarded DELETE succeeds, an unimplemented
method turns out to be implemented, a `dry_run` parameter executes for real —
so a probe's expected response does not tell you it is safe. The gate is not
optional.

1. **Default: issue no HTTP requests.** Contract and source planes only.
2. **The user must ask.** Then require: a base URL, an explicit statement that
   the deployment is non-production (local, sandbox, or staging), and which
   resources are disposable.
3. **Classify every probe before running any of them.** Read Appendix G's
   live probe table and sort each row into one of the three tiers below by
   what its *request* does — never from a remembered list, which cannot
   contain the probes a later amendment adds. Appendix G does not mark the
   tiers; that judgment is this skill's, and it is made per row:
   - **read-only** — the request neither changes state nor degrades service
     for other clients. Run these first.
   - **mutating** — the request can change state, including when it changes
     state only *because* the API fails the check: a guard that does not hold
     lets the request through for real. Needs the second confirmation in
     step 4.
   - **disruptive** — the request degrades service for every other client of
     that deployment while it runs. Needs step 4's confirmation *and* an
     explicit warning of that side effect before running.

   When a row's tier is genuinely unclear, treat it as mutating. Reading its
   Expected column settles most cases: an expected rejection still executes
   for real wherever the guard is the thing that is broken.
4. **Mutating and disruptive probes need a second confirmation** naming the
   disposable fixture the probe may touch.
5. **Never** against production, against an API the user does not own, or with
   credentials the user has not deliberately provided for this purpose.
   Reviewing a third party's *published contract* on the contract plane
   remains available and needs no gate.
6. **Anything not run is reported unverified with the exact `curl`** for the
   user to run by hand. This is also the retreat path when a probe turns out
   to be riskier than it looked.

Read Appendix G for the live probe table; do not work from the list above,
which names the gate tiers rather than the probes.

## 4. Report

Findings table (blockers first), then:

- N/A switches with their reasons (R1.6).
- Unverified rules, each with the plane that was missing and, for runtime
  rules, the `curl` that would settle it.
- The evidence planes actually used.
- Standard version.
- A conformance summary line: `<N> applicable MUSTs: <P> pass, <F> fail,
  <U> unverified`.

**Counting `N`, so two audits of one API agree.** An *applicable MUST* is a
Part I rule that (a) carries at least one capitalized MUST / MUST NOT /
REQUIRED / SHALL clause per R1.1, (b) binds the audited API — its provider
behavior, or a §12 client obligation the provider must surface — rather than
binding this standard's own drafting, and (c) is not scoped to a switch
declared off. Show the subtraction from the live rule count; do not restate a
remembered `N`. Rules whose only keywords are SHOULD or MAY are reported in
the findings table but never counted here.

**One verdict per rule, and a rule is not its strongest clause.** Most rules
carry several clauses. Score `fail` if any clause is demonstrably violated;
`pass` only if the evidence settles *every* MUST clause the rule carries;
otherwise `unverified`, naming which clause went unreached. A rule whose
clauses split — R3.7's media type demonstrated but its `Accept-Patch`
advertisement never shown — is `unverified`, not `pass`: scoring it `pass`
is exactly the inferred pass the plane discipline forbids. Expect `U` to
dominate on excerpt-shaped evidence; a large `U` is the method reporting its
own reach, not a defect in the API.

## 5. Artifacts (offer, don't assume)

1. **Conformance note** — render the §1.9 template (read live) or update the
   existing one: tier, switches with reasons, deviations, N/A declarations.
2. **CI wiring** — add the Spectral ruleset to the target repo's CI so the
   contract plane is checked on every change, not once.
3. **Fix plan** — hand ordered blockers to the normal planning flow
   (writing-plans) if the user wants them fixed now.
4. **Amendments** — deviations that beat the rule go upstream as Part II
   Decision Log proposals (SKILL.md workflow step 6).
