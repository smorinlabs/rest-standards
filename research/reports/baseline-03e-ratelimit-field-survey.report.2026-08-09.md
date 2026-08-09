# Baseline 03e — Rate-Limit Header Convention Survey

*Narrow leaf under `baseline-03`, companion to `baseline-03f` (draft
trajectory). Surveys what the industry actually emits for rate-limit
communication across four populations — the modern AI platforms (OpenAI,
Anthropic, Gemini — mandatory in all evaluations), the legacy 8, newer
dev-tools APIs, and gateways — before ratifying `OP-010`. Run 2026-08-09;
all sources accessed 2026-08-09.*

**Evidence labels:** `[FACT]` = sourced to a primary vendor/standards
document; `[COMPARATIVE]` = cross-source comparison; `[INFERENCE]` =
reasoning. Absences are reported as results.

## 0. The distinction that governs every row

Before any vendor can be classified, "the IETF fields" must be split in two,
because the draft changed shape mid-life.

`[FACT]` **draft-11** (current; 23 May 2026; expires 24 November 2026;
Active I-D, HTTPAPI WG) defines exactly **two** RFC 8941 structured fields:

```
RateLimit-Policy: "burst";q=100;w=60,"daily";q=1000;w=86400
RateLimit: "default";r=50;t=30
```

`RateLimit` parameters: `r` (available quota, required), `t` (effective
window in seconds), `pk` (partition key). `RateLimit-Policy` parameters: `q`
(quota, required), `qu` (quota unit), `w` (window seconds), `pk`. On
`Retry-After`: *"If a response contains both the Retry-After and the
RateLimit header fields, the Retry-After field value SHOULD NOT reference a
point in time earlier than the end of the effective window."*
— https://www.ietf.org/archive/id/draft-ietf-httpapi-ratelimit-headers-11.html

`[FACT]` **draft-03** (7 March 2022) defined **three separate fields**:

```
RateLimit-Limit: 100, 100;w=60
RateLimit-Remaining: 99
RateLimit-Reset: 50
```

`RateLimit-Reset` is `delay-seconds`, chosen because it *"does not rely on
clock synchronization and is resilient to clock adjustment."*
`RateLimit-Limit` and `RateLimit-Reset` are REQUIRED, `RateLimit-Remaining`
RECOMMENDED.
— https://datatracker.ietf.org/doc/html/draft-ietf-httpapi-ratelimit-headers-03

`[FACT]` The pivot is **draft-07**, recorded by the datatracker history as
*"Major refactoring using Structured Fields format"*; draft-08 added Problem
Types. `[INFERENCE]` Therefore draft-06 and earlier = the trio; draft-07 and
later = the structured pair. **The trio field names do not appear in
draft-11 at all** — independently confirmed by grepping the full draft-11
text (1,960 lines). An implementation of the trio is not a partial
implementation of `OP-010` as written; it is an implementation of a
superseded document.
— https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/

`[FACT]` Draft-11's own Appendix D names the de-facto convention it exists
to replace: *"Commonly used header field names are: `X-RateLimit-Limit`,
`X-RateLimit-Remaining`, `X-RateLimit-Reset`"*, with variants
*"x-ratelimit-limit-minute, x-ratelimit-limit-hour, x-ratelimit-limit-day."*
Its problem statement: *"A major interoperability issue in throttling is the
lack of standard headers, because each implementation associates different
semantics to the same header field names; header field names proliferate."*

---

## 1. Comprehensive header table

Convention categories: **X-trio** = the canonical
`X-RateLimit-{Limit,Remaining,Reset}`; **X-variant** = X-prefixed but
dimension-suffixed or partial; **proprietary** = vendor namespace;
**IETF-trio** = unprefixed `RateLimit-{Limit,Remaining,Reset}` (draft-06 and
earlier); **IETF-structured** = `RateLimit`/`RateLimit-Policy` (draft-07+,
i.e. what `OP-010` mandates); **Retry-After only**; **none**.

