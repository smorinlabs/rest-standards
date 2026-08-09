# Baseline 03d — Webhook Signing: What Post-2023 Implementations Actually Use

*Narrow leaf under `baseline-03`, companion to `baseline-03c` (threat model).
Verifies Standard Webhooks governance and adoption against each adopter's own
docs, locates RFC 9421 webhook deployments, and derives the recommended shape
for `OP-016`. Run 2026-08-09.*

All retrievals **2026-08-09** unless a different date is stated. Labels follow repo
convention: `[FACT]` sourced, `[COMPARATIVE]` cross-vendor pattern, `[INFERENCE]`
reasoning, `[OPINION]` judgment.

---

## Headline

`[INFERENCE]` The evidence does not support the current OP-016 precedence ("Prefer
RFC 9421; otherwise HMAC-SHA256..."), and it does not support inverting it into
"RFC 9421 has no webhook deployment" either. It supports **splitting the rule by trust
topology**, because that is the split the market actually made in 2024–2026:

| Topology | What shipped | Evidence |
|---|---|---|
| One provider → its own signed-up customers, shared secret is available | Standard Webhooks scheme: HMAC-SHA256 over `id.timestamp.payload`, signed timestamp | OpenAI, Anthropic, Gemini, Replicate, Polar, Supabase, Etsy, Dodo, Deck, Brex; all 3 webhook-as-a-service platforms |
| Many organizations → many organizations, no shared secret possible | RFC 9421 + published key sets (JWKS / `.well-known`) | UCP (`MUST`), AdCP (baseline-required, HMAC removed in 4.0), Qerko (optional) |

`[FACT]` Zero of the mainstream product-webhook providers or webhook platforms surveyed
mention RFC 9421 anywhere. `[FACT]` Both 2026 cross-organization agentic protocols
mandate it and neither uses Standard Webhooks for the signature.

---

## Q1 — Standard Webhooks: governance and adoption depth

### Governance `[FACT]` (GitHub REST API + repo files, 2026-08-09)

| Property | Value |
|---|---|
| Repo | `standard-webhooks/standard-webhooks`, created 2023-08-27, not archived |
| License | Apache-2.0 |
| Spec version | **1.0.0** — the string `Version: 1.0.0` in `spec/standard-webhooks.md` |
| Spec last substantively edited | **2025-02-16** (`Add timetamps formating details ISO_8601 (#237)` — typos in original commit title); previous edit 2024-04-12 |
| Repo activity | Active: last push 2026-08-05; 1,722 stars, 66 forks, 57 open issues |
| Tags/releases | `v1.0.0` 2025-12-31, `v1.0.1`+`v1.0.2` 2026-02-18 |
| Standards body | **None.** A Technical Steering Committee of eight individuals |

`[FACT]` The tags are **library** releases, not spec releases — the `v1.0.0` release is
titled verbatim *"Version 1.0.0: initial library release"*. `[INFERENCE]` The spec text
has been static for ~18 months while 2026 commits are packaging and dependency
maintenance (Python 1.1.0 bump 2026-07-21, Haskell implementation link 2026-08-05, C#
trusted publishing, dependabot). The project is maintained; the specification is frozen.

`[FACT]` TSC members (README, verbatim list): Brian Cooksey (Zapier), Ivan Gracia
(Twilio), Jorge Vivas (Lob), Matthew McClure (Mux), Nijiko Yonskai (ngrok), Stojan
Dimitrovski (Supabase), Tom Hacohen (Svix), Vincent Le Goff (Kong).

### Governance defect: the asymmetric mode is known-misuse-prone and unfixed `[FACT]`

Issue **#34, "Updating the asymmetric signature recommendations"**, opened by Tom Hacohen
(Svix founder, TSC) **2023-12-10**, still **open**, last comment **2025-11-01**. Hacohen's
own text describes the shipped `v1a,`/ed25519 mode as unsafe when a provider uses one key
pair, verbatim:

> "The main limitation with the current recommendation is that it's unclear that for it
> to be secure it requires a different key per endpoint, even for the asymmetric case.
> This means that people can easily misuse the recommendations (and have security
> issues)"

and the attack, verbatim:

> "Because different customers of the same service will be able to trick the service into
> sending messages to another customer endpoint, and that customer endpoint will trust it
> because it's coming from the service."

`[INFERENCE]` The spec's asymmetric option has carried a documented misuse hazard,
acknowledged by its own lead author, for **2 years 8 months** with no resolution. It is
usable only by a provider that already knows to issue a key pair per endpoint, and it
cannot be cited as a general asymmetric answer.

### The README adopter claim, verified against each vendor's own docs

`[FACT]` README sentence under test, verbatim: *"Standard Webhooks has been adopted by a
variety of companies including: OpenAI, Anthropic, Google Gemini, Kong, Svix, Supabase,
Vanta, Drata, Etsy, PagerDuty, Twilio, TaskRabbit and many others."*

**Adopter table.** "Full-spec" = lowercase `webhook-*` headers + HMAC-SHA256 over
`id.timestamp.payload`, base64, `v1,` prefix. "Via Svix" = delivery is Svix
infrastructure (branded `svix-*` headers), i.e. one vendor's platform, not an independent
implementation choice.

#### A. Verified full-spec, independent implementation

| Provider | First shipped | Names the spec? | Headers | Signature base | Secret | Source |
|---|---|---|---|---|---|---|
| **OpenAI** | 2025-06-24 (own changelog) | **Yes**, links it | `webhook-id`,`webhook-timestamp`,`webhook-signature` | (by reference) | `whsec_` | developers.openai.com/api/docs/guides/webhooks |
| **Google Gemini** (static) | ~2026-05; page updated 2026-08-04 | **Yes**, explicitly | `webhook-id`,`webhook-timestamp`,`webhook-signature` | spec base | signing secret | ai.google.dev/gemini-api/docs/webhooks |
| **Polar** | not dated in docs | **Yes**, links standardwebhooks.com | `webhook-*` | spec base | base64 | polar.sh/docs/integrate/webhooks/delivery |
| **Supabase** (Auth Hooks) | not dated in docs | **Yes**, explicitly | `webhook-*` | spec base | `v1,whsec_<base64>` | supabase.com/docs/guides/auth/auth-hooks |
| **Dodo Payments** | not dated in docs | **Yes**, uses `standardwebhooks` lib | `webhook-*` | spec base | — | docs.dodopayments.com |
| **Deck** (deck.co) | HMAC signing 2025-05; v2 2026-03 | **Yes**, explicitly | `webhook-*` | spec base | — | docs.deck.co/events/webhook-best-practices |
| **Anthropic** | 2026-05-06 (own release notes) | **No** — never names it | `webhook-id`,`webhook-timestamp`,`webhook-signature` | via SDK `unwrap()` | `whsec_` (32-byte) | platform.claude.com/docs/en/managed-agents/webhooks |
| **Replicate** | not dated in docs | **No** | `webhook-*` | `id.timestamp.data`, stated explicitly | `whsec_` | replicate.com/docs/topics/webhooks/verify-webhook |
| **Etsy** | not dated in docs | **No** | `webhook-*` | `webhook-id + "." + webhook-timestamp + "." + raw_body` | `whsec_` | developers.etsy.com/documentation/essentials/webhooks |

Verbatim, OpenAI: *"Webhooks are delivered to an HTTP endpoint you control, following the
Standard Webhooks specification."*
Verbatim, Gemini: *"Gemini strictly follows the Standard Webhooks specification for
security headers."*
Verbatim, Anthropic: *"Every delivery carries the `webhook-id`, `webhook-timestamp`, and
`webhook-signature` headers. Use the SDK's `unwrap()` helper to verify the signature and
parse the event in one step. It throws if the signature is invalid or the payload is more
than 5 minutes old."*
Verbatim, Replicate: *"The content to sign is composed by concatenating the `id`,
`timestamp`, and `data`, separated by the full-stop character (`.`)"*
Verbatim, Deck: *"Deck signs every webhook delivery using the Standard Webhooks
specification."*

`[INFERENCE]` Anthropic, Replicate and Etsy implement the scheme exactly but never cite
it. That is adoption of the *design*, and it also means a third party cannot tell
adoption from independent convergence without reading the byte-level details — which is
why the README list needs the checking it gets here.

#### B. Full-spec scheme, but delivered by Svix (counted separately)

| Provider | Headers | Source |
|---|---|---|
| Resend | `svix-id`,`svix-timestamp`,`svix-signature` | resend.com/docs |
| Clerk | Svix headers; docs say it *"wraps the standardwebhooks library"* | clerk.com/docs |
| Vanta | `svix-*`; verbatim *"Webhooks are powered by Svix"* | developer.vanta.com/docs/webhooks |
| TaskRabbit | Svix headers; *"delivered via Svix… configured in the Svix Partner Dashboard"* | developer.taskrabbit.com/docs/webhooks-1 |

#### C. Variants — same shape, non-conformant details

| Provider | What differs | Source |
|---|---|---|
| **Brex** | `Webhook-Id`/`Webhook-Timestamp`/`Webhook-Signature` (Title-Case, not spec lowercase); no `whsec_`; sample tolerance 60 s vs spec-library 300 s; never names the spec | developer.brex.com/guides/webhooks |
| **Loops** | `webhook-*` headers but `LOOPS_SIGNING_SECRET`, not `whsec_` | loops.so/docs/webhooks |
| **Google Gemini (dynamic)** | `Webhook-Signature` carrying an **RS256 JWT**, verified against `generativelanguage.googleapis.com/.well-known/jwks.json` — not the HMAC scheme at all | ai.google.dev/gemini-api/docs/webhooks |
| **xAI** | Matching header names on the SIP voice-agent surface only; algorithm not stated on the page | docs.x.ai |

#### D. README claims that did NOT survive checking `[FACT]`

| Claimed adopter | What its own docs show |
|---|---|
| **Kong** | Its "Standard Webhooks" artifact is a **gateway plugin that validates inbound** webhooks its customers receive. Kong's own outbound Event Hooks use `x-kong-signature`. Tooling adopter, not an emitter. |
| **PagerDuty** | Bespoke `x-pagerduty-signature`, hex, `v1=` prefix, multi-signature rotation — not the spec. |
| **Twilio** | Event Streams docs give no signing specifics and link back to classic `X-Twilio-Signature` HMAC-**SHA1** over URL + sorted params. |
| **ngrok** | No outbound webhook product located at all; its signature docs cover **verifying inbound** third-party webhooks. |
| **Zapier** | No native outbound signature verification; Zapier community confirms the Webhooks trigger does not support it. |
| **Drata** | No locatable Drata-authored webhook-signature documentation. Unverifiable. |

`[INFERENCE]` *(Count corrected 2026-08-09 during Gate C review.)* Of the
**twelve named** README adopters: **five** verify as independent full-spec
emitters (OpenAI, Google Gemini, Supabase, Etsy; Anthropic implements the
exact shape unnamed); **three** are the platform or deliver via it (Svix
itself; Vanta and TaskRabbit via Svix — platform delivery, not an
independent implementation choice); and **four fail verification as
emitters** (Kong — inbound-validation tooling only; PagerDuty and Twilio —
bespoke schemes; Drata — strictly unverifiable, no locatable docs). ngrok
and Zapier also fail but are TSC-member companies, **not** on the named
list, and must not be counted against it. The attrition remains a
governance finding: the specification's own adoption claim does not survive
primary-source checking without reclassification, and downstream documents
that cite the README inherit the imprecision.

#### E. New adopters found beyond the README

Polar, Dodo Payments, Deck, Replicate, Loops, Brex, xAI (partial), Hookdeck Outpost.

`[COMPARATIVE]` Meanwhile the majority of dev-tool and fintech webhooks checked use
**unrelated bespoke HMAC**: Lob (`Lob-Signature` + `Lob-Signature-Timestamp`), Mux
(`mux-signature`), Cal.com, Linear, Vercel (HMAC-**SHA1**), Knock (Stripe-style but
milliseconds), WorkOS, Paddle, Lemon Squeezy, Notion, Airtable, Slack, LangSmith,
PagerDuty. And a distinct asymmetric/JWT cluster: Kinde (RS256 JWT body + JWKS), Plaid
(`Plaid-Verification`, ES256 JWT + JWKS), Neon Auth (Ed25519 JWS, explicitly its own
scheme), Gemini dynamic webhooks (RS256 + JWKS).

---

## Q2 — RFC 9421 for webhooks specifically

### Is anyone using it for outbound webhooks? Yes — three, all 2026, all cross-org `[FACT]`

**1. UCP — Universal Commerce Protocol** (`ucp.dev`, spec version **2026-04-08**;
org created 2025-11-13, repo `Universal-Commerce-Protocol/ucp` created 2025-12-31,
Apache-2.0, 3,286 stars, pushed 2026-08-09). Google publishes a UCP implementation guide
at `developers.google.com/merchant/ucp`. `[INFERENCE]` That establishes Google as an
implementer/documenter; the "UCP governing body" referenced in the spec is not named on
the pages retrieved, so **do not** describe UCP as a Google specification.

Verbatim normative text:
> "Webhook notifications MUST be signed."

> "MUST sign all webhook payloads per the Message Signatures specification using RFC 9421
> headers (`Signature`, `Signature-Input`, `Content-Digest`)"

> "MUST include `UCP-Agent` header with profile URL for signer identification"

> "All implementations MUST support verifying P-256 (`ES256`) signatures"

Cites RFC 9421, RFC 9530 (Content-Digest), RFC 7517 (JWK), RFC 8941. Key discovery via a
`signing_keys` array at `/.well-known/ucp`. Covered components `@method`, `@authority`,
`@path`, conditionally `@query`, `ucp-agent`, `idempotency-key`, `content-digest`,
`content-type`. Replay protection is at the business layer via idempotency keys, not in
the signature.

**2. AdCP — Ad Context Protocol** (`adcontextprotocol/adcp`, Apache-2.0, created
2025-07-19, latest release **v3.1.12 on 2026-08-08**). Governed by AgenticAdvertising.Org,
a pending 501(c)(6) Delaware trade association; interim board from Scope3, Celtra and
Triton Digital; first AGM 2026-05-06.

Verbatim (`docs/reference/whats-new-in-v3.mdx`):
> "RFC 9421 HTTP Message Signatures are optional in 3.0 and mandatory under AdCP
> Verified."

> "Webhooks are signed under the same RFC 9421 profile — baseline-required for sellers.
> Webhook authentication unifies on the AdCP 9421 profile as a symmetric variant of
> request signing: the seller signs outbound webhook requests with a key published in its
> JWKS at `jwks_uri` … HMAC-SHA256 remains a legacy fallback through 3.x … the entire
> `authentication` object is removed in 4.0."

Verbatim changeset `.changeset/property-list-webhook-rfc9421.md`:
> "Standardize property-list change notifications on the RFC 9421 webhook profile. Keep
> the undefined legacy body-level `signature` field as a required, deprecated
> compatibility marker through 3.x; remove it in 4.0."

Profile detail: Ed25519 or `ecdsa-p256-sha256`; sig-params `created`, `expires`
(≤ `created` + 300 s), `nonce` (≥128 bits base64url), `keyid`, `tag` exactly
`adcp/request-signing/v1`; published test vectors at
`static/compliance/source/test-vectors/request-signing/` including 16+ negative vectors
(`002-wrong-tag`, `003-expired-signature`, `004-window-too-long`, `016-replayed-nonce`, …).

`[INFERENCE]` AdCP is the only body found that is actively **migrating away from HMAC to
9421** for webhooks, with a dated removal (4.0). That is a stronger signal than a
greenfield choice.

**3. Qerko** — Czech-National-Bank-licensed payments fintech. Offers RFC 9421
(`Signature` + `Signature-Input`) alongside a legacy HMAC path (`Signature-SHA256`).
Source: `docs.qerko.com/docs/webhook-interface`. `[OPINION]` Small but a genuine
production fintech data point outside the agentic-protocol niche.

(Also found: Rundun, a two-person SaaS — recorded, given negligible weight.)

### Is there an IETF draft profiling 9421 for webhooks? No `[FACT]`

Datatracker API queries, 2026-08-09:
- `name__contains=message-signatures` → `draft-ietf-httpbis-message-signatures` (now
  RFC 9421), `draft-meunier-http-message-signatures-directory-05`,
  `draft-hoypat-httpbis-message-signatures-ekm-00`,
  `draft-richanna-http-message-signatures-00`. **None profiles webhooks.**
- `name__contains=webhook` → exactly one document: **`draft-knauer-secure-webhook-token-02`**,
  "Secure Webhook Token (SWT)", Stephan Knauer, revision 02 dated **2026-05-14**,
  Standards Track intent, **expires 2026-11-15**, **individual submission, adopted by no
  working group**. It is JWT-based and mentions neither RFC 9421 nor Standard Webhooks.

### Does Web Bot Auth extend to webhooks? No — the charter forecloses it `[FACT]`

IETF **webbotauth** WG, `charter-ietf-webbotauth-01` approved; chairs David Schinazi and
Rifaat Shekh-Yusef; Web and Internet Transport area.

In scope, verbatim: *"cryptographically authenticating access to Web sites for: Crawlers
for search indices, Web archivers…"*

Out of scope, verbatim: *"Authenticating access to content not intended for human
consumption (e.g., HTTP APIs, agent-to-agent interfaces)"*

`[INFERENCE]` This closes the question `baseline-03b` left open. Web Bot Auth will not
grow into a webhook profile — webhooks fall inside its explicit exclusion. `OP-016` must
stop treating Cloudflare's Web Bot Auth deployment as a leading indicator for webhook
signing. It remains valid evidence that RFC 9421 is implementable at scale, and nothing
more. The webhook evidence for 9421 is now UCP/AdCP/Qerko, which is independent of it.

### Consumer-side verification tooling `[FACT]`

No webhook-specific RFC 9421 tooling was found — only generic libraries
(`yaronf/httpsign`, `pyauth/http-message-signatures`, NSign) and, for the two protocols,
their own SDKs plus AdCP's test vectors. No API-gateway plugin verifies 9421 webhooks;
Kong ships one for Standard Webhooks (below).

---

## Q3 — What brand-new (2024–2026) implementations chose

### Dated first-launches, verified against the vendor's own changelog `[FACT]`

| Provider | First shipped | Chose |
|---|---|---|
| OpenAI webhooks | **2025-06-24** (changelog: *"Added support for async event handling with webhooks"*) | Standard Webhooks, named |
| Anthropic (Claude Managed Agents) | **2026-05-06** (release notes: *"Webhooks for Claude Managed Agents are now supported…"*) | Standard Webhooks shape, unnamed |
| Google Gemini | ~2026-05; page last updated 2026-08-04 | Standard Webhooks (static) + RS256 JWT/JWKS (dynamic) |
| ElevenLabs | **2025-03-03** | Bespoke `elevenlabs-signature` |
| Deck (deck.co) | HMAC securing called out 2025-05; v2 multi-destination 2026-03 | Standard Webhooks, named |
| Hookdeck Outpost | OSS **2025-05**; managed GA **2026-04-23** | "Standard Webhooks compatible" |
| UCP | spec **2026-04-08** | RFC 9421, `MUST` |
| AdCP 3.0 | v3.x through **2026-08-08** | RFC 9421, baseline-required; HMAC removed in 4.0 |

### Webhook-as-a-service platforms — the default-setters `[FACT]`

| Platform | Default emitted headers | Standard Webhooks? | RFC 9421? |
|---|---|---|---|
| **Svix** | `svix-id`, `svix-timestamp`, `svix-signature` | Is the reference implementation — but see below | **No mention** on `/security`, `/how`, `/how-manual` |
| **Hookdeck Event Gateway** | `x-hookdeck-signature`, `x-hookdeck-signature-2` (rollover), SHA-256 base64 | No | No |
| **Hookdeck Outpost** (separate OSS product) | — | *"Standard Webhooks compatible"* | No |
| **Convoy** | `X-Convoy-Signature`, name and hash configurable; simple + advanced (`t=`/`v0=`/`v1=`) modes | No | No |
| **Inngest** | pass-through, provider-specific | No | No |
| **Zapier** | none — no outbound signing at all | N/A | N/A |

**The Svix default is materially adverse to the interop argument** `[FACT]`. Verbatim from
`docs.svix.com/receiving/verifying-payloads/how-manual`:

> "Professional and Enterprise tier customers can have the headers white-labeled to use
> the `webhook-` prefix instead of the `svix-` prefix used above."

`[INFERENCE]` The spec's own originator and largest emitter sends **non-conformant header
names by default**, and conformant header names are a **paid feature**. The signature
computation is identical, so a verifier parameterized on header names works; a verifier
hard-coded to the spec's header names does not. This weakens — it does not destroy — the
"one verification function everywhere" claim.

`[FACT]` Convoy's repo `frain-dev/convoy` is actively committed (commits 2026-08-04
through 2026-08-07), so its non-adoption is a live choice, not abandonment.

