# REST API Conventions Series — Part 6/7: Lifecycle & Operations (Versioning, Rate Limits, Caching, Auth Surface & Docs Practice)

*Descriptive survey of eight references. All version strings, rate-limit numbers and support windows retrieved 2026-07-19 and are time-sensitive. Descriptive only — no prescriptions; conflicts are flagged for the later prescriptive phase.*

## TL;DR
- The field splits hardest on **where the version identifier lives**: Stripe and GitHub use dated identifiers in a header (account-pinned for Stripe, per-request for GitHub); Google and Microsoft Graph put a coarse major/channel token in the path (`/v1`, `/v1beta`; `/v1.0`, `/beta`); Azure REST and AWS Query put a dated value in a query/request parameter; Twilio and Shopify use dated path segments; Zalando's guideline forbids URL versioning entirely and mandates compatible in-place evolution.
- On **rate-limit signaling** there is no consensus: GitHub uses `X-RateLimit-*`, Shopify a leaky-bucket `X-Shopify-Shop-Api-Call-Limit`, Microsoft Graph and Twilio are 429+`Retry-After`-centric, Stripe exposes minimal headers and is 429-centric. **None** of the eight has adopted the IETF `RateLimit`/`RateLimit-Policy` standardized fields (draft-11, 23 May 2026).
- On **auth surface** the outlier is AWS SigV4 request signing (secret never on the wire, per-request signature proving identity + integrity + timeliness); everyone else uses bearer-style secrets, and typed key prefixes are now the near-consensus (Stripe `sk_live_`, GitHub `ghp_`, Shopify `shpat_`), with Twilio still defaulting to Account SID + Auth Token HTTP Basic.

## Key findings
1. **Dated versioning has won at the payments/developer-platform tier, but the transport differs.** Stripe (`2026-06-24.dahlia`), GitHub (`2022-11-28`), Twilio (`2010-04-01`), Shopify (`2026-01`) and Azure (`api-version=YYYY-MM-DD`) all use calendar-dated version identifiers — but carry them in a header (Stripe, GitHub), a path segment (Twilio, Shopify), or a query parameter (Azure). Google and Microsoft Graph reject dated versions in favor of coarse path tokens.
2. **Stripe is the only reference that pins accounts.** A Stripe account is locked to the version current at its first request; upgrades are opt-in via dashboard/Workbench, with per-request override via `Stripe-Version`. GitHub's dated header is per-request and defaults to `2022-11-28` when omitted. This is the single biggest divergence on the versioning axis.
3. **Breaking-change definitions are convergent in substance** (add-only is safe; remove/rename/retype/require is breaking), but **new enum values in responses are the contested sub-axis.** Stripe and Azure treat them as breaking unless the client opted in or the enum is explicitly extensible; Zalando invented `x-extensible-enum` (as of its Nov 2025 changelog being deprecated in favor of examples); Google permits them with a warning. This links directly to Part 3's open-enum question.
4. **The `Deprecation` and `Sunset` headers are now both standardized, but adoption is partial.** `Deprecation` became RFC 9745 (Standards Track, March 2025) using an RFC 9651 structured-field date in unix-time `@<unix>` form; `Sunset` is RFC 8594 (Informational, May 2019) using HTTP-date. GitHub and Zalando use both at runtime; Google, Stripe, Microsoft and Shopify lean on changelogs and long fixed windows instead.
5. **Rate-limit header families are genuinely fragmented** and none of the eight has adopted the IETF standard fields.
6. **Conditional-request support is a real split.** GitHub is the standout: every REST response carries an `ETag`, most carry `Last-Modified`, and a `304 Not Modified` from a correctly-authorized conditional request does not count against the primary rate limit. Most other references issue ETags for optimistic concurrency (`If-Match`) rather than for cache-saving reads.
7. **OpenAPI publication is now the norm** — Stripe (`stripe/openapi`), GitHub (`github/rest-api-description`), Azure (`Azure/azure-rest-api-specs`), Twilio (`twilio/twilio-oai`) publish machine-readable specs; Google publishes discovery docs + protos; Zalando *mandates* OpenAPI publication as a rule. Microsoft Graph and Azure additionally gate breaking changes on automated spec-diff tooling.

## Baseline position (RFCs and guideline docs on this surface)

**HTTP caching — RFC 9111** (HTTP Caching, STD 98, June 2022) is the current normative baseline for `Cache-Control`, `ETag`/`If-None-Match`, `Last-Modified`/`If-Modified-Since` and 304 semantics; conditional-request validators derive from RFC 9110 (HTTP Semantics).

**Deprecation — RFC 9745** (The Deprecation HTTP Response Header Field, Standards Track, March 2025). Value is a structured-field Date in unix time, e.g. `Deprecation: @1688169599`. A `deprecation` link relation can point to human-readable migration docs. The header is explicitly a *hint*; deprecation does not change resource behavior. RFC example pairing: `Deprecation: @1688169599` / `Sunset: Sun, 30 Jun 2024 23:59:59 UTC`.

**Sunset — RFC 8594** (The Sunset HTTP Header Field, Informational, May 2019). Uses an HTTP-date (RFC 7231/5322 format), e.g. `Sunset: Wed, 27 Nov 2020 14:34:29 GMT`. The Sunset timestamp MUST NOT be earlier than the Deprecation timestamp; it uses a different date format from Deprecation "for historical reasons." Clients SHOULD treat it as a hint.

