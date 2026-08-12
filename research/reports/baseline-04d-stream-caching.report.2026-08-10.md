# baseline-04d — Caching posture for a streaming response (report, 2026-08-10)

Research leaf under `baseline-04-streaming`, answering the §13.4 register row
**"Caching posture for a stream (§7)"** and `PLAN.md` Phase 8 item 3. Series
`baseline` = prescriptive: **this report proposes; only a ratified record in
`research/decisions/` binds.** `baseline-04-streaming.decision.md` is ratified
and is not re-litigated here.

Label key: **[FACT]** primary-sourced · **[COMPARATIVE]** surveyed practice ·
**[INFERENCE]** reasoning from the above · **[POLICY]** a judgment the project
must make.

All web sources accessed **2026-08-10** unless a different access date is given
on the row.

---

## 1. TL;DR and recommendation

**Verdict: a rule is needed, but not the one the register row implies. Do not
mandate `no-store`. Keep R7.3's three tiers and add one narrow rule that says
how they read on a stream.**

Four findings drive this, in descending order of load.

**Finding 1 — the premise of the §13.4 row is partly wrong, and the rest is not
a conflict.** The row says R7.1's MUST meets machinery "a stream cannot supply."
Two separate claims hide in that sentence and they have different answers.

- R7.1 requires an explicit `Cache-Control`. A stream can supply that: response
  header fields are sent before the body exists, and `Cache-Control` is computed
  from the resource's policy, not from the bytes. **R7.1 is satisfiable on a
  stream with no amendment at all** — E.11 already satisfies it. **[INFERENCE]**
- R7.3's tier-1 *default* pairs `private, no-cache` with strong-`ETag`
  revalidation (R3.10). That pairing is what a live-generated stream cannot
  supply. But **R7.3 contains no BCP 14 keyword in any of its three tiers**
  (rest-api-standard.md:1217–1226 — "the default posture is three-tier",
  "defaults to", "is reserved for", "is permitted only for"), and R1.1
  (rest-api-standard.md:103–107) makes the keywords normative "when, and only
  when, they appear in all capitals." **E.11's `private, no-store` therefore
  breaches nothing.** This is an unpicked default, not a binding conflict.
  **[FACT]** on the text, **[INFERENCE]** on the consequence.

**Finding 2 — the real cacheability hazard is narrower than "streams," and it
splits on method.** RFC 9111 §4 (RFC 9111:500–503): "A cache MUST write through
requests with methods that are unsafe … a cache is not allowed to generate a
reply to such a request before having forwarded the request and having received
a corresponding response." **[FACT]** Every AI-provider streaming endpoint
surveyed is a `POST`, so no cache may answer a later request from a stored copy
of one. RFC 9110 §9.3.3 closes the storage side too: "Responses to POST requests
are only cacheable when they include explicit freshness information … and a
Content-Location header field that has the same value as the POST's target URI."
**[FACT]** A silent `POST` stream carries neither, so it is not storable at all.
RFC 9111 §3.3's incomplete-response storage permission is likewise
**`GET`-only** (RFC 9111:418–421). **[FACT]** The live hazard is therefore the
`GET` stream — which is every `EventSource` stream, every Kubernetes watch,
every Docker `/events` tail — not the `POST` completion stream that dominates
the AI field. **[INFERENCE]**

**Finding 3 — for that `GET` stream, silence is the actual defect, and R7.1
already forbids it.** A `GET`, `200`, `text/event-stream` response with no
`Cache-Control` satisfies every RFC 9111 §3 storage precondition: the status is
final, `no-store` is absent, and `200` is a status "defined as heuristically
cacheable" (RFC 9110:6953). §3.3 then permits a cache to store the response
*while it is still arriving*. **[FACT]** Four of the nine surveyed streaming
implementations send no `Cache-Control` at all — OpenAI, Gemini, Elasticsearch,
Docker — **of which only Docker's `/events` is a `GET`**; the three `POST` cases
are foreclosed by Finding 2 instead. **[COMPARATIVE]** The population genuinely
exposed is small but not empty, and it includes this standard's own worked
example: E.11's `GET /v1/order-exports/{export_id}/events` sits squarely in it.
R7.1's always-emit MUST is the rule that closes the case, and it is already
ratified. **[INFERENCE]**

**Finding 4 — `no-store` is not the field's answer, and `no-cache` is
sufficient for correctness.** The modal value across implementations that emit
anything is `no-cache` (Anthropic, Kubernetes, MDN, Cloudflare Agents, the
nginx-cookbook genre); ASP.NET Core sends `no-cache,no-store`; only Zalando's
whole-API default is `no-store`, and it is not stream-specific.
**[COMPARATIVE]** RFC 9111 §4 makes `no-cache` block *reuse* outright
(RFC 9111:480–482); `no-store` adds only the at-rest storage prohibition
(RFC 9111 §5.2.2.5), which is a confidentiality property of the payload, not of
the response shape. **[FACT]** Mandating `no-store` for every stream would
therefore buy no correctness and would contradict R7.3 tier 2's own reasoning
without new evidence for it.