---

## Q4 — Comparative assessment of the three candidates

| Dimension | (i) RFC 9421 | (ii) Standard Webhooks v1.0.0 | (iii) Bespoke Stripe-style HMAC |
|---|---|---|---|
| **Standards status** | IETF Standards Track, RFC, Feb 2024 `[FACT]` | Vendor-initiated, 8-person TSC, no standards body, spec frozen at 1.0.0 since 2025-02-16 `[FACT]` | None |
| **Signed timestamp / replay** | `created`+`expires`+`nonce` available; AdCP mandates `expires ≤ created+300`; **UCP's own example carries none** and defers replay to idempotency keys `[FACT]` | `webhook-timestamp` signed into the base; libraries hard-code 300 s `[FACT]` | Stripe `t=` + 300 s; but GitHub/Shopify/Linear/Notion sign body only — no replay defense `[COMPARATIVE]` |
| **Asymmetric option** | Native and working: JWKS or `.well-known` key discovery; ES256/ES384 (UCP), Ed25519/P-256 (AdCP) `[FACT]` | **Misuse-prone and unfixed.** `v1a,`/ed25519 is safe only with a per-endpoint key pair; its own author calls that requirement "unclear" and unsafe with one global pair. Redesign open since 2023-12-10 `[FACT]` | None (Plaid/Kinde/Neon each invented their own JWT/JWS scheme) `[COMPARATIVE]` |
| **Key rotation** | New `keyid` published in the key set; consumer refetches — no coordination `[INFERENCE]` | Multiple space-delimited signatures during cutover; requires secret redistribution `[FACT]` | Ad hoc; Stripe overlap ≤24 h; PagerDuty multi-sig `[COMPARATIVE]` |
| **Consumer verification burden** | High: structured-field parsing, signature-base canonicalization, key fetch (SSRF-safe), `alg` allowlist. AdCP's own checklists run to **15 steps (requests) / 14 steps (webhooks)** `[FACT]` | Low: string-concat `id.timestamp.payload`, one HMAC, constant-time compare `[FACT]` | Low, but different every time `[COMPARATIVE]` |
| **Library ecosystem** | Generic only; no webhook-specific tooling found. npm `http-message-signatures` **98,654**/mo; PyPI **151,510**/mo `[FACT]` | 9+ languages from one repo. npm `standardwebhooks` **68,554,790**/mo (inflated — `@clerk/backend` depends on it), PyPI **2,827,815**/mo; npm `svix` 23.6 M, PyPI `svix` 9.66 M `[FACT]` | Per-vendor SDK helper; nothing reusable `[COMPARATIVE]` |
| **Gateway / infra leverage** | None found `[FACT]` | Kong ships a native plugin (min Gateway 3.8; `secret_v1`, `tolerance_second` default 300) `[FACT]` — but it validates **inbound** and presumably keys on `webhook-*`, which Svix does not send by default `[INFERENCE]` | None |
| **Interop in practice** | Two protocols agree on the RFC but differ on algorithm (ES256 vs Ed25519), tag, and covered components — profile-level interop is not automatic `[INFERENCE]` | Real, but eroded by branded headers (Svix default) and Title-Case variants (Brex) `[FACT]` | Zero by construction |
| **Governance/longevity risk** | Low: RFC, permanent. Profile churn risk is at the protocol layer `[INFERENCE]` | Moderate: single-vendor gravity, frozen spec, an open security-relevant issue for 2.5+ y, adopter claims that don't verify `[INFERENCE]` | High: the standard is whatever the vendor last shipped |
| **Deployment evidence for webhooks** | UCP `MUST`, AdCP baseline-required, Qerko optional — all first appearing within ~12 months `[FACT]` | ~9 independent full-spec + 4 via-Svix verified `[FACT]` | Still the numerical majority across dev-tool/fintech APIs `[COMPARATIVE]` |

