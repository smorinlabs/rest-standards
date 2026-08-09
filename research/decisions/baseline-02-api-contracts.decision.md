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

1. **No credible alternative exists** — `[INFERENCE]`, per `baseline-02d`,
   supported by primary-verified comparisons: JSON:API errors and
   `google.rpc.Status` are non-competitors (neither references RFC 9457;
   neither is a cross-vendor movement; JSON:API shares the all-optional
   weakness). What rejectors produce in practice is *n* mutually incompatible
   bespoke envelopes (CAMARA, Australia api.gov.au, US 18F, Anthropic — each
   different).
2. **Standards-setting bodies that adopt it are tightening, and the pattern
   is regional rather than chronological** (`baseline-02d`: continental-EU
   guidelines converge on it; Anglosphere guidelines are silent or custom).
   Dated currency: Zalando MUST; Netherlands tightening from linter-warning
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
   The `code` grammar is snake_case only — `^[a-z][a-z0-9_]*$`, hyphens
   excluded — so the underscore-to-hyphen mapping is injective and two codes
   can never collide on one `type`.
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
   This is an **explicit exception to point 1's https requirement**: the
   normative rule reads "an absolute URI under a domain the provider
   controls — or, for providers operating an IANA-registered URN namespace
   identifier, a URN in that namespace"; the 1:1 `code` binding and
   immutability rules apply identically to both forms.

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

---

## AC-016 (completed) — Idempotency key travels as the `Idempotency-Key` header

**Decision (2026-08-09): RATIFIED.** `AC-016` (accept an idempotency key on
non-idempotent state-changing requests; fingerprint the payload; reject a
reused key carrying a different payload — already proposed, `[POLICY]`) is
completed with the placement: a **request header named `Idempotency-Key`**,
Stripe-semantics. The convention MUST be labeled `[POLICY]` and never cited
as a standard — the IETF draft that standardized this shape expired
2026-04-18 with intended status "(None)".

**Classification:** project policy `[POLICY]` throughout.

**Justification:** the fork is really header vs **query parameter**
(`baseline-02g` corrected the common "body field" framing: Google's
AIP-155 `request_id` transcodes to the query string over REST). For the
header: Stripe's installed convention and the expired draft's shape; UCP
signs an `idempotency-key` header as a covered component of its RFC 9421
profile (`baseline-03d`) — answering the headers-escape-signatures
objection for the architecture ratified in `OP-016`; both OpenAI's and
Anthropic's SDKs carry dormant Stainless machinery wired for exactly this
header (`baseline-02g`); headers keep keys **out of query strings**, which
land in access logs, referrers, and URL-keyed caches by default — a header
can still be logged by proxies and applications, so deployments that need
the stronger guarantee MUST add explicit header-redaction and
cache-key-exclusion configuration. Against the query model: AIP-155 defines
no same-key-different-payload behavior, so it cannot satisfy `AC-016`'s
fingerprinting requirement as specified. The three modern AI vendors ship
no mechanism at all, so no counter-signal exists (`baseline-02g`).

**Options declined:** query parameter (log/proxy leakage; no
fingerprinting) · body field (schema pollution; no surveyed vendor ships
it over REST).

**Confidence: high-moderate.**

**Evidence:** `baseline-02` §7 (AC-016 row) and §8.3 · `baseline-02g` ·
`baseline-03d` (UCP covered components) · `survey-05` (Stripe mechanics).

---

## Money representation (companion to AC-010)

**Decision (2026-08-09): RATIFIED.** Monetary amounts are encoded as
**minor-unit integers** with a separate ISO 4217 `currency` field —
Stripe's model: `"amount": 1099` with `"currency": "usd"` meaning $10.99,
the amount expressed in the currency's smallest unit. `AC-010`'s float ban
stands unchanged beneath this.

**Classification:** project policy `[POLICY]` — all three candidate
encodings are float-safe; the choice is preference among safe options.

**Owner decision note:** the walkthrough recommended decimal string
(Shopify model) for parser-safety-without-currency-tables; **the owner
chose minor-unit integer**, weighting the battle-tested payments-domain
practice. Consequence to carry into Phase 3 drafting: clients need the
ISO 4217 exponent to render amounts (JPY exponent 0, BHD exponent 3), so
the standard must require the `currency` field alongside every amount and
should point at the exponent table.

**Declined:** decimal string (recommended; explicit parsing, no exponent
table) · decimal number with `format: decimal` (plain `JSON.parse` lands
in a float — the corruption AC-010 bans).

**Confidence: moderate** — a genuine fork among safe encodings.

**Evidence:** `survey-03` TL;DR axis 3 and money rows (Stripe minor-unit
integer; Shopify decimal string; Zalando decimal number; no reputable
reference uses floats).

---

## AC-001 (completed) — OpenAPI version pin: 3.1 floor, 3.2 gated on toolchain

