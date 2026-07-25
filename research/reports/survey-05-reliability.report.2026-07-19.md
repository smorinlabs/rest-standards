# REST API Conventions — Part 5/7: Reliability (Idempotency, Concurrency, Async & Bulk)

*Descriptive comparison across eight references. This part covers only API-side reliability; errors (Part 3), rate limiting (Part 6), and webhook retries (Part 7) are out of scope. Retrieval dates: all research conducted July 19, 2026.*

## 1. TL;DR
- The field splits three ways on retry-safety: a **client key in an HTTP header** (Stripe `Idempotency-Key`; Shopify; Zalando/AWS as options), a **field in the request body** (Google AIP-155 `request_id`, AWS `ClientToken`; the OASIS/Azure `Repeatability-*` headers are a header-pair hybrid), and **natural idempotency via PUT/client-supplied IDs** (GitHub, generic REST). No two references implement identical mechanics; even the "famous" Stripe pattern diverges from the IETF draft it inspired.
- Concurrency control is near-consensus in *mechanism* (HTTP ETag + `If-Match` → `412`) but split on *placement* (HTTP header vs. `etag` field in body, per Google AIP-154) and whether it is required. Stripe offers **no** optimistic-concurrency/ETag mechanism at all.
- Long-running work splits between an **operation resource** you poll (Google AIP-151 `operations/{id}`; Microsoft Graph RELO/stepwise) and **header-driven polling** (Azure `202` + `Azure-AsyncOperation`/`Location` + `Retry-After`); Stripe stays largely synchronous and pushes completion via webhook events. Batch is wide-open: Microsoft Graph `$batch` (JSON, 20-item cap, 4 MB), Zalando's `207 Multi-Status`, Google's deprecated global HTTP batch, and file-based bulk import/export as a distinct model. Stripe has no generic batch endpoint.

## 2. Key Findings
1. **Stripe is header-based, POST-only, 24-hour retention, replays first response including errors.** The `Idempotency-Key` header applies to all POST requests (Stripe uses POST for both creates and updates); keys are up to 255 chars, account-scoped, and prunable after ≥24 hours. The saved status code + body of the first request is replayed for retries "regardless of whether it succeeds or fails."
2. **Stripe's two conflict behaviors are distinct and are widely mis-stated.** Same key + *different parameters* → **HTTP 400, `type: idempotency_error`**. Same key + concurrent *in-flight* request → **HTTP 409, `type: invalid_request_error`, `code: idempotency_key_in_use`**. The commonly repeated "409 idempotency_error" conflates the two.
3. **The IETF `Idempotency-Key` draft (draft-ietf-httpapi-idempotency-key-header-07) is EXPIRED** (published 15 Oct 2025, expired 18 Apr 2026, Standards Track, never published as an RFC). It codifies Stripe/PayPal practice: header syntax, uniqueness, expiry, an optional request fingerprint, and mandates a resource-conflict response on concurrent retries and `400` when the header is missing where required.
4. **Google uses a body field, not a header.** AIP-155 puts `request_id` (UUID4, ≤36 chars) *in the request message*; duplicate detection returns the prior successful response; APIs may honor keys for "any reasonable timeframe." The newer AEP-155 renames it `idempotency_key` and adds `first_seen`, explicitly citing the IETF draft as the reason.
5. **Azure/OASIS is a header-pair hybrid.** `Repeatability-Request-ID` (UUID) + `Repeatability-First-Sent` (IMF-fixdate) per OASIS Repeatable Requests v1.0; server returns `Repeatability-Result: accepted|rejected`; unsupported ops must return `501`. Azure Communication Services tracks a **5-minute** window and returns `412` outside it.
6. **Concurrency = ETag + If-Match → 412 almost everywhere, but Google puts the etag in the body.** GitHub supports conditional reads (`If-None-Match`/`If-Modified-Since` → `304`) but **not** conditional writes on unsafe methods. Azure mandates ETag conditional requests. Google AIP-154 carries `etag` as a resource/request field and returns gRPC `ABORTED` (HTTP `409`). Stripe: none.
7. **Batch/bulk is the most fragmented axis.** Microsoft Graph `$batch` (JSON, ≤20 sub-requests, 4 MB, per-item statuses, `dependsOn`, `424` on failed dependency); Zalando mandates `207 Multi-Status`; Google deprecated heterogeneous/global HTTP batch; Stripe has no generic batch; file-based bulk import/export (Shopify JSONL, Microsoft Graph mailbox PST, Azure/Graph export jobs) is a separate async model.

