# REST API Conventions Series — Part 1/7: Foundations & Standards Layer

*Descriptive research. Retrieval date: July 19, 2026. Primary sources: IETF Datatracker/RFC Editor, spec texts, official API docs. Secondary sources used only for corroboration.*

## 1. TL;DR
- The foundational HTTP layer is settled and near-universally adopted at the transport level (RFC 9110/9111 semantics, RFC 3339 timestamps, JSON), but the "API convention" layer above it (Problem Details, Web Linking, JSON Merge/JSON Patch, Sunset/Deprecation, RateLimit, HATEOAS) is adopted piecemeal — most of the eight reference APIs use proprietary shapes rather than the IETF conventions.
- Standards status as of the run date: RFC 9110 and 9111 are Internet Standards (June 2022); RFC 9457 obsoleted 7807 (July 2023); the Deprecation header became RFC 9745 (March 2025); the RateLimit fields (draft-11) and Idempotency-Key (draft-07, expired) remain Internet-Drafts, not RFCs.
- On REST maturity, essentially all eight reference APIs operate at Richardson Level 2 (resources + HTTP verbs); none implements full Level 3 HATEOAS as its primary contract — confirming that "true REST"/hypermedia is largely aspirational in major industry practice.

## 2. Key Findings
1. **The core HTTP semantics/caching RFCs were re-consolidated in 2022 without changing wire behavior.** RFC 9110 (HTTP Semantics) and RFC 9111 (HTTP Caching) restructured the RFC 723x family into version-independent documents; they clarify but do not alter the safe/idempotent/cacheable method properties.
2. **Problem Details (RFC 9457) is widely cited but selectively adopted.** Among the eight references, only Zalando's guidelines mandate it; Stripe, GitHub, Google, Microsoft Graph, Twilio, Shopify, and AWS all use proprietary JSON error shapes.
3. **RFC 3339 timestamps are the field norm, with Twilio the notable deviation** (RFC 2822 dates), and Google explicitly carving out epoch-seconds as an exception.
4. **The Link header (RFC 8288) splits the field cleanly:** GitHub and Shopify use it for pagination; Stripe, Twilio, Google, and Microsoft use body-based cursor pagination.
5. **JSON Merge Patch vs JSON Patch remains genuinely contested;** Merge Patch's "null means delete" rule is the defining semantic and the reason it can't represent an explicit-null update.
6. **The Deprecation header is now a standard (RFC 9745, March 2025); the Sunset header (RFC 8594) is its older companion.** Both remain thinly adopted in production.
7. **Idempotency-Key is a de-facto standard driven by Stripe but still an unfinished IETF draft** — its most recent revision expired in April 2026.
8. **OpenAPI 3.1 achieved full JSON Schema 2020-12 alignment**, ending OpenAPI's divergent schema dialect; most of the eight publish OpenAPI descriptions (Shopify being the main exception, favoring GraphQL).
9. **HATEOAS is effectively absent as a primary contract** across all eight; Level 2 is the de-facto industry definition of "RESTful."

## 3. Per-Standard Sections

### 3.1 RFC 9110 — HTTP Semantics
**What it specifies.** RFC 9110 (June 2022, Internet Standard STD 97; Fielding, Nottingham, Reschke eds.) is the version-independent definition of HTTP semantics shared across HTTP/1.1, /2, and /3. It defines:
- **Method properties (§9.2):** *safe* (no observable server side-effects — GET, HEAD, OPTIONS, TRACE), *idempotent* (multiple identical requests have the same effect as one — GET, HEAD, OPTIONS, TRACE, PUT, DELETE), and *cacheable*. All safe methods are idempotent; PUT and DELETE are idempotent but not safe; **POST and PATCH are neither safe nor idempotent.** GET and HEAD responses are cacheable by default; POST responses are cacheable only with explicit freshness info and a matching Content-Location (rare in practice).
- **Status-code classes (§15):** 1xx informational, 2xx successful, 3xx redirection, 4xx client error, 5xx server error.
- **Conditional requests (§13):** ETag/Last-Modified validators with If-Match, If-None-Match, If-Modified-Since, If-Unmodified-Since, and If-Range preconditions.
- **Content negotiation (§12):** proactive (server-driven) and reactive (agent-driven) negotiation via Accept, Accept-Language, Accept-Encoding; 406 Not Acceptable; Vary.

