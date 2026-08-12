# baseline-04f — Resource ceilings for streams

**Series:** `baseline` (prescriptive — proposes rules, ratifies nothing)
**Parent:** `baseline-04-streaming` · **Sibling survey:** `survey-08-streaming`
**Question:** What resource ceilings should a streaming API publish and enforce?
**Run date / access date for every source below:** 2026-08-10
**Status:** proposal. Nothing here binds the standard until a record lands in
`research/decisions/`.

---

## 1. TL;DR and recommendation

**The claimed gap is half real.** The §13.4 register entry says a held-open
stream "is in none of those dimensions; no maximum duration or per-principal
concurrency ceiling is required." Checked against the ratified text:

| Claimed missing | Verdict | Why |
| --- | --- | --- |
| Maximum stream **duration** | **Real gap** | No rule in §6, §8, §11, or §13 bounds how long a server may hold a response open. `R13.11` bounds a long *poll*, which `§13.5` defines as not a stream. |
| Per-principal **concurrency** ceiling | **Overstated — substantially covered** | `R8.10`'s rate-limit axis default already requires a *published* multi-dimensional posture whose last item is "concurrency separate." A stream is one in-flight request, so a conforming concurrency limiter already counts it. |
| Streamed **collection** item count | **Already covered** | `R6.5` binds "each collection" — including a streamed one, as `R6.4`'s 1.1.0 widening confirms — to document a default and maximum `limit`, enforced by `R11.1`. |

**Recommendation — two proposed rules, both narrow.**

1. **`ST-012` (`MUST`, documentation-first).** Each streaming endpoint documents
   either a maximum hold duration or that the stream is unbounded by design.
   The unbounded declaration is the one `R13.6` already requires, so this rule
   adds no new artifact for watches and event tails. Where a maximum *is*
   declared, the server enforces it **and** ends the stream with `R13.6`'s
   terminal frame rather than by dropping the connection.
2. **`ST-013` (`MUST`, clarification not new ceiling).** A held-open stream
   counts as one unit against `R8.10`'s concurrency dimension for its whole
   lifetime, and the API documents that counting rule. This closes the residual
   concurrency gap by making an existing axis explicit rather than by adding a
   second ceiling.

**Why duration and not a separate concurrency ceiling.** AWS states the
identity outright: with a bounded connection lifetime and a bounded
new-connection rate, peak concurrency is *derived*, not separately configured —
"The maximum number of concurrent connections is determined by the rate of new
connections per second and maximum connection duration of two hours"
([AWS, WebSocket quotas](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-execution-service-websocket-limits-table.html)).
`R11.2` already mandates a rate limit. Bounding duration therefore converts an
already-required rule into a concurrency bound for free; adding a second,
independently tuned ceiling double-counts. `[INFERENCE]`

**Confidence:** high on the gap analysis (read directly from the ratified text);
moderate-high on `ST-012`; moderate on `ST-013`'s `MUST` strength.

---

## 2. Standards-and-currency matrix

Authority classes follow `CONSTRAINTS.md`: **S** = published standard or living
standard; **D** = Internet-Draft (none load-bearing here); **V** = official
vendor documentation (comparative evidence only, never protocol authority);
**C** = primary implementation source code; **X** = security-standards body.

| # | Source | URL | Class | Publication / access |
| --- | --- | --- | --- | --- |
| 1 | WHATWG HTML Living Standard §9.2 (Server-sent events), authoring notes §9.2.7 | `https://html.spec.whatwg.org/multipage/server-sent-events.html` | S | living; accessed 2026-08-10 |
| 2 | OWASP API Security Top 10 2023 — API4:2023 Unrestricted Resource Consumption | `https://owasp.org/API-Security/editions/2023/en/0xa4-unrestricted-resource-consumption/` | X | 2023 edition; accessed 2026-08-10 |
| 3 | MDN — Using server-sent events | `https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events` | V (browser-platform reference) | accessed 2026-08-10 |
| 4 | Anthropic — Rate limits | `https://platform.claude.com/docs/en/api/rate-limits` | V | accessed 2026-08-10 (redirected from `docs.anthropic.com`) |
| 5 | Anthropic — Errors, "Long requests" | `https://platform.claude.com/docs/en/api/errors` | V | accessed 2026-08-10 |
| 6 | Anthropic — Streaming messages | `https://platform.claude.com/docs/en/api/streaming` | V | accessed 2026-08-10 |
| 7 | OpenAI — Rate limits | `https://developers.openai.com/api/docs/guides/rate-limits` | V | accessed 2026-08-10 |
| 8 | OpenAI — Realtime | `https://developers.openai.com/api/docs/guides/realtime` | V | accessed 2026-08-10 |
| 9 | OpenAI — Background mode | `https://developers.openai.com/api/docs/guides/background` | V | accessed 2026-08-10 |
| 10 | Google — Gemini API rate limits | `https://ai.google.dev/gemini-api/docs/rate-limits` | V | accessed 2026-08-10 |
| 11 | Google — Gemini Live API session management | `https://ai.google.dev/gemini-api/docs/live-session` | V | accessed 2026-08-10 |
| 12 | Microsoft — GPT Realtime API for speech and audio (Azure OpenAI) | `https://learn.microsoft.com/en-us/azure/ai-foundry/openai/how-to/realtime-audio` | V | doc date 2026-07-29, updated 2026-07-31; accessed 2026-08-10 |
| 13 | Microsoft — Azure OpenAI quotas and limits | `https://learn.microsoft.com/en-us/azure/ai-foundry/openai/quotas-limits` | V | doc date 2026-05-27, updated 2026-07-29; accessed 2026-08-10 |
| 14 | Stripe — Rate limits (incl. "Concurrency limits") | `https://docs.stripe.com/rate-limits` | V | accessed 2026-08-10 |
| 15 | GitHub — Rate limits for the REST API (secondary rate limits) | `https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api` | V | accessed 2026-08-10 |
| 16 | Google AIP-158 — Pagination | `https://google.aip.dev/158` | V (design guideline) | accessed 2026-08-10 |
| 17 | Twilio — REST API best practices | `https://www.twilio.com/docs/usage/rest-api-best-practices` | V | accessed 2026-08-10 |
| 18 | Shopify — API usage limits | `https://shopify.dev/docs/api/usage/limits` | V | accessed 2026-08-10 |
| 19 | Zalando — RESTful API Guidelines | `https://opensource.zalando.com/restful-api-guidelines/` | V | accessed 2026-08-10 |
| 20 | AWS — Quotas for configuring and running a WebSocket in API Gateway | `https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-execution-service-websocket-limits-table.html` | V | accessed 2026-08-10 |
| 21 | AWS — Amazon SQS short and long polling | `https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-short-and-long-polling.html` | V | accessed 2026-08-10 |
| 22 | Kubernetes source — `apiserver/pkg/endpoints/handlers/get.go` | `https://raw.githubusercontent.com/kubernetes/kubernetes/master/staging/src/k8s.io/apiserver/pkg/endpoints/handlers/get.go` | C | `master`; accessed 2026-08-10 |
| 23 | Kubernetes source — `apiserver/pkg/server/options/server_run_options.go` | `https://raw.githubusercontent.com/kubernetes/kubernetes/master/staging/src/k8s.io/apiserver/pkg/server/options/server_run_options.go` | C | `master`; accessed 2026-08-10 |
| 24 | Kubernetes source — `apiserver/pkg/server/config.go` | `https://raw.githubusercontent.com/kubernetes/kubernetes/master/staging/src/k8s.io/apiserver/pkg/server/config.go` | C | `master`; accessed 2026-08-10 |
| 25 | Kubernetes source — `apimachinery/pkg/apis/meta/v1/types.go` (`ListOptions.TimeoutSeconds`) | `https://raw.githubusercontent.com/kubernetes/kubernetes/master/staging/src/k8s.io/apimachinery/pkg/apis/meta/v1/types.go` | C | `master`; accessed 2026-08-10 |
| 26 | HashiCorp — Consul blocking queries | `https://developer.hashicorp.com/consul/api-docs/features/blocking` | V | accessed 2026-08-10 |

