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
