# Lifecycle & Operations Across Eight REST API References (Part 6, Descriptive)

*Descriptive-only. Retrieval window: July 12–19, 2026. This report documents what each reference does today; it makes no prescriptions. Conflicts and tradeoffs are flagged for the later standard-setting phase. Scope is the seven-area surface below; reliability/idempotency (Part 5) and webhooks (Part 7) are out of scope.*

## TL;DR
- The field splits hardest on **versioning transport** (account-pinned dated vs. dated request header vs. path vs. query parameter vs. media-type) and on **rate-limit header families** (GitHub's `X-RateLimit-*`, Shopify's leaky-bucket `X-Shopify-Shop-Api-Call-Limit`, Stripe's `Stripe-Ratelimit-*`, Microsoft/Twilio's header-light `429`+backoff) — and no reference has adopted the IETF `RateLimit-*` fields, which remain an Internet-Draft.
- There is **near-consensus** that additive changes (new fields, new endpoints, new event types) are non-breaking and that consumers must tolerate them; the sharpest disagreement is whether **new enum values** are breaking (Stripe: usually breaking unless opted-in; Zalando: use `x-extensible-enum`; Google AIP: additive within a major version).
- **Auth surface** is bimodal: seven references use bearer/basic tokens with vendor key prefixes (`sk_live_`, `ghp_`, `shpat_`, Twilio `AC…`/`SK…`), while **AWS SigV4** is the lone outlier requiring per-request HMAC signing — buying tamper-evidence and presigned URLs at the cost of implementation complexity.

## Key Findings
1. **Three distinct dated-versioning models exist and are not interchangeable.** Stripe pins each *account* to a date and reads an optional `Stripe-Version` header override; GitHub reads a dated `X-GitHub-Api-Version` header (default `2022-11-28`); Twilio and Shopify put the date *in the path* (`/2010-04-01/`, `/admin/api/2026-07/`). These differ in who controls the default and in blast radius.
2. **Google AIP and Zalando are guideline documents, not live APIs, and they diverge from the payments/platform crowd.** Both prefer *compatible evolution* over versioning; Zalando explicitly forbids URL versioning (`MUST NOT use URI versioning`) and prescribes media-type versioning only when unavoidable. Google AIP mandates major-version-in-path (`/v1`, `/v1beta1`) with in-place minor updates.
3. **No reference has adopted the IETF `RateLimit-*` standard fields** (still an Internet-Draft, expires 24 Nov 2026). Each ships a proprietary family or no proactive quota headers at all.
4. **GitHub is the caching outlier**: conditional `304 Not Modified` responses do not count against its primary rate limit. Most other references issue weak or no cache validators for their JSON APIs.
5. **Breaking-change definitions are published in detail only by Stripe, GitHub, Google AIP, and Zalando.** Twilio, Shopify (REST), Microsoft Graph, and AWS document policy at coarser granularity.
6. **Sunset signaling is fragmented**: GitHub emits RFC-8594 `Sunset` + `Deprecation` headers on closing-down versions; Shopify emits a proprietary `X-Shopify-API-Deprecated-Reason`; Zalando's guideline recommends `Deprecation`/`Sunset`; Stripe, Twilio, Microsoft, and AWS rely on changelogs/docs and email rather than headers.
7. **OpenAPI/discovery publication is now standard for the platform vendors**: Stripe (`stripe/openapi`), GitHub (`github/rest-api-description`), Azure (`Azure/azure-rest-api-specs`), and Google (discovery docs + protos) all publish machine-readable specs; Twilio and Shopify REST publish docs but with less prominent first-party OpenAPI.

## Baseline Position (standards on this surface)

**RFC 9111 (HTTP Caching, STD 98, June 2022)** is the caching baseline; it defines how `ETag`/`Last-Modified` validators and `Cache-Control` govern stored responses. It is a full Internet Standard.

**Deprecation header** — `draft-ietf-httpapi-deprecation-header` (v09, 27 Sept 2024, "Expires: 31 March 2025"): defines the `Deprecation` HTTP response field signaling that a resource "will be or has been deprecated," plus a `deprecation` link relation. Still an **Internet-Draft**, not an RFC. The draft catalogs non-standard alternatives in the wild (Zapier's `X-API-Deprecation-Date`/`X-API-Deprecation-Info`; IBM's `Deprecated`; Ultipro's use of `Warning` code 299; Clearbit's custom header).

**Sunset header** — **RFC 8594 (May 2019, Informational)**: defines the `Sunset` response header carrying an HTTP-date "when the resource is expected to become unavailable." Explicitly distinguishes two deprecation stages: stage 1 (no longer recommended — `Sunset` *not* appropriate) and stage 2 (decommissioning — `Sunset` appropriate).

**RateLimit fields** — `draft-ietf-httpapi-ratelimit-headers` (v11, "Expires 24 November 2026"): defines `RateLimit` and `RateLimit-Policy` structured-field (RFC 9651) response headers so servers can advertise quota policy and remaining service limits. Still an **Internet-Draft**. It does not mandate `429` (that comes from RFC 6585) and says `RateLimit` fields SHOULD be ignored on responses served from cache (positive current_age). Earlier revisions (draft-polli / draft-ietf-05) defined discrete `RateLimit-Limit`/`RateLimit-Remaining`/`RateLimit-Reset`; the current revision consolidates into a structured `RateLimit` field.

