# Part 1 — Foundations & Standards Layer for HTTP/REST APIs
*Descriptive survey. All facts retrieved 20 July 2026. Descriptive-only: this report documents what the standards say and what adoption looks like; it makes no prescriptive recommendations for the later standard-setting phase.*

## TL;DR
- **The foundational standards layer is stable and well-defined.** RFC 9110/9111 (HTTP semantics + caching, STD 97/98, June 2022) supersede the RFC 7230–7235 family; RFC 9457 (Problem Details, 2023) obsoletes 7807; RFC 3339 (timestamps), RFC 8288 (Web Linking), RFC 7396/6902 (patch formats), RFC 8594 (Sunset), and RFC 9745 (Deprecation, March 2025) are all published. Only two IETF `httpapi` items are still drafts: RateLimit fields (draft-11, May 2026) and Idempotency-Key (draft-07, expired April 2026).
- **Adoption is deeply uneven.** The transport-layer baseline (methods, status codes, RFC 3339 strings, OpenAPI 3.1) is near-universal, but the "opinionated" IETF conventions (Problem Details, Link-header pagination, Sunset/Deprecation, Merge/JSON Patch) are adopted by a minority of the reference set — most large vendors ship proprietary shapes.
- **HATEOAS is effectively dead in practice.** All eight reference APIs operate at Richardson Level 2; none at Level 3. "True REST" per Fielding is aspirational, not descriptive of the field.

## Key Findings
1. **The core HTTP spec was re-modularized in June 2022.** RFC 9110 (HTTP Semantics, STD 97) and RFC 9111 (HTTP Caching, STD 98) consolidate and obsolete the six-document RFC 7230–7235 family (with RFC 9112 covering HTTP/1.1). The method-properties table (safe/idempotent/cacheable) and status-code classes are normative and substantively unchanged.
2. **RFC 9457 (2023) obsoletes RFC 7807 with only three substantive changes** (Appendix D): a new IANA "HTTP Problem Types" registry, clarified handling of multiple problems, and guidance for non-dereferenceable `type` URIs. The wire format is unchanged.
3. **Problem Details adoption among the eight is a minority** — Zalando mandates it; Stripe, GitHub, Google, Microsoft, Twilio, Shopify, and AWS each ship a proprietary error shape.
4. **RFC 3339 is near-universal for timestamp *strings*, but Stripe deviates**, using integer Unix epoch seconds for `created` and similar fields.
5. **The Link header (RFC 8288) is the standards-native pagination mechanism, but only GitHub and Shopify Admin use it prominently** among the eight; Stripe, Google, Microsoft, and Twilio use body-envelope cursors.
6. **Merge Patch (RFC 7396) and JSON Patch (RFC 6902) are both standardized but sparsely adopted** for their canonical media types; most references use custom PATCH or PUT/POST semantics.
7. **The Deprecation header became RFC 9745 (Standards Track, March 2025)** during this cycle; Sunset (RFC 8594) remains Informational. Runtime adoption of both is low.
8. **RateLimit fields and Idempotency-Key remain IETF drafts**, while de-facto equivalents (`X-RateLimit-*`, `Idempotency-Key`) are widely deployed ahead of standardization.
9. **OpenAPI 3.1 (2021) achieves 100% JSON Schema 2020-12 alignment.** Publishing official OpenAPI docs is common but not universal among the eight.
10. **All eight reference APIs sit at Richardson Level 2**; HATEOAS (Level 3) is essentially unimplemented.

---

## Per-Standard Sections

### 1. RFC 9110 — HTTP Semantics (STD 97, June 2022; Fielding, Nottingham, Reschke)
**Status:** Internet Standard. Obsoletes RFC 7231 (semantics), 7232 (conditional), 7233 (range), 7235 (auth), and parts of 7230. All three HTTP versions (1.1, /2, /3) rely on 9110's semantics.

**Method-properties table (§9.2/9.3), verbatim from the RFC registry:**

| Method | Safe | Idempotent | Cacheable |
|---|---|---|---|
| CONNECT | no | no | no |
| DELETE | no | yes | no |
| GET | yes | yes | yes |
| HEAD | yes | yes | yes |
| OPTIONS | yes | yes | no |
| POST | no | no | only with explicit freshness + `Content-Location` |
| PUT | no | yes | no |
| TRACE | yes | yes | no |

