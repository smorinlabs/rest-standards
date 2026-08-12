# baseline-04e — Idempotency-key replay of a streaming request

> ## ⚠ Correction (2026-08-12) — this run's central negative was falsified
>
> **Do not cite this run's precedent claim.** Below, this report states that
> "Every surveyed API that streams has **no** idempotency key at all" and that
> the combination has "**zero published precedent** — nobody has solved this
> because nobody has built it." **That is false.**
>
> **Replicate's Cog does both on one endpoint.** `PUT /predictions/<id>` with
> `Accept: text/event-stream` is documented as "Streaming, idempotent," and a
> repeat **attaches** to the running stream. That was verified independently
> against Cog's own capability table. The claim was not merely incomplete: it
> pointed the opposite way from the evidence, because a shipped implementation
> attaches where this run reasoned toward rejection.
>
> This run also escalated an expired IETF draft's `409` from `SHOULD` to
> `MUST`, presenting a draft's recommendation as a requirement.
>
> The leaf was re-run as
> [`…report.2026-08-10b.md`](baseline-04e-stream-idempotency-replay.report.2026-08-10b.md),
> which supersedes this one on the precedent question and on the response
> shape. `R13.15` as shipped documents **attach or `409`** as a choice, rather
> than mandating `409`, precisely because of what run b found.
>
> This run is kept unedited: the record of what a run concluded is the point of
> keeping it, and a report quietly corrected is a report that cannot be audited.
> Its analysis of the three server states and of "the stored response" for a
> stream survived ratification and is unaffected.

*Research leaf under `baseline-04` (streaming), answering the fourth row of
`rest-api-standard.md` §13.4 "Known unresolved interactions": **what an
`Idempotency-Key` replay of a streaming request delivers, and whether it may
re-execute the underlying work.** Series `baseline` = prescriptive: this report
**proposes** rule text; it does not ratify it. Ratification is a `decisions/`
record.*

*Run 2026-08-10. All web access dates in this report are 2026-08-10 unless a
row states otherwise. The ratified record
`research/decisions/baseline-04-streaming.decision.md` is treated as settled;
§4 and §7 below relate to `R13.10` without reopening it.*

*Rules touched: `R3.9` (idempotency keys), `R12.10` (client obligations on a
truncated stream), `R12.1` (retry discipline), `R13.7` (post-commit error
frames), `R13.9` (stream ↔ operation-resource identity), `R13.10`
(resumption), `R10.9` (`202` operation discovery).*

---

## 1. TL;DR and recommendation

**The single most important finding is a verified negative: no reference in
the field ships the combination this standard permits.** `[COMPARATIVE]` Every
surveyed API that implements an idempotency key does **not** stream responses
(Stripe, Shopify, Twilio, AWS control planes, Azure via the OASIS
`Repeatability-*` headers). Every surveyed API that streams has **no**
idempotency key at all (OpenAI, Anthropic, Google Gemini, AWS Bedrock
Runtime). `R3.9`'s MUST plus §13's streaming allowance therefore manufactures
an intersection with **zero published precedent**. Nobody has solved this
because nobody has built it. That is the reason the standard must rule it
rather than cite it.

**The exposure is real, not hypothetical.** `[INFERENCE]` The AI providers'
billable generation endpoints *are* streaming non-idempotent operations: a
`POST` that costs money, returns `200`, and streams tokens. Where any recovery
from a mid-stream failure is documented at all, it is re-execution-shaped —
AWS Bedrock's `ModelStreamErrorException` says, verbatim, "An error occurred
while streaming the response. Retry your request." `[FACT]` That is re-execution and re-billing
by design, and it is exactly the failure mode `R12.10` was written to stop a
client from walking into on a payment endpoint.

**Recommendation — rule the three server states, and define "the stored
response" for a stream in terms of the *outcome*, not the frame sequence.**
The proposed text is in §6 (`ST-E01`, with an amendment clause riding `R3.9`).
Its three parts:

1. **Original still executing.** The server MUST NOT begin a second execution
   and MUST answer with a defined conflict error. Three independent sources
   agree on this shape (expired IETF draft, Stripe, Shopify), which makes it
   the only part of the answer the field actually supplies.
2. **Original completed (any terminal state, success or failure).** The server
   MUST NOT re-execute. It MUST deliver the recorded outcome. It MAY do so as
   a fresh stream replayed from the first frame, or as a non-streamed
   representation — and it MUST document which, because a client cannot
   discover it.
3. **Original's execution began but reached no terminal state.** The standard
   MUST require the API to document its behavior and MUST forbid silent
   re-execution; it should not mandate a single behavior, because no source
   documents this case (§8).

**Second recommendation — add the structural escape and prefer it.** An API
whose streaming endpoint is a non-idempotent mutation SHOULD decouple
execution from delivery: the mutation is a short non-streaming request that
returns an operation resource (`R10.9`), and the stream is a **safe `GET`**
over that resource, resumable under `R13.10`. This is not invented — it is
OpenAI's shipped `background: true` design `[FACT]`, and under it the replay
question dissolves, because the retried request is a `GET`, which RFC 9110
§9.2.2 already makes idempotent.

**Not recommended: a flat prohibition on streaming non-idempotent
mutations.** Assessed seriously in §7. Google AIP-151 does say "The response
**must not** be a streaming response" `[FACT]`, but at its actual scope — LRO
methods — not all mutations, and a flat ban would forbid the exact shape §13
was ratified to serve.

**Confidence: high on part 1, moderate-high on part 2, moderate on part 3,
high on the verified negative.** The verified negative is the load-bearing
claim and rests on per-vendor primary sources listed in §3.

---

## 2. Standards-and-currency matrix

Authority classes used throughout: **A** = published IETF standards-track RFC
or Internet Standard; **B** = expired or in-flight Internet-Draft, or a
non-IETF committee specification; **C** = vendor or consortium protocol
specification; **D** = vendor API-design guideline; **E** = shipped vendor API
documentation; **F** = engineering design document with no standards status.