**Guideline docs on this surface**: **Zalando** (`opensource.zalando.com/restful-api-guidelines`) treats compatibility as a first-class chapter — `MUST NOT use URI versioning`, maintain backward compatibility, use `x-extensible-enum`, `MUST NOT declare additionalProperties: false`. **Google AIP** (aip.dev): AIP-180 (backwards compatibility), AIP-181 (stability levels), AIP-185 (versioning), AIP-3 (AIPs themselves date-versioned). Both are prescriptive standards *for their own ecosystems*; here they function as reference positions to be checked against the field.

## Side-by-Side Comparison Tables

### Table 1 — Versioning scheme
| Reference | Transport | Format/example | Default when unspecified | Support window |
|---|---|---|---|---|
| Stripe | Account-pinned dated + `Stripe-Version` header override; `/v1/` path prefix is static | `2026-06-24.dahlia` (current); major names Acacia/Basil/Clover/Dahlia | Account's pinned version (set on first request) | Every version since 2011 still honored (per Stripe's versioning essay, "almost a hundred backwards-incompatible upgrades over the past six years" as of 2017) |
| GitHub REST | Dated request header `X-GitHub-Api-Version` | `2022-11-28`; `curl --header "X-GitHub-Api-Version:2026-03-10"` | `2022-11-28` (unversioned requests default to it) | ≥24 months after a newer version ships |
| Google/AIP | Major version in URI path + protobuf package | `/v1`, `/v1beta1`, `/v1alpha5`; channel vs release strategy | n/a (path-explicit) | "reasonable, well-communicated deprecation period" |
| Microsoft Graph | Path segment | `/v1.0/`, `/beta/` | n/a (path-explicit) | Deprecated ≥24 months before retirement |
| Azure (data-plane) | `api-version` query parameter (required) | `?api-version=1.0` (SemVer) or dated; missing → `400` | none — request rejected `400` | Per-service |
| Twilio | Date in path | `/2010-04-01/` (stable since 2010) | Account default API version | Single long-lived version; new features added in place |
| Shopify Admin REST | Date in path (quarterly) | `/admin/api/2026-07/`; header `X-Shopify-API-Version` echoes | Falls *forward* to oldest accessible version | ≥12 months per version, ≥9 months overlap |
| AWS | `Version=` query/body parameter (Query APIs); path prefixes on some REST services | `Version=2012-11-05` style; SigV4 covers it in signature | n/a (per-service) | Per-service |
| Zalando (guideline) | `MUST NOT use URI versioning`; media-type versioning only if unavoidable | `application/x.zalando.cart+json;version=2` | Prefer compatible evolution | n/a |

### Table 2 — Breaking-change definition
| Reference | Additive (non-breaking) | Breaking | New enum value? |
|---|---|---|---|
| Stripe | New resources; new optional params; new response properties; reordering; changing opaque-string length/format incl. ID prefixes; new event types | Removing/renaming fields; type changes; changing defaults | **Usually breaking** unless user opts in (e.g., new payment-method type) or field is clearly dynamic (banks, currencies) |
| GitHub | Additive changes available in all supported versions | Released only in a new dated version | Additive if handled per docs |
| Google AIP-180 | New interfaces/methods/messages/fields/enum values may be added within a major version | Anything violating source/wire/semantic compatibility; changing resource-name formats (breaking even across majors) | **Additive** within major version |
| Zalando | Add optional fields; use `x-extensible-enum` | Incompatible changes deployed to live consumers | Use `x-extensible-enum`, not closed `enum` |
| Microsoft Graph | Adding JSON fields is *not* breaking | Increment major version | Additive |
| Shopify | New fields/resources | Removed/changed fields → new quarterly version | New quarterly version |
| Twilio | New sub-resources, new fields | Marked "(breaking change)" in helper-lib changelogs | Handled in SDK majors |
| AWS | New params/fields | New `Version=` value | Per-service |

### Table 3 — Rate limiting
| Reference | Headers | Status | Retry signal | Quota model |
|---|---|---|---|---|
| GitHub | `X-RateLimit-Limit/Remaining/Reset/Used/Resource` (lowercase over HTTP/2) | `403` or `429` | `retry-after` (secondary); `x-ratelimit-reset` (primary) | 5,000/hr authenticated user; 60/hr unauth; 15,000/hr GHEC app; separate pools (core/search/graphql); secondary: 100 concurrent, 900 pts/min REST |
| Shopify REST | `X-Shopify-Shop-Api-Call-Limit: 32/40` | `429` | `Retry-After` (seconds) | Leaky bucket: 40 bucket / 2-per-sec leak (400/20 Plus); per app+store |
| Microsoft Graph | `Retry-After`; `x-ms-throttle-information` (reason); no `X-RateLimit-*` | `429` | `Retry-After` (seconds; omitted on some resources → backoff) | Token-bucket; per-app-per-tenant (scaled by tenant size S/M/L), per-user, per-resource, service-level fairness |
| Stripe | `Stripe-Ratelimit-Limit/Remaining/Reset`; `Stripe-Rate-Limited-Reason` | `429` | none standard; client exponential backoff w/ jitter | 100 read + 100 write req/sec live mode; 25/sec test mode; global + per-endpoint; separate concurrency limit (`lock_timeout`) |
| Twilio | `Twilio-Concurrent-Requests`; no `X-RateLimit-*`, no `Retry-After` | `429` (error `20429`) | client-side exponential backoff; "always safe to retry" | Account REST concurrency; Verify/Messaging sub-limits |
| Google/AIP | Per-project/per-method quotas; documented per-service | `429`/`403` | n/a | Per-project/per-method quotas |
| AWS | Per-service (some emit `Retry-After`) | `429` / `400 ThrottlingException` | Exponential backoff (SDK default) | Token-bucket per-service |
| Zalando (guideline) | Recommends signaling; no mandated family | `429`/`503` | `Retry-After` | n/a |