**Proposed rule** (new `R13.12`, in §13.1 or a new §13.6; full text and
rationale in §5):

> **R13.12** R7.1 binds a streaming response unchanged: the header section
> precedes the body, so an explicit `Cache-Control` is always computable and a
> stream MUST carry one. Within R7.3's posture, a streaming response takes
> tier 1 — `private, no-cache` — by default, and **tier 1's strong-`ETag`
> revalidation clause does not apply to it**: R3.10 binds resources that support
> conditional update, and a stream is not one. Tier 2 (`no-store`) applies to a
> stream on the same condition as to any other response — a genuinely sensitive
> payload — and MUST NOT be adopted merely because the response is a stream.
> Tier 3 (`public` with `max-age`) MAY be used only for a stream that is a view
> over an immutable retained artifact (R13.10), and then only where every
> resumption input is either part of the request URI or named in `Vary` (R4.11);
> a stream unbounded by design (R13.6) MUST NOT use tier 3.

**Confidence: moderate-high.** The protocol reasoning is certain; the choice of
`no-cache` over `no-store` as the stream default is `[POLICY]`, supported by the
modal field practice but with one authoritative guideline (Zalando) dissenting.

---

## 2. Standards-and-currency matrix

| Source | URL | Authority class | Published / accessed |
| --- | --- | --- | --- |
| RFC 9111, *HTTP Caching* | https://www.rfc-editor.org/rfc/rfc9111.txt | IETF Standards Track (STD 98) | Published 2022-06; accessed 2026-08-10 |
| RFC 9110, *HTTP Semantics* | https://www.rfc-editor.org/rfc/rfc9110.txt | IETF Standards Track (STD 97) | Published 2022-06; accessed 2026-08-10 |
| WHATWG HTML Living Standard, *Server-sent events* | https://html.spec.whatwg.org/multipage/server-sent-events.html | Living standard (not an RFC; no IANA registration for `text/event-stream`) | Living; accessed 2026-08-10 |
| WHATWG Fetch Standard, request cache mode | https://fetch.spec.whatwg.org/ | Living standard | Living; accessed 2026-08-10 |
| nginx `ngx_http_proxy_module` reference | https://nginx.org/en/docs/http/ngx_http_proxy_module.html | Implementation reference documentation | Accessed 2026-08-10 |
| Zalando RESTful API Guidelines, rule 227 | https://raw.githubusercontent.com/zalando/restful-api-guidelines/main/chapters/performance.adoc | Vendor guideline (comparative only) | Accessed 2026-08-10 |
| Microsoft Azure REST API Guidelines | https://raw.githubusercontent.com/microsoft/api-guidelines/vNext/azure/Guidelines.md | Vendor guideline (comparative only) | Accessed 2026-08-10 |
| Google Cloud APIs, *HTTP guidelines* | https://docs.cloud.google.com/apis/docs/http | Vendor guideline (comparative only) | Accessed 2026-08-10 |
| Kubernetes `WithCacheControl` filter | https://raw.githubusercontent.com/kubernetes/kubernetes/master/staging/src/k8s.io/apiserver/pkg/endpoints/filters/cachecontrol.go | Reference implementation source | `master`; accessed 2026-08-10 |
| ASP.NET Core `ServerSentEventsResult` | https://raw.githubusercontent.com/dotnet/aspnetcore/main/src/Http/Http.Results/src/ServerSentEventsResult.cs | Reference implementation source | `main`; accessed 2026-08-10 |
| Elasticsearch `ServerSentEventsRestActionListener` | https://raw.githubusercontent.com/elastic/elasticsearch/main/x-pack/plugin/inference/src/main/java/org/elasticsearch/xpack/inference/rest/ServerSentEventsRestActionListener.java | Reference implementation source | `main`; accessed 2026-08-10 |
| moby `getEvents` route handler | https://raw.githubusercontent.com/moby/moby/master/daemon/server/router/system/system_routes.go | Reference implementation source | `master`; accessed 2026-08-10 |
| MDN, *Using server-sent events* | https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events | Documentation (secondary, widely followed) | Accessed 2026-08-10 |
| Cloudflare Agents, *HTTP and Server-Sent Events* | https://developers.cloudflare.com/agents/api-reference/http-sse/ | Vendor documentation (comparative) | Accessed 2026-08-10 |
| Simon Willison, *How streaming LLM APIs work* (raw header captures) | https://raw.githubusercontent.com/simonw/til/main/llms/streaming-llm-apis.md | Secondary source carrying primary capture (verbatim `curl` header dumps) | Post created 2024-09-21; accessed 2026-08-10 |
| `openai/openai-python` issue 1673 | https://github.com/openai/openai-python/issues/1673 | Issue tracker (secondary) | Opened 2024-08-25; open at access 2026-08-10 |
| Elasticsearch stream-inference API reference | https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-inference-stream-completion | Vendor documentation | Accessed 2026-08-10 |
| AWS Bedrock `InvokeModelWithResponseStream` | https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_InvokeModelWithResponseStream.html | Vendor documentation | Accessed 2026-08-10 |

