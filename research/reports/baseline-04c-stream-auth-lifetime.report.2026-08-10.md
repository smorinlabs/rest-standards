# baseline-04c — Authorization over a long-lived stream's lifetime (report, 2026-08-10)

Research leaf under `baseline-04` / §13.4 item 2 ("Authorization over a
stream's lifetime"), opened by `PLAN.md` Phase 8. Series `baseline` =
prescriptive: **this report proposes; only a ratified record in
`research/decisions/` binds.** Nothing here amends `rest-api-standard.md`.

**Scope ruling inherited, not reopened.** `baseline-04-streaming.decision.md`
ratified WebSockets as an explicit non-goal. WebSocket material appears below
**only as contrast**, because it is the one surface where published
authorization-expiry guidance exists, and the absence of the equivalent for
HTTP response streams is itself the finding.

**Label key:** `[FACT]` primary-sourced · `[COMPARATIVE]` surveyed practice ·
`[INFERENCE]` reasoning from the above · `[POLICY]` a judgment this project
must make.

**Access date for every row below is 2026-08-10** unless a different date is
stated in the row.

---

## 1. TL;DR and recommendation

**The gap is real, the field has no consensus mechanism, and no published
standard addresses it.** Three independent verified negatives establish that
last claim: RFC 9700 (BCP 240) contains no occurrence of "streaming" or
"long-lived"; RFC 9068 contains no occurrence of "streaming", "long-lived", or
"connection"; RFC 7009 addresses revocation propagation between servers and is
silent on requests already in progress. `[FACT]`

**What the field actually does divides into three postures, not two:**

| Posture | Who ships it | Mechanism |
| --- | --- | --- |
| **Bounded stream lifetime, re-authorize on reconnect** | Kubernetes watch; AWS Kinesis `SubscribeToShard`; Microsoft Entra coauthoring sessions | The server closes the stream on a published maximum duration. Authorization is re-run when the client reconnects. |
| **Credential-bound lifetime (de facto)** | Google Vertex AI over gRPC; any gRPC server-streaming RPC | Credentials ride the request headers and cannot be refreshed mid-RPC; the stream fails `UNAUTHENTICATED` when the token expires. |
| **Unbounded and undocumented** | OpenAI, Anthropic, Google Gemini REST | No published maximum stream duration and no published statement of what happens to an in-flight stream when a key is revoked or expires. |

**Recommendation — a three-clause `MUST`, not disclosure-only.** Propose one
new rule (provisional `ST-021`, drafting as `R13.12`) with: a published
maximum stream duration; a prohibition on a stream outliving the validity of
the credential that authorized it; and a documented revocation posture stated
as a bound, not as a promise of immediacy. Full text and per-clause strength
in §5.

**Two sentences on why.** Disclosure-only is the right instrument when no
authority exists to make normative — that is exactly why `R13.5`'s keep-alive
clause is a disclosure duty — but here the authority does exist: `R8.6`
requires every request to be authorized deny-by-default, and RFC 9068 §4
requires the current time to be before `exp` at validation, so a stream that
keeps delivering data past `exp` is not an undocumented choice but an
unsatisfied obligation. The project has already declined the weaker "merely
document it" form once, at `P6-D1`, for a hazard with a smaller blast radius
than unauthorized data delivery.

**Confidence: moderate-high on clauses 1 and 3, moderate on clause 2.** The
mechanism in clause 2 is cheap (one comparison the server already has the
inputs for), but it departs from what the three mandatory deep-dive providers
publish, which is nothing — and a departure from unanimous silence is weaker
evidence than a departure from unanimous practice.

---

## 2. Standards-and-currency matrix

Authority classes follow `research/CONSTRAINTS.md`: IETF and IANA outrank
vendor documentation, which is comparative evidence only and never protocol
authority.

| Source | URL | Authority class | Publication / status | Accessed |
| --- | --- | --- | --- | --- |
| RFC 9700 — Best Current Practice for OAuth 2.0 Security (BCP 240) | `https://www.rfc-editor.org/rfc/rfc9700.txt` | **IETF Best Current Practice** | January 2025; updates RFC 6749, 6750, 6819 | 2026-08-10 |
| RFC 9068 — JWT Profile for OAuth 2.0 Access Tokens | `https://www.rfc-editor.org/rfc/rfc9068.txt` | **IETF Proposed Standard** | October 2021 | 2026-08-10 |
| RFC 7009 — OAuth 2.0 Token Revocation | `https://www.rfc-editor.org/rfc/rfc7009.html` | **IETF Proposed Standard** | August 2013 | 2026-08-10 |
| OpenID Continuous Access Evaluation Profile (CAEP) 1.0 | `https://openid.net/specs/openid-caep-1_0-final.html` | **OpenID Foundation, Standards Track, Final** | Published 29 August 2025 | 2026-08-10 |
| OpenID Shared Signals Framework 1.0 | `https://openid.net/specs/openid-sharedsignals-framework-1_0-final.html` | **OpenID Foundation, Standards Track, Final** | Final (date not re-verified here) | 2026-08-10 |
| Microsoft Entra — Continuous access evaluation | `https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation` | **Vendor documentation** — comparative only | `ms.date` 2026-04-08; `updated_at` 2026-04-24 | 2026-08-10 |
| Kubernetes — kube-apiserver flag reference (via Debian manpage mirror, see §7) | `https://manpages.debian.org/testing/kubernetes-master/kube-apiserver.1` | **Third-party mirror of vendor reference** | Debian testing | 2026-08-10 |
| Kubernetes — Managing Service Accounts | `https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/` | **Vendor documentation** | living | 2026-08-10 |
| Kubernetes — Controlling Access to the Kubernetes API | `https://kubernetes.io/docs/concepts/security/controlling-access/` | **Vendor documentation** | living | 2026-08-10 |
| KEP-1205 — Bound Service Account Tokens | `https://github.com/kubernetes/enhancements/blob/master/keps/sig-auth/1205-bound-service-account-tokens/README.md` | **Vendor design document** | implemented | 2026-08-10 |
| kubernetes/kubernetes issue 120621 | `https://github.com/kubernetes/kubernetes/issues/120621` | **Vendor issue tracker** — weakest class | Closed as not planned | 2026-08-10 |
| grpc/grpc-go issue 4868 | `https://github.com/grpc/grpc-go/issues/4868` | **Vendor issue tracker** | opened 2021-10-13 | 2026-08-10 |
| googleapis/google-cloud-go issue 13533 | `https://github.com/googleapis/google-cloud-go/issues/13533` | **Vendor issue tracker** | opened 2026-01-04, feature request | 2026-08-10 |
| OpenAI — API reference overview (authentication) | `https://developers.openai.com/api/reference/overview` | **Vendor documentation** | living | 2026-08-10 |
| OpenAI — Background mode guide | `https://developers.openai.com/api/docs/guides/background` | **Vendor documentation** | living | 2026-08-10 |
| Anthropic — Errors (long requests, 401) | `https://platform.claude.com/docs/en/api/errors` | **Vendor documentation** | living | 2026-08-10 |
| Anthropic — Authentication (key expiration) | `https://platform.claude.com/docs/en/manage-claude/authentication` | **Vendor documentation** | living | 2026-08-10 |
| Anthropic — Streaming messages | `https://platform.claude.com/docs/en/api/streaming` | **Vendor documentation** | living | 2026-08-10 |
| Google Gemini API — troubleshooting | `https://ai.google.dev/gemini-api/docs/troubleshooting` | **Vendor documentation** | living | 2026-08-10 |
| AWS — Kinesis `SubscribeToShard` | `https://docs.aws.amazon.com/kinesis/latest/APIReference/API_SubscribeToShard.html` | **Vendor documentation** | living | 2026-08-10 |
| AWS — Setting up a streaming transcription | `https://docs.aws.amazon.com/transcribe/latest/dg/streaming-setting-up.html` | **Vendor documentation** | living | 2026-08-10 |
| OWASP — WebSocket Security Cheat Sheet | `https://cheatsheetseries.owasp.org/cheatsheets/WebSocket_Security_Cheat_Sheet.html` | **Community security guidance** — not a standard | living; see §7 for the verification caveat | 2026-08-10 |

**Currency note.** Nothing in this matrix supersedes anything in
`baseline-04-streaming.report.2026-08-10.md`'s matrix; the two are disjoint
except for the vendor streaming pages, whose streaming-mechanics findings are
reused rather than re-derived.

---

## 3. Field evidence

### 3.1 The mandatory deep-dive: OpenAI, Anthropic, Google Gemini

The question asked of each provider was four-part: is streaming request
duration bounded; what is published about an in-flight stream when a key is
revoked or a token expires; is a maximum duration published; is any mid-stream
authorization behavior documented.

**OpenAI.**

`[FACT]` Revocation latency is published, and it is the only such number among
the three: *"Revocations of an API key take effect within a few seconds."* and
*"Most updates that affect authentication results of an API key propagate
within 15 minutes, but can potentially take longer."*
(`https://developers.openai.com/api/reference/overview`.) `[FACT]` The same
page says nothing about requests already in progress — a verified negative
established by asking the question directly of the page text.

`[FACT]` Background mode publishes a retention window, not a duration bound:
*"Response data is temporarily stored to disk for roughly 10 minutes to enable
asynchronous execution and polling."*
(`https://developers.openai.com/api/docs/guides/background`.) Resumption is by
`starting_after` carrying the `sequence_number` of the last received event —
the mechanism `ST-010` / `R13.10` was built from. `[FACT]` The page contains no
statement about authorization or authentication when resuming.

`[INFERENCE]` Resumption is where the gap bites hardest and is also where it is
cheapest to close: a resumption request is a **new** HTTP request, so `R8.6`
already authorizes it. The unresolved case is the original stream, which is one
request.

**Anthropic.**

`[FACT]` A duration threshold is published, but as advice about the *client's*
network, not as a server-enforced maximum: *"Consider using the streaming
Messages API or Message Batches API for long-running requests, especially those
over 10 minutes."* and *"The SDKs validate that your non-streaming Messages API
requests are not expected to exceed a 10-minute timeout."*
(`https://platform.claude.com/docs/en/api/errors`.) The stated reason is
network behavior — *"Some networks may drop idle connections after a variable
period of time"* — not credential lifetime. **No maximum streaming duration is
published.** `[FACT]`

`[FACT]` Credential lifetime *is* published and is short by default at the
shortest preset: API keys are created with an expiration chosen from *"a preset
(3 hours, 1 day, 7 days, or 30 days), a custom duration, or **Never**"*, an
organization may impose a maximum-expiration policy that removes **Never**, and
*"After a key expires, requests made with it return a `401
authentication_error`."*
(`https://platform.claude.com/docs/en/manage-claude/authentication`.) Two other
credential types are explicitly short-lived: Workload Identity Federation
exchanges an IdP JWT for *"a short-lived Claude API access token, and the SDK
refreshes that token automatically before it expires"*, and App Attest tokens
*"expire after one hour."*

`[INFERENCE]` This is the sharpest instance of the gap in the whole survey.
Anthropic ships one-hour App Attest tokens, an SDK that refreshes federation
tokens before expiry, and a streaming API with no published maximum duration —
so the documented artifacts do not, together, say what happens to a stream that
outlives its token. The SDK's automatic refresh cannot help an open response:
refreshing a credential does not re-present it on a request already sent.

`[FACT]` Mid-stream errors are documented, and their shape is the one `R13.7`
codifies: *"When receiving a streaming response over server-sent events (SSE),
an error can occur after the API returns a 200 response. In that case, error
handling doesn't follow these standard mechanisms."* The documented example is
`overloaded_error`. **No authentication or authorization error is documented as
a mid-stream error type.** `[FACT]`

**Google Gemini.**

`[FACT]` The Gemini API troubleshooting page documents retry strategy for 429,
408, and 5xx and says nothing about deadline-exceeded, request-duration
maxima, streaming as a remedy for long requests, or key revocation or expiry
during a request (`https://ai.google.dev/gemini-api/docs/troubleshooting`).

`[COMPARATIVE]` Google Cloud documentation for the Vertex-hosted surface does
publish duration numbers, but they are per-product and none is a *Gemini REST
streaming* maximum: a 10-minute default request timeout on dedicated public
endpoints with a configurable maximum of 3600 seconds, and a 10-minute maximum
for bidirectional streaming on Agent Engine Runtime. These were surfaced by
search across `docs.cloud.google.com` and are **not** re-verified against the
individual pages here (see §7). A widely repeated community claim of a hard
five-minute server-side limit on `streamGenerateContent` **failed
verification** against any Google-published page and is recorded as unverified,
not as evidence.

`[FACT]` The strongest Google data point is not in the REST documentation at
all; it is in the gRPC transport and is covered in §3.3.

**Deep-dive comparison table.**

| Provider | Published max stream duration | Published revocation latency | Documented behavior of an in-flight stream on key revocation or expiry | Mid-stream auth error type documented |
| --- | --- | --- | --- | --- |
| OpenAI | **None.** 10-minute background retention is a storage window, not a duration bound | *"within a few seconds"*; other auth updates *"within 15 minutes"* | **None** — verified negative | **No** |
| Anthropic | **None.** 10 minutes is client-side SDK validation for non-streaming calls | **None** — expiry is documented, latency is not | **None** — verified negative | **No** (only `overloaded_error` is exemplified) |
| Google Gemini | **None on the REST surface.** Vertex-hosted numbers exist per product | **None** located | **None** — verified negative | **No** |

`[COMPARATIVE]` **Three for three, the answer is silence.** No deep-dive
provider publishes a maximum streaming duration, and none publishes what
happens to an open stream when the credential behind it stops being valid.
This is a stronger negative than the streaming survey's negatives, because
these are the three vendors whose entire product is long-lived streaming.

### 3.2 Kubernetes watch — the clearest comparable precedent

`[FACT]` **Authorization is per HTTP request, and a watch is one request.**
Kubernetes documents authentication as step 1 and authorization as step 2 of
handling *the request*: *"Once TLS is established, the HTTP request moves to
the Authentication step"* and *"After the request is authenticated as coming
from a specific user, the request must be authorized."*
(`https://kubernetes.io/docs/concepts/security/controlling-access/`.) Nothing
on that page describes a second evaluation during a request.

`[FACT]` **The lifetime bound is a server-side maximum with jitter, and it is
generous.** The `--min-request-timeout` flag is *"An optional field indicating
the minimum number of seconds a handler must keep a request open before timing
it out. Currently only honored by the watch request handler, which picks a
randomized value above this number as the connection timeout, to spread out
load."* Default **1800** seconds. The general `--request-timeout` default is
**1m0s** and explicitly does not govern watches: *"This is the default request
timeout for requests but may be overridden by flags such as
--min-request-timeout for specific types of requests."* Source: Debian's
manpage rendering of the kube-apiserver reference; see §7 for why this mirror
was used. `[INFERENCE]` A randomized value "above this number" with a
1800-second floor puts an ordinary watch's ceiling at roughly 30 to 60 minutes
— the same order as an hour-long OAuth access token, and materially longer than
Anthropic's shortest key preset.

`[FACT]` **The flag the task named is spelled differently in the product.** The
token-lifetime flag is `--service-account-max-token-expiration`, described as
the maximum validity duration of a token created by the service-account token
issuer, with an over-long `TokenRequest` being issued at the capped duration
instead. It is not `--service-account-max-token-expiry`. Recorded because the
research prompt used the other spelling and an unverified copy would have
propagated it.

`[FACT]` **Bound tokens have no revocation mechanism, by design.** Kubernetes
states that a short-lived token issued via `TokenRequest` cannot be revoked
directly and expires on its own timetable; the documented lever is deleting the
bound object (delete the Pod, or for the legacy Secret-bound form delete the
Secret). KEP-1205 frames validation as presentation-time: *"A recipient of a
token should verify that the token is valid at the time that the token is
presented, and should otherwise reject the token."* `[FACT]` KEP-1205 does
**not** address whether an established connection survives its token's expiry;
its lifecycle discussion is about the kubelet proactively rotating the token on
disk (*"if the token is older than 80 percent of its time to live or if the
token is older than 24 hours"*) and about clients needing to reload it —
*"if in-cluster clients do not periodically reload token from projected volume,
requests would be rejected once the initial token got expired"*, which is a
statement about the **next** request, not the current one.

`[FACT]` **A closely analogous case was raised and declined.** Issue 120621,
*"Kube API Server should close keepalive connections using unauthorized client
certificates"*, reports that *"They will continue reusing a connection
established with an old cert indefinitely unless the connection dies"* and asks
the API server to *"close the keepalive connection after detecting an invalid
cert to force clients to re-establish the connection."* It is **closed as not
planned**. `[INFERENCE]` This is about connection reuse across requests rather
than one long request, so it is not the same fact — but it is the nearest
published expression of the project's disposition, and the disposition is that
the server does not police an already-established connection.

`[INFERENCE]` **Kubernetes' composite answer, stated plainly:** authorize once
at establishment, bound the stream by a published server-side maximum unrelated
to credential lifetime, and let re-authorization happen when the client
reconnects. The security property this buys is a bounded exposure window, not
prompt revocation.

### 3.3 gRPC and Google Vertex AI — credential-bound lifetime in practice

`[FACT]` **gRPC cannot refresh call credentials during an RPC.** The grpc-go
issue *"PerRPCCredentials refresh token documentation is unclear"* (opened
2021-10-13) turns on the fact that credential tokens travel in headers and
therefore cannot be updated during the course of an RPC; the remedy is to
recreate the stream, or to carry the token in each application message instead
of relying on per-RPC credentials. **Caveat recorded:** the retrieved page
content contained the original report and not the maintainer replies, so this
is filed as the reporter's characterization corroborated by the second source
below, not as a maintainer statement. `[COMPARATIVE]`

`[FACT]` **A shipped Google product fails the stream on token expiry.**
`googleapis/google-cloud-go` issue 13533, *"Vertex AI Generative:
ACCESS_TOKEN_EXPIRED"* (opened 2026-01-04, assigned to a Google maintainer,
classified a feature request), reports that when an access token expires during
an active stream the call returns *"UNAUTHENTICATED: Request had invalid
authentication credentials... reason:ACCESS_TOKEN_EXPIRED"* and the stream
ends, with the requested behavior being *"Streaming should survive access token
refresh; the SDK should renew tokens and keep the active stream alive without
returning UNAUTHENTICATED."* The reporter's environment is Kubernetes pods with
service-account authentication where stream duration exceeds token lifetime —
the exact composition this leaf exists to rule on.

`[INFERENCE]` **Read this carefully in both directions.** It demonstrates that
credential-bound stream lifetime is achievable and shipping. It does **not**
prove a deliberate server-side policy: the report is filed against a client
SDK, and the retrieved text does not establish whether the `UNAUTHENTICATED`
arrives on the established stream or on an internal retry that opens a new RPC.
The honest reading is that on Google's gRPC surface a stream does not outlive
its credential, and that the ecosystem regards this as a defect to be fixed
rather than a security property to be preserved.

### 3.4 The eight standard references

Participation was established in `survey-08-streaming.report.2026-08-10.md`
(verification date 2026-08-10) and is reused rather than re-derived; the new
work here is the authorization-lifetime column.

| Reference | HTTP response-streaming surface | Authorization-lifetime guidance | Verified |
| --- | --- | --- | --- |
| **AWS** | Yes, several — Kinesis `SubscribeToShard` over HTTP/2, Transcribe streaming over HTTP/2, Bedrock response streams | **Yes, indirectly and strongly** — see below | 2026-08-10 |
| **Microsoft / Azure** | Azure REST API Guidelines carry no streaming rule at all (`survey-08`) | **Yes, at the identity layer, not the API-guideline layer** — Entra CAE, see §4.1 | 2026-08-10 |
| **Google / AIP** | AIP-151 forbids a streaming response for the long-running-operation pattern (`survey-08`) | **None** — no AIP addresses authorization over a stream's lifetime | 2026-08-10 |
| **Stripe** | **None.** Stripe's own OpenAPI specification greps to 0 occurrences of `event-stream` and 0 of `ndjson` (`survey-08`) | **Non-participant** | 2026-08-10 |
| **GitHub** | **None documented at the wire level** (`survey-08`) | **Non-participant** | 2026-08-10 |
| **Twilio** | **None** — Event Streams delivers to outbound sinks; Media Streams is WebSocket (`survey-08`) | **Non-participant** | 2026-08-10 |
| **Shopify** | **None** — bulk operations produce a JSONL file behind a URL (`survey-08`) | **Non-participant** | 2026-08-10 |
| **Zalando** | **None** — the guidelines grep to 0 occurrences of `event-stream`, `server-sent`, or `ndjson` (`survey-08`) | **Non-participant** | 2026-08-10 |

**Five of eight are non-participants.** `[COMPARATIVE]` That is not a
weakness of the evidence; it is the finding that the guideline layer of the
industry has not met this problem, which is consistent with the standards-layer
negative in §4.2.

**AWS, in detail — the two mechanisms are opposites and both are instructive.**

`[FACT]` **Kinesis bounds the stream hard and cheaply.** *"When the
`SubscribeToShard` call succeeds, your consumer starts receiving events of type
`SubscribeToShardEvent` over the HTTP/2 connection for up to 5 minutes, after
which time you need to call `SubscribeToShard` again to renew the subscription
if you want to continue to receive records."* The renewal is a fresh signed
request, so authorization is re-evaluated every five minutes with no
mid-stream machinery whatever. The API also documents `AccessDeniedException`
as a start-of-call error, not an in-stream event.

`[FACT]` **Transcribe re-presents credentials on every frame — in the request
direction.** The HTTP/2 setup requires *"a signature in the authorization
header that Amazon Transcribe uses as a seed signature to sign the data
frames"*, and each subsequent frame carries a `:chunk-signature` computed over,
among other inputs, `priorSignature` — *"The signature for the previous frame.
For the first data frame, use the signature of the header frame."* `[INFERENCE]`
This is a shipped existence proof that continuous per-frame credential
presentation over one long-lived HTTP/2 request is implementable. Two
qualifications keep it from being a template for this standard: it
authenticates the **client-to-server** direction of a bidirectional exchange,
which is not the response-streaming case §13 governs; and it uses a signature
chain rather than a re-fetched token, so it demonstrates continuous
*authentication* of frames, not continuous *authorization* against a policy
that may have changed.

`[FACT]` **AWS also documents the decoupling explicitly.** For the WebSocket
variant of the same service the presigned URL's *"maximum value for
`X-Amz-Expires` is 300 (5 minutes)"* while the session itself is long-lived —
the credential's validity window and the connection's lifetime are, by design,
different numbers.

---

## 4. Evidence for and against a normative rule

Both sides are stated at full strength. Where a source cuts both ways it
appears on both sides.

### 4.1 FOR requiring lifetime binding or mid-stream re-evaluation

**F1. `[FACT]` The largest published account of continuous evaluation exists,
and its own documentation describes the exact failure mode this leaf names.**
Microsoft Entra's coauthoring limitation is the long-lived-session analogue of
a long-lived stream: *"When multiple users are collaborating on a document at
the same time, CAE might not revoke their access to the document immediately
based on policy change events. In this case, the user loses access completely
after: Closing the document, Closing the Office app, After 1 hour when a
Conditional Access IP policy is set."* Microsoft's own remedy is a **bounded
session lifetime**, configurable down to 15 minutes:
*"the maximum lifetime of coauthoring sessions is reduced to 15 minutes, and
can be adjusted further using the SharePoint Online PowerShell command
`Set-SPOTenant –IPAddressWACTokenLifetime`."* `[INFERENCE]` A vendor with a
purpose-built revocation channel still falls back to bounding the session's
lifetime when the session is long-lived. That is precisely the clause this
report proposes.

**F2. `[FACT]` The threat is not hypothetical at the identity layer.** Entra's
enumerated critical events are *"User Account is deleted or disabled"*,
*"Password for a user is changed or reset"*, *"Multifactor authentication is
enabled for the user"*, *"Administrator explicitly revokes all refresh tokens
for a user"*, and *"High user risk detected by Microsoft Entra ID Protection"*.
Each is a case where an operator has decided a principal must stop receiving
data now. `[INFERENCE]` An open stream is a channel that continues delivering
data to exactly that principal.

**F3. `[FACT]` A standards-track vocabulary for the signal already exists.**
OpenID CAEP 1.0 (Standards Track, published 29 August 2025) defines
`session-revoked` — *"Session Revoked signals that the session identified by
the subject has been revoked"* — and `token-claims-change` — *"Token Claims
Change signals that a claim in a token, identified by the subject claim, has
changed."* `[INFERENCE]` A server that wants to terminate streams on revocation
has a published event vocabulary to consume; the mechanism is not
unprecedented. **It is not an HTTP standard and says nothing about streams**,
which is why it supports a `SHOULD` and not a `MUST`.

**F4. `[FACT]` The composition inside this standard is already broken without
a rule.** `R8.5` wants credentials scoped and expiring; `R8.10`'s token-format
axis defaults to opaque tokens precisely so that revocation is fast, and makes
a client-visible JWT conditional on *"a revocation-propagation plan"*. A stream
that ignores both is a hole in machinery this standard already ratified.
`[INFERENCE]` `R8.10`'s axis is not merely unhelpful here — it is actively
misleading, because it promises a revocation posture the streaming surface does
not honor.

**F5. `[FACT]` A major provider already binds stream lifetime to credential
validity in production.** Google Vertex AI over gRPC ends the stream with
`ACCESS_TOKEN_EXPIRED` (§3.3). `[COMPARATIVE]` Whatever its intent, it
demonstrates that the behavior is compatible with a shipping product at scale.

**F6. `[FACT]` Bounded stream lifetime is the field's dominant answer where
anyone has answered at all.** Kubernetes (default floor 1800 s, randomized
above), AWS Kinesis (5 minutes, hard), Microsoft (1 hour default, 15 minutes
configurable). `[INFERENCE]` Three independent, decade-old, load-bearing
systems converged on the same instrument. That is the strongest comparative
evidence in this report.

**F7. `[FACT]` The one body of published security guidance that addresses this
question at all comes down squarely on the side of acting — and it is guidance
for the adjacent surface, WebSockets, which this standard excludes.** OWASP's
WebSocket Security Cheat Sheet, verbatim: *"Token refresh: Rotate tokens in
long-lived connections to prevent hijacked sessions from persisting."*;
*"Close WebSocket connections when sessions expire. Re-validate user sessions
periodically (every 30 minutes is common) to ensure they remain valid."*;
*"When users log out, close all their WebSocket connections immediately.
Maintain a mapping of sessions to active connections so you can invalidate
WebSocket access the moment logout occurs."*; and *"Don't assume WebSocket
connection equals unlimited access. Check authorization for each action."*

`[INFERENCE]` **The contrast is the finding.** Every one of those four
sentences describes a hazard that an HTTP response stream has in identical
form, and none of them is expressible for HTTP response streams in any
published guidance located by this leaf. The reason is not that the hazard is
absent — it is that WebSockets acquired a security literature and long-lived
HTTP responses did not. `[COMPARATIVE]` The 30-minute re-validation interval is
community guidance, not a standard, and is **not** proposed as a number here;
what it supports is clause 3's direction, not a mandated period.

### 4.2 AGAINST requiring it

**A1. `[FACT]` No published standard requires it, and the negatives are
verified, not assumed.** RFC 9700 (BCP 240) contains **no occurrence** of
"streaming" or "long-lived"; its only lifetime guidance is incidental — §4.14
notes that a mechanism lets the authorization server *"issue access tokens with
a short lifetime and reduced scope"*, and §4.14.2 that *"Refresh tokens SHOULD
expire if the client has been inactive for some time."* (A second extraction
pass over the same RFC reported no access-token-lifetime sentence in §4.14; the
conflict is recorded in §7 item 6 and does not touch the negative.) RFC 9068 contains **no
occurrence** of "streaming", "long-lived", or "connection"; §4 requires only
that *"The current time MUST be before the time represented by the `exp`
claim"* at validation, with validation framed as something a resource server
does on receiving a token. RFC 7009 concerns revocation propagation between
servers — *"In practice, there could be a propagation delay, for example, in
which some servers know about the invalidation while others do not"* — and
**does not address requests already in progress**. `[INFERENCE]` A project rule
here is `[POLICY]`, and must be labeled as such; it cannot be dressed as a
protocol requirement.