| Source | URL | Class | Published / revised | Accessed | Bears on |
| --- | --- | --- | --- | --- | --- |
| RFC 9110, *HTTP Semantics* §9.2.2 (idempotent methods), §6.1 (framing and completeness) | https://www.rfc-editor.org/rfc/rfc9110.txt | **A** — Internet Standard, STD 97 | June 2022 | 2026-08-10 | Retry safety; the "before any part of a response is received" clause; what "complete" means |
| RFC 9457, *Problem Details for HTTP APIs* §3.1 | https://www.rfc-editor.org/rfc/rfc9457.html | **A** — Proposed Standard | July 2023 | 2026-08-10 (prior verification in `baseline-04`, 2026-08-10) | The error shape a conflict response uses (`R5.12`, `R13.7`) |
| `draft-ietf-httpapi-idempotency-key-header-07` | https://www.ietf.org/archive/id/draft-ietf-httpapi-idempotency-key-header-07.txt | **B** — **EXPIRED** Internet-Draft, never an RFC | Revision 07 published 2025-10-15; **expired 2026-04-18**; datatracker status "Expired", last updated 2026-04-18 | 2026-08-10 | The only near-standard text on repeated keys; §2.6 "Concurrent Request" |
| OASIS *Repeatable Requests Version 1.0* | https://docs.oasis-open.org/odata/repeatable-requests/v1.0/cs01/repeatable-requests-v1.0-cs01.html | **B** — OASIS Committee Specification 01 | Approved 2020-07-07 | 2026-08-10 | `Repeatability-Request-ID` / `-First-Sent` / `-Result`; re-execution on a recorded 4xx/5xx |
| Model Context Protocol, *Transports* (Streamable HTTP), revision 2025-06-18 | https://modelcontextprotocol.io/specification/2025-06-18/basic/transports | **C** — consortium protocol spec, not IETF | Revision dated 2025-06-18 | 2026-08-10 | Resumability and redelivery; disconnection is not cancellation |
| gRPC proposal **A6**, *Client Retries* | https://raw.githubusercontent.com/grpc/proposal/master/A6-client-retries.md | **F** — gRFC design document, no standards status | Living document in `grpc/proposal` | 2026-08-10 | *Committed* RPC; commit-on-Response-Headers |
| Google AIP-151, *Long-running operations* | https://google.aip.dev/151 | **D** | State approved; last updated 2025-02-04 | 2026-08-10 | "The response **must not** be a streaming response" |
| Google AIP-155, *Request identification* | https://google.aip.dev/155 | **D** | State approved; last content-bearing change 2024-01-08 (per `baseline-02g`) | 2026-08-09 (prior leaf) | `request_id`; duplicate → prior successful response; no payload fingerprinting |
| Microsoft / Azure REST API Guidelines | https://raw.githubusercontent.com/microsoft/api-guidelines/vNext/azure/Guidelines.md | **D** | Changelog entry 2023-Apr-21 on POST repeatability | 2026-08-10 | "all service operations (including POST) **must** be idempotent"; repeatability headers; **silent on streaming** |
| Zalando RESTful API Guidelines, rule 230 | https://raw.githubusercontent.com/zalando/restful-api-guidelines/main/chapters/http-headers.adoc | **D** | Living document, `main` branch | 2026-08-10 | `Idempotency-Key` at **MAY**; key cache stores the response "regardless of whether it succeeded or failed" |
| Stripe, *Idempotent requests* | https://docs.stripe.com/api/idempotent_requests | **E** | Living documentation | 2026-08-10 | Stored status code and body; ≥24 h; results saved only after execution begins |
| Stripe, *Advanced error handling* (idempotency and retries) | https://docs.stripe.com/error-low-level | **E** | Living documentation | 2026-08-10 | `Idempotent-Replayed: true`; `409` on concurrent conflict; caching of `400`s and `500`s |
| Shopify, *Implementing idempotency* | https://shopify.dev/docs/api/usage/implementing-idempotency | **E** | Living documentation | 2026-08-10 | 24 h tracking; `IDEMPOTENCY_CONCURRENT_REQUEST`; cached response may differ from the original |
| Shopify, *Idempotent requests* | https://shopify.dev/docs/api/usage/idempotent-requests | **E** | Living documentation | 2026-08-10 | Which mutations accept keys |
| AWS Bedrock Runtime, `InvokeModelWithResponseStream` | https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_InvokeModelWithResponseStream.html | **E** | Living documentation | 2026-08-10 | Full parameter list — **no** client or idempotency token; `ModelStreamErrorException` says "Retry your request" |
| AWS Builders' Library, *Making retries safe with idempotent APIs* | https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/ | **F** — vendor engineering essay | Living | 2026-08-10 | "at most once" framing; **silent** on partial or streamed responses |
| OpenAI, *Background mode* | https://developers.openai.com/api/docs/guides/background | **E** | Living documentation | 2026-08-10 | `background: true`; `starting_after` over `sequence_number`; "roughly 10 minutes" retention; **no** dedup statement |
| OpenAI, published OpenAPI specification | https://raw.githubusercontent.com/openai/openai-openapi/master/openapi.yaml | **E** | Fetched 2026-08-09 in `baseline-02g` | 2026-08-09 | Zero idempotency-key parameters |
| Anthropic, *Errors* | https://platform.claude.com/docs/en/api/errors | **E** | Living documentation | 2026-08-09 (prior leaf) | SDK auto-retry with no key on the wire |
| Google Gemini discovery document | https://generativelanguage.googleapis.com/$discovery/rest?version=v1beta | **E** | Revision 20260806 | 2026-08-09 (prior leaf) | Zero `idempot` hits; no `requestId` |

**Currency warnings.** `[FACT]` The IETF `Idempotency-Key` draft is **expired**
and was never published as an RFC; it MUST NOT be cited as a standard, exactly
as `R3.9` already states. `[FACT]` The OASIS Repeatable Requests document is a
Committee Specification, not an OASIS Standard. `[FACT]` gRPC A6 is a design
proposal in a GitHub repository with no standards status whatever; it is used
here as engineering precedent, never as authority.

---

## 3. Field evidence per vendor

### 3.1 The mandatory AI-provider deep dive

