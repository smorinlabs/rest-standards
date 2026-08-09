# Baseline 03g — Five Risk-Based Security Axes: Defaults and Threat-Model Triggers

*Narrow leaf under `baseline-03`, commissioned at Gate C when the owner
decided to research and ratify defaults for the five axes `baseline-03`
§8.2 classified as threat-model-conditional, housed in a deployment-profile
structure. Run 2026-08-09.*

Retrieval date **2026-08-09**. Every RFC quotation below was re-verified against
raw text downloaded from `www.rfc-editor.org`; every OWASP quotation was
extracted from raw HTML of the `owasp.org` page, not from a summarizer. Where a
finding came from a delegated survey, it is marked and the load-bearing ones
were independently re-fetched.

## Method and evidence caveats

- **Two independent extractions agreed** on the OWASP API1/API4/API5 "How To
  Prevent" text and on the Zanzibar abstract (my raw-HTML pull and a second
  agent's). Treat those quotes as high-confidence.
- **`WebSearch` quota was exhausted** (200/200) early in this session. All
  retrieval after that point was direct URL fetching. Consequence: coverage is
  good where documentation URLs are guessable and poor where they are not.
  Named gaps are listed per axis rather than papered over.
- **AWS live documentation pages are JavaScript-rendered and not machine-
  fetchable.** The AWS quotation used below comes from AWS's own documentation
  **source** repository (`awsdocs/amazon-s3-developer-guide`, archived), not the
  live page. Flagged rather than silently substituted.

### Source register (all verified by direct fetch, 2026-08-09)

| Document | Status | Date |
| --- | --- | --- |
| RFC 9700 (BCP 240) Best Current Practice for OAuth 2.0 Security | Best Current Practice; updates 6749, 6750, 6819 | January 2025 |
| RFC 9449 OAuth 2.0 Demonstrating Proof of Possession (DPoP) | Proposed Standard | September 2023 |
| RFC 8705 OAuth 2.0 Mutual-TLS Client Authentication and Certificate-Bound Access Tokens | Proposed Standard | February 2020 |
| RFC 9068 JWT Profile for OAuth 2.0 Access Tokens | Proposed Standard | October 2021 |
| RFC 7662 OAuth 2.0 Token Introspection | Proposed Standard | October 2015 |
| RFC 7009 OAuth 2.0 Token Revocation | Proposed Standard | August 2013 |
| RFC 9421 HTTP Message Signatures | Proposed Standard | (already ratified here) |
| RFC 9470 OAuth 2.0 Step Up Authentication Challenge Protocol | Proposed Standard | September 2023 |
| RFC 9701 JWT Response for OAuth Token Introspection | Proposed Standard | January 2025 |
| RFC 9728 OAuth 2.0 Protected Resource Metadata | Proposed Standard | April 2025 |
| FAPI 2.0 Security Profile (OpenID Foundation) | **Final**, Standards Track | published 22 February 2025 |
| OWASP API Security Top 10 | 2023 edition | 2023 |

`[FACT]` **The OWASP API Security Top 10 has exactly two editions, 2019 and
2023.** Verified on two surfaces: the editions navigation at
`owasp.org/API-Security/editions/2023/en/0x11-t10/` lists only those two, and
the project home page's newest news items are 2024 translations of the 2023
edition. There is no 2024, 2025, or 2026 edition and none is announced.

`[FACT]` **No RFC published since RFC 9449 defines a new sender-constraining
mechanism.** Established by scanning the authoritative `rfc-index.txt` for every
OAuth/token RFC numbered ≥ 9400. The newer relevant publications are RFC 9470
(step-up challenge), RFC 9701 (JWT introspection response), RFC 9728 (protected
resource metadata), and RFC 9901 (SD-JWT) — none of them a new proof-of-possession
scheme.

---

## Axis 1 — Sender-constrained tokens

### Options

1. **Plain bearer** over TLS (RFC 6750). Any holder can use the token.
2. **DPoP** (RFC 9449) — application-layer proof of possession; client signs a
   per-request JWT with a key it holds; works for public clients; survives TLS
   termination at a CDN.
3. **mTLS certificate-bound** (RFC 8705) — token bound to the fingerprint of the
   client's TLS certificate; requires mutual TLS all the way to the resource
   server.
4. **Full request signing** (AWS SigV4-style) — outside OAuth; covered in Axis 4.

### What the standards actually say

`[FACT]` **BCP 240 (RFC 9700) §2.2.1 — the access-token rule is `SHOULD`, not
`MUST`:**

> "Authorization and resource servers SHOULD use mechanisms for
> sender-constraining access tokens, such as mutual TLS for OAuth 2.0 [RFC8705]
> or OAuth 2.0 Demonstrating Proof of Possession (DPoP) [RFC9449] (see
> Section 4.10.1), to prevent misuse of stolen and leaked access tokens."

`[FACT]` **§2.2.2 — the only `MUST` is narrower, and it has an escape hatch:**

> "Refresh tokens for public clients MUST be sender-constrained or use refresh
> token rotation as described in Section 4.14."

`[FACT]` **§4.10 grants an explicit deployment exemption:**

> "Authorization servers therefore SHOULD ensure that access tokens are
> sender-constrained and audience-restricted as described in the following.
> Architecture and performance reasons may prevent the use of these measures in
> some deployments."

`[FACT]` **§4.9.3 states the specific threat sender-constraining answers:**

> "Sender-constrained access tokens, as described in Section 4.10.1, SHOULD be
> used to prevent the attacker from replaying the access tokens on other
> resource servers. If an attacker has only partial access to the compromised
> system, like a read-only access to web server logs, sender-constrained access
> tokens may also prevent replay on the compromised system."

`[FACT]` **§4.10.1 states the limit — this is the sentence that stops the
standard over-claiming:**

> "Note that the security of sender-constrained tokens is undermined when an
> attacker gets access to the token and the key material. This is, in
> particular, the case for corrupted client software and cross-site scripting
> attacks (when the client is running in the browser)."

`[FACT]` **RFC 8705 §1** on what certificate binding buys:

> "Mutual-TLS certificate-bound access tokens ensure that only the party in
> possession of the private key corresponding to the certificate can utilize the
> token to access the associated resources… Binding an access token to the
> client's certificate prevents the use of stolen access tokens or replay of
> access tokens by unauthorized parties."

`[FACT]` **FAPI 2.0 Security Profile (OIDF, Final, 22 February 2025) §5.3.2.1** —
the cleanest named trigger anchor, verified in the raw spec:

> "shall only issue sender-constrained access tokens;"
> "shall use one of the following methods for sender-constrained access tokens:
> MTLS as described in [RFC8705], DPoP as described in [RFC9449];"

`[COMPARATIVE]` FAPI **1.0 Part 2** (12 March 2021) required mTLS only and said
"MTLS is currently the only mechanism for sender-constrained access tokens that
has been widely deployed." FAPI 2.0 adds DPoP as a co-equal option. The
direction of travel over four years is *toward* DPoP acceptance, not away.

### Vendor practice

| Product / spec | DPoP (RFC 9449) | mTLS-bound (RFC 8705) | Status | Source |
| --- | --- | --- | --- | --- |
| Bluesky / atproto | **Mandatory** | n/a | normative in the protocol | `atproto.com/specs/oauth` |
| FAPI 2.0 profile | required (either) | required (either) | Final, Feb 2025 | `openid.net/specs/fapi-security-profile-2_0-final.html` |
| Auth0 | Yes, incl. public clients | Yes (`cnf` thumbprint) | GA/beta not stated | `auth0.com/docs/secure/sender-constraining` |
| Keycloak | Yes | not checked | preview since v23 (Nov 2023) | `keycloak.org/securing-apps/dpop` |
| Kong Gateway (OIDC plugin) | Yes (validates) | Yes (cert-bound) | GA | `developer.konghq.com/plugins/openid-connect/` |
| Spring Authorization Server | Yes | not checked | present in v1.5.8 | `docs.spring.io/spring-authorization-server/` |
| node-oidc-provider | Yes, **default-enabled** | not checked | GA | project docs |
| **Microsoft Entra ID** | **Absent** | **Absent** | proprietary PoP only | `learn.microsoft.com/en-us/entra/msal/dotnet/advanced/proof-of-possession-tokens` |
| Apigee, AWS API Gateway + Cognito | Absent in pages checked | Absent in pages checked | — | vendor docs |
| Okta, PingFederate, ForgeRock/PingAM, Curity (general), Cloudflare | **not verifiable this session** | — | see gaps | — |

