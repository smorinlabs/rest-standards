# REST API Conventions Series — Part 4/7: Collections (Pagination, Filtering, Sorting, Selection & Expansion)

*Descriptive comparison of eight references. All findings retrieved July 19, 2026. Primary sources: official API references, google.aip.dev (AIP-132/158/160), Microsoft Learn (OData/Graph) and the Azure guidelines repo, shopify.dev, jsonapi.org, and the Zalando guidelines repo; secondary sources used only for corroboration.*

## 1. TL;DR
- The field genuinely splits on **three axes**: pagination style (ID-cursor vs opaque-token cursor vs offset/page-number), where the "next" signal lives (body field vs RFC 8288 `Link` header), and filter-syntax family (structured query DSL vs per-field params). It is near-consensus on two points: opaque, non-constructable cursors are preferred at scale, and totals are usually withheld or capped.
- Stripe is **not canonical** on this surface: its object-ID cursors (`starting_after`/`ending_before`), its reliance on `expand[]` in place of field selection, and its separate Search query language are idiosyncratic. Google AIP, Microsoft/OData, and JSON:API are the more complete descriptive baselines.
- Shopify is the key at-scale cursor reference (opaque `page_info` in a `Link` header) but its REST Admin API is **legacy as of October 1, 2024**; document it as a still-operational but frozen pattern.

## 2. Key Findings
1. **Cursor pagination is the majority at-scale posture**, implemented three ways: Stripe uses object IDs as cursors (`starting_after`/`ending_before`); Google AIP-158, Microsoft Graph, and AWS use opaque server tokens (`page_token`/`$skiptoken`/`NextToken`); Shopify embeds an opaque base64 `page_info` in a `Link` header. Offset/page-number survives in GitHub (`page`), Twilio (`Page`), Zalando (allowed but discouraged), and Microsoft's `$skip`.
2. **The "next" signal splits body-vs-header.** Body-field camp: Stripe (`has_more`), Google (`next_page_token`), Microsoft Graph (`@odata.nextLink`), AWS (`NextToken`), Twilio (`next_page_uri`/`next_page_url`). Header camp (RFC 8288 `Link`): GitHub and Shopify. Zalando explicitly permits both.
3. **Totals are usually withheld at scale.** Microsoft/OData offers opt-in `$count=true`; Google AIP allows optional `total_size` (may be an estimate). Stripe exposes `total_count` **only on Search** (≤10,000) — and, per its 2025-03-31 changelog, removed `total_count` expansion from **list** methods entirely. GitHub list endpoints, Shopify REST, Twilio, and AWS expose no total by design.
4. **Filter syntax splits into two families:** structured query DSLs (Microsoft OData `$filter`, Google AIP-160, Stripe Search `query=`, GitHub `q=`) versus per-field/bracket params (Stripe list `created[gte]=`, Shopify per-field, Twilio `DateSent`, AWS `Filters`). Zalando documents both and prefers the simple family unless complexity demands otherwise.
5. **Field selection (sparse fieldsets) is not universal.** OData `$select`, JSON:API `fields[type]`, AIP field masks/`view`, Shopify `fields`, and Zalando `fields` support it; **Stripe offers none** — it relies on `expand[]` for the inverse operation. GitHub, Twilio, and AWS list endpoints generally return fixed shapes.
6. **Expansion mechanisms diverge sharply:** Stripe `expand[]` (max depth 4), Microsoft `$expand`, JSON:API `include`, AIP resource views/`view` enum, Zalando `embed`. The default posture is reference-by-ID/URI almost everywhere; embedding is opt-in.

## 3. Baseline position — RFC 8288, JSON:API, and the guideline docs

**RFC 8288 (Web Linking)** defines the `Link` header carrying typed relations. Canonical form: `<url>; rel="next"`, with registered relation types `next`, `prev`, `first`, `last`. The multi-link form is `<...>; rel="next", <...>; rel="last"`. GitHub and Shopify use this for pagination.