### Table 4 — Caching & conditional reads
| Reference | ETag/Last-Modified | 304 behavior | Notes |
|---|---|---|---|
| GitHub | Most responses return `ETag`; many return `Last-Modified` | **`304` does not count against primary rate limit** (when correctly authorized) | ETags per-page, per-token; expire with token; not on GraphQL |
| Zalando (guideline) | Recommends ETag/conditional for cacheable master data | Standard 304 | Recommends caching for rarely-changing data; supports gzip |
| Google/Microsoft/Shopify/Stripe/Twilio/AWS | Generally not emphasized for JSON API reads; ETags used for specific resources (e.g., S3 objects) | n/a | Payments/transactional APIs largely avoid caching |

### Table 5 — Auth surface
| Reference | Scheme | Key format(s) | Rotation | 401 vs 403 |
|---|---|---|---|---|
| Stripe | Basic auth (secret key as username) / bearer | `sk_live_`, `sk_test_`, `pk_live_`, `pk_test_`, `rk_live_`/`rk_test_` (restricted), `sk_org` (org), `whsec_` (webhook secret, not a key) | Dashboard create/reveal/delete/rotate; live secret shown once | invalid key → invalid request error; deleted/expired → authentication error |
| GitHub | Bearer token | `ghp_` (PAT), `gho_` (OAuth), `ghu_` (user-to-server), `ghs_` (server-to-server), `ghr_` (refresh), `github_pat_` (fine-grained, 93 chars); last 6 chars = CRC32/Base62 checksum | User/app managed; expiry configurable | 401 unauthorized; 403 rate/permission |
| Shopify | `X-Shopify-Access-Token` header | `shpat_` (offline), `shpua_` (online/per-user), `shppa_` (legacy private), `shpca_` (custom), `shpss_` (secret) | App-managed | 401 = missing/incorrect creds; 403 = missing scope; 423/430 for abuse |
| Microsoft Graph | Bearer (OAuth token) | Azure AD access token | AAD-managed | 401/403 |
| Google | Bearer (OAuth) or `x-goog-api-key` | API key or OAuth token | Google-managed | 401/403 |
| AWS | **SigV4 request signing** | Access key ID + secret; `Authorization: AWS4-HMAC-SHA256 Credential=…, SignedHeaders=…, Signature=…` or `X-Amz-*` query params | IAM key rotation | 403 `SignatureDoesNotMatch`/`Unauthorized` |
| Twilio | HTTP Basic | Account SID `^AC[0-9a-fA-F]{32}$` (34 chars); API Key SID `^SK[0-9a-fA-F]{32}$` (34 chars) + secret | `/2010-04-01/Accounts/{Sid}/Keys.json`; Auth Token rotation; secret shown once | 401 (standard); 403 for stale token in Functions |
| Zalando (guideline) | OAuth 2.0 recommended | n/a | n/a | Problem+JSON errors |

### Table 6 — Spec & docs practice
| Reference | Official machine-readable spec | Changelog | Notes |
|---|---|---|---|
| Stripe | `stripe/openapi` (OpenAPI 3.0: `spec3.json/yaml`, plus `spec3.sdk.*`, `fixtures3.*`); OpenAPI 2.0 deprecated | Narrative API changelog by dated version | Vendor extensions `x-expandableFields`, `x-polymorphicResources`, `x-resourceId` |
| GitHub | `github/rest-api-description` (3.0 in `descriptions/`, 3.1 in `descriptions-next/`); stable since 1.1.4 | GitHub Changelog + blog + email | No PRs to spec accepted; auto-generated from validation source |
| Azure | `Azure/azure-rest-api-specs` (TypeSpec/OpenAPI) | Per-service | Drives SDK generation; GA→stable SDK, preview→beta SDK |
| Google | Discovery docs + protobuf sources | AIP changelogs; per-API release notes | AIPs date-versioned (AIP-3, e.g., `v2023-03-28`) |
| Microsoft Graph | Metadata (`$metadata` CSDL); docs on Learn | "What's new" | v1.0 + beta |
| Twilio | Docs site; helper-lib repos with changelogs | Product changelog | First-party OpenAPI less prominent |
| Shopify | Docs site; reference updated first business day after quarterly release | Developer changelog; release notes per version | REST Admin "legacy as of Oct 1, 2024" |
| Zalando (guideline) | Mandates OpenAPI 3.0 single self-contained YAML; publish to `/zalando-apis` dir | Guideline PRs | `x-api-id`, `x-audience` required |

## Per-Reference Notes (lifecycle character sketch)