## 3. Baseline Position (RFC 9110, IETF draft, guideline docs)

**RFC 9110 method semantics.** GET, HEAD, OPTIONS, PUT, DELETE are idempotent; POST and PATCH are not. "Idempotent" means the *intended effect on the server* of N identical requests equals that of one — it does **not** require identical response codes (a second DELETE may return 404). This is the substrate every reference builds on: PUT/DELETE get retry-safety "for free"; POST/PATCH need an added mechanism.

**The IETF `Idempotency-Key` header draft** (`draft-ietf-httpapi-idempotency-key-header-07`, authors J. Jena & S. Dalal, httpapi WG):
- Status: **Expired.** Published 15 October 2025; expired 18 April 2026; Intended Status: Standards Track; never published as an RFC. Datatracker labels it "Expired Internet-Draft (httpapi WG)." Explicitly acknowledges inspiration from PayPal and Stripe and Mark Nottingham's "POST Once Exactly."
- Mechanics: a request header `Idempotency-Key` whose value is a quoted opaque string, e.g. `Idempotency-Key: "8e03978e-40d5-43e8-bc93-6894a57f9324"`. Applies to non-idempotent methods (POST, PATCH).
- Enforcement scenarios (verbatim structure): **First-time** key → process normally. **Duplicate after original completed (Retry)** → server MUST/SHOULD respond with the result of the previously completed operation, success or error. **Duplicate before original completes (Concurrent)** → server MUST/SHOULD respond with a resource conflict error. Missing key where required → HTTP `400` with a `Link` to documentation.
- Adds an optional **idempotency fingerprint** (hash of request payload) so the server can detect same-key/different-body misuse.

**Guideline docs on this surface:**
- **Zalando** ("SHOULD consider to design POST and PATCH idempotent"): offers three patterns — (a) `If-Match` conditional key (ETag), (b) a **secondary key** stored as a resource body property, (c) a client `Idempotency-Key` header ("MAY consider to support"). Explicitly notes the header "is not standardized in an RFC. Our only reference is the usage in the Stripe API." Recommends UUID v4, 24-hour expiry. Mandates `207` for batch/bulk and ETag+`If-Match`→`412` for concurrency.
- **Google AIP/AEP**: prescriptive resource-oriented model — `request_id`/`idempotency_key` in body (AIP-155), `etag` in body (AIP-154), `operations/{id}` for LRO (AIP-151).
- **Azure**: "All service operations must be idempotent"; POST idempotency via repeatability headers; conditional requests via `If-Match`/`If-None-Match`; LRO via `202` + async headers.

## 4. Side-by-Side Comparison Tables

### 4A. Idempotency mechanism
| Reference | Mechanism | Where | Scope | Retention | Replays errors? | Same-key/diff-body |
|---|---|---|---|---|---|---|
| **Stripe** | `Idempotency-Key` header | HTTP header | Per account | ≥24h (prunable) | **Yes** (incl. 500) | 400 `idempotency_error` |
| **IETF draft** | `Idempotency-Key` header | HTTP header | Server-defined | "validity/expiry" defined by server | Yes (return prior result, success or error) | conflict/mismatch error (via fingerprint) |
| **Google AIP-155** | `request_id` (UUID4) | Request body field | Request message | "any reasonable timeframe" (AEP: ≥1 hour) | No (returns prior *successful* response) | equivalent response or error |
| **Azure/OASIS** | `Repeatability-Request-ID` + `Repeatability-First-Sent` | HTTP headers (pair) | Server window | ACS: **5 min** tracked | Returns first-execution state | `412` if outside window |
| **Twilio** | `Idempotency-Token` (select product APIs); `I-Twilio-Idempotency-Token` (inbound webhooks) | HTTP header | Per account | Not published | Partial (per-endpoint) | 409 on some (e.g., alarms limit) |
| **Shopify (REST legacy)** | `unique_token` idempotency key | Body/header per resource | Per shop | 24h (GraphQL) | Yes (cached response) | `IDEMPOTENCY_CONCURRENT_REQUEST` on in-flight |
| **Zalando** | 3 options: `If-Match` / secondary key / `Idempotency-Key` | header / body / header | — | e.g., 24h | Yes (key cache incl. failures) | fingerprint check |
| **AWS** | `ClientToken` / `client-request-token` | Body field | Regional or zonal (EC2) | ≥24h after resource termination (EC2) | Returns equivalent response | `IdempotentParameterMismatch` |
| **GitHub** | None (natural idempotency only) | — | — | — | — | — |