**JSON:API v1.1** (jsonapi.org/format) reserves three query-parameter families and puts pagination links in the body:
- **Pagination:** the `page` family, strategy-agnostic — `page[number]`/`page[size]` for page-based, `page[cursor]` for cursor. Links live in a top-level `links` object with keys `first`, `last`, `prev`, `next` (MUST be omitted or null when unavailable).
- **Sorting:** `sort` — comma-separated fields, `-` prefix for descending (`sort=-created,title`); server MAY apply a default order.
- **Sparse fieldsets:** `fields[TYPE]=a,b` (e.g. `fields[articles]=title,body`).
- **Include (expansion):** `include=author,comments.author` (dot notation for nested).
- **Filter:** `filter` family reserved but syntax left to the server.
- Example: `GET /articles?include=author&fields[articles]=title,body&fields[people]=name`.

**Zalando RESTful API Guidelines** (opensource.zalando.com/restful-api-guidelines) — a guideline document, not an API. Reserved query-parameter names: `q` (default search), `sort`, `fields`, `embed`, `offset`, `cursor`, `limit`. Sort syntax: comma-separated list, `+`/`-` prefix for direction (`/sales-orders?sort=+id`). It **"SHOULD prefer cursor-based pagination, avoid offset-based pagination,"** and defines a `ResponsePage` object with `self`, `first`, `prev`, `next`, `last` (each `uri|cursor`), a `query` object, and an `items` array. Cursors "must never be inspected or constructed by clients" and encode position (`modified_at`, `id`), direction, and a `query_hash`. For GET-with-body search it recommends carrying filters in the body while keeping `cursor`/`limit` in query params, protecting the sequence with a hash over filters. On filtering it distinguishes minimalistic per-field query params (AND-only) from sophisticated query languages for search/catalog use cases: **"Simple query languages are generally preferred over complex ones."**

## 4. Side-by-side comparison tables

### 4a. Pagination mechanics
| Reference | Style | Page-size param | Default / Max | "Next" signal | first/prev/last | Cursor opacity |
|---|---|---|---|---|---|---|
| Stripe | ID-cursor | `limit` | 10 / 100 | Body `has_more` (bool) | prev via `ending_before` only | Cursor = object ID (transparent) |
| GitHub REST | page-number + newer cursor (`before`/`after`) | `per_page` | 30 / 100 | `Link` header (RFC 8288) | next/prev/first/last in `Link` | Opaque `after`/`before` on cursor endpoints |
| Google AIP-158 | opaque token | `page_size` | doc'd (proto example 50 / 1000) | Body `next_page_token` | none defined (forward-only) | Opaque, URL-safe, ~3-day TTL |
| Microsoft Graph | opaque token (`$skiptoken`) or `$skip` | `$top` | per-API | Body `@odata.nextLink` | none (forward-only) | Opaque; don't parse |
| Shopify REST | opaque cursor in `Link` | `limit` | 50 / 250 | `Link` header | next/previous only | Opaque base64 `page_info` |
| Twilio | page-number + `PageToken` cursor | `PageSize` | 50 / 1000 | Body `next_page_uri`/`next_page_url` | first/prev/next (no last) | Opaque `PageToken` |
| Zalando (guideline) | cursor preferred; offset allowed | `limit` | not prescribed | body links or `Link` header | self/first/prev/next/last | Opaque, encrypted, `query_hash` |
| AWS | opaque token | `MaxResults` | per-service (e.g. 50/50; 100/1000) | Body `NextToken` | none (forward-only) | Opaque; 24h TTL on some services |

### 4b. Totals, filter & sort syntax
| Reference | Total count | Filter syntax | Sort syntax | Multi-key sort |
|---|---|---|---|---|
| Stripe | `total_count` on Search only, ≤10,000; removed from list expansion (2025-03-31) | list: per-field + `created[gte]`; Search: `query=` DSL | not exposed on list (implicit created desc) | No |
| GitHub | `total_count` on Search endpoints only | `q=` qualifier DSL (search endpoints) | `sort=` + `order=asc\|desc` | No |
| Google AIP | optional `total_size` (may be estimate, must be documented) | AIP-160 filter DSL (`filter=`) | `order_by` string, comma-sep, ` desc` suffix | Yes |
| Microsoft/OData | opt-in `$count=true` → `@odata.count` | `$filter` OData expression | `$orderby` + `asc`/`desc` | Yes (comma-sep) |
| Shopify REST | none | per-field params (first request only) | `order` param (field + direction) | Limited |
| Twilio | none (removed 2015) | per-field params (`DateSent`, `To`, `From`) | none (fixed per-resource) | No |
| Zalando | not prescribed | `q` + per-field or JSON query lang | `sort` + `+`/`-` prefix | Yes |
| AWS | none | `Filters` (Name/Values pairs) | rarely; service-specific | Rare |