Two currency notes. First, `master` is a moving target: the Kubernetes numbers
below are pinned to a fetch on 2026-08-10 and should be re-verified against a
release tag before ratification. Second, source 12's session-duration figure is
Azure's restatement of OpenAI's Realtime behavior; OpenAI's own public guide
(source 8) carries no number, so the 60-minute figure rests on Microsoft's
documentation, not OpenAI's. That is a single-source claim and is labelled as
such in §5.

---

## 3. Which parts of the claimed gap are real

### 3.1 The register entry, quoted

`rest-api-standard.md` §13.4, "Known unresolved interactions":

> **Resource ceilings for streams** (§11) — R11.1 requires published maxima for
> page size, expansion depth, and bulk item count. A held-open stream is the
> largest unbounded commitment the API makes and is in none of those dimensions;
> no maximum duration or per-principal concurrency ceiling is required.

### 3.2 `R11.1` — does not reach a stream. **Gap confirmed.**

> **R11.1** An API MUST publish and enforce a maximum page size, expansion
> depth, and bulk item count.

All three dimensions are properties of the *request shape* that the client
supplies and the server validates before doing work. A stream's cost is a
property of the *response*, accrued after the status is committed. `R11.1` does
not reach it. `[FACT]` — read directly from the ratified text.

### 3.3 `R8.10` rate-limit axis — the concurrency claim is **overstated**.

The axis default, quoted verbatim from §8.3:

> Multi-dimensional tiered posture, published: per-principal sustained +
> token-bucket burst (start ≈100 rps/account, 25 rps/endpoint, `[POLICY]`
> numbers); unauthenticated per-IP an order of magnitude lower; auth endpoints
> strictly stricter (start ≤5/min per IP+account); failed-auth budget;
> concurrency separate

And `R8.10`'s own normative sentence:

> An API MUST adopt each axis default unless a named trigger applies, and MUST
> record any flip in its conformance note.

Three readings follow, and they matter:

1. The colon after "published" distributes across the whole semicolon-separated
   list. "concurrency separate" is therefore an item of the **published**
   posture, not an unpublished implementation note. So a published concurrency
   dimension **is** already required. `[INFERENCE]` — this rests on reading the
   colon as distributive; the sentence is compressed enough that a reviewer
   could read "published" as governing only the first clause. That ambiguity is
   itself a finding, addressed by `ST-013`.
2. `MUST adopt each axis default` makes adoption mandatory, so the duty is a
   `MUST`-strength published-and-adopted posture, not a recommendation.
3. **A stream is one HTTP request.** A concurrency limiter counts requests in
   flight. A stream held open for an hour is in flight for an hour. So a
   conforming implementation of "concurrency separate" *already* bounds
   concurrent streams per principal — and does so more faithfully than a rate
   limiter, which sees one arrival and nothing after.

Conclusion: the register entry's phrase "no … per-principal concurrency ceiling
is required" is **wrong as written**. What is genuinely missing is far narrower:
the standard nowhere *says* that a held-open stream occupies a concurrency slot
for its lifetime, and nowhere requires the counting rule to be documented. An
implementer could count a stream once at open and release the slot immediately,
which silently degrades the concurrency limiter into a second rate limiter.
Stripe is explicit that long-lived requests are exactly what trip concurrency
limiting (§5.4), which is why the counting rule is worth stating.

### 3.4 `R6.5` — a streamed collection **already owes** a published maximum.

> **R6.5** The pagination request parameters are `cursor` and `limit` (§1.10).
> Each collection MUST document its default and maximum `limit` (the enforcement
> obligation is R11.1).

"Each collection" is unqualified. That §6 binds streamed collections is not a
stretch — the ratified text says so twice:

- `R6.4`, widened in version 1.1.0 by the Phase 6 walk: "Pagination state lives
  only in the body representation — the envelope (R6.1), or **the terminal frame
  where the collection is streamed**."
- §6.1's discussion of the empty collection already describes "zero item frames
  followed by the terminal frame R13.6 requires."

So a **streamed collection** already carries a documented, enforced maximum item
count. `[FACT]`

But that covers exactly one of three stream shapes, and not the two that make
the register entry worth writing:

| Stream shape | Bounded today? | By what |
| --- | --- | --- |
| Streamed collection (an export, a paged list delivered as frames) | **Yes** | `R6.5` documents the max `limit`; `R11.1` enforces it |
| Generative stream (one representation delivered in pieces — the AI-provider case) | **No** | Not a collection. No item count exists to bound. Duration is bounded only by the model's own stopping. |
| Watch or event tail (unbounded by design) | **No, and correctly so** | `R13.6` explicitly blesses "a stream that is unbounded by design — a watch, an event tail" and requires only that the API document it |

The Phase 6 walk did not scope `R6.5`, which is why this was not noticed. Scoping
it now costs nothing: the rule already reads correctly for the streamed case.

### 3.5 Net finding

**One dimension is genuinely unbounded: duration.** Concurrency is covered in
substance and under-stated in text. Streamed-collection size is covered outright.
The register entry should be corrected on ratification, not merely resolved.

---

## 4. Field evidence — the mandatory AI-provider deep-dive

### 4.1 Comparison table

`—` means the vendor's documentation carries no such statement (a verified
negative, probed on 2026-08-10, not an absence of searching).

| Vendor | Rate-limit dimensions | Published max request/stream duration | Published concurrency ceiling | How a stream is counted |
| --- | --- | --- | --- | --- |
| **Anthropic** (Messages API) | RPM, ITPM, OTPM per organization per model; token bucket; Start/Build/Scale/Custom tiers | — for streaming. `504 timeout_error` exists ("The request timed out while processing") with no published number. A 10-minute figure is documented only as an **SDK-side guard** | — for the Messages API. Batch API publishes "Maximum batch requests in processing queue" (200,000 / 300,000 / 500,000 by tier) | Per request against RPM; per token against ITPM/OTPM as tokens are produced. "OTPM rate limits are evaluated in real time as output tokens are produced" — so a long stream is metered **continuously**, not once |
| **OpenAI** (Chat Completions / Responses) | RPM, RPD, TPM, TPD, IPM, plus "audio minutes per minute for some streaming audio models" | — | — (page states none) | Not documented |
| **OpenAI Realtime** | as above | 60 min, per Microsoft's documentation of the same API; OpenAI's own guide states no number. `session.created` carries an `expires_at` field | — (community reports none; no vendor statement found) | Not documented |
| **Google Gemini** (generateContent) | RPM, TPM, RPD; spend-based limits per 10-minute window; priority-inference multiplier 0.3× | — | "Concurrent batch requests: 100" — **Batch API only** | Not documented |
| **Google Gemini Live API** | as above | **15 minutes** audio-only, **2 minutes** audio+video, without compression. Connection lifetime separately "limited … to around 10 minutes", preceded by a `GoAway` message | — | Session resumption tokens "valid for 2 hr after the last sessions termination"; `contextWindowCompression` extends a session "to an unlimited amount of time" |

### 4.2 Anthropic — verified negative on both dimensions

`[FACT]` The rate-limits page enumerates spend limits, RPM, ITPM, OTPM, Batch
queue depth, Managed Agents per-operation RPM, and fast-mode limits. It contains
no concurrent-request, concurrent-connection, or concurrent-stream limit for the
Messages API. Source 4.

`[FACT]` The only duration statement is a client-library guard, quoted from the
errors page (source 5): "The SDKs validate that your non-streaming Messages API
requests are not expected to exceed a 10-minute timeout." The same page warns
"Consider using the streaming Messages API or Message Batches API for
long-running requests, especially those over 10 minutes," and attributes the
hazard to the network rather than to a server policy: "Some networks may drop
idle connections after a variable period of time."