### Migration path — the decisive cell

`[FACT]` The two schemes are **compositional, not mutually exclusive**, and there is a
shipped example. UCP's Order Event Webhook header table lists six headers:

| Header | Role |
|---|---|
| `Webhook-Id` | "Unique event identifier" |
| `Webhook-Timestamp` | "Event occurrence timestamp (unix)" |
| `UCP-Agent` | signer profile URL |
| `Signature-Input` | RFC 9421 |
| `Signature` | RFC 9421 |
| `Content-Digest` | RFC 9530 |

`[FACT — conflict recorded]` The same page's "Example Webhook Request" code block omits
`Webhook-Id` and `Webhook-Timestamp`:

```
POST /webhooks/ucp/orders HTTP/1.1
Host: platform.example.com
Content-Type: application/json
UCP-Agent: profile="https://merchant.example/.well-known/ucp"
Content-Digest: sha-256=:X48E9qOokqqrvdts8nOJRJN3OWDUoyWxBf7kbu9DBPE=:
Signature-Input: sig1=("@method" "@authority" "@path" "content-digest" "content-type");keyid="merchant-2026"
Signature: sig1=:MEUCIQDTxNq8h7LGHpvVZQp1iHkFp9+3N8Mxk2zH1wK4YuVN8w...:
```