**OpenAI.** `[FACT]` Re-verified against the prior leaf `baseline-02g`: the
published OpenAPI specification contains no idempotency-key parameter of any
kind, and the only `in: header` parameter in the whole specification is
`openai-beta`. `[FACT]` The one client-supplied header OpenAI documents,
`X-Client-Request-Id`, carries the instruction "Make this value unique per
request" — the *opposite* of idempotency-key semantics.

What OpenAI does instead is the interesting part. `[FACT]` The Responses API
accepts `background: true`, which creates a retrievable `response` resource;
"Response data is temporarily stored to disk for roughly 10 minutes to enable
asynchronous execution and polling." Every streaming event carries a
`sequence_number`, and a dropped stream is recovered with
`GET /v1/responses/{id}?stream=true&starting_after=<n>` — the guide's own
instruction is "You will want to keep track of a 'cursor' corresponding to the
`sequence_number` you receive in each streaming event." `background: true` and
`stream: true` may be combined, so the original `POST` streams *and* the work
survives the connection. `[FACT]` The background guide contains no statement
about duplicate-request handling or deduplication.

`[INFERENCE]` So a second `POST` with identical inputs produces a second
`response` object with a second `id` and a second charge. OpenAI's answer to
truncation is not replay-by-key; it is **resume-by-`GET`**. The retried request
is safe under RFC 9110 §9.2.2, so the idempotency question never arises.

**Anthropic.** `[FACT]` No idempotency mechanism (`baseline-02g`, re-checked):
the errors page does not contain the word, and the official SDKs "automatically
retry transient failures … twice by default" with no key on the wire.
`[FACT]` Anthropic's documented truncation recovery is a *re-prompt*: capture
the partial response, construct a continuation request. Its own limitation is
stated: "Tool use and extended thinking blocks cannot be partially recovered.
You can resume streaming from the most recent text block." `[INFERENCE]` This
is a **new generation**, not a replay and not a resumption — a second billable
execution by construction.

**Google Gemini.** `[FACT]` The authoritative discovery document (revision
20260806) has zero `idempot` matches and no `requestId` parameter, global or
per-method; Gemini does not implement its own organization's AIP-155
(`baseline-02g`). `[FACT]` Gemini documents no resumption and no truncation
obligation (`survey-08` §4.5, §4.6). `[INFERENCE]` A retried streaming
generation re-executes with no server-side mitigation of any kind.

**The dormant SDK machinery, restated because it is easy to over-read.**
`[FACT]` All four official OpenAI and Anthropic SDKs ship identical
Stainless-generated `idempotencyKey` request options with retry-reuse logic,
and **none assigns a header name**, so nothing is emitted on the wire and a
caller-supplied key is silently dropped (`baseline-02g`, verified by local
greps of shallow clones on 2026-08-09). `[INFERENCE]` If either vendor
switches it on, the placement will be a Stripe-style request header — but this
is latent plumbing, not shipped behavior, and it says nothing about streams.

### 3.2 The eight standard references

**Stripe — the deep-dive target, and the sharpest evidence.** `[FACT]`
Verbatim: "Stripe's idempotency works by saving the resulting status code and
body of the first request made for any given idempotency key, regardless of
whether it succeeds or fails. Subsequent requests with the same key return the
same result, including `500` errors." `[FACT]` And the boundary condition this
leaf needs: "We save results only after the execution of an endpoint begins.
If incoming parameters fail validation, or the request conflicts with another
request that's executing concurrently, we don't save the idempotent result
because no API endpoint initiates the execution. You can retry these
requests." `[FACT]` The concurrent case is an HTTP `409`; Stripe's own status
table reads "409 Conflict — The request conflicts with another request
(perhaps due to using the same idempotent key)", and the error carries
`code: idempotency_key_in_use` (recorded with its message in `survey-05` §4A,
2026-07-19). `[FACT]` A replayed response is marked by the response header
`Idempotent-Replayed: true`. `[FACT]` Retention: "keys expire out of the
system after 24 hours."

`[FACT]` **Stripe does not stream.** The whole API is request/response, and
asynchronous outcomes are delivered by webhook, not by a held-open connection
(`survey-05` §5, 2026-07-19; nothing on the two idempotency pages fetched
today mentions streaming, partial delivery, or incomplete responses).
`[INFERENCE]` Therefore Stripe has never had to answer this question, and its
"stored response" is always a complete status line plus a complete body.
Reading Stripe as authority for streaming replay is reading past the end of
the evidence.

**GitHub.** `[FACT]` No idempotency key of any kind; retry-safety comes only
from natural method idempotence, and conditional requests are explicitly not
supported for unsafe methods (`survey-05` §5, 2026-07-19). No streaming
mutation surface. Contributes a verified negative on both axes.

**Google / AIP.** `[FACT]` AIP-155 puts a `request_id` in the request message
(a query parameter over REST); a duplicate "**should** return the response for
the previously successful request"; "APIs **may** choose any reasonable
timeframe"; and there is **no payload fingerprinting** anywhere in the AIP
corpus (`baseline-02g`, 2026-08-09). `[FACT]` AIP-151 states of a long-running
operation method: "The response **must not** be a streaming response." `[FACT]`
Neither AIP addresses an incomplete or partially delivered response.
`[INFERENCE]` Google's position is structural avoidance: long work returns an
operation resource, and the operation resource is never a stream, so a
`request_id` never has to describe a partial delivery.

**Microsoft / Azure.** `[FACT]` The guidelines state that "to enable customers
to write fault-tolerant applications, _all_ service operations (including
POST) **must** be idempotent," and prescribe the OASIS `Repeatability-*`
headers with a tracked window of "at least 5 minutes," plus `501 Not
Implemented` for an operation that receives valid repeatability headers it does
not support. `[FACT]` **Verified negative:** the word "stream" appears in the
Azure guidelines only inside "downstream" — four occurrences, all in the
downstream-errors section. Azure's guidance is entirely silent on streaming.

`[FACT]` The underlying OASIS specification says the server must return
"the same response code and body as was generated when the original request
with that Repeatability-Request-ID was processed, **or** the response code and
response body resulting from re-executing the request if the original response
code was 4xx or 5xx." It has no text on in-progress originals and none on
partial or streamed responses.