**Status.** Internet Standard. It **obsoletes RFCs 2818, 7231, 7232, 7233, 7235, 7538, 7615, 7694 and portions of 7230**, and updates RFC 3864. The change vs the RFC 7230–7235 family is a reorganization (semantics separated from HTTP/1.1 message syntax, now in RFC 9112) and clarification, not a behavioral change.

**Adoption.** Universal — all eight references operate over HTTP and use these method/status semantics.

### 3.2 RFC 9111 — HTTP Caching
**What it specifies.** RFC 9111 (June 2022, STD 98; obsoletes RFC 7234) defines caches and cache-control. API-relevant parts:
- **Freshness:** `response_is_fresh = (freshness_lifetime > current_age)`; freshness set via `Expires` or the `max-age` / `s-maxage` Cache-Control directives.
- **Cache-Control response directives:** max-age, s-maxage, no-cache, no-store, must-revalidate, proxy-revalidate, private, public, no-transform, plus immutable-family extensions.
- **Cache-Control request directives:** max-age, max-stale, min-fresh, no-cache, no-store, only-if-cached.
- **Validators:** ETag + conditional revalidation (Cache-Control governs *when* to revalidate; ETag governs *how*).

**Status.** Internet Standard.

**Adoption.** Underlying caching is universal, but explicit API-level Cache-Control tuning varies. GitHub, for instance, emits `Cache-Control: max-age=0, private, must-revalidate` plus ETags. Most JSON APIs mark responses private/no-store by default.

### 3.3 RFC 9457 (and predecessor RFC 7807) — Problem Details
**What it specifies.** A standard machine-readable error format with media type `application/problem+json` (and `application/problem+xml`). Members:
- `type` (URI reference identifying the problem type; defaults to `about:blank`)
- `title` (short human-readable summary of the type)
- `status` (HTTP status code, 100–599)
- `detail` (human-readable explanation of this occurrence)
- `instance` (URI reference identifying this specific occurrence)
- plus arbitrary **extension members**, which are namespaced by the problem type.

**What changed from 7807 → 9457.** Small but meaningful: (a) explicit support for **multiple problems** and clearer guidance on registering problem types; (b) a new **IANA problem type registry**; (c) clearer association between a problem `type` and the extension members a client should expect; (d) the `about:blank` type is more clearly defined. RFC 9457 (July 2023) obsoletes RFC 7807; 7807 implementations remain valid.

**Adoption survey (adoption fact only; error-shape internals are Part 3):**
- **Zalando:** adopts — guidelines mandate RFC 7807 Problem JSON (`application/problem+json`); Zalando authored the widely used `problem-spring-web` library.
- **Google Cloud/AIP:** proprietary — AIP-193 mandates `google.rpc.Status` (code/message/details[] with a required `ErrorInfo`). (Note: the newer AEP.dev fork references RFC 9457 and `application/problem+json`.)
- **Microsoft Graph:** proprietary — a top-level `error` object `{code, message, innererror, details[]}` per the Microsoft REST API Guidelines/OData; not problem+json. (Some individual Azure services, e.g., App Configuration, do return `application/problem+json`.)
- **Stripe:** proprietary — top-level `error` object with `type`/`code`/`message`/`param`.
- **GitHub:** proprietary — `{message, errors[], documentation_url}`.
- **Twilio:** proprietary — `{status, message, code, more_info}`.
- **Shopify:** proprietary — `errors` field (string or object/array keyed by field).
- **AWS (contrast):** proprietary — protocol-specific error shapes (`__type`/message, or XML Error).

Net: **1 of 8 (Zalando) mandates Problem Details; 7 of 8 use proprietary shapes.**