**A2. `[FACT]` The ecosystem already tolerates revocation lag as a matter of
published fact.** Entra: *"latency of up to 15 minutes might be observed
because of event propagation time."* OpenAI: auth updates *"propagate within 15
minutes, but can potentially take longer."* RFC 7009 concedes propagation
delay outright. `[INFERENCE]` A `MUST` that a stream terminate "on revocation"
would demand of streams a promptness the surrounding identity infrastructure
does not itself guarantee for ordinary requests. Any revocation clause must be
expressed as a documented bound, not as immediacy.

**A3. `[FACT]` Microsoft tried the obvious cheap fix and reports that it
failed.** *"Microsoft experimented with the 'blunt object' approach of reduced
token lifetimes but found they degrade user experiences and reliability without
eliminating risks."* In CAE sessions they went the other way: *"Token lifetime
increases to long-lived, up to 28 hours."* `[INFERENCE]` This is the strongest
single argument against clause 2. It is partly answered by noting that
Microsoft replaced short lifetimes with a *bounded session plus a revocation
channel*, not with nothing — but the caution stands.

**A4. `[COMPARATIVE]` Nobody in the deep-dive set does it, and the one project
asked to do it declined.** Zero of three AI providers document any mid-stream
authorization behavior; five of eight standard references have no streaming
surface at all; Kubernetes closed the nearest request as **not planned**; and
the Vertex behavior is filed by users as a **bug to be fixed**, not a property
to be kept. `[INFERENCE]` A rule mandating mid-stream re-evaluation would put
this standard's conformance surface ahead of every surveyed implementation —
the `P6-D1` situation again, but with a mechanism cost rather than a naming
cost.