### 4c. Selection & expansion
| Reference | Field selection | Expansion / embedding | Default posture | Depth control |
|---|---|---|---|---|
| Stripe | **None** | `expand[]`, incl. `data.` prefix for lists | reference-by-ID | max depth 4 |
| GitHub | None (fixed) | None (some nested objects embedded by default) | mixed | n/a |
| Google AIP | field masks (`read_mask`) / `view` enum | resource views (`BASIC`/`FULL`) | reference-by-name | n/a |
| Microsoft/OData | `$select` | `$expand` (nested `$select`/`$filter` allowed) | reference | `$expand` ≤20 items on directory objects |
| Shopify REST | `fields` param (comma-sep) | none in REST | reference | n/a |
| Twilio | None | `subresource_uris` (links, not embeds) | reference-by-URI | n/a |
| JSON:API | `fields[type]` | `include` (dot-nested) | reference | server-limited |
| Zalando | `fields` | `embed` | reference | "do with care" |

## 5. Per-reference notes (collections character sketches)

**Stripe.** All top-level list resources share `limit`/`starting_after`/`ending_before`; the response envelope is `{ "object":"list", "data":[...], "has_more":bool, "url":"..." }`. `limit` ranges 1–100, default 10. Cursors are object IDs, not opaque tokens — a deliberate choice ("we want `starting_after` to be very obviously a *pagination parameter* as opposed to a *filter*"). Objects sort newest-first by creation time; there is no `previous_page_uri`, so back-paging uses `ending_before`. List filtering is minimal: per-field params plus range syntax on timestamp fields — `created[gte]`, `created[gt]`, `created[lte]`, `created[lt]` (also a `gte/gt/lte/lt` dictionary form). Example: `GET /v1/charges?created[gte]=1710000000&created[lte]=1710086400&limit=10`. A **separate Search API** (`/v1/charges/search`, minimum API version 2020-08-27, rate-limited to 20 read ops/sec) uses `query=` in Stripe's Search Query Language: `field:value` clauses, `:` exact match, substring on some fields (email, name), numeric `>`/`<` on amount, negation via `-`, AND/OR but **no parentheses**, combined with AND by default. Example: `query=amount>999 AND metadata['order_id']:'6735'`. Search paginates with `page`/`next_page` (not `starting_after`) and exposes `total_count` (accurate to 10,000). Per Stripe's changelog dated 2025-03-31, `total_count` expansion was **removed from list methods** — attempting it now returns a `400 property_forbidden`; it survives only on Search. Search is explicitly eventually consistent: "Don't use search in read-after-write flows... data is searchable in less than a minute... up to an hour behind during outages." Stripe has **no field selection**; instead `expand[]` opts specific reference fields into full objects (`expand[]=customer`; nested `expand[]=data.customer` on lists). Per docs.stripe.com/api/expanding_objects: "Expansions have a maximum depth of four levels (for example, the deepest expansion allowed when listing charges is `data.payment_intent.customer.default_source`)."

**GitHub REST.** Paginates via the RFC 8288 `Link` header; page size via `per_page` (max 100; default 30 for most endpoints). Two families coexist: page-number (`page`) and cursor (`before`/`after`), and several high-volume endpoints (repository activity, issues) now require cursor pagination — a `page` request there returns HTTP 422 ("Pagination with the page parameter is not supported for large datasets, please use cursor based pagination (after/before)"). Example header: `Link: <https://api.github.com/repositories/189621607/activity?per_page=1&page=10&after=djE6ks8AAAADp2rKWQA>; rel="next"`. No body-level next signal; no total on list endpoints. Filtering/sorting is rich only on the **Search API** (`/search/*`): `q=` with qualifiers (`language:`, `stars:>`, `created:`, `in:`), `sort=` (stars/forks/updated/created) + `order=asc|desc`, default "best match." Search caps at 1,000 results, 100/page, and is separately rate-limited (30 req/min authenticated; 10 req/min for code search). Example: `q=windows+label:bug+language:python+state:open&sort=created&order=asc`. No sparse-fieldset or expand mechanism.