`[FACT]` **Microsoft Entra ID, the largest enterprise IdP checked, ships
neither** — both quotes re-verified against raw page text on 2026-08-09.
Verbatim: *"PoP via mutual TLS (mTLS)."* / *"See RFC 8705 for details."* / *"No
support exists currently."* And on the token endpoint: *"Indicates the token type
value."* / *"The only type that Microsoft Entra ID supports is Bearer."*

`[FACT]` **But the same Entra page states an intention:** *"Future APIs will
rely on PoP via mTLS."* It also gives Microsoft's own comparison: *"mTLS is
faster and has the advantage of including man-in-the-middle protections at the
TLS layer; however, it can be difficult to establish mTLS tunnels between the
client and the identity provider and between the client and the resource."*
`[INFERENCE]` The absence is a current-state fact with a stated roadmap, not a
rejection. Treat "Entra ships neither" as dated evidence with a re-check
trigger, not a permanent property.

### The three AI platforms

| Platform | Credential | Header | OAuth surface | Sender-constrained? |
| --- | --- | --- | --- | --- |
| OpenAI | API key, or short-lived token via workload identity federation | `Authorization: Bearer` | OAuth 2.1 + PKCE for Apps SDK / MCP connectors | **No.** mTLS is used only to authenticate *ChatGPT as a client to MCP servers*, explicitly separate from the OAuth token |
| Anthropic | API key `sk-ant-api…`, or WIF bearer token, or App Attest token | `x-api-key`, or `Authorization: Bearer` | WIF token exchange; MCP OAuth 2.1 for connectors | **No.** All token types are plain bearer |
| Google Gemini | API key; OAuth 2.0 client for stricter access control | `x-goog-api-key` | OAuth 2.0 installed-app flow | **No** |

`[FACT]` OpenAI's Apps SDK documentation draws the distinction explicitly: *"Use
mTLS to authenticate ChatGPT as the MCP client. Continue to use OAuth 2.1 to
authenticate the end user and authorize tool access."* This is transport client
authentication, **not** RFC 8705 token binding.

`[FACT]` **The Model Context Protocol authorization spec (2025-11-25 revision)
mandates plain bearer** — *"MCP client **MUST** use the Authorization request
header field… `Authorization: Bearer <access-token>`"* — and mentions neither
DPoP nor mTLS. Its extensions repository lists only "Enterprise-Managed
Authorization" and "Client Credentials"; no proof-of-possession extension
exists. `[INFERENCE]` The agent/connector ecosystem being built right now by all
three AI vendors is standardising on unconstrained bearer tokens.

### Recommended default

> **Default: plain bearer over TLS, paired with the three cheap controls BCP 240
> actually leans on — short access-token TTL, audience restriction, and refresh
> token rotation for public clients. Do not require DPoP or mTLS by default.**
>
> **Rider (`SHOULD`): do not hard-code the `Bearer` scheme in token validation.**
> Accept the `DPoP` authentication scheme as a configuration, so that flipping
> this axis is an operational change rather than a redesign.

**Confidence: moderate-high.** Three converging reasons:

1. `[FACT]` BCP 240 itself only says `SHOULD` for access tokens, and explicitly
   admits "architecture and performance reasons may prevent the use of these
   measures." The single `MUST` — public-client refresh tokens — is satisfiable
   by **rotation**, which requires no client cryptography.
2. `[COMPARATIVE]` The deployable ecosystem is not there. The largest enterprise
   IdP ships neither mechanism; four of the surveyed gateways/IdPs could not be
   confirmed to support either; all three AI platforms and the MCP spec are
   plain bearer. A `MUST` here would put this standard ahead of what its
   implementers can buy.
3. `[FACT]` The protection is narrower than it sounds. BCP 240 §4.10.1 says the
   guarantee collapses under XSS and corrupted client software — precisely the
   public-client threat people reach for DPoP to solve.

### Threat-model triggers that flip the default

| Trigger | Flip to | Evidence |
| --- | --- | --- |
| Financial-grade / open-banking / payment-initiation, or any ecosystem asserting FAPI 2.0 conformance | **Required** sender-constraining, DPoP **or** mTLS | FAPI 2.0 §5.3.2.1 "shall only issue sender-constrained access tokens" `[FACT]` |
| Access tokens traverse intermediaries that log, or are observable in server logs / APM | Sender-constrain | BCP 240 §4.9.3 read-only-log-access scenario `[FACT]` |
| One token is usable at multiple resource servers and audience restriction is not feasible | Prefer **mTLS** — it lets one token work across resource servers where audience restriction would force one token per server | BCP 240 §4.10.2 `[FACT]` |
| Public clients (SPA, mobile) in hostile environments, where exfiltration via network/log is the concern | **DPoP** — works without PKI, survives CDN TLS termination | RFC 9449 §1 `[FACT]`; but see the XSS caveat — do not claim it defeats a compromised client `[FACT]` |
| Confidential server-to-server clients where PKI already exists | **mTLS** | RFC 8705 §2 `[FACT]` |
| Only *some* operations are high-value (large transfers, credential changes, admin actions) | **Do not flip the whole API.** Use RFC 9470 step-up challenge per operation | RFC 9470 abstract `[FACT]` |
| Client is a browser app whose main threat is XSS | **Sender-constraining is the wrong control.** Reduce TTL, isolate the token from page script, use a backend-for-frontend | BCP 240 §4.10.1 `[FACT]` |

---

## Axis 2 — Opaque vs self-contained (JWT) access tokens

### Options

1. **Opaque / reference token** — a handle; resource server calls introspection
   (RFC 7662) or a shared session store.
2. **Self-contained JWT** — validated locally against a JWKS; no round trip.
3. **Phantom token** — opaque on the public wire, swapped for a JWT at the
   gateway.
4. **Split token** — signature held at the gateway, payload sent to the client.

### What the standards say

`[FACT]` **RFC 7009 §3 is the standards-level statement of this exact trade-off**
— and notably it is a *design-choice* note, not a requirement:

> "In the latter case [handles], the authorization server is able to revoke an
> access token previously issued to a client when the resource server relays a
> received access token. In the former case [self-contained], some (currently
> non-standardized) backend interaction between the authorization server and the
> resource server may be used when immediate access token revocation is desired.
> Another design alternative is to issue short-lived access tokens, which can be
> refreshed at any time using the corresponding refresh tokens."

> "Which approach of token revocation is chosen will depend on the overall system
> design and on the application service provider's risk analysis. The cost of
> revocation in terms of required state and communication overhead is ultimately
> the result of the desired security properties."

`[FACT]` **RFC 9068 states the benefit of the JWT side** — resource servers can
consume claims *"directly for authorization or other purposes without any
further round trips to introspection ([RFC7662]) or UserInfo ([OpenID.Core])
endpoints."*

`[FACT]` **RFC 9068 imposes real obligations if you choose JWT:** *"JWT access
tokens MUST be signed"*; *"JWT access tokens MUST NOT use 'none' as the signing
algorithm"*; *"JWT access tokens MUST include this media type in the 'typ' header
parameter… Therefore, the 'typ' value used SHOULD be 'at+jwt'"*; and on the
receiving side, *"The resource server MUST verify that the 'typ' header value is
'at+jwt' or 'application/at+jwt' and reject tokens carrying any other value."*

`[FACT]` **RFC 9068 defines no revocation mechanism at all.** Explicit absence,
confirmed by grep of the full text.

`[FACT]` **RFC 9068 §6 is the under-cited argument against putting JWTs on a
public wire:**

> "As JWT access tokens carry information by value, it now becomes possible for
> clients and potentially even end users to directly peek inside the token claims
> collection of unencrypted tokens."
> "The client MUST NOT inspect the content of the access token: the authorization
> server and the resource server might decide to change the token format at any
> time (for example, by switching from this profile to opaque tokens)…"
> "Whenever client access to the access token content presents privacy issues for
> a given scenario, the authorization server needs to take explicit steps to
> prevent them."

`[FACT]` **RFC 9701 (January 2025)** now standardises a JWT-secured response for
introspection — i.e. the introspection path has been getting *more* standardised,
not less, since the phantom-token pattern was named.

