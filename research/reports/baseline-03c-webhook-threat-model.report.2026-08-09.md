# Baseline 03c — Webhook Signing: Purpose Validation and Scheme-vs-Threat Evaluation

*Narrow leaf under `baseline-03`, raised at Gate C before ratifying `OP-016`.
Validates the purpose of webhook signing from primary sources and evaluates
the surveyed vendors' schemes against the resulting threat model. Mechanics
are documented in `survey-07`; deployment status of RFC 9421 in
`baseline-03b`. Run 2026-08-09; all sources accessed 2026-08-09.*

## 1. Purpose — validated, with one correction

**Claim tested:** "the purpose of webhook signing is security."
**Verdict: confirmed as primary, but incomplete.** Origin authentication plus
payload integrity is the stated purpose in every primary source. GitHub
states an additional *operational* purpose alongside it. No source claims
misrouting-detection or testing as a purpose.

`[FACT]` **Stripe** (docs.stripe.com/webhooks): *"Without verification, an
attacker could send fake webhook events to your endpoint to trigger actions
like fulfilling orders, granting account access, or modifying records. Always
verify that webhook events originate from Stripe before acting on them."*
Signature confirms *"the event wasn't sent or modified by a third party."*

`[FACT]` **GitHub** (validating-webhook-deliveries): *"To ensure that your
server only processes webhook deliveries that were sent by GitHub and to
ensure that the delivery was not tampered with"* … *"This will help you avoid
spending server time to process deliveries that are not from GitHub and will
help avoid man-in-the-middle attacks."* ← operational and security purposes
stated together.

`[FACT]` **Standard Webhooks** spec: *"Webhooks are just HTTP requests from
an unknown source, so verifying the authenticity of webhooks is a requirement
for any secure webhook implementation."* Site: *"every webhook implementation
needs to protect themselves and their users from SSRF, spoofing, and replay
attacks."*

`[FACT]` **OWASP draft** cheat sheet: signing *"lets the subscriber confirm
that a delivery came from the legitimate publisher and that the body was not
tampered with."*