**Stripe** — The archetype of *account-pinned dated versioning*. The first request pins the account to the then-current version (currently `2026-06-24.dahlia`); every later call without a header uses that version. Per-request override via `Stripe-Version` header; account upgrade via Workbench with a documented safety net: "For 72 hours after you've upgraded your API version, you can safely roll back to the version you were upgrading from in Workbench. After you've rolled back, webhooks that were sent with the new object structure and failed will be retried with the old structure." Since 2024-09-30.acacia, cadence is monthly backward-compatible releases plus twice-yearly named majors with breaking changes. Famous, explicit backward-compatibility list (see Table 2 and Appendix). Rate limiting is `429`-centric with `Stripe-Ratelimit-*` and `Stripe-Rate-Limited-Reason` headers; documented default 100 read + 100 write requests/sec live mode (25/sec test mode); guidance is exponential backoff with jitter plus a client-side token bucket. Auth is Basic (secret key as username) with strongly prefixed keys and clean test/live separation. First-party OpenAPI at `stripe/openapi`. Caching not emphasized. Confidence: **High** (primary docs + engineering blog).

**GitHub REST** — *Dated request-header versioning* (`X-GitHub-Api-Version`), calendar-based, default `2022-11-28`, ≥24-month support, unsupported version → `400`. Per GitHub's March 2026 changelog announcing `2026-03-10` (its first calendar version to include breaking changes): "Version 2022-11-28 will continue to be fully supported for at least 24 months from today, and requests that don't include the X-GitHub-Api-Version header will continue to default to 2022-11-28." Emits RFC-7231 `Deprecation` and RFC-8594 `Sunset` headers on closing-down versions; retired version → `410 Gone`; unversioned requests default to next-oldest. The rate-limit exemplar: `X-RateLimit-*` family with separate pools (`core`/`search`/`graphql`), primary 5,000/hr (60/hr unauth, 15,000/hr GHEC apps), plus documented secondary limits (100 concurrent, 900 REST points/min; GET/HEAD/OPTIONS = 1 point, POST/PATCH/PUT/DELETE = 5). Notable caching: conditional `304`s don't count against the primary limit. Token prefixes (`ghp_` etc.) with CRC32 checksum. OpenAPI at `github/rest-api-description`. Confidence: **High**.

**Google Cloud / AIP** — Guideline ecosystem. AIP-185: major version in path/package (`/v1`, never `/v1.0`); alpha/beta appended (`v1beta1`); channel-based vs release-based strategies; the single stable channel updated in place. AIP-181: stable components fully supported for the major version's life; an in-place breaking change is "an extreme course of action" requiring API Governance team approval. AIP-180: source/wire/semantic compatibility; additive within a major version OK; resource-name format changes are breaking even across majors. API visibility labels (`X-Goog-Visibilities: PREVIEW`) offered as an alternative to versioning for incremental changes. AIPs themselves date-versioned (AIP-3). Confidence: **High** (primary aip.dev).

**Microsoft (Azure + Graph)** — Two patterns. **Graph**: path segments `/v1.0/` and `/beta/`; beta may break without notice and is unsupported for production; deprecation ≥24 months before retirement; adding JSON fields is explicitly non-breaking. Throttling is token-bucket, `429` + `Retry-After`, scoped per-app-per-tenant (scaled by tenant size S<50 / M 50–500 / L>500 users), per-user, per-resource, plus service-level fairness; `x-ms-throttle-information` gives the reason; some resources omit `Retry-After` (→ exponential backoff). **Azure data-plane**: the `api-version` query parameter is *required*; missing → `400` with `application/problem+json` (`"title":"API version is not specified"`). Specs at `Azure/azure-rest-api-specs`. Confidence: **High**.

**Twilio** — *Date-in-path* versioning frozen at `/2010-04-01/` since 2010; new capability is added in place rather than via new versions; a per-resource `ApiVersion` field can pin TwiML behavior (`2010-04-01` or `2008-08-01`). The base API returns XML by default (JSON/CSV optional); product APIs (`<product>.twilio.com/vN`) are JSON-only. Rate limiting: `429` with error code `20429` ("Twilio returns this error when your account exceeds allowed concurrency to Twilio's REST API"); documents a `Twilio-Concurrent-Requests` response header ("The current number of concurrent requests for your account… This count includes requests that receive a 429 Too Many Requests") rather than `X-RateLimit-*` or `Retry-After`; guidance is client-side exponential backoff, and "Any requests with Error 429 responses are always safe to retry." Verify/Messaging have their own sub-limits. Auth: HTTP Basic — recommended API Key SID (`SK…`, 34 chars) + secret, or Account SID (`AC…`, 34 chars) + Auth Token for local testing; keys created/rotated via `/2010-04-01/Accounts/{Sid}/Keys.json` (secret shown once); delete → `204` and invalidates tokens. Confidence: **High** for versioning/auth; the absence of `Retry-After` is a confirmed negative finding.