`[INFERENCE]` This is the failure mode a published bound prevents. The server's
real limit is invisible, so the client SDK synthesizes one from an observed
timeout and enforces it locally. Two independent SDK guards (`anthropic` Python
and TypeScript, per source 6's "the SDKs require streaming to avoid HTTP
timeouts") are doing the work a one-line published maximum would do.

`[FACT]` Anthropic **does** meter a stream continuously: "OTPM rate limits are
evaluated in real time as output tokens are produced, counting only the actual
tokens generated. The `max_tokens` parameter does not factor into OTPM rate
limit calculations." So the cost dimension a generative stream actually consumes
is bounded — just not by time or by connection count.

### 4.3 Gemini Live — the field's clearest published duration ladder

`[FACT]` Three separate numbers, all from source 11:

- "audio-only sessions are limited to 15 minutes, and audio-video sessions are
  limited to 2 minutes" (without compression);
- "The lifetime of a connection is limited as well, to around 10 minutes," with a
  `GoAway` message sent beforehand;
- resumption tokens "valid for 2 hr after the last sessions termination."

`[COMPARATIVE]` This is the shape `ST-012` proposes, implemented: a documented
maximum, an in-band warning frame before the cut, and a resumption mechanism that
makes the cut safe. It also demonstrates the exemption — enabling
`contextWindowCompression` makes the session unbounded, and Google documents that
too. A rule that demanded a finite number would forbid Google's own configuration.

### 4.4 Azure/OpenAI Realtime — a duration published *in the protocol*

`[FACT]` Source 12, three separate statements: "The maximum session duration is
60 minutes"; "Realtime sessions have a maximum duration of **60 minutes**"; and
operationally, "Monitor the `session.created` event's `expires_at` field."

`[INFERENCE]` The `expires_at` field is the strongest single design precedent
found: the maximum is not merely published in prose but delivered **in-band on
the first frame**, so a client learns the deadline for *this* stream without
consulting documentation. This standard already has a natural carrier — the
terminal frame (`R13.6`) is the wrong end, but a documented opening frame type
under `R13.5` would be the right one. Not proposed as a rule here (see §10,
declined alternative 4), but recorded.

**Conflict surfaced.** Source 12 states "The Realtime API has specific rate
limits for audio tokens and **concurrent sessions**. Before deploying to
production, review Azure OpenAI quotas and limits" — and source 13, the page it
points at, publishes **no concurrent-session number**. It publishes RPM/TPM per
model (`gpt-realtime` GlobalStandard: 200 RPM / 100,000 TPM at Tier 1, rising to
300 / 150,000 at Tier 6) from which a concurrency bound can only be *derived*.
`[FACT]` on both readings; the conflict is Microsoft's, not this report's. It is
the exact documentation defect `ST-013` is meant to prevent.

---

## 5. Field evidence — the eight standard references

| Reference | Concurrency limit on long-lived connections | Duration limit | Verified on |
| --- | --- | --- | --- |
| **Stripe** | **Yes, published as a first-class dimension.** A dedicated "Concurrency limits" section; `Stripe-Rate-Limited-Reason` header values `global-concurrency` and `endpoint-concurrency`; one published number — Payouts API "30 concurrent requests per business" | — (object lock timeouts exist, described as a different mechanism) | 2026-08-10 |
| **GitHub** | **Yes, exact number.** "No more than 100 concurrent requests are allowed. This limit is shared across the REST API and GraphQL API." | **Yes, as a CPU budget.** "No more than 90 seconds of CPU time per 60 seconds of real time is allowed. No more than 60 seconds of this CPU time may be for the GraphQL API." | 2026-08-10 |
| **Google / AIP** | — | — | 2026-08-10 |
| **Microsoft / Azure** | Asserted for Realtime but not numerically published (see §4.4 conflict) | **Yes, 60 minutes** for Realtime sessions, plus in-band `expires_at` | 2026-08-10 |
| **Twilio** | **Enforced, exposed, not published.** `Twilio-Concurrent-Requests` response header gives "The current number of concurrent requests for your account"; exceeding the ceiling returns `429` / error `20429`. The ceiling itself is account-specific and not in the docs | — | 2026-08-10 |
| **Shopify** | — (leaky-bucket points per second; array input max 250; pagination max 25,000 objects; single query cost max 1,000 points) | — | 2026-08-10 |
| **Zalando** | — | — (rule 153, "MUST use code 429 with headers for rate limits", is the only adjacent guidance) | 2026-08-10 |
| **AWS** | **Deliberate non-participation, with reasoning.** API Gateway WebSocket: "Concurrent connections — Not applicable"; footnote: "API Gateway doesn't enforce a quota on concurrent connections. The maximum number of concurrent connections is determined by the rate of new connections per second and maximum connection duration of two hours." | **Yes.** "Connection duration for WebSocket API — 2 hours — [can be increased] No"; "Idle Connection Timeout — 10 minutes — No" | 2026-08-10 |

### 5.1 Stripe is the strongest single precedent

Quoted verbatim from source 14:

> Concurrency limits restrict the number of simultaneously active requests,
> separate from rate limits. Unlike rate limits, which in general reset after one
> second, the concurrency limit counts how many requests are in progress at any
> given moment. Hitting concurrency limits is less common than rate limit errors,
> and typically indicate long-lived or resource-intensive API requests such as
> list requests or those that include expansions.

and

> Issuing many long-lived requests can trigger concurrency limiting. Requests
> vary in the amount of Stripe server resources they use, and more
> resource-intensive requests can take longer and run the risk of causing the
> concurrency limiter to shed new requests.

`[COMPARATIVE]` Stripe independently arrives at `R8.10`'s "concurrency separate"
and states the causal link this leaf is about: **long-lived requests are the
thing concurrency limiting exists to catch.** Stripe also publishes a
machine-readable reason so a client can distinguish rate exhaustion from
concurrency exhaustion — a distinction this standard currently cannot express,
since `R11.2` gives both a bare `429`.

### 5.2 AWS is the strongest single precedent *against* a separate ceiling

`[FACT]` AWS caps duration hard (2 hours, not increasable) and idle time (10
minutes, not increasable), and then declines to cap concurrency at all, showing
its work: 500 new connections/second × 2 hours = up to 3,600,000 concurrent
connections, and that derived number *is* the bound.

`[INFERENCE]` This is the cleanest available argument for prioritizing duration
over concurrency. `R11.2` already forces the arrival-rate half of the product.
The missing factor is the duration half, and supplying it yields a concurrency
bound without a second knob.

### 5.3 The split across the eight

`[COMPARATIVE]` Three of eight (Google/AIP, Shopify, Zalando) publish nothing on
either dimension. Three of eight (Stripe, GitHub, Twilio) enforce concurrency,
and two of those three publish a number. Two of eight (AWS, Microsoft) publish a
hard duration. **No reference publishes both a duration cap and a concurrency
cap.** They are alternatives in practice, not a pair.

`[FACT]` The Zalando negative was probed twice: once against the guidelines
document itself, and once as a domain-restricted search of
`opensource.zalando.com` for concurrency, timeout, and long-lived-connection
rules. The second probe returned **no guideline hits** — every match was Zalando's
proxy **Skipper**, which does implement a connection-concurrency mechanism:
`-enable-tcp-queue` with `-max-tcp-listener-concurrency`, queueing new connections
past the limit, "to prevent Skipper requesting more memory than available in case
of too many concurrent connections"
(`https://opensource.zalando.com/skipper/operation/operation/`, accessed
2026-08-10).

`[INFERENCE]` That placement is itself the evidence: the same organization puts
`429` in the API *guidelines* (rule 153) and connection-concurrency in the
*proxy's operational configuration*. This is the sharpest available support for
argument 7.2.3 — the mechanism belongs in the standard, the number belongs with
the operator.

---

## 6. Comparable precedent with real numbers

Every number in this section was fetched directly on 2026-08-10. The
`survey-08` report recorded the Kubernetes randomization as
"**[COMPARATIVE, not primary-verified]**" and flagged it "Low — do not rely on
it." **It is now verified against the implementation and the flag registration,
and the survey's description was correct.**

### 6.1 Kubernetes watch — `timeoutSeconds` and its randomization

`[FACT]` The client-facing parameter, verbatim from `ListOptions` in source 25:

```
// Timeout for the list/watch call.
// This limits the duration of the call, regardless of any activity or inactivity.
```

`[FACT]` The server-side default and its randomization, verbatim from source 22
(`handleWatch`):

```go
timeout := time.Duration(0)
if opts.TimeoutSeconds != nil {
	timeout = time.Duration(*opts.TimeoutSeconds) * time.Second
}
if timeout == 0 && minRequestTimeout > 0 {
	timeout = time.Duration(float64(minRequestTimeout) * (rand.Float64() + 1.0))
}
```

`rand.Float64()` returns a value in `[0.0, 1.0)`, so the multiplier lies in
`[1.0, 2.0)` — the effective watch timeout when the client sets none is
**uniformly distributed over `[minRequestTimeout, 2 × minRequestTimeout)`**.

`[FACT]` The default, verbatim from source 24 (`NewConfig`):

```
MinRequestTimeout:              1800,
```

So the shipped default watch hold is **1800 to 3600 seconds (30 to 60 minutes)**.

`[FACT]` The operator-facing description, verbatim from source 23's flag
registration:

> An optional field indicating the minimum number of seconds a handler must keep
> a request open before timing it out. Currently only honored by the watch
> request handler, which picks a randomized value above this number as the
> connection timeout, to spread out load.

**Conflict surfaced.** Older and widely-quoted renderings of this flag's help
text state the randomized value is picked over
`[MinRequestTimeout, MinRequestTimeout*1.5]`. The current implementation uses a
factor in `[1.0, 2.0)`, and the current flag text no longer names a factor at
all. **The source code governs**, and the `1.5×` figure should not be cited.
Note that `master` is a moving branch; pin to a release tag before ratification.

`[INFERENCE]` Three design properties travel together here and are worth
carrying: the cap is a *server* policy with a client-settable override; the
client's override is bounded by nothing published (a client may request an
arbitrarily long `timeoutSeconds`, which is the residual hole); and the default
is *jittered* specifically to avoid synchronized reconnection.