### 4B. Concurrency control
| Reference | Mechanism | Placement | Failure code | Required? |
|---|---|---|---|---|
| **GitHub** | Conditional **reads** only (`If-None-Match`, `If-Modified-Since`) | HTTP header | `304` (reads); conditional writes unsupported | No |
| **Azure** | ETag + `If-Match`/`If-None-Match`, `If-Unmodified-Since` | HTTP header | `412` | Strongly recommended |
| **Google AIP-154** | `etag` (strong or weak, RFC 7232 quoting) | **Resource/request body field** | gRPC `ABORTED` → HTTP `409` | Optional (required for declarative-friendly) |
| **Zalando** | ETag + `If-Match`/`If-None-Match` | HTTP header | `412` | SHOULD/MAY |
| **Stripe** | **None** | — | — | — |
| **Shopify / Twilio / AWS** | Domain/version fields per resource (e.g., Shopify `inventorySetQuantities` compare-and-swap) | body | varies | per-resource |

### 4C. Long-running operations
| Reference | Pattern | Initiation | Poll target | Completion signal | Cancel |
|---|---|---|---|---|---|
| **Google AIP-151** | Operation resource | returns `Operation{name, done:false}` | `GET operations/{id}` | `done:true` + `response`/`error` | Operations service (may support) |
| **Azure** | Header polling | `202` + headers | `Azure-AsyncOperation` (preferred) or `Location` | `provisioningState`: Succeeded/Failed/Canceled | poller/service-dependent |
| **Microsoft Graph** | RELO (resource-based) or stepwise operation | `202` + `Location` | monitor URL / `operations/{id}` | `status: succeeded/failed` (or 3xx redirect to result) | resource-dependent |
| **Stripe** | Largely synchronous | direct response | — | webhook events | — |
| **Shopify** | Bulk Operations (async job) | returns job id | poll job | JSONL result file | cancel mutation |
| **Zalando** | MAY async | `202 Accepted` | service-defined | service-defined | — |

### 4D. Batch / bulk
| Reference | Batch mechanism | Shape | Limits | Atomicity | Partial-failure reporting |
|---|---|---|---|---|---|
| **Microsoft Graph** | `$batch` JSON | `{requests:[{id,method,url,body,headers,dependsOn}]}` | **≤20** sub-requests, 4 MB | Independent (unless `dependsOn`) | per-item `responses[]` with own status; `424` on failed dependency |
| **Zalando** | `207 Multi-Status` | payload with per-item status | — | independent items | per-item status body (even if all fail/async) |
| **Google** | (historic) global HTTP batch — **discontinued**; homogeneous per-API batch only | multipart | — | independent | per-item |
| **Stripe** | **No generic batch** | — | — | — | — |
| **AWS** | Service-specific (e.g., batch APIs per service) | varies | varies | varies | varies |
| **GitHub / Twilio** | No generic REST batch | — | — | — | — |
| **File-based bulk** (Shopify JSONL, Graph mailbox/export jobs) | async import/export | file (JSONL/PST/zip) | per-service | per-row | per-row in result file |

## 5. Per-Reference Notes

### 5.0 Stripe — DEEP DIVE (the reference the requester cares most about)
Stripe's idempotency is the field's touchstone, and its mechanics are precise:

- **Header & methods.** Keys travel in the `Idempotency-Key` request header. "All POST requests accept idempotency keys. Don't send idempotency keys in GET and DELETE requests because it has no effect." Because Stripe models *updates* as POST (not PUT/PATCH), the same header covers both creates and updates — a design point that means retry-safety on updates depends entirely on the key, not on HTTP method semantics.
- **Key format & scope.** "Idempotency keys are up to 255 characters long." Stripe recommends V4 UUIDs "or another random string with enough entropy to avoid collisions." Scope is **per account** (two accounts using the same string get independent results). Empty string is treated as absent.
- **Retention window.** "You can remove keys from the system automatically after they're at least 24 hours old. We generate a new request if a key is reused after the original is pruned." (The error-handling docs corroborate 24-hour expiry.)
- **Replay semantics.** "Stripe's idempotency works by saving the resulting status code and body of the first request made for any given idempotency key, regardless of whether it succeeds or fails. Subsequent requests with the same key return the same result, including 500 errors." Replays carry the header `Idempotent-Replayed: true`. Results are saved "only after the execution of an endpoint begins" — so validation failures and concurrent conflicts are *not* cached and can be retried.
- **Concurrent duplicates in flight.** Returns **HTTP 409 Conflict**, `type: invalid_request_error`, `code: idempotency_key_in_use`, message: "There is currently another in-progress request using this Idempotent Key … Please try again later." (The docs describe this as a 409 "conflict" where "the request conflicts with another request that's executing concurrently.")
- **Same key, different body.** Returns **HTTP 400**, `type: idempotency_error`: "Keys for idempotent requests can only be used with the same parameters they were first used with." Docs: "The idempotency layer compares incoming parameters to those of the original request and errors if they're not the same."
- **Retry signaling.** Stripe sets `Stripe-Should-Retry: true|false` when it can determine retryability; absent means fall back to status code. Caveat: rate-limit `429`s, auth `401`s, and most parameter `400`s run *before* the idempotency layer, so they can produce different results on retry — Stripe advises generating a fresh key when modifying a request to get a successful result.
- **SDK auto-generation.** Stripe's official libraries auto-generate idempotency keys and retry with exponential backoff + jitter when configured (e.g., the Ruby library retries on failure with an idempotency key).
- **Concurrency control.** **Stripe offers no ETag/`If-Match`/version-based optimistic concurrency.** Its status-code table lists 400/401/402/403/409/424/429/5xx — notably **no 412 Precondition Failed**. The only concurrency primitive is the idempotency key itself.
- **Long-running work.** Stripe is largely synchronous: the API returns the resource directly, and asynchronous outcomes (bank confirmations, disputes, recurring charges) are delivered via **webhook events**, which the client is expected to process idempotently. Stripe's docs advise returning a 2xx "prior to any complex logic that could cause a timeout" and retry non-2xx deliveries with exponential backoff for up to 3 days (webhook retry detail belongs to Part 7; Stripe does not publish an exact endpoint-timeout value).
- **Batch.** No generic batch endpoint.

*Confidence:* High on all header/format/retention/replay points (primary docs, verbatim). High on the 400-vs-409 split (subagent verified live-API bodies against Stripe SDK issue trackers; note the primary docs pages describe the 409 as "conflict" but do not print the JSON body). High on the absence of ETag/412 (absence-of-evidence across all three primary sources plus the status-code enumeration).

### 5.1 GitHub REST
Reliability character: **HTTP-native, read-optimized, no write-side idempotency layer.** GitHub supports conditional **reads** — `If-None-Match` with a returned `etag` (e.g., `etag: "644b5b0155e6404a9cc4bd9d8b1ae730"`) and `If-Modified-Since`, both returning `304 Not Modified` (and not counting against rate limit). But: "Conditional requests for unsafe methods, such as POST, PUT, PATCH, and DELETE are not supported unless otherwise noted." No idempotency key. Retry guidance is about rate limits (`retry-after`, `x-ratelimit-remaining`/`-reset`), and it advises spacing writes ≥1s and using webhooks over polling. No generic batch; no LRO operation resource in the core REST API.

### 5.2 Google Cloud / AIP (aip.dev)
Reliability character: **everything is a body field or a resource.** Idempotency via `request_id` (AIP-155): `string request_id = 4 [(google.api.field_info).format = UUID4]`, "restricted to 36 ASCII characters," optional, "providing a request ID must guarantee idempotency"; duplicate → return the previous successful response; honor for "any reasonable timeframe." Client libraries may auto-populate `request_id` (AIP client-libraries/4235). Concurrency via `etag` in the resource body (AIP-154): strong or weak, RFC 7232 quoting ("a valid etag is `"foo"`, not `foo`"), mismatch → `ABORTED` (HTTP 409), missing etag → allowed unless strong consistency required (then `INVALID_ARGUMENT`). LRO via AIP-151: methods return `google.longrunning.Operation` with `done`, `response`, `error`; must implement the Operations service; the significant-time threshold is "a good rule of thumb is 10 seconds"; may return an already-`done` operation for validate-only. AIP-151 also sets an operation-expiry rule-of-thumb of 30 days. The successor **AEP-155** renames the field `idempotency_key`, adds an `IdempotencyKey` component with `first_seen`, and cites the active IETF Idempotency-Key draft as motivation.