The header table and the example disagree. `[INFERENCE]` The normative table should
govern, but the discrepancy must be flagged rather than resolved by assumption. The
string `webhook-signature` appears nowhere on the page — so UCP borrows the Standard
Webhooks **envelope** headers while replacing its **signature** header entirely.

`[INFERENCE]` Forward-compatibility verdict: a provider that adopts Standard Webhooks
today and later adds RFC 9421 keeps `webhook-id` and `webhook-timestamp` unchanged, adds
`Signature-Input`/`Signature`/`Content-Digest`, and runs both during cutover — exactly
AdCP's 3.x transition. **Standard Webhooks is not a dead end.** Its `v1a,`/ed25519
asymmetric mode is the one part that offers no migration credit: it shares no
key-discovery mechanism, algorithm identifier, or wire format with 9421 asymmetric
signing, so a provider needing asymmetric signatures gains nothing by passing through it
on the way to 9421 + published keys.

---

## Q5 — Has either camp addressed the other?

`[FACT]` **Yes — from the Standard Webhooks side, once, in 2023, and never revisited.**
GitHub issue #26 (*"spec: signing the headers"*), comment by `tasn` (Tom Hacohen, Svix
founder, TSC member), **2023-12-08T22:09:25Z**, replying to a contributor who linked
`draft-ietf-httpbis-message-signatures`:

