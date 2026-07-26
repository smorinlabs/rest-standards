# Baseline 03 — Security and Operational Practice

*Prescriptive research. Primary-source status verified live 2026-07-25 against
IETF Datatracker, W3C, and OWASP. Vendor-practice evidence is carried from the
in-repo `survey` series (retrieved 2026-07-19/20) and labeled as such. This
report **proposes** normative rules; nothing here is project policy until
ratified at Gate C.*

**Label key:** `[FACT]` sourced to a primary specification · `[COMPARATIVE]`
deployed vendor practice · `[INFERENCE]` reasoned from sourced facts ·
`[RECOMMENDATION]` proposed rule · `[POLICY]` a choice evidence cannot settle.

---

## 1. Executive recommendation

Ground every operational rule in **a named threat or failure mode**, not in
vendor prevalence — this domain is where popularity is least correlated with
correctness, because the dominant practices predate the current BCPs. Eight
commitments:

1. **TLS 1.2 minimum, 1.3 preferred**, per BCP 195. Reject everything below.
2. **Adopt BCP 240 (RFC 9700) wholesale for OAuth.** It carries hard
   prohibitions — ROPC `MUST NOT` be used, tokens `MUST NOT` appear in URI
   query parameters — that settle several otherwise-contested questions.
3. **Treat authorization as per-object and per-property, not per-endpoint.**
   The top three OWASP API risks are all authorization failures, and none is
   caught by authenticating the caller.
4. **Signal rate limits, and prefer the IETF `RateLimit` fields.** The draft is
   **active** (expires 2026-11-24) — unlike the dead idempotency draft — so
   adopting it is a forward bet with a documented fallback, not a fiction.
5. **Retry only what is safe to retry.** Retry eligibility derives from method
   idempotence (`baseline-01`) and the idempotency-key contract
   (`baseline-02`). Mandate jitter.
6. **Signal deprecation machine-readably** with RFC 9745 `Deprecation` plus RFC
   8594 `Sunset`, and note that the two use *deliberately different* date
   formats.
7. **Prefer RFC 9421 HTTP Message Signatures for webhooks**, with ad-hoc
   HMAC-SHA256 as a documented interim. Every surveyed vendor ships a mutually
   incompatible ad-hoc scheme; a standards-track alternative now exists.
8. **Require W3C Trace Context propagation on the wire**, and keep everything
   else about telemetry an internal control rather than a contract term.

**Overall confidence: high** for 1–3, 5, 6 (published BCPs and standards) ·
**moderate** for 4 and 7 (an active draft and a standard with thin API-side
adoption) · **high** for 8 as a wire requirement.

---

## 2. Source-and-currency matrix

All rows verified 2026-07-25. The **Limitations** column is required by the
prompt and is where this domain's evidence is weakest.