### 3.4 RFC 3339 — Timestamps
**What it specifies.** RFC 3339 (July 2002) is a **profile of ISO 8601** for internet timestamps: `2015-05-28T14:07:17Z`. It makes most fields/punctuation mandatory (full date + time, mandatory `T` separator, mandatory offset or `Z`), restricts the hour to 00–23, and requires a stated offset to UTC. RFC 3339 is a strict *subset* of ISO 8601 — all RFC 3339 timestamps are valid ISO 8601 but not vice versa (ISO 8601 also allows basic format without separators, week dates, ordinal dates, etc.). **Update:** RFC 9557 (2024) refines RFC 3339's interpretation of the `Z`/`-00:00` offset and adds calendar/timezone extensions.

**Adoption / deviations.**
- **RFC 3339/ISO 8601 is the norm:** GitHub ("All timestamps return in UTC time, ISO 8601 format: YYYY-MM-DDTHH:MM:SSZ"), Zalando (mandates RFC 3339, recommends `Z` over `+00:00`), Google/AIP (RFC 3339 with `Z`), Microsoft Graph, Shopify (ISO 8601 with offset, e.g., `2007-12-31T19:00:00-05:00`).
- **Twilio deviates:** returns dates in **RFC 2822** format (e.g., `Fri, 20 Aug 2010 01:13:42 +0000`) — flag for Part 3.
- **Epoch-seconds users (flag for Part 3):** Stripe uses Unix epoch-seconds integers for `created` and most timestamps; Google/AIP explicitly permit a `unix_time`-suffixed integer as a documented exception ("APIs are unable to use RFC 3339 strings for legacy or compatibility reasons... Unix timestamps should use a `unix_time` suffix").

### 3.5 RFC 8288 — Web Linking
**What it specifies.** RFC 8288 (Oct 2017; obsoletes RFC 5988) defines the `Link` header field and a model of typed relationships between resources, plus the IANA Link Relation Types registry. Standard pagination relations: `next`, `prev`(`previous`), `first`, `last`. Example: `Link: <...?page=3>; rel="next", <...?page=50>; rel="last"`. Also widely used: `self`, `alternate`, `canonical`, `describedby`.

**Adoption.**
- **Uses the Link header:** GitHub ("It's important to form calls with Link header values instead of constructing your own URLs"); Shopify Admin REST (cursor-based `Link` header with `rel="next"`/`"previous"`, introduced in API version 2019-07 replacing page-based pagination).
- **Body-based pagination instead:** Stripe (`has_more` + object IDs), Twilio (`next_page_uri`/`meta.next_page_url` in body), Google/AIP (`nextPageToken` in body), Microsoft Graph (`@odata.nextLink` in body).

### 3.6 RFC 7396 (JSON Merge Patch) vs RFC 6902 (JSON Patch)
**JSON Merge Patch (RFC 7396, Oct 2014), media type `application/merge-patch+json`.** A partial document that is recursively merged into the target: present keys overwrite, **`null` means delete the key**, absent keys are left unchanged. Consequence: it **cannot set a member to an explicit `null`** (null is overloaded as delete), and it **cannot patch individual array elements** (arrays are replaced wholesale). Simple and intuitive for object field updates.

**JSON Patch (RFC 6902, April 2013), media type `application/json-patch+json`.** An ordered array of operations — `add`, `remove`, `replace`, `move`, `copy`, `test` — applied atomically (all-or-nothing), using JSON Pointer (RFC 6901) paths. Expressive: can insert/remove array elements by index, set explicit nulls, and do conditional updates via `test`. More verbose and harder to author.

**Adoption reality.** Both are minority formats; most APIs use a proprietary "send the fields you want to change" PATCH without adopting either media type. Zalando's guidelines explicitly bless JSON Merge Patch semantics (the one place they permit `null`). JSON Patch appears more in config/Kubernetes-style and OpenAPI-tooling ecosystems than in the eight consumer-facing references.