**Twilio.** `[FACT]` Header-based key on *selected* product endpoints
(`Idempotency-Token`), not across all writes; the similarly named
`I-Twilio-Idempotency-Token` travels on **outbound webhooks Twilio sends to
you**, which is the opposite direction (`survey-05` §5, 2026-07-19). No
streaming response surface documented. Nothing on partial responses.

**Shopify.** `[FACT]` "Shopify tracks idempotency keys for **24 hours** from
the original request." `[FACT]` "After the original request completes
successfully, any duplicate requests with the same idempotency key receive the
cached GraphQL response without reprocessing." `[FACT]` A duplicate arriving
while the original is still processing gets `IDEMPOTENCY_CONCURRENT_REQUEST`.
`[FACT]` And a caveat no other vendor states: "on rare occasions, the cached
GraphQL response may not be the same as the original one, as the cached
response is constructed from database records, which may have changed since
the original successful response." `[INFERENCE]` That caveat matters here —
Shopify's replay is a *reconstruction*, not a recording, which is precisely
the design that cannot reproduce a frame sequence. Shopify's bulk path is an
asynchronous job producing a downloadable JSONL file, not a streamed mutation.

**Zalando.** `[FACT]` `Idempotency-Key` support is a **MAY** (rule 230). The
key cache stores "the response and the request hash (optionally) of the first
request … regardless of whether it succeeded or failed," with a suggested 24-hour
expiry, and the service "can now look up the _unique request key_ … and serve
the response from the key cache, **instead of re-executing the request**."
`[FACT]` Zalando names the difficulty explicitly: "To grant a reliable
idempotent execution semantic, the resource and the key cache have to be
updated with hard transaction semantics — considering all potential pitfalls of
failures, timeouts, and concurrent requests in a distributed systems. This
makes a correct implementation exceeding the local context very hard." `[FACT]`
Zalando also concedes the provenance: the header "is not standardized in an
RFC. Our only reference are the usage in the Stripe API." Nothing on streaming.

**AWS.** `[FACT]` Control-plane idempotency is a body field `ClientToken`, with
`IdempotentParameterMismatch` on same-token-different-parameters
(`survey-05` §5, 2026-07-19). `[FACT]` **Verified negative on the streaming
side:** the full request parameter list for
`InvokeModelWithResponseStream` — `accept`, `contentType`,
`guardrailIdentifier`, `guardrailVersion`, `modelId`,
`performanceConfigLatency`, `requestMetadata`, `serviceTier`, `trace`, plus
the body — contains **no** client token, idempotency token, or request-ID
parameter. `[FACT]` Its documented mid-stream failure is
`ModelStreamErrorException` (HTTP 424): "An error occurred while streaming the
response. Retry your request." `[FACT]` The AWS Builders' Library idempotency
essay frames the goal as "at most once" but says nothing about concurrent
retries, at-most-once for streams, or partial responses.

### 3.3 Comparison table

Columns: **Key?** = ships a client-supplied idempotency key; **Streams?** =
ships an incrementally delivered response; **Both?** = the intersection this
leaf is about; **In-flight duplicate** = documented behavior when a retry
arrives before the original finished; **Partial-response text** = any documented
behavior for a response that began and did not finish.

| Reference | Key? | Streams? | Both? | In-flight duplicate | Partial-response text |
| --- | --- | --- | --- | --- | --- |
| Stripe | Yes — `Idempotency-Key`, ≥24 h | No | **No** | `409`, `idempotency_key_in_use`; result not saved, retryable | **None** |
| Shopify | Yes — 24 h, selected mutations | No (bulk is a downloadable file) | **No** | `IDEMPOTENCY_CONCURRENT_REQUEST` | **None**; replay is reconstructed from records |
| Twilio | Partial — selected endpoints | No | **No** | Not published | **None** |
| Zalando (guideline) | MAY | Not addressed | **No** | Not addressed | **None** |
| Google AIP (guideline) | `request_id`, MAY, no fingerprint | Forbidden for LRO methods | **No** | Not addressed | **None** |
| Azure (guideline) | SHOULD — OASIS `Repeatability-*`, ≥5 min | Not addressed at all | **No** | Not addressed | **None** |
| AWS control plane | `ClientToken` body field | No | **No** | `IdempotentParameterMismatch` covers mismatch only | **None** |
| AWS Bedrock Runtime | **No** | Yes | **No** | n/a | "Retry your request" — re-execute |
| GitHub | **No** | No | **No** | n/a | **None** |
| OpenAI | **No** | Yes | **No** | n/a | Resume by `GET` + `starting_after`, ~10 min retention |
| Anthropic | **No** | Yes | **No** | n/a | Re-prompt — a new generation, not a replay |
| Google Gemini | **No** | Yes | **No** | n/a | **None** |
| IETF draft-07 (expired) | Defines the header | Not addressed | **No** | "resource conflict error" — `409` | **None** |
| OASIS Repeatable Requests | Defines the headers | Not addressed | **No** | Not addressed | **None** |

`[COMPARATIVE]` **The `Both?` column is empty in fourteen rows.** That is the
report's central finding and it is established by fourteen independent sources
rather than by inference.

---

## 4. Replay versus resumption — the relationship, stated explicitly

These are two different mechanisms with different preconditions, different
carriers, different failure modes, and different retention economics. **Neither
satisfies the other.** `[POLICY]`

| Axis | Idempotency replay (`R3.9`) | Stream resumption (`R13.10`) |
| --- | --- | --- |
| Strength in this standard | **MUST** accept the key on non-idempotent mutations | **SHOULD**, conditioned on "a view over a retained artifact" |
| Carrier | `Idempotency-Key` request header on a repeat of the **same** request | `stream_position` echoed on a **new** request against the same work |
| Identity of the second request | Same method, same URI, same payload fingerprint | Typically a different, safe request against a stored artifact |
| What must be retained | The recorded outcome of the request | The artifact plus the ordering, for a documented window |
| What is delivered | Everything, from the beginning | The tail, from a position |
| Precondition | A completed execution with a recorded outcome | A retained artifact that outlives the connection |
| Failure when the precondition fails | Undefined today — the gap this leaf closes | Defined: reject out-of-window positions with a documented error |
| Retention floor in this standard | **at least 24 hours** (`R3.9`) | API-documented; the shipped exemplar is **~10 minutes** (OpenAI) |