**Decision (2026-08-09): RATIFIED.** `AC-001`'s "3.1 or 3.2" fork closes
as: **MUST publish an OpenAPI document, version 3.1 or — only where the
team's full toolchain (parser, linter, generator, docs renderer) is
verified against it — 3.2.** 3.1 is the unconditional default; a verified
3.2 toolchain makes a 3.2 document fully compliant, not an exception. The JSON Schema 2020-12 dialect pin stands
(strengthened by `baseline-02b`). The recorded re-check triggers
(swagger-parser #2248, openapi-generator #22728) flip the default when
they close.

**Classification:** evidence-backed default.
**Justification:** `[FACT]` (`baseline-02b`) swagger-parser, Redoc, and
openapi-generator carry open unaddressed 3.2 issues; **Spectral silently
ignores 3.2 constructs** — a lint pass that validates nothing; only
Redocly CLI has full support. **Declined:** flat 3.1 (blocks verified 3.2
toolchains) · free choice (invites the silent-lint failure).
**Confidence:** high (floor) · moderate (conditional clause).
**Evidence:** `baseline-02` §7 (AC-001 row) · `baseline-02b`.

---

## Pagination links (companion to AC-013/AC-014) — body envelope only

**Decision (2026-08-09): RATIFIED.** Pagination state lives **only** in the
ratified `AC-014` body envelope (items + continuation state). The standard
does **not** emit RFC 8288 `Link` headers for pagination.

**Classification:** project policy.
**Justification:** one source of truth — dual emission creates two places a
cursor can live, which drift under maintenance. `[COMPARATIVE]` Zalando
forbids the `Link` header with JSON media types (links in the payload);
GitHub's header-driven model is the counter-practice, declined.
**Confidence: moderate-high.**
**Evidence:** `survey-03` (Zalando `Link`-header prohibition) · `survey-04`
(pagination practice) · `baseline-02` §8.4 (listed as the open policy
item).

---

## Field selection

**Decision (2026-08-09): RATIFIED.** Field selection (sparse fieldsets) is
**MAY** — fixed response shapes are a legitimate contract (the
Stripe/GitHub/Twilio/AWS position). When an API offers it, the syntax is
fixed: a `fields` query parameter taking a comma-separated list of
snake_case field names. No OData `$select`, no JSON:API per-type brackets.
Expansion/embedding is a separate mechanism, not settled here.

**Classification:** project policy.
**Declined:** MUST-on-collections (mandates burden the field majority
declines) · forbidding it (blocks bandwidth optimization for large
resources). **Confidence: moderate.**
**Evidence:** `survey-04` finding 5 and field-selection table.

---

## Filter grammar (companion to AC-015)

**Decision (2026-08-09): RATIFIED.** Collection list endpoints filter via
**per-field equality parameters plus bracket range operators**
(`field[gte]`, `field[gt]`, `field[lte]`, `field[lt]`), combined AND-only.
A structured query DSL is permitted **only** as a separately-documented
search endpoint, never mixed into collection listing. `AC-015`'s ban on
exposing storage-engine syntax stands beneath both surfaces.

**Grammar note (reconciles with `AC-007`):** the bracketed operator form is
an explicit, enumerated extension of the ratified query-parameter grammar —
the base name matches `^[a-z][_a-z0-9]*$`, and the suffix is restricted to
exactly `[gte]`, `[gt]`, `[lte]`, `[lt]`. No other bracketed forms are
permitted; `AC-007`'s pattern governs everywhere else.

**Classification:** project policy.
**Justification:** `[COMPARATIVE]` the two families split the field
(per-field: Stripe lists, Shopify, Twilio, AWS · DSL: OData `$filter`,
AIP-160, GitHub `q=`); Zalando: "Simple query languages are generally
preferred over complex ones." The strongest datapoint is Stripe's own
architecture: simple params on lists, the DSL quarantined in a separate
rate-limited, eventually-consistent Search endpoint — the families were
never mixed on one surface.
**Declined:** DSL-on-lists (parser/injection surface, Zalando-warned
complexity) · equality-only (forces a search endpoint for date ranges, the
most common filter). **Confidence: moderate-high.**
**Evidence:** `survey-04` finding 4, Stripe section (list params vs Search
API) · `baseline-02` §8.4.

---

## AC-017 (completed) — Idempotency-key retention: ≥24 hours

**Decision (2026-08-09): RATIFIED.** The stated retention window `AC-017`
requires is set at **≥24 hours**, a floor. Fingerprint semantics per the
ratified `AC-016`.

**Classification:** project policy, on a strongly converged comparative
base.
**Justification:** `[COMPARATIVE]` Stripe ≥24h (prunable), Shopify 24h,
Zalando's guideline example 24h, AWS ≥24h; Google Cloud Deploy's 60
minutes is the short outlier (`baseline-02g`). **Declined:** ≥7 days (no
vendor commits to it) · ≥1 hour (loses overnight-backoff protection).
**Confidence: high — the field's number.**
**Evidence:** `survey-05` idempotency table · `baseline-02g`.

---

## Batch ratification — the fifteen remaining AC principles

**Decision (2026-08-09): RATIFIED en bloc, as proposed** in `baseline-02`
§7, per the Gate C batch-confirm procedure. No principle was pulled out.

Covers: AC-002 (description document authoritative; changes gated on an
automated compatibility check) · AC-005 (never cite RFC 7807 — corroborated
as a live hygiene problem by `baseline-02c`/`02d` findings of stale
citations at Microsoft and Italy's AgID) · AC-006 (top-level JSON object,
never a bare array) · AC-008 (IDs as strings) · AC-009 (RFC 3339
timestamps with explicit offset) · AC-010 (no floating-point money — the
ratified minor-unit-integer encoding rides on it) · AC-011 (null vs
omission explicit in PATCH) · AC-012 (tolerant enum readers; additions
non-breaking) · AC-013 (opaque non-constructable cursors, SHOULD) ·
AC-014 (collection envelope — the ratified body-only pagination rides on
it) · AC-015 (no storage syntax in filters — the ratified filter grammar
rides on it) · AC-018 (bulk atomic-vs-partial explicit, per-item outcomes)
· AC-019 (202 returns an addressable operation resource with terminal
states, expiry, failure representation) · AC-020 (RFC 7240 `Prefer`, MAY)
· AC-021 (RFC 6570 URI Templates, SHOULD).

**Classification:** per each row's evidence class in `baseline-02` §7.
**Evidence:** `baseline-02` §7 (per-row citations).