> "Hey @teliosdev, thanks for the link! We are familiar with the RFC (it's even mentioned
> in the README!). There were a few reasons, but one of the main ones is: one of the
> design goals of this spec is to meet people where they are. So following but cementing
> industry best practices, rather than trying to do something very different. It's much
> better to have a great spec that's widely adopted than a perfect one that's used by no
> one. **With that being said, as HTTP message signatures become more widespread/widely
> supported, this recommendation will probably change (future version of the spec).**"

The design objection to 9421-style canonicalization, same issue, 2023-11-19:

> "One additional challenge that you just mentioned is the rabbit hole of canonicalizing
> headers. I think it would be immensely difficult to maintain a consistent behavior
> across different implementations. So even if we wanted it, I think it's going to add a
> tonne of complexity."

`[FACT]` Absences, all checked 2026-08-09:
- **Zero** issues or pull requests in `standard-webhooks/standard-webhooks` mention "9421".
- The spec document contains **no RFC citations of any kind**.
- The README's "Related efforts → Active Projects" section still links *"IETF HTTP Message
  Signatures"* to the **draft** URL `httpwg.org/http-extensions/draft-ietf-httpbis-message-signatures.html`
  — not RFC 9421 — 2.5 years after publication. The section is prefaced: *"This
  specification is compatible with the rest of them."*