### 3.7 RFC 8594 — Sunset header
**What it specifies.** RFC 8594 (May 2019) defines the `Sunset` HTTP response header, an **HTTP-date** (RFC 7231/IMF-fixdate format, e.g., `Sun, 30 Jun 2024 23:59:59 GMT`) indicating when a resource is expected to become unresponsive, plus a `sunset` link relation. Deprecation does not change resource behavior.

**Adoption.** Thin in production; more common in API-governance guidance than in the eight references' live responses. Typically paired with the Deprecation header.

### 3.8 IETF `httpapi` Working Group outputs — VERIFIED STATUS (run date July 19, 2026)
- **Deprecation header → RFC 9745 (published March 2025, Standards Track / Proposed Standard).** Field `Deprecation` is a Structured-Fields **Item** carrying an RFC 9651 Date in Unix time (e.g., `Deprecation: @1688169599`), which may be past or future; also defines the `deprecation` link relation. Note the format mismatch: `Deprecation` uses Unix-time seconds while the companion `Sunset` header uses HTTP-date "for historical reasons." Registered permanent in the HTTP Field Name Registry. Authors: S. Dalal, E. Wilde. **Maturity: published RFC.**
- **RateLimit header fields → draft-ietf-httpapi-ratelimit-headers-11 (published 23 May 2026, Standards Track, Internet-Draft; expires 24 Nov 2026).** Defines `RateLimit` and `RateLimit-Policy` structured fields for servers to advertise quota policy and current limits. The prior version (draft-10, Sept 2025) received an HTTP Directorate early review by Lucas Pardue on 2026-01-16 marked **"not ready"**: *"However, I'm marking this document as not ready... a document that seems to lack some precision of the core protocol elements, while containing some bloat that distracts."* Authors: Polli (Italian Government), Martinez (Red Hat), Miller (Microsoft). **Maturity: mature but still a draft, not an RFC.** Known implementers cited by the WG: Red Hat 3scale, Kong, Envoy, Azure API Gateway.
- **Idempotency-Key header → draft-ietf-httpapi-idempotency-key-header-07 (latest revision 15 Oct 2025; expired 18 April 2026; currently "Expired & archived" Internet-Draft in the httpapi WG).** Defines the `Idempotency-Key` request header (a Structured-Fields String, e.g., a UUID) to make POST/PATCH fault-tolerant; server responsibilities cover fingerprinting, expiry, and conflict (409) handling. Explicitly credits Stripe and PayPal patterns. Authors: J. Jena, S. Dalal. **Maturity: unfinished draft, currently expired; NOT an RFC.** Known adopters of the pattern (not necessarily the exact draft semantics): Stripe, PayPal, Adyen.

### 3.9 JSON:API 1.1
**What it standardizes.** JSON:API (media type `application/vnd.api+json`; v1.1 is the current spec) defines a complete envelope and interaction convention: a top-level document with `data` / `errors` / `meta`; typed **resource objects** (`type`, `id`, `attributes`, `relationships`, `links`); **compound documents** with `included` and "full linkage"; **sparse fieldsets** (`fields[TYPE]`); **`include`** for related-resource inclusion; sorting (`sort`), pagination (`page`), and a standardized **error object** array. It uses a "never remove, only add" compatibility strategy across versions.

**Adoption.** Widely cited and well-supported by frameworks (Ember Data, Drupal JSON:API, many server libs), but **selectively adopted** among large public APIs — none of the eight references uses JSON:API as its primary format. It is strongest in the Ruby/Ember/Drupal/PHP ecosystems and internal/enterprise APIs rather than the marquee commercial APIs surveyed here.

### 3.10 OpenAPI 3.1 + JSON Schema
**What the alignment is.** OpenAPI 3.1.0 (released Feb 18, 2021) achieves **100% compatibility with JSON Schema draft 2020-12**. OpenAPI 3.0 used a divergent "extended subset" of JSON Schema draft 4 (custom `nullable`, single-string `type`, no `if/then/else`, `patternProperties`, `prefixItems`, etc.). In 3.1 every Schema Object is a valid JSON Schema 2020-12 document; the default dialect is the OpenAPI 3.1 dialect (`https://spec.openapis.org/oas/3.1/dialect/base` = JSON Schema 2020-12 + OpenAPI vocabulary), overridable via `jsonSchemaDialect`/`$schema`. `nullable` is removed in favor of type arrays. 3.1 also made `paths` optional and added first-class `webhooks`.