### Vendor practice

| Vendor | Default access-token format | What flips it | Introspection | Documented TTL | RFC 9068 |
| --- | --- | --- | --- | --- | --- |
| **Auth0** | **Opaque** when no `audience` is requested (usable only at `/userinfo`); **JWT** for the Management API and any registered custom API | requesting an `audience` | not documented on pages checked | **86400 s (24 h)** for a custom API | **Yes** — API Settings exposes a "JWT Profile" control with values `Auth0` and `RFC 9068` |
| **Okta** | **JWT**; refresh tokens opaque; Org-AS structure "is subject to change" | n/a | yes, `/introspect` | Org AS **60 min**; Custom AS 5 min–24 h | not found |
| **Microsoft Entra ID** | Nominally JWT, but *"they should treat them as opaque strings"*; Graph tokens are a proprietary non-JWT format | API registration version; Microsoft-owned vs third-party APIs | not documented | **discrepant across two Microsoft pages** — 60–90 min ("75 minutes on average") vs "one hour"; **20–28 h** with CAE | not found |
| **Google Cloud** | **Opaque** — user, service-account, domain-wide-delegation, federated and credential-access-boundary tokens are all opaque | self-signed service-account JWTs are a separate client-issued mechanism | `tokeninfo` referenced, not quotable | not published | n/a |
| **Keycloak** | **JWT** | — | yes, RFC 7662 endpoint | realm setting, default not extractable | **Yes but off by default** — `at+jwt` header type is an opt-in admin setting |
| **AWS Cognito** | **JWT always**; only refresh tokens opaque | not applicable | **none** — JWKS validation only | **1 hour** default, 5 min–1 day configurable | not found |
| **Spring Authorization Server** | **JWT** (`SELF_CONTAINED` is the coded default) | `accessTokenFormat(REFERENCE)` | yes | **5 minutes** (`Duration.ofMinutes(5)`) | not found in source checked |
| **Ping Identity** | **not discoverable** — six candidate doc URLs 404'd or redirect-looped | — | — | — | — |

`[FACT]` **RFC 9068 conformance is opt-in even where it is supported.** Keycloak
26.2.0 release notes, verbatim: *"If enabled, access tokens will get header type
`at+jwt` in compliance with rfc9068#section-2.1. Otherwise, the access token
header type will be `JWT`. **This setting is turned off by default.**"* Auth0
exposes the same choice as a per-API "JWT Profile" toggle between `Auth0` and
`RFC 9068`. `[INFERENCE]` A standard that requires RFC 9068 conformance is
requiring a non-default configuration from at least two major vendors — state
that as an implementation note rather than assuming it comes free.

`[FACT]` **Even the JWT vendors tell clients to treat tokens as opaque.**
Microsoft Entra, verbatim: *"Although client applications can receive and use
access tokens, they should treat them as opaque strings… Tokens that a Microsoft
API receives might not always be a JWT that can be decoded."* This matches RFC
9068 §6's "The client MUST NOT inspect the content of the access token."

`[FACT]` **On the Auth0 opaque/JWT rule:** the `access-tokens` page states only
*"Access tokens issued for the Management API and access tokens issued for any
custom API that you have registered with Auth0 follow the JWT standard."* The
audience-triggers-JWT behaviour is documented on the separate `get-access-tokens`
page. Both were fetched; cite the page that actually carries the sentence you
rely on.

`[FACT]` **The phantom-token pattern, from its source (Curity, published
2020-03-27), re-verified against the page's raw HTML:** *"The Phantom Token
Approach is a prescriptive pattern for securing APIs and microservices that
combines the security of opaque tokens with the convenience of JWTs."*
Mechanism: *"The reverse proxy looks up the by-value token by calling the
Introspection endpoint of the Token Service"* and *"replaces the by-reference
token with the by-value token in the actual request to the microservice."*

`[Note]` Two independent retrievals of this page returned **different** wordings
of the definition sentence; the one above is the one present in the raw HTML.
This is a live example of why the project bans summarizer-mediated normative
quotes — re-verification changed the text, not just the confidence.

### The revocation-latency reality, measured

`[FACT]` Microsoft Entra's Continuous Access Evaluation documentation (updated
2026-04-24) is the most honest published account of what it costs to get
revocation out of self-contained tokens:

> "Microsoft experimented with the 'blunt object' approach of reduced token
> lifetimes but found they degrade user experiences and reliability without
> eliminating risks."

> "The goal for critical event evaluation is for response to be near real time,
> but latency of up to 15 minutes might be observed because of event propagation
> time; however, IP locations policy enforcement is instant."

> "Token lifetime increases to long-lived, up to 28 hours, in CAE sessions.
> Critical events and policy evaluation drive revocation, not just an arbitrary
> time period."

`[INFERENCE]` The largest deployment of self-contained access tokens in the
industry solved revocation not by shortening TTL but by building an out-of-band
revocation-signal channel (CAEP) — and still documents up to 15 minutes of
propagation. A standard that says "use JWTs and keep the TTL short" is
prescribing the approach Microsoft explicitly reports as having failed for them.

`[FACT]` **AWS Cognito states the failure mode more bluntly than any other
vendor, and this is the single most useful quote on this axis** (re-verified
against raw page text 2026-08-09 — the Cognito developer guide is
server-rendered and does fetch, unlike the S3 API reference):

> "User pool JWTs are self-contained with a signature and expiration time that
> was assigned when the token was created. Revoked tokens can't be used with any
> Amazon Cognito API calls that require a token. However, **revoked tokens will
> still be valid if they are verified using any JWT library that verifies the
> signature and expiration of the token.**"

`[COMPARATIVE]` **The opaque side has published, measured revocation latency and
it is fast.** OpenAI, verbatim: *"Revocations of an API key take effect within a
few seconds. Most updates that affect authentication results of an API key
propagate within 15 minutes, but can potentially take longer."* Set that beside
Cognito's "revoked tokens will still be valid" and Entra's "up to 15 minutes"
*with* a purpose-built revocation channel: the opaque architecture reaches in
seconds what the JWT architecture reaches in minutes and only after extra
machinery.

`[FACT]` Curity states the general principle: *"JWTs are self-contained,
by-value tokens and it is very hard to revoke them, once issued and delivered to
the recipient."*

`[FACT]` **Absence:** no vendor documentation among those fetched (Auth0, Okta,
Keycloak, AWS API Gateway) contains an explicit sentence of the form "access
tokens cannot be revoked before expiry." Curity's "very hard to revoke" and
Cognito's "will still be valid" are the closest published statements.

### Gateway support for the phantom-token swap

| Gateway | Introspect-then-forward-JWT | Evidence |
| --- | --- | --- |
| **Kong** | **Yes** — `openid-connect` plugin supports introspection as an auth method distinct from stateless JWT validation (the term "phantom token" does not appear) | `developer.konghq.com/plugins/openid-connect/` |
| **NGINX** | **Yes, via a Curity-published module**, not NGINX Inc.'s own docs — `nginx-lua-phantom-token-plugin`: *"introspect opaque access tokens and forward JWT access tokens to APIs"* | `github.com/curityio/nginx-lua-phantom-token-plugin` |
| **Envoy** | **No native support.** `ext_authz` is generic (could implement it); the `oauth2` filter forwards the bearer token unchanged | Envoy filter docs |
| **Istio** | **No** — `RequestAuthentication` covers JWKS-based JWT validation only | `istio.io` authz-jwt task |
| **AWS API Gateway** | **No native support** — the JWT authorizer validates locally against `jwks_uri` and never calls introspection; a Lambda authorizer could be hand-built | `docs.aws.amazon.com/.../http-api-jwt-authorizer.html` |
| **Apigee** | **Inconclusive** — a "Using third-party OAuth tokens" page exists but is client-side rendered and unreadable | URL confirmed, body not retrievable |
| **Cloudflare, Gloo/Solo.io** | **not discoverable** at URLs checked | — |

`[COMPARATIVE]` This tempers the phantom-token recommendation: of the gateways
checked, exactly one (Kong) documents the introspection half natively, and the
best-known NGINX implementation is published by Curity — the pattern's own
author — rather than by NGINX. `[INFERENCE]` Phantom token is a sound
architecture but not a commodity gateway feature; treat adopting it as
integration work, not configuration.