---

## 3. Field evidence

### 3.1 Method matters before values do

Read the table with RFC 9111 §4 in hand. A cache may never answer a `POST` from
storage, so on a `POST` stream the `Cache-Control` value governs only whether
bytes rest in an intermediary; on a `GET` stream it governs reuse as well.

### 3.2 Mandatory deep-dive — OpenAI, Anthropic, Google Gemini

**I could not probe any of the three streaming endpoints directly: all require
credentials, and this environment has none.** The header evidence below is a
verbatim `curl` capture published by a third party on **2024-09-21** — a single
source, roughly 23 months stale at access. Presence claims from it are stronger
than absence claims; treat "no `Cache-Control`" as "none appeared in that
capture," not as a standing guarantee. **[COMPARATIVE]**

| Provider | Streaming endpoint and method | `Cache-Control` in the 2024-09-21 capture | Other cache-relevant headers |
| --- | --- | --- | --- |
| OpenAI | `POST /v1/chat/completions` with `stream: true`, `content-type: text/event-stream; charset=utf-8` | **absent** | `access-control-expose-headers: X-Request-ID`; no `vary` |
| Anthropic | `POST /v1/messages` with `stream: true`, `content-type: text/event-stream; charset=utf-8` | **`cache-control: no-cache`** | none further |
| Google Gemini | `POST …:streamGenerateContent?alt=sse`, `content-type: text/event-stream` | **absent** | `content-disposition: attachment`; `vary: Origin`, `vary: X-Origin`, `vary: Referer` |

Corroboration attempted and what it produced:

- **Live unauthenticated probes (2026-08-10)** reached *error* responses at each
  provider's edge, not a stream. They are recorded here as baseline colour and
  are **not** evidence about the streaming `200`: OpenAI `GET /v1/models` → `401`
  with **no** `cache-control`; Gemini `GET /v1beta/models` → `403` with **no**
  `cache-control`; Anthropic `GET /v1/messages` → `405` with
  `cache-control: private, max-age=0, no-store, no-cache, must-revalidate,
  post-check=0, pre-check=0` — a Cloudflare-shaped edge default, not an
  application header. **[FACT]** on the probes, **[INFERENCE]** on the
  attribution.
- **`openai/openai-python` issue 1673** (opened 2024-08-25, still open at
  access): a user reports that an upstream caching proxy stored `create_and_poll`
  responses and the run never reached `completed`, and writes "I would expect the
  `Cache-Control` header to be set to `no-cache`, so that an upstream caching
  proxy does not store the agent response while polling the same API URL."
  **[FACT]** This is a `GET` polling endpoint, not a stream, but it is direct
  evidence that OpenAI's omission of `Cache-Control` has produced a real
  interoperability failure on the `GET` side of its own API. **[INFERENCE]**
- No provider **documents** a `Cache-Control` value for its streaming endpoints.
  Searched OpenAI, Anthropic, and Gemini streaming documentation 2026-08-10;
  none states one. **[FACT]**

**Reading.** The AI field is not united: one of three emits `no-cache`, two emit
nothing. Because all three are `POST`, none of the three is exposed to
cross-request reuse, which is the most plausible reason two of them have shipped
for years with no header and no reported incident. **[INFERENCE]**

### 3.3 The eight standard references

| API | Streaming or long-lived HTTP response? | `Cache-Control` posture | Basis and date |
| --- | --- | --- | --- |
| **Stripe** | **Non-participant.** No SSE or chunked-stream endpoint in the v1 REST API. | Whole-API `cache-control: no-cache, no-store` (observed on `GET /v1/charges` → `401`, live probe) | Live probe 2026-08-10; non-participation verified 2026-08-10 |
| **GitHub** | **Non-participant.** REST API has no streaming endpoint; the closest long-lived surface is webhook delivery, not a stream. | Comparative baseline only: `GET https://api.github.com/` → `200` with `cache-control: public, max-age=60, s-maxage=60` plus a weak `ETag` | Live probe 2026-08-10; non-participation verified 2026-08-10 |
| **Google / AIP** | **Partial.** Google's HTTP guidelines define server-streaming and restrict cloud APIs to half-duplex, but say nothing about caching a stream. | **No guidance.** The only caching sentence is a disclaimer: "HTTP protocol semantics that need to be handled by the server-side implementations of APIs are controlled by the server stack. Only rely on such semantics if these features are explicitly documented as part of the API spec, such as caching support." | https://docs.cloud.google.com/apis/docs/http accessed 2026-08-10 |
| **Microsoft / Azure** | **Non-participant.** The Azure REST API Guidelines contain no occurrence of "stream", "Server-Sent", or `text/event-stream`; long-running work is routed to the LRO/status-monitor pattern instead. | Caching guidance is conditional-request-shaped only: honour `If-Match` / `If-None-Match`, return `ETag`, and "For more control over caching, please refer to the `cache-control` HTTP header." No default value is stated. | Guidelines.md (vNext) accessed 2026-08-10 |
| **Twilio** | **Non-participant, and out of scope.** Twilio Media Streams is WebSocket-based, which §1.2 places outside this standard; Event Streams is a webhook/sink product, not an HTTP response stream. | n/a | Verified 2026-08-10 |
| **Shopify** | **Non-participant.** Admin API is REST plus GraphQL with webhooks and bulk-operation *files*; no SSE or chunked-stream response endpoint. | n/a. (A probe of `shopify.dev` returned the docs CDN, not the API, and is discarded as evidence.) | Verified 2026-08-10 |
| **Zalando** | **Non-participant on streaming**, but the only reference that states a caching *default*. | Rule 227, "**MUST** document cachable `GET`, `HEAD`, and `POST` endpoints": "servers and clients should always set the Cache-Control header to no-store and assume the same setting, if no Cache-Control header is provided," with the recommended value `Cache-Control: no-cache, no-store, must-revalidate, max-age=0`, and the warning that "Caching is in best case complex … in worst case inefficient" so "client side as well as transparent web caching should be avoided, unless the service supports and requires it." | performance.adoc accessed 2026-08-10 |
| **AWS** | **Participant.** Bedrock `InvokeModelWithResponseStream` and `ConverseStream` return `application/vnd.amazon.eventstream` — a binary framed stream, not SSE. | **Unverified.** The API reference documents no `Cache-Control`; the endpoint is SigV4-signed and cannot be probed without credentials. | API reference accessed 2026-08-10 |