**A5. `[INFERENCE]` The cost of true mid-stream re-evaluation is architectural,
not incremental.** It requires the streaming handler to hold a reference to the
authorization decision, to poll or subscribe to a change signal, and to
interleave that with frame production — and, for the opaque-token default
`R8.10` prescribes, to introspect repeatedly, converting one introspection per
request into N per stream. `research/CONSTRAINTS.md` forbids prescribing
deployment platforms, and a rule that in practice can only be discharged with a
CAEP receiver or a shared revocation cache edges toward exactly that.

**A6. `[FACT]` Bounding by credential lifetime has a real functional cost and
a documented failure mode.** Anthropic's shortest key preset is 3 hours, but
App Attest tokens expire after one hour and federation tokens are refreshed
automatically by the SDK; Kubernetes' bound tokens default to one hour with
kubelet rotation at 80 percent of TTL. `[INFERENCE]` Under a credential-bound
rule, a legitimate long generation or long watch is cut off mid-work through no
fault of the caller — which is the behavior the Vertex reporters are asking
Google to remove.

**A6a. `[INFERENCE]` Name who becomes nonconformant, as `P6-D1` did.** The
clearest precedent in this report would fail clause 2. Kubernetes bounds a
watch by `--min-request-timeout` — a server-side number **unrelated to
credential lifetime**, floor 1800 seconds, randomized above it — while its
bound service-account tokens default to a one-hour lifetime. Nothing in the
Kubernetes documentation retrieved says the API server ends an established
watch at the token's `exp`, and issue 120621's "closed as not planned"
disposition points the other way, so a watch opened with a one-hour token
plausibly outlives that token by design. **This is inference, not a verified
fact** — no Kubernetes page states either behavior — but it is the honest
reading, and it means clause 2 would make the field's most battle-tested watch
implementation nonconformant on this rule. `P6-D1` accepted an equivalent cost
knowingly and recorded it rather than softening the rule; the same choice is
proposed here, and clause 2's **moderate** confidence in §5.3 is where the cost
is priced.