### The three AI platforms

`[FACT]` All three issue **opaque API keys** as the primary credential
(`sk-ant-api…` for Anthropic in `x-api-key`; bearer API keys for OpenAI;
`x-goog-api-key` for Gemini). None publishes a JWT access-token format for its
main REST surface. Their OAuth/agent surfaces (OpenAI Apps SDK/MCP, Anthropic
Workload Identity Federation and App Attest) issue "short-lived access tokens"
whose **format is not stated in the documentation** — an explicit absence
checked on both vendors' authentication pages. `[INFERENCE]` That silence is
itself consistent with RFC 9068 §6: a caller is not meant to parse the token, so
the vendor has no reason to publish its format.

`[FACT]` Anthropic documents **expiry**, not revocation latency: *"After a key
expires, requests made with it return a `401 authentication_error`… expired keys
cannot be reactivated."* No propagation time is published — absence.
`[FACT]` Google documents *"fast-acting leaked key enforcement that quickly
stops the usage of leaked keys detected by our systems"* with no numeric latency.
`[COMPARATIVE]` Only OpenAI publishes a number ("within a few seconds" /
"within 15 minutes"), and all three are opaque-key architectures.

### Recommended default

> **Default: opaque (reference) tokens on the public wire. If services behind the
> trust boundary need claims, use the phantom-token pattern — introspect once at
> the edge and forward an RFC 9068 `at+jwt` internally. Never put a self-contained
> token in a client's hands as the default.**
>
> When a JWT *is* issued to clients (see triggers), it MUST conform to RFC 9068:
> signed, never `alg: none`, `typ` of `at+jwt`, `aud` validated by the resource
> server — and it MUST be paired with a revocation-propagation mechanism, because
> RFC 9068 supplies none.

**Confidence: moderate.** The field genuinely splits (Auth0/Okta/Entra ship JWTs;
Google and all three AI platforms ship opaque). The default rests on three
arguments:

1. `[FACT]` RFC 7009 §3 identifies the handle style as the one where "the
   authorization server **is able to** revoke." For a public API, revocation is a
   security-visible promise — a leaked credential must die on command.
2. `[FACT]` RFC 9068 §6: a JWT on the wire is readable by every client and end
   user. That is a privacy surface an opaque token simply does not have.
3. `[COMPARATIVE]` It is consistent with what this standard has already ratified —
   API keys for server-to-server single-trust are opaque by construction, and
   all three mandated AI comparators ship opaque keys.

### Threat-model triggers that flip the default

| Trigger | Flip to | Evidence |
| --- | --- | --- |
| Contractual or regulatory **instant-revocation** requirement (offboarding SLA, incident response, "revoke within N seconds") | Stay **opaque** (introspection revokes at the event), or add an explicit revocation-propagation mechanism. A short JWT TTL is only a **bounded-exposure fallback** — it bounds worst-case validity *after* a revocation event at the full TTL, it does not revoke; genuinely "instant" requirements cannot be met by TTL alone | RFC 7009 §3 `[FACT]`; Entra's measured ≤15-min CAE propagation `[FACT]` |
| Many services validating at high volume, **measured** introspection latency/availability is the bottleneck, and services span administrative domains so a shared session store is unavailable | **JWT (RFC 9068)** with short TTL | RFC 9068 §2.2.3 `[FACT]` |
| Validation must survive an authorization-server outage, or run at the edge / partitioned | **JWT** | `[INFERENCE]` from RFC 9068's no-round-trip property |
| Resource servers operated by third parties you cannot issue introspection credentials to | **JWT** + RFC 9728 protected-resource metadata | RFC 9728 `[FACT]` |
| You have an API gateway in the path | **Phantom token** — takes both properties, and is the reason to prefer it over a straight choice | Curity pattern `[FACT]` |
| Token would carry tenant IDs, internal roles, entitlements, or PII | Stay **opaque** — a JWT exposes all of it to the client and end user | RFC 9068 §6 `[FACT]` |
| You choose JWT and need revocation anyway | Add a CAEP-style revocation-signal channel; **budget for propagation lag, do not assume immediacy** | Entra CAE `[FACT]` |

---

## Axis 3 — Rate-limit aggressiveness (posture, not header format)

Header format is already ratified (429 + `Retry-After` `MUST`; draft-11
`RateLimit` fields `SHOULD`). This axis is about the *numbers and their shape*.

### What the standards say

`[FACT]` **OWASP API4:2023 prescribes the control and explicitly refuses to
prescribe the number:**

> "Implement a limit on how often a client can interact with the API within a
> defined timeframe (rate limiting)."
> "Rate limiting should be fine tuned based on the business needs. Some API
> Endpoints might require stricter policies."

It also asks for `[FACT]`: *"Limit/throttle how many times or how often a single
API client/user can execute a single operation (e.g. validate an OTP, or request
password recovery without visiting the one-time URL)"*, *"Define and enforce a
maximum size of data on all incoming parameters and payloads"*, and *"Configure
spending limits for all service providers/API integrations."* **No numeric value
appears anywhere on the page.**

`[FACT]` **OWASP API2:2023 supplies the auth-endpoint rule verbatim** — this is
the primary-source basis for a stricter tier, and it is a comparative
requirement, not an absolute one:

> "Implement anti-brute force mechanisms to mitigate credential stuffing,
> dictionary attacks, and brute force attacks on your authentication endpoints.
> This mechanism should be stricter than the regular rate limiting mechanisms on
> your APIs."

> "Credential recovery/forgot password endpoints should be treated as login
> endpoints in terms of brute force, rate limiting, and lockout protections."

`[FACT]` API2:2023 also corroborates this standard's already-ratified auth
boundary: *"API keys should not be used for user authentication. They should
only be used for API clients authentication."*

### Vendor practice — published numbers

| Vendor | Sustained limit | Scope | Burst mechanism |
| --- | --- | --- | --- |
| **Stripe** (self-verified) | **100 rps** live, **25 rps** sandbox global; **25 rps** per individual endpoint | per account; also per endpoint | no vendor-side burst published; recommends client-side token bucket |
| Stripe read allocation | *"must not exceed an average of 500 [reads] per transaction"* over rolling 30 days, floor 10,000/month; *"Write API requests have no allocation limit"* | per account | — |
| **GitHub** | **5,000 req/hr** authenticated (15,000 Enterprise Cloud); **60 req/hr** unauthenticated per IP; App installations 5,000–12,500/hr | per token / per IP | — |
| GitHub secondary limits | *"No more than 100 concurrent requests"*; *"No more than 900 points per minute"* (GET=1, mutating=5); *"No more than 90 seconds of CPU time per 60 seconds of real time"* | per user/app | cost-weighted |
| **Shopify** | GraphQL 100 pts/s Standard → 2,000 pts/s Enterprise; REST bucket 40 req, leak 2 req/s | per app + store | **leaky bucket**, numeric for REST |
| **Slack** | Tier 1 *"1+ per minute"* → Tier 4 *"100+ per minute"* | per method, per workspace | *"Slack does not share precise burst limits externally"*; design target 1 req/s |
| **Discord** | *"All bots can make up to 50 requests per second"* | per bot | per-route buckets via headers; declines to publish fixed numbers |
| **Atlassian Jira Cloud** | 65,000–500,000 points/hr by plan; burst 100 rps GET/POST, 50 rps PUT/DELETE | per tenant | token bucket |
| **AWS API Gateway** | *"10,000 requests per second (RPS) with an additional burst capacity… using a maximum bucket capacity of 5,000 requests"* | per account per Region | **token bucket**, explicit |

`[FACT]` **The widely-repeated "Stripe: 100 read + 100 write per second" is not
current Stripe documentation.** Verified independently twice on 2026-08-09.
Stripe publishes a global 100 rps (live) / 25 rps (sandbox), a 25 rps
per-endpoint default, resource-specific limits, a **separate concurrency
dimension**, and a *read-request allocation* measured per transaction over 30
days. Anyone citing the old formulation is citing folklore.

`[COMPARATIVE]` Stripe's `Stripe-Rate-Limited-Reason` header takes the values
`global-rate`, `endpoint-rate`, `global-concurrency`, `endpoint-concurrency`,
`resource-specific` — direct evidence that a mature public API's posture is
**multi-dimensional**, not one number.