Safe = "essentially read-only." PATCH (defined in RFC 5789, not in 9110's table) is **neither** safe nor idempotent. **Status-code classes (§15):** 1xx informational, 2xx successful, 3xx redirection, 4xx client error, 5xx server error. **Conditional requests:** ETag/Last-Modified validators with `If-Match`, `If-None-Match`, `If-Modified-Since`, `If-Unmodified-Since`, `If-Range`; responses `304 Not Modified`, `412 Precondition Failed`. **Content negotiation (§12):** proactive (server-driven, via `Accept`/`Accept-Language`/`Accept-Encoding` with q-values) and reactive; `406 Not Acceptable`; `Vary` lists request headers influencing selection.

**Change vs 7230–7235:** primarily editorial consolidation and clarification; no wholesale semantic changes to methods or status classes. (An open erratum notes "cacheable" is arguably a system capability, not a pure method property.)

### 2. RFC 9111 — HTTP Caching (STD 98, June 2022)
**Status:** Internet Standard; obsoletes RFC 7234.
**Response directives (§5.2.2):** `max-age`, `s-maxage`, `no-cache`, `no-store`, `no-transform`, `must-revalidate`, `proxy-revalidate`, `must-understand` (new in 9111), `private`, `public`. **Request directives (§5.2.1):** `max-age`, `max-stale`, `min-fresh`, `no-cache`, `no-store`, `no-transform`, `only-if-cached`. **Freshness (§4.2):** lifetime computed from `s-maxage` > `max-age` > `Expires` > heuristic; `Age` header; stale-serving rules. **Validation (§4.3):** conditional revalidation using ETag/Last-Modified. **Vary (§4.1)** partitions the cache key. The `Warning` header is **obsoleted** by 9111.

### 3. RFC 9457 / RFC 7807 — Problem Details for HTTP APIs
**Status:** RFC 9457 (2023; Nottingham, Wilde, Dalal) is an Internet Standard obsoleting RFC 7807 (2016). **Media types:** `application/problem+json` and `application/problem+xml`.

**Format:** a JSON object with `type` (URI reference; defaults to `about:blank` when absent), `title` (human-readable summary), `status` (HTTP code, 100–599), `detail` (occurrence-specific), `instance` (URI reference for the occurrence), plus arbitrary extension members. Verbatim example:
```
HTTP/1.1 403 Forbidden
Content-Type: application/problem+json
{ "type": "https://example.com/probs/out-of-credit",
  "title": "You do not have enough credit.",
  "detail": "Your current balance is 30, but that costs 50.",
  "instance": "/account/12345/msgs/abc",
  "balance": 30, "accounts": ["/account/12345","/account/67890"] }
```
**Changes from 7807 (Appendix D — exactly three, verbatim):** (1) "Section 4.2 introduces a registry of common problem type URIs"; (2) "Section 3 clarifies how multiple problems should be treated"; (3) "Section 3.1.1 provides guidance for using type URIs that cannot be dereferenced." The `about:blank` default and optional `type` are **inherited from 7807**, not new. The **IANA "HTTP Problem Types" registry** (Specification Required policy, experts Nottingham & Dalal) currently holds only three entries: `about:blank` (registered by 9457), plus `https://iana.org/assignments/http-problem-types#date` (Date Not Acceptable, 400) and `#ohttp-key` (400) — both registered via **RFC 9458 (Oblivious HTTP)**, not 9457.

**Adoption survey** (adoption fact only; error-shape internals deferred to Part 3):
- **Zalando:** adopts (mandates `application/problem+json`).
- **Stripe:** proprietary — `error.type` ∈ {`api_error`, `card_error`, `invalid_request_error`}, `error.code`, `decline_code`.
- **Google/AIP:** proprietary — `google.rpc.Status` shape per AIP-193 (`{"error":{"code":<HTTP-status-int>,"message","status":<enum>,"details":[]}}`).
- **Microsoft (Graph/Azure):** proprietary — OData-derived `{"error":{"code","message","innererror","details":[]}}`; does **not** use Problem Details (though it does not name-check or explicitly reject the RFC).
- **Twilio:** proprietary — `code`, `message`, `more_info`, `status`.
- **GitHub:** proprietary — `{message, documentation_url, errors[]}`.
- **Shopify Admin REST:** proprietary — `{errors: ...}` (string or field-keyed object).
- **AWS:** proprietary — protocol-specific shapes (JSON `__type`/`Code`+`Message`); never Problem Details.

### 4. RFC 3339 — Timestamps (2002; Standards Track)
A conformant subset ("profile") of ISO 8601 for the Gregorian calendar. Requires complete date+time; **mandates a time offset** (`Z` or `±hh:mm`); restricts hour to 00–23 (ISO allows 24); makes most punctuation mandatory. **Notable divergences from ISO 8601:** RFC 3339 permits `-00:00` (connoting an *unknown* preferred offset), which ISO 8601 forbids; and RFC 3339 tolerates a space instead of `T` as separator, which ISO 8601 does not. Example: `2007-04-05T14:30-02:00`. Fractional seconds allowed but not required. (RFC 9557 later extended this with IXDTF timezone names — outside this part's scope.)

**Field deviation:** Stripe uses integer Unix epoch seconds ("Time at which the object was created. Measured in seconds since the Unix epoch," e.g., `"created": 1662738023`) rather than RFC 3339 strings — **flagged for Part 3.** GitHub returns UTC ISO 8601/RFC 3339 strings (e.g., `2014-02-27T15:05:06+01:00`). Google AIP uses `google.protobuf.Timestamp` (RFC 3339 on the JSON wire). Most others use RFC 3339 strings.

### 5. RFC 8288 — Web Linking (2017; Standards Track, obsoletes RFC 5988)
Defines the `Link` header and the IANA Link Relation Types registry. **Syntax:** target IRI in angle brackets, then `;`-separated params; `rel` carries registered relations or absolute-URI extension relations. Registered relations include `next`, `prev`/`previous`, `first`, `last`, `self`, `canonical`, `alternate`, `edit`, `enclosure`. Verbatim example:
```
Link: </TheBook/chapter2>; rel="previous"; title*=UTF-8'de'letztes Kapitel,
      </TheBook/chapter4>; rel="next"; title*=UTF-8'de'nächstes Kapitel
```
**Adoption:** GitHub is the canonical adopter (`Link` header with `rel=next/last`, page/before/after cursors); Shopify Admin REST uses it with `rel=next/previous` and an opaque `page_info` cursor. Stripe, Google, Microsoft, and Twilio use body-envelope pagination (Stripe: `has_more` + `starting_after`/`ending_before`; Google: `nextPageToken`; Twilio: `next_page_uri` in body). Zalando's guidelines describe body pagination objects (`self`/`first`/`prev`/`next`/`last`) and lean toward opaque cursors over the header.

### 6. RFC 7396 (JSON Merge Patch) vs RFC 6902 (JSON Patch) — both Standards Track
**Merge Patch (7396, 2014; obsoletes 7386):** media type `application/merge-patch+json`; a partial JSON object where present keys overwrite, **`null` means delete the key**, absent keys are untouched; recursive on nested objects; arrays are replaced wholesale. Key limitation: cannot set a value *to* null (null always deletes) and cannot express array-element operations. Example: `{"title":"Hello!","author":{"familyName":null},"phoneNumber":"+01-123-456-7890"}` (deletes `familyName`).

**JSON Patch (6902):** media type `application/json-patch+json`; an ordered array of operations applied **atomically**. Six ops — `add`, `remove`, `replace`, `move`, `copy`, `test` — with paths in JSON Pointer (RFC 6901). Example: `[{"op":"replace","path":"/baz","value":"boo"},{"op":"add","path":"/hello","value":["world"]},{"op":"remove","path":"/foo"}]`.

**Adoption:** Zalando's guidelines explicitly recommend **both** (Merge Patch for simple field updates; JSON Patch when array-index changes are needed), noting Merge Patch "quickly turns out to be too limited." Most other references use custom PATCH or PUT/POST semantics rather than the standard media types (detail deferred to Parts 3/5).

### 7. RFC 8594 — The Sunset HTTP Header Field (May 2019; **Informational**)
`Sunset` carries a single HTTP-date timestamp for when a resource is expected to become unresponsive; treated as a hint. Also defines a `sunset` link relation. Example: `Sunset: Sat, 31 Dec 2018 23:59:59 GMT`. It applies to stage-2 decommissioning, not stage-1 "no longer preferred." Adoption is low; Zalando recommends `Sunset` (paired with `Deprecation`) but discourages the `sunset`/`deprecation` link relations.

### 8. IETF `httpapi` Working Group Outputs (status verified 20 July 2026)
- **Deprecation header → NOW AN RFC.** **RFC 9745** (Standards Track / Proposed Standard, March 2025; Dalal, Wilde). `Deprecation` is an **Item Structured Header** whose value **MUST be a Date** per RFC 9651 §3.3.7 (an `@`-prefixed Unix time). Example: `Deprecation: @1688169599`, typically paired with `Sunset: Sun, 30 Jun 2024 23:59:59 UTC` (the two use **different** date formats). Registers the `deprecation` link relation. Adopters: Zalando mandates it; broad runtime adoption remains low.
- **RateLimit fields → DRAFT.** `draft-ietf-httpapi-ratelimit-headers-11`, published 23 May 2026, Intended Status Standards Track, expires 24 November 2026 (Polli/Martinez/Miller). Defines `RateLimit` and `RateLimit-Policy` (Structured Fields). Example (draft-11 sliding window): `RateLimit-Policy: "sliding";q=12;w=1` and `RateLimit: "sliding";q=12;r=1;t=1`. Earlier drafts used discrete `RateLimit-Limit`/`RateLimit-Remaining`/`RateLimit-Reset`. An early HTTP-directorate review (16 Jan 2026) marked -10 **"not ready"** pending an editorial pass. De-facto predecessors (`X-RateLimit-*`, `X-Shopify-Shop-Api-Call-Limit`) are widely deployed.
- **Idempotency-Key header → EXPIRED DRAFT.** `draft-ietf-httpapi-idempotency-key-header-07`, published 15 October 2025, **expired 18 April 2026**; WG document, no -08 as of July 2026. Value is an **Item Structured Header String** (`sf-string` per RFC 8941 §3.3.3); UUID recommended. Example: `Idempotency-Key: "8e03978e-40d5-43e8-bc93-6894a57f9324"`. De-facto adoption is strong ahead of the standard (Stripe; the draft cites PayPal and Stripe as inspiration).

### 9. JSON:API 1.1 (community spec, jsonapi.org; media type `application/vnd.api+json`)
Standardizes: a top-level document envelope (`data` / `errors` / `meta`, plus optional `jsonapi`, `links`, `included`); resource objects (`type` + `id` + `attributes` + `relationships` + `links`); compound documents requiring **"full linkage"** via `included[]`; **sparse fieldsets** (`fields[TYPE]=a,b`); **inclusion** of related resources (`include=author,comments.author`); sorting (`sort=`), pagination (`page[...]`), filtering (`filter`); and an **`errors[]` array** of error objects (`id`, `status`, `code`, `title`, `detail`, `source.pointer`, `links`, `meta`). **Adoption reality:** widely cited and influential, **selectively adopted** — none of the eight reference APIs is a native JSON:API implementation; it is most common in framework ecosystems (Drupal, Ember, Rails `active_model_serializers`). Its `errors[]` array shape competes directly with RFC 9457's single-object shape.

### 10. OpenAPI 3.1 + JSON Schema
OpenAPI 3.1.0 released **18 Feb 2021**; its headline change is **100% alignment with JSON Schema draft 2020-12** (3.0 used a divergent JSON-Schema-draft-04-derived subset). The 3.1 Schema Object is a JSON Schema *vocabulary* extending Core + Validation; default dialect is `https://spec.openapis.org/oas/3.1/dialect/base` (= 2020-12 + OAS vocab), overridable via `jsonSchemaDialect` and per-schema `$schema`. Practical effects: `type` can be an array (`type: [string, "null"]` replaces `nullable`), a top-level `webhooks` object, and optional `paths`. **Standards-relationship / publication among the eight** (docs-practice detail deferred to Part 6): GitHub publishes an official OpenAPI description; Twilio and Stripe publish OpenAPI specs; Microsoft Graph exposes OpenAPI/CSDL metadata; Google AIP is proto/gRPC-first (OpenAPI generated/secondary); Zalando mandates OpenAPI for its teams; AWS uses its own Smithy/JSON model rather than publishing OpenAPI as the primary contract.

### 11. Fielding's REST Dissertation + Richardson Maturity Model
Fielding's 2000 dissertation defines REST via constraints: client–server, stateless, cacheable, uniform interface, layered system, and (optional) code-on-demand; the uniform-interface sub-constraints include **HATEOAS** ("hypermedia as the engine of application state"). Fielding's 2008 position is categorical — verbatim: *"If the engine of application state (and hence the API) is not being driven by hypertext, then it cannot be RESTful and cannot be a REST API. Period."* **Richardson Maturity Model:** Level 0 (single URI, single verb — "swamp of POX"/RPC); Level 1 (multiple resource URIs, still one verb); Level 2 (correct HTTP verbs + status codes — "what most people call REST"); Level 3 (hypermedia controls / HATEOAS — "true REST").

**Adoption reality (first-class deliverable):** all eight reference APIs operate at **Level 2**. Stripe, GitHub, Google, Microsoft, Twilio, Shopify, and AWS use resource URIs with correct verbs and status codes but do not drive application state through hypermedia. GitHub is the closest to a Level-3 *flavor* (it emits `Link` headers and some hypermedia URLs in bodies), but clients are not expected to navigate purely via discovered links, so it is best characterized as **Level 2 with hypermedia affordances**. No reference API requires HATEOAS-style discovery. **Conclusion: "true REST"/Level 3 is essentially absent from major industry practice; Level 2 is the de-facto standard.**

---

## Adoption Matrix
Cells: **A** = adopts · **P** = partial · **I** = ignores. Columns: Stripe · GitHub · Google/AIP · Microsoft · Twilio · Shopify · Zalando · AWS.

| Standard | Stripe | GitHub | Google | Microsoft | Twilio | Shopify | Zalando | AWS |
|---|---|---|---|---|---|---|---|---|
| HTTP methods/status (9110) | A | A | A | A | A | A | A | A |
| Conditional/ETag (9110) | P | A (304 uncounted vs limit) | P (etag) | P | I | P | P | P |
| Caching (9111) | I | P (cache-control/etag) | I | I | I | I | P | P |
| Problem Details (9457) | I | I | I | I | I | I | **A** | I |
| RFC 3339 timestamps | **I** (epoch secs) | A | A | A | A | A | A | P (varies) |
| Web Linking / Link (8288) | I | **A** | I | I | I | **A** (page_info) | P (body objs) | I |
| Merge Patch (7396) | I | I | I | I | I | I | **A** | I |
| JSON Patch (6902) | I | I | I | P (Graph) | I | I | **A** | I |
| Sunset (8594) | I | I | I | I | I | I | **A** | I |
| Deprecation (9745) | I | I | I | I | I | I | **A** | I |
| RateLimit draft | I (custom) | P (`X-RateLimit-*`) | I | I | P (custom) | P (`X-Shopify…`) | I | P (custom) |
| Idempotency-Key draft | **A** (de-facto) | I | I | I | I | I | P (patterns) | I |
| JSON:API 1.1 | I | I | I | I | I | I | I | I |
| OpenAPI 3.1 | A | A | P (proto-first) | P | A | P | A (mandated) | I (Smithy) |
| HATEOAS / Level 3 | I | I | I | I | I | I | I | I |

---

## Where the Standards Layer Is Settled vs Contested in Practice
**Settled (near-consensus):** HTTP method semantics and status-code classes (RFC 9110); RFC 3339 as the timestamp *string* format when a string is used; OpenAPI 3.1 + JSON Schema 2020-12 as the description standard; Level-2 REST as the operating point.

**Contested (standard exists but practice splits):** error format (Problem Details vs proprietary — proprietary dominates); pagination transport (Link header vs body envelope/cursor — split); partial update (Merge Patch vs JSON Patch vs custom — mostly custom); deprecation/sunset signaling (RFC 9745/8594 vs proprietary/out-of-band — mostly proprietary); rate-limit signaling (draft RateLimit vs `X-RateLimit-*` — de-facto headers dominate); idempotency (draft Idempotency-Key vs de-facto header — de-facto leads on the header name); HATEOAS (Fielding Level 3 vs Level-2 pragmatism — pragmatism wins decisively).

---

## Contested Axes Register (scoped to this part)

| Axis | Options observed | Who does what | Tradeoff (one line) | How contested |
|---|---|---|---|---|
| **Error format** | RFC 9457 `problem+json` vs proprietary JSON envelopes | Zalando=Problem Details; Stripe/Google/Microsoft/Twilio/GitHub/Shopify/AWS=proprietary | Standard shape → generic tooling interop; proprietary → domain-fit + legacy stability | **Split, proprietary-dominant** |
| **Pagination transport** | RFC 8288 Link header vs body-envelope cursors/tokens | GitHub/Shopify=Link header; Stripe/Google/Microsoft/Twilio=body cursor; Zalando=body objects | Link header is standards-native + cache/tooling-friendly; body cursors self-describe for JSON-only clients | **Split** |
| **Partial update** | Merge Patch (7396) vs JSON Patch (6902) vs custom PATCH/PUT | Zalando=both standards; most=custom | Merge Patch simple but can't null/array-edit; JSON Patch expressive but verbose; custom convenient but non-portable | **Wide-open** |
| **Timestamp format** | RFC 3339 strings vs Unix epoch integers | Most=RFC 3339 strings; Stripe=epoch seconds | Strings human-readable/self-describing; epoch integers compact/unambiguous but opaque | **Near-consensus for strings; notable holdout** |
| **Deprecation/Sunset signaling** | RFC 9745 + RFC 8594 headers vs proprietary/out-of-band | Zalando=headers; most=docs/email/versioning | Runtime headers → client tooling; out-of-band simpler but not machine-actionable | **Wide-open (low adoption)** |
| **Rate-limit signaling** | Draft `RateLimit`/`RateLimit-Policy` vs `X-RateLimit-*` de-facto | None use draft syntax; GitHub/Shopify/Twilio use proprietary | Standard eases client throttling libs; de-facto headers already work | **Split, de-facto-dominant; standard not final** |
| **Idempotency** | Draft `Idempotency-Key` vs de-facto `Idempotency-Key` vs none | Stripe=de-facto header; draft mirrors it; many APIs=none | Header-based idempotency → safe retries; adds server-side storage/complexity | **Near-consensus on header name; standard still a draft** |
| **REST maturity** | Fielding Level 3/HATEOAS vs Level-2 pragmatism | All eight=Level 2; none=Level 3 | Hypermedia decouples clients from URLs but adds complexity few clients exploit | **Near-consensus for Level 2; Level 3 rejected in practice** |

---

## Examples Appendix (verbatim payloads, header lines, concrete numbers — grouped by standard)

**RFC 9110 method table** (§9, retrieved 2026-07-20): GET yes/yes/yes; HEAD yes/yes/yes; OPTIONS yes/yes/no; TRACE yes/yes/no; PUT no/yes/no; DELETE no/yes/no; POST no/no/(cacheable only with explicit freshness + `Content-Location`); CONNECT no/no/no.

**RFC 9111 directives** (§5.2): response = `max-age`, `s-maxage`, `no-cache`, `no-store`, `no-transform`, `must-revalidate`, `proxy-revalidate`, `must-understand`, `private`, `public`; request = `max-age`, `max-stale`, `min-fresh`, `no-cache`, `no-store`, `no-transform`, `only-if-cached`.

**RFC 9457** `application/problem+json` (403 out-of-credit): see §3 body above. IANA registry entries: `about:blank`; `https://iana.org/assignments/http-problem-types#date` (400); `#ohttp-key` (400).

**RFC 3339:** `2007-04-05T14:30-02:00`; deviation `-00:00` allowed (ISO 8601 forbids). Stripe: `"created": 1662738023` (integer epoch seconds; Stripe docs: "Measured in seconds since the Unix epoch").

**RFC 8288 Link header:**
```
Link: </TheBook/chapter2>; rel="previous", </TheBook/chapter4>; rel="next"
```
GitHub: `Link: <https://api.github.com/user/repos?page=3&per_page=100>; rel="next", <...?page=50&per_page=100>; rel="last"`.

**GitHub rate-limit headers:** `x-ratelimit-limit: 5000`, `x-ratelimit-remaining: 4998`, `x-ratelimit-reset: 1666023299` (UTC epoch seconds). Per GitHub Docs "Rate limits for the REST API": *"The primary rate limit for unauthenticated requests is 60 requests per hour… your personal rate limit of 5,000 requests per hour"* (Enterprise Cloud apps get 15,000/hr).

**Shopify rate limits:** header `X-Shopify-Shop-Api-Call-Limit`; per Shopify Dev docs, standard plan has a *"bucket size of 40 requests per app, per store"* with a leaky-bucket *"leak rate of two requests per second"*; Shopify Plus bucket is **400**. Exceeding returns *"a 429 Too Many Requests error and a Retry-After header."*

**RFC 7396 Merge Patch:** `{"title":"Hello!","author":{"familyName":null}}` (deletes `familyName`). **RFC 6902 JSON Patch:** `[{"op":"replace","path":"/baz","value":"boo"},{"op":"remove","path":"/foo"}]`.

**RFC 9745 Deprecation:** `Deprecation: @1688169599` with `Sunset: Sun, 30 Jun 2024 23:59:59 UTC`. **RFC 8594 Sunset:** `Sunset: Sat, 31 Dec 2018 23:59:59 GMT`.

**RateLimit draft-11:** `RateLimit-Policy: "sliding";q=12;w=1` / `RateLimit: "sliding";q=12;r=1;t=1`.

**Idempotency-Key draft-07:** `Idempotency-Key: "8e03978e-40d5-43e8-bc93-6894a57f9324"` (`sf-string`). **Stripe:** `-H "Idempotency-Key: AGJ6FJMkGQIpHUTX"` on POST. Per Stripe's "Idempotent requests" reference: *"Idempotency keys are up to 255 characters long… we suggest using V4 UUIDs,"* and keys may be removed *"after they're at least 24 hours old."* Per Stripe's "API v2 overview": replays match *"API v1 if they use the same idempotency key and occur within 24 hours of each other. API v2 if they use the same idempotency key, are made to the same API, occur within the scope of the same account or sandbox, and occur within 30 days of each other."*

**Twilio error fields:** `code`, `message`, `more_info`, `status` (e.g., Code `21201`, `more_info` → docs/errors/21201).

**Google AIP-193 error:** `{ "error": { "code": <http-status-int>, "message": ..., "status": <enum>, "details": [...] } }`.

**Microsoft error:** `{ "error": { "code": "badRequest", "message": "…", "target": "query", "innererror": { "code": "requiredFieldMissing" } } }`.

**Stripe error:** `error.type` ∈ {`api_error`, `card_error`, `invalid_request_error`}, `error.code`, `decline_code`.

**JSON:API 1.1:** media type `application/vnd.api+json`; sparse fieldsets `fields[articles]=title,body`; `include=author`; `errors[]` with `source.pointer`.

**OpenAPI 3.1 dialect id:** `https://spec.openapis.org/oas/3.1/dialect/base` (JSON Schema 2020-12 + OAS vocab); `nullable` replaced by `type: [string, "null"]`.

---

## Caveats
- **Drafts in flux:** RateLimit (draft-11, expires 24 Nov 2026; flagged "not ready" at the -10 review) and Idempotency-Key (draft-07, expired 18 Apr 2026, no -08 yet). Internet-Drafts must not be cited as stable standards.
- **RFC 9457 Appendix D lists exactly three changes;** the `about:blank` default and optional `type` behaviors are inherited from 7807, not new. Only `about:blank` was registered *by* 9457 itself; `#date`/`#ohttp-key` came via RFC 9458.
- **Adoption cells reflect the primary/observed behavior of each vendor's flagship REST surface** as of retrieval. Per-API nuance across API versions (e.g., Stripe v1 vs v2 idempotency windows; Stripe v2 accepts RFC-3339-style behavior differences) is detailed in later parts. Full per-API error/pagination/patch analysis is deferred to Parts 2–7 by scope.
- **Microsoft does not name-check or explicitly "reject" RFC 7807/9457;** it simply mandates an OData-derived error object, which is not Problem Details.
- **Twilio's public "API responses" page renders the error example primarily in XML;** the JSON field names (`code`, `message`, `more_info`, `status`) are confirmed via the documented property table rather than a rendered JSON blob.
- **Some publication facts** (e.g., which vendors publish OpenAPI) were verified via official-doc corroboration rather than exhaustive inspection of every spec file.
- **Conditional-request and caching adoption cells** are conservative: several vendors support ETags on specific endpoints but not uniformly; treat the "P" cells as "supported on some resources, not a blanket policy."