**Where they touch, and where they do not.** `[INFERENCE]` They converge in
exactly one configuration: the execution completed, over a retained artifact,
inside both windows. There, replay is resumption from position zero, and a
server can implement one on top of the other. They diverge everywhere else:

- **Resumption cannot answer replay's question.** Resumption presupposes the
  work exists. If the truncation happened because the *server* died before the
  work reached a terminal state, there is no artifact to resume, and `R13.10`
  by its own terms does not bind (content generated fresh per request with no
  addressable backing store is not "a view over a retained artifact").
- **Replay cannot answer resumption's question.** Replay re-delivers from the
  first frame. A client that consumed 47 of 200 frames and applied them must
  either be able to discard and restart, or must de-duplicate by frame — and
  nothing in `R3.9` gives it a frame identity to de-duplicate on. Only
  `R13.10`'s `stream_position` does, and `R13.10` is a SHOULD that many
  conforming APIs will not implement.
- **The retention floors disagree.** `[FACT] + [INFERENCE]` `R3.9` mandates at
  least 24 hours. The only shipped stream artifact retains for roughly 10
  minutes (OpenAI), and the field's other resumable stream — Kubernetes watch —
  keeps roughly 5 minutes of change history (`survey-08` §4.6). If "replay the
  stored response" is read as "re-stream the recorded frames," `R3.9` silently
  imposes a 24-hour frame-retention obligation that no implementation in the
  field meets, on a payload class an order of magnitude larger than a JSON
  response body. **This is the strongest reason the proposed rule defines the
  stored response as the recorded *outcome*, not the recorded frame sequence.**

**The standards-layer anchor.** `[FACT]` RFC 9110 §9.2.2 sanctions automatic
retry of a *non-idempotent* method only under narrow conditions, and describes
the riskiest common practice this way: "a client might automatically retry a
POST request if the underlying transport connection closed **before any part of
a response is received**, particularly if an idle persistent connection was
used." A truncated stream is by definition the case where part of a response
*was* received. `[INFERENCE]` HTTP's own text therefore excludes the truncated
stream from the one situation in which re-sending a POST is conventionally
tolerated. Recovery from a truncated stream must be **delivery-shaped** —
resume, or replay a recorded outcome — and must not be **execution-shaped**.
RFC 9110 §6.1 supplies the matching definition: "A message is considered
'complete' when all of the octets indicated by its framing are available."

**The structural precedent, and it maps onto this standard exactly.**
`[COMPARATIVE]` gRPC's retry design (A6) defines an RPC as *committed* — no
longer retryable — in two cases, the first being "The client receives
Response-Headers." Its stated reasoning: "The metadata (or its absence) it is
transmitted to the client application. This may fundamentally change the state
of the client, so we cannot safely retry if a failure occurs later in the
RPC's life." That is precisely this standard's **status-committed** boundary
(§1 glossary, `R13.7` versus `R13.8`). `[INFERENCE]` gRPC's answer to "the
stream broke at message 47" is: the transport does not retry; it surfaces the
error to the application. This standard should say the same thing at the HTTP
layer — after status commitment, the client's recovery is never an automatic
re-execution.

**A consortium protocol that agrees, flagged as new evidence.**
`[FACT]` The Model Context Protocol's Streamable HTTP transport (revision
2025-06-18) states that "Disconnection **SHOULD NOT** be interpreted as the
client cancelling its request," that "To avoid message loss due to
disconnection, the server **MAY** make the stream resumable," and — for
resumption — that a server "**MAY** use this header to replay messages that
would have been sent after the last event ID, *on the stream that was
disconnected*" while it "**MUST NOT** replay messages that would have been
delivered on a different stream."

`[INFERENCE]` MCP's recovery is delivery-replay bound to one stream identity;
it never re-issues the JSON-RPC request, because a re-issued request would be a
new request with a new `id` and therefore a new execution. Three independent
designs — gRPC, OpenAI `background: true`, MCP — converge on the same
architecture: **separate the execution from its delivery, then recover the
delivery.**

> **Scope note on a survey negative, recorded rather than resolved.**
> `survey-08` reports a verified negative — no reference in its set emits `id:`
> or honors `Last-Event-ID` — and `baseline-04` declined `Last-Event-ID` partly
> on that ground. MCP is **outside** that reference set (it is a protocol
> specification, not one of the surveyed REST APIs), so it does not falsify the
> survey's claim as scoped. It is nonetheless the first `Last-Event-ID`-honoring
> specification this project has found, and `baseline-04`'s own
> would-invalidate list names exactly that discovery. **This leaf records it as
> new evidence for a future run and does not reopen the ratified `R13.10`
> decision.**

---

## 5. Evidence, separately, for and against a strict rule

### 5.1 FOR a strict rule — replay must not re-execute, and must state what it delivers

1. **Two conforming servers behave incompatibly on a payment endpoint.**
   `[INFERENCE]` `R3.9` says "replay the stored response"; `R12.10` routes a
   truncated-stream client into that path by forbidding key-less replay. With
   "stored response" undefined for a stream, server A treats a stream that
   ended at frame 47 as having no stored response and re-executes; server B
   replays from frame 1. Both satisfy every word of the current text. On a
   disbursement endpoint the difference is one payment or two. Interoperability
   failures that turn on money are the canonical case for a MUST.

2. **The standard already made itself responsible.** `[POLICY]` `R3.9` is a
   MUST across non-idempotent state-changing requests with no streaming
   carve-out, and §13 permits streaming responses without excluding
   non-idempotent methods. Having required both, the standard cannot leave
   their interaction to the reader — that is the argument §13.4 itself makes by
   recording the gap.

3. **The in-flight case has genuine three-source agreement.** `[FACT]` The
   expired IETF draft §2.6: "The request was retried before the original
   request completed. The resource SHOULD respond with a resource conflict
   error," with `409 Conflict` in its error section. Stripe: `409`,
   `idempotency_key_in_use`, result deliberately not saved so the client may
   retry. Shopify: `IDEMPOTENCY_CONCURRENT_REQUEST`. Three independent
   implementations of the same shape is the strongest evidence available on any
   part of this question, and a truncated stream whose work is still running is
   exactly this case.