| Source | Class | Date | Status verified today | Limitations | URL |
| --- | --- | --- | --- | --- | --- |
| RFC 9700 — OAuth 2.0 Security BCP | **BCP 240** | Jan 2025 | Best Current Practice. Updates RFC 6749, 6750, 6819. | Addresses OAuth only; says nothing about API-key models. | https://datatracker.ietf.org/doc/rfc9700/ |
| RFC 9325 — TLS/DTLS Recommendations | **BCP 195** | Nov 2022 | Best Current Practice. Obsoletes RFC 7525. | Cipher guidance dates; re-verify each cycle. | https://datatracker.ietf.org/doc/rfc9325/ |
| RFC 9421 — HTTP Message Signatures | Standards Track | Feb 2024 | Internet Standards Track. | **No surveyed API has adopted it.** Adoption risk is real. | https://datatracker.ietf.org/doc/rfc9421/ |
| RFC 9745 — Deprecation Header | Standards Track | Mar 2025 | Internet Standards Track. Value is an RFC 9651 structured Date (`@<unix>`). | Young; tooling support unproven. | https://datatracker.ietf.org/doc/rfc9745/ |
| RFC 8594 — Sunset Header | **Informational** | May 2019 | **Informational — not Standards Track.** Syntax is `HTTP-date`. Not obsoleted or updated. | Weaker authority class than RFC 9745, which it pairs with. | https://datatracker.ietf.org/doc/rfc8594/ |
| `draft-ietf-httpapi-ratelimit-headers` | **Active Internet-Draft** | draft-**11**, 2026-05-23 | **ACTIVE.** Expires **2026-11-24**. Intended RFC status "(None)." Defines `RateLimit` and `RateLimit-Policy`. | Not a standard. May change or expire. Contrast with the dead idempotency draft. | https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/ |
| W3C Trace Context | **W3C Recommendation** | 23 Nov 2021 | Recommendation, version 1.0. Defines `traceparent` and `tracestate`. Errata exist; no Level 2 published. | Format is fixed at version `00`. | https://www.w3.org/TR/trace-context/ |
| OWASP API Security Top 10 | Industry risk list | **2023 edition** | Current — no newer edition than 2023. | **Awareness document, not a control framework.** Ranked by community survey and incident data, not by measured impact on any given API. Must not be cited as a conformance standard. | https://owasp.org/API-Security/editions/2023/en/0x11-t10/ |

### Currency corrections this thread produced

- `[FACT]` **RFC 8594 (Sunset) is Informational, not Standards Track**, while
  RFC 9745 (Deprecation) *is* Standards Track. The pair is routinely described
  as equivalent; they are not.
- `[FACT]` **The two headers use deliberately different date formats.**
  `Deprecation` carries an RFC 9651 structured Date (`Deprecation: @1688169599`);
  `Sunset` carries an `HTTP-date` (`Sunset: Sun, 30 Jun 2024 23:59:59 UTC`).
  RFC 9745 additionally requires that the Sunset timestamp **MUST NOT** be
  earlier than the Deprecation timestamp.
- `[FACT]` **The RateLimit draft is active, not expired**, at draft-11 dated
  2026-05-23 with expiry 2026-11-24. This is a materially different posture
  from the Idempotency-Key draft (`baseline-02`), which expired 2026-04-18 and
  is marked "no longer active."
- `[FACT]` **The RateLimit draft defines RFC 9457 problem types** —
  `quota-exceeded`, `temporary-reduced-capacity`, `abnormal-usage-detected` —
  which connects directly to `baseline-02` AC-003.
- `[FACT]` **RFC 9110 obsoletes RFC 2818**, so TLS-for-HTTP guidance citing
  RFC 2818 is citing an obsolete document (carried from `baseline-01`).

### Gap this thread closes against the `survey` series

`[FACT]` Verified by search across `research/reports/`: **no `survey` report
mentions RFC 9421, RFC 9700, RFC 9325, W3C Trace Context, `traceparent`, or
OWASP.** RFC 9745 is the sole security-or-operations standard the survey
covered.

`[INFERENCE]` This is the widest gap of the three baseline threads, and it
follows from the survey's method rather than from carelessness. The survey read
vendor API *documentation*, and security posture, TLS configuration, and
telemetry contracts are largely not published there — they live in trust
centers, compliance attestations, and internal runbooks. A thread that starts
from the standards registry finds them; a thread that starts from developer
docs cannot.

---

## 3. Threat and failure model

Recommendations below are tied to these, not to prevalence.