| Provider | Header names (verbatim) | Reset semantics | Scope dimensions | Retry-After on 429 | IETF fields? | Category | Source |
|---|---|---|---|---|---|---|---|
| **OpenAI** | `x-ratelimit-limit-requests` `x-ratelimit-limit-tokens` `x-ratelimit-remaining-requests` `x-ratelimit-remaining-tokens` `x-ratelimit-reset-requests` `x-ratelimit-reset-tokens` `x-ratelimit-limit-project-tokens` `x-ratelimit-remaining-project-tokens` `x-ratelimit-reset-project-tokens` | **Go-style duration string** (`1s`, `6m0s`, `3s`) | requests + tokens; RPM/RPD/TPM/TPD; per-model; org **and** project level | Yes, seconds (`56`), *"when present"* | No | X-variant | developers.openai.com/api/docs/guides/rate-limits |
| **Anthropic** | `anthropic-ratelimit-requests-{limit,remaining,reset}` `anthropic-ratelimit-tokens-*` `anthropic-ratelimit-input-tokens-*` `anthropic-ratelimit-output-tokens-*` `anthropic-priority-input-tokens-*` `anthropic-priority-output-tokens-*` `retry-after` | **RFC 3339 timestamp** | RPM/ITPM/OTPM per model class; org tier (Start/Build/Scale/Custom); workspace sub-limits; Priority Tier | Yes, `retry-after` seconds | No | proprietary | platform.claude.com/docs/en/api/rate-limits |
| **Google Gemini** | **none** | n/a | RPM, TPM, RPD, spend-based (rolling 10 min); per tier | Not documented | No | none | ai.google.dev/gemini-api/docs/rate-limits |
| **Stripe** | `Stripe-Rate-Limited-Reason` (429 only) | n/a — no quota state at all | per-account RPS, per-endpoint, per-resource, concurrency | **Not documented** | No | proprietary | docs.stripe.com/rate-limits |
| **GitHub** | `x-ratelimit-limit` `x-ratelimit-remaining` `x-ratelimit-used` `x-ratelimit-reset` `x-ratelimit-resource` | **UTC epoch seconds** | requests/hour; per resource class | `retry-after` on secondary limits only | No | X-trio | docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api |
| **Twilio** | `Twilio-Concurrent-Requests` `Twilio-Request-Duration` `Twilio-Request-Id` | n/a | concurrency | Not documented | No | proprietary | twilio.com/docs/usage/rest-api-best-practices |
| **Shopify (REST)** | `X-Shopify-Shop-Api-Call-Limit: 32/40` `Retry-After: 2.0` | n/a (bucket ratio) | leaky bucket, per app per shop | Yes, decimal seconds | No | proprietary | shopify.dev/docs/api/admin-rest/usage/rate-limits |
| **Shopify (GraphQL)** | **none** — `extensions.cost.throttleStatus` in body (`maximumAvailable`, `currentlyAvailable`, `restoreRate`) | n/a | query cost points | Not specified | No | none | shopify.dev/docs/api/usage/rate-limits |
| **Microsoft Graph (general)** | `Retry-After: 10` | n/a | per tenant / per app | Yes, seconds | No | Retry-After only | learn.microsoft.com/en-us/graph/throttling |
| **Microsoft Graph (identity/access)** | `x-ms-throttle-limit-percentage` (0.8–1.8) `x-ms-resource-unit` `x-ms-throttle-scope` `x-ms-throttle-information` | n/a | resource units | Yes (some resources excepted) | No | proprietary | learn.microsoft.com/en-us/graph/throttling-limits |
| **Microsoft SharePoint Online / Graph beta** | `RateLimit-Limit: 1200` `RateLimit-Remaining: 120` `RateLimit-Reset: 5` | **delta-seconds** | app 1-minute resource-unit limit; emitted only at ≥80% consumption; app permissions only; best-effort | Yes; on 429 `Retry-After: 31` **matches** `RateLimit-Reset: 31` | **YES — draft-03 trio, explicitly cited** | IETF-trio | learn.microsoft.com/en-us/sharepoint/dev/general-development/how-to-avoid-getting-throttled-or-blocked-in-sharepoint-online + devblogs.microsoft.com (2022-10-18) |
| **Google APIs / AIP** | **none** | n/a | quota per project/region | Not defined | No | none | google.aip.dev/193 |
| **Zalando guidelines (rule 153)** | prescribes `X-RateLimit-Limit` `X-RateLimit-Remaining` `X-RateLimit-Reset` | **"relative time in seconds"** | n/a (guideline) | permitted as alternative (HTTP-date or seconds) | No — cites RFC 6585 only | X-trio | github.com/zalando/restful-api-guidelines `chapters/http-status-codes-and-errors.adoc` |
| **Microsoft/Azure REST API Guidelines** | **no rate-limit rule exists**; `retry-after` only for long-running operations | n/a | n/a | n/a | No | none | github.com/microsoft/api-guidelines `vNext/azure/Guidelines.md` |
| **Cloudflare API** | `Ratelimit: "default";r=50;t=30` `Ratelimit-Policy: "burst";q=100;w=60` `retry-after` | `t` = *"the time next window resets"* — **unit not stated in the doc** | 1200 req/5 min per user/account token; 200/s per IP; GraphQL by query cost | Yes, *"number of seconds, rounded up"*, 429 only | **YES — structured form, but no citation** | **IETF-structured** | developers.cloudflare.com/fundamentals/api/reference/limits/ + cloudflare-docs repo source |
| **Resend** | `ratelimit-limit` `ratelimit-remaining` `ratelimit-reset` `retry-after` (+ `x-resend-daily-quota`, `x-resend-monthly-quota`) | **delta-seconds** (*"How many seconds until the limits are reset"*) | 10 req/s per team across all keys; daily/monthly send quotas | Yes, seconds | **YES — trio, explicitly citing draft-06** | IETF-trio | resend.com/docs/api-reference/rate-limit |
| **GitLab** | `RateLimit-Limit: 60` `RateLimit-Name: throttle_authenticated_api` `RateLimit-Observed: 67` `RateLimit-Remaining: 33` `RateLimit-Reset: 1609844400` `RateLimit-ResetTime: Tue, 05 Jan 2021 11:00:00 GMT` (429) | **Unix epoch seconds** — *contradicts* draft delta-seconds; plus an RFC 2616 HTTP-date twin | per user / per IP throttles, named | Yes, `Retry-After: 30` | **Names yes, semantics no** | IETF-trio (non-conformant) | docs.gitlab.com/administration/settings/user_and_ip_rate_limits/ |
| **Supabase (Management API)** | `X-RateLimit-Limit` `X-RateLimit-Remaining` `X-RateLimit-Reset` | delta-seconds | 120 req/min per user per project/org; identity = OAuth App ID → User ID → IP | Not documented | **No — but the page links the IETF draft and calls them *"official HTTP specification headers"*** | X-trio (false claim) | supabase.com/docs/reference/api/introduction |
| **Supabase (Auth)** | **none** | n/a | per-IP token bucket, capacity 30 | Not documented | No | none | supabase.com/docs/guides/auth/rate-limits |
| **Discord** | `X-RateLimit-Limit: 5` `X-RateLimit-Remaining: 0` `X-RateLimit-Reset: 1470173023` `X-RateLimit-Reset-After: 1` `X-RateLimit-Bucket: abcd1234` `X-RateLimit-Global` `X-RateLimit-Scope` | **both**: `Reset` = epoch seconds (fractional); `Reset-After` = delta-seconds (fractional) | per bucket/route; global vs per-route; scope `user`\|`global`\|`shared` | Yes, fractional seconds | No | X-trio (+ dual reset) | docs.discord.com/developers/topics/rate-limits |
| **Atlassian Jira Cloud** | `X-RateLimit-Limit: 100000` `X-RateLimit-Remaining: 0` `X-RateLimit-Reset: 2025-10-08T15:00:00Z` `X-RateLimit-NearLimit` `RateLimit-Reason: jira-quota-global-based` | **ISO 8601 timestamp** | global quota | Yes, `Retry-After: 1847` | No | X-trio | developer.atlassian.com/cloud/jira/platform/rate-limiting/ |
| **Linear** | `X-RateLimit-Requests-{Limit,Remaining,Reset}` `X-RateLimit-Endpoint-Requests-{Limit,Remaining,Reset}` `X-RateLimit-Endpoint-Name` `X-Complexity` `X-RateLimit-Complexity-{Limit,Remaining,Reset}` | **UTC epoch milliseconds** | requests + GraphQL complexity; global + per-endpoint; per authenticated user | **No — returns HTTP 400** with `RATELIMITED` body code | No | X-variant | linear.app/developers/rate-limiting |
| **Vercel** | `X-RateLimit-Limit` `X-RateLimit-Remaining` `X-RateLimit-Reset` | **not documented** | per team, per project | Not documented | No | X-trio | vercel.com/docs/rest-api |
| **Notion** | `Retry-After` | delta-seconds (*"integer number of seconds"*) | per integration; body `rate_limit_reason` | Yes — and also on **529** | No | Retry-After only | developers.notion.com/reference/request-limits |
| **Slack** | `Retry-After: 30` | delta-seconds | per-method Tier 1–4 + Special | Yes | No | Retry-After only | docs.slack.dev/apis/web-api/rate-limits/ |
| **Polar** | `Retry-After` | **unit not stated** | per org/customer or OAuth2 client; per environment | Yes | No | Retry-After only | polar.sh/docs/api-reference/introduction |
| **Groq** | `x-ratelimit-limit-requests: 14400` `x-ratelimit-limit-tokens: 18000` `x-ratelimit-remaining-requests: 14370` `x-ratelimit-remaining-tokens: 17997` `x-ratelimit-reset-requests: 2m59.56s` `x-ratelimit-reset-tokens: 7.66s` `retry-after: 2` | **Go-style duration string** | requests (RPD) + tokens (TPM); per org, per model | Yes, 429 only | No | X-variant (OpenAI-shaped) | console.groq.com/docs/rate-limits |
| **Together AI** | `x-ratelimit-reset` — **429 responses only** | delta-seconds | requests + tokens (`dynamic_request_limited`/`dynamic_token_limited`); per org, per model | No — `x-ratelimit-reset` substitutes | No | X-variant | docs.together.ai/docs/serverless/rate-limits |
| **xAI** | **none** | n/a | per team tier, per model (Console only) | Not documented | No | none | docs.x.ai/developers/rate-limits |
| **Mistral** | **none** | n/a | TPM, RPS, audio sec/min, OCR pages/min; per model, per workspace | Not documented | No | none | docs.mistral.ai/admin/billing-usage/usage-limits |
| **Cohere** | **none** | n/a | per key type (trial/production), per endpoint | Not documented | No | none | docs.cohere.com/docs/rate-limits + /reference/errors |