4. **The field's silence is not a considered rejection.** `[COMPARATIVE]` None
   of the fourteen sources in §3.3 declines to specify streaming replay; every
   one of them simply never built the combination. Silence produced by absence
   is not evidence that specification is unnecessary.

5. **The unmitigated case is a live double-charge.** `[FACT]` AWS Bedrock's
   documented instruction on a mid-stream failure is "Retry your request," on a
   billable inference call with no idempotency token available. `[FACT]`
   Anthropic's documented recovery is a re-prompt that produces a new
   generation. `[INFERENCE]` These are the standard's own hazard, shipped: an
   API that streams a non-idempotent operation and offers no key hands its
   clients a choice between losing the work and paying twice.

6. **HTTP's own text points the same way.** `[FACT]` RFC 9110 §9.2.2 tolerates
   an automatic POST retry only "before any part of a response is received." A
   rule that forbids re-execution once frames have been delivered is a
   codification of that boundary at the application layer, not an invention.

### 5.2 AGAINST a strict rule — the over-specification case

1. **This is a shape the field structurally avoids, and standardizing it may
   bless it.** `[FACT]` Google AIP-151 forbids a streaming response for LRO
   methods; Azure's guidelines never mention streaming and require every
   operation including POST to be idempotent; Stripe, Shopify, Twilio, and
   GitHub simply do not stream mutations. `[INFERENCE]` Four of the eight
   standard references have arranged their designs so this question cannot
   arise. Writing careful replay semantics could legitimize a pattern the most
   experienced API programs decline to ship.

2. **Frame-sequence replay is expensive and possibly unimplementable at the
   stated floor.** `[FACT] + [INFERENCE]` `R3.9`'s ≥24-hour retention against
   OpenAI's ~10 minutes and Kubernetes' ~5 minutes of history is a two-order-of-
   magnitude gap. Requiring 24-hour retention of complete frame sequences for
   every non-idempotent streaming endpoint imposes a storage obligation no
   surveyed implementation carries.

3. **A replayed stream can be a lie, and one vendor says so.** `[FACT]`
   Shopify: "the cached GraphQL response may not be the same as the original
   one, as the cached response is constructed from database records, which may
   have changed." `[INFERENCE]` For a stream, the divergence is worse than for
   a body: a reconstructed frame sequence can differ in ordering, chunking, and
   interleaving while carrying the same final outcome, and a client that
   applied the first 47 frames has no way to tell an equivalent replay from a
   different one.

4. **Zalando, which specified this pattern, warns that correctness is very
   hard.** `[FACT]` Verbatim: the resource and the key cache must be updated
   "with hard transaction semantics — considering all potential pitfalls of
   failures, timeouts, and concurrent requests in a distributed systems. This
   makes a correct implementation exceeding the local context very hard." That
   is for a non-streaming body. `[INFERENCE]` A stream adds a second
   commitment — the frame log — to the same transaction.

5. **The only authority is expired.** `[FACT]` The IETF draft is expired and
   was never an RFC, and even at its most complete it addresses only
   "concurrent" and "completed"; it has nothing on incomplete delivery. `R3.9`
   is already labelled `[POLICY]` with no standards backing. `[INFERENCE]`
   Elaborating a `[POLICY]` rule with more `[POLICY]` sub-rules, in a corner
   with no implementations, increases the standard's conformance surface
   without increasing the evidence under it.

6. **The genuine source conflict on failed originals is unresolved in the
   field.** See §6.4 — Stripe replays recorded errors including `500`s, while
   OASIS directs re-execution when the recorded response was 4xx or 5xx. Two
   respected sources give opposite instructions for the same state.

---

## 6. Proposed rule text

Provisional identifiers **`ST-E01`** and **`ST-E02`**. *Numbering note:
`baseline-04` used the `ST-*` series through `ST-020`, and the sibling Phase-8
leaf `baseline-04c` has independently claimed `ST-021` against the same §13.4
table. To avoid a silent collision between concurrently drafted leaves, this
leaf uses **leaf-scoped** provisional IDs — `E` for `baseline-04e` — and the
ratifying decision record assigns the final `ST-*` numbers.* The proposal has
two rules plus an amendment clause riding `R3.9`.

### 6.1 Amendment riding `R3.9` — define "the stored response"

> **`R3.9`, added clause.** For the purpose of this rule, **the stored
> response** of a request is the outcome the server recorded for that request:
> its status code, its response headers, and — where the response was a
> complete non-streaming body — that body. Where the response was a **stream**
> (§13), the stored response is the stream's **terminal state** together with
> whatever representation of the result the API documents as replayable. An
> API MUST document what a replay of a streaming request delivers.

**Classification:** `[POLICY]`. **Confidence: moderate-high.** The definition
is this standard's choice; it is forced by the retention arithmetic in §4 and
by the fact that no source defines "stored response" for a stream.

### 6.2 New rule `ST-E01` — replay of a streaming request never re-executes

> **`ST-E01`** Where an API accepts an idempotency key (`R3.9`) on a request
> whose success response is a stream (§13), a repeat of that request carrying
> the same key and the same payload fingerprint MUST NOT cause the underlying
> work to be executed a second time. The server MUST answer according to the
> state of the original execution:
>
> 1. **Original still executing.** The server MUST NOT begin a second
>    execution and MUST return a defined conflict error — `409 Conflict`,
>    served as `application/problem+json` per `R5.12` with a `type` and `code`
>    from the `R5.16` catalog. The server MUST NOT record this response as the
>    stored response for the key.
> 2. **Original reached a terminal state** — success or failure, including a
>    stream that ended with an `error` frame (`R13.7`). The server MUST NOT
>    re-execute and MUST deliver the stored response as defined in `R3.9`. It
>    MAY deliver it as a stream replayed from the first frame, or as a
>    non-streamed representation of the same outcome; whichever it does, the
>    API MUST document it, and a replayed response MUST be distinguishable from
>    a first response.
> 3. **Original's execution began and reached no terminal state** — the
>    connection closed and the work was abandoned, cancelled, or lost. The
>    API MUST document its behavior for this case. If it re-executes, it MUST
>    document that it does, because the client cannot otherwise know that its
>    keyed retry can charge twice.
>
> A stream that ends without its terminal frame (`R12.10`) is **not** a
> terminal state and MUST NOT be treated as one.
>
> An API MAY satisfy this rule structurally instead of by replay, by not
> streaming the mutation: see `ST-E02`.

