# Outbound Webhook Design Across Eight References + Standard Webhooks — Part 7 (Descriptive)

## TL;DR
- The field agrees on the fundamentals — HTTPS POST, HMAC-SHA256 signatures, at-least-once delivery, no ordering guarantee, consumer-side idempotency by event ID — but splits sharply on **envelope shape** (typed JSON event object vs header-driven vs form-encoded vs cloud-message wrapper), **signature header format** (compound `t=/v1=` vs bare `sha256=` hex vs base64 vs URL+params HMAC vs JWT), **retry windows** (minutes to 3 days), and **thin-vs-fat payloads**.
- Stripe is the richest reference: typed `evt_…` event object, compound `Stripe-Signature` header with timestamp + versioned HMAC, ~3-day retry, per-endpoint API-version pinning, a 30-day Events API for reconciliation, and a newer "thin events" model — but it is not canonical; Twilio (form-encoded, URL-based HMAC-SHA1), Microsoft Graph (subscription + validationToken handshake, thin by default), and AWS/Google (message-bus contrast models) each diverge on core axes.
- The **Standard Webhooks** spec (v1.0.0, Svix-initiated) is the emerging clig.dev-analog for webhooks: it codifies `webhook-id`/`webhook-timestamp`/`webhook-signature` headers, an HMAC-SHA256 scheme over `id.timestamp.payload`, `whsec_` secrets, and full-stop event naming — adopted by OpenAI, Anthropic, Google Gemini and others, but none of the eight legacy references have migrated their primary schemes to it. *(Correction 2026-08-09, count refined in review: the adopter list here came from the spec's README; `baseline-03d` verified it against each vendor's own docs — of the twelve named, five verify as independent emitters, three are platform/via-Svix, and four fail verification. Cite `baseline-03d` §D's corrected count, not the README list.)*

## Key Findings
1. **Envelope shape is the widest split.** Stripe/Standard Webhooks/Zalando use a typed JSON envelope with a `type` field; GitHub and Shopify push resource JSON in the body and put the event name in headers (`X-GitHub-Event`, `X-Shopify-Topic`); Twilio sends `application/x-www-form-urlencoded` form fields (no JSON, no envelope); Microsoft Graph wraps a `value[]` array of notification objects; AWS SNS wraps a JSON message with the real payload stringified in a `Message` field; Google Pub/Sub wraps a base64-encoded `message.data`.
2. **Signature schemes are nearly all HMAC-SHA256 but formatted incompatibly.** Stripe: `Stripe-Signature: t=…,v1=…` (hex). GitHub: `X-Hub-Signature-256: sha256=…` (hex) plus legacy SHA-1 `X-Hub-Signature`. Shopify: `X-Shopify-Hmac-Sha256` (base64). Standard Webhooks: `webhook-signature: v1,<base64>`. Twilio is the outlier: **HMAC-SHA1** over the URL + sorted POST params, base64, in `X-Twilio-Signature`. Graph uses no body HMAC by default — it relies on `clientState` echo + a `validationToken` handshake (and JWT validation tokens for rich/resource-data notifications). AWS SNS uses an RSA signature over canonicalized fields with a published cert URL. Google Pub/Sub uses an OIDC JWT bearer token.
3. **Timestamp/replay protection is not universal.** Stripe embeds `t=` in the signature and recommends a 300-second tolerance; Standard Webhooks signs the timestamp and its libraries default to a 5-minute window; Shopify and GitHub sign only the body (no timestamp in the signature → no built-in replay protection, though Shopify sends `X-Shopify-Triggered-At` and GitHub `X-GitHub-Delivery` for app-level dedup).
4. **Retry windows span three orders of magnitude.** GitHub: **no automatic retries** at all (manual redelivery within 3 days). Shopify: **8 attempts over 4 hours** (changed Sept 10, 2024 from 19/48h). Stripe: **up to three days**, exponential backoff (live mode). Microsoft Graph: retries **up to 4 hours**. AWS SNS: 3 retries default (~35s); AWS EventBridge: **24 hours / up to 185 attempts**. Standard Webhooks recommends a multi-day schedule with jitter.
5. **Ordering: unanimous "none."** Every reference explicitly disclaims ordering; consumers are told to use event timestamps and to dedupe by event ID.
6. **Thin-vs-fat is a live philosophical split even within Stripe.** Stripe classic ("snapshot") events are fat (`data.object` full resource); Stripe's newer "thin events" (v2) carry only IDs and require a fetch. Microsoft Graph is thin by default (notification carries IDs; you re-query, or opt into encrypted "rich" resource data). GitHub and Shopify are fat. Standard Webhooks documents both and leans toward thin for auditability/future-proofing.
7. **Subscription lifecycle is Graph's signature divergence.** Graph subscriptions expire (max ~3 days / 4,230 min for mail; other resources vary) and must be renewed via PATCH; creation requires a synchronous `validationToken` echo (200, text/plain, ≤10s). Stripe/Shopify endpoints are long-lived and managed via API; AWS SNS/Google Pub/Sub require a one-time subscription-confirmation handshake.
8. **Reconciliation sources differ.** Stripe uniquely offers a first-class Events API (`GET /v1/events`, 30-day retention) as a reconciliation backstop; Shopify and Graph tell you to reconcile against the main Admin/Graph API; GitHub offers a deliveries API for redelivery within 3 days.

## Baseline Position — Standard Webhooks Spec (v1.0.0) + HTTP semantics

**Retrieved 2026-07-19.** The Standard Webhooks specification (standardwebhooks.com; repo `standard-webhooks/standard-webhooks`; Version 1.0.0; Apache-2.0; Svix-initiated) is the closest thing to a cross-industry webhook convention. Its Technical Steering Committee members (per Svix's announcement) are Brian Cooksey (Zapier), Ivan Gracia (Twilio), Jorge Vivas (Lob), Matthew McClure (Mux), Nijiko Yonskai (ngrok), Stojan Dimitrovski (Supabase), Tom Hacohen (Svix), and Vincent Le Goff (Kong). It is a set of guidelines plus reference libraries in 9+ languages (Python, JS/TS, Java/Kotlin, Rust, Go, Ruby, PHP, C#, Elixir), not a schema-description language (it explicitly defers payload shape description to JSON Schema/OpenAPI/AsyncAPI). Its README lists adopters verbatim as "OpenAI, Anthropic, Google Gemini, Kong, Svix, Supabase, Vanta, Drata, Etsy, PagerDuty, Twilio, TaskRabbit and many others."

**Envelope (recommended, not required):** JSON body with:
- `type`: a full-stop-delimited, hierarchical event type (e.g. `user.created`, `invoice.paid`); identifiers limited to `[a-zA-Z0-9_]`.
- `timestamp`: ISO 8601, when the event occurred.
- `data`: the event data (may be squashed to top level).

Example:
```json
{ "type": "example.event", "timestamp": "2022-11-03T20:26:10.344522Z", "data": { "foo": "bar", "fizzbuzz": 2 } }
```

**Signature scheme:** three headers, all prefixed `webhook-`:
- `webhook-id` — unique message ID, stable across retries, doubles as idempotency key.
- `webhook-timestamp` — integer Unix seconds.
- `webhook-signature` — space-delimited list of signatures.

Signed content is `msg_id.timestamp.payload` (full-stop delimited), HMAC-SHA256 with the secret. Header value is version-prefixed: `v1,` for symmetric HMAC, `v1a,` for asymmetric ed25519, followed by base64. Example content string and header:
```
msg_2KWPBgLlAfxdpx2AI54pPJ85f4W.1674087231.{"type":"contact.created",...}
webhook-signature: v1,K5oZfzN95Z9UVu1EsfQmfVNQhnkZ2pj9o9NDN/H/pI4= v1a,hnO3f9T8Ytu9…
```
Multiple space-delimited signatures support zero-downtime secret rotation (sign with old + new key during cutover; consumer tries each until one matches). Secret: random 24 bytes (192 bits) to 64 bytes (512 bits), base64-encoded, prefixed `whsec_` (asymmetric: `whsk_`/`whpk_`). Replay protection: spec says verify timestamp "within some allowable tolerance"; reference libraries hardcode `WEBHOOK_TOLERANCE_IN_SECONDS = 5 * 60` (5 minutes past/future).

**Payload philosophy:** documents thin vs full; recommends payloads under ~20 KB; leans thin for performance, flexibility, future-proofing, and access auditability (with a large asset → send a URL).

**Delivery/operational recommendations:** at-least-once; retry over multiple days with exponential backoff + jitter (example schedule: immediately, 5s, 5m, 30m, 2h, 5h, 10h, 14h, 20h, 24h ≈ 75h total); success = `2xx`; `3xx` = failure (don't follow redirects); `410 Gone` → disable endpoint; `429`/`502`/`504` → throttle; honor `Retry-After`. Recommended request timeout 15–30s. Endpoint-management API and event-type filtering on producer side recommended.

**HTTP semantics baseline:** All references use HTTP POST over HTTPS (except Twilio permits HTTP in dev). "Success" is universally a `2xx`. There is no HTTP-level ordering or exactly-once guarantee; webhooks inherit HTTP's at-most-once-per-attempt semantics, which is why every provider layers retries (→ at-least-once) and every consumer must dedupe.

## Side-by-Side Comparison Tables

### Envelope
| Reference | Transport body | Event-type location | Envelope fields |
|---|---|---|---|
| Stripe (snapshot) | JSON event object | `type` field (`invoice.paid`) | `id` (`evt_…`), `object:"event"`, `api_version`, `created`, `data.object`, `data.previous_attributes`, `livemode`, `pending_webhooks`, `request{id,idempotency_key}`, `type` |
| Stripe (thin/v2) | JSON, minimal | `type` field | `id`, `type`, `created`, `related_object{id,type,url}`, `reason`; fetch full via API |
| GitHub | Resource JSON (or form-encoded) | `X-GitHub-Event` header + `action` field in body | body = resource(s) + `action` + `repository` + `sender`; headers carry event/delivery IDs |
| Shopify | Resource JSON | `X-Shopify-Topic` header (`orders/create`) | body = resource; metadata all in `X-Shopify-*` headers |
| Microsoft Graph | JSON `{"value":[…]}` | `changeType` field in each notification | per-notification: `subscriptionId`, `subscriptionExpirationDateTime`, `changeType`, `resource`, `resourceData{@odata.id,@odata.type,id}`, `clientState`, `tenantId`; optional `validationTokens[]` |
| Twilio | `application/x-www-form-urlencoded` | implicit (per configured URL / `EventType` param) | flat form params: `MessageSid`, `From`, `To`, `Body`, `AccountSid`, `MessageStatus`, etc.; no envelope |
| Zalando (guideline) | JSON | Event Type name (registered) | Event Type (name, category, owning app, schema, compatibility mode); "Data Change" category ~ CloudEvents-like; unique event id required |
| AWS SNS (contrast) | JSON text | none (topic-based) | `Type`, `MessageId`, `TopicArn`, `Message` (payload stringified), `Timestamp`, `SignatureVersion`, `Signature`, `SigningCertURL`, `UnsubscribeURL` |
| AWS EventBridge (contrast) | JSON | `detail-type` + `source` fields | `version`, `id`, `detail-type`, `source`, `account`, `time`, `region`, `resources[]`, `detail{}` |
| Google Pub/Sub (contrast) | JSON | none (topic + attributes) | `message{data(base64),messageId,publishTime,attributes{}}`, `subscription` |

### Event naming
| Reference | Convention | Tense/casing | Example |
|---|---|---|---|
| Stripe | `resource.verb` dot notation | past tense, snake_case segments | `invoice.paid`, `payment_intent.succeeded`, `customer.subscription.updated` |
| GitHub | header event + `action` field | event = noun, action = past-tense verb | event `issues` + `action: "opened"` |
| Shopify | `resource/verb` slash topic | present-tense verb, lowercase | `orders/create`, `products/update` (GraphQL: `ORDERS_CREATE`) |
| Microsoft Graph | `changeType` enum | present-tense words | `created`, `updated`, `deleted` |
| Twilio | per-product param values | mixed | `MessageStatus=delivered`; Verify events `challenge.approved` |
| Zalando | functional-name + hierarchical | registered functional names | e.g. `order-management…`; data-change vs general categories |
| Standard Webhooks | full-stop hierarchical | recommends past tense | `user.created`, `invoice.paid` |
| AWS EventBridge | free `detail-type` string | human phrase | `"EC2 Instance State-change Notification"` |

### Signatures
| Reference | Header | Algorithm | Encoding | Signed content | Replay protection |
|---|---|---|---|---|---|
| Stripe | `Stripe-Signature` | HMAC-SHA256 | hex | `t + "." + raw_body` | `t=` timestamp, 300s default tolerance |
| GitHub | `X-Hub-Signature-256` (+ legacy `X-Hub-Signature` SHA-1) | HMAC-SHA256 (SHA-1 legacy) | hex, `sha256=` prefix | raw body | none in signature (`X-GitHub-Delivery` for dedup) |
| Shopify | `X-Shopify-Hmac-Sha256` | HMAC-SHA256 | base64 | raw body | none in signature (`X-Shopify-Triggered-At` header) |
| Twilio | `X-Twilio-Signature` | HMAC-SHA1 | base64 | full URL + alphabetically-sorted POST params concatenated | none (idempotency via SID) |
| Microsoft Graph | none (basic) / JWT `validationTokens` (rich) | — / RS256 JWT | — | — | `clientState` echo; JWT for resource-data |
| Standard Webhooks | `webhook-signature` | HMAC-SHA256 (or ed25519) | base64, `v1,`/`v1a,` prefix | `id.timestamp.payload` | `webhook-timestamp`, 5-min lib default |
| AWS SNS | in-body `Signature` + `SigningCertURL` | RSA (SignatureVersion 1/2) | base64 | canonicalized field set | timestamp field |
| Google Pub/Sub | `Authorization: Bearer <JWT>` | OIDC RS256 | JWT | token claims (aud/email) | JWT `exp` |

### Delivery / retry
| Reference | Posture | Retry policy | Success | Timeout | Auto-disable |
|---|---|---|---|---|---|
| Stripe | at-least-once | up to 3 days, exp backoff (live); 3 tries/few hours (sandbox) | 2xx | "quickly"; community ~10–20s | after 3 days failing → disable + email |
| GitHub | fire-once | **none automatic**; manual redeliver ≤3 days | 2xx | 10s (Cloud) | disables after repeated failures |
| Shopify | at-least-once | **8 attempts / 4 hours** exp backoff (since 2024-09-10; was 19/48h) | 2xx (within 5s) | **5s** hard | 8 consecutive failures → subscription deleted (Admin API) |
| Microsoft Graph | at-least-once | retries up to **4 hours** | 2xx (202 recommended) | 10s (validation); notification deadline docs vary | subscription expiry; lifecycle events |
| Twilio | at-least-once (callbacks) | varies by product | 2xx (TwiML for messaging) | — | — |
| Standard Webhooks | at-least-once | multi-day + jitter (rec.) | 2xx | 15–30s rec. | `410` → disable |
| AWS SNS | at-least-once | 3 retries default, ~20s apart (~35s w/ timeout); custom delivery policy | 2xx | ~15s | — |
| AWS EventBridge | at-least-once | **24h, up to 185 attempts**, exp backoff + jitter | — | — | DLQ (SQS) for drops |
| Google Pub/Sub | at-least-once | configurable exp backoff; redeliver until ack | 102/200/201/204 = ack | ack deadline | dead-letter topic |

### Verification handshake / lifecycle
| Reference | Endpoint proof | Subscription model |
|---|---|---|
| Stripe | none (signature only) + IP allowlist option | long-lived; Webhook Endpoints API; ≤16 endpoints/account; event-type filtering at creation |
| GitHub | `ping` event on creation | long-lived; per-repo/org/app; event selection |
| Shopify | none (HMAC) | long-lived; REST/GraphQL subscription; topic filtering |
| Microsoft Graph | **`validationToken` echo** (200 text/plain ≤10s) | **expiring** (max ~3 days mail; renew via PATCH); `clientState`; lifecycle notifications |
| Twilio | none | configured per number/service; Verify has webhook resource API |
| AWS SNS | **SubscribeURL confirmation** (visit URL / ConfirmSubscription) | topic subscription; unsubscribe URL |
| Google Pub/Sub | none (JWT); no domain-ownership proof | push subscription config; ack deadline |
| Standard Webhooks | none prescribed | endpoint-management API recommended |

### Payload philosophy
| Reference | Default | Alternative | Rationale cited |
|---|---|---|---|
| Stripe | fat (snapshot `data.object`) | thin events (v2, fetch via API) | thin = version-stability, latest data, smaller payload |
| GitHub | fat (full resource) | — (25 MB cap; dropped if larger) | full context |
| Shopify | fat | — | full resource; reconcile via Admin API |
| Microsoft Graph | **thin** (IDs; re-query) | rich/encrypted resource data (beta/opt-in) | security/consistency; encryption for rich |
| Twilio | flat form params (medium) | — | callback context |
| Standard Webhooks | documents both | — | leans thin (audit, future-proof) |
| AWS/Google | payload in `Message`/`data` | — | bus decoupling |

## Per-Reference Notes

### Stripe (deep dive)
Stripe's webhook is a typed, self-describing JSON **Event object**. Verbatim envelope (snapshot/v1):
```json
{
  "id": "evt_1M7ABCD1234567890",
  "object": "event",
  "api_version": "2025-06-30.preview",
  "created": 1699900000,
  "data": { "object": { "id": "pi_3M7…", "object": "payment_intent", "amount": 5000, "currency": "usd", "status": "succeeded" } },
  "livemode": true,
  "pending_webhooks": 0,
  "request": { "id": null, "idempotency_key": null },
  "type": "payment_intent.succeeded"
}
```
Field semantics (from Stripe API reference / SDK typings): `api_version` = the Stripe API version used to render `data` **at event creation** (static thereafter; populated for events ≥ 2014-10-31); `data.previous_attributes` = changed fields for `*.updated` events; `livemode` boolean; `pending_webhooks` = count of endpoints not yet returning 2xx; `request.id`/`request.idempotency_key` = the API request that triggered the event (null if Stripe-automatic, e.g. subscription renewal).

**Naming:** `resource.verb`, past tense (`invoice.paid`, `customer.subscription.updated`); subresources like `invoice.payment_succeeded`.

**Signature:** `Stripe-Signature` header, single line, e.g.:
```
Stripe-Signature: t=1492774577, v1=5257a869e7ecebeda32affa62cdca3fa51cad7e77a0e56ff536d0ce8e108d8bd, v0=6ffbb59b…
```
`t=` = Unix timestamp; `v1=` = HMAC-SHA256 hex of `"{t}.{raw_body}"` keyed by the endpoint's `whsec_…` secret; `v0=` is a test-only fake scheme. Verify with `stripe.webhooks.constructEvent(rawBody, sigHeader, secret)`; default tolerance 300s; must use raw body (JSON re-serialization breaks HMAC). Secret rotation: roll in Dashboard/Workbench; old secret can be expired immediately or after up to 24h; during overlap Stripe generates one signature per active secret. IP allowlisting offered as defense-in-depth.

**Delivery:** at-least-once; **live mode retries up to three days with exponential backoff** (Stripe docs, verbatim: "Stripe attempts to deliver events to your destination for up to three days with an exponential back off in live mode"; the frequently-cited "~16 attempts" figure appears only in secondary sources, not Stripe's own docs); sandbox retries three times over a few hours. Any non-2xx (including 4xx) triggers retry. Events may arrive out of order (use `created`); duplicates possible (dedupe by `id`). After 3 days failing → endpoint disabled + email.

**Lifecycle/reconciliation:** Webhook Endpoints (event destinations) API; ≤16 endpoints/account; event-type selection at creation; API-version pinning per endpoint (up to 3 uniquely-versioned snapshot destinations). **Events API (`GET /v1/events`, up to 30 days retention)** is the reconciliation backstop; filter by up to 20 types, cursor paginate. Manual resend: Dashboard within 15 days, CLI within 30 days.

**Thin events:** newer v2 model — notification carries `id`, `type`, `related_object{id,type,url}`, `reason`; you call `fetchEvent()`/`fetchRelatedObject()` or `stripe.events.retrieve()`. Thin and snapshot payloads **cannot be mixed on one endpoint** and each destination has its own signing secret. Thin-events-for-v1-resources is in private preview (per docs retrieved 2026-07-19). Recommended consumer architecture: verify → enqueue → return 200 fast → process async; never inline heavy work.

### GitHub
Header-driven fat webhook. Verbatim request head:
```
POST /payload HTTP/1.1
X-GitHub-Delivery: 72d3162e-cc78-11e3-81ab-4c9367dc0958
X-Hub-Signature: sha1=7d38cdd689735b008b3c702edd92eea23791c5f6
X-Hub-Signature-256: sha256=d57c68ca6f92289e6987922ff26938930f6e66a2d161ef06abdf1859230aa23c
User-Agent: GitHub-Hookshot/044aadd
Content-Type: application/json
X-GitHub-Event: issues
X-GitHub-Hook-ID: 292430182
X-GitHub-Hook-Installation-Target-ID: 79929171
X-GitHub-Hook-Installation-Target-Type: repository
{ "action": "opened", "issue": {…}, "repository": {…}, "sender": {…} }
```
Event type in `X-GitHub-Event`; sub-action in body `action`. Payload can be JSON or `x-www-form-urlencoded`. **25 MB payload cap** (larger events not delivered). Signature: HMAC-SHA256 hex with `sha256=` prefix (validate with `timingSafeEqual`); legacy SHA-1 `X-Hub-Signature` kept for compatibility. **No timestamp in signature → no built-in replay protection.** Delivery: return 2xx within 10s. **No automatic retries** — the sharpest divergence; manual redelivery via UI/REST API within 3 days; recommended pattern is a scheduled script using the deliveries API to detect+redeliver failures. Dedupe by `X-GitHub-Delivery`.

### Shopify
Header-driven fat webhook. Metadata headers: `X-Shopify-Topic` (`orders/create`), `X-Shopify-Hmac-Sha256` (base64), `X-Shopify-Shop-Domain`, `X-Shopify-Webhook-Id` (per-delivery, dedupe key), `X-Shopify-Event-Id` (per merchant action), `X-Shopify-Triggered-At`, `X-Shopify-Api-Version`. Signature: base64 HMAC-SHA256 of the **raw body** keyed by app client secret — note base64, not hex (a common cross-provider gotcha). Secret rotation: after rotating client secret, up to ~1h for new-secret HMACs. **Retry policy changed 2024-09-10** — Shopify's dev changelog states verbatim: "Webhooks will now be retried a total of 8 times over 4 hours using an exponential backoff schedule" (previously 19 over 48h; older sources still cite the old numbers — flag descriptively). **5-second hard timeout.** 8 consecutive failures → subscription auto-deleted (if Admin-API-created) + email to emergency dev address; re-registration required. No ordering; expect 2–5% duplicates (dedupe `X-Shopify-Webhook-Id`; correlate merchant action via `X-Shopify-Event-Id`). Payloads versioned like Admin API; on version sunset, "fall-forward" to oldest supported (signal via `X-Shopify-Api-Version` response header). Reconcile via Admin API.

### Microsoft Graph (subscription model)
Subscription-based, thin-by-default. Create via `POST /subscriptions` with `changeType`, `notificationUrl`, `resource`, `expirationDateTime`, `clientState`, optional `lifecycleNotificationUrl`. **Validation handshake:** Graph POSTs `?validationToken=…`; endpoint must return it as text/plain, 200, within 10s, or creation fails. Notification envelope:
```json
{ "value": [ { "subscriptionId":"…", "subscriptionExpirationDateTime":"…", "changeType":"created", "resource":"…", "resourceData":{"@odata.id":"…","@odata.type":"…","id":"…"}, "clientState":"…", "tenantId":"…" } ] }
```
Authenticity: **`clientState` echo comparison** (basic) — no body HMAC; for rich notifications (resource data), a `validationTokens[]` array of JWTs must be validated against `login.microsoftonline.com` keys, and resource data is encrypted with your supplied cert. Delivery: retries up to 4 hours if access token expired/endpoint down; respond 202 quickly. **Expiring subscriptions** (max varies: e.g. mail ~4,230 min / <3 days; others differ; any request under 45 min is auto-set to 45 min) → renew via PATCH; lifecycle notifications (`reauthorizationRequired`, `subscriptionRemoved`, `missed`) keep channels alive; pair with delta queries to backfill misses. Thin by default; you re-query the resource by ID.

### Twilio (callback model — notable divergence)
Twilio's "webhooks" are HTTP callbacks that carry **`application/x-www-form-urlencoded`** bodies — no JSON envelope, no event `type` field; the fields vary by product (SMS: `MessageSid`, `From`, `To`, `Body`, `AccountSid`; status callbacks: `MessageStatus`, etc.) and Twilio adds new params without notice. Signature: **`X-Twilio-Signature`, HMAC-SHA1** (not SHA-256), keyed by the account AuthToken, base64, computed over the **exact full request URL + all POST params sorted alphabetically and concatenated (name+value, no delimiter)**. This URL-dependence makes it brittle behind proxies/load balancers (scheme/port/path rewrites break it; empty params must be preserved). Verify via SDK `RequestValidator`/`validateRequest`. No timestamp → replay possible; idempotency via the resource SID. Supports HTTP Basic/Digest auth on the callback URL and optional SSL cert validation; sends from an IP pool (no fixed allowlist); adds `X-Home-Region` header. Newer Twilio products (e.g. Event Streams / Verify) offer JSON webhooks; Twilio sits on the Standard Webhooks steering committee.

### Zalando (event guidelines within API guidelines)
Zalando doesn't ship a public outbound-webhook product; its RESTful API and Event Guidelines treat **events as first-class interface artifacts** published to an internal event bus (Nakadi-style), registered as **Event Types** (name, well-known Event Category [General vs Data Change], owning application/team, OpenAPI Schema Object, compatibility mode). Rules relevant to this surface: events MUST conform to a well-known category; MUST carry unique event identifiers; SHOULD be designed for idempotent consumption; SHOULD provide a means for explicit event ordering; Data Change Events SHOULD match API representations and use hash partitioning; MUST maintain backward compatibility; SHOULD avoid `additionalProperties`. Event/hostname/permission names follow a registered **functional naming schema** (`<functional-domain>-<functional-component>`). It is a governance/schema stance, not an HTTP-signature/retry spec — closest in spirit to Standard Webhooks + AsyncAPI.

### Google (contrast — Pub/Sub push)
Google's cross-service eventing is **Pub/Sub**, not classic per-product webhooks. A **push subscription** turns Pub/Sub into a webhook-like deliverer: HTTPS POST with body `{"message":{"data":"<base64>","messageId":"…","publishTime":"…","attributes":{}},"subscription":"projects/…/subscriptions/…"}`. The real payload is base64 in `message.data`. Authenticity: **OIDC JWT bearer token** in `Authorization` header (verify signature, `aud`, and service-account `email`), not an HMAC over the body. Ack = return 102/200/201/204; any other code = nack → redelivery. Retries: configurable exponential backoff; dead-letter topic for poison messages. No domain-ownership proof; endpoint needs a CA-signed TLS cert. Implies: cloud-native identity-based auth + at-least-once bus semantics, contrasting with shared-secret HMAC webhooks. (Some Google products, e.g. Gemini, adopt Standard Webhooks directly.)

### AWS (contrast — SNS / EventBridge)
AWS offers **SNS** (pub/sub fanout to HTTP/S endpoints) and **EventBridge** (event bus + API destinations) rather than classic webhooks. **SNS HTTP/S:** POST with `x-amz-sns-message-type` header (`SubscriptionConfirmation`/`Notification`/`UnsubscribeConfirmation`) and JSON body carrying `Type`, `MessageId`, `TopicArn`, `Message` (the actual payload, stringified), `Timestamp`, `SignatureVersion`, `Signature`, `SigningCertURL`, `UnsubscribeURL`. **One-time subscription confirmation:** endpoint must visit `SubscribeURL` (or call `ConfirmSubscription` with `Token`, valid 2 days). Authenticity: recreate an **RSA signature** from canonicalized fields using the published cert. Retries: 3 by default (~20s apart, ~35s with timeouts); custom delivery policy; 5xx/429 retryable, others permanent. **EventBridge** envelope: `{version,id,detail-type,source,account,time,region,resources[],detail{}}`; at-least-once delivery with **exponential backoff — AWS docs verbatim: "By default, EventBridge retries sending the event for 24 hours and up to 185 times with an exponential back off and jitter"**; DLQ (SQS) for undelivered; schema registry + discovery; 256 KB max event. Implies: durable bus decoupling, IAM/signature auth, and archive/replay instead of a provider-hosted event log.

## Agreements vs Divergences

**Agreements (near-consensus):**
- HTTPS POST transport; `2xx` = success (Pub/Sub also accepts 102).
- At-least-once delivery + explicit "no ordering guarantee" (Stripe, Shopify, Graph, SNS, EventBridge, Pub/Sub, Standard Webhooks).
- Consumer must dedupe by a unique event/delivery ID (`evt_…` / `X-GitHub-Delivery` / `X-Shopify-Webhook-Id` / `webhook-id` / `MessageId`).
- HMAC-SHA256 is the dominant body-signing primitive (Stripe, GitHub, Shopify, Standard Webhooks) — Twilio (SHA-1), Graph (clientState/JWT), AWS (RSA), Google (JWT) diverge.
- Recommended consumer architecture is uniform: verify → ack fast → process async (queue-first).

**Divergences (with tradeoffs, descriptive):**
- **Envelope typed vs header-driven vs form-encoded vs bus-wrapped.** Typed (Stripe/SW) is self-describing and routable without headers but couples schema to versioning; header-driven (GitHub/Shopify) keeps bodies as clean resource representations but forces header parsing; form-encoded (Twilio) is trivially parseable by legacy web frameworks but can't nest and lacks an envelope; bus-wrapped (SNS/Pub/Sub) double-encodes the payload, adding a decode step.
- **Signature format.** Compound `t=,v1=` (Stripe) and `v1,`/`v1a,` (SW) bind timestamp + versioning for rotation and replay defense at the cost of a custom parser; bare `sha256=`/base64 (GitHub/Shopify) is simpler but omits replay protection; URL-based HMAC (Twilio) authenticates the destination too but breaks behind proxies; JWT/RSA (Google/AWS) leverages PKI/identity but needs key-fetch + cert-pin logic.
- **Retry window.** None (GitHub) → 4h (Shopify/Graph) → 24h (EventBridge) → 3 days (Stripe). Short windows + auto-disable (Shopify) protect the sender but punish consumer outages; long windows (Stripe) tolerate multi-day downtime; no-retry (GitHub) shifts all reliability onto the consumer's polling.
- **Thin vs fat.** Fat avoids a round-trip but risks stale/oversized payloads and leaks data to every endpoint; thin (Graph default, Stripe v2) guarantees fresh data, enables access auditing, and decouples payload from API version at the cost of an extra fetch and rate-limit exposure.
- **Verification handshake.** Graph's `validationToken` echo and AWS's `SubscribeURL` confirm endpoint ownership before sending; Stripe/GitHub/Shopify skip ownership proof and rely on signatures.
- **Reconciliation source.** Stripe's dedicated Events API (30-day) is unique; others reconcile against the main API (Shopify/Graph) or offer only short redelivery windows (GitHub 3 days).

## Contested Axes Register (scoped to Part 7)

| Axis | Options observed | Who does what | Tradeoff (one line) | How contested |
|---|---|---|---|---|
| Envelope shape | typed JSON object / header+resource / form-encoded / bus-wrapper | Stripe, SW, Zalando (typed); GitHub, Shopify (header+resource); Twilio (form); SNS, Pub/Sub (wrapper); EventBridge (typed bus) | Self-describing routing vs clean resource bodies vs legacy-parseable vs double-encoded | **Wide-open** |
| Naming convention | `resource.verb` past / header+`action` / `resource/verb` present / `changeType` enum / free string | Stripe, SW (dot past); GitHub (header+action); Shopify (slash present); Graph (enum); EventBridge (free) | Dotted past-tense self-documents; header/enum needs lookup | **Split** |
| Signature scheme | HMAC-SHA256 / HMAC-SHA1 / clientState+JWT / RSA / OIDC JWT | Stripe, GitHub, Shopify, SW (SHA-256); Twilio (SHA-1); Graph (clientState/JWT); SNS (RSA); Pub/Sub (JWT) | Shared-secret simple but symmetric; PKI/identity stronger but heavier | **Near-consensus on HMAC-SHA256**, with notable outliers |
| Timestamp/replay protection | signed timestamp w/ tolerance / none / JWT exp | Stripe (300s), SW (300s lib); GitHub, Shopify (none in sig); Graph/Google (JWT exp) | Replay defense vs simpler signing | **Split** |
| Retry policy shape | none / 4h / 24h / 3 days / multi-day | GitHub (none); Shopify, Graph (4h); EventBridge (24h/185); Stripe (3 days); SNS (3/~35s); SW (multi-day rec.) | Sender protection + auto-disable vs consumer-outage tolerance | **Wide-open** |
| Ordering posture | no guarantee (universal) | all references | Simplicity + scalability vs consumer must reorder | **Consensus (none)** |
| Thin vs fat payload | fat / thin / both | GitHub, Shopify, Stripe-snapshot (fat); Graph, Stripe-v2 (thin); SW (both) | Round-trip avoidance vs freshness/audit/version-decoupling | **Split** |
| Verification handshake | ownership proof / signature-only | Graph (validationToken), SNS (SubscribeURL) (proof); Stripe, GitHub, Shopify, Twilio, Pub/Sub (none) | Anti-SSRF/spoof registration vs simpler onboarding | **Split** |
| Event versioning | per-endpoint pinning / API-version fall-forward / schema registry / none | Stripe (pin, `api_version`); Shopify (fall-forward + header); EventBridge, Zalando (registry/compat mode); GitHub/Twilio (add fields silently) | Explicit pinning stable but managed; silent evolution simpler but brittle | **Split** |
| Retention/replay window | 30-day events API / 3-day redeliver / reconcile via main API / archive-replay / DLQ | Stripe (30d Events API); GitHub (3d); Shopify, Graph (main API); EventBridge (archive), Pub/Sub/SNS (DLQ) | First-class replay store vs external reconciliation burden | **Split** |

## Examples Appendix

**Standard Webhooks** (v1.0.0, Apache-2.0; retrieved 2026-07-19):
- Envelope: `{ "type": "example.event", "timestamp": "2022-11-03T20:26:10.344522Z", "data": {...} }`
- Signed content: `msg_2KWPBgLlAfxdpx2AI54pPJ85f4W.1674087231.{payload}`
- Header: `webhook-signature: v1,K5oZfzN95Z9UVu1EsfQmfVNQhnkZ2pj9o9NDN/H/pI4= v1a,hnO3f9T8Ytu9…`
- Headers: `webhook-id`, `webhook-timestamp` (int unix seconds), `webhook-signature`
- Secret: `whsec_` + base64, 24–64 bytes; tolerance `WEBHOOK_TOLERANCE_IN_SECONDS = 5*60`
- Retry example: immediately, 5s, 5m, 30m, 2h, 5h, 10h, 14h, 20h, 24h (~75h)
- Adopters (README verbatim): OpenAI, Anthropic, Google Gemini, Kong, Svix, Supabase, Vanta, Drata, Etsy, PagerDuty, Twilio, TaskRabbit

**Stripe** (retrieved 2026-07-19):
- Event object with `id evt_…`, `api_version`, `data.object`, `data.previous_attributes`, `livemode`, `pending_webhooks`, `request{id,idempotency_key}`, `type`
- `Stripe-Signature: t=1492774577, v1=5257a869e7ecebeda32affa62cdca3fa51cad7e77a0e56ff536d0ce8e108d8bd, v0=6ffbb59b…`
- HMAC-SHA256 hex over `"{t}.{raw_body}"`; secret `whsec_…`; tolerance 300s
- Retry (docs verbatim): "up to three days with an exponential back off in live mode"; sandbox 3/few hours; ≤16 endpoints; Events API 30-day; resend Dashboard 15d / CLI 30d
- Thin event: `{id, type, related_object{id,type,url}, reason}`; separate secret per destination

**GitHub** (retrieved 2026-07-19):
- Headers `X-GitHub-Event: issues`, `X-GitHub-Delivery: 72d3162e-…`, `X-Hub-Signature-256: sha256=d57c68ca…`, legacy `X-Hub-Signature: sha1=7d38cdd…`, `User-Agent: GitHub-Hookshot/…`
- Body `{ "action": "opened", "issue": {…}, "repository": {…}, "sender": {…} }`
- 25 MB cap; 10s timeout; no auto-retry; manual redeliver ≤3 days

**Shopify** (retrieved 2026-07-19):
- Headers `X-Shopify-Topic: orders/create`, `X-Shopify-Hmac-Sha256` (base64), `X-Shopify-Webhook-Id`, `X-Shopify-Event-Id`, `X-Shopify-Triggered-At`, `X-Shopify-Api-Version`, `X-Shopify-Shop-Domain`
- HMAC-SHA256 base64 over raw body, keyed by app client secret
- Retry (changelog verbatim): "retried a total of 8 times over 4 hours using an exponential backoff schedule" (since 2024-09-10; prev 19/48h); 5s timeout; 8 failures → subscription deleted
- GraphQL topic `ORDERS_CREATE`; REST topic `orders/create`

**Microsoft Graph** (retrieved 2026-07-19):
- Create: `POST /subscriptions {changeType,notificationUrl,resource,expirationDateTime,clientState,lifecycleNotificationUrl}`
- Validation: echo `?validationToken=…` as text/plain 200 ≤10s
- Notification `{ "value":[{subscriptionId,changeType,resource,resourceData{@odata.id,@odata.type,id},clientState,tenantId}] }`
- Lifecycle events `reauthorizationRequired|subscriptionRemoved|missed`; retry ≤4h; renew via PATCH; mail max ~4,230 min

**Twilio** (retrieved 2026-07-19):
- `Content-Type: application/x-www-form-urlencoded`; params e.g. `MessageSid`, `From`, `To`, `Body`, `AccountSid`, `MessageStatus`
- `X-Twilio-Signature`: base64 HMAC-SHA1 over (full URL + alphabetically-sorted POST params concatenated), keyed by AuthToken
- `X-Home-Region` header; IP pool; SDK `RequestValidator`

**AWS SNS** (retrieved 2026-07-19):
```
POST / HTTP/1.1
x-amz-sns-message-type: Notification
{ "Type":"Notification","MessageId":"22b80b92-…","TopicArn":"arn:aws:sns:…","Message":"Hello world!","Timestamp":"2012-05-02T00:54:06.655Z","SignatureVersion":"1","Signature":"EXAMPLE…","SigningCertURL":"https://sns.us-west-2.amazonaws.com/…pem","UnsubscribeURL":"…" }
```
- Confirmation: `SubscriptionConfirmation` with `SubscribeURL`/`Token` (valid 2 days); default 3 retries ~20s apart

**AWS EventBridge** (retrieved 2026-07-19):
```json
{ "version":"0","id":"1d7a4ac6-…","detail-type":"Scheduled Event","source":"aws.events","account":"123456789012","time":"2016-12-30T18:44:49Z","region":"us-east-1","resources":["arn:aws:events:…:rule/SampleRule"],"detail":{} }
```
- at-least-once; retry 24h / up to 185 attempts exp backoff + jitter; DLQ via SQS; 256 KB max

**Google Pub/Sub** (retrieved 2026-07-19):
```json
{ "message": { "data":"SGVsbG8gQ2xvdWQgUHViL1N1YiE=","messageId":"2070443601311540","publishTime":"2021-02-26T19:13:55.749Z","attributes":{} }, "subscription":"projects/myproject/subscriptions/mysubscription" }
```
- Auth: `Authorization: Bearer <OIDC JWT>` (verify sig/aud/email); ack = 102/200/201/204; dead-letter topic

## Caveats
- **Currency/volatility:** Retry numbers and thin-event availability change. All figures retrieved 2026-07-19. Shopify's retry policy demonstrably changed (19/48h → 8/4h on 2024-09-10) and many secondary sources still cite the old numbers — treat any "19 attempts / 48 hours" claim as pre-Sept-2024. Stripe's thin-events-for-v1 is in private preview as of retrieval.
- **Secondary-source reliance for numbers:** Stripe does not publish an exact per-attempt schedule or a numeric endpoint timeout; the "~16 attempts" and "10–20s timeout" figures come from Stripe-engineering-adjacent secondary sources and community docs, not a single authoritative Stripe table (the enricher confirmed "~16 attempts" appears only in secondary sources such as eventdock.app). Confidence: medium. The "up to three days," "30-day Events API," "≤16 endpoints," and "15-day Dashboard / 30-day CLI resend" figures are from Stripe's own docs (high confidence).
- **Graph timeout/latency:** docs and secondary guides vary on the exact notification-processing deadline (202 recommended; validation 10s is firm). Subscription max lifetimes vary per resource — the ~4,230-min mail figure is one documented value; verify per resource.
- **Twilio JSON webhooks:** Twilio's classic messaging/voice callbacks are form-encoded; some newer Twilio products emit JSON and Twilio participates in Standard Webhooks — so "Twilio = form-encoded" is true of its historically dominant callback surface, not every Twilio product.
- **Zalando:** describes an internal event-bus/Event-Type governance model, not an outbound HTTP-webhook product; mapping it onto the webhook axes (signature/retry) is partial by nature — it contributes naming, versioning, and idempotency stances only.
- **Confidence on divergence classifications:** the Contested Axes "How contested" labels are analytical judgments from the assembled evidence, not quantified metrics.