**Rate limiting — IETF draft-ietf-httpapi-ratelimit-headers-11** (RateLimit header fields for HTTP), published on the Standards Track by the IETF HTTPAPI Working Group on 23 May 2026 — still an Internet-Draft, not yet an RFC. It defines two structured-field response headers: `RateLimit` (current state; `r`=remaining quota, `t`=time window) and `RateLimit-Policy` (advertised policy; `q`=quota, `w`=window seconds, optional `qu` quota-unit, optional `pk` partition key). Example: `RateLimit-Policy: "burst";q=100;w=60,"daily";q=1000;w=86400`. It registers a `quota-exceeded` problem type and explicitly warns clients MUST NOT treat available quota as an SLA. **None of the eight references emit these fields yet**; the widely-copied `X-RateLimit-*` is a de-facto convention that predates the spec (Cloudflare adopted the draft Sep 2025).

**Guideline docs on this surface:**
- **Zalando RESTful API Guidelines** (a guideline document, not an API; verified against the repo): `MUST not use URL versioning`, `MUST not use media type versioning`, prefer compatible in-place evolution; `SHOULD add Deprecation and Sunset header to responses` and `SHOULD add monitoring for` them; `MUST reflect deprecation in API specifications` via `deprecated: true`; `MUST obtain approval of clients before API shut down`; `MUST collect external partner consent on deprecation time span`; `MUST publish OpenAPI specification`. The info object `MUST` carry a semver `version` and an `x-api-id` (pattern `^[a-z0-9][a-z0-9-:.]{6,62}[a-z0-9]$`).
- **Google AIP / Cloud API Design Guide**: major-version-in-path only (`v1`, never `v1.0/v1.1`); channel-based `v1beta`/`v1alpha`; AIP-180 backwards-compatibility rules; AIP-185 recommends removing beta functionality only after a "sufficient period; we recommend 180 days," and giving beta releases "a reasonable transition period; we recommend 180 days."
- **Microsoft Azure REST API Guidelines** (guideline doc): `DO use a required query parameter named api-version`; `DO use YYYY-MM-DD date values, with a -preview suffix`; `DO NOT include a version number segment in any operation path`; `DO return HTTP 400 MissingApiVersionParameter` if omitted; preview versions may be retired after 90 days' notice and must not stay in preview >1 year; `DO NOT remove values from your enumeration list.`

## Side-by-side comparison tables

### Table 1 — Versioning
| Reference | Scheme | Identifier & transport | Example | Support window |
|---|---|---|---|---|
| Stripe | Dated, account-pinned + named major releases | `Stripe-Version` header; path `/v1` | `2026-06-24.dahlia` (current); override `Stripe-Version: 2026-06-24.dahlia` | Compat maintained with every version since 2011 |
| GitHub REST | Calendar-dated, per-request | `X-GitHub-Api-Version` header | `X-GitHub-Api-Version: 2022-11-28` (default) | ≥24 months per version after successor ships |
| Google/AIP | Major-version + stability channel in path | Path `/v1`, `/v1beta`, `/v1alpha` | `google.library.v1` | Stable: life of major version; beta shutdown ~180 days after GA |
| MS Graph | Coarse channel in path | Path `/v1.0`, `/beta` | `graph.microsoft.com/v1.0/...` | Deprecated version declared ≥24 months before retirement |
| Azure REST | Dated query parameter (required) | `?api-version=YYYY-MM-DD[-preview]` | `?api-version=2021-06-04` | Preview retire ≥90 days; stable long-lived |
| Twilio | Single frozen dated path | Path `/2010-04-01/` | `/2010-04-01/Accounts/{sid}/Messages` | Stable since 2010; features added in place |
| Shopify Admin REST | Quarterly dated path | Path `/admin/api/2026-01/` | `/admin/api/2026-01/products.json` | Each version ~12 months; REST Admin legacy as of Oct 1 2024 |
| AWS (Query APIs) | Dated `Version=` parameter | Query `Version=YYYY-MM-DD` | EC2 `Version=2016-11-15`; STS `Version=2011-06-15` | Version pinned for years; features added in place |
| Zalando (guideline) | No URL versioning; compatible evolution | semver `version` in OpenAPI info only | `version: 1.1.0` | N/A (guideline) |

### Table 2 — Deprecation / Sunset signaling
| Reference | `Deprecation` hdr | `Sunset` hdr | Removed-version behavior | Primary channel |
|---|---|---|---|---|
| Stripe | No (docs/changelog) | No | N/A (never removes) | API changelog, version-aware docs |
| GitHub | Yes (HTTP-date, RFC 7231) | Yes (RFC 8594) | `410 Gone` for closed-down version | Changelog + email + blog |
| Google/AIP | proto `deprecated=true` | Not header-based | New major version; deprecation clock | Docs/comments |
| MS Graph | Docs-level | Docs-level | Version increment | Changelog |
| Azure | Docs-level | Docs-level | 400 for missing api-version | azure-rest-api-specs review board |
| Twilio | Rare | Rare | Endpoint removal w/ notice | Changelog |
| Shopify | Docs-level | Docs-level | Version drops off supported list | Developer changelog, quarterly |
| Zalando (guideline) | SHOULD (both headers) | SHOULD | Consumer consent required first | OpenAPI `deprecated: true` |