- Nothing on `docs.svix.com` mentions RFC 9421.
- Neither UCP nor AdCP mentions Standard Webhooks.

`[INFERENCE]` **There is no convergence plan, but there is a stated convergence
condition**, set by the maintainer and not yet re-evaluated: "as HTTP message signatures
become more widespread/widely supported, this recommendation will probably change." The
2026 UCP/AdCP evidence is the first material movement on that condition since the
statement was made, and the project has not responded to it. `[OPINION]` The stale draft
link is a small but telling sign that the related-efforts section is not being tracked.

---

## Recommended shape for OP-016

### Which candidates the evidence supports naming

`[OPINION]` Name **two**, selected by trust topology rather than ranked by preference, and
drop bespoke Stripe-style HMAC from the rule entirely (it has familiarity and nothing
else; every property it offers, the Standard Webhooks scheme offers with interop).

| Precedence | When | Why the evidence supports it |
|---|---|---|
| **1 — default** | The provider issues each consumer a secret (the ordinary product webhook) | ~9 independent verified full-spec implementations including OpenAI, Anthropic, Gemini, Replicate, Etsy; every major LLM-platform webhook launch of 2025–2026 (OpenAI, Anthropic, Gemini; xAI partially) chose it; a gateway plugin exists; ~19× the PyPI usage of 9421 tooling (npm ratio is larger but transitively inflated) |
| **2 — required for cross-organization delivery** | Consumers are third parties with whom no secret can be pre-shared, or the provider needs one key pair, HSM custody, or non-repudiation | Standard Webhooks' asymmetric mode is safe only with per-endpoint keys, a requirement its own author calls unclear, with the redesign open 2 y 8 m; UCP `MUST`s 9421, AdCP makes it baseline-required and deletes HMAC in 4.0 |