**A7. `[INFERENCE]` The blast radius is smaller than it first appears, for one
class of stream.** A stream over a *fixed* result set — an export, a stored
operation result — delivers only data the principal was authorized to receive
at establishment. The severe case is the open-ended stream over a live source
(a watch, an event tail), where post-revocation data keeps arriving. A rule
that does not distinguish these two over-charges the first.

### 4.3 Where the two sides actually meet

`[INFERENCE]` Every FOR item that has a shipped implementation behind it —
F1, F5, F6 — implements **bounded lifetime**, not continuous re-evaluation.
Every AGAINST item with real force — A3, A4, A5 — is aimed at **continuous
re-evaluation**, not at bounding. The evidence does not conflict about what to
do; it converges on bounding and diverges only on whether re-evaluation should
be added on top. That convergence is what the proposed rule is built around,
and it is why the strengths differ per clause.

---

## 5. Proposed rule text, classification, and confidence

### 5.1 Is disclosure-only sufficient?

**Proposed answer: no, but disclosure is a necessary component.** `[POLICY]`

The project has ruled both ways, and the two precedents are distinguishable on
one axis — whether an authority exists to make normative:

| Precedent | Ruling | Why |
| --- | --- | --- |
| Keep-alive (`ST-016`, drafted into `R13.5`) | **Disclosure only.** No interval mandated | *"There is no protocol mechanism to point at"* — the only authority is an authoring note in a Living Standard, and the field shows three mechanisms across three references |
| Conflicting `stream` modifier (`P6-D1` amendment) | **Disclosure declined.** Rejection mandated | The weaker form *"would mandate rejection in the easy case... and merely ask for documentation in the hard one"* |

