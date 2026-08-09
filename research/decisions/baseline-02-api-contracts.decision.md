# Decisions — baseline-02 (API contracts)

*Gate C ratification record. One entry per decided principle; entries are
appended as the walkthrough proceeds. A `baseline` report proposes; only this
file ratifies. Classification per `PLAN.md` Phase 2: protocol requirement ·
evidence-backed default · project policy · exception · unresolved question.*

---

## AC-003 — Errors MUST be servable as RFC 9457 `application/problem+json`

**Decision (2026-08-09): RATIFIED as MUST, with amendments (a)–(c) below.**
Decided by the project owner in the Gate C walkthrough after two rounds of
additional research (`baseline-02d`, `baseline-02e`, `baseline-02f`).

**Classification:** evidence-backed default (the mandate itself) resting on a
published Standards Track RFC; amendments (b) and (c) contain labeled
`[POLICY]` components.

### The ratified rule

Every API **MUST** be *capable of returning* every error response as
`application/problem+json` per RFC 9457 when the client requests it.
Obligation follows Zalando rule `[#176]`'s proven wording ("must be capable
of returning"), scoped to **responses the application itself generates**.

**Exception (named carve-out):** errors emitted by infrastructure components
outside application control — reverse proxies, CDNs, WAFs, rate limiters,
load balancers that terminate the request before it reaches application
code — which MUST be documented as such. Evidence that the carve-out is real
rather than an escape hatch: Cloudflare's own edge served a legacy
`text/plain` body for a real enforced rate limit despite
`Accept: application/problem+json` (`baseline-02e`, live-verified
2026-08-09).

### Amendments ratified with the rule

- **(a) Obligation wording** — "capable of returning" + the infrastructure
  carve-out, as above.
- **(b) `type` semantics** — the standard rules explicitly on `type`
  (detailed design ratified separately as the AC-004 amendment below):
  stable absolute `https` URI under a provider-controlled domain, 1:1 with
  the required `code` extension via a published template; dereferencing is a
  courtesy, never a contract; human documentation lives in a separate
  `documentation` member.
- **(c) Registry independence** — nothing in the standard is premised on the
  IANA HTTP Problem Types registry (6 entries in 3 years, half from an
  unpublished draft; formally closed to application-specific values;
  unmentioned by the largest deployment in existence).

### Justification — re-argued at ratification

The original argument ("the field is converging on RFC 9457") is
**withdrawn**: `baseline-02d` falsified `baseline-02c`'s absence claim
(CAMARA's TSC formally decided "not to be implemented", 2024-04-15, and its
current guide still ships a proprietary `ErrorInfo`). The mandate stands on
the replacement argument, which is better evidenced:

1. **No credible alternative exists.** JSON:API errors and `google.rpc.Status`
   are primary-verified as non-competitors (neither references RFC 9457;
   neither is a cross-vendor movement; JSON:API shares the all-optional
   weakness). What rejectors produce in practice is *n* mutually incompatible
   bespoke envelopes (CAMARA, Australia api.gov.au, US 18F, Anthropic — each
   different).
2. **Adoption trend among standards-setting bodies is toward it**, with dated
   currency: Zalando MUST; Netherlands tightening from linter-warning
   (published 2025-08-27) to MUST (2026-07-09 draft); Belgium SHOULD;
   ACME/RFC 8555 at internet scale since 2019; Cloudflare network-wide
   2026-03-11 with a measured 55–64× payload/token reduction for agent
   consumers.
3. **The 2026 agent-consumption argument** (machine-parseable errors for LLM
   agents) is a new, quantified benefit class postdating the original
   research.

**Confidence: moderate** — lowered from high-moderate when the absence claim
fell; the fork is genuine (CAMARA proves reasonable organizations decline),
but rejecting now means designing a bespoke envelope with no standards
backing, which the evidence shows does not converge.

**Fork honesty:** a reviewer weighting ecosystem familiarity over standards
alignment could still invert this. Options offered and declined at
ratification: MUST with literal-Cloudflare `type` layout (docs URL in `type`,
dispatch on `code`) — declined for the documented collision defect
(`baseline-02e` Q3(c)) and because RFC 9457 §3.1.1's consumer MUST points at
`type`; SHOULD — declined as recreating the many-shapes problem; bespoke
envelope — declined per justification 1.

**Evidence:** `baseline-02` §7 (AC-003 row) and §8.1 · `baseline-02c` (as
amended 2026-08-09) · `baseline-02d` · `baseline-02e` · `baseline-02f`.

**Consequences for drafting (Phase 3):**
- Adopt the Zalando-style client-robustness note: clients must not *rely* on
  a problem document being returned (infrastructure may intervene).