**Who publishes official OpenAPI docs (standards-relationship fact only; docs practice is Part 6):**
- **Microsoft Graph:** yes — official OpenAPI (v1.0/beta, OpenAPI 3.0.4) in the `microsoftgraph/msgraph-metadata` repo, plus OData `$metadata`.
- **Twilio:** yes — official OpenAPI 3.0 specs (GA) in `twilio/twilio-oai`.
- **Zalando:** yes — guidelines are "API-first"/OpenAPI-centric; publishes the Problem schema in OpenAPI.
- **Google:** partial — proto/gRPC-first; OpenAPI generated for some surfaces.
- **Stripe:** yes — publishes an OpenAPI spec for its API (community-consumed via `stripe/openapi`).
- **GitHub:** yes — publishes an official OpenAPI 3.x description (`github/rest-api-description`).
- **AWS:** partial — Smithy/JSON models rather than canonical OpenAPI.
- **Shopify:** exception — no clearly official OpenAPI for Admin REST; Shopify steers developers to the Admin GraphQL schema.

### 3.11 Fielding's REST Dissertation + Richardson Maturity Model
**The constraints.** Roy Fielding's 2000 dissertation defines REST as an architectural style with constraints: client–server, statelessness, cacheability, uniform interface, layered system, and (optional) code-on-demand. The uniform-interface constraint includes **HATEOAS** (hypermedia as the engine of application state) — clients should navigate via server-provided links, not out-of-band knowledge. Fielding has publicly and forcefully maintained that hypermedia is a precondition for calling an API "REST": in his 2008 post "REST APIs must be hypertext-driven," he wrote, *"If the engine of application state (and hence the API) is not being driven by hypertext, then it cannot be RESTful and cannot be a REST API. Period."*

**The Richardson Maturity Model (Leonard Richardson, 2008; popularized by Martin Fowler).** Level 0: single URI, single verb (RPC/"swamp of POX"). Level 1: multiple resource URIs. Level 2: proper HTTP verbs + status codes. Level 3: hypermedia controls (HATEOAS).

**Adoption reality (first-class finding).** Across the eight references, the operating level is **Level 2**: resource-oriented URIs + HTTP verbs + status codes, with pagination/relationship links provided pragmatically (Link header or body) rather than as a discoverable hypermedia engine. None of the eight requires clients to navigate purely by hypermedia; documentation + fixed URL templates are the real contract. Secondary sources converge on this: DOTNET.REST's maturity-levels documentation states that *"In practice, Level 2 has become the de facto standard for what most developers consider 'RESTful' today,"* and that *"The highest level—Hypermedia Controls (HATEOAS)—is rarely seen in production APIs despite being theoretically 'pure REST.'"* The academic survey "RESTful or RESTless — Current State of Today's Top Web APIs" (arXiv 1902.10514) reaches the same conclusion for the top public APIs: *"concepts such as resource linking (HATEOAS) are hardly ever applied. For today's top Web APIs we... have to conclude that they most commonly remain RESTless."* Partial hypermedia does appear (Stripe's `url`/`has_more` list fields; GitHub's `_links` and hypermedia Link relations; AIP's `nextPageToken`), but as convenience, not as a HATEOAS contract. This is the clearest standard-vs-practice gap in the entire foundational layer.

## 4. Adoption Matrix
Rows = standards; columns = the eight references. Cells: **A** = adopts, **P** = partial, **I** = ignores/uses alternative.