### Gateways and infrastructure

| Product | Headers emitted (verbatim) | Default? | Reset semantics | IETF? Which revision | Source |
|---|---|---|---|---|---|
| **Kong `rate-limiting`** | `RateLimit-Limit` `RateLimit-Remaining` `RateLimit-Reset` **and** `X-RateLimit-{Limit,Remaining}-{Second,Minute,Hour,Day,Month,Year}` `Retry-After` | **On by default**; `hide_client_headers=true` suppresses. There is **no** `header_type` option | **delta-seconds** (`max(1, window - floor(...))`); on 429 `Retry-After` = same value | **IETF-trio.** Docs cite the WG Internet-Draft with **no revision number**, *"may change in the future to respect specification updates"*; CHANGELOG 2.0.0 pins origin to `draft-polli-ratelimit-headers-01` | developer.konghq.com/plugins/rate-limiting/ + `kong/plugins/rate-limiting/handler.lua` |
| **Kong `rate-limiting-advanced`** | same set | same | delta-seconds | same generic citation | developer.konghq.com/plugins/rate-limiting-advanced/reference/ |
| **Envoy `ratelimit` + `local_ratelimit`** | `X-RateLimit-Limit` (value form `10, 10;w=1;name="per-ip", 1000;w=3600`) `X-RateLimit-Remaining` `X-RateLimit-Reset` | `enable_x_ratelimit_headers`; enum is exactly `OFF` (default) and `DRAFT_VERSION_03` | delta-seconds | **NOT an IETF-field emitter.** X-prefixed names carrying draft semantics; the cited document is `draft-polli-ratelimit-headers-03` — the *individual* pre-WG draft, superseded at rev 05 in 2020 | envoyproxy.io API ref + `api/envoy/extensions/common/ratelimit/v3/ratelimit.proto` |
| **Tyk Gateway** | `X-RateLimit-Limit` `X-RateLimit-Remaining` `X-RateLimit-Reset` | `rate_limit_response_headers`: `quotas` (default) or `rate_limits`; added 5.13.0 | **unix epoch seconds in both modes** (source: `internal/rate/headers.go`; the header-constant comment saying "seconds until reset" is stale) | No — zero matches for `RateLimit-Policy`/`draft-ietf-httpapi` in the repo | Tyk docs + repo |
| **Azure API Management** | `Retry-After` (default name, overridable) + **operator-named** `remaining-calls-header-name` / `total-calls-header-name` (no defaults — emit nothing unless set) | Retry-After on by default; quota headers off | delta-seconds | **No.** Cannot produce the structured field under any configuration | learn.microsoft.com/en-us/azure/api-management/rate-limit-by-key-policy |
| **Zuplo** | `Retry-After` only | `headerMode`: `none` \| `retry-after` (default `retry-after`) | delta-seconds | No | Zuplo reference docs |
| **AWS API Gateway** | **none** on the data-plane 429 | n/a — arbitrary headers only via manual `PutGatewayResponse` | n/a | No | docs.aws.amazon.com |
| **Apigee X / Edge** | **none** — JSON fault body only | n/a | `ratelimit.*.expiry.time` is epoch-ms, internal flow variable only | No | Apigee policy docs |
| **nginx OSS / NGINX Plus** | **none** | no header directive exists | n/a | No | `ngx_http_limit_req_module` docs |
| **HAProxy** | **none** | 0 matches for `RateLimit`/`X-RateLimit`/`Retry-After` in the 31,010-line 3.0 config manual | n/a | No | HAProxy configuration manual |
| **Traefik** | **none** | no header option in the middleware schema | n/a | **Explicitly declined**: issue #6113 *"Support RateLimit headers Internet Draft"* open since 2019-12-30; implementation PR #6111 **closed unmerged 2020-08-18** | doc.traefik.io + repo issues |
| **Cloudflare Rate Limiting Rules (WAF)** | **none documented**; legacy product documented `Retry-After` | n/a | delta-seconds (legacy) | No | Cloudflare WAF docs |
| **Cloudflare Workers rate-limit binding** | **none** — `limit()` returns `{ success }` | n/a | n/a | No | Workers docs |