`[INFERENCE]` This item resembles the second, not the first, on three counts.
First, an authority does exist for the underlying obligation: `R8.6` requires
every request to be authorized deny-by-default, and RFC 9068 §4 requires the
current time to precede `exp` at validation — so delivering data to a principal
whose credential has expired is not an unruled choice, it is an unsatisfied
obligation that §13 currently lets an API describe its way out of. Second, the
harm is a security harm with a named beneficiary: a revoked or expired
principal continues receiving data. Third, disclosure-only produces a
conformance note that reads "we do not re-evaluate authorization mid-stream"
and satisfies `R1.7` while changing nothing — the "conformant by confession"
outcome `P6-D1` refused.

**What disclosure-only *does* buy, and is kept:** a client cannot plan for
truncation-on-expiry it was never told about, exactly as `R13.5` argues for
keep-alives. So the documentation duty is clause 3 of the rule, not the whole
of it.

### 5.2 The proposed rule

Provisional principle **`ST-021`**, continuing `baseline-04`'s `ST-*` series;
drafting target **a new rule `R13.12` in §13**, scoped by the `streaming`
applicability switch. The exact subsection is a drafting choice and is not part
of this proposal: §13 currently numbers rules sequentially through its
subsections, with `R13.11` alone in §13.5, so a new **§13.6** preserves that
ordering while placing the rule inside §13.2 would not. A second, smaller
amendment is proposed for `R8.10` in §5.4.

