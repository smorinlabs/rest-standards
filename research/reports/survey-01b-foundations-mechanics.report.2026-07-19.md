# REST API Conventions Series — Part 1b: Foundations Supplement (Specification Mechanics)

*Descriptive research. Primary spec texts retrieved 18–19 July 2026. Draft revisions cited inline. Quotes are verbatim from RFC Editor / IETF Datatracker / official spec sites.*

## 1. TL;DR
- Every foundational standard's implementation-grade mechanics are captured here verbatim and are reproducible from this report alone: RFC 9457 (Problem Details), RFC 9110 (conditional requests), RFC 9111 (caching), RFC 8288 (Link), RFC 7396/6902/6901 (patch + pointer), RFC 3339/9557 (timestamps), RFC 8594/9745 (sunset/deprecation), the RateLimit draft, the Idempotency-Key draft, OpenAPI 3.1, and JSON:API 1.1.
- Two items are UNSTABLE Internet-Drafts and MUST be flagged as such in any prescriptive standard: **RateLimit** (draft-ietf-httpapi-ratelimit-headers-**11**, active, expires 24 Nov 2026) and **Idempotency-Key** (draft-ietf-httpapi-idempotency-key-header-**07**, EXPIRED 18 Apr 2026 — "no longer active").
- No corrections to Part 1 are required; all Part 1 status facts are confirmed. One clarification: RFC 9745 (Deprecation) uses an RFC 9651 structured-field Date (the `@<unix>` form), while the Sunset header retains the older HTTP-date format — the two deliberately differ "for historical reasons."

---

## 2. Per-standard mechanics

### Item 1 — RFC 9457, Problem Details for HTTP APIs
Standards Track, July 2023 (M. Nottingham, E. Wilde, S. Dalal); obsoletes RFC 7807. Media types: `application/problem+json` and `application/problem+xml`. The canonical data model is a JSON object.

**Members (§3.1).** Governing rule: *"If a member's value type does not match the specified type, the member MUST be ignored -- i.e., processing will continue as if the member had not been present."*
- **`type`** (string, URI reference) — identifies the problem type; *"Consumers MUST use the 'type' URI (after resolution, if necessary) as the problem type's primary identifier."* When absent, *"its value is assumed to be 'about:blank'."* Relative URIs resolve against the document base URI (RFC 3986 §5); absolute URIs are RECOMMENDED. May be non-resolvable (e.g., a `tag:` URI such as `tag:example@example.org,2021-09-17:OutOfLuck`). *"Consumers SHOULD NOT automatically dereference the type URI"* except when providing developer/debug information.
- **`status`** (number) — the HTTP status code. *"The 'status' member, if present, is only advisory ... Generators MUST use the same status code in the actual HTTP response, to assure that generic HTTP software that does not understand this format still behaves correctly."*
- **`title`** (string) — short human-readable summary. *"It SHOULD NOT change from occurrence to occurrence of the problem, except for localization."*
- **`detail`** (string) — human-readable explanation specific to this occurrence. *"Consumers SHOULD NOT parse the 'detail' member for information."*
- **`instance`** (string, URI reference) — identifies the specific occurrence; may be dereferenceable or an opaque unique identifier.

**`about:blank` (§4.2.1).** Used when the problem has no additional semantics beyond the HTTP status code; when so used, the `title` SHOULD be the HTTP status phrase for that code.

**Extension members (§3.2).** *"Problem type definitions MAY extend the problem details object with additional members that are specific to that problem type."* Namespacing: there is no formal namespace mechanism — authors choose names carefully, and names must conform to the XML `Name` rule (§2.3 of the XML spec) for `+xml` compatibility. Consumer rule: *"Clients consuming problem details MUST ignore any such extensions that they don't recognize."* Extensions are flat top-level members (e.g., `balance`, `accounts`).

**Multiple problems (§3).** No standardized "batch" container. *"When an API encounters multiple problems that do not share the same type, it is RECOMMENDED that the most relevant or urgent problem be represented in the response. While it is possible to create generic 'batch' problem types that convey multiple, disparate types, they do not map well into HTTP semantics."* Multiple problems of the *same* type are conveyed via a problem-type-specific extension array — the spec's own example uses an `errors` array whose members carry `detail` plus a JSON Pointer `pointer`.

**Interplay with HTTP status code (§1, §3.1.2).** The status code conveys generic semantics to HTTP intermediaries (client libraries, caches, proxies); the problem detail conveys API-specific detail in the body. Both MUST agree. Problem details *"most naturally fit the semantics of 4xx and 5xx responses"* but can be used with any status.