### 3.4 Comparable precedent — implementations that actually stream

| Implementation | What streams | `Cache-Control` emitted | Evidence |
| --- | --- | --- | --- |
| **Kubernetes** kube-apiserver | `GET …?watch=1` — an unbounded `GET` stream | **`no-cache, private`**, applied by a global filter to every API response including watch, with the comment "because all servers are protected by authn/authz" and a link to Google's HTTP-caching guidance; set only "if it is not already set" | `endpoints/filters/cachecontrol.go`, `master`, accessed 2026-08-10 |
| **ASP.NET Core** (.NET 10) `TypedResults.ServerSentEvents` | first-class SSE result | **`no-cache,no-store`**, plus `Pragma: no-cache`, `Content-Encoding: identity`, and `DisableBuffering()`; no explanatory comment in source | `ServerSentEventsResult.cs`, `main`, accessed 2026-08-10 |
| **Elasticsearch** `POST _inference/completion/{id}/_stream` | SSE completion stream | **none.** `ServerSentEventsRestActionListener` sets `Content-Type: text/event-stream` and nothing else; a repository-wide code search for `Cache-Control` returns only `RestStatus.java` and two JWT client-side files | Source plus `gh api search/code` over `elastic/elasticsearch`, accessed 2026-08-10 |
| **Docker Engine** `GET /events` | unbounded chunked JSON/NDJSON stream | **none.** `getEvents` sets a negotiated `Content-Type` and `200`; no `Cache-Control` anywhere in the file | `system_routes.go`, `master`, accessed 2026-08-10 |
| **MDN** SSE reference example | canonical authoring example | **`no-cache`**, alongside `X-Accel-Buffering: no` | Accessed 2026-08-10 |
| **Cloudflare Agents** SSE reference | vendor SSE reference snippet | **`no-cache`**, with `Connection: keep-alive` | Accessed 2026-08-10 |

**Tally across the nine streaming implementations named in §3.2 and §3.4** —
three AI providers (OpenAI, Anthropic, Gemini), four shipped implementations
(Kubernetes, ASP.NET Core, Elasticsearch, Docker), and two reference snippets
(MDN, Cloudflare Agents):

| Value emitted | Count | Which |
| --- | --- | --- |
| `no-cache` (alone or with `private`) | 4 | Anthropic, Kubernetes, MDN, Cloudflare Agents |
| `no-cache` **and** `no-store` together | 1 | ASP.NET Core |
| nothing at all | 4 | OpenAI, Gemini, Elasticsearch, Docker |
| `no-store` **alone** | **0** | — |

So **every one of the five header-emitting implementations includes `no-cache`,
and not one sends `no-store` without it.** **[COMPARATIVE]**

### 3.5 The client side: browsers already opt out

WHATWG HTML's `EventSource` constructor, step 11, sets **request's cache mode to
`"no-store"`** (steps 10–12: set `Accept: text/event-stream`, set cache mode,
set initiator type). **[FACT]** WHATWG Fetch defines that mode as "Fetch behaves
as if there is no HTTP cache at all." **[FACT]** So for every browser-consumed
SSE stream the *user-agent* cache is already out of the picture regardless of
what the server sends. What remains exposed are shared and intermediary caches
and non-browser clients. **[INFERENCE]**