> **R13.12** An API that streams MUST publish a **maximum stream duration**,
> and MUST end a stream that reaches it with the documented terminal frame
> (R13.6) rather than by closing the connection silently.
>
> A stream MUST NOT continue past the expiry of the credential that authorized
> it. Where the credential presented at establishment carries or implies an
> expiry, the server MUST end the stream at or before that expiry, delivering
> an `error` frame (R13.7) whose problem type states that the credential
> expired, and MUST NOT rely on the maximum stream duration to bound it
> instead. Where the credential carries no expiry, the maximum stream duration
> is the bound.
>
> An API SHOULD end an in-flight stream when the authorization that permitted
> it is revoked or withdrawn, and MUST document its revocation posture as an
> upper bound on how long a stream may continue after revocation — never as an
> assertion that revocation is immediate. An API that does not terminate on
> revocation at all MUST say so, and its maximum stream duration is then the
> exposure window it is publishing.
>
> Resumption (R13.10) is a new request and is authorized as one (R8.6); an API
> MUST NOT treat a resumption position as evidence of authorization.
>
> > Provenance: `ST-021` · research leaf `baseline-04c` under §13.4 item 2 ·
> > project policy `[POLICY]` throughout, composing ratified R8.5, R8.6, and
> > R8.10 with R13.6, R13.7, and R13.10; grounded in RFC 9068 §4 for the
> > expiry clause · confidence moderate-high on clauses 1 and 3, moderate on
> > clause 2.