### Libraries

| Library | Behavior | Source |
|---|---|---|
| **express-rate-limit** | `standardHeaders` accepts `'draft-6'` (separate `RateLimit-Policy`, `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`), `'draft-7'` and `'draft-8'` (**combined structured `RateLimit` + `RateLimit-Policy`**). *"draft-9 contains no functional or format changes from draft-8."* `legacyHeaders` emits `X-RateLimit-Limit/-Remaining/-Reset` and **defaults to `true`** | express-rate-limit.mintlify.app/reference/configuration |

---

## 2. Findings by population

### Population 1 — the mandatory three AI platforms

`[FACT]` **All three diverge from each other and none emits IETF fields.**
Each picked a different reset encoding:

```
OpenAI     x-ratelimit-reset-tokens: 6m0s                 (Go duration string)
Anthropic  anthropic-ratelimit-tokens-reset: <RFC 3339>   (absolute timestamp)
Gemini     (no header at all)
```

`[FACT]` OpenAI and Anthropic both split quota by **requests vs tokens** — a
dimension the trio convention cannot express, which is why both fell back to
name-mangling (`-requests`/`-tokens` suffix at OpenAI,
`-requests-`/`-input-tokens-`/`-output-tokens-` infix at Anthropic).
`[COMPARATIVE]` Draft-11 *can* express this: `RateLimit-Policy` carries a
`qu` (quota unit) parameter with values such as `"requests"`. Neither vendor
uses it.

