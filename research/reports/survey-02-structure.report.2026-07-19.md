# REST API Conventions Series — Part 2/7: Structure (Resources, URLs, Methods & Status Codes)

## 1. TL;DR
- Across the eight references there is a genuine **de facto consensus on the shape of a REST surface** — resource-oriented nouns, plural collections, hierarchy expressed via path segments, and standard HTTP verbs — but that consensus fractures sharply on five axes: **update method (POST vs PUT vs PATCH)**, **path casing (kebab vs camel vs snake vs Pascal)**, **custom-action syntax (`:verb` vs sub-path vs body-flag)**, **validation status (400 vs 422)**, and **whether the API is resource-oriented at all** (AWS's RPC protocols).
- **Stripe is confirmed to be a partial outlier, not a template**: its own reference states the API "accepts form-encoded request bodies, returns JSON-encoded responses, and uses standard HTTP response codes, authentication, and verbs." It uses **POST for both create and update** (no PATCH/PUT), returns **200 (not 201)** on creates, and returns a **`deleted: true` object body** rather than 204 on deletes. These are widely emulated but diverge from every formal guideline document (AIP, Azure, Zalando) in the set.
- **AWS marks the boundary of the convention space**: its high-volume services (EC2 `ec2Query`, DynamoDB `awsJson1_0`) are **RPC, not REST** — every call is a `POST /` with the operation in an `Action=` parameter or `X-Amz-Target` header — while **S3 (`restXml`) is genuinely resource-oriented** within the same company. AWS buys uniform tooling/codegen and signing at the cost of HTTP-native semantics, caching, and readability.

## 2. Key findings (numbered)

1. **Resource-orientation is the default for 7 of 8; AWS is the split.** Stripe, GitHub, Google/AIP, Microsoft (Azure + Graph), Twilio, Shopify REST, and the Zalando guideline all model nouns + standard verbs. AWS's flagship services (EC2, DynamoDB, CloudWatch, most control-plane APIs) are action/RPC-based; only S3, Lambda-style `restJson1` services, and a few others use resource URLs.

2. **Plural collections are near-universal.** AIP mandates plural collection identifiers; Zalando mandates "Pluralize Resource Names"; Azure/Graph, GitHub, Stripe, Twilio, Shopify all use plural (`/customers`, `/repos`, `/orders`, `/Accounts`). The only systematic exception is singleton/config resources and AWS RPC action names.

3. **Path casing is genuinely contested — four different house styles.** Zalando and Zalando-style URL rules mandate **kebab-case** path segments (`/sales-order-items`); AIP resource *names* use **camelCase** collection identifiers (`userEvents`); Twilio's classic API uses **PascalCase** path segments and PascalCase query/body params (`/Accounts/{sid}/Messages.json`, `FriendlyName`); Stripe/GitHub/Shopify use **snake_case** or lowercase single words. There is no cross-field winner.

4. **Update method is the single sharpest structural split.** Azure mandates **PATCH + JSON Merge Patch (RFC 7396)**; Microsoft general guidelines say PUT is "dangerous" for partial update; AIP uses **PATCH with an `update_mask` field mask**; Zalando uses **PUT for full replace, PATCH for partial**; GitHub and Shopify use **PATCH/PUT**; **Stripe uses POST for updates** (no PATCH/PUT at all); AWS RPC uses POST-to-root for everything.

5. **Stripe's form-encoding + POST-for-everything is verified and deliberate.** The Stripe API Reference states: "The Stripe API is organized around REST. Our API has predictable resource-oriented URLs, accepts form-encoded request bodies, returns JSON-encoded responses, and uses standard HTTP response codes, authentication, and verbs." Updates are `POST /v1/{resource}/{id}`; custom actions are `POST /v1/payment_intents/{id}/capture`. Nested objects use bracket notation (`shipping[address][line1]`), arrays use indexed keys (`items[0][price]`). Its newer v2 namespace accepts JSON.

6. **404-vs-403 existence-hiding is a documented GitHub policy, not folklore.** GitHub Docs ("Troubleshooting the REST API") state verbatim: "GitHub uses a 404 Not Found response instead of a 403 Forbidden response to avoid confirming the existence of private repositories." This is a deliberate security-driven divergence from literal HTTP semantics.

7. **400-vs-422 for validation is split.** GitHub returns **422 Unprocessable Entity** ("Validation failed, or the endpoint has been spammed"); Azure mandates **400** for malformed/unrecognized fields; Zalando explicitly says business/semantic validation failures **should use 400, not 422**; Stripe uses **400** for invalid params (with a typed error body).

8. **Nesting depth in practice is shallow (1–3 levels); flat + query-filter is the common alternative.** GitHub goes deepest routinely (`/repos/{owner}/{repo}/issues/{n}/comments`); Shopify nests one level (`/products/{id}/variants/{id}`) but also offers flat (`/variants/{id}`); Stripe strongly prefers **flat resources with `?customer=` filters** over nesting; Zalando caps at **3 sub-resources**; AIP alternates collection/id segments arbitrarily deep but discourages deep hierarchy.

## 3. Baseline position — what RFC 9110 and the guideline documents prescribe

### RFC 9110 (HTTP Semantics, the substrate)
- **Safe methods**: GET, HEAD, OPTIONS, TRACE (no observable state change requested).
- **Idempotent methods**: PUT, DELETE, and the safe methods. **POST and PATCH are neither safe nor idempotent by default.**
- PUT = "create or replace the entire target resource with the enclosed representation" (full replacement).
- POST = "process the enclosed representation according to the resource's own semantics" — the catch-all for non-idempotent/create/action.
- PATCH (RFC 5789) = partial modification; not idempotent unless the patch document is designed to be.
- 201 Created for a create that yields a new resource (with Location); 202 Accepted for async; 204 No Content for success-with-no-body; 409 Conflict when the request conflicts with current resource state; 412 for failed preconditions.

### Google AIP (aip.dev) — resource-oriented design, prescriptive
- **AIP-121**: model individually-named resources (nouns); a small set of **standard methods** (List, Get, Create, Update, Delete); **custom methods only when standard methods don't fit**. Custom methods use ordinary HTTP verbs (usually POST) with a **custom verb in the URI**, never a new HTTP verb.
- **AIP-122**: resource names are URI-path-shaped without leading slash (`publishers/123/books/les-miserables`); segments **alternate collection-id / resource-id**; **collection identifiers must be plural, camelCase, ASCII** (`/[a-z][a-zA-Z0-9]*/`); where no plural exists ("info"/"moose"), the singular is used and words must not be "coined" by adding "s".
- **AIP-123**: `plural` must be the lowerCamelCase plural of the singular; pattern variables use snake_case.
- **AIP-136**: custom method URI form is `POST /v1/{name=publishers/*/books/*}:archive`; the **`:` + camelCase verb** must match the RPC verb name; GET may be used for data retrieval, POST if there are side effects or the body could exceed URL limits.
- Standard update uses **PATCH with a field mask (`update_mask`)**.

### Microsoft — Azure REST Guidelines + Graph
- Azure: **DO create/update with PATCH + JSON Merge Patch (RFC 7396), media type `application/merge-patch+json`**; **DO use PUT for wholesale create/replace**; PUT "is dangerous" for partial modification because unknown fields may be dropped. Azure REST supports **GET, HEAD, PUT, POST, PATCH** (DELETE exists per-resource but is not in the request-components enumeration).
- Azure: **DO fail with 400** if the request is malformed or any JSON field/value is not understood by the version. **UPSERT via PATCH**: PATCH to nonexistent resource = create; if UPSERT unsupported, PATCH to nonexistent resource **MUST return 409**.
- Azure: use nouns, plural collections (`/customers`, `/customers/5`); expose a stable URL with a unique identifier in addition to friendly URLs; accommodate clients that cap URLs at 2,083 chars, respond **414** beyond parse limits.
- Graph: relationships via **OData navigation properties** as path segments (`/users/{id}/manager`, `/users/{id}/messages`); `@odata.bind` to set relationships; `$ref` to address just the link; `$expand` to inline related entities. PATCH returns 204; POST-create returns 201.

### Zalando RESTful API Guidelines (guideline doc, not an API)
- **Kebab-case path segments** matching `^[a-z][a-z\-0-9]*/`; **snake_case query params** (`^[a-z][_a-z0-9]*/`) and snake_case JSON property names — "never camelCase"; **Hyphenated-Pascal-Case HTTP headers**.
- **Pluralize resource names**; model singletons as a collection of cardinality 1 (maxItems=minItems=1).
- **MUST Avoid Trailing Slashes** (no duplicate/trailing slashes; results identical with/without any trailing slash — trailing slash MUST NOT carry semantics); **should not use `/api` as base path**.
- **"MUST Avoid Actions — Think About Resources"**: model actions as resources (e.g., a lock is created with PUT/POST to a lock resource; a cancel is a message to a "cancellations" collection).
- **Resource types (root paths) ≤ 8; ≤ 3 sub-resources deep.**
- Status codes: curated subset; **201** (POST/PUT create), **202** async, **204** (PUT/DELETE/PATCH), **207** for batch/bulk (always, even on total failure); business/semantic validation failures **use 400, not 422**; **409** for conflict (and for non-idempotent POST where resource already exists, e.g., payment); **412** for If-* precondition conflicts (not 409); **410** for intentionally-deleted; SHOULD NOT use redirection codes.

## 4. Side-by-side comparison tables

### Table A — Resource modeling & orientation
| Reference | Orientation | Resource naming | Collection vs singleton | Where non-CRUD actions live |
|---|---|---|---|---|
| Stripe | Resource-oriented REST | Lowercase nouns (`payment_intents`, `customers`) | Collections; some singletons (account) | `POST /v1/{res}/{id}/{verb}` (e.g., `/capture`, `/confirm`, `/cancel`) |
| GitHub | Resource-oriented REST + hypermedia | Nouns (`repos`, `issues`, `pulls`) | Collections; `/user` singleton (auth'd) | Sub-path + method (`PUT .../star`, merge, lock) |
| Google/AIP | Resource-oriented (RPC-transcoded) | camelCase collections, alternating name segments | Both; singletons explicit | Custom methods `:verb` |
| Microsoft Azure | Resource-oriented | Nouns, plural | Both | Action patterns / POST |
| Microsoft Graph | Resource-oriented (OData) | Nouns, navigation properties | Both (entity sets, singletons) | OData actions/functions; `me/sendMail` |
| Twilio | Resource-oriented REST | PascalCase nouns (`Messages`, `Calls`) | Collections under `/Accounts/{sid}` | POST to sub-resource; some action-ish resources |
| Shopify Admin REST | Resource-oriented REST (legacy) | snake_case nouns (`products`, `orders`) | Collections; nested variants | Sub-path (`/orders/{id}/cancel.json`) |
| Zalando (guideline) | Prescribes resource-orientation | kebab-case domain nouns | Collections; singleton = collection card. 1 | "Actions as resources" (letterbox model) |
| AWS | **RPC/action for EC2, DynamoDB, most control plane; REST only for S3, restJson1 services** | `Action=RunInstances`, `X-Amz-Target: DynamoDB_20120810.PutItem` | N/A (operations, not resources) | Everything is an action |

### Table B — Noun number & casing
| Reference | Path segments | Query param casing | JSON field casing |
|---|---|---|---|
| Stripe | plural, lowercase/snake | snake_case | snake_case |
| GitHub | plural, lowercase | snake_case | snake_case |
| Google/AIP | plural, **camelCase** collection ids | camelCase | camelCase (proto→JSON) |
| Azure/Graph | plural, camelCase | camelCase / OData `$`-prefixed | camelCase |
| Twilio | plural, **PascalCase** (`/Messages`) + `.json` | **PascalCase** (`FriendlyName`) | **snake_case** in responses (`friendly_name`) |
| Shopify | plural, snake_case, `.json` suffix | snake_case | snake_case |
| Zalando | plural, **kebab-case** | **snake_case** | snake_case |
| AWS (query) | N/A (root `/`) | PascalCase params (`InstanceId`, `MaxResults`) | PascalCase JSON keys (DynamoDB `TableName`) |

### Table C — Nesting & relationships
| Reference | Typical depth | Nest vs reference | Max observed |
|---|---|---|---|
| Stripe | Flat + query filter (`?customer=`) | References; nesting mainly for actions | ~2 (`/{res}/{id}/{action}`) |
| GitHub | Deep nesting routine | Nests | 4+ (`/repos/{o}/{r}/issues/{n}/comments`) |
| Google/AIP | Alternating, discourages deep | Nests via resource names; discourages embedding | arbitrary but discouraged |
| Azure | Hierarchy for containment | Both | moderate |
| Graph | Navigation-property chains | Nests (navigation props) | moderate–deep |
| Twilio | One level under `/Accounts/{sid}` | Nests | 2–3 |
| Shopify | One level (`/products/{id}/variants`) + flat alt | Both | 2 |
| Zalando | ≤ 3 sub-resources (rule) | Both | 3 (capped) |
| AWS | none (flat root) | references by ARN/ID in body | 0 |

### Table D — HTTP method usage & PATCH format
| Reference | Create | Update | Delete | PATCH format | Method override |
|---|---|---|---|---|---|
| Stripe | **POST** (→200) | **POST** (→200) | DELETE (→200 + `deleted:true`) | n/a (POST form-encoded) | none needed |
| GitHub | POST (→201) | PATCH / PUT | DELETE (→204) | proprietary JSON body | — |
| Google/AIP | POST create | **PATCH + `update_mask`** | DELETE | field mask | — |
| Azure | PUT (create/replace) / PATCH | **PATCH + JSON Merge Patch** | DELETE | `application/merge-patch+json` | — |
| Graph | POST (→201) | **PATCH** (→204) | DELETE | JSON Merge Patch (OData) | — |
| Twilio | POST | **POST** (create+update per docs) | DELETE | n/a (form-encoded) | — |
| Shopify | POST (→201) | **PUT** | DELETE | full/partial JSON | — |
| Zalando | POST (server ids) / PUT (client ids) | **PUT full / PATCH partial** | DELETE | JSON Merge or JSON Patch | Idempotency-Key (may) |
| AWS | POST-to-root (Action/Target) | POST-to-root | POST-to-root | n/a | n/a (S3 uses real PUT/GET/DELETE) |

### Table E — Status code working subsets
| Reference | Create | Update | Delete | Validation | Not-found/authz | Conflict |
|---|---|---|---|---|---|---|
| Stripe | 200 | 200 | 200 (body `deleted:true`) | 400 (typed error) | 404; 402 for card/pay | 409 (alerts) |
| GitHub | 201 | 200 | 204 | **422** | **404 to hide private** | 409 (some) |
| Google/AIP | 200/201 | 200 | 200/204 | 400 (google.rpc.Status) | 404/403 | 409 |
| Azure | 201 (+Location) / 200 | 200/204 | 204 | **400** | 404 | **409** (PATCH-to-missing w/o UPSERT) |
| Graph | 201 | 204 | 204 | 400 | 404 | 409 |
| Twilio | 201 | 200 | 204 | 400 | 404 | 409 (Studio dup exec) |
| Shopify | 201 | 200 | 200 | **422** | 404 | 422/409 |
| Zalando | 201 (+Location for POST) | 200/204 | 204 | **400 (not 422)** | 404/410 | 409 (or 412 for If-*) |
| AWS (query/json) | 200 body / **400/500 on error** | " | " | 400 + error `Code`/`__type` | 400 + typed code | 400 typed |

### Table F — URL mechanics
| Reference | Trailing slash | Version in path | Path param style | Special segments |
|---|---|---|---|---|
| Stripe | not significant | `/v1/`, `/v2/` prefix | `/{id}` opaque tokens (`cus_`, `pi_`) | `expand[]` |
| GitHub | **trailing slash → 404** | header (`X-GitHub-Api-Version`) | `{owner}/{repo}` named | hypermedia `_url` templates |
| Google/AIP | no trailing | `/v1/` | resource-name segments | `:verb`, `*` wildcard |
| Azure | — | `?api-version=` query | `/{id}` + friendly + stable | `/my` shortcut |
| Graph | — | `/v1.0/`, `/beta/` | `/{id}` | `$ref`,`$expand`,`$count`,`$query` |
| Twilio | `.json` suffix, no trailing | `/2010-04-01/` or `/v1/` | `/{Sid}` (typed SIDs `AC`,`SM`) | `.json`/`.xml` format suffix |
| Shopify | `.json` suffix required | `/admin/api/{version}/` | `/{id}.json` | `.json` resource suffix |
| Zalando | **MUST avoid trailing** | (Part 6 scope) | kebab segments | none |
| AWS | root `/` only | date-versioned (`Version=2016-11-15`) | none | `X-Amz-Target` header |

## 5. Per-reference notes

**Stripe.** The most-copied commercial API, and a deliberate simplifier. Resource-oriented URLs (`/v1/customers/{id}`), but the verb discipline is minimal: **POST creates, POST updates, GET reads, DELETE deletes** — there is no PATCH or PUT anywhere in v1. Request bodies are **`application/x-www-form-urlencoded`** by default (v1); nested data uses bracket notation `shipping[address][line1]`, arrays use `items[0][price]`. Its newer **v2** namespace accepts JSON bodies and requires the `Stripe-Version` header, and supports `Idempotency-Key` on all POST/DELETE. Creates return **200, not 201**, and no Location header. Deletes return **200 with a `{"id":..., "deleted":true}`** body, not 204. Custom/non-CRUD actions are sub-path POSTs: `POST /v1/payment_intents/{id}/capture`, `/confirm`, `/cancel`, `/increment_authorization`. Errors are 4xx with a typed body (`error.type` ∈ `card_error|invalid_request_error|api_error|idempotency_error`, plus `code`, `param`, `doc_url`, `request_log_url`). Expansion depth is capped at 4 levels. Confidence: high (primary docs).

**GitHub.** Resource-oriented with genuine hypermedia (`*_url` link templates in every payload, and a root document at `https://api.github.com/`). Deep, readable nesting: `/repos/{owner}/{repo}/issues/{issue_number}/comments`. Uses the full verb set — POST create (201), PATCH/PUT update, DELETE (204). Two structurally important idiosyncrasies: (1) **404 instead of 403** to hide private-resource existence ("GitHub uses a 404 Not Found response instead of a 403 Forbidden response to avoid confirming the existence of private repositories"); (2) **422 Unprocessable Entity** for validation ("Validation failed, or the endpoint has been spammed"). **Trailing slash → 404.** Non-CRUD actions use expressive sub-paths and PUT for idempotent toggles (star, subscription). Version is pinned by header (`X-GitHub-Api-Version: 2022-11-28`), not path. Pull requests are modeled as a sub-type of issues (shared endpoints for labels/assignees). Confidence: high.

**Google Cloud / AIP.** Not a single API but the most rigorous *prescriptive* resource-oriented standard. Resource names are hierarchical, alternating collection/id (`publishers/123/books/les-miserables`); **collection ids MUST be plural camelCase**. Five standard methods; anything else is a **custom method** with a `:verb` suffix (`POST .../books/*:archive`) and the verb must be a non-standard-verb-noun in camelCase. Updates use **PATCH + `update_mask`** field mask. Long-running operations return an `Operation` to poll. Errors map to `google.rpc.Status`. Explicitly warns that "having an API that is identical to the underlying database schema is actually an anti-pattern." Confidence: high.

**Microsoft (Azure + Graph).** Two related but distinct standards. **Azure REST Guidelines**: PATCH + **JSON Merge Patch (`application/merge-patch+json`, RFC 7396)** for create/update; PUT for wholesale replace; PUT called "dangerous" for partial updates; **400** for any unrecognized field; **UPSERT via PATCH** with **409** if PATCH hits a nonexistent resource and UPSERT is unsupported; same JSON schema across PUT/PATCH/GET/POST on a path; POST may return a **Location** header for server-named resources; supported methods enumerated as GET/HEAD/PUT/POST/PATCH. **Graph** layers **OData**: relationships as **navigation properties** (`/users/{id}/manager`, `/users/{id}/messages`), `@odata.bind` to set a relationship, `$ref` for link-only, `$expand` to inline; write size limit 4 MB (413 beyond); PATCH → 204, POST → 201. Confidence: high.

**Twilio.** Classic Twilio (`api.twilio.com/2010-04-01/`) is a distinctive casing outlier: **PascalCase path segments** (`/Accounts/{AccountSid}/Messages.json`), **PascalCase request params** (`FriendlyName`, `To`, `From`), but **snake_case response fields** (`friendly_name`, `account_sid`) — a documented request/response asymmetry. Format is selected by a **`.json`/`.xml`/`.csv` suffix** (classic API defaults to XML; product APIs `*.twilio.com/v1` are JSON-only). Bodies are **form-encoded** (`--data-urlencode`). **POST does both create and update** (docs: "POST: Create or update a resource"). Typed SID path params (`AC…`, `SM…`, `MG…`, `FW…`). Studio v2 introduced **409 Conflict** for creating an Execution when one is already active. Confidence: high.

**Shopify Admin REST.** Resource-oriented but now **legacy**. URL shape: `/admin/api/{version}/{resource}.json` with a **mandatory `.json` suffix** and date-versioned path (`2026-07`). One level of nesting (`/products/{id}/variants/{id}.json`) plus flat alternates (`/variants/{id}.json`). Full verb set — POST create (201), **PUT update**, DELETE. Validation returns **422** (`{"errors":[...]}`), rate limit 429 (`Exceeded 2 calls per second`). **REST→GraphQL migration is the key currency caveat**: Shopify's own docs state "The REST Admin API is a legacy API as of October 1, 2024. Starting April 1, 2025, all new public apps must be built exclusively with the GraphQL Admin API." Per Shopify's migration guidance, the REST product/variant endpoints "won't receive new features… and will be unable to increase their variant limit past 100," while the new GraphQL product APIs support the expanded **2048-variant** model — so any "current" reading of Shopify's REST structure is frozen and no longer receiving features. Non-CRUD actions are sub-path POSTs (`/orders/{id}/cancel.json`). Confidence: high.

**Zalando RESTful API Guidelines.** A guideline document (not an API), and the most explicit on this surface. Kebab-case paths, snake_case query params and JSON fields ("never camelCase"), Hyphenated-Pascal-Case headers. Pluralize resources; singleton = collection of cardinality 1. **≤ 8 root resource types, ≤ 3 sub-resource levels.** MUST avoid trailing slashes and MUST NOT give them semantics. **"MUST Avoid Actions — Think About Resources"**: no verbs in URLs; model an action as a resource (create a lock; POST to a "cancellations" collection). Curated status subset (201/202/204/207/400/401/403/404/405/406/409/410/412/415/423/428/429…); **business validation → 400 not 422**; **207 for all batch/bulk**; **409** for conflict / already-exists non-idempotent POST; **412** for If-* conflicts. PUT full replace, PATCH partial. Confidence: high.

**AWS — full contrast treatment.** AWS is in this set precisely to mark where the convention space ends. All AWS services are modeled in Smithy (AWS: "All AWS services are modeled in Smithy to thoroughly document the API contract including operations and behaviors like protocols, authentication…"), and their HTTP APIs fall into six Smithy-defined protocols (`aws.protocols` namespace):

| Protocol | Nature | Content-Type | Response | Example services |
|---|---|---|---|---|
| `restXml` | **RESTful** verbs/URIs | varies | XML | **S3** |
| `restJson1` | **RESTful** verbs/URIs/status codes | `application/json` | JSON | **Lambda**, API Gateway |
| `awsJson1_0` | **RPC** (POST `/`) | `application/x-amz-json-1.0` | JSON | **DynamoDB** |
| `awsJson1_1` | **RPC** (POST `/`) | `application/x-amz-json-1.1` | JSON | ECS, and many |
| `awsQuery` | **RPC**, form body | `application/x-www-form-urlencoded` | XML | CloudWatch (traditionally), SNS, CloudFormation, STS |
| `ec2Query` | **RPC**, EC2 extension of awsQuery | `application/x-www-form-urlencoded` | XML | **EC2** |

- **EC2 (`ec2Query`)**: "The Amazon EC2 Query API provides HTTP or HTTPS requests that use the HTTP verb GET or POST and a Query parameter named `Action`." So `?Action=RunInstances&InstanceId=…&Version=2016-11-15`. Operation is in a parameter, not the URL; params are PascalCase; response is XML. Pagination via `MaxResults`/`NextToken`.
- **DynamoDB (`awsJson1_0`)**: **every operation is `POST /`** dispatched by the **`X-Amz-Target: DynamoDB_20120810.PutItem`** header, `Content-Type: application/x-amz-json-1.0`. AWS docs: "The X-Amz-Target header contains the name of a DynamoDB operation… (This is also accompanied by the low-level API version, in this case 20120810.) The payload (body) of the request contains the parameters for the operation, in JSON format." JSON keys are PascalCase; values are type-tagged (`{"S":...}`, `{"SS":[...]}`).
- **S3 (`restXml`) — the in-house REST contrast**: standard verbs on resource URLs (`PUT /{key}` with the bucket in the host). This is genuine REST inside the same company whose flagship compute/database APIs are RPC.
- **Error behavior**: contrary to common lore, awsJson/awsQuery return **real 400 (client) / 500 (server) codes**, not 200-on-error. AWS confirms: "If DynamoDB can't process a request, it returns an HTTP error code and message." Error *identity* lives in the body/header, not HTTP: awsJson uses **`__type`** (full Shape ID in 1.0; bare shape name in 1.1) and/or `X-Amzn-Errortype`; awsQuery/restXml use an XML **`<Error><Code>…</Code></Error>`** with a `Type` (Sender/Receiver) element; ec2Query wraps errors in `<Response><Errors><Error>`.
- **What the divergence buys**: one uniform request model per protocol → trivial codegen across AWS's 240+ fully featured services (AWS states "more than 240 fully featured services" as of 2026), uniform SigV4 signing, no URL-length limits (body carries everything), and easy batching. **What it costs**: no HTTP caching (everything is POST), no intermediary/proxy method semantics, error identity invisible to HTTP tooling, operation intent invisible in the URL, and reduced human readability. `awsQuery` is explicitly **deprecated for new services** in the Smithy spec ("This protocol is deprecated and SHOULD NOT be used for any new service"), and AWS is migrating some query services (CloudWatch, SQS) to `awsJson`. Confidence: high (Smithy spec + AWS docs + botocore metadata).

## 6. Agreements vs divergences

**De facto standards (near-consensus across the 7 REST references):**
- Resource-oriented nouns + standard HTTP verbs (all but AWS RPC).
- **Plural collections** (`/customers`, `/orders`, `/repos`).
- Hierarchy expressed through path segments; opaque IDs in path params.
- GET is safe/read; DELETE removes; POST is the create/action catch-all.
- JSON response bodies; typed error objects.
- Version appears as structure (path prefix `/v1/`, `/admin/api/{date}/`) or a header — never absent.

**Genuine divergences (with descriptive tradeoffs):**
- **Update verb.** POST (Stripe, Twilio) is dead-simple and needs no PATCH-format decision, but loses idempotency guarantees and conflates create/update semantically. PATCH+MergePatch (Azure, Graph) is HTTP-correct and bandwidth-efficient but can't represent explicit-null cleanly. PATCH+field-mask (AIP) is unambiguous but non-standard. PUT-full (Zalando/Shopify) is idempotent but risks dropping unknown fields.
- **Path casing.** kebab (Zalando) is URL-idiomatic and case-insensitive-safe; camel (AIP) aligns path with proto/JSON field names; PascalCase (Twilio) matches SDK class names but is unusual in URLs; snake/lowercase (Stripe/GitHub) is the pragmatic majority. No correctness winner — purely a consistency choice.
- **Custom-action syntax.** `:verb` (AIP) keeps the resource identity clean and greppable; sub-path POST (Stripe/GitHub/Shopify) is intuitive but blurs noun/verb; action-as-resource (Zalando) is purest-REST but can feel contrived ("cancellations letterbox").
- **Validation status.** 422 (GitHub/Shopify) distinguishes syntactic-OK/semantically-invalid; 400 (Azure/Zalando/Stripe) is simpler and Zalando explicitly rejects 422 for business validation. Split with no consensus.
- **Create response.** 201+Location (Azure/Graph/GitHub/Twilio/Shopify/Zalando) is HTTP-correct; 200-no-Location (Stripe) is simpler but non-conforming.
- **Delete response.** 204 (most) vs 200+`deleted:true` body (Stripe) — the latter enables audit/echo at the cost of HTTP idiom.
- **404-vs-403.** GitHub's blanket 404-to-hide-existence trades HTTP precision for security; most others use 403 for authz and 404 for real absence.
- **Nesting philosophy.** GitHub embraces deep nesting for readability; Stripe rejects it for flat + query filters (easier caching/refactoring); Zalando caps at 3.

## 7. CONTESTED AXES REGISTER (scoped to Part 2)

| Axis | Options observed | Who does what | Tradeoff (one line) | How contested |
|---|---|---|---|---|
| Resource vs action orientation | Resource-oriented REST / RPC-action | REST: Stripe, GitHub, AIP, MS, Twilio, Shopify, Zalando. RPC: AWS EC2/DynamoDB/most control-plane (S3 is REST) | REST = HTTP-native semantics & caching; RPC = uniform codegen/signing, no URL limits | **Split** (7 vs AWS; but AWS is single-vendor) |
| Plural vs singular collections | Plural / singular-singleton | Plural: all REST refs. Singular only for singletons (`/user`, config) | Plural reads naturally for collections; singleton needs a rule | **Near-consensus** (plural) |
| Path casing | kebab / camel / snake / Pascal | kebab: Zalando; camel: AIP; snake/lower: Stripe, GitHub, Shopify; Pascal: Twilio | Cosmetic but must be globally consistent | **Wide-open** (4 styles) |
| Query-param casing | snake / camel / Pascal | snake: Zalando, Stripe, GitHub, Shopify; camel/`$`: Azure/Graph; Pascal: Twilio, AWS | Should match field casing for coherence | **Split** |
| Nesting policy | deep / shallow-capped / flat+filter | deep: GitHub; ≤3: Zalando; 1-level+flat: Shopify; flat+`?filter`: Stripe; alternating: AIP | Deep = readable hierarchy; flat = cache/refactor friendly | **Split** |
| Custom-action syntax | `:verb` / sub-path POST / action-as-resource / body-flag | `:verb`: AIP; sub-path POST: Stripe, GitHub, Shopify; action-as-resource: Zalando; Action= param: AWS | `:verb` keeps nouns clean; sub-path intuitive; resource-model purest | **Split** |
| Update method | POST / PUT / PATCH-merge / PATCH-mask / PUT+PATCH | POST: Stripe, Twilio; PATCH-merge: Azure, Graph; PATCH-mask: AIP; PUT: Shopify; PUT+PATCH: Zalando, GitHub | POST simplest/non-idempotent; PATCH correct but format-dependent; PUT idempotent but lossy | **Wide-open** (sharpest split) |
| PATCH body format | JSON Merge Patch / JSON Patch / field-mask / proprietary / n/a | Merge Patch: Azure, Graph; mask: AIP; Merge-or-Patch: Zalando; proprietary JSON: GitHub; n/a (POST): Stripe, Twilio | Merge simple but null-ambiguous; JSON Patch precise but verbose; mask explicit but non-standard | **Split** |
| Validation status | 400 / 422 | 400: Azure, Zalando, Stripe, AIP; 422: GitHub, Shopify | 422 distinguishes semantic vs syntactic; 400 simpler | **Split** |
| 404 vs 403 (existence hiding) | 404-to-hide / 403-for-authz | 404-hide: GitHub (documented policy); 403+404-real: most others | 404-hide = security; 403 = HTTP precision | **Split / near-consensus against blanket 404** |
| Create success code | 201(+Location) / 200 / 202 | 201: GitHub, Azure, Graph, Twilio, Shopify, Zalando; 200: Stripe; 202 async: Zalando, AIP | 201+Location HTTP-correct; 200 simpler | **Near-consensus (201)**, Stripe outlier |
| Delete success code | 204 / 200+body | 204: GitHub, Graph, Twilio, Zalando, Azure; 200+`deleted:true`: Stripe; 200: Shopify | 204 idiomatic; 200+body enables echo/audit | **Near-consensus (204)**, Stripe/Shopify outliers |
| Trailing slash | forbidden / insignificant / suffix `.json` | forbidden: Zalando; 404: GitHub; `.json` suffix: Twilio, Shopify; insignificant: Stripe | Trailing slash ambiguity breaks caches/routers | **Split** |
| Request body format | JSON / form-encoded | JSON: GitHub, AIP, Azure, Graph, Shopify, Zalando, Stripe-v2; form-encoded: Stripe-v1, Twilio, AWS-query | JSON native for nesting; form-encoded legacy-simple | **Split** (form-encoded is minority/legacy) |

## 8. EXAMPLES APPENDIX (verbatim artifacts + concrete numbers)

### Stripe (retrieved Jul 2026)
- Update (form-encoded POST, no PATCH):
  ```
  POST /v1/subscriptions/sub_123456789
  Content-Type: application/x-www-form-urlencoded
  --data-urlencode 'proration_behavior=create_prorations'
  --data-urlencode 'cancel_at_period_end=false'
  ```
- Custom action: `POST https://api.stripe.com/v1/payment_intents/{id}/capture`; also `/confirm`, `/cancel`, `/increment_authorization`.
- Delete response: `{ "id": "pm_12345", "object": "payment_method", "deleted": true }` with **200 OK**.
- Error body: `{ "error": { "type": "invalid_request_error", "message": "No such credit_note: 'cn_123'", "param": "id", "code": "resource_missing", "doc_url": "...", "request_log_url": "..." } }`
- Nested form encoding: `shipping[address][line1]`, array `items[0][price]`. Expansion max depth **4**. List `limit` 1–100, default 10.
- v2: JSON bodies, `Stripe-Version` header required (e.g., `2024-09-30.acacia`), `Idempotency-Key` on all POST/DELETE.

### GitHub (retrieved Jul 2026)
- Path: `/repos/{owner}/{repo}/issues`; deep: `/repos/{owner}/{repo}/issues/{issue_number}/comments`.
- Headers: `Accept: application/vnd.github+json`, `X-GitHub-Api-Version: 2022-11-28`.
- 404-hides-private (verbatim): "GitHub uses a 404 Not Found response instead of a 403 Forbidden response to avoid confirming the existence of private repositories."
- 422 body: `HTTP/1.1 422 Unprocessable Content … { "message": "content is not valid Base64", "documentation_url": "..." }`; generic message "Validation failed, or the endpoint has been spammed."
- Trailing slash → 404 (documented: "adding a trailing slash to the endpoint will result in a 404 Not Found").
- Content update via PUT with base64 `content`; delete via DELETE with `sha`. Custom media types on issue comments: `application/vnd.github.raw+json`, `.text+json`, `.html+json`, `.full+json`.

### Google / AIP (retrieved Jul 2026)
- Resource name: `publishers/123/books/les-miserables`.
- Custom method: `post: "/v1/{name=publishers/*/books/*}:archive"`, body `"*"`.
- Collection id regex `/[a-z][a-zA-Z0-9]*/`, plural camelCase; nested collections may omit parent prefix (`/projects/1/users`).
- Update: PATCH + `update_mask`. Retrieval GET; POST if body may exceed URL limits.

### Microsoft Azure / Graph (retrieved Jul 2026)
- Azure PATCH: `application/merge-patch+json` (RFC 7396); PUT for full replace; 400 for unknown field; PATCH-to-missing without UPSERT → **409**; POST may return `Location`.
- Graph nav property: `POST /users` with `"manager@odata.bind": "https://graph.microsoft.com/v1.0/users/{managerId}"` → **201 Created**; `PATCH /users/{id}` same body → **204 No Content**; `/users/{id}/manager/$ref` for link only.
- Graph write size limit **4 MB** (3 MB for some attachments), else **413**.
- URL length: accommodate 2,083-char clients; **414** beyond parse limit.

### Twilio (retrieved Jul 2026)
- Update (form-encoded, PascalCase params): `POST https://messaging.twilio.com/v1/Services/{ServiceSid}` `--data-urlencode 'SmartEncoding=true'`.
- Response fields snake_case: `{ "account_sid": "...", "friendly_name": "My Service!", "sid": "MG…", "sticky_sender": false, ... "links": { "phone_numbers": ".../PhoneNumbers" } }`.
- Classic path: `/2010-04-01/Accounts/{Sid}/Messages.json`; `.json`/`.xml`/`.csv` suffix (classic defaults XML; product APIs JSON-only).
- `ValidityPeriod` default 36000s (range 1–36000). `<Dial>` custom params: name ≤32 bytes, value ≤128 bytes, ≤8 params.
- Studio v2: create Execution for active contact → **409 Conflict**; POST to Flows body ≤ **1 MB**.

### Shopify Admin REST (retrieved Jul 2026)
- URL: `/admin/api/2026-07/products/{id}.json`; nested: `/products/632910392/variants.json`; flat alt: `/variants/808950810.json`.
- Create: `POST /admin/api/{v}/orders.json` `{ "order": { "line_items": [...] } }` → 201.
- Update: `PUT /admin/api/{v}/products/{id}.json` `{"product":{"id":...,"title":"A new title"}}`.
- Errors: `HTTP/1.1 422 Unprocessable Entity { "errors": [ "The fulfillment order is not in an open state." ] }`; `429 { "errors": "Exceeded 2 calls per second for api client." }`.
- Legacy dates (verbatim Shopify): "The REST Admin API is a legacy API as of October 1, 2024. Starting April 1, 2025, all new public apps must be built exclusively with the GraphQL Admin API." Product/variant REST capped at **100 variants**; GraphQL supports **2048**.

### Zalando guideline (retrieved Jul 2026)
- Path segment regex `^[a-z][a-z\-0-9]*/` (kebab); query param `^[a-z][_a-z0-9]*/` (snake); property regex `^[a-z_][a-z_0-9]*/`.
- Problem+JSON error: `HTTP/1.1 400 Bad Request` `Content-Type: application/problem+json` `{ "type":"/problems/validation-error", "title":"Validation Failed", "status":400, "detail":"Email address is not in a valid format", "instance":"..." }`.
- Status subset & method map (from published ruleset): 201→[POST,PUT]; 202→[POST,PUT,DELETE,PATCH]; 204→[PUT,DELETE,PATCH]; 207→[POST]; 409→[POST,PUT,DELETE,PATCH]; 412→[PUT,DELETE,PATCH]; 415→[POST,PUT,DELETE,PATCH].
- Limits: ≤8 root resource types; ≤3 sub-resource levels.

### AWS (retrieved Jul 2026)
- DynamoDB `POST /` with `X-Amz-Target: DynamoDB_20120810.PutItem`, `Content-Type: application/x-amz-json-1.0`; PascalCase keys; type-tagged values (`{"S":...}`, `{"SS":[...]}`). BatchWriteItem: ≤25 items, ≤16 MB, item ≤400 KB.
- EC2 `ec2Query`: GET/POST with `Action=` param + `Version=2016-11-15`; PascalCase params; XML response; `MaxResults`/`NextToken` pagination.
- S3 `restXml`: `PUT /my-application.log HTTP/1.1` / `Host: mybucket.s3.amazonaws.com` → 200 + `etag`.
- Error shapes: awsJson `__type` (full Shape ID in 1.0, bare name in 1.1) + `X-Amzn-Errortype`; awsQuery/restXml `<ErrorResponse><Error><Type>Sender</Type><Code>…</Code></Error><RequestId/></ErrorResponse>` with 400/500; ec2Query `<Response><Errors><Error><Code/></Error></Errors></Response>`.
- Protocol content-types: awsJson1_0 `application/x-amz-json-1.0`; awsJson1_1 `application/x-amz-json-1.1`; awsQuery/ec2Query `application/x-www-form-urlencoded`; restJson1 `application/json`. `awsQuery` deprecated for new services. Modeled across AWS's 240+ fully featured services.

## 9. Recommendations (staged, for the later prescriptive phase)

These are decision-scaffolding for the standard-setting phase, not prescriptions of this descriptive part.

- **Stage 1 — Lock the near-consensus first (low controversy, high leverage).** Adopt resource-oriented design with plural collections, path-segment hierarchy, opaque IDs, standard verbs, 201+Location on create, 204 on delete. Six of the seven REST references and all three guideline docs already agree; the only defensible deviation is Stripe's 200-on-everything, which should be treated as an ergonomic choice, not a model. *Threshold to revisit:* only if the target consumer base is dominated by a single form-encoded/SDK-first ecosystem.

- **Stage 2 — Force explicit decisions on the wide-open axes.** Path casing and update method have no correctness winner and must be picked by fiat then enforced by linter (as Zalando/AIP do with regexes and Spectral rules). For updates, the cleanest fully-HTTP-conformant choice in the field is **PATCH + JSON Merge Patch** (Azure/Graph) unless field-level explicit-null semantics matter, in which case **PATCH + field mask** (AIP). Reserve **POST-for-update** only if you deliberately follow Stripe's "no PATCH" simplification. *Threshold:* if partial updates with explicit null-clearing are common, reject Merge Patch in favor of JSON Patch or field masks.

- **Stage 3 — Decide the security/precision tradeoffs deliberately.** Choose 400-vs-422 (recommend documenting one; Zalando's "400 for business validation" is the simplest to enforce) and 404-vs-403 (adopt GitHub-style 404-hiding only for resources whose *existence* is sensitive; use 403 elsewhere). *Threshold:* multi-tenant APIs with confidential resource namespaces tip toward blanket 404.

- **Stage 4 — Define custom-action syntax once and never mix.** Pick `:verb` (AIP), sub-path POST (Stripe/GitHub), or action-as-resource (Zalando). The field is genuinely split; the only wrong answer is inconsistency within one API.

- **Explicitly out-of-model:** AWS RPC protocols (`awsQuery`/`ec2Query`/`awsJson`) are a boundary marker, not a candidate convention — reference them only to justify *why* the standard is HTTP-resource-oriented, and note that even AWS uses real REST (S3, `restJson1`) where developer ergonomics matter most.

## 10. Caveats
- **Volatility.** Shopify's REST→GraphQL timeline and Stripe's v1/v2 split are actively evolving; figures retrieved July 2026. Shopify REST is frozen (legacy) and no longer gains features.
- **Guideline vs practice.** AIP, Azure, Zalando are *prescriptive documents*; real Google/Microsoft/Zalando-team APIs deviate in places. GitHub's 404/422 behaviors are observed + documented but individual endpoints vary.
- **AWS single-vendor split.** AWS's RPC/REST divergence is *intra-company*; it marks the boundary of the space rather than a competing multi-vendor camp. The "200-on-error" belief about AWS RPC is a myth — real 400/500 codes are returned with error identity in the body (`__type`/`X-Amzn-Errortype` for JSON, `<Error><Code>` for XML). CloudWatch's awsQuery→awsJson migration is in progress as of 2025; treat "CloudWatch = awsQuery" as the historical mapping.
- **Secondary sources.** Some casing/status specifics were corroborated via third-party summaries (apistylebook, DeepWiki, community threads, botocore metadata) where primary docs required auth or were paginated; these are flagged inline and are consistent with primary specs.
- **Out of scope (other Parts):** error-body schemas (Part 3), pagination/collections (Part 4), idempotency/reliability semantics beyond method properties (Part 5), auth and versioning-scheme mechanics (Part 6), webhooks (Part 7).