### Table 3 — Rate limiting
| Reference | Model | Headers | 429 behavior | Scope / numbers |
|---|---|---|---|---|
| Stripe | Token bucket, per-account | Limited `Stripe-Ratelimit-*`; `Stripe-Rate-Limited-Reason` on 429 | 429; exponential backoff | 100 parallel req/s live; 25 req/s test; some endpoints (e.g. Search) stricter |
| GitHub | Fixed hourly buckets + secondary point system | `x-ratelimit-limit/remaining/used/reset`; `retry-after` | 403 or 429, `x-ratelimit-remaining: 0` | 5,000/hr authed PAT; 60/hr unauth; 15,000/hr Enterprise Cloud apps; 1,000/hr `GITHUB_TOKEN`/repo |
| Google | Per-project quotas | Console-managed, not header-standard | 429 | Per-project/per-method |
| MS Graph | Token-bucket cost; per-app+tenant+user+resource | `Retry-After` (not always present); `RateLimit-Reason` | 429 (sometimes 503) | Outlook: 10,000 requests / 10 min per app+mailbox |
| Twilio | Concurrency + per-product | `Retry-After` on 429 | 429; averaged over 10s | Product-specific |
| Shopify REST | Leaky bucket | `X-Shopify-Shop-Api-Call-Limit: 32/40`; `Retry-After` | 429 | Bucket 40 / leak 2 req/s standard; 400 / 20 req/s Plus |
| AWS | Token bucket per service/API | Service-specific | 429 / `ThrottlingException` | Per-service, per-region |
| Zalando (guideline) | Recommends 429 | Recommends standard headers | 429 | N/A |

### Table 4 — Caching / conditional reads
| Reference | ETag | Last-Modified | 304 effect | Notes |
|---|---|---|---|---|
| GitHub | Yes (all) | Yes (most) | 304 does **not** count vs primary rate limit | If-None-Match / If-Modified-Since; HEAD supported |
| Stripe | Not for read caching | No | N/A | Recommends client-side caching; no conditional reads |
| Google | Varies by service | Varies | Standard | — |
| MS Graph | Yes (`If-Match` concurrency) | Sometimes | Standard | delta queries preferred over polling |
| Azure | `ETag` + `If-Match` required for optimistic concurrency | Yes | Standard | Guideline mandates ETag support for updates |
| Twilio | Generally no | No | N/A | — |
| Shopify | Limited | No | N/A | Leans on webhooks |
| Zalando (guideline) | SHOULD for concurrency | — | — | — |

### Table 5 — Auth surface
| Reference | Scheme | Key format / prefix | 401 vs 403 | Rotation surface |
|---|---|---|---|---|
| Stripe | Secret key as HTTP Basic username (Bearer also accepted) | `sk_live_`, `sk_test_`, `pk_` (publishable), restricted keys | 401 auth, 403 perms | Dashboard → Developers → API keys; manual |
| GitHub | Bearer/token header | `ghp_` PAT, `gho_` OAuth, `ghu_` user-to-server, `ghs_` server-to-server, `ghr_` refresh, `github_pat_` fine-grained | 401 bad cred, 403/404 perms | Developer settings; expiry supported |
| Google | OAuth bearer / API key | — | 401/403 | Cloud console |
| MS Graph | OAuth 2.0 bearer | JWT access tokens | 401 expired, 403 no access | Entra ID |
| Twilio | HTTP Basic | Account SID `AC…` + Auth Token; or API Key `SK…` + secret | 401 | Secondary Auth Token + promote endpoint |
| Shopify | Custom header | `shpat_` (offline), `shpua_`, `shpss_` | 401 expired, 403 valid-but-no-access | App settings; online tokens expire |
| AWS | **SigV4 request signing** | Access key `AKIA…` + secret (never sent) | 403 SignatureDoesNotMatch | IAM; STS temp creds |
| Zalando (guideline) | OAuth 2.0 bearer, `uid` scope | — | — | — |

### Table 6 — Spec & docs practice
| Reference | OpenAPI/spec source | Changelog | Notes |
|---|---|---|---|
| Stripe | `stripe/openapi` (spec3, v1+v2, preview & latest) | Dated API changelog | Version-aware reference docs |
| GitHub | `github/rest-api-description` (3.0 bundled + 3.1 in `descriptions-next`) | GitHub Changelog + docs version picker | No PRs to spec accepted |
| Google | Discovery docs + protobuf sources | Per-service release notes | AIP repo public |
| Azure | `Azure/azure-rest-api-specs` (TypeSpec/OpenAPI) | Per-service | Breaking-change review board + azure-openapi-diff |
| MS Graph | Metadata (`$metadata` CSDL) + docs repo | Microsoft Graph changelog | — |
| Twilio | `twilio/twilio-oai` | Product changelog | — |
| Shopify | Published reference | Quarterly developer changelog | REST reference maintained but legacy |
| Zalando | Mandates OpenAPI publication (rule) | Guideline changelog on GitHub | API Linter service |

## Per-reference notes (lifecycle character sketches)

**Stripe** — The maximalist of backward compatibility: it has maintained compatibility with every API version since 2011 via account pinning plus an internal request/response transformation layer ("gates" + version modules that downgrade responses in reverse chronological order). Version identifiers are dates with named major-release suffixes (Acacia, Basil, Dahlia); the current version is `2026-06-24.dahlia`. Monthly releases are additive-only; twice-yearly named releases carry breaking changes. Override per request with `Stripe-Version`; upgrade the account default in Workbench (72-hour rollback window). Its published backward-compatibility list is the field's de-facto reference (see appendix). Rate limits are 100 parallel req/s in live mode, 25 req/s in test, 429-centric with a `Stripe-Rate-Limited-Reason` header but otherwise minimal public rate-limit headers. Auth is a secret key (`sk_live_`/`sk_test_`) used as HTTP Basic username; publishable `pk_` keys for client-side. OpenAPI published at `stripe/openapi`.