**Classification:** clause 1 is `[COMPARATIVE]` — three independent
implementations (expired IETF draft §2.6, Stripe `409`
`idempotency_key_in_use`, Shopify `IDEMPOTENCY_CONCURRENT_REQUEST`). Clause 2
is `[COMPARATIVE]` on "do not re-execute" (Stripe, Zalando 230, AIP-155, OASIS
all say replay rather than re-run) and `[POLICY]` on permitting a non-streamed
replay. Clause 3 is `[POLICY]` — a documentation duty standing in for a rule
the evidence cannot support.

**Confidence:** high (clause 1), moderate-high (clause 2), moderate
(clause 3).

**Why clause 2 does not mandate frame-for-frame replay.** `[INFERENCE]` Three
reasons, each independently sufficient: the retention arithmetic of §4
(24-hour floor against a 10-minute shipped artifact); Shopify's documented
reconstruction caveat, which shows that a replay is not generally a recording;
and the client's inability to de-duplicate replayed frames without
`stream_position`, which `R13.10` makes optional.

### 6.3 New rule `ST-E02` — the structural alternative, at SHOULD

> **`ST-E02`** An API SHOULD NOT stream the response to a non-idempotent
> mutation whose repeated execution has an external effect that cannot be
> reversed — a payment, a disbursement, a message send, a metered charge.
> Where such a capability needs incremental delivery, the API SHOULD split it:
> the mutation is a non-streaming request that returns an operation resource
> (`R10.9`, `R13.9`), and the incremental delivery is a **safe** request over
> that resource, resumable under `R13.10`. An API that does stream such a
> mutation MUST comply with `ST-E01` and MUST state in its conformance note
> (`R1.7`) which of `ST-E01`'s three cases it implements and how.

**Classification:** `[COMPARATIVE]` on the split — it is OpenAI's shipped
`background: true` design `[FACT]`, it is the architecture gRPC A6 and MCP both
arrive at `[FACT]`, and Google's AIP-151 states the strong form of the same
instinct for LRO methods `[FACT]`. `[POLICY]` on the SHOULD-NOT and on the
scoping to irreversible external effects.

**Confidence: moderate-high.** The construction is shipped and independently
re-derived; the strength (SHOULD, not MUST) is chosen because a flat MUST
would forbid the exact pattern §13 exists to serve (§7.2).

**Why this is the recommended path.** `[INFERENCE]` Under the split, the
recovery request is a `GET`, which RFC 9110 §9.2.2 makes idempotent by method
semantics, so the replay question never arises. `R13.9` already requires one
identity across the two channels and already makes the operation resource
authoritative — the machinery this needs is ratified and present.

### 6.4 The one conflict this leaf must adjudicate rather than average

`[FACT]` **Stripe** replays the recorded result "regardless of whether it
succeeds or fails … including `500` errors," and warns that retrying with a
*fresh* key is inadvisable "because the original key may have produced side
effects." `[FACT]` **OASIS Repeatable Requests** directs the opposite: return
the original response "**or** the response code and response body resulting
from **re-executing** the request if the original response code was 4xx or
5xx."

`[POLICY]` **Stripe should govern in this standard, and `ST-E01` clause 2
follows it.** Three reasons. (a) Scope fit: OASIS's re-execution allowance was
written for repeatable requests with no streaming and no post-commit error
channel; under `R13.7` a failure discovered after status commitment is
delivered **inside a `200`**, so "the original response code was 5xx" does not
even describe the case this leaf governs. (b) Safety asymmetry: re-executing
after a recorded failure is precisely the double-execution `R12.10` exists to
prevent, and Stripe's warning about side effects is the operator experience
behind that. (c) Consistency: `R13.7` already makes a stream-ending `error`
frame a terminal frame and a complete result (`R12.10`), so treating a recorded
failure as replayable rather than re-runnable keeps one definition of
"terminal" across §12, §13, and §3.

---

## 7. Declined alternatives

**7.1 Mandate frame-for-frame replay from frame 1.** Declined. `[INFERENCE]`
It requires retaining complete frame sequences for the `R3.9` floor of at least
24 hours, against a field where the longest shipped stream artifact lives about
10 minutes; and it delivers frames a client may already have applied, with no
mandatory `stream_position` to de-duplicate against. Permitted by `ST-E01`
clause 2, never required.

**7.2 Forbid streaming responses on non-idempotent mutations outright.**
Assessed seriously, declined as a MUST, adopted in weakened form as `ST-E02`.

*The case for it is real.* `[FACT]` AIP-151 states "The response **must not**
be a streaming response." `[FACT]` Azure requires all operations including POST
to be idempotent and never mentions streaming. `[COMPARATIVE]` Four of the
eight standard references avoid the shape entirely. A prohibition would delete
the §13.4 gap rather than fill it, and would need no new machinery.

*Why it is declined.* `[FACT]` AIP-151's prohibition binds **long-running
operation methods**, not all mutations — quoted at its real scope, it is a rule
about what an LRO method returns, and Google's own Gemini product streams
generation endpoints that charge money. `[INFERENCE]` A flat ban would forbid
the dominant shipped pattern of the entire AI-provider segment — a `POST` that
charges and streams — which is the pattern §13 was ratified to serve; a
standard that forbids its own primary use case will be ignored at exactly that
point, and ignoring it takes `R12.10` and `R13.7` down with it. `ST-E02`
therefore takes the prohibition's *substance* (do not stream irreversible
mutations; split them) at SHOULD strength, with an explicit compliance route
for APIs that decline.