**Google Cloud / AIP.** The most prescriptive spec on this surface. AIP-158/132: the request carries `page_size` (int32, not required; if omitted the server picks a documented default; if over max the API "should coerce down to the maximum permitted page size," not error; a negative value "must send an INVALID_ARGUMENT error") and `page_token` (opaque, URL-safe, non-parseable). The proto example states: "If unspecified, at most 50 books will be returned. The maximum value is 1000; values above 1000 will be coerced to 1000." Page tokens may expire — "a good rule of thumb is three days." The response carries `next_page_token` (empty = end-of-collection, "the only way to communicate end-of-collection") and an optional `total_size` field, which "may be an estimate (but the API should explicitly document that)." An optional `skip` int32 allows offset-into-results. **Filtering (AIP-160):** a single `filter` string in a CEL-like language: comparisons (`= != < <= > >= :`), the `:` HAS operator (substring on strings, `field:*` for existence), `AND`/`OR`/`NOT` (and `-` for NOT), traversal via `.`, and functions via `call(arg)`. Notably OR binds tighter than AND ("`a AND b OR c` evaluates `a AND (b OR c)`"); whitespace-separated literals are a fuzzy AND. **Sorting:** `order_by` string, comma-separated, ` desc`/` asc` suffix. **Selection:** field masks (`read_mask`) and resource `view` enums (e.g. `BASIC`/`FULL`).

**Microsoft (Azure guidelines + Graph).** OData vocabulary throughout. Microsoft Graph server-driven paging returns `@odata.nextLink` (a full URL embedding `$skiptoken` or `$skip`); client-side paging uses `$top`/`$skip`/`$skipToken`. Example: `"@odata.nextLink": "https://graph.microsoft.com/v1.0/users?$top=3&$skiptoken=X'44537074...'"`. Different APIs have different default/max page sizes and different over-`$top` behavior (ignore, cap, or error). `$count=true` returns `@odata.count` (first page only for directory objects). Filtering: `$filter` OData expressions with `eq/ne/gt/ge/lt/le`, `and/or/not`, functions `startswith`/`endswith`/`contains`, and lambda `any`/`all` — e.g. `$filter=(name eq 'Milk' or name eq 'Eggs') and price lt 2.55`, or `$filter=resourceProvisioningOptions/any(x:x eq 'Team')`. Sorting: `$orderby=ReleaseDate asc, Rating desc`. Selection: `$select=id,displayName`. Expansion: `$expand` with nested options — `$expand=Products($filter=DiscontinuedDate eq null;$select=...;$orderby=...)`; on Entra directory objects `$expand` returns ≤20 items with no `@odata.nextLink`. Advanced directory-object queries require header `ConsistencyLevel: eventual` + `$count=true`, else HTTP 400 `Request_UnsupportedQuery`. The **Azure REST Guidelines** (a separate design doc) prescribe `skip` (int, default/min 0), optional `top` (int, min 1, default infinity), optional `maxpagesize`, mandatory `$count=true` support, and a strict compound evaluation order: **filter → sort → paginate**. The guideline internally debates `top` vs `maxpagesize`; the current decision favors `top`.