### 6.2 Consul blocking queries

`[FACT]` Source 26, verbatim:

- "If not set, the wait time defaults to 5 minutes."
- "This is limited to 10 minutes."
- "A small random amount of additional wait time is added to the supplied maximum
  `wait` time to spread out the wake up time of any concurrent requests. This
  adds up to `wait / 16` additional time to the maximum duration."

`[COMPARATIVE]` Consul is the second independent implementation to jitter its
hold, and unlike Kubernetes it publishes a **server-side ceiling on the
client-requested value** (10 minutes). That is the hole Kubernetes leaves open,
closed.

### 6.3 AWS SQS long polling

`[FACT]` Source 21, verbatim: "The maximum long polling wait time is 20 seconds."
Short polling (`WaitTimeSeconds` = 0) is the default. "An empty response is sent
only if the polling wait time expires."

`[COMPARATIVE]` Confirms the `R13.11` shape already ratified: a documented
maximum hold, and an ordinary success response with an empty result on expiry.

### 6.4 Server-Sent Events — browser connection limits per origin

`[FACT]` The standards-level statement is weaker than the folklore. WHATWG HTML
Living Standard §9.2.7 (authoring notes), source 1:

> Clients that support HTTP's per-server connection limitation might run into
> trouble when opening multiple pages from a site if each page has an
> `EventSource` to the same domain.

The specification names the hazard and sets **no number**.

`[FACT]` The numbers come from the browser-platform reference, source 3,
verbatim:

> When **not used over HTTP/2**, SSE suffers from a limitation to the maximum
> number of open connections, which can be especially painful when opening
> multiple tabs, as the limit is *per browser* and is set to a very low number
> (6). … This limit is per browser + domain … When using HTTP/2, the maximum
> number of simultaneous *HTTP streams* is negotiated between the server and the
> client (defaults to 100).

`[INFERENCE]` Under HTTP/1.1 the ceiling is **6 concurrent SSE connections per
browser per origin**, and it is enforced by the *client*, not the server, so no
server-side rule can relax it. Under HTTP/2 the corresponding number is the
negotiated `SETTINGS_MAX_CONCURRENT_STREAMS`, defaulting to 100 in practice. The
practical consequence for this standard is that a browser-facing streaming API
already operates under a concurrency ceiling it does not set and cannot publish —
an argument for documenting the *server's* posture rather than pretending the
server owns the number.

### 6.5 The numbers side by side

| System | Mechanism | Published maximum hold | Jittered? | Client override |
| --- | --- | --- | --- | --- |
| AWS SQS `ReceiveMessage` | long poll | 20 s | no | `WaitTimeSeconds`, bounded at 20 s |
| Consul blocking query | long poll | 10 min (default 5 min) | yes, up to `wait / 16` | `wait`, bounded at 10 min |
| Gemini Live (audio+video) | session | 2 min | not stated | `contextWindowCompression` removes the bound |
| Gemini Live (audio-only) | session | 15 min | not stated | as above |
| Gemini Live connection | transport | ~10 min, `GoAway` first | not stated | session resumption token |
| Kubernetes watch | stream | 1800–3600 s default range | yes, factor in `[1.0, 2.0)` | `timeoutSeconds`, **unbounded above** |
| Azure/OpenAI Realtime | session | 60 min, `expires_at` in-band | not stated | none documented |
| AWS API Gateway WebSocket | connection | 2 h hard, 10 min idle | not stated | none — "Can be increased: No" |