**Minimal example** (`about:blank` pattern, §4.2.1):
```
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json

{
  "type": "about:blank",
  "title": "Bad Request",
  "status": 400
}
```

**Extended example with extensions** (§3, verbatim):
```
HTTP/1.1 403 Forbidden
Content-Type: application/problem+json
Content-Language: en

{
 "type": "https://example.com/probs/out-of-credit",
 "title": "You do not have enough credit.",
 "detail": "Your current balance is 30, but that costs 50.",
 "instance": "/account/12345/msgs/abc",
 "balance": 30,
 "accounts": ["/account/12345",
              "/account/67890"]
}
```

---

### Item 2 — RFC 9110, Conditional requests
STD 97, June 2022 (Fielding, Nottingham, Reschke). Conditional-request semantics; supersedes RFC 7232, whose normative text is restated here.

**Precondition evaluation order.** A server evaluates in this sequence: (1) **`If-Match`** — if present and fails → 412; (2) **`If-Unmodified-Since`** — evaluated only if `If-Match` is absent; if it fails → 412; (3) **`If-None-Match`** — if present and fails → 304 (GET/HEAD) or 412 (other methods); (4) **`If-Modified-Since`** — evaluated only if `If-None-Match` is absent and method is GET/HEAD; then (5) **`If-Range`**. Precedence rules: *"A recipient MUST ignore If-Modified-Since if the request contains an If-None-Match header field"*; likewise `If-Unmodified-Since` is ignored when `If-Match` is present.

**Comparison algorithms.** `If-Match` uses the **strong** comparison — *"If a listed ETag has the W/ prefix indicating a weak entity tag, this comparison algorithm will never match it."* `If-None-Match` uses the **weak** comparison. ETags are strong validators by default; `W/` marks a weak validator (semantic equivalence, not byte-for-byte).

**304 vs 412 (verbatim).** *"An origin server MUST NOT perform the requested method if the condition evaluates to false; instead, the origin server MUST respond with either a) the 304 (Not Modified) status code if the request method is GET or HEAD or b) the 412 (Precondition Failed) status code for all other request methods."* A 304 *"MUST generate any of the following header fields that would have been sent in a 200 (OK) response to the same request: Cache-Control, Content-Location, Date, ETag, Expires, and Vary,"* and carries no message body.

**API usage.** `If-None-Match` for cache revalidation (GET/HEAD) and create-if-absent (`If-None-Match: *` on PUT → 412 if the resource exists). `If-Match` for optimistic-concurrency / lost-update prevention on PUT/PATCH.

**Cached-read flow ending in 304:**
```
GET /config.json HTTP/1.1
Host: api.example.com
If-None-Match: "v1001"

HTTP/1.1 304 Not Modified
ETag: "v1001"
Cache-Control: max-age=0, must-revalidate
```

**Guarded-write flow ending in 412:**
```
PUT /config.json HTTP/1.1
Host: api.example.com
If-Match: "cfg-42"
Content-Type: application/json

{"timeout": 60, "retries": 3}

HTTP/1.1 412 Precondition Failed
```
(The server holds a newer ETag, e.g. `"cfg-43"`, so the strong comparison fails and the write is refused without modifying the resource.)

---

### Item 3 — RFC 9111, Caching essentials for APIs
STD 98, June 2022 (Fielding, Nottingham, Reschke). *(Companion RFC 9110 is STD 97; the two share the "HTTP Semantics/Caching" STD-numbering scheme.)*

**Directive strings for common API postures (response `Cache-Control`):**
- Never store (sensitive/authenticated API responses): `Cache-Control: no-store`
- Private, non-shareable: `Cache-Control: private, no-store`
- Store but always revalidate before reuse, paired with an ETag: `Cache-Control: max-age=0, must-revalidate`
- Storable but revalidate every reuse: `Cache-Control: no-cache`
- Shared-cache lifetime override: `Cache-Control: s-maxage=60`

**Directive semantics (§5.2.2).** `no-store` = MUST NOT store any part of the request/response. `no-cache` = *"the response can be stored in caches, but the response must be validated with the origin server before each reuse, even when the cache is disconnected."* `must-revalidate` = may reuse while fresh; once stale MUST revalidate or generate a 504 if the origin is unreachable. `private` = only private (non-shared) caches may store. `max-age=N` = *"the response remains fresh until N seconds after the response is generated."* `s-maxage=N` overrides `max-age` for shared caches only.

**Freshness (§4.2, in brief).** A response is fresh while `age < freshness_lifetime`. The freshness lifetime is derived, in priority order, from: `s-maxage` (shared caches), else `max-age`, else `Expires` minus `Date`, else a heuristic (a common heuristic is 10% of the time since `Last-Modified`). `age` is measured from response generation and counts transit time and time spent in intermediate caches. *"If an origin server wishes to force a cache to validate every request, it can assign an explicit expiration time in the past."*