**Twilio.** Two coexisting styles. The classic `2010-04-01` API (Messages, Calls) uses body-level `first_page_uri`, `next_page_uri`, `previous_page_uri`, `uri`, `page` (zero-indexed), `page_size`, plus a resource array (`messages`). Newer product APIs (Verify, Serverless, Proxy, Studio) wrap pagination in a `meta` object with keys `page`, `page_size`, `first_page_url`, `previous_page_url`, `next_page_url`, `url`, `key` (`key` names the resource array). Query params `PageSize` (default 50, max 1000) and `Page`; cursor via opaque `PageToken` embedded in the next URL (clients follow, don't construct). **No total count** — Twilio removed totals/number-of-pages/last-page-uri in 2015 for performance and directs users to the Usage Records API for aggregates; absolute paging (`Page=N` jumps) is deprecated. Filtering is per-field (`To`, `From`, `DateSent`, with inequality forms like `DateSent>=2017-02-28`). No field selection; related data is exposed as `subresource_uris` links. No sort controls. Newer meta example: `{"meta":{"page":0,"page_size":50,"first_page_url":"https://verify.twilio.com/v2/Services?PageSize=50&Page=0","previous_page_url":null,"next_page_url":null,"key":"services","url":"..."},"services":[]}`.

**Shopify Admin REST — KEY reference, currency-flagged.** Cursor-based pagination via the RFC 8288 `Link` header since API 2019-07; the `page` param was removed and now returns HTTP 400 ("page cannot be passed"). Page size via `limit` (max 250, default 50). Example: `Link: <https://{shop}.myshopify.com/admin/api/{version}/products.json?limit=250&page_info=eyJkaXJlY3Rpb24iOiJuZXh0IiwibGFzdF9pZCI6...>; rel="next"`. `page_info` is an opaque base64 blob encoding direction, `last_id`, `last_value`, and the original filters/sort. Critically, **once you hold a `page_info`, the only other param you may send is `limit`** — filters and sort are locked into the cursor; changing them requires restarting at page 1. `since_id` is an alternative relative cursor. No total count. Filtering is per-field and only on the first request (e.g. `created_at_min`, `collection_id`, `published_status`); range searches can time out if the sort order differs from the search field (docs recommend `/admin/orders.json?created_at_min=2020-10-21&order=created_at`). Sort via `order`; field selection via `fields` (comma-separated). **Currency caveat:** per shopify.dev/docs/api/admin-rest, "The REST Admin API is a legacy API as of October 1, 2024. Starting April 1, 2025, all new public apps must be built exclusively with the GraphQL Admin API." Product/variant REST endpoints did not receive the expanded (2,048) variant model; public apps had to migrate to GraphQL by Feb 1, 2025 (custom apps Apr 1, 2025). REST remains operational for existing apps in maintenance mode; a full sunset date is to be announced.

**Zalando (guideline).** Covered in §3. Distinctive for explicitly reserving `q`/`sort`/`fields`/`embed`/`offset`/`cursor`/`limit`, preferring cursors, permitting either body links or a `Link` header, and defining a full `ResponsePage` schema with a `query_hash` for cursor stability under GET-with-body search.

**AWS — light treatment.** Opaque `NextToken` in the response body; request `MaxResults` (per-service default/max, e.g. GuardDuty `ListFilters` default 50/max 50; Step Functions `ListActivities` default 100/max 1000). The first call omits the token; absence of `NextToken` = end. Some tokens expire (Step Functions: 24h → HTTP 400 `InvalidToken`). Filtering via a structured `Filters` array of `{Name, Values}` pairs (e.g. EC2 `instance-state-name`); `MaxResults` and `Filters` compose. Token field names vary across services (`NextToken`, `Marker`, `NextContinuationToken`). Most operations are eventually consistent; S3 list is now strong read-after-write. No sort, no field selection, no expansion. One-line contrast: AWS is the purest "opaque forward-only token + structured filter object" model.

## 6. Agreements vs divergences (descriptive)

**Agreements:**
- Opaque, non-constructable cursors are the preferred at-scale mechanism (Google, Microsoft, Shopify, AWS, Zalando). Tradeoff: no random page access, but stable under insert/delete and roughly O(1) at depth.
- Reference-by-ID/URI is the default response posture; embedding/expansion is opt-in everywhere it exists. Tradeoff: avoids N+1 payload bloat by default, at the cost of extra round-trips.
- Rich filter/sort DSLs are gated behind dedicated Search endpoints for the "simple CRUD list" references (Stripe, GitHub). Tradeoff: keeps list endpoints cheap and cacheable, but pushes eventual-consistency and cost onto search.

**Divergences with tradeoffs:**
- **Cursor semantics:** Stripe's ID cursors are transparent and client-constructable (you already know the last object's ID); everyone else's are opaque. Tradeoff: Stripe's are debuggable and resumable from any known ID; opaque tokens hide implementation and can encode filters/sort for stability but can expire.
- **Next-signal location:** body field (Stripe/Google/Microsoft/AWS/Twilio) vs `Link` header (GitHub/Shopify). Tradeoff: the header keeps the body a pure resource list and is HTTP-idiomatic (RFC 8288) but is easy to miss and harder to consume in some tooling; body fields are self-describing but couple envelope to payload.
- **Totals:** opt-in count (Microsoft `$count`, Google `total_size`, Stripe Search) vs none-by-design (GitHub list, Shopify, Twilio, AWS). Tradeoff: counts enable UI page numbers but are expensive and unstable at scale — Twilio removed them, Stripe caps at 10k and dropped list-method expansion.
- **Filter expressiveness:** full boolean DSL (OData/AIP) vs AND-only per-field params (Stripe list/Shopify/Twilio/AWS). Tradeoff: DSLs are expressive but parser-heavy and a security surface; per-field params are simple and cacheable but cannot express OR/complex logic.
- **Selection:** present (OData/JSON:API/AIP/Zalando/Shopify) vs absent (Stripe/Twilio/AWS/GitHub). Tradeoff: `$select` cuts payload but complicates caching and response typing; Stripe's inverse `expand[]` keeps a stable default shape.