**GitHub REST** — Calendar-versioned via `X-GitHub-Api-Version`; introduced versioning in 2022 with `2022-11-28` as the baseline, which remains the default when the header is omitted. Best-in-class on the operational surface: nine distinct rate-limit buckets, `x-ratelimit-*` headers, secondary-limit protection, and the standout conditional-request behavior where an authorized 304 costs nothing against the primary limit. Deprecation/Sunset headers used; unsupported versions return `400`, closed-down versions `410 Gone`. Token prefixes (`ghp_` etc. + CRC32/Base62 checksum) are the model much of the industry copied. Spec at `github/rest-api-description`; PRs to the spec are not accepted.

**Google Cloud / AIP** — Doctrine, not just an API. Major version only in the path (`v1`), never minor/patch; stability communicated via channels (`v1alpha`, `v1beta`, stable), with beta a superset of stable and alpha a superset of beta. AIP-180 codifies backward-compatibility; breaking changes require a new major version with a deprecation clock. New enum values in response messages are permitted but flagged as a client hazard that "should document this." Beta functionality recommended to shut down ~180 days after reaching stable.

**Microsoft (Azure + Graph)** — Two distinct regimes. Azure REST guidelines mandate a required `api-version=YYYY-MM-DD[-preview]` query parameter, forbid path version segments, return `400 MissingApiVersionParameter` when omitted, and gate every change through `azure-rest-api-specs` + `azure-openapi-diff` (47 change types detected) and a weekly Breaking Change Review Board. Microsoft Graph uses coarse path channels (`/v1.0`, `/beta`), declares deprecations ≥24 months ahead, and is throttling-heavy: a token-bucket cost model scoped per-app/per-tenant/per-user/per-resource, 429 + `Retry-After` (not always present, sometimes `Retry-After: 0`), occasionally 503.

**Twilio** — The frozen-version outlier: the path has read `/2010-04-01/` for its entire life; new capabilities are added in place rather than by version bump (the older v2008 API is deprecated). Auth is HTTP Basic with Account SID (`AC…`, 34 chars) + Auth Token, though API Keys (`SK…` + secret) are recommended for production; auth-token rotation has dedicated secondary-token create/delete + promote endpoints. Rate limits are averaged over 10 seconds, 429 + `Retry-After`. OpenAPI at `twilio/twilio-oai`.