`[COMPARATIVE]` The spread is 20 seconds to 2 hours — four orders of magnitude —
plus one documented unbounded configuration. **No single number can be
standardized.** What is common across all eight rows is that the number is
*published*.

---

## 7. Evidence for and against

### 7.1 FOR requiring a published maximum duration and concurrency posture

1. **Security standard names the control.** OWASP API4:2023 (source 2) lists
   "Execution timeouts" and "Maximum allocable memory" among required limits, and
   names "file descriptors" and "processes" among the resources exhausted. A
   held-open stream consumes exactly those. `R11.1`'s provenance is already
   "security requirement (OWASP API4:2023)" — so extending the same provenance to
   duration is consistent, not novel. `[FACT]` on the quotes; `[INFERENCE]` on
   the mapping.
2. **Universal practice among systems that hold connections open.** Eight
   independent implementations (§6.5), zero of which leave the bound
   undocumented while enforcing one. `[COMPARATIVE]`
3. **The undocumented bound is the observed failure.** Anthropic documents a
   `504 timeout_error` with no number and ships SDK-side guards synthesized from
   it; Microsoft points at a quotas page that does not carry the concurrent-session
   number it promises. Both are documentation defects with client-visible
   consequences. `[FACT]`
4. **`R13.6`'s truncation contract is weaker without it.** `R13.6` makes a client
   treat a close without a terminal frame as truncated, and `R12.10` obliges
   clients to act on that. A server that hits an *undocumented* cap and simply
   closes makes every successful long stream look like a failure. The fix is
   cheap and already has a home: end at the cap with the terminal frame `R13.6`
   requires. `[INFERENCE]`
5. **Duration is the higher-leverage half.** `R11.2` already mandates a rate
   limit; AWS shows that rate × duration derives concurrency (§5.2). One new
   requirement closes both. `[INFERENCE]`, grounded in `[FACT]` from source 20.
6. **Jitter has two independent implementations.** Kubernetes (factor in
   `[1.0, 2.0)`) and Consul (`wait / 16`), both for the stated purpose of
   de-synchronizing reconnects. That is enough to support a `SHOULD`.
   `[COMPARATIVE]`

### 7.2 AGAINST

1. **Unbounded streams are legitimate, and the standard already says so.**
   `R13.6`: "A stream that is unbounded by design — a watch, an event tail — has
   no normal end; such an API MUST document that the stream is unbounded." Gemini
   documents a configuration that extends sessions "to an unlimited amount of
   time." A rule demanding a finite maximum would forbid a shipping design at the
   only vendor that publishes the most numbers. `[FACT]`
2. **A duration cap without resumption degrades correctness.** Kubernetes' cap is
   safe only because `resourceVersion` plus `410 Gone` make the re-watch sound;
   Gemini's is safe only because of resumption tokens and a `GoAway` warning. In
   this standard, resumption is `R13.10`, a **`SHOULD`**, not a `MUST`. A `MUST`
   on capping duration would therefore mandate cutting streams that have no
   conforming way to resume. This is the single strongest argument for a
   documentation duty over an enforcement duty. `[INFERENCE]`, grounded in `[FACT]`
   from sources 11 and 22 and the ratified `R13.10`.
3. **The number belongs in deployment configuration, and every implementation
   agrees.** Kubernetes exposes `--min-request-timeout`; Consul exposes `wait`
   under a server ceiling; SQS exposes a queue attribute. The mechanism lives in
   the API; the number lives with the operator. That is precisely this standard's
   split between a normative rule and `R8.10`'s per-deployment numbers.
   `[COMPARATIVE]`
4. **The field does not treat this as table stakes for ordinary REST.** Three of
   eight standard references publish nothing on either dimension (§5.3), and AWS
   deliberately declines to enforce a concurrency quota. `[FACT]`
5. **Concurrency is not reliably publishable.** Twilio enforces a ceiling and
   exposes live usage in a header but does not publish the number, because it is
   account-specific. Under HTTP/1.1 a browser-facing stream is already capped at
   6 per origin by the *client* (§6.4). A rule demanding a published concurrency
   *number* would be widely deviated from; a rule demanding a published *posture*
   would not. `[COMPARATIVE]`
6. **`R8.10` may already suffice.** If the distributive reading of "published:"
   in §3.3 is right, the concurrency duty is discharged and `ST-013` is
   editorial. `[INFERENCE]`

### 7.3 Documentation duty versus enforcement duty — the resolution

`R11.1` requires both ("MUST publish **and** enforce"). `R13.11` requires
documentation only ("MUST document its maximum hold duration"). Which model fits?

`[POLICY]` **Follow `R13.11`, with a conditional enforcement clause.** The
reasoning:

- An **unconditional enforcement duty** collides with argument 7.2.1 and 7.2.2:
  it would forbid the watch and event-tail designs `R13.6` explicitly permits,
  and would mandate cutting streams whose resumption mechanism is only a `SHOULD`.
- A **pure documentation duty** is too weak in one specific place. If a server
  documents a maximum and then abandons the connection at it, every client
  reports truncation under `R13.6`. The *shape* of the ending must be governed
  even though the *existence* of a maximum need not be.
- The resolution is conditional: **documenting is unconditional; enforcing is
  what you owe once you have declared a number.** An API that declares itself
  unbounded per `R13.6` is exempt from the enforcement half by construction,
  because it has declared no number to enforce.

**Should unbounded streams be exempt if documented as unbounded per `R13.6`?**
**Yes**, and the exemption should be explicit rather than implied — otherwise a
conformance reviewer reading `ST-012` alone would score every watch API
non-conforming. `R13.6`'s existing sentence is the exemption; `ST-012` should
cite it by number so the two rules read as one contract.

---

## 8. Anti-patterns