**Example header lines:**
```
Cache-Control: no-store
Cache-Control: private, no-store
Cache-Control: max-age=0, must-revalidate
ETag: "v1001"
```

---

### Item 4 — RFC 8288, Web Linking (Link header)
Standards Track, October 2017 (M. Nottingham); obsoletes RFC 5988.

**Serialization grammar (§3).** Each link-value carries one target URI-Reference in angle brackets, followed by `;`-delimited parameters: `<URI-Reference>; param=value; param=value`. *"If the URI-Reference is relative, parsers MUST resolve it as per [RFC3986], Section 5"* — and *"any base URI carried in the payload body is NOT used."* Multiple links are comma-separated within one field-line; a header may also be split across multiple `Link:` lines.

**`rel`/`rev` value ABNF (§3.3, verbatim):**
```
relation-type *( 1*SP relation-type )
where:
relation-type = reg-rel-type / ext-rel-type
reg-rel-type  = LOALPHA *( LOALPHA / DIGIT / "." / "-" )
ext-rel-type  = URI ; Section 3 of [RFC3986]
```
*"Note that extension relation types are REQUIRED to be absolute URIs in Link header fields and MUST be quoted when they contain characters not allowed in tokens, such as a semicolon (';') or comma (',')."* The `rel` parameter *"MUST NOT appear more than once in a given link-value; occurrences after the first MUST be ignored."* `rev` is deprecated.

**Quoting rules.** Parameter values may be tokens or quoted-strings. By convention `title` is quoted and `hreflang` is a token; *"Senders wishing to maximize interoperability will send them in those forms."* Established link-params: `rel`, `anchor`, `rev` (link model) plus `hreflang`, `media`, `title`, `title*`, `type` (serialisation target attributes).

**API-relevant relation types.** Pagination: `next`, `prev`, `first`, `last`. Also `self`, `describedby`, `deprecation`, `sunset`.

**Pagination Link line:**
```
Link: <https://api.example.com/items?page=3>; rel="next",
      <https://api.example.com/items?page=1>; rel="first",
      <https://api.example.com/items?page=50>; rel="last",
      <https://api.example.com/items?page=1>; rel="prev"
```

**Deprecation/sunset Link line:**
```
Link: <https://api.example.com/deprecation>; rel="deprecation"; type="text/html",
      <https://api.example.com/sunset-policy>; rel="sunset"; type="text/html"
```

---

### Item 5 — RFC 7396 (JSON Merge Patch) and RFC 6902 (JSON Patch, with RFC 6901 pointers)

**RFC 7396 — JSON Merge Patch** (Standards Track, October 2014; Hoffman, Snell). Media type `application/merge-patch+json`.

Algorithm (§2, verbatim pseudocode):
```
define MergePatch(Target, Patch):
  if Patch is an Object:
    if Target is not an Object:
      Target = {} # Ignore the contents and set it to an empty Object
    for each Name/Value pair in Patch:
      if Value is null:
        if Name exists in Target:
          remove the Name/Value pair from Target
      else:
        Target[Name] = MergePatch(Target[Name], Value)
    return Target
  else:
    return Patch
```
Rules: members present in the patch are added or replace existing values; **`null` in the patch deletes** the corresponding member; **arrays are replaced wholesale** (no element-wise merge — a non-object patch replaces the whole target). **Explicit-null limitation:** because `null` means delete, merge patch *cannot* set a member's value to `null` — use JSON Patch when that is required.

Before / patch / after (§ example, verbatim):
```
Original:  { "a": "b", "c": { "d": "e", "f": "g" } }
Patch:     { "a": "z", "c": { "f": null } }
Result:    { "a": "z", "c": { "d": "e" } }
```

**RFC 6902 — JSON Patch** (Standards Track, April 2013; Bryan, Nottingham). Media type `application/json-patch+json`. A patch is a JSON array of operation objects. Paths are JSON Pointers (RFC 6901): `/`-separated tokens; `~1` escapes `/`, `~0` escapes `~`; `-` denotes the array-append position.