### 5.3 Microsoft (Azure REST guidelines + Microsoft Graph)
Reliability character: **standards-forward headers, two LRO flavors, real JSON batch.** Azure guidelines: POST idempotency via `Repeatability-Request-ID` (UUID) + `Repeatability-First-Sent` (IMF-fixdate) per OASIS Repeatable Requests v1.0; server returns `Repeatability-Result: accepted|rejected`; ops that don't support repeatability must return `501` when valid repeatability headers are present. ACS tracks a **5-minute** window; outside it → `412` "Repeatability first sent header was not in 5 minutes window." Concurrency: `If-Match`/`If-None-Match`/`If-Unmodified-Since` → `412`. LRO: `202 Accepted` + `Azure-AsyncOperation` (always prefer if present) or `Location`, plus `Retry-After`; `provisioningState` terminal values Succeeded/Failed/Canceled; clients must accept ≥4 KB URLs. Microsoft Graph: RELO ("resource-based long running operation," preferred) or stepwise operation modeled as a navigation property (`202` + `Location` → `GET …/operations/{id}` returning `status: notStarted|running|succeeded|failed`; OneDrive copy monitor returns `percentageComplete` and, on completion, a redirect to the result). Graph `$batch` JSON batching: POST to `https://graph.microsoft.com/v1.0/$batch`, body `{"requests":[{"id","method","url","body","headers","dependsOn"}]}`, **limited to 20 individual requests** with a combined payload under **4 MB** (a 21-item batch is rejected outright), each response carrying its own status, `424 Failed Dependency` cascading on `dependsOn`, and batches should be "either fully sequential or fully parallel."

### 5.4 Twilio
Reliability character: **header key, uneven coverage.** Twilio accepts an `Idempotency-Token` on select product endpoints — e.g., the Monitor Alarms API: "This operation is idempotent, ensuring that repeated requests with the same Idempotency-Token will not create duplicate alarms" (with a 100-alarm cap returning `409 Conflict` when exceeded). The IETF draft's implementation list records "Twilio uses custom HTTP header named `I-Twilio-Idempotency-Token` in webhooks" — but note that header is on **inbound webhooks Twilio sends to you** (to dedupe retries), a different direction from a client-supplied request key. Coverage of client-supplied idempotency across all write endpoints is incomplete. No documented generic ETag/concurrency or batch on the core REST API.