### Auth-endpoint tiers — published evidence

| Vendor | Endpoint | Limit |
| --- | --- | --- |
| **Auth0** | Change Password | burst 10, sustained **1/minute** |
| Auth0 | User Info | burst 10, sustained **5/minute** |
| Auth0 | global Authentication API | 100/second |
| Auth0 | same-user login throttle | *"If one IP address makes 20 login attempts in one minute to the same user account… Auth0 allows the user 10 attempts per minute."* |
| Auth0 | brute-force protection default | threshold **10** failed attempts (IP + user identifier); block expires 30 days after last failure |
| **Okta** | `/api/v1/authn` and `/oauth2/v1/token` | **4/second** per username, stated to be *"to prevent brute force attacks"* |
| **Cloudflare** (recommended rules) | login | 4/min → challenge; 10/10min → challenge; 20/hour → block 1 day |
| Cloudflare | OTP endpoint | 5/min → block 10 min (counting 401/403 only) |
| **Supabase Auth** | `/verify` 360/hr (burst 30); `/token` 1800/hr (burst 30); MFA 15/hr; signup email 2/hr | per IP or project |
| **Discord** | 401/403/429 responses | *"this limit is 10,000 per 10 minutes"* — then a Cloudflare ban |

`[COMPARATIVE]` Auth0's Change Password endpoint at 1/minute sits **6,000×
below** its own 100/second general authentication ceiling. The gap between
ordinary and auth-surface limits in the field is orders of magnitude, not a
modest tightening.

### The three AI platforms

| Platform | Tier structure | Qualification | Representative numbers |
| --- | --- | --- | --- |
| **OpenAI** | Free, Tier 1–5 | explicit spend ladder: *"$5 paid"* → *"$1,000 paid"*; *"As your spend on our API goes up, we automatically graduate you to the next usage tier."* | GPT-4o: Tier 1 500 RPM / 30,000 TPM → Tier 5 10,000 RPM / 30,000,000 TPM |
| **Anthropic** | Evaluation, Start, Build, Scale, Custom | **not** a spend ladder: *"Organizations are placed on a tier automatically based on usage history and account standing"*; the $500 / $1,000 / $200,000 figures are monthly **spend caps**, not entry criteria | Claude Sonnet 5: Start 1,000 RPM / 2M ITPM / 400k OTPM → Scale 10,000 RPM / 10M ITPM / 2M OTPM. *"The API uses the token bucket algorithm… your capacity is continuously replenished up to your maximum limit, rather than being reset at fixed intervals."* |
| **Google Gemini** | Free, Tier 1–3 | spend **and tenure** gated: Tier 2 requires *"Paid $100 + 3 days from first successful payment"*; Tier 3 *"Paid $1,000 + 30 days"*. Rolling 10-minute spend caps ($10/10min Tier 1, $200/10min Tiers 2–3) | **Per-model RPM/TPM/RPD not published** — *"Rate limits… can be viewed in Google AI Studio"*, behind auth |

`[COMPARATIVE]` All three tie quota to **spend**, and two of the three split
quota into **requests and tokens** as separate dimensions — because their unit
cost per request varies by orders of magnitude. This is the clearest field
evidence that request counting alone is the wrong primitive when per-request
cost is variable.

`[FACT]` Gemini is a documented **absence**: the concrete per-model numbers are
not in public documentation at all. (Consistent with this repo's prior finding
that Gemini emits no rate-limit headers either.)

### Recommended default posture

> **Default: a published, multi-dimensional, tiered posture — not a single
> number. Specifically:**
>
> 1. **Per-authenticated-principal sustained limit with a burst allowance**,
>    implemented as a token bucket. Starting point for a typical public SaaS:
>    **100 req/s sustained per account, burst bucket ≈ 2× sustained drained over
>    ≤10 s**, and a **per-endpoint default of 25 req/s**. `[POLICY]` — these
>    numbers are Stripe-shaped, chosen because Stripe is the best-documented
>    general-purpose public API; they are a starting point owners tune, not a
>    protocol requirement.
> 2. **Unauthenticated traffic limited separately and per-IP, an order of
>    magnitude lower.** GitHub's 60/hr against 5,000/hr authenticated is the
>    reference ratio.
> 3. **Authentication-surface endpoints on a strictly stricter tier** — login,
>    token, password reset, OTP verify, signup. Starting point: **≤5 requests per
>    minute per (IP, account)** with escalating lockout.
> 4. **Failed-authentication responses counted on their own budget**, separate
>    from the request-rate budget.
> 5. **Concurrency limited as a distinct dimension** from rate.
> 6. **Every limit documented**, with its unit, window, and scope.

**Confidence: high on the shape; low-moderate on the specific numbers.** The
shape is supported by OWASP API4/API2 plus convergent practice across eight
vendors. The numbers are `[POLICY]` — `[FACT]` OWASP explicitly declines to give
any, and vendor numbers vary across four orders of magnitude, so no number here
can claim evidentiary force.

### Threat-model triggers that change the posture

| Trigger | Change | Evidence |
| --- | --- | --- |
| Per-request cost varies by more than ~10× (search, bulk, expansions, inference) | Switch from request counting to **cost/point/token accounting** | GitHub 900 points/min, Shopify points, OpenAI/Anthropic TPM `[COMPARATIVE]` |
| API calls trigger **metered third-party spend** (SMS, email, inference, biometrics) | Add **spend caps**, not just rate caps | OWASP API4:2023 *"Configure spending limits for all service providers/API integrations"* `[FACT]` |
| Consumer accounts / credential-stuffing exposure | Auth-endpoint tier + account lockout + failed-attempt budget | OWASP API2:2023 "stricter than the regular rate limiting" `[FACT]`; Auth0, Okta, Cloudflare numbers `[COMPARATIVE]` |
| Multi-tenant with shared capacity | Add **per-tenant fair-share** above per-key limits | Shopify per-app-per-store `[COMPARATIVE]` |
| Free tier or trial abuse | Tie quota to **spend tier**; consider tenure gating | all three AI platforms `[COMPARATIVE]`; Gemini adds a tenure requirement `[FACT]` |
| Long-running or resource-intensive requests | Add a **concurrency** limit distinct from rate | Stripe `[FACT]` |
| Endpoint returns identical responses for valid and invalid input (enumeration surface) | Rate-limit on the **response class**, not just the request | Cloudflare's 401/403-only counting rule `[FACT]` |

---

## Axis 4 — Replay-window length

Already ratified: webhooks use a 5-minute timestamp tolerance convention. This
axis generalises that to signed requests.

### What the standards say

`[FACT]` **RFC 9449 §11.1 — the only `MUST` on window length in any of these
documents, and it is deliberately unquantified:**

> "To limit this, servers MUST only accept DPoP proofs for a limited time after
> their creation (preferably only for a relatively brief period on the order of
> seconds or minutes)."

`[FACT]` **The same section makes the window insufficient on its own:**

> "In the context of the target URI, servers can store the jti value of each DPoP
> proof for the time window in which the respective DPoP proof JWT would be
> accepted to prevent multiple uses of the same DPoP proof… When strictly
> enforced, such a single-use check provides a very strong protection against
> DPoP proof replay, but it may not always be feasible in practice, e.g., when
> multiple servers behind a single endpoint have no shared state."

`[FACT]` **On clock skew — and the escape from it:**

> "Note: To accommodate for clock offsets, the server MAY accept DPoP proofs that
> carry an iat time in the reasonably near future (on the order of seconds or
> minutes). Because clock skews between servers and clients may be large, servers
> MAY limit DPoP proof lifetimes by using server-provided nonce values containing
> the time at the server rather than comparing the client-supplied iat time to the
> time at the server. Nonces created in this way yield the same result even in the
> face of arbitrarily large clock skews."

`[FACT]` **FAPI 2.0 §5.3.2.1 gives the only hard numbers in any standard here:**

> "to accommodate clock offsets, shall accept JWTs with an iat or nbf timestamp
> between 0 and 10 seconds in the future but shall reject JWTs with an iat or nbf
> timestamp greater than 60 seconds in the future."

`[FACT]` **FAPI 2.0 Note 3 gives the rationale, and it is worth quoting in full
because it is the only published reasoning about the size of a skew allowance:**