`[FACT]` Anthropic is the only one of the three that does not use a
`ratelimit` field-name stem at all; it namespaces everything under
`anthropic-`. Its `anthropic-ratelimit-tokens-*` headers report *"the values
for the most restrictive limit currently in effect."*

`[FACT]` Gemini is a **documented absence**, confirmed across three Google
surfaces: the Gemini rate-limits page, the Gemini troubleshooting page, and
AIP-193. Google directs developers to AI Studio to read their limits rather
than to a response header. No `Retry-After` is documented either.

`[COMPARATIVE]` Groq's headers are a byte-level clone of OpenAI's shape,
including the duration-string reset — evidence that the "OpenAI-compatible"
surface is propagating a rate-limit convention alongside the
request/response schema.

### Population 2 — the legacy 8

`[FACT]` **One premise correction: "zero IETF adoption among the legacy 8"
is narrowly false.** Microsoft SharePoint Online / Graph beta emits the
unprefixed trio and cites the IETF work by name:

> *"SharePoint Online also returns the [IETF RateLimit headers] for selected
> limits in certain conditions… These headers are currently in **beta** and
> subject to change. At the time when the headers were adopted, the IETF
> specification was in draft. The current implementation is based on the
> **draft-03** of the IETF specification."*

Scope is narrow in four documented ways: beta, best-effort, app permissions
only, emitted only at ≥80% consumption. `[INFERENCE]` It is adoption of a
superseded revision, so it is **not** evidence for `OP-010` as written —
draft-11 does not define those field names.