**Shopify Admin REST** — Quarterly dated path versions (`2026-01`, released Jan/Apr/Jul/Oct), each supported ~12 months. The REST Admin API is officially **legacy** as of Oct 1 2024; new public apps must use the GraphQL Admin API since Apr 2025 (noted only for REST's lifecycle status; GraphQL is out of scope). Leaky-bucket rate limiting with `X-Shopify-Shop-Api-Call-Limit: n/40` and `Retry-After`; standard bucket 40 / leak 2 req/s, Shopify Plus 400 / 20 req/s. Auth via `X-Shopify-Access-Token: shpat_…` (NOT `Authorization: Bearer`, which returns 401); 401 = expired token, 403 = valid token without access. **Currency caveat:** Shopify money fields are version- and currency-sensitive — amounts are returned in shop currency alongside a separate `presentment_currency`, and the multi-currency representation has shifted across API versions, so amounts must be read against the pinned version and explicit currency code rather than assumed.

**Zalando** — A guideline document, deliberately contrarian on versioning: `MUST not use URL versioning`, `MUST not use media type versioning`, prefer compatible evolution ("on the wire" compatibility only, not generated-code compatibility). Deprecation is a formal, consent-driven process: `deprecated: true` in the spec, `SHOULD` emit Deprecation + Sunset headers, `MUST` obtain consumer/partner consent before shutdown, `MUST` monitor usage of sunset-scheduled APIs. `MUST publish OpenAPI`. It invented `x-extensible-enum` to force clients to tolerate new enum values; as of the 2025-11-27 changelog this is being deprecated in favor of examples-based open enumerations (transition in progress).

**AWS** — Two contrasts. On **auth**, SigV4 is the field's outlier: no bearer token on the wire; each request carries `Authorization: AWS4-HMAC-SHA256 Credential=…, SignedHeaders=…, Signature=…` derived from a four-step HMAC chain over a canonical request, proving identity + integrity + timeliness at once (SigV4a adds ECDSA-P256 for multi-region). The secret access key is never transmitted; a mismatch yields `403 SignatureDoesNotMatch`. On **versioning**, older Query-protocol services carry a dated `Version=YYYY-MM-DD` request parameter (EC2 `Version=2016-11-15`, STS `Version=2011-06-15`) — required in raw Query requests, auto-supplied by SDKs/CLI, and pinned for years with new features added in place; newer JSON/REST services bake the version into the service model rather than a per-request parameter. AWS's posture is strong backward compatibility within a service instead of version bumps ("Some AWS services maintain multiple API versions to support backward compatibility. By default, SDK and AWS CLI operations use the latest available API version").

## Agreements vs divergences

**Agreements:**
- Dated identifiers dominate at the developer-platform tier (Stripe, GitHub, Twilio, Shopify, Azure, AWS Query).
- Substance of breaking vs additive is near-universal: add-only is safe; remove/rename/retype/require is breaking.
- 429 is the universal "too many requests" signal; exponential backoff (ideally with jitter) is universally recommended.
- OpenAPI (or equivalent machine-readable spec) publication is now standard practice.
- 401 = authentication failure, 403 = authorization failure is the shared intent (GitHub sometimes returns 404 to avoid leaking resource existence; AWS uses 403 SignatureDoesNotMatch).

**Divergences (each with tradeoff):**
- **Version transport (header vs path vs query/param).** Header (Stripe/GitHub) keeps URLs clean and stable but is invisible and easy to omit; path (Google/MS/Twilio/Shopify) is explicit and cache-friendly but couples clients to URL structure; query/param (Azure/AWS) is explicit but pollutes every request.
- **Account-pinning vs per-request.** Stripe's pinning guarantees no accidental breakage but concentrates enormous maintenance cost in a transformation layer; per-request/explicit (everyone else) is simpler to operate but shifts the upgrade burden onto clients.
- **New enum values.** Treating them as breaking (Stripe/Azure) protects naive clients but slows evolution; permitting them (Google; Zalando's extensible-enum contract) speeds evolution but demands defensive clients.
- **Runtime deprecation headers vs changelog-only.** Headers (GitHub/Zalando) enable automated client alerting but require discipline; changelog-only (Stripe/Google/Shopify) is simpler but relies on humans reading release notes.
- **Rate-limit transparency.** Rich headers (GitHub, Shopify) let clients self-throttle; sparse/429-only (Stripe, Graph) simplifies the server contract but forces reactive backoff.
- **Conditional reads.** GitHub's "free 304s" strongly incentivize caching; most others don't reward it, so clients poll.
- **Signed requests vs bearer.** SigV4 (AWS) never exposes the secret and detects tampering but is complex to implement by hand; bearer tokens are trivial but a leaked token is game-over until rotated.

## CONTESTED AXES REGISTER (scoped to Part 6)

| Axis | Options observed | Who does what | Tradeoff (one line) | How contested |
|---|---|---|---|---|
| Versioning scheme/transport | Account-pinned dated header · per-request dated header · path major/channel · dated query param · dated path | Stripe (pinned header); GitHub (per-req header); Google/MS Graph (path channel); Azure/AWS (query/param); Twilio/Shopify (dated path) | Header=clean URLs but hidden; path=explicit but coupling; query=explicit but verbose | **Wide-open** |
| Account-pinning vs per-request version | Pinned default · per-request/explicit default | Stripe pins; everyone else per-request/explicit | Pinning=no accidental breaks, huge maint cost; per-request=simple ops, client burden | **Split** (Stripe alone) |
| Breaking-change strictness | Never break (transform layer) · new major version · in-place with long notice | Stripe (never); Google/MS/Shopify (new version); AWS/Twilio (in-place) | Never-break=max client safety, max provider cost | **Split** |
| New enum values in responses = breaking? | Breaking unless opt-in · permitted-with-warning · extensible-enum contract | Stripe/Azure (breaking); Google (permitted+warn); Zalando (x-extensible-enum) | Breaking=safe naive clients; permitted=faster evolution | **Split** (links to Part 3 open-enum) |
| Sunset signaling | Runtime Deprecation+Sunset headers · docs/changelog only | GitHub/Zalando (headers); Stripe/Google/MS/Shopify (docs) | Headers=automatable alerts; docs=simpler but human-dependent | **Split** |
| Rate-limit header family | `X-RateLimit-*` · leaky-bucket call-limit · 429+Retry-After only · minimal vendor headers · IETF `RateLimit-*` | GitHub (X-RateLimit); Shopify (Shop-Api-Call-Limit); Graph/Twilio (Retry-After); Stripe (minimal); nobody (IETF) | Rich headers=client self-throttle; sparse=simpler contract | **Wide-open** |
| Quota model | Fixed window · leaky bucket · token-bucket cost · concurrency | GitHub (fixed hourly); Shopify (leaky); Stripe/Graph/AWS (token/cost); Twilio (concurrency) | Bucket=bursts OK; fixed=simple but boundary spikes | **Wide-open** |
| Conditional-request support | Free 304s not counted · ETag for concurrency only · none | GitHub (free 304s); Azure/Graph (If-Match concurrency); Stripe/Twilio/Shopify (none) | Free 304s=cheap polling; none=clients waste quota | **Split** |
| Key-prefix convention | Typed prefixes+checksum · unprefixed SID+token · access-key+signature | Stripe/GitHub/Shopify (prefixed); Twilio (SID+token); AWS (access key) | Prefixes=secret-scanning + identifiability; unprefixed=legacy | **Near-consensus** (prefixing winning) |
| 401 vs 403 line | 401 auth / 403 perms · 404 to hide existence · 403 SignatureDoesNotMatch | Most (401/403); GitHub (404 sometimes); AWS (403 sig) | Strict split=clear semantics; 404-hide=security | **Near-consensus** |
| Signed requests vs bearer | SigV4 signing · bearer/API-key | AWS (SigV4); all others (bearer/key) | Signing=no secret on wire, tamper-proof, complex; bearer=simple, leak-fragile | **Split** (AWS alone) |

## EXAMPLES APPENDIX (verbatim artifacts & concrete numbers; retrieved 2026-07-19)

### Stripe
- Current version: `2026-06-24.dahlia` (Stripe API changelog). Named major releases carry breaking changes (e.g. Acacia `2024-09-30.acacia`); monthly releases additive-only.
- Per-request override header: `Stripe-Version: 2026-06-24.dahlia`. Org API keys MUST send `Stripe-Version`.
- Rate limits (Stripe Support, verbatim): "Stripe has a rate limit of 100 parallel requests per second for live mode transactions, and 25 parallel requests per second for test mode transactions." On 429, docs state responses "include a Stripe-Rate-Limited-Reason header that explains why the request was rate-limited." Concurrency limiter separate; `lock_timeout` 429s auto-retried by SDKs. Search and some endpoints stricter.
- Upgrade: Workbench, 72-hour rollback window.
- Auth: `-u sk_test_<redacted-see-README>:` (secret key as Basic username, empty password); Bearer also accepted.
- Backward-compatible changes (verbatim Stripe list): adding new API resources; adding new optional request parameters to existing methods; adding new properties to existing responses; changing the order of properties in existing responses; changing the length or format of opaque strings (object IDs, error messages) including adding/removing fixed prefixes like `ch_` (IDs assumed ≤255 chars, store as `VARCHAR(255)`); adding new event types (listeners must handle unfamiliar types gracefully).
- Enum rule (Stripe API custodian): returning a new enum value is only backward-compatible if the user opted into it (e.g. new payment method type) or the enum is clearly non-static (banks, currencies); otherwise new response enum values are breaking (e.g. `2025-07-30` changelog added a new payment-review close reason as a versioned change).
- OpenAPI: `github.com/stripe/openapi`, `spec3.{json,yaml}`, `/latest/` (v1+v2), `/preview/`; OpenAPI 2.0 deprecated.

### GitHub
- Version header: `curl --header "X-GitHub-Api-Version:2022-11-28" https://api.github.com/zen`. Default when omitted: `2022-11-28`. Unsupported version → `400`; retired/closed-down version → `410 Gone`. Requests without a valid `User-Agent` rejected.
- Support window: previous version supported ≥24 months after a successor ships.
- Deprecation header (HTTP-date, RFC 7231): `Deprecation: Wed, 27 Nov 2019 14:34:29 GMT`. Sunset (RFC 8594): `Sunset: Fri, 27 Nov 2020 14:34:29 GMT`.
- Rate limits (GitHub Docs, verbatim): unauthenticated = 60 requests/hour; authenticated PAT = 5,000 requests/hour; GitHub App owned by a GitHub Enterprise Cloud organization = 15,000 requests/hour; `GITHUB_TOKEN` = 1,000 requests/hour per repository. Git LFS bucket: 3,000 req/min authed.
- Headers: `x-ratelimit-limit`, `x-ratelimit-remaining`, `x-ratelimit-used`, `x-ratelimit-reset` (UTC epoch seconds); `retry-after` on secondary limits. Exceed primary → 403 or 429 with `x-ratelimit-remaining: 0`.
- Secondary-limit point costs (third-party corroboration; GitHub does not fully publish internals): GET/HEAD/OPTIONS 1 pt; POST/PATCH/PUT/DELETE 5 pts.
- `GET /rate_limit` does not count against the primary limit. Example resources object: `"core": {"limit": 5000, "remaining": 4999, "reset": 1372700873, "used": 1}`.
- Conditional requests (verbatim): "Making a conditional request does not count against your primary rate limit if a 304 response is returned and the request was made while correctly authorized with an Authorization header." Example: `If-None-Match: "0c05f64..."` → `HTTP/2 304` with `x-ratelimit-remaining` unchanged. HEAD supported to avoid transferring bodies.
- Token prefixes: `ghp_` (PAT), `gho_` (OAuth), `ghu_` (user-to-server), `ghs_` (server-to-server), `ghr_` (refresh); `github_pat_` (fine-grained, 93 chars). Underscore separator (non-Base64); last 6 chars are a CRC32/Base62 checksum; plan for tokens up to 255 chars.
- Spec: `github/rest-api-description` (3.0 bundled in `descriptions/`, 3.1 in `descriptions-next/`, stable since 1.1.4).

### Google / AIP
- Path version: `v1`, `v1beta1`, `v1alpha5`; protobuf package suffix `google.library.v1`. No minor/patch (`v1`, never `v1.0`).
- Channel-based: beta must be a superset of stable; alpha a superset of beta. AIP-185 (verbatim): "The beta channel's functionality may be removed after it has been deprecated for a sufficient period; we recommend 180 days" and "a beta release should allow users a reasonable transition period; we recommend 180 days." Deprecated functionality must not graduate channels.
- Deprecation marker: proto `option deprecated = true;`.
- AIP-180 backward-compatibility: must not remove/rename components in same major version; must not add required request fields; must not change field types even if wire-compatible; resource names must not change; enum values freely addable to request-only enums but "may be added" with caution to response enums, which "should document this."
- AIP versioning of the AIPs themselves is date-based ISO-8601 (`v2023-03-28`).

### Microsoft (Azure + Graph)
- Azure: `PUT https://service.azure.com/users/Jeff?api-version=2021-06-04`. Missing param → `400` with error code `MissingApiVersionParameter` and message "The api-version query parameter (?api-version=) is required for all requests." Preview suffix `-preview`; preview retire ≥90 days; max 1 year in preview. Extensible enums: SHOULD NOT accept a value not defined for the requested api-version; DO NOT remove enum values.
- Azure spec-diff: `azure-openapi-diff` detects 47 change types; Breaking Change Review Board weekly office hours; specs in `Azure/azure-rest-api-specs` (TypeSpec/OpenAPI).
- Graph: current version `v1.0`; beta at `graph.microsoft.com/beta`. Deprecation declared ≥24 months before retirement (GA APIs and versions).
- Graph throttling (verbatim, Microsoft Learn): `HTTP/1.1 429 Too Many Requests` / `Content-Type: application/json` / `Retry-After: 10` / `{"error":{"code":"TooManyRequests","innerError":{"code":"429",...},"message":"Please retry again later."}}`. Token-bucket cost model; scoped per app+tenant (tenant sizes S<50, M 50–500, L>500 users), per-app-across-tenants, per-user, per-resource. Outlook service limit: 10,000 API requests in a 10-minute period per app ID + mailbox combination (~16–17 req/s). Some endpoints omit `Retry-After`; `Retry-After: 0` can occur (still back off). `ConsistencyLevel: eventual` raises request cost. Batch requests evaluated individually against limits; batch envelope returns 200 even if members return 429.
- Graph concurrency/ETag: `If-Match` for optimistic concurrency; delta queries recommended over polling.

### Twilio
- Path version frozen at `/2010-04-01/`. Verbatim: `https://api.twilio.com/2010-04-01/Accounts/{sid}/Messages`. v2008 deprecated; 2010-04-01 is the "latest stable" version.
- Auth: HTTP Basic. `-u $YOUR_ACCOUNT_SID:$YOUR_AUTH_TOKEN` (Account SID pattern `^AC[0-9a-fA-F]{32}$`, 34 chars) for testing; API Key `SK…` + secret recommended for production. `securitySchemes: accountSid_authToken: scheme: basic` in twilio-oai.
- Auth Token rotation: secondary-token create/delete endpoint + a promote endpoint; response returns "the promoted Auth Token that must be used to authenticate future API requests."
- Rate limits: 429 + `Retry-After`; "When API rate limits are calculated, they are averaged over 10 seconds." IAM/SCIM endpoint limits not publicly documented.
- Spec: `twilio/twilio-oai` (`twilio_api_v2010.yaml`).

### Shopify Admin REST
- Path version: `/admin/api/2026-01/products.json`. Releases quarterly (Jan/Apr/Jul/Oct); each supported ~12 months. REST Admin API legacy as of Oct 1 2024; new public apps must use GraphQL since Apr 2025.
- Response header carries active version: `X-Shopify-API-Version: 2023-01`.
- Rate limits: leaky bucket, standard bucket 40, leak 2 req/s; Shopify Plus bucket 400, leak 20/s; Advanced leak 4/s. Header: `X-Shopify-Shop-Api-Call-Limit: 32/40` (current/bucket). On overflow: 429 + `Retry-After` (seconds). Extra GET limit: `page` offset >100,000 → 429 (e.g. `?limit=250&page=401` = offset 100,250); page-based pagination deprecated in 2019-07.
- Verbatim header sample: `"X-Shopify-Shop-Api-Call-Limit": ["1/40"]`, `"X-Shopify-API-Version": ["2023-01"]`.
- Auth: `-H "X-Shopify-Access-Token: shpat_xxxxxxxxxxxx"`. `Authorization: Bearer` returns 401. Token prefixes: `shpat_` (offline/permanent admin), `shpua_` (user), `shpss_` (app client secret / server). 401 = expired token; 403 = valid token without access. Must use canonical `{shop}.myshopify.com`.
- **Currency caveat:** money fields are version- and currency-sensitive; amounts return in shop currency with a separate `presentment_currency`, and the multi-currency representation has changed across API versions — read amounts against the pinned version and explicit currency code, never assume a single currency.

### AWS
- Query-protocol versioning: dated `Version=YYYY-MM-DD` request parameter. EC2 `Version=2016-11-15`, STS `Version=2011-06-15`. Verbatim EC2: `https://ec2.amazonaws.com/?Action=RunInstances&ImageId=ami-2bb65342&MaxCount=3&MinCount=1&Placement.AvailabilityZone=us-east-1a&Monitoring.Enabled=true&Version=2016-11-15&X-Amz-Algorithm=AWS4-HMAC-SHA256&...`. Verbatim STS: `https://sts.amazonaws.com/?Version=2011-06-15&Action=GetSessionToken&DurationSeconds=1800&AUTHPARAMS`. Common-parameter definition (verbatim): "Version — The API version that the request is written for, expressed in the format YYYY-MM-DD." Required in raw Query requests; SDKs/CLI auto-supply. Newer JSON/REST services bake version into the service model, no per-request `Version=`. SDK config `api_versions` (e.g. `ec2 = 2015-03-01`) can pin.
- Backward compatibility: EC2 pinned at 2016-11-15 and STS at 2011-06-15 for years; features added in place. SDK doc (verbatim): "Some AWS services maintain multiple API versions to support backward compatibility. By default, SDK and AWS CLI operations use the latest available API version." STS version also appears in every response namespace, e.g. `<AssumeRoleResponse xmlns="https://sts.amazonaws.com/doc/2011-06-15/">`.
- Auth SigV4: `Authorization: AWS4-HMAC-SHA256 Credential=AKIAIOSFODNN7EXAMPLE/20130524/us-east-1/s3/aws4_request, SignedHeaders=host;range;x-amz-date, Signature=<hex>`. Signing key = `HMAC-SHA256(HMAC-SHA256(HMAC-SHA256(HMAC-SHA256("AWS4"+SecretKey,Date),Region),Service),"aws4_request")`. Secret access key never sent. Presigned query auth: `X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=…&X-Amz-Date=…&X-Amz-Expires=86400&X-Amz-SignedHeaders=host&X-Amz-Signature=…` (S3 presign max 7 days). SigV4a: `AWS4-ECDSA-P256-SHA256` + `X-Amz-Region-Set` for multi-region. `403 SignatureDoesNotMatch` on mismatch. Streaming: `x-amz-content-sha256: STREAMING-AWS4-HMAC-SHA256-PAYLOAD`.

### Baseline RFC artifacts
- RFC 9745 example: `Deprecation: @1688169599` / `Sunset: Sun, 30 Jun 2024 23:59:59 UTC` (deprecated Fri 30 Jun 2023 23:59:59 UTC).
- RFC 8594: `Sunset: Wed, 27 Nov 2020 14:34:29 GMT`; clients SHOULD treat as hint.
- IETF ratelimit draft-11 (23 May 2026): `RateLimit-Policy: "burst";q=100;w=60,"daily";q=1000;w=86400`; `RateLimit` with `r`/`t` parameters; registers a `quota-exceeded` problem type; MUST NOT treat quota as an SLA.

## Recommendations
*(For the later prescriptive phase — staged, with the benchmarks that would change each decision. Descriptive framing: these identify where the field's evidence points, not what the standard must adopt.)*

1. **First, resolve the version-transport axis, because it constrains everything downstream.** The evidence shows two coherent camps: dated-header (Stripe/GitHub) and dated-path/param (Twilio/Shopify/Azure/AWS). If the future standard prioritizes clean, stable resource URLs and per-request flexibility, the GitHub `X-{Vendor}-Api-Version` dated-header model is the best-documented template; if it prioritizes explicit, cache-key-visible, hard-to-omit versioning, the Azure required-`api-version` model is the strictest. *Threshold to switch camps:* if the consumer base is dominated by CDN/cache-heavy GET traffic, path/param versioning becomes materially more attractive.

2. **Decide account-pinning vs per-request explicitly and early — it is a one-way door.** Stripe's pinning delivers the strongest client-safety guarantee in the set but requires a permanent server-side transformation layer; no other reference pays that cost. Adopt pinning only if you can commit to indefinite compatibility engineering; otherwise follow the majority per-request/explicit model. *Benchmark:* if you cannot staff a compatibility/transformation layer, do not pin.

3. **Standardize deprecation signaling on the now-ratified RFCs regardless of version transport.** `Deprecation` (RFC 9745, `@<unix>`) + `Sunset` (RFC 8594, HTTP-date) + a `deprecation` Link relation give automatable client alerting; GitHub and Zalando already model this. Pair with a changelog and a fixed minimum window (GitHub's 24 months and Google's ~180-day beta window are the concrete anchors). *Benchmark:* adopt runtime headers if any consumers run unattended automation; changelog-only is defensible only for small, human-monitored integrations.

4. **On rate limiting, choose a quota model first, then expose headers.** Leaky/token bucket (Shopify, Stripe, Graph, AWS) tolerates bursts and is the majority; fixed hourly windows (GitHub) are simplest but cause boundary spikes. Whichever is chosen, emitting remaining/limit/reset headers (GitHub/Shopify style) demonstrably lets clients self-throttle. *Forward-looking benchmark:* track IETF `RateLimit`/`RateLimit-Policy` adoption — if a second major provider beyond Cloudflare ships it, re-evaluate emitting the standard fields alongside vendor headers.

5. **Reward conditional reads.** GitHub's "authorized 304s don't count against the limit" is the single highest-leverage caching incentive in the set and is cheap to specify. *Benchmark:* if polling-heavy clients dominate traffic, this should be non-negotiable.

6. **On auth, default to prefixed bearer/API-key tokens (the near-consensus) and reserve request signing for high-assurance surfaces.** Typed prefixes + checksum (GitHub/Stripe/Shopify) materially improve secret-scanning and identifiability at near-zero cost. SigV4-style signing buys tamper-evidence and never-on-the-wire secrets but at high client complexity — justified only where those properties are required. Keep the 401-auth / 403-authz line strict.

## Caveats
- **Currency of numbers:** all version strings, rate-limit numbers and support windows retrieved 2026-07-19 and change frequently — Stripe's current version, Shopify's quarterly version and GitHub's default header date in particular.
- **Secondary sources:** GitHub secondary-limit point costs and some Shopify/Stripe rate-limit figures come from third-party engineering write-ups corroborating primary docs; treat exact point costs as indicative. GitHub's own docs note secondary-limit internals are not fully published.
- **AWS backward-compatibility statement** is strongest in an archived older EC2 document; the live SDK doc corroborates the "latest by default / multiple versions for compatibility" posture. The old-vs-new versioning contrast is a synthesis from protocol behavior + SDK docs, not a single verbatim AWS sentence.
- **IETF RateLimit draft-11** is an Internet-Draft on the Standards Track, not yet an RFC; syntax may still change. Cloudflare adopted it (Sep 2025); none of the eight references here have.
- **Scope:** OAuth/OIDC protocol flows, GraphQL/gRPC, gateways, SDK design and event streaming are out of scope; reliability is Part 5, webhooks Part 7. Shopify's GraphQL Admin API (now the recommended surface) is noted only where it bears on REST's lifecycle status.
- **Zalando `x-extensible-enum`:** as of the 2025-11-27 changelog it is being deprecated in favor of examples-based open enumerations; both states are noted since the transition is in progress.
- **Stripe test-mode limit:** stated as 25 parallel req/s by Stripe Support; some secondary sources describe test limits as "lower" or "~25% of live." The Support figure is used here as the primary.