> "Clock skew is a cause of many interoperability issues. Even a few hundred
> milliseconds of clock skew can cause JWTs to be rejected for being 'issued in
> the future'. The DPoP specification [RFC9449] suggests that JWTs are accepted
> in the reasonably near future (on the order of seconds or minutes). This
> document goes further by requiring authorization servers to accept JWTs that
> have timestamps up to 10 seconds in the future. 10 seconds was chosen as a
> value that does not affect security while greatly increasing interoperability.
> Implementers are free to accept JWTs with a timestamp of up to 60 seconds in
> the future. Some ecosystems have found that the value of 30 seconds is needed
> to fully eliminate clock skew issues. To prevent implementations switching off
> iat and nbf checks completely this document imposes a maximum timestamp in the
> future of 60 seconds."

`[FACT]` **FAPI 2.0 §6.2 names window-shortening as the primary replay
mitigation, and states its limit:** *"Resource servers use short-lived DPoP
nonces to reduce the time window where a request can be replayed"*; *"Resource
servers implement replay prevention using the jti header"*; and — important for a
standard that has already adopted RFC 9421 — *"This may also allow reuse of the
DPoP proof with an altered request, as DPoP does not sign the body of HTTP
requests nor most headers. For example, for a payment request the attacker might
be able to specify a different amount or destination account."*

`[FACT]` **RFC 9421 §7.2.2** treats replay as a three-part problem: sufficient
component coverage, a `nonce` parameter, and `created`/`expires` — *"the signer
can provide a timestamp for when the signature was created and a time at which
the signer considers the signature to be expired, limiting the utility of a
captured signature value."*

### Field practice — the windows actually shipped

| Scheme | Window | Symmetry | Source |
| --- | --- | --- | --- |
| **AWS S3 REST auth** | **15 minutes** | ± (skew in either direction) | AWS docs source |
| **Standard Webhooks** reference libraries | **±300 s** | **symmetric** — rejects "too old" *and* "too new" | JS library source |
| Standard Webhooks **spec text** | *"within some allowable tolerance"* — **no number** | — | spec |
| **Stripe** | 300 s default tolerance | — | prior repo research |
| **AdCP** | `expires ≤ created + 300` | one-sided | prior repo research |
| **UCP** | no timestamp in its own example; defers replay to idempotency keys | — | prior repo research |
| **FAPI 2.0** | future skew +10 s normal, +60 s hard ceiling | **asymmetric** | spec |
| **Azure Communication Services** repeatability | 5 minutes, `412` outside | — | prior repo research |

`[FACT]` **AWS S3, verbatim** (from AWS's own documentation source repository —
the live page is JavaScript-rendered and could not be fetched):

> "A valid time stamp (using either the HTTP `Date` header or an `x-amz-date`
> alternative) is mandatory for authenticated requests. Furthermore, the client
> timestamp included with an authenticated request must be within 15 minutes of
> the Amazon S3 system time when the request is received. If not, the request
> will fail with the `RequestTimeTooSkewed` error code. The intention of these
> restrictions is to limit the possibility that intercepted requests could be
> replayed by an adversary."

`[FACT]` **The Standard Webhooks 5-minute figure is a library default, not spec
text.** The specification says only *"Make sure to verify the `webhook-timestamp`
header has a timestamp that is within some allowable tolerance of the current
timestamp to prevent replay attacks."* The number lives in the reference
implementation, verified in source:
`const WEBHOOK_TOLERANCE_IN_SECONDS = 5 * 60; // 5 minutes`, applied in both
directions. Cite it as a library convention; never as a specification
requirement.

`[FACT]` **The Standard Webhooks spec pairs the window with dedup**, and even
suggests the same duration for the dedup cache: *"Use the `webhook-id` header as
an idempotency key to prevent accidentally processing the same webhook more than
once (e.g. save the IDs in redis for 5 minutes)."*

### Recommended default

> **Default: a 300-second (5-minute) past tolerance and a 60-second future
> tolerance on any signed timestamp, paired with a mandatory replay cache keyed
> on the message/request identifier (`webhook-id`, `jti`, or `nonce`) retained for
> at least the full past window. Servers MUST synchronise clocks (NTP); the
> future tolerance exists to absorb skew, not to permit pre-dating.**
>
> **The window is never sufficient alone.** A deployment that cannot operate a
> shared replay cache must say so explicitly and accept that it has a
> 300-second replay exposure.

`[Note]` **This does not reopen the ratified webhook decision.** The 5-minute
webhook tolerance already ratified here stands unchanged; this axis generalises
it to signed requests and adds the future-side bound and the dedup requirement
that the webhook convention leaves to implementations.

**Confidence: high.** 300 s is the field's convergent value across Stripe,
Standard Webhooks libraries, AdCP, and Azure, and it keeps this axis consistent
with the webhook decision already ratified here. The **asymmetry** is the
non-obvious part and it is well grounded: FAPI 2.0 requires accepting +10 s and
rejecting beyond +60 s, with published reasoning. A symmetric ±300 s (the
Standard Webhooks library shape) accepts requests dated five minutes in the
future, which buys nothing and widens the pre-generation window RFC 9449 §11.2
warns about.

### Threat-model triggers

| Trigger | Change | Evidence |
| --- | --- | --- |
| **Interactive, synchronous** signed requests with no store-and-forward stage (DPoP proofs, API request signing) | Tighten to **30–60 s** | RFC 9449 §11.1 "seconds or minutes" `[FACT]` |
| **High-value financial operations** | Tighten, and add **server-provided nonces** | FAPI 2.0 §6.2 "short-lived DPoP nonces to reduce the time window" `[FACT]` |
| You can issue **server-provided nonces** | Window becomes server-clock-driven; **client clock skew stops mattering entirely** | RFC 9449 §11.1 "yield the same result even in the face of arbitrarily large clock skews" `[FACT]` |
| Clients on **unmanaged or unsynchronised clocks** (IoT, embedded, consumer devices) | Loosen the past window toward **15 minutes** — but only with dedup | AWS S3 15-min precedent, explicitly justified by skew `[FACT]` |
| **Store-and-forward** delivery: queues, batched retries, long-haul relays | Loosen the past window; keep the future window tight | `[INFERENCE]` — the retry timestamp is refreshed per attempt in Standard Webhooks, so the window applies to the *attempt*, not the event |
| No shared state across the server fleet (dedup cache infeasible) | **Tighten the window**, since it is now the only control | RFC 9449 §11.1 names exactly this case `[FACT]` |
| Signature does **not** cover the body (RFC 9421 without RFC 9530, DPoP proofs) | The window does not stop **altered** replays — add body binding | FAPI 2.0 §6.2 payment-amount example `[FACT]`; consistent with this repo's ratified RFC 9421 + RFC 9530 pairing |

---

## Axis 5 — Centralized object-level authorization enforcement

### What OWASP actually says — the load-bearing text

`[FACT]` **API1:2023 "How To Prevent", verbatim and complete:**

> - "Implement a proper authorization mechanism that relies on the user policies
>   and hierarchy."
> - "Use the authorization mechanism to check if the logged-in user has access to
>   perform the requested action on the record in every function that uses an
>   input from the client to access a record in the database."
> - "Prefer the use of random and unpredictable values as GUIDs for records' IDs."
> - "Write tests to evaluate the vulnerability of the authorization mechanism. Do
>   not deploy changes that make the tests fail."

`[FACT]` **API1:2023 never uses the word "centralized", and never mentions a
gateway, policy engine, or external component.** Confirmed by two independent
full-page raw-text extractions. What it does require is a check *"in every
function that uses an input from the client to access a record in the database"*
— a **per-call-site** obligation, expressed against a **single shared**
mechanism ("*the* authorization mechanism").

`[FACT]` **API5:2023 is where externalisation appears — and it is about
*function*-level, not *object*-level, authorization:**

> "Your application should have a consistent and easy-to-analyze authorization
> module that is invoked from all your business functions. Frequently, such
> protection is provided by one or more components external to the application
> code."
> "The enforcement mechanism(s) should deny all access by default, requiring
> explicit grants to specific roles for access to every function."

`[COMPARATIVE]` **This asymmetry is real, not a paraphrase artifact** — verified
against full raw HTML of both pages by two independent extractions. OWASP
gestures at external components for the *coarse* risk (which endpoints you may
call) and stays silent on architecture for the *fine* risk (which records you may
touch). That is the correct instinct, because the fine decision needs per-record
data the external component does not have.