**Shopify Admin REST** — *Date-in-path quarterly* versioning (`/admin/api/2026-07/`); a new version each quarter at 5pm UTC; ≥12-month support with ≥9-month overlap; an inaccessible target *falls forward* to the oldest accessible version ("requests to a retired 2026-10 are served as 2027-01"); `X-Shopify-API-Version` echoes the version served. Deprecations flagged via the proprietary `X-Shopify-API-Deprecated-Reason` header (value is a URL for REST; changed to a comma-separated field list for the GraphQL Admin API as of 2025-04); GET requests don't return the header for property-level deprecations. Rate limiting: leaky bucket `X-Shopify-Shop-Api-Call-Limit: 32/40` + `429` + `Retry-After`; standard 40 bucket / 2-per-sec leak (400/20 for Plus); per app+store; an extra offset throttle beyond 100,000 results and a variant throttle at 50,000 variants. Auth via `X-Shopify-Access-Token` header, `shpat_`/`shpua_`/`shppa_` prefixes; 401 = missing/incorrect creds, 403 = missing scope, plus 423/430 for abuse. **Currency/status caveat**: Shopify's REST Admin API is officially "a legacy API as of October 1, 2024" — all new integrations are directed to the GraphQL Admin API — so its REST lifecycle mechanics are maintained but effectively frozen-legacy. Confidence: **High**.

**Zalando RESTful API Guidelines** — A guideline document, not a live API. Verified stance: `MUST NOT use URI versioning` (paths must not contain versions); prefer compatible evolution; when versioning is unavoidable, use media-type versioning (`application/x.zalando.cart+json;version=2`) via content negotiation. Preferred extension order: new resource variant → new endpoint/API (new domain name) → parallel new version. Compatibility: treat OpenAPI as open for extension (`MUST NOT declare additionalProperties: false`); use `x-extensible-enum` instead of closed `enum`. The deprecation chapter recommends `Deprecation`/`Sunset` headers and that new clients must not adopt deprecated APIs. Mandates OpenAPI 3.0 publication (single self-contained YAML copied to `/zalando-apis`), `x-api-id` (immutable UUID), `x-audience`, SemVer `info.version`. Confidence: **High**.

**AWS** — The auth-surface outlier: **SigV4** per-request HMAC-SHA256 signing over a canonical request (method, path, query, headers, payload hash), expressed either in the `Authorization: AWS4-HMAC-SHA256 Credential=…/YYYYMMDD/region/service/aws4_request, SignedHeaders=…, Signature=…` header or as `X-Amz-*` query parameters (presigned URLs). Requests must reach AWS within ~5 minutes (S3 signed portions valid 15 minutes). The SigV4a variant uses ECDSA (`AWS4-ECDSA-P256-SHA256` + `X-Amz-Region-Set`) for multi-region. **Cost/benefit**: buys tamper-evidence, credential-non-transmission, and presigned-URL delegation; costs implementation complexity (SDKs strongly recommended over hand-rolling). Versioning is per-service via a `Version=` parameter (e.g., Query APIs) rather than a global path/header scheme. There is no single unified AWS spec; API references are service-level. Confidence: **High** for SigV4/auth; **Medium** on `Version=` generality (varies by service).

## Agreements vs. Divergences