`[FACT]` Microsoft is simultaneously the most and least consistent vendor
surveyed: Graph identity/access emits `x-ms-throttle-*`, general Graph emits
only `Retry-After`, SharePoint emits the IETF trio, and the Azure REST API
Guidelines contain **no rate-limit rule at all**.

`[FACT]` **Stripe emits no quota state.** A 429 plus
`Stripe-Rate-Limited-Reason` (values `global-rate`, `endpoint-rate`,
`global-concurrency`, `endpoint-concurrency`, `resource-specific`), no
documented `Retry-After`. `[INFERENCE]` The most-imitated API in the
industry declines the entire convention and tells clients to use exponential
backoff with jitter.

`[FACT]` **Twilio likewise emits no quota state** —
`Twilio-Concurrent-Requests` reports load, not quota.

`[FACT]` Shopify runs two incompatible schemes in one product: REST gets
`X-Shopify-Shop-Api-Call-Limit: 32/40` (a bucket ratio) with
`Retry-After: 2.0`; GraphQL gets no headers and puts cost in
`extensions.cost.throttleStatus`.

`[FACT]` Zalando's rule 153 mandates the X-trio and defines
`X-RateLimit-Reset` as *"The relative time in seconds when the rate limit
window will be reset"* — the opposite of GitHub's epoch reading of the
same-shaped header. It cites RFC 6585 and never mentions the IETF draft.

### Population 3 — newer / dev-tools APIs

`[FACT]` **One structured-field emitter found: Cloudflare's own API.**
Documented headers, verbatim:

```
Ratelimit: "default";r=50;t=30
Ratelimit-Policy: "burst";q=100;w=60
```

`[FACT]` Structurally draft-11; the example values are identical to
draft-11's own. `[FACT]` The page never cites IETF, RFC 8941, or any
revision number. `[INFERENCE]` "Cloudflare adopted the IETF draft" is
inference from syntax, not a vendor claim; the tracked revision cannot be
pinned. `[FACT]` Casing is `Ratelimit`, not `RateLimit` — conformant (HTTP
field names are case-insensitive), but a naive case-sensitive parser would
miss it.

`[FACT]` **Live check, 2026-08-09 18:01 UTC** — the finding is *documented*,
not *observed*: an unauthenticated `GET /client/v4/user` returned
`HTTP/2 400` with no `Ratelimit` headers. `[INFERENCE]` Consistent with the
headers being scoped to authenticated requests; wire behavior unverified —
treat as documented-only.

`[FACT]` **One explicit old-draft emitter with a citation: Resend** — *"in
conformance with the sixth IETF standard draft"*, linking draft-06, emitting
lowercase `ratelimit-limit` / `ratelimit-remaining` / `ratelimit-reset`.
`[INFERENCE]` Cited correctly and conformant to what it cites — but draft-06
predates the structured-fields refactor, so not `OP-010` compliance either.

`[FACT]` **One false standards claim: Supabase.** The Management API
introduces its headers as *"following official HTTP specification headers"*
with the anchor pointing at the IETF draft — then lists `X-RateLimit-*`
names **no revision of that draft has ever defined**. `[INFERENCE]` A client
author trusting the prose would build a draft parser and find nothing to
parse.

`[FACT]` **Discord ships the reset ambiguity's workaround in production**,
emitting both encodings side by side: `X-RateLimit-Reset: 1470173023`
(epoch, fractional) and `X-RateLimit-Reset-After: 1` (delta, fractional).

`[FACT]` **Five surfaces document rate limits with no response headers
whatsoever**: xAI, Mistral, Cohere, Supabase Auth, Google APIs/AIP.

`[FACT]` **Two vendors deliberately withhold state on the happy path.**
Together AI: *"successful requests come back without rate-limit headers"* —
only `x-ratelimit-reset` on a 429; a client cannot pre-emptively throttle at
all. Groq sends `retry-after` only on 429s but the other headers always.

`[FACT]` **Two vendors break the expected 429 contract.** Linear returns
**HTTP 400** with a `RATELIMITED` body code. Notion extends `Retry-After`
handling to **529**.

### Population 4 — gateways and infrastructure

`[FACT]` **No gateway or proxy surveyed emits the draft-11 structured
fields, in any configuration.** Twelve products checked; `RateLimit-Policy`
emitters among them: **zero**.