Four failure modes observed in the sources above, each with the concrete
instance that exhibits it. Every one is prevented by a clause in §9's proposals.

1. **Enforcing a bound you do not publish.** The server holds a limit; the
   documentation names only the error, not the number; client libraries then
   synthesize a bound from an observed timeout and enforce it locally. Observed:
   Anthropic documents `504 timeout_error` — "The request timed out while
   processing" — with no value, and separately documents that "The SDKs validate
   that your non-streaming Messages API requests are not expected to exceed a
   10-minute timeout" (sources 5 and 6). The cost is that the guard lives in
   every client rather than once in the contract, and drifts from the server's
   real behavior. `[FACT]` on the quotes; `[INFERENCE]` on the cost.

2. **Ending a capped stream by closing the connection.** A server reaches its
   maximum hold and drops the socket without a closing frame. Under `R13.6` the
   client must classify that as truncation, so every *successful* long stream is
   reported as a failure and, under `R12.10`, acted on as one. The field avoids
   this deliberately: Gemini sends a `GoAway` message before the ~10-minute
   connection limit, and AWS API Gateway "returns a status code when the client
   is idle for 10 minutes or reaches the maximum 2 hour connection lifetime."
   `[COMPARATIVE]`

3. **Counting a stream once, at open.** A concurrency limiter that increments on
   arrival and decrements before the response completes measures arrivals, not
   occupancy — which is what the rate limiter already measures, so the second
   dimension buys nothing. Stripe names the opposite behavior as the point of the
   mechanism: "the concurrency limit counts how many requests are in progress at
   any given moment," and "Issuing many long-lived requests can trigger
   concurrency limiting" (source 14). No reference was found *exhibiting* this
   anti-pattern; it is the degenerate implementation `ST-013` exists to exclude,
   and is recorded as `[INFERENCE]` rather than as observed practice.

4. **Documenting that a limit exists, at a page that does not carry it.** The
   reader is told a ceiling governs them and is sent somewhere to find it; the
   destination does not publish it. Observed: Microsoft's Realtime page states
   "The Realtime API has specific rate limits for audio tokens and concurrent
   sessions. Before deploying to production, review Azure OpenAI quotas and
   limits," and the referenced quotas page publishes RPM and TPM per model but no
   concurrent-session figure (sources 12 and 13). This is worse than silence,
   because it converts an unpublished limit into an apparently published one.
   `[FACT]`

A fifth candidate was considered and rejected as *not* an anti-pattern:
**declining to enforce a concurrency ceiling at all.** AWS does exactly this and
shows its reasoning — with a hard duration cap and a bounded new-connection rate,
peak concurrency is derived rather than configured (§5.2). That is a defensible
design, not a defect, and `ST-013` is written to accommodate it: it governs how a
stream is *counted* against whatever posture `R8.10` produced, not whether a
numeric ceiling must exist.

---

## 9. Proposed rule text

Proposals only. Provisional IDs continue the `ST-` series (`ST-001` … `ST-011`
are ratified in `research/decisions/baseline-04-streaming.decision.md`).

### `ST-012` — Stream duration bound

**Strength:** `MUST` (documentation) + `MUST` (enforcement, conditional)
**Switch scope:** `streaming` applicability switch, per §13's switch scope
**Classification:** `[POLICY]` on the requirement; `[COMPARATIVE]` on the
publish-your-number practice (eight implementations, §6.5); security-grounded in
OWASP API4:2023's "Execution timeouts"
**Confidence:** moderate-high

> **ST-012** For each streaming endpoint, an API MUST document either the maximum
> duration for which the server will hold the response open, or that the stream
> is unbounded by design. The unbounded declaration is the one R13.6 already
> requires; an API that has made it owes nothing further under this rule.
>
> Where a maximum duration is documented, the server MUST enforce it, and MUST
> end the stream at the maximum with the terminal frame R13.6 requires rather
> than by closing the connection without one — a server-initiated close at a
> published limit is a normal end, not a truncation, and a client MUST NOT be
> made to report it as one.
>
> A documented maximum SHOULD be randomized per connection within a published
> range rather than applied as a fixed deadline, so that streams opened together
> do not expire together.

Notes for the ratification walk:

- The third paragraph is the jitter `SHOULD`, supported by two independent
  implementations (Kubernetes' `[1.0, 2.0)` factor, Consul's `wait / 16`). It is
  the weakest of the three and could be dropped without harming the rule.
- The rule sets **no number**. §6.5's four-orders-of-magnitude spread is the
  evidence that no number is standardizable; the number is a deployment choice.
- Interaction with `R13.10` (resumable streams, a `SHOULD`): an API that both
  declares a maximum and offers resumption gives clients a sound continuation.
  An API that declares a maximum without resumption is conforming but degrades
  under long workloads. Worth a non-normative note rather than a rule.

### `ST-013` — Streams occupy a concurrency slot

**Strength:** `MUST`
**Switch scope:** `streaming` switch on the streaming clause; the underlying
concurrency dimension is `R8.10`'s and is unconditional
**Classification:** `[POLICY]` — a clarification of an already-ratified axis
**Confidence:** moderate on the `MUST`; high that the clarification is needed

> **ST-013** A held-open stream occupies one unit of the concurrency dimension of
> R8.10's rate-limit axis for the whole time the server holds it open, not merely
> at the moment the request arrives. An API MUST document how streams are counted
> against its published concurrency posture. Where a request is rejected because
> the concurrency posture rather than the rate posture is exhausted, the 429
> required by R11.2 SHOULD distinguish the two conditions in its problem `code`.

Notes for the ratification walk:

- The first sentence prevents the degenerate implementation that counts a stream
  once at open, which turns the concurrency limiter back into a rate limiter.
- The `SHOULD` in the last sentence is modelled directly on Stripe's
  `Stripe-Rate-Limited-Reason` header values `global-concurrency` and
  `endpoint-concurrency`. This standard would carry the distinction in `code`
  rather than a proprietary header, because `R5.13`/`R5.16` already own that
  vocabulary and `R11.4` discourages new proprietary quota headers.