Six operations (each op object *"MUST have exactly one 'op' member"* and *"exactly one 'path' member"*):
- **`add`** — `{"op":"add","path":"/a/b","value":X}` — adds a member or inserts into an array.
- **`remove`** — `{"op":"remove","path":"/a/b"}` — target MUST exist.
- **`replace`** — `{"op":"replace","path":"/a/b","value":X}` — target MUST exist.
- **`move`** — `{"op":"move","from":"/a","path":"/b"}` — *"The 'from' location MUST NOT be a proper prefix of the 'path' location; i.e., a location cannot be moved into one of its children."*
- **`copy`** — `{"op":"copy","from":"/a","path":"/b"}`.
- **`test`** — `{"op":"test","path":"/a","value":X}` — *"The target location MUST be equal to the 'value' value for the operation to be considered successful"* (equality is type-specific: strings by code point, numbers by numeric value, arrays/objects by recursive membership).

Atomicity & error behavior (§5): operations apply in sequence; *"if any of them fail then the whole patch operation should abort"* and the target document is left unchanged. Invalid `op` values, missing required targets, and failed `test` operations are all errors.

Verbatim patch on a sample target (optimistic-concurrency pattern):
```
Target: { "version": 5, "data": "important stuff" }

Patch:
[
  { "op": "test", "path": "/version", "value": 5 },
  { "op": "replace", "path": "/data", "value": "updated stuff" },
  { "op": "replace", "path": "/version", "value": 6 }
]

Result: { "version": 6, "data": "updated stuff" }
```
If `/version` is not 5 at apply time, the `test` fails and none of the three operations are applied.

---

### Item 6 — RFC 3339 (with RFC 9557 refinement), date-time
RFC 3339 Standards Track, July 2002 (Klyne, Newman) — a profile of ISO 8601. RFC 9557 (Internet Extended Date/Time Format, IXDTF), Standards Track, April 2024 (Sharma, Bormann), updates it.

Grammar (§5.6, verbatim):
```
date-fullyear  = 4DIGIT
date-month     = 2DIGIT   ; 01-12
date-mday      = 2DIGIT   ; 01-28, 01-29, 01-30, 01-31 based on month/year
time-hour      = 2DIGIT   ; 00-23
time-minute    = 2DIGIT   ; 00-59
time-second    = 2DIGIT   ; 00-58, 00-59, 00-60 based on leap-second rules
time-secfrac   = "." 1*DIGIT
time-numoffset = ("+" / "-") time-hour ":" time-minute
time-offset    = "Z" / time-numoffset
partial-time   = time-hour ":" time-minute ":" time-second [time-secfrac]
full-date      = date-fullyear "-" date-month "-" date-mday
full-time      = partial-time time-offset
date-time      = full-date "T" full-time
```

Offset rules: `Z` = zero offset. `+00:00` implies *"that UTC is the preferred reference point for the specified time."* `-00:00` originally meant *"the time in UTC is known, but the offset to local time is unknown."*

**RFC 9557 refinement.** It *"updates RFC 3339 in the specific interpretation of the local offset Z, which is no longer understood to 'imply that UTC is the preferred reference point.'"* `Z` now carries the same "offset to local time unknown" meaning `-00:00` had; because ISO 8601:2000+ disallows `-00:00`, *"Z is preferred."* The `+00:00` semantics are unchanged. RFC 9557 also adds an optional IXDTF suffix, e.g. `2022-07-08T00:14:07Z[Europe/London]`, made critical with `!` (e.g. `[!Europe/London]`).

Fractional seconds: optional, arbitrary digit count; RFC 3339 §5.3 calls this *"the only one rarely used option."*

Valid / invalid examples:

| Value | Status | Reason |
|---|---|---|
| `1996-12-19T16:39:57-08:00` | Valid | numeric offset form |
| `1985-04-12T23:20:50.52Z` | Valid | fractional seconds + `Z` |
| `1990-12-31T23:59:60Z` | Valid | leap second (`time-second` = 60) |
| `2024-03-23T13:59:30Z` | Valid | UTC |
| `2024-03-23 13:59:30Z` | RFC 3339 valid (§5.6 NOTE permits a space instead of `T` by mutual agreement); many parsers reject | space separator |
| `2024-13-01T00:00:00Z` | Invalid | month > 12 |
| `2024-03-23T24:00:00Z` | ISO-8601-valid, **RFC-3339-invalid** | ISO 8601 permits hour 24; RFC 3339 restricts to 00-23 |
| `2024-03-23T13:59:30` | Invalid | missing required `time-offset` |
| `20240323T135930Z` | ISO-8601-valid (basic format), **RFC-3339-invalid** | RFC 3339 requires the `-`/`:` separators |
| `2024-03-23T13:59:30+5:00` | Invalid | offset hour must be 2 digits |

---

### Item 7 — RFC 8594 (Sunset) + RFC 9745 (Deprecation)