### 5.5 Shopify Admin REST (with currency caveat)
Reliability character: **legacy REST idempotency, migrating to GraphQL.** The REST Admin API is **legacy as of October 1, 2024**; all new apps must use GraphQL (from April 2025). REST idempotency uses a `unique_token` idempotency key; recommended UUID; "POST requests that process credit card payments, create billing attempts for subscriptions, or capture revenue details accept idempotency keys." In GraphQL, keys are tracked **24 hours**; concurrent duplicates return `IDEMPOTENCY_CONCURRENT_REQUEST`; completed duplicates return the cached response (with a caveat that the cached response is reconstructed from DB records and may differ slightly). Bulk uses async **Bulk Operations** (submit query → job id → poll → JSONL file), and the `@idempotent` directive applies **per row** in JSONL (each row needs its own key). **Currency caveat:** money amounts are dual-valued — `shop` currency vs. `presentment` currency (customer's). Transactions default to presentment; `?in_shop_currency=true` switches. Refund creation on multi-currency orders **must pass `currency`** (presentment) or errors. GraphQL exposes `MoneyBag` with both `shopMoney` and `presentmentMoney`. Any idempotent retry of a money-mutating request must preserve the currency field or risk a different (or failed) result.

### 5.6 Zalando RESTful API Guidelines (a guideline, not an API)
Reliability character: **the most explicit menu of options.** Three idempotency patterns (conditional `If-Match`/ETag; secondary key stored in body; client `Idempotency-Key` header), each with tradeoffs spelled out. The `Idempotency-Key` header spec (verbatim): UUID format, "Idempotency keys expire after 24 hours," a key cache storing response + optional request hash "regardless of whether it succeeded or failed," and an explicit note it is non-RFC ("Our only reference is the usage in the Stripe API"). Concurrency: ETag + `If-Match`/`If-None-Match` → `412` (with the nuance that failed `If-None-Match` on GET/HEAD should return `304`, not `412`). Batch: **MUST use `207`** for batch/bulk "even if all individual parts fail or are processed asynchronously." Async: MAY return `202`.

### 5.7 AWS (light contrast)
Reliability character: **body-field client tokens, scoped, SDK-generated.** `ClientToken` (a.k.a. `client-request-token`) is a body/parameter field, not a header — "a unique, case-sensitive string of up to 64 ASCII characters" (36 for some services). EC2 has **Regional vs. zonal** idempotency scoping; RunInstances chooses based on how the AZ is specified. Retry with same token + same params → succeeds without repeating; same token + different params → `IdempotentParameterMismatch` (EC2) / `ConflictException` (ECS). SDKs/CLI auto-generate a token if none is supplied. AWS's Builders' Library frames idempotency as "at most once," stores request parameters for deep validation, and (for EC2) bounds token validity to "the lifetime of the resource, plus an interval." Example: `ClientToken=550e8400-e29b-41d4-a716-446655440000`.

## 6. Agreements vs. Divergences

**Agreements.**
- RFC 9110 baseline is universal: PUT/DELETE idempotent, POST not; everyone adds a mechanism only for POST/PATCH.
- ETag + `If-Match` → `412` is the shared *concept* for optimistic concurrency (GitHub reads, Azure, Zalando, Google — though Google relocates it to the body).
- 24 hours is the modal retention window (Stripe, Shopify GraphQL, Zalando) — but not universal (Azure ACS 5 min; Google ≥1 hour/AEP; AWS resource-lifetime+interval).
- UUIDv4 is the near-universal recommended key format.
- `202 + poll` is the shared LRO skeleton; the disagreement is *what* you poll.

**Divergences (with tradeoffs).**
- **Header vs. body-field.** Header (Stripe/Twilio/Shopify) is transport-agnostic and easy to add via middleware, but non-standard and invisible in OpenAPI bodies. Body-field (Google/AWS) is self-documenting in the schema and survives protocol translation (gRPC↔REST), but couples the key to the message definition. Azure's header-pair adds a *timestamp* enabling bounded windows without unbounded key storage — at the cost of clock-sync fragility (ACS `412` on skew).
- **Replay of errors.** Stripe/Zalando cache and replay failures; Google/Google-Payments explicitly do **not** cache error results. Tradeoff: replaying a cached `400`/`500` gives deterministic retries but can strand a client on a stale error; not caching errors lets a transient failure be genuinely retried but risks duplicate side effects if the first attempt actually succeeded.
- **Same-key/different-body.** Stripe/AWS/IETF-fingerprint all reject as misuse (mismatch error). This protects against accidental key reuse but requires storing request parameters/fingerprints.
- **Concurrency present or absent.** Stripe deliberately has none (its whole model is retry-dedup, not lost-update prevention); Azure/Google/Zalando/GitHub-reads all provide it. Tradeoff: ETag concurrency prevents lost updates but pushes version-tracking onto clients (must GET, hold etag, handle `412`).
- **LRO operation-resource vs. header-polling.** Google/Graph-stepwise give a first-class, addressable, outlive-the-request operation object (discoverable, cancellable) at the cost of extra server state; Azure's header polling is lighter but the poll target is a URL you must extract from headers and its retention/semantics vary by service.
- **Batch mechanism & atomicity.** Graph `$batch` is a real envelope with dependencies but hard-capped at 20 and non-atomic; Zalando's `207` is a status convention, not an envelope format; Google retired heterogeneous batching entirely; file-based bulk trades latency for scale. No reference guarantees all-or-nothing batch atomicity by default — independent per-item results are the norm.

## 7. Contested Axes Register (scoped to this part)

| Axis | Options observed | Who does what | Tradeoff (one line) | How contested |
|---|---|---|---|---|
| **Idempotency carrier** | header / body-field / natural (PUT+client ID) | Header: Stripe, Twilio, Shopify(REST), Zalando(opt), IETF. Body: Google `request_id`, AWS `ClientToken`. Natural: GitHub, generic PUT | Header = transport-agnostic but non-schema; body = self-documenting but coupled to message | **Split** (roughly even; header leads for payments, body for cloud RPC) |
| **Key retention window** | 5 min / ≥1 h / 24 h / resource-lifetime | Azure ACS 5 min; Google ≥1 h (AEP); Stripe/Shopify/Zalando 24 h; AWS EC2 lifetime+interval | Short = cheap storage, strands late retries; long = safe late retries, storage growth | **Wide-open** (order-of-magnitude spread) |
| **Replay of errors** | replay cached error / never cache errors | Replay: Stripe, Zalando. Never: Google, Google Payments | Replay = deterministic; never = genuine retry but duplicate-risk | **Split** |
| **Same-key/different-body** | reject as mismatch / (unspecified) | Reject: Stripe (400), AWS (`IdempotentParameterMismatch`), IETF (fingerprint) | Rejecting needs stored params/fingerprint; protects against misuse | **Near-consensus** (those who address it agree) |
| **Concurrency placement** | ETag in header / etag in body / none | Header: GitHub(reads), Azure, Zalando. Body: Google AIP-154. None: Stripe | Header = HTTP-standard, cache-friendly; body = survives gRPC, explicit | **Split** |
| **Precondition required?** | required / recommended / optional / n-a | Google: required for declarative-friendly; Azure: strongly recommended; Zalando: SHOULD/MAY; Stripe: n/a | Requiring prevents lost updates but burdens all clients | **Split** |
| **LRO pattern** | operation resource / header polling / synchronous+webhook | Google, Graph(stepwise): op resource. Azure, Graph(RELO): header polling. Stripe: sync+webhook | Op resource = discoverable/cancellable, more state; headers = lighter, less uniform | **Split** |
| **Batch mechanism** | JSON `$batch` envelope / `207` convention / none / file-based | Graph: `$batch`. Zalando: `207`. Stripe/GitHub/Twilio: none. File: Shopify/Graph export | Envelope = one round-trip, capped; none = simple, chatty; file = scale, latency | **Wide-open** |
| **Batch atomicity** | independent items / dependency-ordered / all-or-nothing | Independent: Graph (default), Zalando, Google. Ordered: Graph `dependsOn`→424. All-or-nothing: none by default | Independent = partial success reporting; atomic = simpler client logic, rarer | **Near-consensus on independent** |

## 8. Examples Appendix (verbatim artifacts + concrete numbers)

**Stripe.**
- Request: `curl https://api.stripe.com/v1/charges -u sk_test_...: -H "Idempotency-Key: AGJ6FJMkGQIpHUTX" -d amount=2000 -d currency=usd -d description="Charge for Brandur" -d customer=cus_A8Z5MHwQS7jUmZ`
- Replay header: `Idempotent-Replayed: true`. Retry-hint header: `Stripe-Should-Retry: true|false`.
- Key length: ≤255 chars. Retention: ≥24h (prunable). Scope: per account. Format: V4 UUID recommended.
- Same-key/diff-body → HTTP 400, `type: idempotency_error`: "Keys for idempotent requests can only be used with the same parameters they were first used with."
- Concurrent in-flight → HTTP 409, `type: invalid_request_error`, `code: idempotency_key_in_use`: "There is currently another in-progress request using this Idempotent Key … Please try again later."
- Replay rule: "saving the resulting status code and body of the first request … regardless of whether it succeeds or fails … including 500 errors."
- Status-code table includes 400/401/402/403/409/424/429/5xx (no 412).

**IETF draft (07).**
- Header example: `Idempotency-Key: "8e03978e-40d5-43e8-bc93-6894a57f9324"` and random-string form `Idempotency-Key: "clkyoesmbgybucifusbbtdsbohtyuuwz"`.
- Dates: published 15 Oct 2025; expired 18 Apr 2026; Standards Track.
- Missing-key-where-required → `400` + `Link` to docs. Concurrent retry → resource conflict error. Retry after completion → prior result (success or error).

**Google AIP.**
- `string request_id = 4 [(google.api.field_info).format = UUID4];` "Restricted to 36 ASCII characters. // A random UUID is recommended."
- LRO: `rpc CreateBook(...) returns (google.longrunning.Operation) { option (google.longrunning.operation_info) = { response_type: "Book", metadata_type: "OperationMetadata" }; }`
- LRO poll (from AIP examples): `GET /v1/operations/xyz → 200 { "done": true, "response": { … } }`.
- AIP-154 etag: `// A valid etag is "foo", not foo`; mismatch → `ABORTED` (HTTP 409). Change log: "2021-03-05: Changed the etag error from FAILED_PRECONDITION (which becomes HTTP 400) to ABORTED (409)."
- Thresholds: LRO significant-time "rule of thumb is 10 seconds"; operation-expiry rule-of-thumb 30 days.

**Azure / OASIS.**
- `Repeatability-Request-ID` = client-generated GUID (36-char hex UUID); `Repeatability-First-Sent` = IMF-fixdate UTC; response `Repeatability-Result: accepted|rejected`; unsupported → `501`.
- ACS: "currently, the tracked duration is 5 minutes"; outside window → `412`.
- LRO initiation: `POST …/virtualMachines/{vm}/start?api-version=2019-12-01` → `202` with header `Azure-AsyncOperation: https://management.azure.com/…/operations/{operation-id}?api-version=…` and `Retry-After`. Terminal `provisioningState`: Succeeded/Failed/Canceled. Clients must accept ≥4 KB URLs.

**Microsoft Graph.**
- `$batch` body: `{ "requests": [ { "id": "1", "method": "GET", "url": "..." }, { "id": "2", "dependsOn": [ "1" ], "method": "GET", "url": "..." } ] }`; endpoint `POST https://graph.microsoft.com/v1.0/$batch`; **≤20 requests**; combined payload under **4 MB** (21-item batch rejected outright); `424 Failed Dependency` on failed `dependsOn`.
- LRO: `HTTP/1.1 202 Accepted` + `Location: https://graph.microsoft.com/v1.0/storage/operations/123`; poll `GET …/operations/{id}` → `{ "status": "inprogress"|"succeeded"|"failed" }`; OneDrive copy monitor → `{ "percentageComplete": 100.0, "resourceId": "…", "status": "completed" }`. Recommended poll interval for connector schema: every 1 minute.

**Twilio.**
- Monitor Alarms: "repeated requests with the same Idempotency-Token will not create duplicate alarms"; "up to 100 alarms per account"; over-limit → `409 Conflict`.
- Inbound webhook dedupe header: `I-Twilio-Idempotency-Token: idempotency-token-goes-here`.

**Shopify.**
- REST legacy notice: "The REST Admin API is a legacy API as of October 1, 2024."
- Idempotency: `unique_token`; UUID recommended; GraphQL tracks keys **24 hours**; in-flight duplicate → `IDEMPOTENCY_CONCURRENT_REQUEST`.
- Bulk `@idempotent` per JSONL row (each row its own key).
- Currency: default presentment; `?in_shop_currency=true`; `MoneyBag { shopMoney, presentmentMoney }`; refund on multi-currency order must pass `currency` (presentment) or error.

**Zalando.**
- `Idempotency-Key` header: `format: uuid`, `example: "7da7a728-f910-11e6-942a-68f728c1ba70"`, "Idempotency keys expire after 24 hours."
- "MUST use code 207 for batch or bulk requests."
- Concurrency: `If-Match: <entity-tag>` → `412` on mismatch.

**AWS.**
- EC2: `…&ClientToken=550e8400-e29b-41d4-a716-446655440000`; token "up to 64 ASCII characters" (36 for some services); Regional vs. zonal scoping; mismatch → `IdempotentParameterMismatch`; EC2 token valid ≥24h after instance termination.
- ECS: `"clientToken": "550e8400-e29b-41d4-a716-44665544"`; mismatch → `ConflictException`.

**GitHub.**
- Conditional read: `curl -H "if-none-match: \"644b5b0155e6404a9cc4bd9d8b1ae730\""` → `HTTP/2 304`.
- "Conditional requests for unsafe methods, such as POST, PUT, PATCH, and DELETE are not supported unless otherwise noted."

## 9. Caveats
- **Stripe's exact JSON error bodies** (in §5.0 for the 400/409 cases) are corroborated from live-API responses captured in Stripe's own SDK issue trackers; the primary docs pages describe the 409 as a "conflict" but do not print the JSON body verbatim. The 400-vs-409 distinction (parameter-mismatch vs. in-flight-conflict) corrects the widely repeated "409 idempotency_error" shorthand.
- **Stripe's webhook endpoint timeout is not published as a single figure.** Stripe docs advise returning 2xx "prior to any complex logic that could cause a timeout" and retrying non-2xx for up to 3 days; third-party sources cite conflicting timeout values (5s/20s/30s), so no exact number is asserted here. Webhook *retry* behavior is deferred to Part 7.
- **IETF draft is expired**, not withdrawn; a successor revision may re-appear. Treat it as descriptive of prevailing practice, not a normative standard.
- **Shopify's REST API is legacy**; the idempotency mechanics documented here are still live but the platform is steering to GraphQL, where the equivalent is the `@idempotent` directive. GraphQL/gRPC internals are out of scope; only the REST-side reliability posture is reported.
- **Google's global HTTP batch** deprecation is historic (announced 2018; turndown phased through 2019–2020); homogeneous per-API batch endpoints persist. Exact current per-API batch limits vary by service and were not enumerated.
- Some retention/window numbers are service-specific instances (e.g., Azure ACS 5 min) rather than platform-wide guarantees; do not generalize a single service's number to the whole vendor.
- Twilio's client-supplied idempotency coverage is uneven across endpoints; the `I-Twilio-Idempotency-Token` is an *inbound* (Twilio→you) webhook dedupe header, distinct from a client request key.