### 5.3 Per-clause strength, classification, and confidence

| Clause | Strength | Class | Why not stronger | Why not weaker | Confidence |
| --- | --- | --- | --- | --- | --- |
| Published maximum stream duration | **MUST** | `[POLICY]`, evidenced `[COMPARATIVE]` by three independent decade-old systems (F6) | It is a published number, not a mandated number — no citable authority fixes a value, exactly as `R13.5` mandates no keep-alive interval | Without a ceiling there is no exposure window to reason about, and a no-expiry API key leaves clause 2 vacuous | **moderate-high** |
| Stream MUST NOT outlive the credential | **MUST** | `[POLICY]`, grounded in RFC 9068 §4 and R8.6 | Not a protocol requirement: no RFC governs an in-progress response, so this cannot be `[FACT]`. It also overrules the field's clearest precedent — a Kubernetes watch plausibly outlives its bound token (A6a) — which is why confidence is only moderate | A SHOULD would be discharged by a sentence in a conformance note, which §5.1 rejects; and the server already knows `exp`, so the mechanism cost is one comparison | **moderate** |
| End on revocation | **SHOULD** | `[POLICY]` | A MUST would demand of streams a promptness the identity layer does not guarantee for ordinary requests (A2), and would in practice require a revocation-signal receiver, brushing the deployment-platform boundary (A5) | Silence leaves `R8.10`'s revocation posture unhonored on the streaming surface (F4) | **moderate-high** |
| Document the revocation posture as a bound | **MUST** | `[POLICY]`, modeled on `R13.5`'s keep-alive disclosure | — | A client cannot plan for an exposure window it was never told about | **high** |
| Resumption is authorized as a new request | **MUST** | `[INFERENCE]` from ratified R8.6; near-definitional | — | It closes the one place where a client-supplied position could be mistaken for a capability token | **high** |

### 5.4 The consequential amendment this rule implies

`[POLICY]` `R8.10`'s **token-format axis** currently reads that a
client-visible JWT must be *"paired with a revocation-propagation plan"* and
that an *"instant-revocation SLA"* is a trigger to stay opaque. Both phrases
promise a revocation posture that the streaming surface does not deliver.
Proposed: the axis gains one sentence noting that where the `streaming` switch
is on, the revocation-propagation plan must state its effect on in-flight
streams, pointing at `R13.12`. This is an addition to an existing row, not a
new axis; a sixth axis is declined in §6.

**Overlap flagged, not poached.** Clause 1 supplies a published maximum stream
duration, which is also the missing ceiling named in §13.4's fifth item
("Resource ceilings for streams"). This leaf proposes the number as an
*authorization* bound only; per-principal stream concurrency is that item's
question and is deliberately left alone. If both items are ruled in one walk,
the duration ceiling should be stated once and cited twice, never drafted
twice.

---

## 6. Declined alternatives

**D1. Disclosure-only — an API states its posture and no mechanism is
required.** Declined per §5.1: an authority for the underlying obligation
exists, the harm is a security harm, and the resulting conformance note would
read as compliance while changing nothing. **Retained in part** as clause 3.

**D2. `MUST` re-evaluate authorization mid-stream on a fixed interval.**
Declined on four grounds: no authority (A1); the surrounding identity
infrastructure publishes 15-minute propagation as normal (A2); the largest
implementer reports the analogous blunt approach as having failed (A3); and
zero of the surveyed implementations do it (A4). A fixed interval would also
be an invented number — the objection that kept `R13.5` from mandating a
keep-alive interval.

**D3. A sixth `R8.10` deployment-profile axis, "stream authorization
lifetime", with a default and flip triggers.** Genuinely attractive: §8.3 is
where a risk-tiered default belongs, and the shape would be familiar. Declined
because §8.3's five axes each govern a **deployment-wide** choice made once
per API (token format, replay window, rate-limit posture), while this is a
**per-response** obligation gated by the `streaming` switch, and because
`baseline-03g`'s axis defaults each rest on a threat-model literature this leaf
does not have — there is no OWASP entry, no FAPI profile, and no RFC for
stream authorization lifetime. Placing it in §13 keeps the progressive
disclosure `P6-D0` ratified: an API that does not stream never reads it.
**Partially adopted** as the one-sentence `R8.10` amendment in §5.4.

**D4. Require a periodic in-band re-authentication frame — the client resends
its credential on the stream, as AWS Transcribe chains signatures.** Declined:
Transcribe's chain runs in the request direction of a bidirectional exchange
(§3.4), which is not the response-streaming case §13 governs; a response
stream has no client-to-server channel to carry a fresh credential without
inventing one. Inventing one would mint a reserved frame type on a single
non-analogous exemplar, which is the evidentiary bar `ST-009` was already
flagged for barely clearing.

**D5. Bound every stream to a short fixed maximum, such as five minutes, on
the Kinesis model.** Declined as a number this project cannot source. Kinesis'
5 minutes, Kubernetes' 1800-second floor, and Entra's 1 hour differ by more
than an order of magnitude and reflect three different workloads. The rule
requires a **published** maximum and leaves the value to the API, exactly as
`R13.11` requires a published maximum hold duration for long-polling without
fixing it.