- **The register entry in §13.4 should be corrected, not merely resolved.** Its
  claim that "no … per-principal concurrency ceiling is required" contradicts
  `R8.10`'s ratified axis default.

### `ST-014` — Editorial: scope `R6.5` to streamed collections

**Strength:** none — a scoping note, not a new rule
**Classification:** `[POLICY]`
**Confidence:** high

`R6.5` already binds a streamed collection, and `R6.4`'s 1.1.0 widening is the
precedent. The Phase 6 walk did not scope `R6.5`, so the standard has never said
so. One clause in `R6.5`'s provenance note, or one sentence in §13, closes it:
*a streamed collection documents and enforces its maximum `limit` exactly as an
unstreamed one does.* No text change to the rule itself is needed.

---

## 10. Declined alternatives

1. **A fixed numeric maximum in the standard** (for example, "a stream MUST NOT
   be held longer than one hour"). Declined: the field's published maxima span
   20 seconds to 2 hours plus one unbounded configuration (§6.5). Any number
   would be arbitrary and immediately deviated from. `R8.10`'s flip-trigger
   machinery is the correct home for numbers, and a numeric axis could be added
   there later without touching §13.

2. **Adding "stream duration" to `R11.1`'s list.** Declined: `R11.1`'s three
   dimensions are request-shape maxima the client supplies and the server
   validates *before* committing a status. Duration is a server-side hold accrued
   *after* commitment — the same boundary `R13.7` and `R13.8` are built on.
   `R13.11` already set the precedent that a hold duration belongs in §13.
   Putting it in `R11.1` would also drag it out of the `streaming` switch's
   scope, since `R11.1` is unconditional.

3. **A separate per-principal concurrent-*stream* ceiling, distinct from
   `R8.10`'s concurrency dimension.** Declined on two independent grounds. First,
   `R8.10` already requires a published concurrency dimension and a stream is one
   in-flight request (§3.3), so a second ceiling double-counts. Second, AWS
   demonstrates that once duration and arrival rate are bounded, concurrency is
   derived rather than configured (§5.2). `ST-013` clarifies the existing axis
   instead.

4. **Requiring the deadline in-band, on an opening frame** — the Azure
   `expires_at` pattern (§4.4). Declined for this cycle, with regret: it is the
   best design found, but it would require reserving an opening-frame type and a
   member name in §1.10, which is a larger change than this leaf's evidence
   supports. Recorded as a candidate for a later cycle. `R13.5`'s frame-typing
   obligation is the hook it would attach to.

5. **A `MUST` on jittering the maximum.** Declined: two implementations support
   the practice, which is enough for a `SHOULD` and not enough for a `MUST`. A
   fixed deadline is not incorrect, merely operationally worse under synchronized
   client fleets.

6. **Requiring a distinct HTTP status or a proprietary header for concurrency
   exhaustion.** Declined: `R11.2` fixes `429` for quota exhaustion and `R11.4`
   discourages new proprietary quota headers. `ST-013` carries the distinction in
   the problem `code` instead, which is the vocabulary `R5.16` already governs.

---

## 11. What could not be verified

1. **OpenAI's own maximum Realtime session duration.** OpenAI's public Realtime
   guide (source 8) states no number. The 60-minute figure rests solely on
   Microsoft's documentation of the same API (source 12). Treat it as a
   single-source claim about Azure's deployment, not a verified OpenAI-platform
   fact. Community sources reporting 60 minutes were found but are not primary
   and are not cited as evidence. A **stale 30-minute figure also circulates**
   for Azure Realtime, including in a Microsoft Q&A thread surfaced during this
   run; the fetched primary page (source 12, doc date 2026-07-29) states 60
   minutes twice and governs. Do not cite 30.

2. **Any concurrent-session number for Azure OpenAI Realtime.** Source 12 asserts
   such limits exist and points at source 13, which does not publish them. The
   pointer is verified; the number is not obtainable from public documentation.

3. **Whether Anthropic enforces a server-side maximum streaming duration.** The
   `504 timeout_error` in source 5 proves a server-side timeout exists for
   *processing*; no page states whether it applies to a committed stream, nor its
   value. The 10-minute figure is documented only as an SDK-side validation.

4. **Whether OpenAI or Gemini enforce an unpublished per-key concurrency
   ceiling.** Both rate-limit pages are silent. Silence is recorded as a verified
   negative on *publication*, which is the claim this report makes; it is not
   evidence about enforcement.

5. **Twilio's actual concurrency numbers.** Twilio publishes the mechanism, the
   header, and the error code, but the ceiling is account-specific and absent
   from public documentation. Product-specific figures surfaced in search (for
   example, 30 concurrent Function invocations per account) were not confirmed
   against a primary Twilio page within this run and are therefore not carried in
   the §5 table.

6. **Kubernetes numbers against a release tag.** All four Kubernetes facts were
   read from `master` on 2026-08-10. `MinRequestTimeout: 1800` and the
   `rand.Float64() + 1.0` factor should be re-confirmed against the tag the
   standard intends to cite before ratification. The conflicting `1.5×` figure in
   older circulating copies of the flag help text is *not* what the current
   implementation does; do not cite it.

7. **Whether `R8.10`'s "published:" distributes across the whole list.** This is
   a reading of compressed ratified prose, not a fact. §3.3 states both readings.
   If the narrow reading is correct, `ST-013` becomes load-bearing rather than
   editorial, and its `MUST` strength is better justified.

8. **HTTP/2 `SETTINGS_MAX_CONCURRENT_STREAMS` defaults in shipping browsers.**
   The figure of 100 comes from source 3, a browser-platform reference, not from
   RFC 9113 (which specifies no default ceiling and recommends implementations
   not set it below 100). Per-browser current values were not probed.