`[FACT]` **Kong** is the only product emitting the unprefixed trio, on by
default, alongside its own `X-RateLimit-*-{Second…Year}` family — both sets
in the same response. Reset is delta-seconds and equals `Retry-After` on a
429 (source-verified in `handler.lua`). Kong's docs cite the WG draft with
no revision; CHANGELOG 2.0.0 pins the origin to
`draft-polli-ratelimit-headers-01`. `[COMPARATIVE]` Kong's period-suffixed
variants are literally the interoperability failure catalogued in draft-11's
Appendix D.

`[FACT]` **Envoy is not an IETF-field emitter** despite appearances: enum
values exactly `OFF` (default) and `DRAFT_VERSION_03`; emitted names keep
the `X-` prefix; the cited document is `draft-polli-ratelimit-headers-03`,
the individual pre-WG draft superseded in 2020.

`[FACT]` **Traefik declined explicitly and on the record**: issue #6113 open
since 2019-12-30; implementation PR #6111 closed unmerged 2020-08-18.

`[FACT]` **Azure API Management provides the mechanism and no convention**:
quota header names have no defaults — the operator, not the product, is the
standards body.

`[FACT]` **Six products emit nothing natively**: nginx, HAProxy, Traefik,
Apigee, AWS API Gateway (data plane), Cloudflare's own WAF rules / Workers
binding.

`[FACT]` **Tyk emits the X-trio with epoch-seconds reset in both modes** —
and its own header-constant source comment ("seconds until reset") is stale
against the code (`time.Now().Add(limits.Reset).Unix()`). `[COMPARATIVE]` A
product whose own comment disagrees with its own code about reset semantics
is the ambiguity in miniature.

---

## 3. Convention landscape

### 3.1 There is a dominant convention, and it is not the IETF's

`[COMPARATIVE]` Counting the 31 API surfaces in the table (excluding
gateways and libraries):

| Category | Count | Members |
|---|---:|---|
| `X-RateLimit-{Limit,Remaining,Reset}` trio | 6 | GitHub, Discord, Vercel, Supabase Mgmt, Atlassian Jira, Zalando (prescription) — Tyk also emits it but is a gateway, outside this 31-surface count |
| X-prefixed dimension-suffixed variants | 4 | OpenAI, Groq, Linear, Together AI |
| Vendor-proprietary namespace | 5 | Anthropic, Stripe, Twilio, Shopify REST, MS Graph identity |
| **Unprefixed IETF trio (draft-06 or earlier)** | **3** | MS SharePoint (draft-03, cited), Resend (draft-06, cited), GitLab (names only) |
| **IETF structured fields (draft-07+ / draft-11 shape)** | **1** | Cloudflare API |
| `Retry-After` only | 4 | Slack, Notion, Polar, MS Graph general |
| No headers at all | 7 | Gemini, xAI, Mistral, Cohere, Supabase Auth, Google APIs/AIP, Shopify GraphQL |

`[COMPARATIVE]` **The X-prefixed family is the plurality — 10 of 31
(6 trio + 4 variants; corrected 2026-08-09, Tyk removed as a gateway) — but
not a majority, and not stable.** The larger finding: **11 of 31 surfaces
publish no quota state at all** (7 no headers + 4 `Retry-After` only).
`[INFERENCE]` The field has not converged on a rate-limit header convention;
it has converged on `Retry-After` as the only header everyone agrees about,
with quota state treated as an optional vendor extra.

### 3.2 The reset ambiguity, quantified

`[COMPARATIVE]` **Six-plus mutually incompatible encodings appear under
names that look identical:**

| Encoding | Emitters |
|---|---|
| Unix epoch **seconds** | GitHub, Discord (`X-RateLimit-Reset`), GitLab, Tyk |
| Unix epoch **milliseconds** | Linear |
| delta-seconds | Zalando (prescribed), Kong, Envoy, MS SharePoint, Resend, Supabase, Together, Discord (`Reset-After`), draft-03 and draft-11 |
| Go-style duration string (`6m0s`) | OpenAI, Groq |
| RFC 3339 timestamp | Anthropic |
| ISO 8601 timestamp | Atlassian Jira |
| RFC 2616 HTTP-date (second header) | GitLab (`RateLimit-ResetTime`) |
| Unstated | Vercel, Polar, Cloudflare (`t`) |

`[FACT]` GitHub's `x-ratelimit-reset` is *"in UTC epoch seconds"*.
Zalando's `X-RateLimit-Reset` is *"the relative time in seconds"*. **Same
header name, opposite meaning, both documented as authoritative.** The
classic ambiguity is live today, not historical.