**D6. Terminate the stream by closing the connection when the credential
expires.** Declined because it is indistinguishable from truncation under
`R12.10`, which would make every expiry look like a network failure and invite
the non-idempotent replay `R12.10` forbids. The `error` frame path (`R13.7`)
already exists and already carries a catalogued `type` and `code`.

**D7. Require `401` semantics mid-stream.** Structurally impossible and
already settled: the status is committed as `200`, and `P6-D2` ratified that
the in-band problem object **omits** `status`. Recorded only because a reviewer
will ask.

---

## 7. What could not be verified

1. **The kube-apiserver flag text was read from a Debian manpage mirror, not
   from `kubernetes.io`.** The canonical reference page
   (`https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/`)
   was fetched and its options list was **truncated before** the descriptions
   of `--min-request-timeout`, `--request-timeout`, and
   `--service-account-max-token-expiration`. The Debian rendering is a
   third-party mirror of the same generated reference; the flag names and the
   1800-second and 1m0s defaults should be re-confirmed against the canonical
   page before ratification.

2. **No primary Google source was found for a Gemini REST streaming maximum
   duration.** The widely repeated five-minute claim traces to community
   threads and failed verification. The Vertex-hosted numbers (10-minute
   default request timeout, 3600-second maximum on dedicated public endpoints,
   10-minute bidirectional-streaming maximum on Agent Engine Runtime) came from
   a domain-restricted search and were **not** re-fetched from their individual
   pages. Treat them as unverified until they are.

3. **Whether Vertex AI's `ACCESS_TOKEN_EXPIRED` is emitted on the established
   stream or on an internal retry is not established** by the issue text
   retrieved. This distinction decides whether F5 is evidence of a server-side
   policy or of client-library behavior.

4. **The grpc-go maintainer position was not retrieved.** The issue page
   returned the original report without the comment thread; the claim that
   per-RPC credentials cannot be refreshed mid-RPC is filed as the reporter's
   characterization plus corroboration from the Vertex case, not as a
   maintainer statement.

5. **The OWASP quotations in F7 were re-fetched directly from
   `cheatsheetseries.owasp.org` and are verbatim** — an earlier search-result
   extraction had blended them with non-OWASP pages and was discarded. The
   residual caveat is only that OWASP cheat sheets are living community
   documents with no version marker on the page, so the quotations are dated to
   the 2026-08-10 access and no publication date is claimed.

6. **The RFC negatives rest on prompted extraction, with one cross-check.** RFC
   9700's negative was confirmed **twice, on two different surfaces** — the
   `.txt` at `rfc-editor.org` and the `.html` — both returning no occurrence of
   "streaming" or "long-lived" and no discussion of re-evaluating authorization
   during an in-progress request. The two passes **disagreed about §4.14's
   content**: the first quoted *"they allow the authorization server to issue
   access tokens with a short lifetime and reduced scope"*, the second reported
   that §4.14 contains no access-token-lifetime sentence. The disagreement is
   surfaced rather than averaged; it affects only an incidental quotation in
   A1, not the negative that A1 rests on. **RFC 9068 and RFC 7009 were each
   cross-checked on a second surface as well** — RFC 9068 `.txt` then `.html`,
   RFC 7009 `.html` then `.txt` — and both passes agreed in both cases: no
   occurrence of "streaming" or "long-lived" in either document, no occurrence
   of "connection" in RFC 9068, no discussion in RFC 9068 of validating a token
   more than once for one request, and no discussion in RFC 7009 of requests
   already in progress or connections already established. RFC 7009's
   propagation-delay sentence was returned identically by both passes. **The
   residual exposure is that all six passes used the same extraction model**, so
   a different-model re-check is still the ideal; the two-source minimum is
   nonetheless met for each of the three negatives.

7. **No documented security incident was found** in which a revoked principal
   demonstrably continued receiving data over an open HTTP response stream.
   The Vertex case is the inverse (a stream terminated when users wanted it to
   survive), and the Entra coauthoring text describes the exposure as a
   limitation rather than as an incident. **The threat in §4.1 is argued from
   mechanism and vendor-documented limitations, not from a published breach.**
   A reviewer should weigh it accordingly.

8. **Anthropic's and OpenAI's in-flight behavior was established as a
   documentation absence, not as observed behavior.** No live probe was run —
   the standard's public-repo rules and the absence of credentials preclude it.
   A single controlled experiment (open a stream, revoke the key, observe)
   would convert three verified negatives into three positive facts and is the
   highest-value follow-up available.

9. **Whether any IETF working-group item now addresses this** was not
   re-enumerated. `baseline-04`'s recorded 2026-08-10 baseline is that the
   `httpapi` docket carries three active working-group drafts, none on
   streaming; that baseline is reused here rather than re-probed.

---

## 8. Conditions that would change this recommendation

1. **A deep-dive provider publishes a maximum streaming duration or an
   in-flight revocation behavior.** Converts the central `[COMPARATIVE]`
   negative into a two-sided axis and would likely raise clause 2's confidence.
2. **An IETF item addresses authorization over an in-progress HTTP response.**
   Would move clauses 1 and 2 from `[POLICY]` toward `[FACT]` and could make
   `MUST` mid-stream re-evaluation citable.
3. **CAEP receivers become a commodity gateway feature.** `baseline-03g` found
   the phantom-token swap was not a commodity (one of seven gateways
   documented it natively); if revocation-signal consumption becomes
   configuration rather than integration work, A5 weakens and clause 3 could
   rise to `MUST`.
4. **A published incident of post-revocation data delivery over a stream.**
   Would convert §4.1 from mechanism-argument to demonstrated harm and would
   justify revisiting D2.
5. **Google resolves issue 13533 by keeping streams alive across token
   refresh.** Would remove the only shipped exemplar of credential-bound stream
   lifetime and would materially weaken clause 2.