| Standard | Stripe | GitHub | Google/AIP | MS (Graph/Azure) | Twilio | Shopify | Zalando | AWS |
|---|---|---|---|---|---|---|---|---|
| RFC 9110/9111 (HTTP semantics/caching) | A | A (ETags, Cache-Control) | A | A | A | A | A | A |
| RFC 9457/7807 Problem Details | I (proprietary `error`) | I (`message`/`errors`) | I (`google.rpc.Status`) | I (Graph `error`; some Azure use problem+json) | I (`status/message/code/more_info`) | I (`errors`) | **A** (mandated) | I (protocol-specific) |
| RFC 3339 timestamps | P (epoch-seconds) | A (ISO 8601 Z) | A (RFC 3339; epoch permitted) | A (ISO 8601) | **I** (RFC 2822) | A (ISO 8601) | A (RFC 3339, prefers Z) | P (varies by service) |
| RFC 8288 Link header | I (body) | **A** | I (body token) | I (`@odata.nextLink`) | I (`next_page_uri`) | **A** (since 2019-07) | P (recommends both) | I (body tokens) |
| RFC 7396 Merge Patch / 6902 JSON Patch | I (proprietary PATCH) | I (proprietary PATCH) | P (field masks) | P (Graph PATCH/merge-ish) | I | I | **A** (Merge Patch blessed) | I |
| RFC 8594 Sunset | I | I | P (guidance) | P (guidance) | I | I | P (guidance) | I |
| RFC 9745 Deprecation | I | P (uses `Sunset`/custom) | P (guidance) | P (guidance) | I | P (version deprecation via docs) | P (guidance) | I |
| RateLimit draft (IETF) | I (proprietary) | I (`x-ratelimit-*`) | I | P (Retry-After; ARM `x-ms-ratelimit-*`) | I (429 only) | I (`X-Shopify-Shop-Api-Call-Limit`) | P (guidance) | I |
| Idempotency-Key draft | **A** (originator) | P (some endpoints) | P | P | P | P | A (references Stripe) | P (per-service) |
| JSON:API 1.1 | I | I | I | I | I | I | I | I |
| OpenAPI 3.1 published | A (spec) | A | P (some) | A | A | I (GraphQL) | A | P (Smithy) |
| HATEOAS / RMM Level 3 | Level 2 | Level 2 (partial links) | Level 2 | Level 2 | Level 2 | Level 2 | Level 2 | Level 2 |

*Cell nuance notes: "epoch-seconds" for Stripe means integer Unix time in bodies; Google "epoch permitted" is a documented exception; Microsoft `x-ms-ratelimit-*` are proprietary Azure Resource Manager headers; Idempotency-Key "P" indicates the pattern is supported on some POST endpoints but not standardized across the API. Twilio's rate-limit column is "I" because only a 429 status (not a named RateLimit header) is documented officially.*

## 5. Where the Standards Layer Is Settled vs Contested (in Practice)