`[FACT]` **The OWASP Authorization Cheat Sheet** (a *published* cheat sheet, not
a draft) supplies the mechanism guidance API1 omits:

> "Permission should be validated correctly on every request… The technology used
> to perform such checks should allow for global, application-wide configuration
> rather than needing to be applied individually to every method or class.
> Remember an attacker only needs to find one way in. Even if just a single access
> control check is 'missed', the confidentiality and/or integrity of a resource
> can be jeopardized."

`[FACT]` **The technologies it then names are all in-process middleware, not
gateways or policy engines:** Jakarta EE Filters / Spring Security, Django
middleware, .NET Core filters, Laravel middleware.

`[FACT]` **And its counterweight, from the same document:**

> "Implement defense in depth. Do not depend on any single framework, library,
> technology, or control to be the sole thing enforcing proper access control."

### The engines

`[FACT]` **Zanzibar** (Pang et al., USENIX ATC 2019) — the problem statement and
the scale that justifies it:

> "Zanzibar provides a uniform data model and configuration language for
> expressing a wide range of access control policies from hundreds of client
> services at Google, including Calendar, Cloud, Drive, Maps, Photos, and
> YouTube."
> "Zanzibar scales to trillions of access control lists and millions of
> authorization requests per second to support services used by billions of
> people. It has maintained 95th-percentile latency of less than 10 milliseconds
> and availability of greater than 99.999% over 3 years of production use."

| Engine | Model | Governance / status | Maturity |
| --- | --- | --- | --- |
| **OpenFGA** | ReBAC (Zanzibar) | CNCF hosted | accepted 2022-09-14; **Incubating since 2025-10-28** |
| **SpiceDB** / AuthZed | ReBAC (Zanzibar) | open source, Apache-2.0, commercially backed | current; hosted "AuthZed Cloud" |
| **Ory Keto** | ReBAC (Zanzibar) | open source, Ory | actively maintained |
| **WorkOS FGA** (ex-Warrant) | ReBAC | commercial SaaS | current |
| **Permify** | ReBAC/RBAC/ABAC | **acquired by FusionAuth 2025-11-20**; docs moved to `fusionauth.io/permify-docs/`; integrated offering "early 2026" still pending | in transition |
| **OPA** | policy (Rego) | CNCF | accepted 2018-03-29, incubating 2019-04-02, **Graduated 2021-01-29** |
| **Cedar / Amazon Verified Permissions** | RBAC + ABAC, verification-oriented | AWS, Apache-2.0 language | AVP GA date **not confirmable** from any fetchable AWS page |
| **Oso** | policy (Polar) | commercial Oso Cloud | **embedded library deprecated but explicitly not EOL** |

`[FACT]` **Oso's own repository states the migration precisely:** *"We have
deprecated the legacy Oso open source library… we are not end-of-lifing (EOL) the
library and we'll continue to provide support and critical bug fixes."* The
company's current product, Oso Cloud, is *"a centralized authorization service."*
`[COMPARATIVE]` This is a live instance of the in-process-library →
centralized-service migration this axis is about.

`[FACT]` **AWS describes Verified Permissions as externalisation:** *"Verified
Permissions enables your developers to build secure applications faster by
externalizing authorization and centralizing policy management and
administration."* Note the mechanism it then describes is still a **call from
your code**: *"In your application's code, you preface requests made to your
operations with a call to the Cedar authorization engine, asking 'Is this request
authorized?'"* — centralised decision, in-path enforcement.

### The documented failure mode of centralization

This is the counter-evidence, and it comes from the engines' own documentation.

`[FACT]` **SpiceDB puts the data-synchronisation burden on the application,
explicitly:**

> "It is the application's responsibility to keep the relationships within SpiceDB
> up-to-date and reflecting the state of the application; how an application does
> so can vary based on the specifics of the application."

Its recommended mitigation is a distributed-transaction pattern: *"The most
common and straightforward way to store relationships in SpiceDB is to use a
2 phase commit-like approach, making use of a transaction from the relational
database along with a WriteRelationships call to SpiceDB."*

`[FACT]` **OPA documents the lag as an additive quantity:**

> "The total lag between the external data source being updated and OPA being
> updated is the sum of the lag for an update between the data source and the
> synchronizer plus the lag for an update between the synchronizer and OPA."

And for the always-fresh alternative: *"It is crucial in this approach for the
OPA-enabled service to handle the case when OPA returns no decision."*

`[INFERENCE]` These two quotes describe one structural trade-off from two
directions: a centralized engine that is fast holds a **replica** of object state
and can be stale; a centralized engine that is always fresh takes a **synchronous
dependency** and introduces a no-decision failure mode. An in-handler check
reading the primary database in the same transaction has neither problem — which
is precisely why object-level authorization resists being lifted out of the
handler, and why OWASP's API1 guidance stays in the function.

### The three AI platforms

| Platform | Granularity | Key scoping |
| --- | --- | --- |
| **OpenAI** | Organizations → Projects; per-project isolation and per-project rate/spend limits | project-level in every page checked |
| **Anthropic** | Organizations → Workspaces (max 100/org). *"API keys are scoped to a single workspace and can only access resources within that workspace."* Workspace-scoped resources: Files API files, Message Batches, Skills | **workspace-level**; no documented per-object key scoping |
| **Google Cloud (Vertex AI / Gemini)** | *"You can manage access at the project level or resource level… For most Agent Platform resources, access can only be controlled by the project, folder, and organization. Access to individual resources can be granted only for specific resource types"* | project-level by default; **true object-level IAM for a named, limited subset only** |

`[COMPARATIVE]` Google is the only one of the three documenting object-level IAM
at all, and only for an enumerated subset of resource types. OpenAI and Anthropic
stop at project/workspace granularity. `[INFERENCE]` Even the largest new API
platforms are shipping tenant-scoped credentials rather than per-object
authorization surfaces — the object-level decision stays inside their services.

### Recommended default