**7.3 Rule that `R13.10` resumption satisfies `R3.9` for streams.** Declined.
`[INFERENCE]` `R13.10` is a SHOULD conditioned on a retained artifact, while
`R3.9` is an unconditional MUST; a conditional mechanism cannot discharge an
unconditional obligation. Their preconditions, carriers, and failure modes
differ (§4), and resumption is unavailable in precisely the case that motivates
this leaf — a server that died before the work reached a terminal state has no
artifact to resume.

**7.4 Treat a truncated stream as "no stored response" and permit
re-execution.** Declined. `[INFERENCE]` This is the behavior that makes two
conforming servers differ by one payment, and RFC 9110 §9.2.2 declines to
sanction automatic retry once any part of a response has been received. It
survives only as `ST-E01` clause 3's documented-behavior escape, where it must
be disclosed rather than assumed.

**7.5 Require `Idempotent-Replayed: true` on the wire.** Declined for this
leaf. `[COMPARATIVE]` It is Stripe-only among the references, and this standard
governs reserved header names in §1.10 with an IANA-registry test that a
vendor-invented header does not pass. `ST-E01` clause 2 requires only that a
replay be *distinguishable*, leaving the mechanism to the API — a body member,
a documented header, or the operation resource's own state.

**7.6 Mandate a specific `409` problem `type` URI or `code` string.**
Declined. `[POLICY]` `R5.13` and `R5.16` already bind the `type`/`code`
template and the catalog; naming a literal string here would fork that
machinery for one case.

---

## 8. What could not be verified

1. **The aborted-execution case is undocumented everywhere.** Every source
   found covers "still in progress" (draft-07 §2.6, Stripe `409`, Shopify
   `IDEMPOTENCY_CONCURRENT_REQUEST`) and "completed" (replay the recorded
   result). **No source documents** what a server should do when execution
   began, the connection dropped, the work was abandoned, and no terminal state
   exists. MCP's "Disconnection **SHOULD NOT** be interpreted as the client
   cancelling its request" is the nearest adjacent statement, and it is about
   cancellation semantics, not about idempotency-key state. `ST-E01` clause 3
   is written as a documentation duty for exactly this reason — the rule does
   not depend on resolving a question the field has not answered.

2. **Whether any private or unpublished API combines a key with a stream.**
   The `Both?` column of §3.3 is empty across fourteen published sources; it
   cannot be established that no such deployment exists, only that none is
   documented.

3. **Stripe's behavior on a hypothetical streaming endpoint.** Stripe has none,
   so its "stored response" semantics have never been tested against a partial
   delivery. Any reading of Stripe as authority for streaming replay is
   extrapolation and is labelled as such throughout.

4. **OpenAI's real resumption window.** The vendor states "roughly 10 minutes";
   `survey-08` §11.1 records community reports of a five-minute limit
   ("`starting_after` … no more than 5 minutes old") that the vendor
   documentation does not confirm. Unresolved, and it does not change any
   proposal here — the retention gap argument holds at either figure.

5. **Whether OpenAI, Anthropic, or Google will activate the dormant Stainless
   `idempotencyKey` machinery.** The plumbing exists with no header name
   assigned (`baseline-02g`, 2026-08-09). If any of them switches it on for a
   streaming endpoint, that vendor becomes the field's first data point on this
   question and this leaf should be rerun.

6. **Vendor-documentation absences are bounded by what was fetched.** The
   negatives in §3 rest on the specific pages and machine-readable
   specifications listed in §2, plus `baseline-02g`'s own stated caveat that
   its absences come from published specifications and fetched pages rather
   than exhaustive site crawls.

7. **Twilio's in-flight-duplicate behavior.** Not published on any page found;
   the `409` recorded in `survey-05` for the Monitor Alarms API is a
   resource-limit conflict, not an idempotency-key conflict, and the two should
   not be conflated.

8. **Whether `R12.10`'s replay clause and `ST-E01` should merge.** `R12.10`
   tells the client not to replay without a key; `ST-E01` tells the server what
   a keyed replay must do. They are two halves of one contract in two sections,
   and whether the drafting keeps them apart is a Phase-8 drafting decision this
   report does not make.

---

## 9. Sources

Primary standards and specifications:

- https://www.rfc-editor.org/rfc/rfc9110.txt (RFC 9110, §6.1, §9.2.2)
- https://www.rfc-editor.org/rfc/rfc9457.html (RFC 9457 §3.1)
- https://www.ietf.org/archive/id/draft-ietf-httpapi-idempotency-key-header-07.txt (expired)
- https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/ (status: Expired)
- https://docs.oasis-open.org/odata/repeatable-requests/v1.0/cs01/repeatable-requests-v1.0-cs01.html
- https://modelcontextprotocol.io/specification/2025-06-18/basic/transports
- https://raw.githubusercontent.com/grpc/proposal/master/A6-client-retries.md

Guidelines:

- https://google.aip.dev/151 · https://google.aip.dev/155
- https://raw.githubusercontent.com/microsoft/api-guidelines/vNext/azure/Guidelines.md
- https://raw.githubusercontent.com/zalando/restful-api-guidelines/main/chapters/http-headers.adoc
- https://raw.githubusercontent.com/zalando/restful-api-guidelines/main/chapters/http-requests.adoc

Shipped API documentation:

- https://docs.stripe.com/api/idempotent_requests · https://docs.stripe.com/error-low-level
- https://shopify.dev/docs/api/usage/implementing-idempotency · https://shopify.dev/docs/api/usage/idempotent-requests
- https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_InvokeModelWithResponseStream.html
- https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/
- https://developers.openai.com/api/docs/guides/background
- https://raw.githubusercontent.com/openai/openai-openapi/master/openapi.yaml
- https://platform.claude.com/docs/en/api/errors
- https://generativelanguage.googleapis.com/$discovery/rest?version=v1beta

Prior repo research relied on for previously verified vendor detail:

- `research/reports/baseline-02g-idempotency-key-practice.report.2026-08-09.md`
- `research/reports/survey-05-reliability.report.2026-07-19.md`
- `research/reports/survey-08-streaming.report.2026-08-10.md`
- `research/reports/baseline-04-streaming.report.2026-08-10.md`
- `research/decisions/baseline-04-streaming.decision.md` (ratified; not reopened)