| # | Threat / failure | Where it bites | Primary mitigations |
| --- | --- | --- | --- |
| T1 | **Broken object-level authorization** (OWASP API1:2023) | Caller is authenticated and authorized for the *endpoint* but not the *object*. Enumerating IDs returns other tenants' data. | OP-005, OP-006 |
| T2 | **Broken property-level authorization** (API3:2023, consolidating 2019's excessive data exposure and mass assignment) | Response includes fields the caller may not read; request writes fields the caller may not set. | OP-007, OP-008 |
| T3 | **Credential exposure in transit or logs** | Token in a URI leaks to access logs, browser history, `Referer`, and analytics. | OP-002, OP-003 |
| T4 | **Resource exhaustion** (API4:2023) | Unbounded page sizes, expansions, or bulk operations turn one request into unbounded work. | OP-009, OP-010 |
| T5 | **Retry storms** | Clients retrying non-idempotent or non-retryable failures amplify an incident. Synchronized retries without jitter reconverge. | OP-011, OP-012 |
| T6 | **Replay** | A captured webhook or request is replayed. Signature without a timestamp does not prevent this. | OP-016, OP-017 |
| T7 | **Silent breaking change** | Consumers discover a break in their own production. | OP-013, OP-014 |
| T8 | **Telemetry as an attack surface** | Untrusted trace headers propagated unbounded; high-cardinality labels exhaust the metrics backend; diagnostics leak internals. | OP-018, OP-019, OP-020 |
| T9 | **Duplicate or out-of-order event processing** | At-least-once delivery with no ordering guarantee is the norm; consumers assuming otherwise corrupt state. | OP-021, OP-022 |
| T10 | **SSRF via API-supplied URLs** (API7:2023) | Webhook target URLs and callback parameters let a caller reach internal networks. | OP-023 |

---

## 4. Comparative vendor matrix

`[COMPARATIVE]` From `survey-06-lifecycle-operations` (both runs) and
`survey-07-webhooks`. Scope limited to this thread's domain, per the prompt.

| Dimension | The field |
| --- | --- |
| **Auth surface** | Seven of eight use bearer/basic tokens with typed key prefixes (`sk_live_`, `ghp_`, `shpat_`; Twilio `AC…`/`SK…`). **AWS SigV4 is the lone outlier**, requiring per-request HMAC signing — buying tamper-evidence and presigned URLs at the cost of implementation complexity. Twilio still defaults to Account SID + Auth Token HTTP Basic. |
| **Rate-limit signaling** | No consensus. GitHub `X-RateLimit-*`; Shopify leaky-bucket `X-Shopify-Shop-Api-Call-Limit`; Stripe `Stripe-Ratelimit-*` and largely 429-centric; Microsoft Graph and Twilio 429 + `Retry-After`-centric. **None of the eight has adopted the IETF `RateLimit` fields.** |
| **Versioning transport** | Dated header, account-pinned: Stripe (`Stripe-Version`, account locked at first request). Dated header, per-request: GitHub (`X-GitHub-Api-Version`, default `2022-11-28`). Dated path: Twilio (`/2010-04-01/`), Shopify (`/admin/api/2026-07/`). Path token: Google (`/v1`), Microsoft Graph (`/v1.0`, `/beta`). Query parameter: Azure (`api-version=`). **Forbidden entirely**: Zalando (`MUST NOT use URI versioning`). |
| **Deprecation windows** | Shopify ≥12 months per version with ≥9 months overlap. Others vary; no convergence. |
| **Webhook signature** | Nearly all HMAC-SHA256, formatted incompatibly. Stripe `Stripe-Signature: t=…,v1=…` hex. GitHub `X-Hub-Signature-256: sha256=…` hex plus legacy SHA-1. Shopify `X-Shopify-Hmac-Sha256` base64. Standard Webhooks `webhook-signature: v1,<base64>`. Twilio outlier: **HMAC-SHA1** over URL + sorted POST params. Graph: no body HMAC by default — `clientState` echo plus `validationToken` handshake. AWS SNS: RSA over canonicalized fields with a published cert URL. Google Pub/Sub: OIDC JWT bearer. |
| **Replay protection** | Not universal. Stripe embeds `t=` and recommends 300 s tolerance; Standard Webhooks signs the timestamp with a 5-minute default. **Shopify and GitHub sign only the body** — no timestamp in the signature, hence no built-in replay protection — though both send delivery identifiers (`X-Shopify-Triggered-At`, `X-GitHub-Delivery`) enabling app-level dedup. |
| **Retry windows** | Span three orders of magnitude. GitHub: **no automatic retries** (manual redelivery within 3 days). AWS SNS: 3 retries, ~35 s. Shopify: 8 attempts over 4 hours (changed 2024-09-10 from 19 attempts/48 h). Microsoft Graph: up to 4 hours. Stripe: up to 3 days, exponential. AWS EventBridge: 24 hours, up to 185 attempts. |

`[INFERENCE]` The webhook rows are the clearest case in this entire research
effort where **convergence in intent coexists with total divergence in
mechanics**. Everyone signs; no two verify identically. That is precisely the
condition a standards-track specification exists to fix, and RFC 9421 is that
specification.

---

## 5. Anti-patterns

| Anti-pattern | Concrete failure | Threat |
| --- | --- | --- |
| **Bearer tokens in URLs** | Leaks to access logs, browser history, `Referer` headers, and analytics pipelines. `[FACT]` RFC 9700 states clients **MUST NOT** pass access tokens in a URI query parameter. | T3 |
| **Long-lived broad tokens** | A single leaked credential grants full access indefinitely. Blast radius equals token scope times lifetime. | T3 |
| **Authentication mistaken for object authorization** | The caller is a valid user, so the object is returned — regardless of whether it is *their* object. The single most common API breach pattern. | T1 |
| **Predictable identifiers as access control** | Sequential IDs plus missing object checks equals trivial enumeration. Unguessable IDs are defence in depth, never a substitute for a check. | T1 |
| **Mass assignment / excessive exposure** | Binding a request body straight to a model lets a caller set `role` or `account_id`; returning a whole model leaks internal fields. | T2 |
| **Unlimited expensive operations** | No cap on page size, expansion depth, or bulk item count converts one request into unbounded server work. | T4 |
| **Undocumented proprietary rate headers** | Clients cannot back off correctly against headers they cannot discover or parse. | T5 |
| **Retrying every failure** | Retrying a 400 or 422 will never succeed; retrying a non-idempotent POST without a key duplicates state. | T5 |
| **Retries without jitter** | Synchronized clients retry in lockstep and reconverge into repeated thundering herds. Exponential backoff alone does not desynchronize. | T5 |
| **Leaking sensitive diagnostics** | Stack traces, SQL fragments, and internal hostnames in error bodies hand an attacker a map. | T8 |
| **Unbounded high-cardinality labels** | User IDs or request IDs as metric labels produce unbounded time series and take down the metrics backend. | T8 |
| **Propagating untrusted trace fields without limits** | `tracestate` from an untrusted caller, propagated unbounded, is an injection and amplification vector. | T8 |
| **Silent breaking shutdowns** | Removing an endpoint with no `Deprecation`/`Sunset` signal and no notice. | T7 |
| **Indefinite deprecation** | "Deprecated" for years with no sunset date trains consumers to ignore the signal entirely. | T7 |
| **Unsigned webhooks** | Any party who learns the endpoint URL can forge events. | T6 |
| **Verifying the signature after parsing JSON** | The parser runs on unauthenticated attacker-controlled bytes. Verify over the **raw body** first. | T6 |
| **Replay window with no delivery dedup** | A timestamp tolerance alone permits replay *within* the window. Needs a delivery ID and consumer-side dedup. | T6 |
| **Slow synchronous webhook handlers** | Doing work before acknowledging causes provider timeouts and redelivery, multiplying load during an incident. | T9 |
| **Assuming exactly-once delivery** | Every surveyed provider is at-least-once. Non-idempotent consumers corrupt state on the first duplicate. | T9 |
| **Assuming event order** | No surveyed provider guarantees ordering. Consumers ordering by arrival will apply a stale update over a fresh one. | T9 |

---

## 6. Contested choices

| Choice | Recommendation | Basis and tradeoffs | Conf. |
| --- | --- | --- | --- |
| **OAuth/OIDC vs API keys** | Conditional by client type. OAuth/OIDC where a user delegates authority or a third party acts on their behalf; API keys acceptable for server-to-server with a single trust relationship. | `[FACT]` RFC 9700 governs OAuth but is silent on API keys. `[COMPARATIVE]` seven of eight ship key-based auth. Neither is universally correct. | High |
| **Opaque vs self-contained tokens** | Opaque by default for public APIs. | Self-contained (JWT) removes an introspection round-trip but makes revocation hard and leaks claims to anyone holding the token. | Moderate |
| **Sender-constraining** | SHOULD for high-value scopes; MAY generally. | mTLS or DPoP defeats simple token replay. Real deployment cost. | Moderate |
| **Scope and audience design** | Scopes per resource-and-action; audience always bound to the API. | Coarse scopes make least privilege unachievable; over-fine scopes are unusable. | Moderate |
| **Rate-limit headers while IETF work is draft** | Emit IETF `RateLimit`/`RateLimit-Policy` as primary. Document any legacy proprietary headers as deprecated aliases. | `[FACT]` draft-11 is **active**, expires 2026-11-24. `[COMPARATIVE]` **zero of eight** have adopted it. A forward bet — but a documented fallback makes it reversible, and the alternative is inventing a ninth proprietary scheme. | Moderate |
| **429 vs overload 5xx** | 429 for a *known quota* decision; 503 for capacity overload. Both carry `Retry-After`. | `[FACT]` 429 is RFC 6585; 503 is RFC 9110 §15.6.4. Conflating them denies the client the information needed to choose between backoff and failover. | High |
| **Retry eligibility and budgets** | Retry only idempotent methods, or non-idempotent requests carrying an idempotency key. Mandate jitter. Cap with a client-side budget. | Derives from `baseline-01` method properties and `baseline-02` AC-016. | High |
| **Correlation IDs: client-supplied vs generated** | Accept a client-supplied ID, but always generate and return the authoritative one. | Client-supplied alone is untrusted and may collide; server-only prevents the client correlating its own retries. | Moderate |
| **Trace propagation and privacy** | Require W3C `traceparent`. Accept `tracestate` with a **size and entry cap**. Never propagate to an untrusted third party. | `[FACT]` W3C Recommendation, Nov 2021. Trace IDs can carry inferable information across a trust boundary. | High |
| **Minimum telemetry contract** | On the wire: a correlation ID on every response and `traceparent` propagation. Everything else internal. | The prompt requires separating wire contract from internal guidance; this is the split. | High |
| **Deprecation channels and windows** | Machine-readable `Deprecation` + `Sunset` + a `deprecation` Link relation to human documentation, plus out-of-band notice. `[POLICY]` on window length. | `[FACT]` RFC 9745 defines the Link relation usage. `[COMPARATIVE]` Shopify ≥12 months with ≥9 months overlap is the only concrete published window found. | High for mechanism, `[POLICY]` for duration |
| **Webhook HMAC vs RFC 9421** | Prefer RFC 9421. Permit ad-hoc HMAC-SHA256 over `id.timestamp.payload` as a documented interim. | `[FACT]` RFC 9421 is Standards Track (Feb 2024) and solves canonicalization across intermediaries. `[COMPARATIVE]` adoption among the eight is **zero**; Standard Webhooks is the pragmatic interim with real adoption. | Moderate |
| **Key rotation** | Support overlapping active secrets and publish the rotation procedure. | Without overlap, rotation is a hard cutover that drops events. | High |
| **Replay tolerance** | Signed timestamp with a bounded window (300 s is the convergent value) **plus** delivery-ID dedup. | `[COMPARATIVE]` Stripe 300 s; Standard Webhooks 5 min. Window alone is insufficient — see anti-patterns. | High |
| **Ordering guarantees** | Guarantee none. Publish a per-event monotonic version or timestamp so consumers can discard stale updates. | `[COMPARATIVE]` no surveyed provider guarantees order. Promising it is unimplementable at scale. | High |
| **Dead-letter and redelivery** | Provide consumer-visible delivery history and manual redelivery. | `[COMPARATIVE]` GitHub offers manual redelivery within 3 days and no automatic retries at all — a viable model. | Moderate |

---

## 7. Proposed normative principles

Provisional IDs use the `OP-` prefix. The **Obs.** column answers the prompt's
requirement to separate externally observable contract terms from internal
controls: **W** = observable on the wire or required in public documentation ·
**I** = internal control, recommended but not externally verifiable.

| ID | Str. | Obs. | Rule | Threat | Evidence | Conf. |
| --- | --- | --- | --- | --- | --- | --- |
| OP-001 | MUST | W | Serve only over TLS 1.2 or higher; prefer 1.3. Reject SSL 2/3 and TLS 1.0/1.1. | T3 | [BCP 195](https://datatracker.ietf.org/doc/rfc9325/) | High |
| OP-002 | MUST NOT | W | Accept or emit access tokens in URI query parameters. | T3 | [BCP 240](https://datatracker.ietf.org/doc/rfc9700/) | High |
| OP-003 | MUST NOT | W | Use the OAuth resource owner password credentials grant. | T3 | [BCP 240](https://datatracker.ietf.org/doc/rfc9700/) — "MUST NOT be used" | High |
| OP-004 | MUST | W | Where OAuth is used: require PKCE for public clients, exact-string redirect URI matching, and avoid the implicit grant. | T3 | [BCP 240](https://datatracker.ietf.org/doc/rfc9700/) | High |
| OP-005 | MUST | I | Authorize every request against the specific object, not only the endpoint. | T1 | [OWASP API1:2023](https://owasp.org/API-Security/editions/2023/en/0x11-t10/) | High |
| OP-006 | MUST NOT | I | Rely on identifier unguessability as an access control. | T1 | [OWASP API1:2023](https://owasp.org/API-Security/editions/2023/en/0x11-t10/) | High |
| OP-007 | MUST | I | Authorize readable and writable properties per caller, not per endpoint. | T2 | [OWASP API3:2023](https://owasp.org/API-Security/editions/2023/en/0x11-t10/) | High |
| OP-008 | MUST | W | Bind request bodies to an explicit allow-list of writable fields. | T2 | [OWASP API3:2023](https://owasp.org/API-Security/editions/2023/en/0x11-t10/) | High |
| OP-009 | MUST | W | Publish and enforce a maximum page size, expansion depth, and bulk item count. | T4 | [OWASP API4:2023](https://owasp.org/API-Security/editions/2023/en/0x11-t10/) | High |
| OP-010 | MUST | W | Apply rate limits and communicate them via `RateLimit` and `RateLimit-Policy`. **Contingency: if the draft expires unadvanced on 2026-11-24, fall back to a documented proprietary scheme and stop citing it.** | T4, T5 | [draft-11](https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/); leaf `baseline-03b` | **Low-moderate** (lowered) — design judged sound by HTTPDIR, but 7 years without advancing, no Last Call, expires 2026-11-24 |
| OP-011 | MUST | W | Return 429 for quota exhaustion and 503 for capacity overload; include `Retry-After` on both. | T5 | [9110 §15.6.4](https://www.rfc-editor.org/rfc/rfc9110.html#section-15.6.4), RFC 6585 | High |
| OP-012 | MUST | W | Document which failures are retryable. Require exponential backoff **with jitter**. Never advise retrying a non-idempotent request without an idempotency key. | T5 | `baseline-01`, `baseline-02` AC-016 | High |
| OP-013 | MUST | W | Signal deprecation with the `Deprecation` header (RFC 9651 structured Date) and a `Sunset` header (HTTP-date) whose timestamp is not earlier. | T7 | [9745](https://datatracker.ietf.org/doc/rfc9745/), [8594](https://datatracker.ietf.org/doc/rfc8594/) | High |
| OP-014 | MUST | W | Pair every deprecation with a `deprecation` Link relation to human-readable migration documentation, and never deprecate without a sunset date. | T7 | [9745](https://datatracker.ietf.org/doc/rfc9745/) | High |
| OP-015 | MUST | W | Use one uniform version-marker placement across the API. | T7 | `survey-06` `[COMPARATIVE]`, both runs concurring | Moderate — placement *choice* remains `[POLICY]` |
| OP-016 | MUST | W | Sign every outbound webhook. Prefer RFC 9421; otherwise HMAC-SHA256 over a base that includes delivery ID, timestamp, and raw body. | T6 | [9421](https://datatracker.ietf.org/doc/rfc9421/); leaf `baseline-03b` | **Moderate-high** (raised) — Cloudflare ships RFC 9421 verification in production via Web Bot Auth (backed by Cloudflare, Amazon, Akamai, OpenAI). Proves the mechanism at edge scale, **but for inbound bot auth, not webhook consumers** — so "prefer," not "must use" |
| OP-017 | MUST | W | Verify the signature over the **raw body before parsing**, enforce a bounded timestamp window, and deduplicate by delivery ID. | T6 | `[INFERENCE]`; `survey-07` `[COMPARATIVE]` (300 s convergent) | High |
| OP-018 | MUST | W | Return a correlation identifier on every response, including errors. | T8 | `[INFERENCE]` | High |
| OP-019 | MUST | W | Propagate W3C `traceparent`. Accept `tracestate` subject to a published size and entry cap. | T8 | [Trace Context](https://www.w3.org/TR/trace-context/) | High |
| OP-020 | MUST NOT | W | Expose stack traces, query fragments, internal hostnames, or dependency detail in any response body. | T8 | [OWASP API8:2023](https://owasp.org/API-Security/editions/2023/en/0x11-t10/) | High |
| OP-021 | MUST | W | Document delivery as at-least-once with no ordering guarantee, and publish a per-event monotonic version or timestamp. | T9 | `survey-07` `[COMPARATIVE]` | High |
| OP-022 | MUST | W | Require consumers to acknowledge before processing, and publish the acknowledgement timeout and retry schedule. | T9 | `survey-07` `[COMPARATIVE]` | High |
| OP-023 | MUST | I | Validate and restrict caller-supplied URLs (webhook targets, callbacks) against internal address ranges. | T10 | [OWASP API7:2023](https://owasp.org/API-Security/editions/2023/en/0x11-t10/) | High |
| OP-024 | MUST | W | Support overlapping active webhook signing secrets and publish the rotation procedure. | T6 | `[INFERENCE]` | High |
| OP-025 | SHOULD | W | Issue scoped, expiring credentials. Avoid long-lived unscoped tokens. | T3 | [BCP 240](https://datatracker.ietf.org/doc/rfc9700/) | High |

---

## 8. Conflicts and open questions

### 8.1 Research-resolvable

- **RFC 9421 API-side adoption.** Zero of eight surveyed references use it, and
  the survey never examined it. Whether webhook consumer libraries support it
  is an empirical question and the sole reason OP-016 says "prefer" rather than
  "must use." Suggested leaf: `baseline-03b-message-signatures-adoption`.
- **RateLimit draft trajectory.** Expires 2026-11-24. Whether it is renewed,
  advanced, or abandoned materially changes OP-010. This needs a calendar
  check, not a research thread.
- ~~**The `survey-06` divergence.**~~ **Withdrawn 2026-07-25.** An earlier
  draft of this report claimed the two `survey-06` runs disagreed on
  versioning transport. Comparing the report bodies rather than their
  summaries shows they agree: both place Google and Microsoft Graph on coarse
  major-version path tokens and Azure on a dated query parameter. The runs
  foreground different vendors in their TL;DRs, which reads as disagreement
  until checked. OP-015 is **not** blocked, and agreement across two
  independent runs raises its confidence rather than lowering it.

### 8.2 Risk-based conditional

These change with threat model rather than being resolvable by evidence:
sender-constrained tokens · opaque vs self-contained tokens · rate-limit
aggressiveness · replay window length · whether object-level authorization
needs a centralized enforcement point.

### 8.3 Organization policy

Deprecation window length and version-overlap duration · support tiers ·
API-key vs OAuth per client class · dead-letter retention · whether to publish
proprietary rate headers alongside the IETF fields during transition.

### 8.4 A standing tension worth recording

`[INFERENCE]` OP-010 and OP-016 both recommend a standard that **no surveyed
vendor has adopted**. That is a deliberate posture, not an oversight: this
project is writing a standard, not describing the market, and in both cases the
dominant practice is a set of mutually incompatible proprietary schemes whose
only advantage is incumbency. But the posture has a real cost — client
libraries in the wild handle `X-RateLimit-*` and Stripe-style signatures, not
the standardized forms. Both rules therefore carry documented fallbacks. A
reviewer who weights ecosystem compatibility above standards alignment could
legitimately invert either, and both belong on the Gate C agenda.

---

## 9. Dependency handoff

**To `baseline-01`** — consumed, not re-decided: method idempotence as the
basis for retry eligibility; 429 and 503 semantics; cache directives for
authenticated responses.

**To `baseline-02`** — consumed, not re-decided: the idempotency-key contract
that OP-012 relies on; RFC 9457 problem-document shape, which the RateLimit
draft's three problem types plug into; the event representation that webhooks
carry, as distinct from delivery mechanics owned here; compatibility taxonomy
underlying OP-013 through OP-015.

---

## 10. Confidence and invalidating assumptions

**Overall confidence: high** for rules resting on published BCPs and standards
(OP-001 through OP-009, OP-011 through OP-014, OP-017 through OP-025).
**Moderate** for OP-010, OP-015, and OP-016, each for a stated reason.

Assumptions that would materially change these recommendations:

1. **That the API is public or crosses a trust boundary.** A fully internal API
   inside one trust domain could justify relaxing OP-002, OP-019, and OP-023.
2. **That clients include browsers or third-party integrators.** Server-to-
   server-only traffic with no delegated authority weakens the OAuth rules and
   strengthens the API-key case.
3. **That the service is multi-tenant.** OP-005 and OP-007 are motivated by
   cross-tenant exposure; single-tenant deployments face a smaller risk.
4. **That data sensitivity is moderate or higher.** Public non-sensitive data
   makes OP-025 and sender-constraining hard to justify.
5. **That webhooks are part of the product.** OP-016, OP-017, OP-021, OP-022,
   and OP-024 all fall away without outbound events.
6. **That the RateLimit draft survives.** If it expires unrenewed after
   2026-11-24, OP-010 must fall back to a documented proprietary scheme, and
   the standard should say so rather than cite a dead draft.
7. **That no regulatory regime applies.** This report deliberately invents no
   regulatory requirements, per the prompt. A regulated deployment adds
   controls not analyzed here.

**Research completeness note.** This report recommends a risk-grounded baseline
and distinguishes standards (RFC 9421, 9745), BCPs (9700, 9325), Informational
documents (8594), active drafts (RateLimit), dead drafts (Idempotency-Key, via
`baseline-02`), and vendor conventions throughout, as the prompt requires. It
declines to produce a generic security checklist: every rule is tied to a
numbered threat in §3, and the two rules that diverge from universal vendor
practice are argued in §8.4 rather than asserted.