## 7. CONTESTED AXES REGISTER

| Axis | Options observed | Who does what | Tradeoff (one line) | How contested |
|---|---|---|---|---|
| Cursor vs offset | ID-cursor / opaque-token cursor / offset / page-number | ID-cursor: Stripe. Opaque token: Google, MS Graph, AWS, Shopify. Page-number: GitHub, Twilio, MS `$skip`. Offset allowed: Zalando, Azure `skip` | Cursors are stable & scale; offset gives random access but drifts under writes | **Split** (cursor leads for scale; offset persists for UI paging) |
| Next-signal location | Body field / `Link` header (RFC 8288) | Body: Stripe `has_more`, Google `next_page_token`, MS `@odata.nextLink`, AWS `NextToken`, Twilio `next_page_uri`. Header: GitHub, Shopify | Header is HTTP-idiomatic but missable; body is self-describing but couples envelope | **Split** (body majority) |
| Totals exposure | Opt-in count / capped / none-by-design | Opt-in: MS `$count`, Google `total_size`, Stripe Search (≤10k). None: GitHub list, Shopify, Twilio, AWS | Counts power page UIs but are costly/unstable at scale | **Split**, leaning to omission at scale |
| Page-size defaults/max | default 10–100; max 50–1000 | Stripe 10/100; GitHub 30/100; Shopify 50/250; Twilio 50/1000; AWS per-service; Google proto 50/1000 | Higher max = fewer round-trips but bigger payloads/timeouts | **Wide-open** (no shared default) |
| Over-max behavior | Coerce down / error / ignore | Coerce: Google (AIP-158), Stripe (≤100). Error or cap: MS Graph (varies by API) | Coercing is forgiving; erroring is explicit | **Split** |
| Filter-syntax family | Structured DSL / per-field params | DSL: OData `$filter`, AIP-160, Stripe Search `query=`, GitHub `q=`. Per-field: Stripe list, Shopify, Twilio, AWS `Filters` | DSL expressive but heavy; per-field simple but AND-only | **Split** (Zalando notes both, prefers simple) |
| Sort syntax | `$orderby` / `order_by` / `sort`+`-`prefix / `sort`+`order` param / none | OData `$orderby`; AIP `order_by`; Zalando/JSON:API `sort` w/ `+`/`-`; GitHub `sort`+`order`; Stripe/Twilio none | Prefix vs separate-param vs suffix keyword — all express asc/desc | **Wide-open** (naming unstandardized) |
| Selection mechanism | `$select` / `fields[type]` / field mask / `fields` / none | OData `$select`; JSON:API `fields[type]`; AIP `read_mask`/`view`; Shopify/Zalando `fields`; none: Stripe, Twilio, AWS, GitHub | Cuts payload but complicates caching/typing | **Split** |
| Expansion mechanism | `expand[]` / `$expand` / `include` / `view`/`embed` / none | Stripe `expand[]` (depth 4); MS `$expand`; JSON:API `include`; AIP views; Zalando `embed` | Fewer round-trips vs N+1 & payload bloat | **Split** |
| Search GET-vs-POST | GET with query string / POST or GET-with-body for large queries | GET: Stripe Search, GitHub, OData, AIP. GET-with-body: Zalando (filters in body, cursor in query) | GET is cacheable/idiomatic but URL-length-limited; body handles large queries but breaks caching | **Split / near-open** |