> **Default: centralize the authorization *decision*; keep the *enforcement call
> site* inside every handler that resolves a client-supplied identifier to an
> object. One shared authorization component, invoked from every such call site,
> deny-by-default. Do NOT place object-level authorization in the API gateway.**
>
> Corollaries this standard should state:
> - Every endpoint that accepts an object identifier MUST perform an
>   object-level check; endpoint-level authorization is not a substitute.
> - Identifier unguessability MUST NOT be treated as an access control (already
>   ratified here as `OP-006`; OWASP's GUID advice is hardening, not a control).
> - Automated tests MUST cover the authorization matrix, and a failing
>   authorization test MUST block deployment — this is the one OWASP API1
>   prevention bullet that is routinely dropped.

**Confidence: high.** All three OWASP sources point the same way: API1 requires
the check *"in every function"*; API5 asks for *"a consistent and easy-to-analyze
authorization module"*; the Authorization Cheat Sheet asks for *"global,
application-wide configuration"* and then names **in-process middleware**. The
gateway is excluded on evidence, not taste: SpiceDB's and OPA's own docs show
that a component outside the application either replicates object state or takes
a synchronous data dependency.

### Threat-model triggers

| Trigger | Change | Evidence |
| --- | --- | --- |
| Permissions are **relationship-derived** — sharing, folder/group inheritance, groups-of-groups | Adopt a **Zanzibar-style ReBAC service** (OpenFGA, SpiceDB, Ory Keto) | Zanzibar paper problem statement `[FACT]` |
| **"List everything I can see"** at scale | Adopt ReBAC — a per-row in-handler check cannot answer reverse queries efficiently | `[INFERENCE]` from Zanzibar's design goals |
| **Multi-tenant with cross-tenant sharing** | Externalize the decision; keep enforcement in-path | `[INFERENCE]` |
| **Regulated domain** needing policy auditable and reviewable **separately from code** | Externalize to a policy language (Cedar, Rego) | AWS AVP *"externalizing authorization and centralizing policy management"* `[FACT]` |
| **Many services in many languages** needing one policy semantic | Externalize | AVP / OPA positioning `[FACT]` |
| Authorization needs **formal analysis / provable properties** | **Cedar** — *"designed for analysis using Automated Reasoning… proving that your security model is what you believe it is"* | Cedar repo `[FACT]` |
| Single service, ownership is a column on the row, no sharing graph | **Stay embedded.** Externalizing buys a replication problem and a network hop for no gain | SpiceDB + OPA data-staleness quotes `[FACT]` |
| Any externalized engine adopted | Keep the **in-path call site** and deny-by-default; do not treat the engine as the sole control | OWASP Cheat Sheet *"Do not depend on any single framework, library, technology, or control"* `[FACT]` |

---

## Deployment profile skeleton

Drop-in table for the decision record. "Default" is the recommended setting for a
typical public SaaS API; "flip triggers" are the named threat-model conditions
that change it.

| # | Axis | Recommended default | Flip triggers |
| --- | --- | --- | --- |
| 1 | **Sender-constrained tokens** | **Bearer over TLS** + short TTL + audience restriction + refresh-token rotation for public clients. Token validation `SHOULD NOT` hard-code the `Bearer` scheme. *(BCP 240 §2.2.1 is `SHOULD`; the only `MUST`, §2.2.2, is satisfiable by rotation.)* | → **DPoP or mTLS required**: FAPI 2.0 / open-banking / payment initiation · tokens visible to logging intermediaries · public clients in hostile environments (DPoP) · existing PKI, server-to-server (mTLS) · one token across many resource servers (mTLS)  → **RFC 9470 step-up instead of a global flip** when only some operations are high-value  → **Not the right control** when the threat is XSS in a browser client |
| 2 | **Opaque vs JWT access tokens** | **Opaque on the public wire**; phantom-token (introspect at the edge, forward `at+jwt` internally) where a gateway exists — note this is integration work, only Kong documents it natively. If JWT is issued to clients: RFC 9068 conformant (**opt-in at Keycloak and Auth0, not a default**) + explicit revocation-propagation mechanism. | → **JWT**: measured introspection bottleneck across administrative domains · validation must survive AS outage or run at the edge · third-party resource servers you cannot credential  → **Stay opaque**: instant-revocation SLA · token would carry tenant IDs, roles, entitlements or PII (RFC 9068 §6)  → If JWT chosen, **budget for revocation lag** — Entra CAE documents up to 15 minutes |
| 3 | **Rate-limit aggressiveness** | **Multi-dimensional, tiered, published.** Per-principal sustained + token-bucket burst (start ≈100 req/s account, 25 req/s endpoint) · unauthenticated an order of magnitude lower, per-IP · **auth endpoints strictly stricter** (start ≤5/min per IP+account) · failed-auth on its own budget · concurrency as a separate dimension. Numbers are `[POLICY]`; OWASP publishes none. | → **Cost/point/token accounting** when per-request cost varies >10× · **spend caps** when calls trigger metered third-party spend (OWASP API4) · **auth tier + lockout** on credential-stuffing exposure · **per-tenant fair-share** for noisy neighbours · **spend/tenure-gated tiers** for free-tier abuse · **response-class limiting** on enumeration surfaces |
| 4 | **Replay window** | **300 s past / 60 s future**, asymmetric, **plus** a mandatory replay cache keyed on message ID / `jti` / `nonce` held ≥ the past window. NTP required. Window alone is never sufficient. | → **Tighten to 30–60 s**: interactive synchronous signing · high-value financial ops · no shared dedup state  → **Server-provided nonces** remove clock skew from the problem entirely (RFC 9449 §11.1)  → **Loosen toward 15 min**: unmanaged client clocks (AWS S3 precedent) · store-and-forward delivery — never without dedup  → **Window does not stop altered replays** when the signature omits the body — add body binding (RFC 9530) |
| 5 | **Object-level authorization** | **Centralize the decision, keep enforcement in the handler.** One shared authorization component invoked from every call site that resolves a client-supplied ID to an object; deny-by-default; **not in the gateway**. Authorization tests block deployment. | → **Zanzibar-style ReBAC** (OpenFGA/SpiceDB/Keto): relationship-derived permissions · "list what I can see" at scale · multi-tenant cross-tenant sharing  → **Policy language** (Cedar/Rego): regulated domain needing code-independent audit · many services, many languages · formal analysis required (Cedar)  → **Stay embedded**: single service, ownership is a column, no sharing graph  → **Always** keep the in-path call site — no engine is the sole control |

---

## Gaps, absences, and re-check triggers

**Reported absences** (per project rule — these are findings, not omissions):

- `[FACT]` **No OWASP API Security Top 10 edition newer than 2023** exists as of
  2026-08-09 (two surfaces checked).
- `[FACT]` **OWASP publishes no numeric rate-limit value anywhere** in API4:2023
  — it explicitly defers to business needs.
- `[FACT]` **OWASP API1:2023 says nothing about centralization.** The
  centralization language lives in API5:2023 and the Authorization Cheat Sheet.
- `[FACT]` **RFC 9068 defines no revocation mechanism.**
- `[FACT]` **The Standard Webhooks specification states no numeric tolerance** —
  300 s is a reference-library constant.
- `[FACT]` **Google Gemini publishes no per-model rate-limit numbers** in public
  docs (behind the AI Studio console), and emits no rate-limit headers.
- `[FACT]` **Microsoft Entra ID supports neither RFC 9449 nor RFC 8705.**
- `[FACT]` **No sender-constraining in any of OpenAI, Anthropic, or Gemini**, and
  the MCP authorization spec mandates plain `Bearer`.

**Retrieval limitations to disclose if any of these become load-bearing:**

- `WebSearch` quota exhausted at 200/200 for this session; all later retrieval was
  direct-URL fetching. DPoP support for **Okta, PingFederate, ForgeRock/PingAM,
  Curity (general OAuth), and Cloudflare** could not be confirmed either way —
  these are *gaps*, not negative findings.
- **AWS live docs are JS-rendered**; the S3 15-minute quotation comes from AWS's
  archived documentation source repository. Re-verify against the live page
  before publishing it as normative.
- **Amazon Verified Permissions GA date** not confirmable from any fetchable AWS
  page. Whether **Cedar underpins core AWS IAM** is unconfirmed — do not assert
  it.
- **Ping Identity** could not be characterised on either axis 1 or axis 2 — six
  candidate documentation URLs 404'd or redirect-looped. Absent, not negative.
- **Microsoft's own two pages disagree** on the default Entra access-token
  lifetime: the access-tokens page says *"a random value ranging between 60-90
  minutes (75 minutes on average)"*, the CAE page says *"By default, access
  tokens are valid for one hour."* Cite whichever page you rely on and note the
  discrepancy rather than averaging it.
- **Apigee's third-party-token validation mechanism** and **Cloudflare/Gloo
  gateway introspection support** are unreadable or undiscoverable — gaps, not
  absences.
- **Keycloak's numeric default access-token lifespan** could not be extracted.
- **Okta's 4/second per-username limit** on `/api/v1/authn` and `/oauth2/v1/token`
  rests on a **single fetch, not cross-verified** against a second Okta page.
  Widen before ratifying it as evidence for the auth-endpoint tier. The
  Auth0 and Cloudflare numbers in that table are multi-page and stronger.
- **Verified this pass** (raw-text re-fetch, 2026-08-09): the Curity
  phantom-token definition, the AWS Cognito revocation sentence, and both
  Microsoft Entra sender-constraining sentences. The Curity re-fetch **changed
  the quoted text** relative to one delegated retrieval — an argument for keeping
  the re-verification rule.

**Additional re-check trigger:**

- **Microsoft Entra mTLS** — the proof-of-possession page states *"Future APIs
  will rely on PoP via mTLS."* If Entra ships RFC 8705 support, the central
  adoption argument behind axis 1's bearer default weakens materially. This is
  the single highest-value thing to watch on this axis.

**Dated re-check triggers to add to the register:**

- **OpenFGA CNCF maturity** — incubating since 2025-10-28; re-check for
  graduation, which would strengthen the axis-5 externalization trigger.
- **Permify / FusionAuth integration** — "early 2026" announcement still pending
  as of 2026-08-09; the project's independent status is in flux.
- **MCP authorization spec** — currently plain `Bearer` with no PoP extension. If
  a DPoP or mTLS extension lands, axis 1's "the agent ecosystem is unconstrained
  bearer" claim expires.
- **FAPI 2.0 adoption outside finance** — the cleanest trigger anchor for axis 1;
  watch whether non-financial ecosystems adopt it.