**RFC 8594, Sunset** — Informational, May 2019 (E. Wilde). Value is an **HTTP-date** (RFC 7231 IMF-fixdate). Defines the `sunset` link relation. Applies to the *decommission* stage (when the resource becomes unresponsive), not the "no longer preferred" stage. *"Clients SHOULD treat Sunset timestamps as hints."*
```
Sunset: Sat, 31 Dec 2025 23:59:59 GMT
```

**RFC 9745, Deprecation** — Standards Track, March 2025 (S. Dalal, E. Wilde). Value is an **RFC 9651 structured-field Date**: a bare Integer prefixed with `@`, denoting Unix time in seconds. A past value = already deprecated; a future value = will be deprecated. Defines the `deprecation` link relation — *"Refers to documentation (intended for human consumption) about the deprecation of the link's context."* The header is a hint and does not change resource behavior.

**Combining the two (RFC 9745, verbatim).** *"The timestamp given in the Sunset HTTP header field MUST NOT be earlier than the one given in the Deprecation header field."* And: *"for historical reasons the Sunset HTTP header field uses a different data format for date."* The spec's own combined example (deprecated Fri 30 Jun 2023 23:59:59 UTC; sunset Sun 30 Jun 2024 23:59:59 UTC):
```
Deprecation: @1688169599
Sunset: Sun, 30 Jun 2024 23:59:59 UTC
```

**Combined example — deprecated in the past, sunset in the future, with a deprecation link:**
```
Deprecation: @1688169599
Sunset: Sat, 31 Dec 2026 23:59:59 GMT
Link: <https://api.example.com/deprecation>; rel="deprecation"; type="text/html"
```

---

### Item 8 — RateLimit header fields ⚠️ DRAFT / UNSTABLE
**draft-ietf-httpapi-ratelimit-headers-11** — active Internet-Draft (httpapi WG). Published 23 May 2026; **expires 24 November 2026**; Intended Status: Standards Track. Authors: **Roberto Polli** (Team Digitale, Italian Government), **Alejandro Martinez Ruiz** (Red Hat), **Darrel Miller** (Microsoft). **This is work in progress and MUST NOT be cited as stable.** Both fields are Structured Field **Lists** (RFC 9651). This revision consolidated the older `RateLimit-Limit` / `RateLimit-Remaining` / `RateLimit-Reset` triplet into the two structured fields below — do not assume the old names.

**`RateLimit-Policy` (§3).** A non-empty List of policy items; each item's value is a String (the policy name). Parameters:
- **`q`** (REQUIRED, Integer) — the quota allocated by this policy, measured in quota units.
- **`qu`** (OPTIONAL, String) — the quota unit; default `"requests"`; also `"content-bytes"`, `"concurrent-requests"`.
- **`w`** (OPTIONAL, Integer) — the time window in seconds.
- **`pk`** (OPTIONAL, Byte Sequence) — the partition key.

**`RateLimit` (§4).** Each item's value is a String identifying the policy the service-limit is reported for. Parameters:
- **`r`** (REQUIRED, Integer) — the currently available (remaining) quota, in quota units.
- **`t`** (OPTIONAL, Integer) — the effective window in seconds (time within which no more than the available quota may be used).
- **`pk`** (OPTIONAL, Byte Sequence) — the partition key.

**Partition keys.** Represented as a Structured-Fields Byte Sequence — base64 wrapped in colons: `pk=:<base64>:`. (E.g. `pk=:dHJpYWwxMjEzMjM=:` decodes to `trial121323`; `pk=:QXBwLTk5OQ==:` decodes to `App-999`.) `pk` is the single partition-key parameter on both fields; there are no partition-key sub-parameters. Other parameters are permitted and *"can be regarded as comments"*; vendor params SHOULD be prefixed.

**Verbatim example header lines (draft-11, §§3–4 and Appendix B):**
```
RateLimit-Policy: "burst";q=100;w=60,"daily";q=1000;w=86400
RateLimit-Policy: "peruser";q=100;w=60;pk=:cHsdsRa894==:
RateLimit-Policy: "peruser";q=65535;qu="content-bytes";w=10;pk=:sdfjLJUOUH==:
RateLimit: "default";r=50;t=30
RateLimit: "default";r=999;pk=:dHJpYWwxMjEzMjM=:
RateLimit: "default";r=300000000;t=60;pk=:QXBwLTk5OQ==:
```

---

### Item 9 — Idempotency-Key header ⚠️ EXPIRED DRAFT / UNSTABLE
**draft-ietf-httpapi-idempotency-key-header-07** — **EXPIRED** Internet-Draft (httpapi WG), status now shown as *"This Internet-Draft is no longer active."* Published 15 October 2025; **expired 18 April 2026**; Intended Status was Standards Track. Authors: **Jayadeba Jena** (PayPal) and **Sanjay Dalal**. **This document is expired and unstable.**

