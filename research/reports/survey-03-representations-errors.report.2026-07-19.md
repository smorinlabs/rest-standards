# REST API Conventions Series — Part 3/7: Representations & Errors

*Descriptive survey of request/response body conventions, data-type representations, and error shapes across eight references. Scope is strictly "what the JSON looks like" for success and failure; pagination/reliability/lifecycle/webhooks are out of scope (Parts 4–7). Retrieval dates noted per source; most primary content retrieved July 2026.*

## TL;DR
- **The field splits hardest on four axes:** (1) **field casing** — snake_case (Stripe, GitHub, Twilio bodies, Shopify, Zalando-mandated) vs camelCase (Microsoft Graph, Google's proto3-derived JSON); (2) **timestamp format** — Stripe's integer Unix-epoch **seconds** vs RFC 3339/ISO 8601 strings (GitHub, Google, Microsoft, Zalando) vs Twilio's RFC 2822 strings; (3) **money** — Stripe's minor-unit integers vs Zalando's decimal numbers vs Shopify's decimal strings (no reputable reference uses floats); (4) **error shape** — proprietary shapes at every commercial API vs RFC 9457 `application/problem+json` mandated only by Zalando's guideline. No reference is canonical; Stripe is a key comparator but idiosyncratic on timestamps and money.
- **Envelopes split three ways:** Stripe uses a typed `{"object":"list","data":[…]}` list envelope plus an `object` discriminator on every resource; GitHub returns **bare JSON arrays** for collections while Twilio/Graph/Google/Shopify return lightly-wrapped or resource-keyed bare objects; JSON:API is the formalized envelope extreme and Zalando mandates a top-level object (never a bare array).
- **Error bodies converge in intent, not in shape:** every model carries a machine-readable code + human message + a correlation/request ID (in headers, sometimes in the body). Google's `google.rpc.Status` and Microsoft's OData-derived `error{code,message,target,details,innererror}` are the most fully specified proprietary models; only Zalando mandates the IETF standard (RFC 9457).

## Key Findings

1. **Casing is a clean two-way split with a proto3 twist.** Stripe, GitHub, and Twilio (bodies) use `snake_case`; Microsoft Graph uses `camelCase`. Google's REST/JSON surface uses `camelCase`, but it is a *derived* mapping: `.proto` files define `lower_snake_case` field names (AIP-140), which the proto3 JSON mapping converts to `lowerCamelCase` (`first_name` → `firstName`); proto3 JSON parsers accept both forms on input. Zalando *mandates* `snake_case` (`^[a-z_][a-z_0-9]*$`, "never camelCase") for both body properties and query params, giving body/query casing consistency. Twilio is the notable inconsistency — snake_case bodies but **PascalCase** query/POST params (`To`, `From`, `Body`).

2. **Stripe's typed envelope and `object` field are unique in the set.** Every Stripe resource carries an `object` string discriminator (`"charge"`, `"customer"`, `"list"`, `"payment_intent"`). Lists are objects: `{"object":"list","url":"/v1/charges","has_more":false,"data":[…]}`. GitHub returns bare JSON arrays for collections (lean but non-extensible — no room for pagination metadata without a breaking change). Zalando forbids bare-array top levels. JSON:API is the formalized extreme (`data`/`errors`/`included`/`meta`, with `data` and `errors` forbidden to coexist).

3. **Timestamps are the sharpest scalar divergence.** Stripe uses integer Unix-epoch **seconds** (field `created`, e.g. `1710000000`). GitHub, Google, Microsoft, and Zalando use RFC 3339 / ISO 8601 **strings**. Twilio is a third variant: RFC 2822 GMT strings (`"Fri, 13 Aug 2010 01:16:24 +0000"`). Field naming also diverges: Stripe `created`; GitHub `created_at`/`updated_at`; Google `create_time`/`update_time` (imperative, per AIP-142 — never `created_time`); Twilio `date_created`/`date_updated`/`date_sent`; Zalando recommends the `_at` suffix.

4. **Money splits three ways, with no float adopter.** Stripe: integer minor units (`amount: 1000`) + lowercase ISO-4217 `currency: "usd"`. Zalando: JSON `number` with OpenAPI `format: decimal` (example `99.95`) + `currency` in ISO-4217, explicitly warning "don't convert the amount field to float/double." Shopify Admin REST: decimal **strings**. Google offers `google.type.Money` (int64 units + int32 nanos). Nobody in the set recommends floats.

5. **Identifiers span the full spectrum.** Stripe: prefixed opaque strings (`cus_`, `ch_`, `pi_`, `pm_`) that double as human-scannable type hints. GitHub: numeric `id` **plus** an opaque base64 `node_id` (bridge to GraphQL global IDs). Twilio: prefixed 34-char SIDs (`AC…`, `SM…`/`MM…`, `CA…`, `MG…`, `PN…`). Google: full/relative resource **names** (`publishers/123/books/les-miserables`) with an optional UUID4 `uid`. Zalando: mandates opaque **string** IDs (never numbers), stable and never recycled, suggesting UUID/ULID.

6. **Error shapes are all proprietary except Zalando.** Stripe: `{"error":{type,code,message,param,doc_url,…}}`. GitHub: `{message, errors:[{resource,field,code}], documentation_url}`. Google: `google.rpc.Status` `{error:{code,message,status,details:[…]}}` with typed `details` (`ErrorInfo`, `BadRequest`, `Help`, `LocalizedMessage`). Microsoft Graph/Azure: `{error:{code,message,target,details:[],innererror:{…}}}` with recursively-nested `innererror`. Only Zalando mandates RFC 9457 `application/problem+json`.

7. **Request/correlation IDs are near-universal but named differently.** Stripe `Request-Id` header (plus `request_log_url` in the error body). GitHub `X-GitHub-Request-Id`. Microsoft/Azure `x-ms-request-id` (+ `x-ms-client-request-id`), with Graph also embedding `request-id`/`client-request-id` inside `innerError`. AWS `x-amzn-RequestId` (S3 `x-amz-request-id`).

## Baseline Position — what the standards & guideline docs prescribe for this surface

**RFC 9457 (Problem Details for HTTP APIs).** Standards Track, authored by M. Nottingham, E. Wilde, and S. Dalal, published July 2023 as the product of the IETF "Building Blocks for HTTP APIs" working group; it obsoletes RFC 7807. Media type `application/problem+json`. Canonical members, all optional: `type` (URI reference identifying the problem type; default `about:blank`), `title` (short human summary that "does not vary for different occurrences"), `status` (integer HTTP status; advisory, must equal the real response status), `detail` (human explanation of this occurrence), `instance` (URI reference for this occurrence). Extension members allowed (e.g. a validation `errors` array, `balance`/`cost`). Verbatim from the RFC:
```
HTTP/1.1 403 Forbidden
Content-Type: application/problem+json
{ "type":"https://example.com/probs/out-of-credit",
  "title":"You do not have enough credit.", "status":403,
  "detail":"Your current balance is 30, but that costs 50.",
  "instance":"/account/12345", "balance":30,
  "accounts":["/account/12345","/account/67890"] }
```

**RFC 3339 / ISO 8601 (timestamps).** `{year}-{month}-{day}T{hour}:{min}:{sec}[.{frac_sec}]Z`; four-digit year; zero-padded components; up to 9 fractional-second digits; `Z` (or numeric offset) required. Proto3 serializers always emit UTC `Z`; parsers accept offsets.

**JSON:API (envelope extreme).** Top-level object must contain at least one of `data`, `errors`, `meta`. Errors live in a top-level `errors` array of objects with members such as `id`, `status`, `code`, `title`, `detail`, `source` (`pointer`/`parameter`). `data` and `errors` must not coexist.

**Zalando RESTful API Guidelines (guideline doc, not an API).** MUST: property names `snake_case` (`^[a-z_][a-z_0-9]*$`), never camelCase; query params `snake_case` (`^[a-z][_a-z0-9]*$`). MUST: top-level JSON is an object (no bare arrays), Internet-JSON (RFC 7493) compliant. MUST: dates RFC 3339 with uppercase `T`/`Z`, UTC without offset recommended. SHOULD: date/time fields end in `_at`. MUST: booleans must not be null; null and absent MUST have the same semantics (exception: JSON Merge Patch RFC 7396, where null means delete). SHOULD: enum values `UPPER_SNAKE_CASE`. MUST: money as `type: number, format: decimal` + ISO-4217 `currency`, never float. MUST: numeric fields declare a precision format (`int32`/`int64`/`bigint`/`float`/`double`/`decimal`). MUST: IDs are opaque strings (not numbers), stable, never recycled. MUST: errors use RFC 9457 `application/problem+json`. Forbids the RFC 8288 `Link` header with JSON media types (links embedded in payload instead).

**Google AIP (design guideline, implemented by Google Cloud).** AIP-122: every resource has a string `name` = its resource name; IDs are strings; no self-links/tuples. AIP-140: proto field names `lower_snake_case`. AIP-142: absolute time = `google.protobuf.Timestamp` (RFC 3339 JSON string), fields end in `_time`, imperative form. AIP-148: standard fields `name`, `parent`, `uid` (UUID4, OUTPUT_ONLY), `display_name`, `create_time`, `update_time`, `delete_time`, `expire_time`. AIP-154: `etag` string (RFC 7232; weak etags `W/`-prefixed). AIP-193: errors are `google.rpc.Status`; per the AIP-193 changelog, "**2023-05-10: Require ErrorInfo for all error responses**" — all error responses must include an `ErrorInfo` within `details`; `ErrorInfo.reason` is UPPER_SNAKE_CASE matching `[A-Z][A-Z0-9_]+[A-Z0-9]`.

**Microsoft REST API Guidelines (basis for Graph & Azure).** Error body based on OData v4: single `error` object with required `code`+`message`, optional `target`, `details[]`, `innererror`. Per the Microsoft REST API Guidelines (microsoft/api-guidelines, `Guidelines.md`): "Services SHOULD have a relatively small number (about 20) of possible values for 'code,' and all clients MUST be capable of handling all of them. … Introducing a new value for 'code' that is visible to existing clients is a breaking change and requires a version increase." Per the Microsoft Graph guidelines, "The top-level error code MUST match the HTTP response status code description, converted to camelCase, as listed in the Status Code Registry (iana.org)." More-specific codes go in `innererror` (nested recursively) or `details`; clients traverse to the deepest code they understand. `message` is developer-facing, not localized, not for end users. Azure guidelines add: use the same JSON schema for PUT/PATCH/GET/POST on a path; include `Retry-After` for transient errors.

## Side-by-side comparison tables

### Table A — Casing / Envelope / Null

| Reference | Body casing | Query-param casing | Envelope | Null vs absent policy |
|---|---|---|---|---|
| Stripe | snake_case | snake_case (bracketed ops, e.g. `created[gte]`) | Typed: every object has `object`; lists `{"object":"list","data":[…],"has_more"}` | Fields present with explicit `null` (e.g. `"mandate": null`); empty metadata `{}` |
| GitHub | snake_case | snake_case | Bare objects; **bare arrays** for collections | Fields present; `null` used; summary vs detailed representations omit some fields |
| Google (AIP) | camelCase (proto3 JSON from snake proto) | camelCase | Bare resource object; list wraps named array + `nextPageToken` | Default/empty values often omitted; null ≈ absent |
| Microsoft Graph | camelCase | camelCase (+ OData `$`-prefixed) | Bare object; collections under `value` with `@odata.*` | Fields may be omitted; `@odata.*` control fields |
| Twilio | snake_case | **PascalCase** (`To`, `From`, `Body`) | Bare object; lists have `first_page_uri`/`next_page_uri`/`page`/`page_size` | Fields present with `null` (`date_sent`, `price`) |
| Shopify Admin REST | snake_case | snake_case | Resource-keyed bare object (`{"product":{…}}`) | Fields present; null used |
| Zalando (guideline) | snake_case (MANDATED) | snake_case (MANDATED) | Top-level object MANDATED (no bare arrays) | null == absent MUST match; booleans never null; empty array `[]` not null; Merge-Patch null = delete |
| AWS (light) | Mixed PascalCase (JSON protocols) | n/a (POST bodies / query protocol) | Bare/service-specific | Service-specific |

### Table B — Scalars: Time, Money, IDs

| Reference | Timestamp format | Time field names | Money | Currency | ID format |
|---|---|---|---|---|---|
| Stripe | Integer Unix epoch **seconds** | `created` | Integer minor units (`amount:1000`) | lowercase ISO-4217 (`"usd"`) | Prefixed opaque (`cus_`,`ch_`,`pi_`) |
| GitHub | RFC 3339 string `…Z` (UTC) | `created_at`,`updated_at` | n/a | n/a | Numeric `id` + base64 `node_id` |
| Google (AIP) | RFC 3339 string (`Z`, UTC) | `create_time`,`update_time` | `google.type.Money` (int64 units + nanos) | ISO-4217 | Resource `name` path; optional UUID4 `uid` |
| Microsoft Graph | RFC 3339 / ISO 8601 string | `createdDateTime`,`lastModifiedDateTime` | service-specific | ISO-4217 | GUID strings |
| Twilio | RFC 2822 string (GMT) | `date_created`,`date_updated`,`date_sent` | Decimal string `price` + `price_unit` | ISO-4217 in `price_unit` | Prefixed 34-char SID |
| Shopify Admin REST | ISO 8601 string | `created_at`,`updated_at` | Decimal **string** | Shop vs presentment ambiguity (caveat) | Numeric `id` (+ GraphQL `gid://shopify/…`) |
| Zalando (guideline) | RFC 3339 string, upper T/Z, UTC preferred | `_at` suffix | `number`/`decimal` + `currency` (never float) | ISO-4217 (`iso-4217` format) | Opaque string (UUID/ULID), never numeric |
| AWS (light) | epoch seconds or ISO 8601 (service-specific) | service-specific | service-specific | ISO-4217 | ARNs / service-specific |

### Table C — Errors

| Reference | Media type | Error envelope | Machine code | Per-field detail | Correlation ID |
|---|---|---|---|---|---|
| Stripe | application/json | `{"error":{…}}` | `type` (enum) + string `code` | `param` (single field name) | `Request-Id` header; `request_log_url` in body |
| GitHub | application/json | `{message, errors:[…], documentation_url}` | string `code` (`missing_field`, `custom`…) | `errors[]` `{resource,field,code}` | `X-GitHub-Request-Id` header |
| Google | application/json | `{error:{code,message,status,details:[]}}` | `status` enum + typed `details` (`ErrorInfo.reason`) | `BadRequest.fieldViolations[]` | `RequestInfo.request_id` in details |
| Microsoft Graph | application/json | `{error:{code,message,innerError,…}}` | string `code` + nested `innerError.code` | `details[]` `{code,message,target}` | `request-id`/`client-request-id` in `innerError`; `x-ms-request-id` header |
| Azure (guideline) | application/json | `{error:{code,message,target,details:[],innererror:{}}}` | small stable `code` set + recursive `innererror` | `details[]`, `target` | `x-ms-request-id`, `x-ms-correlation-request-id` |
| Twilio | application/json / XML | flat `{code,message,more_info,status}` | integer `code` (e.g. `20404`) | none (flat) | error dictionary via `more_info` URL |
| Zalando (guideline) | application/problem+json | RFC 9457 problem object | `type` URI | extension members / `207` batch | per general guidance |
| AWS (light) | application/json or XML | JSON `{message}` + `x-amzn-ErrorType` header; S3/EC2 XML `<Error><Code><Message>` | exception name in `x-amzn-ErrorType`/`<Code>` | `x-amzn-RequestId` / `x-amz-request-id` |

## Per-reference notes (representation character sketches)

**Stripe.** The idiosyncratic-but-internally-consistent outlier. snake_case; a self-describing typed model where every object announces its `object` type and lists are themselves objects (`object:"list"`). Two choices are unusual in the modern field: integer Unix-epoch-**seconds** timestamps (`created`) and integer **minor-unit** money (`amount` + lowercase `currency`). Prefixed opaque IDs (`cus_`, `ch_`) double as type hints. First-class user-extensible `metadata` (string→string map, unset a key via empty value) and a `livemode` boolean on resources. Errors are a single `error` object typed by `type` (`card_error`, `invalid_request_error`, …) with string `code`, `message`, `param`, `doc_url`, plus card-specific `decline_code`. Correlation via `Request-Id`. Retrieved docs.stripe.com, July 2026.

**GitHub REST.** snake_case, bare objects, and — distinctively — **bare JSON arrays** for collections. Dual identifiers: numeric `id` plus opaque base64 `node_id`. RFC 3339 UTC timestamps (`created_at`/`updated_at`); GitHub historically migrated some payloads away from integer epoch to `YYYY-MM-DDTHH:MM:SSZ`. Minimalist errors: top-level `message` + `documentation_url`, with `422` "Validation Failed" carrying an `errors[]` array of `{resource, field, code}` (codes `missing_field`, `invalid`, `already_exists`, `custom`). 10-second request timeout. Correlation via `X-GitHub-Request-Id`.

**Google Cloud / AIP.** The most rigorously specified reference. proto-first: `lower_snake_case` proto → `camelCase` JSON. Identity is a **resource name** path, not a bare ID; optional UUID4 `uid`. Standard fields codified (`name`, `create_time`, `update_time`, `etag`, `display_name`). Errors are `google.rpc.Status` (`code`, `message`, `details[]`) with typed detail messages (`ErrorInfo` with UPPER_SNAKE_CASE `reason` + `domain` + `metadata`, `BadRequest.fieldViolations`, `Help`, `LocalizedMessage`). ErrorInfo required on all errors since May 2023.

**Microsoft (Azure guidelines + Graph).** camelCase (Graph). The OData-derived error model is the most structurally elaborate: single `error` with required `code`+`message`, optional `target`, `details[]`, and a recursively-nested `innererror` for progressively more specific codes (clients "traverse to the deepest one they understand"). Top-level `code` set kept small (~20) and stable; new top-level codes are breaking and must match the HTTP status description in camelCase. `message` explicitly developer-only, unlocalized. Correlation IDs embedded in `innerError` and returned as `x-ms-request-id`.

**Twilio.** snake_case bodies but **PascalCase** query/POST params — a body/param casing mismatch unique in the set. Timestamps are RFC 2822 GMT strings. IDs are prefixed 34-char SIDs. Errors are a **flat** object `{code, message, more_info, status}` (integer `code` and `status`), with `more_info` pointing to the error dictionary; the XML variant wraps `<RestException>`. Legacy 2010-04-01 API defaults to XML; product APIs are JSON-only. Retrieved twilio.com/docs, July 2026.

**Shopify Admin REST.** snake_case, resource-keyed bare objects (`{"product":{…}}`), money as decimal **strings**. Now a legacy API: REST Admin marked legacy Oct 1 2024, with product/variant REST endpoints deprecating in favor of GraphQL. Currency caveat: REST order money fields' shop-vs-presentment-currency semantics are ambiguous/undocumented (community-reported), while GraphQL exposes explicit `presentmentPrices` and `{amount, currencyCode}` — a reason to treat Shopify REST money fields with care.

**Zalando (guideline, not an API).** The prescriptive counterpoint: mandates snake_case (body + query), top-level objects (no bare arrays), RFC 3339 UTC timestamps with `_at` suffix, opaque string IDs, decimal money (never float), UPPER_SNAKE_CASE enums, null==absent semantics, and — alone in the set — RFC 9457 `application/problem+json` for all errors.

**AWS (light — full treatment in Part 2).** No single representation: JSON protocols use PascalCase fields; errors range from JSON `{"message":…}` + `x-amzn-ErrorType` header (WAF, Bedrock, Network Firewall) to XML `<Error><Code><Message><RequestId>` (S3, EC2). Correlation via `x-amzn-RequestId` / `x-amz-request-id`.

## Agreements vs Divergences

**Agreements (near-consensus):**
- Machine code + human message + correlation ID in every error model.
- ISO-4217 currency codes wherever money appears.
- RFC 3339/ISO 8601 string timestamps for everyone *except* Stripe (epoch integer) and Twilio (RFC 2822).
- Opaque or at least stable IDs; Google/Zalando/Stripe explicitly commit to ID stability and non-recycling.
- `message` fields are developer-facing, not for end users / not localized (explicit at Microsoft and Google).

**Divergences (with descriptive tradeoffs):**
- *Casing:* snake_case dominates payment/dev-tool APIs and aids DB-column alignment; camelCase aligns with JS/proto tooling. Google's derived mapping means the same API reads snake in proto and camel in JSON.
- *Envelope:* Stripe's typed envelope enables generic client handling and self-description at the cost of verbosity; bare arrays (GitHub) are lean but non-extensible.
- *Timestamps:* epoch integers (Stripe) are compact and unambiguous but not human-readable and lose sub-second/timezone expressivity; RFC 3339 strings are readable and standard; Twilio's RFC 2822 is human-readable but weakest for machine parsing.
- *Money:* integer minor units (Stripe) are exact and arithmetic-safe but require currency-exponent knowledge; decimal numbers (Zalando) risk float coercion in JS parsers; decimal strings (Shopify) are parse-safe but require string→decimal conversion.
- *Error shape:* RFC 9457 gives cross-API uniformity and tooling but is adopted (in this set) only by Zalando's guideline; proprietary shapes (Stripe/GitHub/Google/MS) are entrenched and each optimized for their own client libraries.

## CONTESTED AXES REGISTER (scoped to Part 3)

| Axis | Options observed | Who does what | Tradeoff (one line) | How contested |
|---|---|---|---|---|
| Field casing | snake_case / camelCase / PascalCase | snake: Stripe, GitHub, Twilio(body), Shopify, Zalando(mandate) · camel: MS Graph, Google(JSON) · Pascal: AWS JSON protocols, Twilio(query) | Readability/DB-alignment vs JS/proto-tooling alignment | **Split** (two roughly even camps) |
| Envelope vs bare | Typed envelope / bare object / bare array / JSON:API | Typed: Stripe · bare object: MS Graph, Google, Twilio, Shopify · bare array: GitHub · object-mandated: Zalando · JSON:API extreme | Extensibility & self-description vs leanness | **Split** |
| Null vs absent | Always-present-null / omit-when-empty / null==absent | present+null: Stripe, Twilio · omit: Google · null==absent mandate: Zalando · Merge-Patch null=delete: Zalando, MS PATCH | Predictable shape vs payload size; PATCH delete semantics | **Split** |
| Timestamp format | epoch seconds int / RFC 3339 string / RFC 2822 string | epoch: Stripe · RFC 3339: GitHub, Google, MS, Zalando · RFC 2822: Twilio | Compactness vs human-readability vs standard parsing | **Near-consensus on RFC 3339**, two dissenters |
| Money representation | minor-unit int / decimal number / decimal string / Money type | int: Stripe · decimal number: Zalando · decimal string: Shopify · Money type: Google | Arithmetic safety vs float-coercion risk vs conversion overhead | **Wide-open** (no float adopters) |
| ID format | prefixed opaque / numeric+node / SID / resource-name / UUID | Stripe prefixed · GitHub numeric+node_id · Twilio SID · Google name · Zalando opaque-string mandate | Type-scannability vs global-uniqueness vs hierarchy encoding | **Wide-open** |
| Error shape | RFC 9457 / proprietary nested / proprietary flat | 9457: Zalando · nested: MS(innererror), Google(details) · Stripe error obj · flat: Twilio | Cross-API standard/tooling vs client-library optimization | **Split, newer guidance trends to 9457** |
| Validation-error shape | array of field violations / single param / none | GitHub `errors[]`, Google `fieldViolations[]`, MS `details[]` · Stripe single `param` · Twilio none | Multi-field reporting richness vs simplicity | **Split** |
| Enum casing | lower_snake / UPPER_SNAKE / camel | Stripe lower_snake (`card_error`) · Google/Zalando UPPER_SNAKE · MS camel (`badRequest`) | Distinguishing enums from fields vs language idiom | **Split** |
| Open vs closed enums | closed / open / tolerant reader | Zalando open (`x-extensible-enum`, now deprecated internally) · MS Graph "service may add codes anytime" · Google ErrorInfo extensible | Evolvability without version bump vs client-parse safety | **Split** |
| Metadata pattern | user-settable string map / none | Stripe `metadata{}` first-class · others none standard | Customer extensibility vs schema discipline | **Near-consensus that Stripe is distinctive** |
| Correlation-ID surfacing | header only / header+body | Stripe header+`request_log_url` · GitHub header · MS header+innerError · Twilio more_info · AWS header | Ease of support-workflow correlation | **Near-consensus (all surface it; naming differs)** |

## EXAMPLES APPENDIX (verbatim payloads, header lines, concrete numbers)

### Stripe (docs.stripe.com, July 2026)
List envelope + resource:
```json
{ "object": "list", "url": "/v1/charges", "has_more": false, "data": [ { "id": "ch_3QWZ2e2eZvKYlo2C0XjzQp0a", "object": "charge", "amount": 1000, "amount_captured": 1000, "created": 1710000000, "currency": "usd", "customer": "cus_QZ9a1b2c3d4e5f", "livemode": false, "metadata": {}, "paid": true, "payment_intent": "pi_3QWZ2d2eZvKYlo2C0abc1234" } ] }
```
"Time at which the object was created. Measured in seconds since the Unix epoch." "Three-letter ISO currency code, in lowercase."
Error object:
```json
{ "error": { "code": "card_declined", "decline_code": "lost_card", "doc_url": "https://stripe.com/docs/error-codes/card-declined", "message": "Your card was declined.", "param": "limit", "request_log_url": "https://dashboard.stripe.com/test/logs/req_123", "type": "invalid_request_error" } }
```
Error `type` enum: `api_connection_error`, `api_error`, `authentication_error`, `card_error`, `idempotency_error`, `invalid_request_error`, `rate_limit_error`. Request-ID header: `Request-Id`. Expansion max depth: 4 levels.

### GitHub (docs.github.com, July 2026)
Validation error:
```
HTTP/1.1 422 Unprocessable Entity
Content-Length: 149
{ "message": "Validation Failed", "errors": [ { "resource": "Issue", "field": "title", "code": "missing_field" } ] }
```
Other errors: `{"message":"Problems parsing JSON"}` (400); `{"message":"Body should be a JSON object"}` (400); rate-limit 403/429 with `{"message":"API rate limit exceeded…","documentation_url":"…"}`. IDs: `"id": 1, "node_id": "MDU6SXNzdWUx"`. Timestamp: `"created_at": "2013-02-27T19:35:32Z"`. Header: `X-GitHub-Request-Id: E3A2:160030:DC7C75:F76DCA:6990609D`. Timeout: 10 seconds. Version header example: `X-GitHub-Api-Version:2022-11-28`.

### Google (cloud.google.com / aip.dev, July 2026)
Error:
```json
{ "error": { "code": 403, "message": "Cloud Pub/Sub API has not been used in project 87380058272 before or it is disabled…", "status": "PERMISSION_DENIED", "details": [ { "@type": "type.googleapis.com/google.rpc.ErrorInfo", "reason": "SERVICE_DISABLED", "domain": "googleapis.com", "metadata": { "consumer": "projects/87380058272", "service": "pubsub.googleapis.com" } } ] } }
```
Timestamp JSON: `"2017-01-15T01:30:15.01Z"`. Resource name: `publishers/123/books/les-miserables`. `ErrorInfo.reason` regex `[A-Z][A-Z0-9_]+[A-Z0-9]` (≤63 chars). `uid` = UUID4. Timestamp valid range: `0001-01-01T00:00:00Z` to `9999-12-31T23:59:59Z`. AIP-193 changelog: "2023-05-10: Require ErrorInfo for all error responses."

### Microsoft Graph / Azure (learn.microsoft.com + microsoft/api-guidelines, July 2026)
```json
{ "error": { "code": "badRequest", "message": "Uploaded fragment overlaps with existing data.", "innerError": { "code": "invalidRange", "request-id": "request-id", "date": "date-time" } } }
```
With `details`:
```json
{ "error": { "code": "invalidRequest", "message": "The request is malformed or incorrect.", "innerError": { "code": "99901", "message": "InvalidInput", "details": [ { "InvalidReferralForCoSellConversion": [ "If PartnerLed referral has no solution it cannot be converted to co-sell referral" ] } ] } } }
```
Azure guideline error: `{ "error": { "code": "BadArgument", "message": "Previous passwords may not be reused", "target": "password", "innererror": { "code": "PasswordError", "innererror": { "code": "PasswordDoesNotMeetPolicy" } } } }`. Rules (verbatim, `Guidelines.md`): "Services SHOULD have a relatively small number (about 20) of possible values for 'code,' and all clients MUST be capable of handling all of them. … Introducing a new value for 'code' … is a breaking change and requires a version increase." Graph: "The top-level error code MUST match the HTTP response status code description, converted to camelCase, as listed in the Status Code Registry (iana.org)."

### Twilio (twilio.com/docs, July 2026)
JSON error (400):
```json
{ "status": 400, "message": "No to number is specified", "code": 21201, "more_info": "http://www.twilio.com/docs/errors/21201" }
```
Minimal 404: `{ "status": 404, "message": "The requested resource was not found" }`. `code` (integer) and `more_info` (string) are optional; `status` (integer) and `message` always present. XML variant:
```xml
<TwilioResponse><RestException><Status>400</Status><Message>No to number is specified</Message><Code>21201</Code><MoreInfo>http://www.twilio.com/docs/errors/21201</MoreInfo></RestException></TwilioResponse>
```
Resource body (snake_case + RFC 2822):
```json
{ "sid": "SM1f0e8ae6ade43cb3c0ce4525424e404f", "date_created": "Fri, 13 Aug 2010 01:16:24 +0000", "date_updated": "Fri, 13 Aug 2010 01:16:24 +0000", "date_sent": null, "account_sid": "ACXXXX…", "to": "+15305431221", "from": "+15104564545", "body": "A Test Message", "status": "queued", "flags":["outbound"], "api_version": "2010-04-01", "price": null }
```
"Twilio returns all dates and times in GMT using the RFC 2822 format." SID patterns: Account `^AC[0-9a-fA-F]{32}$`, Message `^(SM|MM)[0-9a-fA-F]{32}$` (34 chars), Call `CA…`, Phone Number `PN…`, Messaging Service `MG…`. Media limits: 10 `media_url` max/message; 5 MB image / 500 KB other.

### Shopify Admin REST (2026)
REST Admin API marked legacy Oct 1 2024. Product/variant REST endpoints deprecating; per Shopify's developer changelog the new GraphQL product APIs "increase per-product variant support from our historical max of 100 to a new limit of 2048," and "With the 2024-04 API release, we will also be deprecating management of both variants and options via the GraphQL Product Input object and the /products and /variants REST API endpoints." All public apps required on GraphQL product APIs as of Feb 1 2025. Money as decimal strings in REST; GraphQL exposes `price { amount currencyCode }` and `presentmentPrices`. Order money shop-vs-presentment currency ambiguity flagged in Shopify community.

### Zalando (opensource.zalando.com/restful-api-guidelines, 2026)
Problem JSON example:
```
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json
{ "type": "/problems/validation-error", "title": "Validation Failed", "status": 400, "detail": "Email address is not in a valid format", "instance": "/problems/validation-error#/email" }
```
Property regex `^[a-z_][a-z_0-9]*$`; query-param regex `^[a-z][_a-z0-9]*$`. Money schema: `Money: {amount: {type:number, format:decimal, example:99.95}, currency:{type:string, format:iso-4217, example:EUR}}`. Timestamp example `2015-05-28T14:07:17Z` (preferred over offsets). Enum values `UPPER_SNAKE_CASE`. Batch/bulk errors: `207 Multi-Status`.

### AWS (light — full analysis in Part 2)
JSON-protocol error:
```
HTTP/1.1 400 Bad Request
x-amzn-RequestId: b0e91dc8-3807-11e2-83c6-5912bf8ad066
x-amzn-ErrorType: ValidationException
{"message":"1 validation error detected: Value null at 'TargetString' failed to satisfy constraint…"}
```
S3 XML error: `<Error><Code>NoSuchKey</Code><Message>The resource you requested does not exist</Message><Resource>/mybucket/myfoto.jpg</Resource><RequestId>4442587FB7D0A2F9</RequestId></Error>` with header `x-amz-request-id`. EC2 XML: `<Response><Errors><Error><Code>…</Code><Message>…</Message></Error></Errors><RequestID>…</RequestID></Response>`.

## Caveats
- **Guideline docs vs live behavior.** Zalando, Google AIP, and the Microsoft/Azure guidelines are prescriptive documents; individual Google Cloud and Microsoft services deviate (e.g. some Google errors also carry a legacy `errors[]` array with `{message,domain,reason}` alongside `google.rpc.Status`). Treat guideline mandates as "what the doc says," not a guarantee for every endpoint.
- **Stripe is a reference, not a canon.** Its epoch-seconds timestamps and minor-unit money are minority choices in the wider field; documented here descriptively.
- **Shopify REST is a moving target.** REST Admin is legacy as of Oct 2024; money/currency semantics are best verified against GraphQL. The REST currency-field ambiguity is community-reported rather than authoritatively documented.
- **Twilio JSON error bodies** are only rendered in the markdown/JSON-tab version of the docs page; field ordering differs between docs (status, message, code, more_info) and live responses (code, message, more_info, status) — order is not semantically significant. Docs escape slashes (`\/`); live responses do not.
- **Field casing for Google** is a derived mapping; whether you see snake or camel depends on whether you read proto or JSON. Query-param casing consistency claims are strongest for Zalando (explicit) and weakest where docs are silent.
- **Retrieval dates:** Stripe, GitHub, Google, Microsoft, Twilio, Zalando content retrieved July 2026; Shopify deprecation-timeline dates (Oct 1 2024 legacy; Feb 1 / Apr 1 2025 migration deadlines) are from Shopify and third-party migration guides.
- **Out-of-scope reminder:** pagination/filter/expansion response shapes (Part 4), reliability/idempotency (Part 5), lifecycle/versioning/auth (Part 6), webhooks (Part 7), and AWS's full body/error analysis (Part 2).