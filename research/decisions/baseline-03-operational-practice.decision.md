# Decisions — baseline-03 (operational practice)

*Gate C ratification record. One entry per decided principle; entries are
appended as the walkthrough proceeds. A `baseline` report proposes; only this
file ratifies. Classification per `PLAN.md` Phase 2: protocol requirement ·
evidence-backed default · project policy · exception · unresolved question.*

---

## OP-016 — Webhook signing: MUST sign; scheme selected by trust topology

**Decision (2026-08-09): RATIFIED as amended — the original single-preference
rule is replaced by a topology split.** Decided by the project owner in the
Gate C walkthrough after a purpose-validation leaf (`baseline-03c`) and an
adoption leaf (`baseline-03d`) commissioned when the owner asked to step back
to first principles ("what is this for, and what are newer implementations
doing?").

**Classification:** evidence-backed default (the signing MUST and the signed
base); project policy `[POLICY]` (naming the concrete schemes — Standard
Webhooks is a vendor-TSC specification, not a standards-body product).

### The ratified rule

Every outbound webhook **MUST** be signed over a base that binds a unique
delivery ID, a timestamp, the raw body, **and any metadata the consumer is
expected to act on**. The scheme is selected by trust topology:

- **Shared-secret topology** (the ordinary product webhook — provider issues
  each consumer a secret): use the **Standard Webhooks** scheme —
  HMAC-SHA256 over `id.timestamp.payload`, carried in `webhook-id`,
  `webhook-timestamp`, `webhook-signature` headers, `whsec_` secrets.
- **Cross-organization topology** (consumers cannot hold a shared secret, or
  the provider requires single key custody, HSMs, or non-repudiation): use
  **RFC 9421 HTTP Message Signatures with RFC 9530 `Content-Digest` as a
  covered component** and keys published at a discoverable key set,
  retaining `webhook-id`/`webhook-timestamp` as the delivery envelope.
- Bespoke per-vendor HMAC schemes are **dropped from the rule** — every
  property they offer, the Standard Webhooks scheme offers with interop.
- **SHA-1 is prohibited** — rationale: NIST retires SHA-1 for all
  applications by 2030-12-31 and SHA-256 costs nothing more. (Explicitly
  *not* "collisions break HMAC-SHA1" — RFC 6194 §3.3: they do not.)
- **Consumer-side obligations attach to the rule** (placement vs OP-017 /
  OP-024 to be settled in Phase 3 drafting): constant-time comparison;
  enforced, bounded, non-zero timestamp tolerance (~5 min convergent
  default); dedupe on the signed delivery ID for at least the tolerance
  window; fail closed on a missing/empty/default secret at configuration
  load.
- **Warning note:** Standard Webhooks' `v1a`/ed25519 asymmetric mode is safe
  only with a distinct key pair per endpoint — with one shared signing key a
  tenant can register a victim's URL and obtain valid signatures for
  messages it controls (documented by the spec's own lead author, issue #34,
  open since 2023-12-10). Any asymmetric requirement routes to the RFC 9421
  branch.

### Invariants (from `baseline-03c` §6, ratified with the rule)

I1 raw-byte base · I2 signed timestamp with enforced non-zero tolerance ·
I3 signed unique delivery ID + dedupe · I4 sign everything the consumer acts
on · I5 constant-time compare · I6 per-endpoint secrets ≥256 bits with
overlap rotation · I7 fail closed on empty secret · I8 SHA-1 prohibited ·
I9 reject unknown/legacy schemes · I10 verification as the default path
(SDK helper, test vectors, negative vectors) · I11 RFC 9421 ⇒ RFC 9530
covered component · I12 state the boundaries (authn, not authz; not a TLS
substitute) · I13 HTTPS-only delivery.

### Justification

1. **Purpose validated** (`baseline-03c`): authenticity + integrity from
   every primary source; GitHub adds an operational purpose. All six
   documented failures (CVEs + disclosures) are receiver-side verification
   failures, zero cryptanalytic — hence the consumer obligations and the
   verification-ergonomics invariant carry as much weight as scheme choice.
2. **The topology split is what the field did, 2024–2026**
   (`baseline-03d`): ~9 verified independent full-spec Standard Webhooks
   emitters (OpenAI, Anthropic, Google Gemini, Replicate, Etsy, Supabase,
   Polar, Dodo, Deck) + 4 via Svix; every major LLM-platform webhook launch
   of 2025–2026 chose it. Meanwhile UCP **MUST**s RFC 9421 for webhooks and
   AdCP makes it baseline-required with HMAC deleted in 4.0 — both
   cross-organization protocols where no shared secret is possible.
3. **The schemes compose**: UCP keeps the `webhook-id`/`webhook-timestamp`
   envelope under 9421 signature headers; AdCP's 3.x→4.0 transition is the
   shipped migration playbook. Standard Webhooks today is not a dead end.

**Confidence:** high (signing MUST + signed base) · moderate-high (Standard
Webhooks as shared-secret default — discounted for frozen spec, vendor TSC,
six of twelve README adopters failing verification, Svix's branded default
headers) · moderate (9421 cross-org branch — two protocols and one fintech,
all under ~12 months old).

**Honesty note carried into the record:** no documented in-the-wild *replay*
incident was located; the signed-timestamp requirement rests on
defense-in-depth reasoning and unanimous vendor/spec practice, and the
standard should say so rather than overstate.

**Fork honesty — options offered and declined at ratification:**
invariants-only with SW as a cited example (declined: softer conformance
review; the interop win of naming the scheme outweighs vendor-TSC
discomfort, which the `[POLICY]` label records); original 9421-preference
with the 9530 fix (declined: zero product-webhook 9421 deployments verified;
contradicts the adoption data); Standard-Webhooks-only (declined: leaves
cross-org delivery with no sanctioned answer given SW's documented
asymmetric hazard).

### Corrections to prior repo positions (annotated in place)

- `baseline-03b`: Web Bot Auth cannot be a leading indicator for webhook
  signing — the IETF webbotauth charter explicitly excludes HTTP APIs and
  agent-to-agent interfaces. Scope-cap note added 2026-08-09.
- `baseline-03` §8.1 / `survey-07`: "no adoption of RFC 9421" is true of the
  eight legacy references but no longer true of the field (UCP, AdCP,
  Qerko). `survey-07`'s Standard Webhooks adopter list corrected to cite
  `baseline-03d`'s verified table.

### Dated re-check triggers

- **AdCP 4.0 release** — HMAC removal completes; confirms or falsifies the
  cross-org 9421 trend.
- **Standard Webhooks issue #34 resolution / spec v1.1** — a fixed
  asymmetric mode would soften the warning note.
- **2026-11-15** — `draft-knauer-secure-webhook-token-02` expires (only IETF
  webhook-security draft; individual submission, no WG).
- **Any UCP revision after 2026-04-08** — resolves its header-table vs
  example discrepancy (recorded in `baseline-03d`).

**Evidence:** `baseline-03` §7 (OP-016 row) · `baseline-03b` (as annotated) ·
`baseline-03c` · `baseline-03d` · `survey-07` (as corrected).

---

## OP-010 — Rate limits: MUST published mechanism; SHOULD pinned draft fields

**Decision (2026-08-09): RATIFIED as re-framed — the original MUST on the
draft fields is replaced by rule shape (b).** Decided after two
commissioned leaves (`baseline-03e` field survey, `baseline-03f` draft
trajectory) when the owner asked for industry-adoption research including
OpenAI, Anthropic, and Gemini. **Absorbs pile item 22** (proprietary
headers alongside IETF fields).

**Classification:** protocol requirement (429 per RFC 6585 + `Retry-After`
per RFC 9110, with our MUST a deliberate tightening of RFC 6585 §4's MAY —
house policy over a *published* standard); project policy `[POLICY]` (the
SHOULD on unpublished draft fields).

### The ratified rule

1. **MUST** apply rate limits and communicate exhaustion via the published
   HTTP mechanism: return `429` with `Retry-After` (cross-reference
   `OP-011`; do not restate it).
2. **SHOULD** additionally advertise quota state using `RateLimit` and
   `RateLimit-Policy` **in the syntax of
   `draft-ietf-httpapi-ratelimit-headers-11`**. `[POLICY]` These are an
   unpublished Internet-Draft, not a standard; they MUST NOT be described
   as standards-compliant, and the pinned revision MUST be cited wherever
   referenced.
3. **MUST** document any proprietary quota headers explicitly, including —
   for any reset field — whether the value is an **absolute epoch
   timestamp or a delta in seconds** (the one documented ambiguity that
   breaks real clients: GitHub and Zalando define the same header name
   with opposite meanings). This clause absorbs item 22: proprietary and
   draft fields MAY coexist; whatever is emitted is documented.

### Why the original MUST fell

Four independent grounds (`baseline-03f`), any one sufficient: RFC 2026
§2.2 prohibits claiming compliance with an Internet-Draft; the wire format
is moving (editors' open PR #166 renames `r`→`a`, `t`→`w`); the draft
itself only says MAY emit; neither field is IANA-registered even
provisionally. Ecosystem measurement (`baseline-03e`/`03f`): three
incompatible wire formats shipped across the draft's life; every wild IETF
citation implements the superseded trio (draft-06 or earlier); the current
format has exactly two emitters (one off-by-default library option, one
0-star crate); the reference parser from the same team cannot parse it;
Atlassian ships it behind a non-conforming `Beta-` prefix.

### Contingency — re-keyed off expiry

The draft has expired and revived **three times** (max dormancy ~15.5
months); RFC 2026 §2.2 restarts the clock on any revision, so the previous
2026-11-24 expiry trigger would false-fire. Replaced by three triggers:

| Trigger | Condition | Action |
|---|---|---|
| **Upgrade** | `RateLimit` appears in the IANA HTTP Field Name Registry as `permanent` (`curl -s https://www.iana.org/assignments/http-fields/field-names.csv \| grep -i ratelimit`) | SHOULD → MUST; drop `[POLICY]` |
| **Re-pin** | A new revision changes wire syntax (watch PR #166, issue #158) | Update the pin; drop to MAY if the format churns again pre-publication |
| **Withdraw** | 18 months with no revision and no WGLC, or the draft leaves the WG's active list unrevived, or the WG concludes | Drop the SHOULD; keep 429 + `Retry-After` + documented-proprietary |

Review cadence semi-annual; next **2027-02-09**. 2026-11-24 is only a
check for whether a draft-12 appeared.

**Confidence:** moderate-high (mandate clause) · moderate (SHOULD clause).

**Options declined:** original MUST-with-expiry-contingency (conflicts
with RFC 2026; mandates a two-emitter format; false-firing trigger) ·
`Retry-After`-only (forfeits the self-upgrading path).

### Corrections to prior repo positions (annotated in place)

- `baseline-03b`'s "editorial objection → format probably stable"
  inference falsified by the editors' own review response; struck through
  with a dated note.
- `baseline-03e`'s closing endorsement of the 2026-11-24 trigger is
  overridden by `baseline-03f`'s revival-mechanics finding (conflict
  surfaced in `03f`, not averaged).

**Evidence:** `baseline-03` §7 (OP-010 row) · `baseline-03b` (as
annotated) · `baseline-03e` · `baseline-03f`.

---

## OP-015 (completed) — Version marker: major version in path

**Decision (2026-08-09): RATIFIED.** `OP-015` (one uniform version-marker
placement — already proposed as MUST, both `survey-06` runs concurring) is
completed with the concrete choice: **major version in the path** (`/v1`),
AIP-style — never `/v1.0`; minor and patch versions never appear in URIs;
evolution within a major is additive (compatible-evolution-first); a major
bump is a rare last resort (AIP-181's "extreme course of action" posture).
Deprecation and sunset signaling ride `OP-013`/`OP-014` (RFC 9745 /
RFC 8594); window lengths remain a separate open item.

**Classification:** project policy `[POLICY]` — the placement choice has no
standards authority; `survey-06` documents four live models.

**Justification:** `[COMPARATIVE]` path-major is the only model requiring
zero client discipline and zero server pinning infrastructure, and it is
where AIP, Twilio, Shopify, and the field majority landed. The
dated-header model (GitHub `X-GitHub-Api-Version`; Stripe account-pinning
with 72-hour rollback) is stronger engineering for continuous evolution
but assumes infrastructure ordinary teams will not build. `[FACT]`
Zalando's "MUST NOT use URI versioning" is noted as the principled
counter-position; the BCP 190 concern is satisfied by the house-standard
scope reading (a standard constraining its own URI space — position to be
ratified as pile addition C).

**Options declined:** dated request header (best-in-class but
infrastructure-heavy) · required query parameter (Azure; uniformly noisy) ·
media-type/none (Zalando purism; weakest tooling, hardest operations).

**Confidence: moderate** — a genuine fork; the dated-header model is the
strongest alternative and a team with Stripe-grade pinning infrastructure
could reasonably invert this.

**Evidence:** `survey-06` (both runs: versioning-transport table,
Stripe/GitHub/AIP/Azure/Twilio/Shopify models) · `baseline-03` §7 (OP-015
row).

---

## Deprecation window and version overlap

**Decision (2026-08-09): RATIFIED.** A deprecated **major** version
remains fully supported for **≥12 months** after its successor ships, and
the sunset date is announced the day deprecation starts. Signaling
mechanics per the already-proposed `OP-013`/`OP-014` (`Deprecation` +
`Sunset` headers, RFC 9745/8594; migration-doc link; never deprecate
without a sunset date).

**Classification:** project policy.
**Justification:** `[COMPARATIVE]` GitHub commits ≥24 months; Shopify runs
≥12 months per version with ≥9 months overlap — the tightest surveyed for
a whole major; Stripe's honor-everything-forever depends on
account-pinning infrastructure this standard does not assume. The floor
permits APIs to promise more (GitHub's 24). **Declined:** ≥24 months as
the floor (doubles double-run burden for every break) · ≥6 months (no
surveyed vendor runs a major that short). **Confidence: moderate.**
**Evidence:** `survey-06` (support windows) · `baseline-03` §8.3.

---

## Webhook retry and dead-letter policy

**Decision (2026-08-09): RATIFIED.** Failed webhook deliveries are retried
with exponential backoff for **≥72 hours**; after retries exhaust, the
delivery is held in a **dead-letter store for ≥30 days** with a
self-service **redelivery API**.

**Classification:** project policy.
**Justification:** `[COMPARATIVE]` Stripe retries ~3 days and separately
retains 30 days of events for reconciliation — the only surveyed model
giving consumers self-service recovery; Shopify retries 8 times over 4
hours then deletes the subscription (aggressive outlier); GitHub never
auto-retries (manual redelivery ≤3 days). **Declined:** short-retry +
ledger (drops day-long-outage consumers to manual recovery) · retries-only
(an outage longer than the window loses data permanently).
**Confidence: moderate.**
**Evidence:** `survey-07` (retry policies) · `survey-05` · `baseline-03`
§8.3 (dead-letter retention listed as org policy).

---

## Support tiers

**Decision (2026-08-09): RATIFIED — AIP-style, explicit in the path.**
Three stability tiers, marked **inside the version path segment**:
`/v1alpha…` (experimental — no stability promise, may change or vanish
without notice, excluded from the deprecation policy, MAY be
access-gated), `/v1beta…` (preview — best-effort stability, breaking
changes with notice, outside the 12-month floor), `/v1` (GA — the full
ratified deprecation policy applies). Graduation from a pre-GA tier to GA
is an explicit client migration, deliberately: the stability contract
changed.

**Classification:** project policy, on AIP-185's channel convention.
**Justification:** the tier marker reuses the already-ratified
major-version path segment — one mechanism, visible at every call site,
log line, and code review; a docs-only tier is invisible exactly where the
risk is taken. **Declined:** docs-only tiers · header-carried tiers.
**Confidence: moderate-high.**
**Evidence:** `survey-06` (AIP-185 channels, Google section) ·
`baseline-03` §8.3.

---

## Auth mechanism per client class

**Decision (2026-08-09): RATIFIED — conditional by client class; the
boundary is authority, not preference.** Both mechanisms are supported:
**OAuth/OIDC is REQUIRED** wherever a user delegates authority or a third
party acts on a user's behalf (an API key cannot carry scoped, revocable,
per-user consent — it authenticates a caller, not a delegation). **API
keys are acceptable** for server-to-server traffic with a single trust
relationship. Wherever OAuth is used, the BCP 240 rules already proposed
as `OP-003`/`OP-004` apply (no password grant, PKCE for public clients,
exact redirect-URI matching, no implicit grant).

**Classification:** evidence-backed default (the OAuth-side rules are
BCP 240, a published Best Current Practice); project policy (the
client-class boundary itself).
**Justification:** `[FACT]` RFC 9700 governs OAuth and is silent on API
keys; `[COMPARATIVE]` seven of eight surveyed vendors ship key-based auth —
keys are a legitimate mechanism, not a legacy one; neither mechanism is
universally correct (`baseline-03` settled-axes table, High confidence).
**Declined:** OAuth-everywhere (ceremony without security in single-trust
paths) · leave-per-API (someone ships a key where user authority is
exercised). **Confidence: high.**
**Evidence:** `baseline-03` settled-axes row "OAuth/OIDC vs API keys",
OP-003/OP-004 rows · RFC 9700.

---

## Deployment profile — the five risk-based security axes

**Decision (2026-08-09): RATIFIED — defaults plus flip triggers, as
recommended by `baseline-03g`.** The owner elected to research and decide
these now (rather than defer), housed as a deployment profile: each axis
carries a ratified **default** and the named **threat-model triggers**
that flip it. The full trigger tables and evidence live in `baseline-03g`;
the skeleton:

| Axis | Default | Flip triggers (headline) |
| --- | --- | --- |
| Sender-constrained tokens | **Bearer over TLS** + short TTL + audience restriction + refresh-token rotation for public clients; validation SHOULD NOT hard-code the `Bearer` scheme | FAPI 2.0 / open banking → DPoP or mTLS required; tokens visible to logging intermediaries; public clients in hostile environments (DPoP); existing PKI server-to-server (mTLS); per-operation value → RFC 9470 step-up, not a global flip |
| Token format | **Opaque on the public wire**; phantom-token where a gateway exists (integration work — only Kong documents it natively); a client-issued JWT MUST be RFC 9068-conformant + paired with a revocation-propagation plan | Measured introspection bottleneck / AS-outage tolerance / third-party resource servers → JWT; instant-revocation SLA or claims carrying PII → stay opaque |
| Rate-limit aggressiveness | **Multi-dimensional tiered posture, published**: per-principal sustained + token-bucket burst (start ≈100 rps/account, 25 rps/endpoint, `[POLICY]` numbers); unauthenticated per-IP an order of magnitude lower; auth endpoints strictly stricter (start ≤5/min per IP+account); failed-auth budget; concurrency separate | >10× per-request cost variance → cost/token accounting; metered third-party spend → spend caps; credential-stuffing exposure → lockout tier; multi-tenant → fair-share; free-tier abuse → spend/tenure gating |
| Replay window | **300 s past / 60 s future, asymmetric** + mandatory dedup cache (`jti`/nonce/message-ID) held ≥ the past window; NTP required; the window alone is never sufficient. Does not reopen the ratified webhook 5-min convention | Interactive high-value signing → 30–60 s; server-provided nonces remove skew entirely; unmanaged clocks / store-and-forward → up to 15 min, never without dedup; signature omits body → add RFC 9530 binding |
| Object-level authorization | **Centralize the decision, enforce in the handler** — one shared component invoked at every call site resolving a client-supplied ID; deny-by-default; **never gateway-enforced**; authz tests block deployment | Relationship-derived permissions / list-visibility at scale / cross-tenant sharing → Zanzibar-style ReBAC; regulated audit or polyglot fleet → policy language (Cedar/Rego); single service + ownership column → stay embedded |

**Classification:** project policy throughout, grounded in published
standards (BCP 240, RFC 9449/8705/9068/7009, FAPI 2.0, OWASP API Top 10
2023) and verified vendor practice including the three AI platforms.

**Key evidence lines (full detail in `baseline-03g`):** BCP 240's
access-token sender-constraining is SHOULD with a deployment escape hatch,
and its only MUST is satisfiable by rotation; Microsoft Entra ships
neither DPoP nor RFC 8705 today and the MCP spec mandates plain `Bearer`;
Cognito documents that revoked JWTs still verify; Entra CAE documents
≤15-min revocation lag; OWASP API1:2023 requires the check "in every
function" and never says "centralized"; FAPI 2.0 supplies the reasoned
+10 s/+60 s future-skew bounds.

**Dated re-check triggers (added to the register):** Entra shipping mTLS
PoP ("Future APIs will rely on PoP via mTLS") — highest-value watch item ·
any MCP proof-of-possession extension · OpenFGA CNCF graduation ·
FAPI 2.0 adoption outside finance.

**Evidence:** `baseline-03g` throughout · `baseline-03` §8.2.

---

## Batch ratification — the 22 remaining OP principles

**Decision (2026-08-09): RATIFIED en bloc, as proposed** in `baseline-03`
§7, per the Gate C batch-confirm procedure. No principle was pulled out.
This closes the last open item in the Gate C pile.

Covers: transport & credentials (OP-001 TLS 1.2+/prefer 1.3 · OP-002 no
tokens in query strings · OP-003/004 BCP 240 OAuth rules) · authorization
(OP-005 object-level on every request — the five-axes profile's in-handler
enforcement rides on it · OP-006 unguessability is not access control ·
OP-007 per-caller property authorization · OP-008 writable-field
allow-lists) · capacity & retries (OP-009 published page/expansion/bulk
maxima · OP-011 429-quota vs 503-overload with `Retry-After` — the
ratified OP-010 cross-references it · OP-012 documented retryability,
backoff with jitter, idempotency key before retrying non-idempotent
requests) · deprecation signaling (OP-013/014 `Deprecation` + `Sunset` +
migration link — the 12-month window decision rides on them) · webhook
consumer obligations (OP-017 raw-body-before-parse verification, bounded
window, dedupe — invariants I1–I3 · OP-021 at-least-once/no-ordering +
monotonic version · OP-022 ack-before-processing with published timeout ·
OP-024 overlapping signing secrets — invariant I6) · observability &
hygiene (OP-018 correlation ID on every response · OP-019 W3C
`traceparent`/capped `tracestate` · OP-020 no internal detail in
responses · OP-023 SSRF guard on caller-supplied URLs · OP-025 scoped
expiring credentials, SHOULD).

**Classification:** per each row's evidence class in `baseline-03` §7.
**Evidence:** `baseline-03` §7 (per-row citations).