**Syntax (§2.1, verbatim).** *"Idempotency-Key is an Item Structured Header [RFC8941]. Its value MUST be a String (Section 3.3.3 of [RFC8941])."*
```
Idempotency-Key: "8e03978e-40d5-43e8-bc93-6894a57f9324"
```
A UUID (RFC 4122) or similar random identifier is RECOMMENDED. The key MUST be unique and MUST NOT be reused with a different request payload.

**Server behaviors (as drafted).**
- *Fingerprinting (§2.4):* a server MAY compute an idempotency fingerprint from the payload — checksum of the entire payload, checksum of selected elements, per-field value match, or a request digest/signature.
- *Enforcement (§2.6):* first-time request (key + fingerprint unseen) → process normally. Duplicate after the original completed (a **retry**) → *"respond with the result of the previously completed operation, success or an error."* Duplicate while the original is still in flight (a **concurrent request**) → *"respond with a resource conflict error."*
- *Error handling (§2.7):* missing key on a documented idempotent operation → **400 Bad Request**; reuse of a key with a *different* payload → **422 Unprocessable Content**; retry while the original is still processing → **409 Conflict**. Error bodies use `application/problem+json` or an informative `Link; rel="describedby"`.

**Request / replay exchange:**
```
POST /v1/payments HTTP/1.1
Host: api.example.com
Idempotency-Key: "8e03978e-40d5-43e8-bc93-6894a57f9324"
Content-Type: application/json

{"amount": 50, "currency": "USD"}

HTTP/1.1 201 Created
Content-Type: application/json

{"id": "pay_123", "status": "succeeded"}
```
Replay — same key, same payload, after completion — returns the stored result:
```
POST /v1/payments HTTP/1.1
Host: api.example.com
Idempotency-Key: "8e03978e-40d5-43e8-bc93-6894a57f9324"
Content-Type: application/json

{"amount": 50, "currency": "USD"}

HTTP/1.1 201 Created
Content-Type: application/json

{"id": "pay_123", "status": "succeeded"}
```
Concurrent (in-flight) conflict — §2.7 verbatim body:
```
HTTP/1.1 409 Conflict
Content-Type: application/problem+json
Content-Language: en
{
  "type": "https://developer.example.com/idempotency",
  "title": "A request is outstanding for this Idempotency-Key",
  "detail": "A request with the same Idempotency-Key for the
  same operation is being processed or is outstanding.",
}
```

---

### Item 10 — OpenAPI 3.1 mechanics that matter for a standard
Version 3.1.x aligns Schema Objects with **JSON Schema Draft 2020-12** ("Any valid JSON Schema 2020-12 document is now a valid OpenAPI 3.1 schema"). Required top-level fields are `openapi` and `info`; the document must additionally contain at least one of `paths`, `webhooks`, or `components`.

**`jsonSchemaDialect`** — top-level string declaring the default schema dialect for all Schema Objects; its schema default is `https://spec.openapis.org/oas/3.1/dialect/base`. To pin plain JSON Schema 2020-12:
```yaml
openapi: 3.1.0
info:
  title: My API
  version: 1.0.0
jsonSchemaDialect: https://json-schema.org/draft/2020-12/schema
paths: {}
```

**Type arrays replace `nullable`.** The OpenAPI-specific `nullable` keyword is removed; add `"null"` to a JSON Schema type array:
```yaml
# OpenAPI 3.0:  {type: string, nullable: true}
# OpenAPI 3.1:
email:
  type: [string, "null"]
  description: Can be a string or null
```

**`webhooks` top-level object** — keys are webhook names; values are Path Item Objects (replaces the 3.0 `x-webhooks` extension pattern):
```yaml
webhooks:
  newPet:
    post:
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/Pet'
      responses:
        '200':
          description: Webhook received
```

---

### Item 11 — JSON:API 1.1 shapes (comparison material)
Media type `application/vnd.api+json`. Top-level rules: `data` and `errors` MUST NOT coexist; a document MAY contain `jsonapi`, `links`, `included`, `meta`; `included` MUST NOT be present if there is no top-level `data`. Resource objects always require `type` and (except client-generated creates) `id`.

**Minimal document — top-level envelope, one resource object, one `included` entry** (verbatim from jsonapi.org/examples):
```
HTTP/1.1 200 OK
Content-Type: application/vnd.api+json

{
  "data": [{
    "type": "articles",
    "id": "1",
    "attributes": {
      "title": "JSON:API paints my bikeshed!",
      "body": "The shortest article. Ever.",
      "created": "2015-05-22T14:56:29.000Z",
      "updated": "2015-05-22T14:56:28.000Z"
    },
    "relationships": {
      "author": {
        "data": {"id": "42", "type": "people"}
      }
    }
  }],
  "included": [
    {
      "type": "people",
      "id": "42",
      "attributes": {
        "name": "John",
        "age": 80,
        "gender": "male"
      }
    }
  ]
}
```