**Settled / near-consensus:**
- HTTP method semantics, status-code classes, and caching primitives (RFC 9110/9111) — universal.
- RFC 3339/ISO 8601 timestamps as the default representation — near-universal, with Twilio (RFC 2822) and epoch-seconds users (Stripe; Google's documented exception) as the recognized exceptions.
- JSON as the default representation; OpenAPI as the default description language (with JSON Schema 2020-12 alignment now the norm in 3.1).
- Idempotency via a client-supplied key on unsafe methods — conceptually settled even though the IETF text is not.

**Contested in practice:**
- Error format: Problem Details vs proprietary shapes (7/8 proprietary).
- Pagination transport: Link header vs body cursors.
- Partial update: Merge Patch vs JSON Patch vs proprietary PATCH.
- Deprecation signaling: RFC 9745/8594 headers vs docs-only vs custom headers.
- Rate-limit signaling: no standard adopted; every API rolls its own header names.
- REST maturity: Level 2 pragmatism vs Level 3 HATEOAS purity (effectively resolved *against* HATEOAS in practice).

## 6. Contested Axes Register (scoped to Part 1)

| Axis | Options observed | Who does what | Tradeoff (one line) | How contested |
|---|---|---|---|---|
| **Error format** | (a) RFC 9457/7807 problem+json; (b) proprietary JSON error object | Zalando → 9457/7807; Stripe, GitHub, Google, MS Graph, Twilio, Shopify, AWS → proprietary | Standard buys cross-API tooling/interop; proprietary buys freedom to model domain-specific fields | **Split** (leans proprietary, ~1/8 standard) |
| **Pagination transport** | (a) RFC 8288 Link header; (b) body-embedded cursors/tokens | GitHub, Shopify → Link header; Stripe, Twilio, Google, MS Graph → body | Header keeps body pure and is discoverable; body cursors are easier for clients that ignore headers | **Split** |
| **Partial update semantics** | (a) Merge Patch (7396); (b) JSON Patch (6902); (c) proprietary PATCH | Zalando → Merge Patch; most others → proprietary field PATCH; JSON Patch mostly in config/k8s ecosystems | Merge Patch is simple but can't express explicit-null or array-element edits; JSON Patch is expressive but verbose | **Wide-open** |
| **Deprecation signaling** | (a) RFC 9745 Deprecation + (b) RFC 8594 Sunset headers; (c) docs/changelog only; (d) custom headers | Governance guidelines (Zalando, Google, MS) reference headers; most references signal via docs/versioning | Runtime headers enable client tooling/auto-warnings; docs-only is simpler but invisible to code | **Wide-open** (headers newly standardized, thin adoption) |
| **Rate-limit signaling** | (a) IETF RateLimit draft fields; (b) proprietary `X-RateLimit-*`/vendor headers; (c) 429 + Retry-After only | Nobody ships the IETF draft names; GitHub `x-ratelimit-*`, Shopify `X-Shopify-Shop-Api-Call-Limit`, MS Retry-After/ARM `x-ms-ratelimit-*`, Twilio 429-only | Standard names would enable generic client backoff; proprietary names entrench per-API client code | **Wide-open** (standard still a draft) |
| **Idempotency mechanism** | (a) `Idempotency-Key` header (IETF draft); (b) none/verb-idempotency only | Stripe (originator), PayPal, Adyen use the header; many APIs rely only on PUT/DELETE idempotency | Header makes POST retriable safely; adds server-side key storage/fingerprinting burden | **Near-consensus on the pattern**, but spec unfinished |
| **REST maturity / HATEOAS** | (a) Level 2 (verbs+resources); (b) Level 3 (hypermedia) | All eight operate at Level 2; none mandates Level 3 hypermedia navigation | Level 2 is simpler and documentation-driven; Level 3 promises evolvability but adds client/server complexity rarely used | **Near-consensus against Level 3** in practice |

## 7. Caveats
- **Drafts in flux:** The RateLimit fields (draft-11) and Idempotency-Key (draft-07) are Internet-Drafts, not RFCs; the Idempotency-Key draft is currently **expired** (18 April 2026) and could change or lapse. Cite them as "work in progress." The Deprecation header, by contrast, is now RFC 9745 (verified published March 2025).
- **Adoption cells reflect default/primary behavior**, not every endpoint. Large APIs are internally inconsistent (e.g., Stripe's v1 vs v2 namespaces differ on pagination and idempotency scope; "Microsoft"/"Azure" is not monolithic — Graph is proprietary on errors while some Azure services use problem+json).
- **Error-shape internals, epoch-seconds field details, pagination cursor mechanics, and deprecation/versioning workflow are deliberately deferred** to Parts 2–7 per scope; this part records only the adoption fact.
- **Secondary sources** were used for RMM adoption characterization (DOTNET.REST "de facto Level 2"; arXiv 1902.10514 "RESTless") and corroborate but do not originate the finding; the per-API level assessments are inferred from each API's documented contract.
- **Shopify OpenAPI publication is unconfirmed** as official; treat the "I" cell as "not clearly published" rather than definitively absent.
- **RFC 3339 update:** RFC 9557 (2024) updates RFC 3339's `Z`/offset interpretation; this does not change the base profile most APIs implement.
- **Twilio rate-limit headers:** only a 429 status is documented officially; no named `RateLimit`/`X-RateLimit` response header was confirmed in Twilio's docs — reported as "not documented" rather than asserting a header name.