### Proposed rule wording

> **OP-016 (MUST).** Sign every outbound webhook over a base that binds a unique delivery
> ID, a timestamp, and the raw body. Where the provider issues each consumer a shared
> secret, use HMAC-SHA256 over `id.timestamp.body` carried in `webhook-id`,
> `webhook-timestamp` and `webhook-signature` headers (the Standard Webhooks scheme).
> Where consumers are third parties who cannot hold a shared secret — or where one key
> pair, hardware key custody, or non-repudiation is required — use RFC 9421 HTTP Message
> Signatures with `Content-Digest` (RFC 9530) and keys published at a discoverable key
> set, retaining `webhook-id`/`webhook-timestamp` as the delivery envelope.

> **Note to OP-016.** Standard Webhooks' `v1a,`/ed25519 asymmetric mode is safe only when
> the provider uses a distinct key pair per endpoint; with one shared signing key a
> tenant can register a victim's URL and receive valid signatures for messages it
> controls. Prefer RFC 9421 with published keys for any asymmetric requirement.

### Confidence

| Claim in the rule | Confidence | Basis |
|---|---|---|
| Signing is `MUST`; base binds ID + timestamp + raw body | **High** | Unchanged; universal across both camps |
| Standard Webhooks scheme as the shared-secret default | **Moderate-high** | Many verified independent implementations and every recent AI-provider launch, discounted for: frozen spec, no standards body, four of the twelve named README adopters failing verification (three more deliver only via Svix), Svix's non-conformant default headers |
| RFC 9421 for cross-organization/asymmetric delivery | **Moderate** | Two protocols and one fintech, all within ~12 months; no IETF webhook profile; no webhook-specific consumer tooling; UCP and AdCP already differ on algorithm and covered components |
| `v1a,`/ed25519 needs per-endpoint keys and is not a general asymmetric answer | **High** | The spec's own lead author documents the cross-tenant forgery and calls the per-endpoint requirement "unclear"; issue open since 2023-12-10 |
| Web Bot Auth does **not** extend to webhooks | **High** | Charter's explicit out-of-scope clause |