**Error object** — error objects go in a top-level `errors` array; every member is optional (`id`, `links`, `status`, `code`, `title`, `detail`, `source.pointer`/`source.parameter`, `meta`). `status` is a string. Verbatim:
```
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/vnd.api+json

{
  "errors": [
    {
      "status": "422",
      "source": { "pointer": "/data/attributes/firstName" },
      "title": "Invalid Attribute",
      "detail": "First name must contain at least two characters."
    }
  ]
}
```

---

## 3. EXAMPLES APPENDIX (all verbatim artifacts, grouped by standard)

**RFC 9457 — Problem Details**
```
// Minimal
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json

{ "type": "about:blank", "title": "Bad Request", "status": 400 }

// With extensions (verbatim §3)
HTTP/1.1 403 Forbidden
Content-Type: application/problem+json
Content-Language: en

{
 "type": "https://example.com/probs/out-of-credit",
 "title": "You do not have enough credit.",
 "detail": "Your current balance is 30, but that costs 50.",
 "instance": "/account/12345/msgs/abc",
 "balance": 30,
 "accounts": ["/account/12345",
              "/account/67890"]
}
```

**RFC 9110 — Conditional requests**
```
// Cached read → 304
GET /config.json HTTP/1.1
If-None-Match: "v1001"

HTTP/1.1 304 Not Modified
ETag: "v1001"
Cache-Control: max-age=0, must-revalidate

// Guarded write → 412
PUT /config.json HTTP/1.1
If-Match: "cfg-42"
Content-Type: application/json

{"timeout": 60, "retries": 3}

HTTP/1.1 412 Precondition Failed
```

**RFC 9111 — Caching**
```
Cache-Control: no-store
Cache-Control: private, no-store
Cache-Control: max-age=0, must-revalidate
Cache-Control: no-cache
Cache-Control: s-maxage=60
```

**RFC 8288 — Link**
```
Link: <https://api.example.com/items?page=3>; rel="next",
      <https://api.example.com/items?page=1>; rel="first",
      <https://api.example.com/items?page=50>; rel="last",
      <https://api.example.com/items?page=1>; rel="prev"

Link: <https://api.example.com/deprecation>; rel="deprecation"; type="text/html",
      <https://api.example.com/sunset-policy>; rel="sunset"; type="text/html"
```

**RFC 7396 — JSON Merge Patch**
```
Original:  { "a": "b", "c": { "d": "e", "f": "g" } }
Patch:     { "a": "z", "c": { "f": null } }
Result:    { "a": "z", "c": { "d": "e" } }
```

**RFC 6902 — JSON Patch**
```
Target: { "version": 5, "data": "important stuff" }
Patch:
[
  { "op": "test", "path": "/version", "value": 5 },
  { "op": "replace", "path": "/data", "value": "updated stuff" },
  { "op": "replace", "path": "/version", "value": 6 }
]
Result: { "version": 6, "data": "updated stuff" }
```

**RFC 3339 / 9557**
```
1996-12-19T16:39:57-08:00      (valid, offset)
1985-04-12T23:20:50.52Z        (valid, fractional + Z)
1990-12-31T23:59:60Z           (valid, leap second)
2022-07-08T00:14:07Z[Europe/London]   (RFC 9557 IXDTF suffix)
2024-03-23T24:00:00Z           (ISO 8601 valid; RFC 3339 INVALID)
20240323T135930Z               (ISO 8601 basic; RFC 3339 INVALID)
```

**RFC 8594 / 9745 — Sunset + Deprecation**
```
Deprecation: @1688169599
Sunset: Sun, 30 Jun 2024 23:59:59 UTC

// deprecated past, sunset future, with link
Deprecation: @1688169599
Sunset: Sat, 31 Dec 2026 23:59:59 GMT
Link: <https://api.example.com/deprecation>; rel="deprecation"; type="text/html"
```

**RateLimit (draft-11)**
```
RateLimit-Policy: "burst";q=100;w=60,"daily";q=1000;w=86400
RateLimit-Policy: "peruser";q=100;w=60;pk=:cHsdsRa894==:
RateLimit-Policy: "peruser";q=65535;qu="content-bytes";w=10;pk=:sdfjLJUOUH==:
RateLimit: "default";r=50;t=30
RateLimit: "default";r=999;pk=:dHJpYWwxMjEzMjM=:
RateLimit: "default";r=300000000;t=60;pk=:QXBwLTk5OQ==:
```