## 8. Examples appendix (verbatim, grouped by reference)

**Stripe**
- List envelope: `{ "object":"list", "data":[...], "has_more":true, "url":"/v1/charges" }`. `limit` 1–100, default 10.
- Range filter: `GET /v1/charges?created[gte]=1710000000&created[lte]=1710086400&limit=10`. Operators: `gt`,`gte`,`lt`,`lte`.
- Search: `curl -G https://api.stripe.com/v1/charges/search --data-urlencode "query=amount>999 AND metadata['order_id']:'6735'"`; paginates with `page`/`next_page`; min API version 2020-08-27; rate limit 20 read ops/sec.
- `total_count`: "only accurate up to 10,000... isn't included by default" — Search only; list-method expansion removed 2025-03-31 (`400 property_forbidden`).
- Expand max depth 4: deepest listing charges = `data.payment_intent.customer.default_source`. Nested list expand starts with the `data.` prefix.

**GitHub**
- `Link: <https://api.github.com/repositories/189621607/activity?per_page=1&page=10&after=djE6ks8AAAADp2rKWQA>; rel="next"`
- `curl --url "https://api.github.com/repos/octocat/Spoon-Knife/issues?per_page=2"` — `per_page` max 100, default 30.
- 422 on large datasets: "Pagination with the page parameter is not supported for large datasets, please use cursor based pagination (after/before)".
- Search: `q=windows+label:bug+language:python+state:open&sort=created&order=asc`; max 1,000 results, 100/page; search rate limit 30 req/min (code search 10 req/min).

**Google AIP**
- Proto: `page_size` — "If unspecified, at most 50 books will be returned. The maximum value is 1000; values above 1000 will be coerced to 1000." Over-max coerces; negative → INVALID_ARGUMENT.
- `next_page_token` empty = end-of-collection ("the only way to communicate end-of-collection"). Optional `total_size` "may be an estimate (but the API should explicitly document that)." Page-token TTL rule of thumb ~3 days.
- AIP-160 operators: `= != < <= > >= :`; `:` = HAS (substring / `field:*` existence); `-`/`NOT`; "OR operator has higher precedence than AND... `a AND b OR c` evaluates `a AND (b OR c)`."
- Sort: `order_by` comma-separated with ` desc` suffix.

**Microsoft / OData**
- `"@odata.nextLink": "https://graph.microsoft.com/v1.0/users?$top=3&$skiptoken=X'44537074...'"`
- `$filter`: `?$filter=(name eq 'Milk' or name eq 'Eggs') and price lt 2.55`; lambda `$filter=resourceProvisioningOptions/any(x:x eq 'Team')`.
- `$orderby=ReleaseDate asc, Rating desc`; `$select=id,displayName`; `$expand=Products($filter=DiscontinuedDate eq null)`.
- Advanced directory queries need `ConsistencyLevel: eventual` + `$count=true`, else 400 `Request_UnsupportedQuery`. `$expand` ≤20 items on directory objects.
- Azure guidelines: evaluation order filter→sort→paginate; `skip` default/min 0; `top` min 1, default infinity; `maxpagesize` optional.