`[FACT]` GitLab shows the sharper version: the IETF's own field names with a
Unix-epoch value — exactly what draft-03 chose *not* to do. `[INFERENCE]`
**Emitting the IETF names does not imply IETF semantics.** A client that
detects the unprefixed names and assumes delta-seconds will be wrong on
GitLab by roughly 1.6 billion.

### 3.3 Where the IETF fields actually appear in the wild

`[FACT]` Four appearances, three different categories, only one
`OP-010`-shaped:

1. **Cloudflare's API** — structured `Ratelimit`/`Ratelimit-Policy`,
   documented, no revision cited, not wire-verified here. **The only
   draft-07+-shaped vendor emitter found across all four populations.**
2. **Microsoft SharePoint Online / Graph beta** — the trio, explicitly
   draft-03, beta, best-effort, conditional.
3. **Resend** — the trio, explicitly draft-06.
4. **Kong** — the trio, on by default, generic citation, origin
   `draft-polli-…-01`.

Plus one library: **express-rate-limit** — the only implementation surveyed
that can emit draft-7/8 structured fields — shipping with `legacyHeaders`
(the `X-RateLimit-*` trio) **defaulting to `true`**.

`[FACT]` **Zero emitters of `RateLimit-Policy` as a rate-limiting product
feature. Zero implementations anywhere citing draft-11, draft-10, or
draft-09.** Highest revision cited by any implementation surveyed:
**draft-06** (Resend); modal citation: draft-03 or the pre-WG
`draft-polli-*` lineage.

### 3.4 Two systematic failure modes worth naming

`[COMPARATIVE]` **Names without semantics** (GitLab; Envoy in mirror image):
field names adopted, value encoding not — standards-aware clients are
actively misled, worse off than with an honestly proprietary header.

`[COMPARATIVE]` **Claims without names** (Supabase): the draft cited in
prose while `X-RateLimit-*` is emitted. `[INFERENCE]` Both failure modes are
invisible to anyone surveying by header name alone; any claim that vendor X
"supports the IETF RateLimit headers" must be resolved to a revision number
and a verbatim header line before it means anything.

### 3.5 What this implies for mandating `OP-010`

`[INFERENCE]` — this section only; ratification is the Gate C walkthrough's.

- **The trio and the structured fields are different standards, and the
  evidence points in opposite directions.** Every implementation citing the
  IETF work cites draft-06 or earlier — the superseded shape. A `MUST` on
  draft-11 is a `MUST` on a wire format with **one** documented vendor
  emitter (Cloudflare, uncited) and **one** opt-in library mode.
- **The de-facto convention is genuinely broken, so "follow the field" is
  not a safe fallback either.** Six-plus incompatible reset encodings under
  near-identical names is precisely the interoperability failure draft-11's
  problem statement describes. The draft solves a real problem, whatever its
  process trouble.
- **The gateway layer will not do this for you.** Six of twelve products
  emit nothing; none emits the structured fields; Traefik declined on the
  record. A structured-fields mandate is mandating something the deployment
  substrate cannot currently produce — implementers hand-roll it in
  application code.
- **The most-deployed IETF-capable implementation ships the de-facto
  convention on and the standard fields off** (express-rate-limit's
  defaults) — a concrete measure of the migration cost.
- ~~**The expiry contingency (re-check 2026-11-24) remains the right
  instrument**~~ *(overridden 2026-08-09 by `baseline-03f`, which had the
  draft's three expiry-revival cycles in view — the ratified triggers are
  IANA registration / wire-syntax re-pin / 18-month abandonment, reviewed
  semi-annually)* — **this survey still narrows what to watch for**: not
  another trio adopter — it is a second `RateLimit-Policy` emitter, or
  Cloudflare stating publicly which revision it tracks.

### 3.6 Known weaknesses in this survey

- Cloudflare's structured fields are documented, not wire-verified (the one
  unauthenticated probe returned 400 with no such headers).
- The Anthropic header-name list rests on one canonical page; OpenAI's two
  "sources" are two renderings of one document.
- Slack, Notion, Cohere, Groq, Polar, Vercel, and Together rows each rest on
  a single primary vendor page.
- The session's search budget was exhausted partway; later discovery used
  sitemaps and direct URL probing. Mistral's and xAI's rate-limit pages
  moved (old paths 404); xAI's page is client-rendered, so its absence claim
  rests on the rendered fetch alone.
- Kong `rate-limiting-advanced` runtime source and Zuplo's runtime are
  closed; those rows rest on vendor docs.