The same section supplies the resumption channel: on reconnect the UA sets
`Last-Event-ID` **in the request header list**, not the URI. **[FACT]** A header
is invisible to a cache key unless `Vary` names it (RFC 9111 §4, third reuse
condition: "request header fields nominated by the stored response (if any)
match those presented"). **[FACT]** That is the concrete mechanism behind the
tier-3 clause in the proposed rule. **[INFERENCE]**

### 3.6 Buffering is not caching — and the field conflates them

This is the single most misleading thing in the practitioner literature, and it
explains the `no-cache` habit better than any caching argument does.

nginx primary documentation, verbatim:

- `proxy_cache` — **default `proxy_cache off;`**. Caching is off unless a zone is
  configured. **[FACT]**
- `proxy_buffering` — **default `proxy_buffering on;`**. "When buffering is
  disabled, the response is passed to a client synchronously, immediately as it
  is received." And: "Buffering can also be enabled or disabled by passing
  `yes` or `no` in the `X-Accel-Buffering` response header field." **[FACT]**
- `proxy_ignore_headers` — when not disabled, `Expires`, `Cache-Control`,
  `Set-Cookie`, `Vary`, and `X-Accel-Expires` "set the parameters of response
  caching," while `X-Accel-Buffering` "enables or disables buffering of a
  response." **[FACT]** Two disjoint mechanisms, two disjoint controls.

**Consequence.** `Cache-Control: no-cache` does **nothing** about the failure
practitioners actually hit — a proxy holding frames until a buffer fills, so a
real-time stream arrives in batches. The lever for that is `proxy_buffering off`
or `X-Accel-Buffering: no`. Community reports of SSE breaking behind Cloudflare
and cloudflared describe buffering, not cache reuse. **[INFERENCE]** Much of the
field's `Cache-Control: no-cache` on streams is therefore cargo cult carried
along in copy-pasted header triples (`Content-Type` / `Cache-Control` /
`Connection: keep-alive`), which is exactly the triple MDN, Cloudflare Agents,
and the nginx-cookbook genre all reproduce. **[INFERENCE]** It is a *harmless*
cargo cult — the value happens to be right for the `GET` case — but it means the
prevalence of `no-cache` is weaker evidence for `no-cache` than the raw count
suggests, and it should not be read as the field having reasoned about
RFC 9111 §3.3.

---

## 4. Evidence, stated separately in each direction

### 4.1 FOR mandating `no-store` on streams

1. **RFC 9111 §3.3 explicitly permits storing a response that is still
   arriving** — the one thing the §13.4 row assumed impossible: "If the request
   method is GET, the response status code is 200 (OK), and the entire response
   header section has been received, a cache MAY store a response that is not
   complete … provided that the stored response is recorded as being
   incomplete." **[FACT]** `no-store` is the only directive that forecloses this
   at the storage layer; `no-cache` permits storage and only blocks reuse.
2. **`no-cache` leaves bytes at rest in an intermediary.** RFC 9111 §5.2.2.5 is
   the only directive that says a cache "MUST NOT intentionally store the
   information in non-volatile storage and MUST make a best-effort attempt to
   remove the information from volatile storage as promptly as possible after
   forwarding it." **[FACT]** For a stream carrying sensitive output this is a
   material difference.
3. **A stream cannot supply tier 1's revalidation half.** R7.3 tier 1 justifies
   `no-cache` by "cheap 304s, zero staleness." A live-generated stream has no
   stored representation to revalidate against, so the justification evaporates
   and tier 1 degrades to "block reuse" — which `no-store` also does, plus more.
   **[INFERENCE]**
4. **One authoritative guideline already defaults the whole API to `no-store`.**
   Zalando rule 227: "servers and clients should always set the Cache-Control
   header to no-store and assume the same setting, if no Cache-Control header is
   provided." **[COMPARATIVE]** Its stated reason — caching is "in best case
   complex … in worst case inefficient" — applies with extra force to a response
   whose length, completion, and validity are all unknown when the header is
   written.
5. **The standard's own worked example already does it.** E.11 sends
   `private, no-store` on a non-sensitive export stream (rest-api-standard.md:2897).
   Ratifying the example is the lowest-churn resolution. **[POLICY]**
6. **`no-store` is the only value that is safe without further analysis.** Under
   tier 1 or tier 3 an implementer must additionally reason about `Vary`, about
   `Last-Event-ID`, and about whether the artifact is immutable. `no-store`
   requires none of that. **[INFERENCE]**

### 4.2 AGAINST mandating `no-store` on streams

1. **Zero of nine surveyed streaming implementations send `no-store` alone.**
   Four send `no-cache`, one sends `no-cache,no-store`, four send nothing
   (§3.4 tally). **[COMPARATIVE]** A `MUST no-store` would put this standard
   outside the field it documents, on a question where no implementer reports
   harm.
2. **`no-cache` is already sufficient for correctness.** RFC 9111 §4: a cache
   "MUST NOT reuse a stored response unless … the stored response does not
   contain the no-cache directive …, unless it is successfully validated."
   **[FACT]** Nothing stale, wrong, or cross-user can be served under `no-cache`.
   The residual difference is confidentiality-at-rest, which is a property of the
   *payload*, not of the *response shape* — and R7.2 plus R7.3 tier 2 already
   govern payload sensitivity for every response including streams.
   **[INFERENCE]**
3. **The reuse hazard does not exist at all on a `POST` stream.** RFC 9111 §4:
   "A cache MUST write through requests with methods that are unsafe … a cache is
   not allowed to generate a reply to such a request before having forwarded the
   request and having received a corresponding response." **[FACT]** And §3.3's
   incomplete-storage permission is `GET`-only. **[FACT]** Every AI-provider
   stream surveyed is a `POST`. A blanket `no-store` would impose a rule whose
   motivating hazard is absent from the largest streaming population in the
   field. **[INFERENCE]**
4. **Shared caches are already blocked for the credentialed case.** RFC 9111 §3:
   a cache MUST NOT store "if the cache is shared: the Authorization header field
   is not present in the request … or a response directive is present that
   explicitly allows shared caching." **[FACT]** Bearer-token APIs — which is
   what R8.2 makes this standard's APIs — get shared-cache protection from the
   protocol before any header is chosen. The genuine residue is (a) private
   caches, which heuristic cacheability does reach, and (b) query-parameter
   credentials such as Gemini's `?key=`, where no `Authorization` header exists
   to trigger the bullet — and R8.2 already forbids that form outright.
   **[INFERENCE]**
5. **The existing three tiers already cover the case, once you notice R7.3 has
   no keyword.** E.11's `no-store` was never a violation; a stream choosing
   `private, no-cache` would not be one either. Elevating either to a MUST is a
   new obligation, not a repair. **[INFERENCE]**
6. **R7.3 names blanket `no-store` an anti-pattern for a reason that survives
   here.** A stream over a retained, immutable artifact — a completed export
   replayed as NDJSON, a published changelog tail — is genuinely cacheable, and
   is precisely the population R13.10 already carves out as "a view over a
   retained artifact." Forbidding tier 3 on it discards a real performance lever
   on the one stream class where it exists. **[INFERENCE]**
7. **A strong `ETag` on a stream is not protocol-impossible, so the
   "machinery a stream cannot supply" premise is too strong.** RFC 9110 §8.8.1:
   "There are a variety of strong validators used in practice. The best are based
   on strict revision control, wherein each change to a representation always
   results in a unique node name and revision identifier being assigned before
   the representation is made accessible to GET." **[FACT]** Only the *hash*
   form carries the pre-transmission constraint — "sufficient if the data is
   available prior to the response header fields being sent." **[FACT]** So a
   version-based validator computed before transmission is legitimate, and a
   stream over a retained artifact could carry one. What blocks tier 1's
   revalidation on a stream is therefore not "no ETag is computable" but that
   R3.10 binds resources supporting *conditional update*, and a stream is not
   one. **[INFERENCE]** The rule should say that, not the false thing.

### 4.3 The one surfaced conflict, adjudicated rather than averaged

**Zalando rule 227 vs R7.3 tier 2.** Zalando defaults the whole API to
`no-store` and recommends `no-cache, no-store, must-revalidate, max-age=0`;
R7.3 names blanket `no-store` an anti-pattern. These are irreconcilable at the
level of the default, and both are vendor-class guidance, not protocol.

**Which should govern here: R7.3.** Three reasons. First, `CONSTRAINTS.md`
admits vendor guidelines "only as comparative evidence of established practice,
never as protocol authority," and Zalando's `no-store` default is *not*
established practice in the streaming population — zero of nine. Second,
Zalando's rule is explicitly a whole-API default written for an organisation that
elects to avoid caching entirely; it is not evidence about streams. Third, R7.3
is already ratified in this standard, and a leaf answering a narrow §13.4 row is
not the place to overturn a ratified posture on evidence that predates it and
does not address streaming. **[POLICY]**

---

## 5. Proposed rule text

**Classification: `[POLICY]` on the default choice, resting on protocol facts
for every clause's mechanism. Strength: MUST on the header, MUST NOT on the two
prohibitions, MAY on tier 3. Confidence: moderate-high.**

> **R13.12** A streaming response MUST carry an explicit `Cache-Control`.
> R7.1 is not relaxed by incremental delivery: the response header section is
> transmitted before the body exists, and the directive is a property of the
> resource's policy rather than of its bytes, so the header is always computable.
>
> Within R7.3's posture, a streaming response takes **tier 1 —
> `Cache-Control: private, no-cache`** — by default, and **tier 1's strong-`ETag`
> revalidation clause does not apply to it.** R3.10 binds a resource that
> supports conditional update; a stream is not one, so there is no stored
> representation for a `304` to refer to. On a stream, `no-cache` is operative
> for its other half — RFC 9111 §4 forbids a cache to reuse the response to
> satisfy any other request without revalidating, and with no validator offered
> the cache must forward.
>
> **Tier 2 (`no-store`)** applies to a stream on exactly the same condition as to
> any other response — a genuinely sensitive payload, judged from what the frames
> carry — and MUST NOT be adopted merely because the response is a stream.
>
> **Tier 3 (`public` with `max-age`)** MAY be used only where the stream is a
> view over an immutable retained artifact (R13.10's test), and then only where
> every input that selects a resumption point is either part of the request URI
> or named in `Vary` (R4.11). A stream that is unbounded by design (R13.6) MUST
> NOT use tier 3. The clause exists because SSE resumption travels in the
> `Last-Event-ID` **request header**, which is invisible to a cache key unless
> `Vary` names it — a cache would otherwise answer a resume-from-position-N
> request with a stored stream that begins at position 1.
>
> > Provenance: research leaf `baseline-04d`, answering the §13.4 register row
> > "Caching posture for a stream" · the header obligation restates ratified
> > R7.1; the tier-1 inapplicability finding and the `Vary` condition are
> > protocol-grounded (RFC 9111 §3, §3.3, §4; RFC 9110 §8.8.1; WHATWG HTML
> > `EventSource`) · the choice of tier 1 over tier 2 as the stream default is
> > `[POLICY]`, backed by all 5 header-emitting streaming implementations
> > surveyed — every one includes `no-cache`, none sends `no-store` alone — and
> > against Zalando rule 227 · confidence moderate-high.

### 5.1 Clause-by-clause strength rationale

| Clause | Strength | Why not stronger | Why not weaker |
| --- | --- | --- | --- |
| explicit `Cache-Control` on a stream | **MUST** | — | It is R7.1 restated for a case §13.4 recorded as doubtful; 4 of 11 surveyed implementations omit it, and RFC 9111 §3 then makes a `GET` stream storable on heuristic grounds alone |
| default tier 1 (`private, no-cache`) | **default, no keyword** — matching R7.3's own register | A MUST would forbid the legitimate tier-2 and tier-3 cases below | Silence is what produced this register row |
| tier-1 `ETag` clause inapplicable | **statement of fact, not an obligation** | Nothing to require | Without it the standard reads as demanding machinery R3.10 does not actually bind |
| `no-store` not adopted merely for shape | **MUST NOT** | — | This is the whole content of the R7.3 anti-pattern applied to the case that most tempts it |
| tier 3 gated on immutability plus `Vary` | **MAY, conditioned** | No surveyed implementation exercises it, so a SHOULD would over-promise | A flat prohibition discards R13.10's retained-artifact case, where caching is genuinely correct |
| tier 3 forbidden on an unbounded stream | **MUST NOT** | — | An unbounded stream has no complete representation and no meaningful freshness lifetime |

### 5.2 Consequential edits the ratification walk must decide

These are **options for the walk**, not part of the rule. Recorded so the fix is
not half-applied.

1. **§13.4 register row.** On ratification the row "Caching posture for a stream
   (§7)" is discharged and should be removed from the unresolved table, with the
   verdict recorded in `decisions/`. `PLAN.md` line 348 already licenses this:
   "Some items may prove to need no rule; that verdict is itself recorded."
   The row's own wording should not be carried forward — it states that a stream
   cannot supply a strong `ETag`, which §4.2 item 7 shows is too strong.
2. **Appendix E.11 `Cache-Control: private, no-store`** (rest-api-standard.md:2897).
   Two honest readings exist, and the walk should pick one rather than leave the
   example ambiguous:
   - *It is a deliberate tier-2 call* — order data is sensitive. Then E.11's
     prose should say so, because as written the reader cannot tell tier 2 from
     a reflex.
   - *It is belt-and-braces* — then it is at minimum imprecise, because
     RFC 9111 §5.2.2.5 states that `no-store` "applies to both private and
     shared caches," making the `private` token redundant, and under R13.12 the
     example would better exercise the default as `private, no-cache`.
   **Recommendation: change E.11 to `private, no-cache`** and let a separate
   sentence note that a sensitive export would take `no-store` instead. That way
   the worked example demonstrates the default rather than the exception.
   **[POLICY]**
3. **Conformance checklist row for R7.3** (rest-api-standard.md:2368) reads
   "Three-tier posture applied; no blanket `no-store`" — an obligation-shaped
   gloss on a rule that carries no BCP 14 keyword. The walk should decide
   whether R7.3 gains keywords or the checklist row is softened. Out of scope
   for this leaf, but it is the same defect this leaf uncovered.
4. **§1.8 switch scope.** R13.12 should be scoped by the `streaming`
   applicability switch, like R13.1/R13.2 and unlike R13.3 — an API that does
   not stream owes nothing here, because R7.1 already binds all its responses.

---

## 6. Declined alternatives

| Alternative | Why declined |
| --- | --- |
| **A. `MUST` send `Cache-Control: no-store` on every streaming response** (ratifying E.11 as a rule) | Zero of nine surveyed streaming implementations do this. It buys no correctness over `no-cache` (RFC 9111 §4 already blocks reuse), and its only added property — no bytes at rest — is a payload judgement that R7.2 and R7.3 tier 2 already make for every response. It would also make this standard contradict its own named anti-pattern in the one place implementers most expect the anti-pattern to bite. |
| **B. No rule at all — delete the §13.4 row and rely on R7.1 plus R7.3** | Half right, and it is the closest runner-up. R7.1 genuinely covers the MUST, and R7.3's lack of keywords means there was never a conflict to fix. But three things remain uncovered by silence: readers currently believe (per §13.4's own wording) that R7.1 is unsatisfiable on a stream; nothing tells them tier 1's `ETag` clause is inapplicable rather than merely unavailable; and the `Last-Event-ID` cache-key hazard is real, undocumented anywhere in the standard, and invisible to anyone who has not read WHATWG HTML step 11. A one-paragraph rule closes all three at low cost. |
| **C. Exempt streaming responses from R7.1** | Refuted by mechanism: the header section precedes the body, so compliance costs nothing. Refuted by consequence: the four implementations that emit nothing are exactly the population RFC 9111 §3's heuristic-cacheability bullet reaches, and `openai/openai-python` issue 1673 is a shipped instance of the resulting failure on the same vendor's `GET` surface. |
| **D. Require a strong `ETag` on a stream over a retained artifact, restoring tier 1 in full** | Protocol-legitimate — RFC 9110 §8.8.1's revision-identifier form needs no body — but it has zero field precedent, and R13.10's `stream_position` already supplies resumption with better-fitting semantics than conditional requests. Requiring a validator nobody would use is cost without benefit. |
| **E. Adopt Zalando rule 227's whole-API `no-store` default** | Out of scope for a §13 leaf, contradicts a ratified R7.3, and rests on guidance that predates and does not address streaming. Recorded in §4.3 as a surfaced conflict with a stated adjudication rather than averaged away. |
| **F. Add an `X-Accel-Buffering` or anti-buffering obligation alongside the caching rule** | Tempting because §3.6 shows buffering is the failure practitioners actually hit, but it is a deployment concern about a specific proxy vendor's non-standard header, which `CONSTRAINTS.md` puts outside scope ("Do not prescribe server frameworks, client libraries, deployment platforms"). It belongs in the informative `streaming-profile.md`, not in a rule. |

---

## 7. What I could not verify

| Claim | Status | What would settle it |
| --- | --- | --- |
| OpenAI's current streaming `Cache-Control` | **Unverified.** The only capture found is dated 2024-09-21 and shows the header absent. No credentials available to probe `POST /v1/chat/completions` with `stream: true`. Absence in a single 23-month-old capture is weak evidence. | One authenticated `curl -v` against the live endpoint |
| Google Gemini's current streaming `Cache-Control` | **Unverified**, same basis and same capture. `content-disposition: attachment` in that capture is a content-sniffing guard, not a caching directive. | One authenticated `curl -v` against `:streamGenerateContent?alt=sse` |
| Anthropic's `cache-control: no-cache` on the streaming `200` | **Single-source and dated.** Corroborated only by secondary write-ups that appear to derive from the same capture. The live `405` probe shows a Cloudflare-shaped header and says nothing about the streaming `200`. | One authenticated `curl -v` against `POST /v1/messages` with `stream: true` |
| Whether any of the three providers *documents* a caching posture | **Verified negative** for the pages searched on 2026-08-10, but a documentation search cannot prove absence across every page of three large sites. | — |
| AWS Bedrock `InvokeModelWithResponseStream` response headers | **Unverified.** Not documented in the API reference; the endpoint is SigV4-signed and unprobeable without credentials. | One signed request with header capture |
| Azure OpenAI's SSE `Cache-Control` | **Not attempted.** Requires a provisioned resource; treated as covered by the Microsoft non-participation row, which concerns the *guidelines*, not the service. | A provisioned Azure OpenAI deployment |
| Whether real-world shared caches actually exercise RFC 9111 §3.3 incomplete storage | **Unverified.** The permission is in the RFC; I found no measurement of which CDN or proxy implementations use it. The proposed rule does not depend on this — R7.1 closes the case either way — but it would move confidence on §4.1 item 1. | A survey of nginx, Varnish, Squid, and CDN cache-storage behaviour on a truncated `200` |
| Whether Elasticsearch or Docker have ever had a reported cache-related stream incident | **Not searched exhaustively.** Their header silence is verified from source; the operational consequence is not. | Issue-tracker search on both projects |
| Twilio and Shopify non-participation | **Verified from product shape** (WebSocket media streams; GraphQL plus webhooks plus bulk files) rather than from an exhaustive endpoint inventory. | A full endpoint inventory of both APIs |

---

## 8. Unresolved questions this leaf raises but does not answer

1. **R7.3 carries no BCP 14 keyword while the conformance checklist treats it as
   an obligation.** This leaf needed the fact and found it; repairing it is a §7
   question, not a §13 one.
2. **`Vary` on streams generally.** R13.2 requires `Vary: Accept` where one
   endpoint negotiates both shapes, and R4.11 governs `Vary` broadly, but no rule
   addresses request headers that select *content within* a stream. The proposed
   R13.12 handles the tier-3 case; the general case is untouched.
3. **Interaction with the idempotency-key row of §13.4.** If R3.9's "stored
   response" is ever defined for a stream, that store is a cache in all but name
   and will need its own posture. The two register rows should probably be ruled
   in the same walk.