**Twilio**
- Classic: `{ "first_page_uri":"/2010-04-01/Accounts/AC.../Messages.json?PageSize=1&Page=0", "next_page_uri":null, "previous_page_uri":null, "page":0, "page_size":1, "uri":"...", "messages":[...] }`
- Newer meta: `{ "meta":{ "page":0, "page_size":50, "first_page_url":"https://verify.twilio.com/v2/Services?PageSize=50&Page=0", "previous_page_url":null, "next_page_url":null, "key":"services", "url":"..." }, "services":[] }`
- `PageSize` default 50, max 1000. Cursor via opaque `PageToken` in the next URL. No total count (removed 2015).
- Filter: `?DateSent>=2017-02-28`; also `To`, `From`.

**Shopify REST**
- `Link: <https://{shop}.myshopify.com/admin/api/{version}/products.json?limit=250&page_info=eyJkaXJlY3Rpb24iOiJuZXh0IiwibGFzdF9pZCI6...>; rel="next"`
- `limit` max 250, default 50. `page` param removed → 400 "page cannot be passed." After the first request only `limit` may accompany `page_info`. `since_id` alternative cursor. Sort matched to search field to avoid timeouts (`/admin/orders.json?created_at_min=2020-10-21&order=created_at`).
- Currency: "The REST Admin API is a legacy API as of October 1, 2024. Starting April 1, 2025, all new public apps must be built exclusively with the GraphQL Admin API." Product/variant public-app migration deadline Feb 1, 2025.

**AWS**
- `{ "filterNames":[...], "nextToken":"string" }`; request `?maxResults=MaxResults&nextToken=NextToken`.
- GuardDuty `ListFilters`: `MaxResults` default 50, max 50. Step Functions `ListActivities`: default 100, max 1000, token expires 24h → 400 `InvalidToken`.
- Filter object: `Filters:[{Name:"instance-state-name", Values:["stopped"]}]`. Token names vary: `NextToken`/`Marker`/`NextContinuationToken`.

**JSON:API / RFC 8288 (baseline)**
- `GET /articles?include=author&fields[articles]=title,body&fields[people]=name`. Pagination via `page` family; body `links`: `first`/`last`/`prev`/`next`. Sort: `sort=-created,title`.
- RFC 8288 header: `<url>; rel="next"`; multi: `<...>; rel="next", <...>; rel="last"`.

**Zalando**
- Sort: `/sales-orders?sort=+id` (`+` asc, `-` desc). Reserved params: `q`, `sort`, `fields`, `embed`, `offset`, `cursor`, `limit`.
- `ResponsePage`: `self`, `first`, `prev`, `next`, `last` (`uri|cursor`), `query` object, `items` array. "SHOULD prefer cursor-based pagination, avoid offset-based pagination." "Simple query languages are generally preferred over complex ones."

## 9. Caveats
- **Currency.** All findings retrieved July 19, 2026. Shopify REST is explicitly legacy/frozen (Oct 1, 2024); treat its patterns as still-operational but not evolving. GitHub is mid-migration toward cursor pagination on high-volume endpoints; page-number behavior varies per endpoint. Stripe's `total_count` list expansion was removed 2025-03-31 — verify against the live API reference before relying on it anywhere.
- **Per-API variation within a vendor.** Microsoft Graph default/max page sizes, over-`$top` behavior, and which of `$skip`/`$skiptoken` appears all vary by resource. AWS defaults/maxes and token names vary per service — figures cited are representative, not universal.
- **Twilio `start`/`end`.** These appear in some live classic JSON payloads as integer record indices but are NOT in Twilio's documented pagination property table; the documented fields are the six URI/page fields plus the resource array. In the default XML of the 2010 API, attribute names are lowercase without underscores (`firstpageuri`, `nextpageuri`, `pagesize`).
- **Stripe cursor nature.** Its cursors are object IDs, not opaque tokens — an intentional design difference, not an omission.
- **Guideline vs deployed API.** Zalando and the Azure REST Guidelines are design documents, not deployed APIs; Microsoft Graph is the deployed OData realization and its behavior sometimes diverges from the guideline (e.g. `maxpagesize` vs `$top`). The Google AIP `50/1000` figures are illustrative proto-comment examples, not a mandated default.
- **Scope.** Rate limiting (Part 6), general body/envelope conventions (Part 3), and reliability/consistency mechanics beyond what's documented for lists (Part 5) are out of scope; read-after-write notes here are limited to what each vendor documents for list/search endpoints.