**Idempotency-Key (draft-07, EXPIRED)**
```
Idempotency-Key: "8e03978e-40d5-43e8-bc93-6894a57f9324"

HTTP/1.1 409 Conflict
Content-Type: application/problem+json
Content-Language: en
{
  "type": "https://developer.example.com/idempotency",
  "title": "A request is outstanding for this Idempotency-Key",
  "detail": "A request with the same Idempotency-Key for the
  same operation is being processed or is outstanding.",
}
```

**OpenAPI 3.1**
```yaml
openapi: 3.1.0
info: { title: My API, version: 1.0.0 }
jsonSchemaDialect: https://json-schema.org/draft/2020-12/schema
paths: {}
# nullable → type array
email:
  type: [string, "null"]
# webhooks
webhooks:
  newPet:
    post:
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/Pet' }
      responses:
        '200': { description: Webhook received }
```

**JSON:API 1.1**
```
{ "data": [{ "type":"articles","id":"1",
  "attributes":{...},
  "relationships":{"author":{"data":{"id":"42","type":"people"}}} }],
  "included":[{ "type":"people","id":"42","attributes":{...} }] }

{ "errors":[{ "status":"422",
  "source":{"pointer":"/data/attributes/firstName"},
  "title":"Invalid Attribute",
  "detail":"First name must contain at least two characters." }] }
```

---

## 4. Corrections to Part 1
**None.** All Part 1 status facts are confirmed by the primary texts:
- RFC 9110 (STD 97) and RFC 9111 (STD 98) are Internet Standards published June 2022.
- RFC 9457 obsoleted RFC 7807 in July 2023.
- RFC 9745 (Deprecation) published March 2025, using an RFC 9651 structured-field Date in `@<unix>` form.
- RateLimit is at draft-11 (published 23 May 2026, expires 24 Nov 2026).
- Idempotency-Key draft-07 expired April 2026 — exact expiry 18 April 2026; the Datatracker status is now "no longer active."

*(Clarifying note, not a correction: Part 1's "June 2022 Internet Standards" for 9110/9111 are more precisely STD 97 and STD 98 respectively.)*

---

## 5. Caveats — draft instability, errata, ambiguities
- **Draft instability.** RateLimit (draft-11) and Idempotency-Key (draft-07) are Internet-Drafts and are explicitly *"work in progress"* — field names and semantics may still change and MUST NOT be cited as stable. RateLimit draft-11 consolidated the older `RateLimit-Limit` / `RateLimit-Remaining` / `RateLimit-Reset` triplet into the two structured fields `RateLimit-Policy` and `RateLimit`; any standard relying on the old triplet is out of date. Idempotency-Key draft-07 is **EXPIRED** (18 April 2026); a prescriptive standard adopting it must either wait for republication or freeze against a specific revision and note the risk.
- **RFC 3339 errata.** Verified Errata (ID 4110) note that the Appendix A ABNF should read `time-hour = 2DIGIT ; 00-23` (the original said `00-24`); the main-body prose already restricts the hour to 00-23, so conforming implementations are unaffected.
- **RFC 9557 vs RFC 3339 tension.** The reinterpretation of `Z` (to mean "offset to local time unknown," matching the old `-00:00`) is an update effective April 2024; implementations predating it may still follow the original `+00:00`/`-00:00` distinction. RFC 9557 explicitly notes it *"does not formally deprecate"* the `-00:00` syntax.
- **JSON Merge Patch explicit-null limitation.** Merge patch cannot set a member to `null` (null means delete). Where APIs need to distinguish "clear this field" (`null`) from "leave unchanged" (omit) — e.g., PATCH semantics — use RFC 6902 JSON Patch instead. (OpenAPI's optional-vs-nullable modeling captures the same three-state distinction.)
- **RFC 8594 status asymmetry.** Sunset is **Informational**, whereas its companion Deprecation (RFC 9745) is **Standards Track** — a governance asymmetry worth noting when mandating both.
- **RFC 9457 multiple-problem gap.** There is no standardized multi-problem container; the "most relevant/urgent problem" guidance is a SHOULD, and same-type multi-error arrays (e.g., `errors` with `pointer`) are conventions demonstrated in examples, not normatively named members — a prescriptive standard should define its own extension shape explicitly.
- **RFC 8288 quoting nuance.** Extension (URI) relation types MUST be quoted when they contain `;` or `,`; the token vs quoted-string equivalence for `title`/`hreflang` is interoperable but historically inconsistent, so senders should follow the conventional forms.