**Agreements**
- **Additive changes are non-breaking** — universal (new fields, new endpoints, new event types). Consumers must ignore unknown fields.
- **`429` is the throttle status** — all references that throttle use `429 Too Many Requests` (GitHub also uses `403`).
- **Dated versioning is popular** — Stripe, GitHub, Twilio, Shopify all use ISO-date version identifiers, though via different transports.
- **Key prefixing for secret-scanning** — Stripe, GitHub, Shopify all use typed prefixes; this is a genuine cross-field convention (GitHub's blog explicitly cites Slack and Stripe as precedent).
- **Multi-month deprecation windows** — GitHub (24mo), Microsoft (24mo), Shopify (12mo/9mo overlap) converge on generous windows.

**Divergences (with tradeoffs)**
- **Versioning transport** — Account-pinned dated (Stripe) minimizes consumer action but requires server-side transform machinery; header-dated (GitHub) is explicit and cache-neutral but easy to forget (defaults to oldest); path-dated (Twilio/Shopify) is visible and cache-friendly but pollutes URLs and couples version to resource identity; query-param (Azure/AWS) is simple but easy to omit (Azure rejects with `400`); media-type (Zalando) enables content negotiation but complicates tooling. Tradeoff axis: *consumer effort vs. server complexity vs. URL cleanliness*.
- **New enum values** — Stripe treats them as usually-breaking (protecting strict clients) vs. Zalando/Google designing for open enums (favoring evolution speed). Links to Part 3's open-enum question.
- **Sunset signaling** — Header-based (GitHub, Zalando-recommended) enables automated client reaction; docs/email-based (Stripe, Twilio, Microsoft, AWS) is human-readable but not machine-actionable. Shopify's proprietary header is machine-readable but non-standard.
- **Rate-limit transparency** — GitHub/Shopify/Stripe publish remaining-quota headers (proactive client shaping) vs. Microsoft/Twilio publishing only reactive `429`+backoff guidance (Microsoft adds `Retry-After`; Twilio adds a concurrency-count header). Tradeoff: *observability vs. server flexibility to change limits dynamically*.
- **Caching** — GitHub invests in conditional requests (and rewards them by not counting 304s); payments/transactional APIs largely disable caching. Tradeoff: *read-scaling vs. freshness guarantees for money-movement*.
- **Signed requests vs. bearer** — AWS SigV4 buys tamper-evidence and presigning at the cost of complexity; everyone else's bearer/basic tokens are simpler but rely on TLS + secret secrecy.

## Contested Axes Register (scoped to Part 6)

| Axis | Options observed | Who does what | Tradeoff (one line) | How contested |
|---|---|---|---|---|
| Versioning transport | Account-pinned dated; dated request header; path (dated or major); query param; media-type | Stripe (account+header); GitHub (header); Twilio/Shopify/Google/MS-Graph (path); Azure/AWS (query); Zalando (media-type, discouraged) | Consumer effort vs. server complexity vs. URL cleanliness | **Wide-open** |
| Breaking-change strictness | Detailed public list; policy-level; guideline rules | Stripe/GitHub/AIP/Zalando (detailed); Twilio/Shopify/MS/AWS (coarser) | Precision aids clients but constrains the provider | **Split** |
| New enum values breaking? | Breaking-unless-opted-in; additive within major; use extensible-enum | Stripe (usually breaking); Google AIP (additive); Zalando (`x-extensible-enum`) | Strict-client safety vs. evolution speed | **Split** |
| Sunset signaling | RFC-8594 `Sunset`+`Deprecation`; proprietary header; docs/email only | GitHub (RFC headers); Shopify (`X-Shopify-API-Deprecated-Reason`); Zalando (recommends RFC); Stripe/Twilio/MS/AWS (docs/email) | Machine-actionable vs. human-readable | **Split** |
| Rate-limit header family | `X-RateLimit-*`; `X-Shopify-Shop-Api-Call-Limit`; `Stripe-Ratelimit-*`; `Twilio-Concurrent-Requests`; none (`Retry-After` only); IETF `RateLimit-*` | GitHub; Shopify; Stripe; Twilio; MS-Graph/AWS; (nobody on IETF) | Proactive shaping vs. dynamic-limit flexibility | **Wide-open** |
| Quota model | Fixed req/hr; leaky bucket; token bucket; points/cost; per-project | GitHub (fixed/hr + points); Shopify (leaky bucket); Stripe/MS/AWS (token bucket); Google (project quota) | Burst tolerance vs. predictability | **Split** |
| Conditional-request support | 304s exempt from limit; standard 304; none/minimal | GitHub (exempt); Zalando (recommends); most others (minimal) | Read-scaling vs. freshness for transactional data | **Near-consensus against** (only GitHub/Zalando lean in) |
| Key-prefix convention | Typed prefixes; SID patterns; access keys | Stripe (`sk_`/`pk_`/`rk_`); GitHub (`gh*_`); Shopify (`shp*_`); Twilio (`AC`/`SK`); AWS (`AKIA…`) | Scannability/identifiability vs. opacity | **Near-consensus (yes)** |
| 401 vs 403 line | 401 auth-missing/bad, 403 scope/permission; 403 also for rate/signature | Shopify (explicit 401/403 split); GitHub (403 for rate+perm); AWS (403 signature); Stripe (error types) | Consistent semantics vs. security-through-ambiguity | **Split** |
| Signed-requests vs. bearer | SigV4 per-request signing; bearer/basic token | AWS (SigV4); all others (bearer/basic) | Tamper-evidence/presigning vs. simplicity | **Near-consensus (bearer), AWS lone outlier** |

## Examples Appendix (verbatim artifacts by reference)

**Stripe**
- Current version: `2026-06-24.dahlia`; major names: Acacia, Basil, Clover, Dahlia. Monthly example: `2024-09-30.acacia` → `2024-10-31.acacia` (safe upgrade).
- Per-request override: `Stripe-Version: <date>` header; organization API keys MUST include `Stripe-Version`.
- Backward-compatible list (verbatim categories): "Adding new API resources. Adding new optional request parameters to existing API methods. Adding new properties to existing API responses. Changing the order of properties in existing API responses. Changing the length or format of opaque strings, such as object IDs, error messages… This includes adding or removing fixed prefixes (such as `ch_` on charge IDs)… store IDs in a `VARCHAR(255) COLLATE utf8_bin` column… Adding new event types."
- Enum rule (verbatim): backwards-compatible to return a new enum value only if "the user opts into it with your integration, like a new payment method type" or "the enum field is clearly not static like a list of banks or currencies."
- Historical breaking changes: `2014-09-08` (bank account status enum), `2015-10-01` (`bank_accounts`→`external_accounts`), `2016-02-19` (`name`→`account_holder_name`), `2017-05-25` (`managed` bool→`type` enum with values `standard`/`express`/`custom`).
- Upgrade rollback (verbatim): "For 72 hours after you've upgraded your API version, you can safely roll back to the version you were upgrading from in Workbench."
- Rate-limit headers: `Stripe-Ratelimit-Limit: 100` / `Stripe-Ratelimit-Remaining: 97` / `Stripe-Ratelimit-Reset: 1716998460`; `Stripe-Rate-Limited-Reason` on 429. Limits: 100 read + 100 write/sec live mode; 25/sec test mode. Lock contention → `429` with `lock_timeout`.
- Keys: `sk_live_`, `sk_test_`, `pk_live_`, `pk_test_`, `rk_live_`, `rk_test_`, `sk_org`, `whsec_` (webhook secret). Auth = Basic (secret key as username, empty password).
- OpenAPI: `stripe/openapi` → `openapi/spec3.json`, `spec3.yaml`, `spec3.sdk.*`, `fixtures3.*`; extensions `x-expandableFields`, `x-polymorphicResources`, `x-resourceId`.

**GitHub REST**
- `curl --header "X-GitHub-Api-Version:2022-11-28" https://api.github.com/zen`; default `2022-11-28`; unsupported → `400`.
- Deprecation/Sunset (verbatim): "Deprecation — The date when the API version will be closing down, formatted as an HTTP date per RFC 7231." / "Sunset — … Follows RFC 8594. For example: `Fri, 27 Nov 2020 14:34:29 GMT`." Retired → `410 Gone`; unversioned requests "default to the next oldest supported version."
- Rate-limit headers: `X-RateLimit-Limit: 5000` / `X-RateLimit-Remaining: 4987` / `X-RateLimit-Reset: 1706140800` / `X-RateLimit-Used: 13` / `X-RateLimit-Resource: core`.
- `/rate_limit` JSON: `"core": {"limit":5000,"used":1,"remaining":4999,"reset":1691591363}`, `"search":{"limit":30,...}`, `"graphql":{"limit":5000,...}`, `"code_search":{"limit":10,...}`, `"scim":{"limit":15000,...}`.
- Secondary limits (verbatim): "No more than 100 concurrent requests… No more than 900 points per minute are allowed for REST API endpoints." Point costs: GET/HEAD/OPTIONS = 1; POST/PATCH/PUT/DELETE = 5.
- Conditional (verbatim): "Making a conditional request does not count against your primary rate limit if a 304 response is returned and the request was made while correctly authorized with an Authorization header." Example: `curl -H "Authorization: Bearer YOUR-TOKEN" https://api.github.com/meta --include --header 'if-none-match: "644b5b0155e6404a9cc4bd9d8b1ae730"'` → `HTTP/2 304`.
- Token prefixes: `ghp_`, `gho_`, `ghu_`, `ghs_`, `ghr_`, `github_pat_` (fine-grained, 93 chars); last 6 chars = CRC32/Base62 checksum; `_` separator (underscore chosen for double-click selectability).
- OpenAPI: `github/rest-api-description` (`descriptions/` = 3.0, `descriptions-next/` = 3.1; "we don't currently accept pull requests that directly modify the description").

**Google / AIP**
- "Google APIs use `v1`, not `v1.0`, `v1.1`, or `v1.4.2`." Alpha/beta: `v1beta1`, `v1alpha5`.
- Visibility label: `GET /v1/projects/my-project/topics HTTP/1.1` … `X-Goog-Visibilities: PREVIEW`.
- Gemini example: `curl -X POST "https://generativelanguage.googleapis.com/v1/interactions" -H "x-goog-api-key: $GEMINI_API_KEY"`.
- AIP-180 (verbatim): "Existing client code must not be broken by a service updating to a new minor or patch release." AIP-181: in-place stable breaking change "requires the approval of the API Governance team."
- AIP versioning (AIP-3): date tags `v2023-03-28`.

**Microsoft Graph / Azure**
- Graph: `https://graph.microsoft.com/v1.0/{resource}` and `/beta/{resource}`.
- Throttling response (verbatim): `HTTP/1.1 429 Too Many Requests` / `Retry-After: 10` / body `{"error":{"code":"TooManyRequests","innerError":{"code":"429",...},"message":"Please retry again later."}}`.
- Teams sub-limit (verbatim): "A maximum of one request per second per app per tenant can be issued on a given channel or chat." Tenant sizes: "S - under 50 users, M - between 50 and 500 users, and L - above 500 users."
- Azure data-plane: `https://{myconfig}.azconfig.io/kv?api-version=1.0`; missing → `400` with `application/problem+json` (`"title":"API version is not specified"`).
- Specs: `Azure/azure-rest-api-specs`.

**Twilio**
- Base URL `https://api.twilio.com/2010-04-01`; stable since 2010.
- Response `ApiVersion`: `<ApiVersion>2010-04-01</ApiVersion>` (XML default) / `"api_version": "2010-04-01"` (JSON).
- Auth: `curl -G https://api.twilio.com/2010-04-01/Accounts -u $TWILIO_API_KEY:$TWILIO_API_KEY_SECRET`. Account SID `^AC[0-9a-fA-F]{32}$` (34 chars); API Key SID `^SK[0-9a-fA-F]{32}$` (34 chars).
- Rate limit: HTTP `429`, error `20429` ("Twilio returns this error when your account exceeds allowed concurrency to Twilio's REST API"); header `Twilio-Concurrent-Requests` ("This count includes requests that receive a 429 Too Many Requests"); no `Retry-After`; guidance = client-side exponential backoff; "Any requests with Error 429 responses are always safe to retry."
- Key mgmt: `POST https://api.twilio.com/2010-04-01/Accounts/{AccountSid}/Keys.json`; delete → `204`, invalidates access tokens; "Store the `secret`… you won't be able to retrieve it again."

**Shopify Admin REST**
- URL: `/admin/api/2026-07/products.json`; `X-Shopify-API-Version` echoes served version.
- Support (verbatim): "Each stable version is supported for a minimum of 12 months, with at least nine months of overlap between consecutive versions." Falls forward: "requests to a retired 2026-10 are served as 2027-01."
- Rate-limit + deprecation header block: `X-Shopify-Shop-Api-Call-Limit → 1/40`, `X-Shopify-API-Version → 2026-04`, `X-Shopify-API-Deprecated-Reason → https://shopify.dev/api/release-notes/2023-01#metafields-rest`.
- Leaky bucket: bucket 40 / leak 2 per sec (Plus: 400 / 20); "if the header displays 39/40 requests, then after a wait period of ten seconds, the header displays 19/40 requests." Over → `429 Too Many Requests` + `Retry-After`.
- Offset throttle (verbatim): "a request to `GET /admin/collects.json?limit=250&page=401` would generate an offset of 100,250 … and return a 429 response."
- Auth: `X-Shopify-Access-Token` header (verbatim example: `-H "X-Shopify-Access-Token: ${SHOP_TOKEN}"`); prefixes `shpat_` (offline), `shpua_` (online), `shppa_` (legacy private), `shpca_` (custom). Status (verbatim): "401 Unauthorized — The necessary authentication credentials are not present in the request or are incorrect." / "403 Forbidden — … This status is generally returned if you haven't requested the appropriate scope for this action." 401 body: `{"errors":"[API] Invalid API key or access token (unrecognized login or wrong password)"}`; 429 body: `{"errors":"Exceeded 2 calls per second for api client."}`.
- Deprecation header value: URL for REST; GraphQL 2025-04+ returns a field list, e.g. `X-Shopify-API-Deprecated-Reason: Shop.products, Shop.productVariants`.
- **Currency/status caveat**: REST Admin API is "a legacy API as of October 1, 2024."

**AWS**
- SigV4 header: `Authorization: AWS4-HMAC-SHA256 Credential=AKIAIOSFODNN7EXAMPLE/20130524/us-east-1/s3/aws4_request, SignedHeaders=host;range;x-amz-date, Signature=<sig>`.
- Presigned query form: `?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=<key>/20130721/us-east-1/s3/aws4_request&X-Amz-Date=20130721T201207Z&X-Amz-Expires=86400&X-Amz-SignedHeaders=host&X-Amz-Signature=<sig>`.
- Canonical request (verbatim): `HTTPRequestMethod + '\n' + CanonicalURI + '\n' + CanonicalQueryString + '\n' + CanonicalHeaders + '\n' + SignedHeaders + '\n' + HexEncode(Hash(RequestPayload))`.
- Timing: "a request must reach AWS within five minutes"; S3 signed portions valid 15 minutes. SigV4a: `X-Amz-Region-Set`, `AWS4-ECDSA-P256-SHA256`.
- Version param example (S3 subresource): `?versioning`.

**Zalando (guideline)**
- `MUST NOT use URI versioning` — "Path must not contain versions."
- Media-type versioning: `application/x.zalando.cart+json;version=2`.
- `x-extensible-enum` mandated over `enum`; `MUST NOT declare additionalProperties: false`.
- Meta: `x-api-id` pattern `^[a-z0-9][a-z0-9-:.]{6,62}[a-z0-9]$` (example `d0184f38-b98d-11e7-9c56-68f728c1ba70`); `x-audience` enum (`component-internal`, `business-unit-internal`, `company-internal`, `external-partner`, `external-public`); SemVer `info.version` (e.g., `1.1.0`).
- Publication: OpenAPI copied to the `/zalando-apis` directory of the deployment artifact (preferred over the legacy `.well-known/schema-discovery` endpoint).

**IETF baseline**
- `draft-ietf-httpapi-ratelimit-headers-11` (Expires 24 Nov 2026): `RateLimit` + `RateLimit-Policy` structured fields; SHOULD be ignored on cached responses.
- `draft-ietf-httpapi-deprecation-header-09` (Expires 31 Mar 2025): `Deprecation` field + `deprecation` link relation.
- RFC 8594 (May 2019): `Sunset: Fri, 27 Nov 2020 14:34:29 GMT`.
- RFC 9111 (STD 98, June 2022): HTTP caching.

## Caveats
- **Numbers drift.** Rate limits, support windows, and "current version" strings are dated to July 2026; Stripe's `2026-06-24.dahlia`, Shopify's `2026-07`, and GitHub's example `2026-03-10` will change. Re-verify limit numbers before relying on them.
- **Guideline docs ≠ live behavior.** Zalando and Google AIP describe *what an API should do*; individual Google/Zalando services may deviate. Microsoft Graph follows the Microsoft REST guidelines, but Azure data-plane services vary service-by-service.
- **Secondary sources flagged.** Stripe's `Stripe-Ratelimit-*` header names are attested by community/tooling sources corroborating primary docs; the 100/sec live and 25/sec test figures are now both confirmed by Stripe Support and docs. Twilio's *absence* of `Retry-After` is a confirmed negative finding (Twilio documents `Twilio-Concurrent-Requests` and backoff instead).
- **Shopify token prefixes** originate primarily from Shopify's April 2020 API changelog and staff confirmation in Community, not a clean shopify.dev table; the `X-Shopify-Access-Token` header itself is verbatim-confirmed on shopify.dev.
- **AWS heterogeneity.** "AWS" is not one API; SigV4 is universal, but `Version=` parameter usage, throttling headers, and error shapes vary per service. Statements here generalize from IAM/S3/Query-API references.
- **Out of scope** (per series): OAuth/OIDC protocol flows (the auth *surface* is in scope, the flows are not), GraphQL/gRPC, gateways/infra, SDK internals, event streaming/webhooks (Part 7), reliability/idempotency (Part 5).