- Media-type interop hazard to document: `application/problem+json` is often
  not treated as a subset of `application/json` by libraries; clients should
  send it explicitly in `Accept`. Consider Cloudflare's mirroring pattern
  (same body under `application/json` when requested) as a compatibility
  measure — left to Phase 3.
- Never cite the IANA registry as a conformance surface (per amendment (c)).

---

## AC-004 (amended) — `type`/`code` binding, `documentation` member, `about:blank` ban

**Decision (2026-08-09): RATIFIED as amended.** Confirmed by the project
owner in the Gate C walkthrough, as the knock-on of AC-003 amendment (b).

**Classification:** project policy (labeled `[POLICY]`), grounded in
primary-sourced failure evidence; the underlying member requirements remain
an evidence-backed default.

### The ratified design

AC-004's required members stand (`type`, `title`, `status`, stable
machine-readable `code` extension), with these amendments:

1. **`type` is the normative identifier; `code` is its short form.** Every
   problem type has exactly one of each, bound by a fixed template the
   standard mandates: `<https base>/<code, underscores to hyphens>` — e.g.
   `code: "out_of_credit"` ⇒ `type: "https://problems.example.com/out-of-credit"`.
   **The standard fixes the template shape; each API declares its base URI.**
2. **Neither `type` nor `code` may change once published.** A change of
   meaning is a new problem type with a new pair.
3. **Dereferencing `type` is a courtesy, never a contract.** A provider MAY
   serve a redirect from the `type` URI to current documentation; clients
   MUST NOT depend on it resolving.
4. **Human documentation lives in a separate `documentation` member** (named
   `documentation`, not Belgif's `href`, to avoid hypermedia-link
   connotations), which MAY change over time and MAY vary by environment.
5. **`type` MUST be present on every problem document, and `about:blank`
   MUST NOT be used.** Ban rationale: AC-004 requires a discriminating
   `code` on every document, and RFC 9457 §4.2.1 defines `about:blank` as
   carrying "no additional semantics beyond that of the HTTP status code" —
   the combination is self-contradictory. Banning removes the conflict
   rather than managing a two-mode rule.
6. A `urn:` `type` is permitted for providers operating an IANA-registered
   URN namespace identifier (keeps Belgif-/ACME-style APIs conformant).

### Deliberate deviations from RFC 9457 — all permitted by it, all `[POLICY]`

- Declining the §3.1.1/§4 `SHOULD` on resolvability (same written deviation
  Zalando and Belgif made).
- Requiring `type` (RFC makes it optional).
- Forbidding `about:blank` (RFC registers and defaults to it). This is the
  only element stricter than any surveyed adopter.

**Evidence:** `baseline-02f` throughout — RFC 9457 §3.1.1/§3.2/§4/§4.2.1
close reading; IETF WG record (editors' back-compat constraint,
primary-sourced); ASP.NET Core .NET 7→8 `type` identity break;
`httpstatuses.com` domain repurposing; Belgif's live `href`-churn warning;
Cloudflare's stability promises living on extension members
(`baseline-02e`).

---

## AC-007 (completed) — Field casing: snake_case, bodies and query parameters

**Decision (2026-08-09): RATIFIED.** `AC-007` (one casing convention across
bodies and query parameters — already proposed as MUST) is completed with
the concrete choice: **`snake_case`**, enforced by the Zalando-style
pattern `^[a-z_][a-z_0-9]*$` for body properties and `^[a-z][_a-z0-9]*$`
for query parameters.

**Classification:** project policy `[POLICY]` — the survey establishes that
no evidence can settle the choice itself; the value is uniformity.

**Justification:** `[COMPARATIVE]` the field splits two ways — snake_case
(Stripe, GitHub, Twilio bodies, Shopify; Zalando mandates it) vs camelCase
(Microsoft Graph; Google's JSON, which is machine-derived from snake_case
protos via the proto3 JSON mapping). Zalando is the only surveyed guideline
mandating body+query consistency, which `AC-007` requires; Twilio's
snake-bodies/PascalCase-params split is the documented anti-pattern.
snake_case also matches the conventions this standard has already aligned
with (Standard Webhooks headers/secrets, Cloudflare's extension members).

**Options declined:** camelCase (JS idiom; rarer for query params among
surveyed vendors) · deferral (costs a structural lock that field-selection
and filter-grammar items want settled).

**Confidence:** high that one convention is required (`AC-007` as
proposed); the concrete pick is policy, not evidence.

**Evidence:** `survey-03` finding 1 and Table A · `survey-02` finding 4 ·
`baseline-02` §7 (AC-007 row).