### Corrections this leaf makes to prior repo positions

1. `baseline-03b` raised OP-016's confidence partly because Cloudflare's Web Bot Auth
   proved the mechanism. That inference must now be **capped**: the webbotauth charter
   explicitly excludes HTTP APIs and agent-to-agent interfaces, so the trajectory does not
   reach webhooks. The 9421 case now rests on UCP/AdCP/Qerko instead — different evidence,
   similar strength, correctly scoped.
2. `baseline-03` recorded "no surveyed API has adopted [RFC 9421]" for webhooks. That was
   true of the eight legacy references and is **no longer true of the field**: as of 2026
   at least two multi-party protocols mandate it for webhook delivery.
3. `survey-07` repeated the Standard Webhooks README adopter list verbatim. Four of the
   twelve names fail verification as emitters and three more deliver only via Svix (see
   the corrected count in §D). Any downstream text should cite the verified
   table above, not the README.

### Dated re-check triggers

| Date / event | What to re-check |
|---|---|
| **2026-11-15** | `draft-knauer-secure-webhook-token-02` expires — the only IETF webhook-security draft |
| **AdCP 4.0 release** | Removal of the `authentication` object completes the HMAC → 9421 migration; confirms or falsifies the trend |
| **Standard Webhooks spec v1.1 / issue #34** | A resolved asymmetric scheme would change the Q4 asymmetric cell and possibly the `v1a,` ban |
| **Any UCP revision after 2026-04-08** | Resolves the `Webhook-Id`/`Webhook-Timestamp` table-vs-example conflict |

### Absences reported explicitly

- No IETF draft profiles RFC 9421 for webhooks (datatracker queried by name, 2026-08-09).
- No webhook-specific RFC 9421 verification library or gateway plugin found.
- No RFC 9421 support at Svix, Hookdeck (either product), Convoy, Inngest, Knock, Zapier,
  WorkOS, OpenAI, Anthropic, ElevenLabs, or Deck — each checked directly.
- No Standard Webhooks mention in UCP or AdCP; no RFC 9421 mention in the Standard
  Webhooks spec, issues, or Svix docs.
- Drata: no locatable webhook-signature documentation despite the README claim.
- Fireworks AI: webhook feature suggested by third parties; vendor docs returned 404;
  unverified in both directions.
- Cohere, Mistral (outbound), Groq, Together AI, Turso, Unkey: no documented signed
  outbound webhooks located.

### Data-quality caveats

- npm `standardwebhooks` at 68.5 M downloads/month is **inflated by transitive
  dependency** — `@clerk/backend@3.16.1` declares `"standardwebhooks": "^1.0.0"`
  (verified against the npm registry, 2026-08-09). Treat the ratio as directional
  evidence of ecosystem reach, never as a count of adopting companies. `openai@7.4.0`
  declares no dependencies, so it is not a contributor.
- OpenAI's own page does not restate the algorithm, signing base, or tolerance in its own
  prose; it satisfies them by reference to the linked spec and by shipping an SDK helper.
- Google Gemini's webhook feature is ~3 months old with at least three overlapping
  documentation pages; re-verify before citing in a ratified decision.
- ElevenLabs' `t=`/`v0=` format and 30-minute tolerance come from SDK inspection, not
  ElevenLabs prose.
- "Google backs UCP" is **not** an established claim. What is established: Google
  publishes a UCP implementation guide at `developers.google.com/merchant/ucp`, and the
  UCP governing body is unnamed on the pages retrieved.