`[INFERENCE]` Per-endpoint secrets (Stripe: *"Stripe generates a unique
secret key for each endpoint"*) give misrouted-delivery detection as a side
effect. **Not claimed as a purpose by any source** — do not cite it as one.

### Absence finding (material)

`[FACT]` **OWASP has no published webhook cheat sheet as of 2026-08-09.** The
"Webhook Security Guidelines Cheat Sheet" exists only under
`OWASP/CheatSheetSeries/cheatsheets_draft/` and does not appear in the
published index at `cheatsheetseries.owasp.org`. Proposed via issue #357 /
PR #383. The standard must cite it as an unratified draft, never as OWASP
guidance.

## 2. Threat model (consumer exposing an HTTPS endpoint)

| ID | Threat | Stopped by |
|---|---|---|
| T1 | **Forgery/spoofing** — anyone can POST | Body signature. Dominant real-world failure |
| T2 | **Tampering after TLS termination** — CDN/WAF/gateway terminates TLS; TLS is hop-by-hop there | End-to-end signature over the body. This is the residual value of a signature *given* TLS |
| T3 | **Replay of a captured valid delivery** | Signed timestamp bounds it; dedupe on a *signed* delivery ID closes it |
| T4 | **Cross-endpoint / cross-tenant replay** | Per-endpoint secrets, signed tenant identity, or signed destination URI |
| T5 | **Unsigned metadata** — consumer dispatches on a header outside the signature | Put it inside the signature |
| T6 | **Timing attack on comparison** | Constant-time compare |
| T7 | **Algorithm weakness** (HMAC-SHA1) | See §3(a) |
| T8 | **Secret leakage / rotation failure** | Overlapping-validity rotation; fail-closed on empty secret |
| T9 | **SSRF** — (a) provider-side, consumer registers an internal URL; (b) consumer-side, thin handler dereferences a forged ID | Signature stops (b) only |
| T10 | **Verification skipped due to operational friction** (raw-body breakage) | Not cryptographic; empirically the most consequential |
| T11 | **Downgrade** to a legacy/weaker scheme | Reject unknown schemes rather than falling back |
| T12 | **DoS** — unbounded processing of junk | Cheap signature check first (GitHub's stated operational rationale) |

**T3's precondition matters.** Under TLS, obtaining a valid
`(body, signature)` pair is *not* passive network capture — it requires
consumer-side logs/APM/proxy capture, a leaked delivery, or a compromised
intermediary. Timestamp and delivery-ID dedupe are **complementary, not
alternatives**: the tolerance window bounds how long IDs must be retained.

**T5's precondition matters too.** `[INFERENCE]` Exploitable only by an
attacker who can rewrite headers — i.e. post-TLS-termination, the same
population as T2, not a remote network attacker. RFC 9421 §7.2.1 names the
class: *"Any portions of the message not covered by the signature are
susceptible to modification by an attacker without affecting the signature…
the unsigned content injected by the attacker would subvert the trust
conveyed by the valid signature."*

**Not addressed by any body signature:** payload confidentiality;
consumer→provider authentication; provider-side SSRF; ordering; delivery
guarantees. **mTLS** binds channel identity but dies at a TLS-terminating
intermediary — mTLS and body signatures solve *different halves*; neither
substitutes for the other. RFC 9421 §7.1.2: *"The use of HTTP message
signatures does not negate the need for TLS… Message signatures provide
message integrity over the covered message components but do not provide any
confidentiality."*

### Documented incidents and CVEs (all primary-verified)

| Ref | What | Threat |
|---|---|---|
| **CVE-2026-4986** | WPForms Lite 1.10.0.1–1.10.0.4, **>5M installs**. *"the PayPal Commerce webhook processed incoming events without first verifying that PayPal actually sent them."* A forged `PAYMENT.CAPTURE.COMPLETED` marks a pending transaction completed. Fixed 1.10.0.5 (2026-05-12). **11 independent researchers found it.** | T1 |
| **CVE-2026-41432** (GHSA-xff3-5c9p-2mr4) | QuantumNous/new-api <0.12.10, High, CVSS 7.1, 2026-04-22. *"When the HMAC secret is empty, any attacker can compute valid webhook signatures, effectively bypassing signature verification entirely."* | T8/T1 |
| **CVE-2025-53548** (GHSA-9mp4-77wg-rwx9) | @clerk/backend <2.4.0 + 8 sibling packages, High, CVSS 7.5, 2025-07-09. *"Applications that use the `verifyWebhook()` helper to verify incoming Clerk webhooks are susceptible to accepting improperly signed webhook events."* **Flaw in the provider's own SDK.** *(Secondary aggregators describe an inverted check; the GHSA does not — treat that detail as unconfirmed.)* | T1 |
| **CVE-2026-56357** | n8n <1.123.15 / <2.5.0 — GitHub Webhook Trigger node *"does not verify HMAC-SHA256 signatures on incoming requests."* | T1 |
| **CVE-2022-36885** (Jenkins SECURITY-1849, severity Low) | *"GitHub Plugin 1.34.4 and earlier does not use a constant-time comparison when checking whether the provided and computed webhook signatures are equal."* Fixed 1.34.5. **Only located CVE for timing.** | T6 |
| Jack Cable, 2018-03-13 | Forged `{payment:{status:success,provider:stripe}}` authorized his account; criticized Stripe for documenting verification *"as a sidenote"* with example code that *"doesn't include any signature verification."* *(Dated — Stripe's current docs carry a dedicated verification section in the main flow.)* | T1 |

### Absence findings

- **No documented in-the-wild webhook *replay* incident found.** Every
  located CVE and disclosure is missing-or-broken verification (T1/T6/T8),
  never replay of a captured valid delivery (T3). The threat motivating the
  *signature* has many instances; the threat motivating the *signed
  timestamp* has none located. Counter-consideration: replay is
  low-observability and would often be indistinguishable from duplicate
  delivery in logs, so absence of reports is weak evidence of absence.
- **Could not verify NIST SP 800-131A Rev 2 / Rev 3-ipd HMAC table
  verbatim** — PDF text extraction failed on both. RFC 6194 plus the NIST
  2022 announcement carry the claim instead.
- **Could not retrieve HackerOne #508459** (Omise webhook SSRF → AWS keys);
  page returned no body. Cited nowhere.

## 3. Scheme × threat matrix

Legend: **Y** protected · **P** partial · **N** unprotected · – n/a

| Scheme | T1 forge | T2 tamper post-TLS | T3 replay window | T4 cross-endpoint | T5 metadata | T6 timing | T7 alg | T8 rotation |
|---|---|---|---|---|---|---|---|---|
| **Stripe** `t=,v1=` HMAC-SHA256 over `{t}.{raw_body}` | Y | Y | **Y** signed `t`, 5-min lib default | Y per-endpoint secret | Y — type and event id are in the signed body | Y documented | Y SHA-256 | **Y** 24h overlap, one signature per active secret |
| **GitHub** `sha256=` HMAC over raw body | Y | Y | **N** no signed timestamp; dedupe key `X-GitHub-Delivery` is itself **unsigned** | P per-hook secret but operator-chosen; reuse opens it | **N** `X-GitHub-Event` is the sole event-type carrier and is unsigned | Y documented (`secure_compare`/`timingSafeEqual`) | P SHA-256 + legacy SHA-1 header still sent → T11 surface | **N** no documented overlap; hard cutover |
| **Shopify** base64 HMAC over raw body | Y | Y | **N** `X-Shopify-Triggered-At` unsigned | **N** key is the **app** client secret, shared across all shops; `X-Shopify-Shop-Domain` unsigned | **N** topic/shop/triggered-at all unsigned | Y documented (`timingSafeEqual`) | Y SHA-256 | P ~1h rotation, rotates the whole app's auth |
| **Twilio** base64 HMAC-**SHA1** over URL + sorted POST params | Y | P signs params, not a raw body — a JSON body would be uncovered | **N** no timestamp — and Twilio permits plain HTTP callbacks (*"Twilio can use the HTTP protocol for callbacks"*), so on an HTTP endpoint capture-and-replay is a **passive** attack with no window bound | **Y — strongest**: full URL is signed, so a delivery cannot be replayed at a different endpoint | – no metadata headers | Y via SDK `RequestValidator` | **P** see §3(a) | P AuthToken is the account-wide API credential; huge blast radius |
| **MS Graph** default: `clientState` echo | **N** shared secret echoed in the body: proves knowledge, gives **no** integrity; leaks to anyone who ever sees one notification | **N** | **N** | N | N | no guidance given | – | P recreate subscription |
| **AWS SNS** RSA over canonicalized fields | **Y+ asymmetric** (consumer cannot forge) | Y | P signed `Timestamp` field | Y | Y fields canonicalized into the signature | – | Y | Y via `SigningCertURL`. New risk `[INFERENCE, unverified]`: consumer must pin the cert URL host or an attacker supplies their own cert — AWS's own guidance not fetched this run |
| **Google Pub/Sub** OIDC JWT bearer | Y+ asymmetric sender auth | **N — the JWT does not cover the body**; a post-TLS intermediary can alter the payload and the token stays valid | P JWT `exp`, short-lived | Y `aud` binds the endpoint | N | – | Y | Y Google-managed keys |
| **Standard Webhooks** HMAC-SHA256 over `id.timestamp.payload` | Y | Y | **Y** signed timestamp, 5-min lib default | **Y** signed `msg_id` + per-endpoint secret | **Y** id and timestamp explicitly inside the signed base | Y mandated | Y SHA-256, optional ed25519 `v1a` | **Y** multiple space-delimited signatures |

**Most complete against the enumerated threats: Standard Webhooks, then
Stripe. Weakest: Microsoft Graph default (no payload integrity at all).**

### (a) HMAC-SHA1 — not broken, but prohibited-by-trajectory

`[FACT]` **RFC 6194 §3.3 ("HMAC-SHA-1"), in its entirety:** *"As of today,
there is no indication that attacks on SHA-1 can be extended to HMAC-SHA-1."*
SHA-1 collision work (SHAttered class) attacks *collision resistance*; HMAC's
unforgeability rests on the compression function behaving as a PRF, not on
collision resistance. **Raw-SHA-1 collision results therefore do not yield
webhook forgery here.**

`[FACT]` **Twilio's own docs anticipate the criticism:** *"Twilio does not
use SHA-1 alone. The critical component of `HMAC-SHA1` that distinguishes it
from `SHA-1` alone is the use of your Twilio AuthToken as a complex secret
key."*

`[FACT]` **NIST, 2022-12-15:** transition away from SHA-1 *"for applying
cryptographic protection to all applications"* by **December 31, 2030**.

`[INFERENCE]` Twilio's scheme is not presently forgeable. But a greenfield
standard published now would be specifying a primitive with a published
retirement date, at zero benefit — SHA-256 costs nothing more. **Prohibit
SHA-1, and state the reason accurately (retirement schedule), not
inaccurately (collisions).** Getting that reason wrong is itself a defect a
reviewer will catch.

### (b) Graph has no payload integrity — but a thin architecture that limits the harm

`[FACT]` *"The **clientState** property is required. Setting the property
allows your service to confirm that change notifications you receive
originate from Microsoft Graph. For this reason, the value of the property
should remain secret and known only to your application and the Microsoft
Graph service."*
`[FACT]` *"Validate the **clientState** property. It must match the value
originally submitted with the subscription creation request. If there's a
mismatch, don't consider the change notification as valid."*
`[FACT]` The `validationToken` handshake proves endpoint ownership at
subscribe time; *"If the endpoint validation fails, Microsoft Graph doesn't
create the subscription."*

`[INFERENCE]` `clientState` is a **bearer secret, not a signature**:
replayable forever once observed, zero integrity over the body. Graph's real
mitigation is architectural — basic notifications are **thin** (id only), so
the consumer re-queries Graph with its own credentials and never acts on
attacker-supplied data. **Payload thinness substitutes for payload
integrity**, and the standard should say so: the strength a signature must
carry is a function of whether the consumer acts on payload contents.

### (c) Bearer tokens authenticate the sender, not the message

`[INFERENCE]` Google Pub/Sub's OIDC JWT proves who connected; it does not
bind the body. A signature and a bearer token are not interchangeable
controls.

## 4. Criticisms per scheme

- **GitHub** — no signed timestamp, so no replay bound; the recommended
  dedupe key `X-GitHub-Delivery` is unsigned; `X-GitHub-Event` is the sole
  event-type carrier and is unsigned, so **every consumer dispatches business
  logic on unsigned data**; legacy SHA-1 `X-Hub-Signature` still shipped
  alongside SHA-256 (downgrade surface); no documented secret-overlap
  rotation. Community discussions (#174434, #182735) converge on "roll your
  own replay protection."
- **Twilio** — SHA-1 (Twilio maintains a standing docs rebuttal, itself
  evidence the criticism is persistent); signing the *exact full URL* is
  brittle behind proxies and load balancers that rewrite
  scheme/host/port/path, and empty params must be preserved. That brittleness
  is a **correctness** criticism that drives operators to disable
  validation — the T10 pathway. Plain HTTP callbacks are permitted. AuthToken
  doubles as the account API credential.
- **Stripe** — the `constructEvent` raw-body pitfall is a long, live issue
  trail on `stripe/stripe-node` (#356, #494, #1254, #1294):
  `express.json()`/body-parser re-serialization silently breaks the HMAC.
  Second recurring cause: test-mode secret against live-mode events. Docs now
  warn explicitly (*"Any manipulation to the raw body of the request causes
  the verification to fail"*); Standard Webhooks generalizes it (*"even a
  stray space can cause the signature to be invalid"*). The `v0` test scheme
  requires explicit downgrade discipline.
- **Shopify** — the HMAC key is the **app-wide client secret**, so one leaked
  secret forges for every merchant; shop domain, topic and triggered-at all
  unsigned; no replay bound.
- **Microsoft Graph** — no payload integrity by default; `clientState`
  compromised by a single observed notification (any log line, any APM
  trace); no constant-time guidance for comparing it.
- **Cross-cutting (Standard Webhooks)** — *"every webhooks provider
  implements them differently and with varying quality"*; providers
  *"reinvent the wheel every time and repeat the same costly mistakes"*;
  *"these incompatibilities mean that no tools are being built to help
  senders send, consumers consume."*
- **The criticism that dominates empirically** — all six documented failures
  located are **receiver-side verification failures**; **zero** are
  cryptanalytic breaks of a vendor scheme. Five were in receiver-written
  integration code (WPForms, new-api, n8n, Jenkins plugin, Cable's target);
  one — Clerk CVE-2025-53548 — was in **the provider's own SDK helper**, the
  more damning case, because every consumer who did the right thing and used
  the vendor library was still vulnerable.
  `[INFERENCE, moderate-high — six cases is a sample, not a census.]` **The
  scheme is not where webhook security fails.**

## 5. RFC 9421 delta — adds coverage and agility, misses the body

RFC 9421, Standards Track, February 2024 (Backman/Richer/Sporny). Deployment
status is already settled in-repo by `baseline-03b`; not re-derived.

### What it adds over a well-designed HMAC with signed timestamp

| Capability | RFC 9421 mechanism | Best vendor equivalent |
|---|---|---|
| Signed creation time | `created` (RECOMMENDED) | Stripe `t=`, SW `webhook-timestamp` |
| **Signer-asserted expiry** | `expires` | **none** — no surveyed scheme lets the signer bound its own signature |
| Per-signature uniqueness | `nonce`; §2.5 *"Enforcing uniqueness of the nonce parameter"* | SW `msg_id` (approximate) |
| Key identification / rotation | `keyid` | Stripe/SW multi-signature convention (bespoke) |
| Algorithm agility + downgrade defense | `alg` + HTTP Signature Algorithms registry + §7.3.6 | Stripe's `v1`-only rule (bespoke) |
| Application-profile identification | `tag` | **none** — prevents cross-protocol signature confusion |
| **Explicit covered-component list** | `Signature-Input`; §7.2.1 names "Insufficient Coverage" as a threat | **none** — closes the GitHub/Shopify unsigned-metadata gap directly |
| Destination binding | `@target-uri`, `@authority` | Twilio's URL signing (brittle, non-standard) |
| Guidance on what *not* to sign | §7.2.3 — `Via`/`Forwarded` and intermediary-rewritten components | **none** — the spec anticipates exactly what makes Twilio brittle |
| Asymmetric option, with warning | §7.3.3: with symmetric crypto *"a verifier is capable of generating a valid signature… An attacker that is able to compromise a verifier would be able to then impersonate a signer"* | SW `v1a` ed25519 |
| Standardized parsing | one grammar | N incompatible vendor formats |
| Profile obligation | §2.5: applications MAY add requirements, *"it MUST enforce them during the signature verification process, and signature verification MUST fail if the signature does not conform"* | — |

§7.2.2 (Signature Replay) is the spec's own statement of the T3/T4 problem,
verbatim: *"Since HTTP message signatures allow sub-portions of the HTTP
message to be signed, it is possible for two different HTTP messages to
validate against the same signature… Even with sufficient component coverage,
a given signature could be applied to two similar HTTP messages, allowing a
message to be replayed by an attacker with the signature intact. To
counteract these kinds of attacks, it's first important for the signer to
cover sufficient portions of the message to differentiate it from other
messages. In addition, the signature can use the nonce signature parameter…
the signer can provide a timestamp for when the signature was created and a
time at which the signer considers the signature to be expired, limiting the
utility of a captured signature value."*

### What it costs — and the finding that matters most

**RFC 9421 does not sign the body.** §1, verbatim: *"this specification does
not define a means to directly cover HTTP message content (defined in
Section 6.4 of [HTTP]); rather, it relies on the Digest specification
[DIGEST] to provide a hash of the message content, as discussed in Section
7.2.8."*

`[INFERENCE, high confidence]` For webhooks — where body integrity **is** the
requirement — RFC 9421 alone is not a webhook signature scheme. It must be
paired with `Content-Digest` (RFC 9530) as a covered component. **A consumer
who implements 9421 without Content-Digest has signed metadata and nothing
that matters.** This is a two-RFC stack and a genuine footgun, and it is
**not** covered by `baseline-03b`, which tested only deployment.

Other costs: canonicalization surface (structured-field serialization
`sf`/`key`/`bs`, derived components, field-combination rules) versus
`HMAC(t + "." + raw_body)`; §7.2.3's intermediary-rewrite problem generalizes
Twilio's brittleness to any component chosen badly; §1 concedes deployments
*"may be running in environments that do not provide complete access to or
control over HTTP messages."*

**Verdict:** RFC 9421 addresses the threat model **better** on coverage,
agility, key identification and replay parameters, and **worse** on the
single property webhooks most need, unless explicitly paired with RFC 9530.
It is a better *framework*, not a better *drop-in*. This supports keeping
OP-016 at "prefer," while making the Content-Digest pairing mandatory inside
the standard's 9421 profile.

## 6. Net input for OP-016

**Current text:** *"Sign every outbound webhook. Prefer RFC 9421; otherwise
HMAC-SHA256 over a base that includes delivery ID, timestamp, and raw
body."*
**The three-element base is confirmed correct** — each element maps to a
distinct threat (raw body → T1/T2, timestamp → T3, delivery ID → T3/T4).

### Invariants the evidence requires of any signing scheme

- **I1 Raw-byte base.** Sign the body exactly as received on the wire; never
  a parsed or re-serialized form.
- **I2 Signed timestamp with an enforced, bounded, non-zero tolerance.**
  5 minutes is the field's convergent default (Stripe libs, SW libs, OWASP
  draft, webhooks.fyi). Stripe: *"Don't use a tolerance value of `0`."*
  Stripe regenerates the timestamp per retry, so a tight window does not
  conflict with retries.
- **I3 Signed unique delivery ID, plus consumer dedupe for at least the
  tolerance window.** The ID must be **inside** the signature — GitHub's is
  not, which is why its dedupe advice is unsound against an active attacker.
- **I4 Everything the consumer dispatches or authorizes on must be inside
  the signature** — event type, tenant identity, destination. If it lives in
  an unsigned header, either sign it or do not act on it. *(Closes the GitHub
  and Shopify gaps.)*
- **I5 Constant-time comparison, normative.** CVE-2022-36885 proves it is
  not theoretical.
- **I6 Per-endpoint secrets** (not per-account, not per-app), ≥256 bits,
  overlapping validity, multiple simultaneous signatures during cutover,
  procedure published. *(Check overlap with existing OP-024 rather than
  duplicating.)*
- **I7 Fail closed on a missing, empty, or default secret** — reject at
  configuration load, not at verify time. CVE-2026-41432: an empty key still
  produces a computable HMAC.
- **I8 Prohibit SHA-1, for the right reason** — NIST retires SHA-1 for all
  applications on 2030-12-31 and SHA-256 is free; *not* "collisions break
  HMAC-SHA1."
- **I9 Reject unknown or legacy schemes rather than falling back** (Stripe's
  `v1`-only rule; RFC 9421 §7.3.6).
- **I10 Verification must be the default path, not an appendix** — raw-body
  access documented, SDK helper on the primary example, published test
  vectors, and the helper itself tested against negative vectors (Clerk
  shipped a broken one). All six located failures were receiver-side; a
  scheme hard to implement correctly is, in effect, insecure.
- **I11 If RFC 9421 is used, `Content-Digest` (RFC 9530) MUST be a covered
  component.**
- **I12 State the boundaries** — a signature is authentication, not
  authorization; it does not replace TLS, provide confidentiality, or address
  provider-side SSRF.
- **I13 Deliver over HTTPS only, and require it.** Stripe requires it in
  live mode; Twilio does not. Without TLS, capture-and-replay becomes a
  passive network attack, converting T3 from defense-in-depth into a live
  concern.

### Graded, not invariant

Asymmetric signing (ed25519/RSA) where a compromised consumer must not be
able to forge to others (RFC 9421 §7.3.3) · thin payloads as a deliberate
lever — the weaker the payload integrity, the thinner the payload (Graph's
implicit design) · IP allowlist and mTLS as defense-in-depth only; Stripe
pairs allowlisting **with** signatures.

### Proposed amendments to OP-016

1. Extend the signed base to *"delivery ID, timestamp, raw body, **and any
   metadata the consumer is expected to act on**."* (I4)
2. Make the RFC 9421 branch **conditional on Content-Digest coverage**.
   (I11)
3. Add or pair **consumer-side obligations** — constant-time compare,
   enforced tolerance, dedupe, fail-closed on empty secret. OP-016 is
   currently provider-side only, and the evidence says failures are
   receiver-side. Check overlap with OP-017/OP-024 first.
4. Add an explicit **SHA-1 prohibition** with the retirement-date rationale.
   (I8)

### Confidence

**High:** threat enumeration; the invariants; the RFC 9421 Content-Digest
gap (RFC text verbatim); the HMAC-SHA1 verdict (RFC 6194 §3.3 + NIST +
Twilio's own rebuttal).
**Moderate-high:** "failures are receiver-side, not cryptanalytic" — six
documented cases, not a census.
**Moderate:** the strength of the T3 (replay) case. No documented
in-the-wild replay incident was found; the timestamp requirement rests on
defense-in-depth reasoning and unanimous vendor/spec practice rather than
incident evidence. **Say so in the decision record rather than
overstating it.**

---

## Sources

Vendor primary: [Stripe webhooks](https://docs.stripe.com/webhooks) ·
[GitHub validating deliveries](https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries) ·
[GitHub webhook headers](https://docs.github.com/en/webhooks/webhook-events-and-payloads) ·
[Shopify HTTPS webhooks](https://shopify.dev/docs/apps/build/webhooks/subscribe/https) ·
[Twilio security](https://www.twilio.com/docs/usage/security) ·
[Graph webhooks](https://learn.microsoft.com/en-us/graph/change-notifications-delivery-webhooks) ·
[Graph overview](https://learn.microsoft.com/en-us/graph/change-notifications-overview)

Spec: [RFC 9421](https://www.rfc-editor.org/rfc/rfc9421.txt) (§1, 2.3, 2.5,
7.1.2, 7.2.1, 7.2.2, 7.2.3, 7.3.3, 7.3.6) ·
[RFC 6194 §3.3](https://www.rfc-editor.org/rfc/rfc6194.txt) ·
[Standard Webhooks spec](https://github.com/standard-webhooks/standard-webhooks/blob/main/spec/standard-webhooks.md) ·
[standardwebhooks.com](https://www.standardwebhooks.com/)

Security: [NIST SHA-1 transition](https://csrc.nist.gov/news/2022/nist-transitioning-away-from-sha-1-for-all-apps) ·
[GHSA-xff3-5c9p-2mr4](https://github.com/advisories/GHSA-xff3-5c9p-2mr4) ·
[GHSA-9mp4-77wg-rwx9](https://github.com/clerk/javascript/security/advisories/GHSA-9mp4-77wg-rwx9) ·
[Jenkins 2022-07-27](https://www.jenkins.io/security/advisory/2022-07-27/) ·
[CVE-2026-4986 writeup](https://blog.himanshuanand.com/2026/07/reporter-11-10-people-found-the-wpforms-paypal-bug-before-me-cve-2026-4986/) ·
[CVE-2026-56357](https://www.vulncheck.com/advisories/n8n-webhook-forgery-via-missing-hmac-sha256-signature-verification-in-github-webhook-trigger) ·
[OWASP draft cheat sheet](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets_draft/Webhook_Security_Guidelines_Cheat_Sheet.md)
*(unpublished)* ·
[OWASP published index](https://cheatsheetseries.owasp.org/) *(absence
check)*

Secondary, labelled:
[Jack Cable 2018](https://lightningsecurity.io/blog/bypassing-payments-using-webhooks/) ·
[webhooks.fyi replay prevention](https://webhooks.fyi/security/replay-prevention)
*(Hookdeck/ngrok-supported)* · stripe/stripe-node issues
[#356](https://github.com/stripe/stripe-node/issues/356),
[#494](https://github.com/stripe/stripe-node/issues/494),
[#1254](https://github.com/stripe/stripe-node/issues/1254),
[#1294](https://github.com/stripe/stripe-node/issues/1294)
