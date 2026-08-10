# survey-08 — Streaming responses (report, 2026-08-10)

Research leaf under Phase 6. Series `survey` = **descriptive**: this report documents
what the field does today and makes **no** recommendation. It proposes no rule, no
`MUST`/`SHOULD`/`MAY`, and no house policy. The prescriptive leaf
(`baseline-04-streaming`) and the owner's ratification gate decide.

Label key: **[FACT]** primary-sourced · **[COMPARATIVE]** surveyed practice across
references · **[INFERENCE]** reasoning from the above. The `[POLICY]` label used by
`baseline` reports is deliberately absent — this report makes no policy.

Every retrieval in this report was performed on **2026-08-10** unless a different date
is stated on the row.

---

## 1. TL;DR

The field has converged on **one framing** and split on **everything else**.

Server-Sent Events is the dominant framing for incremental *generated* content: all
three deep-dive AI providers emit `text/event-stream`-shaped frames, and so does every
other AI-adjacent surface examined. But `text/event-stream` **is not registered in the
IANA media-type registry** — verified by four independent probes (§3.2) — and its
defining document is a WHATWG Living Standard section, not an RFC. `application/x-ndjson`
is likewise unregistered. The only IANA-registered JSON-stream media type,
`application/json-seq` (RFC 7464, Standards Track), is shipped by **no** reference
surveyed here.

Below that single agreement the field diverges on every axis the ratification gate
will turn on:

- **Negotiation.** All four predicted mechanisms exist and are in live use. Google
  alone uses three of them simultaneously across two API generations (§5.3).
- **Termination.** OpenAI's Chat Completions and Assistants APIs end with the
  `data: [DONE]` sentinel; OpenAI's own **Responses API does not** — it ends with a
  typed `response.completed` event. Gemini's new Interactions API **adopted**
  `data: [DONE]`. Anthropic uses neither, ending on a typed `message_stop`.
- **Post-commit errors.** Every reference emits a private in-band error object. **Zero**
  emit RFC 9457 `application/problem+json` inside a stream. **Zero** use HTTP trailer
  fields. AWS Bedrock carries HTTP status codes (424, 429, 500…) as *data labels inside
  a 200 response*, which is the sharpest illustration of the axis.
- **Resumption.** The expectation that nobody implements it is **wrong**. OpenAI's
  Responses API implements resumption via `GET …?stream=true&starting_after=<n>` over a
  `sequence_number` cursor, and Kubernetes has implemented `resourceVersion` watch
  resumption for a decade. But **no** reference surveyed implements the SSE spec's own
  `id:` / `Last-Event-ID` mechanism — every resumption in the field is a private
  cursor on a private parameter.
- **Chunked framing.** Kubernetes is the only reference whose *documentation* is phrased
  in `Transfer-Encoding: chunked` terms. That phrasing is HTTP/1.1-specific: RFC 9112
  §7.1 defines chunked coding, and HTTP/2 and HTTP/3 have no such coding.

Non-participation is real and large: Stripe, Twilio, Zalando, Google AIP, and the Azure
REST API Guidelines publish **no** HTTP response-streaming surface or guidance at all,
each verified negatively against a primary artifact (§5.9).

---

## 2. Key findings

1. **[FACT] `text/event-stream` is not in the IANA media-type registry.** Four probes,
   all 2026-08-10: (a) `https://www.iana.org/assignments/media-types/text.csv` — 105
   registered `text/*` subtypes, no `event-stream`; (b) the full registry index
   `media-types.xhtml` (1,023,718 bytes) — **zero** occurrences of the string
   `event-stream`; (c) the per-type URL
   `https://www.iana.org/assignments/media-types/text/event-stream` returns **HTTP 404**
   while the control probe `.../text/html` returns **HTTP 200**; (d) the provisional
   registry contains no `event-stream` either. The WHATWG HTML Living Standard's own
   IANA-considerations section carries a registration *template* prefaced "This
   registration is for community review and will be submitted to the IESG for review,
   approval, and registration with IANA." **[INFERENCE]** The registration was drafted
   and, as of the retrieval date, never completed. This is directly comparable to the
   `Operation-Location` finding in `baseline-02i`, where a widely-shipped name failed
   the IANA-registry test.

2. **[FACT] `application/x-ndjson` and `application/jsonl` are also unregistered;
   `application/json-seq` is registered and unused.** Zero hits for `ndjson` across the
   full registry index and the 1,776-entry `application.csv`. The NDJSON specification
   (v1.0.0, last updated 2014-10-19, no standards body) itself says only "The MediaType
   [RFC6838] for Newline Delimited JSON **SHOULD** be *application/x-ndjson*."
   jsonlines.org says the media type "may be `application/jsonl`, but this is not yet
   standardized" and asks for help writing an RFC. Meanwhile `application/json-seq`
   (RFC 7464, **Standards Track**, February 2015, registered 2015-01-08) *is* in the
   registry — and **[COMPARATIVE]** no reference in this survey ships it over HTTP.

3. **[COMPARATIVE] All four negotiation mechanisms are in live use, and Google uses
   three.** Request-body flag (`"stream": true`): OpenAI, Anthropic, Gemini Interactions.
   Distinct method name: Gemini `:streamGenerateContent`, AWS Bedrock
   `/invoke-with-response-stream`. Query parameter: Gemini `alt=sse`, Kubernetes
   `?watch=1`, OpenAI Responses `?stream=true` on the *resume* GET. `Accept` header:
   **no** reference surveyed uses `Accept: text/event-stream` as its primary negotiation
   — the WHATWG spec itself makes it optional ("User agents **may** set (`Accept`,
   `text/event-stream`)"), and AWS Bedrock's `accept:` header selects the *event-stream
   envelope*, not the decision to stream.

4. **[FACT] OpenAI's two current APIs terminate differently.** Chat Completions: "tokens
   will be sent as **data-only** server-sent events … with the stream terminated by a
   `data: [DONE]` message." Responses API: the spec's worked example ends at
   `event: response.completed` with **no** `[DONE]` frame, and the `DoneEvent` schema
   (`event: done` / `data: [DONE]`) is bound to the Assistants `threads/runs` surface.
   **[INFERENCE]** A client written against Chat Completions' terminator will not
   recognise the end of a Responses stream, and vice versa — within one vendor.

5. **[FACT] Resumption exists, but never via `Last-Event-ID`.** OpenAI Responses:
   `GET /v1/responses/{response_id}?stream=true&starting_after=<integer>`, documented as
   "The sequence number of the event after which to start streaming," with every stream
   event carrying a required `sequence_number` field. Kubernetes: `resourceVersion` on
   `?watch=1`, with `410 Gone` when the history window (etcd3 default ~5 minutes) has
   passed. Docker: `since`/`until` timestamps on `/events`. **Verified negative:** no
   `id:` field appears in any Anthropic, OpenAI, or Gemini wire example examined, and no
   reference documents honouring `Last-Event-ID`.

6. **[FACT] Anthropic states the post-commit error problem in its own words.** From the
   errors reference: "When receiving a streaming response over server-sent events (SSE),
   an error can occur after the API returns a 200 response. **In that case, error
   handling doesn't follow these standard mechanisms.**" The in-band shape is a private
   schema — `{"type": "error", "error": {"type": "overloaded_error", "message":
   "Overloaded"}}` — described as one "which would normally correspond to an HTTP 529 in
   a non-streaming context."

7. **[FACT] AWS Bedrock carries HTTP status codes as in-band data.** The response to
   `POST /model/{modelId}/invoke-with-response-stream` is `HTTP/1.1 200` with
   `Content-Type: application/vnd.amazon.eventstream`; the documented response body
   union includes `modelStreamErrorException` (**HTTP Status Code: 424**),
   `throttlingException` (**429**), `modelTimeoutException` (**408**),
   `internalServerException` (**500**), `serviceUnavailableException` (**503**), and
   `validationException` (**400**). **[INFERENCE]** This is the axis in its purest form:
   the status code that *would* have been sent is demoted to a label inside a body whose
   status line already reads 200. `application/vnd.amazon.eventstream` is **not** IANA
   registered (grep of `application.csv`: the only `vnd.amazon.*` entry is
   `vnd.amazon.mobi8-ebook`).

8. **[FACT] Azure OpenAI documents a post-commit content-filter block that keeps the
   200.** Verbatim: "**Status 200 with finish_reason: "content_filter"**: Charged for
   both prompt and completion tokens generated before filtering." Its Asynchronous
   Filter mode adds an in-band annotation channel with `content_filter_offsets`
   (`check_offset`, `start_offset`, `end_offset`) and a stated guarantee: "The content
   filtering signal is guaranteed within a ~1,000-character window of the
   policy-violating content."

9. **[FACT] Only Kubernetes documents streaming in HTTP/1.1 chunked terms.** Its watch
   examples show `200 OK` / `Transfer-Encoding: chunked` / `Content-Type:
   application/json` with a body that is a concatenation of JSON documents. RFC 9112
   §7.1 scopes chunked coding to HTTP/1.1 ("Transfer-Encoding was added in HTTP/1.1");
   HTTP/2 and HTTP/3 have no chunked transfer coding. **[INFERENCE]** A version-neutral
   statement about incremental bodies can rest on the *absence of `Content-Length`* and
   on media-type semantics, but not on `Transfer-Encoding`.

10. **[FACT] Trailer fields are structurally unsuited to carrying a post-commit error,
    by the RFC's own text.** RFC 9110 §6.5.1: "A trailer section is only possible when
    supported by the version of HTTP in use and enabled by an explicit framing
    mechanism"; "in most cases, the trailers are simply discarded"; and decisively
    "**Because of the potential for trailer fields to be discarded in transit, a server
    SHOULD NOT generate trailer fields that it believes are necessary for the user agent
    to receive.**" **[COMPARATIVE]** No reference in this survey uses trailers.

11. **[COMPARATIVE] Two of the three deep-dive providers now dual-type their events.**
    Anthropic: "Each event uses an SSE event name (for example, `event: message_stop`),
    and includes the matching event `type` in its data." OpenAI Responses and Gemini
    Interactions do the same. OpenAI Chat Completions does neither — its frames are
    data-only with the discriminator implicit in `object: "chat.completion.chunk"`.

12. **[FACT] The SSE spec specifies no numeric default retry and no keep-alive
    mechanism, only an authoring note.** Reconnection time "must initially be an
    implementation-defined value, probably in the region of a few seconds." The one
    concrete number in the whole section is advisory: "To protect against such proxy
    servers, authors can include a comment line (one starting with a ':' character)
    **every 15 seconds or so**." **[COMPARATIVE]** Only Anthropic ships a documented
    keep-alive frame (`event: ping`), and it publishes no interval for it.

13. **[FACT] Google AIP-151 forbids streaming for the pattern this standard's `R10.9`
    covers.** Verbatim: "The response type **must** be `google.longrunning.Operation`"
    and "The response **must not** be a streaming response." **[INFERENCE]** Google's
    own guideline draws the streaming/operation-resource boundary as mutually exclusive
    per method, while Google's own Gemini product ships both shapes on different methods.

14. **[COMPARATIVE] `EventSource` is not how anyone authenticates to these APIs.** Every
    deep-dive provider authenticates with a request header (`x-api-key`,
    `Authorization: Bearer <access-token>`, `x-goog-api-key`) that the browser
    `EventSource` constructor cannot set. **[INFERENCE]** Every documented browser path
    is therefore a `fetch`-based reader or a server-side proxy; none of the three
    documents a browser-direct `EventSource` integration, and none documents CORS
    behaviour or `Access-Control-Expose-Headers` for its streaming endpoints.

---

## 3. Standards layer

Read the vendor material in §4 and §5 against this table. Authority class is stated
before content, deliberately.

| Document | Authority class | Status / date | What it governs here |
|---|---|---|---|
| WHATWG HTML Living Standard §9.2, *Server-sent events* | **Living standard** (WHATWG), **not an RFC**, no IETF consensus, continuously revised with no version number | Living; retrieved 2026-08-10 | The `text/event-stream` grammar, UTF-8 requirement, `EventSource` reconnection algorithm, `Last-Event-ID` |
| IANA Media Types registry | **IANA registry** | Index date-stamped to 2026-08-06; retrieved 2026-08-10 | Registration status of `text/event-stream` (**absent**), `application/x-ndjson` (**absent**), `application/json-seq` (**present**) |
| RFC 9112, *HTTP/1.1* | **IETF Internet Standard**, STD 99 | Published June 2022 | Chunked transfer coding; scoped to HTTP/1.1 |
| RFC 9110, *HTTP Semantics* | **IETF Internet Standard**, STD 97 | Published June 2022 | Trailer fields; status-code semantics |
| RFC 7464, *JavaScript Object Notation (JSON) Text Sequences* | **IETF Standards Track** | Published February 2015 | Registers `application/json-seq` |
| NDJSON specification (ndjson/ndjson-spec) | **No standards body** — a GitHub repository | v1.0.0, last update 2014-10-19 | Recommends `application/x-ndjson` |
| jsonlines.org | **No standards body** — a project website | Retrieved 2026-08-10 | Defines JSON Lines; explicitly says no registered media type |
| WHATWG Fetch Standard | **Living standard** (WHATWG) | Living; retrieved 2026-08-10 | CORS-safelisted response headers; `Access-Control-Expose-Headers` |
| `draft-gupta-httpapi-events-query` | **Internet-Draft**, individual submission, not WG-adopted | Published July 2025, **expired 2026-01-05** (unverified — from search results, see §11.1.9) | HTTP Events Query — never a published standard |
| `draft-ietf-alto-incr-update-sse` | **Internet-Draft**, WG document (ALTO), rev 22 | Status and expiry **unverified** — from search results, see §11.1.9 | SSE for ALTO incremental updates; domain-specific, never a published standard |

### 3.1 WHATWG Server-Sent Events — the operative text

Source: `https://html.spec.whatwg.org/multipage/server-sent-events.html`, retrieved
2026-08-10. Authority class: WHATWG Living Standard.

**[FACT] Media type and encoding.** "This event stream format's MIME type is
`text/event-stream`." "Event streams in this format **must always be encoded as
UTF-8**."

**[FACT] The complete field vocabulary.** The interpretation algorithm recognises
exactly four field names; any other field "is ignored".

| Field | Verbatim processing rule |
|---|---|
| `event` | "Set the event type buffer to the field value." |
| `data` | "Append the field value to the data buffer, then append a single U+000A LINE FEED (LF) character to the data buffer." |
| `id` | "If the field value does not contain U+0000 NULL, then set the last event ID buffer to the field value. Otherwise, ignore the field." |
| `retry` | "If the field value consists of only ASCII digits, then interpret the field value as an integer in base ten, and set the event stream's reconnection time to that integer. Otherwise, ignore the field." |

**[FACT] Comments and record separation.** "If the line starts with a U+003A COLON
character (:) — Ignore the line." A blank line dispatches the event. Multi-line `data`
is joined with U+000A, so three `data:` lines yield one payload containing two embedded
newlines (worked example in §10.1).

**[FACT] Reconnection.** The reconnection time "must initially be an
**implementation-defined** value, probably in the region of a few seconds" — **there is
no numeric default in the specification**. On reconnect: "If the `EventSource` object's
last event ID string is not the empty string: … Set (`Last-Event-ID`, lastEventIDValue)
in request's header list."

**[FACT] What stops a client permanently.** "Otherwise, if res's status is not 200, or
if res's `Content-Type` is not `text/event-stream`, then **fail the connection**." And:
"Once the user agent has failed the connection, **it does not attempt to reconnect**."
Separately, "a client can be told to stop reconnecting using the **HTTP 204 No Content**
response code." **[INFERENCE]** A server that responds `202` or `503` to an `EventSource`
request, or that mislabels the media type, terminates the client permanently rather than
triggering a retry — a behaviour that has no analogue in ordinary request/response HTTP.

**[FACT] Accept is optional.** "User agents **may** set (`Accept`,
`text/event-stream`) in request's header list." **[INFERENCE]** A server cannot rely on
the `Accept` header being present to detect an `EventSource` client.

**[FACT] Keep-alive is an authoring note, not a mechanism.** "Legacy proxy servers are
known to, in certain cases, drop HTTP connections after a short timeout. To protect
against such proxy servers, authors can include a comment line (one starting with a ':'
character) every 15 seconds or so."

**[FACT] Credentials.** "Setting `withCredentials` to true will set the credentials mode
for connection requests to url to `include`." **[INFERENCE]** Combined with the absence
of any header-setting affordance on the `EventSource` constructor, the only
`EventSource`-native credential is a cookie.

### 3.2 IANA registrations — probed, not inferred

**[FACT] `text/event-stream` — ABSENT.** Four independent probes on 2026-08-10:

| Probe | URL | Result |
|---|---|---|
| `text/*` subregistry CSV | `https://www.iana.org/assignments/media-types/text.csv` | 105 entries; no `event-stream` row |
| Full registry index | `https://www.iana.org/assignments/media-types/media-types.xhtml` | 1,023,718 bytes; **0** occurrences of `event-stream` |
| Per-type registration page | `https://www.iana.org/assignments/media-types/text/event-stream` | **HTTP 404** |
| Control probe (same URL shape) | `https://www.iana.org/assignments/media-types/text/html` | **HTTP 200** |
| Provisional registry | `https://www.iana.org/assignments/provisional-standard-media-types/provisional-standard-media-types.xhtml` | no `event-stream` |

**[FACT] The WHATWG registration template, verbatim in part** (source:
`https://html.spec.whatwg.org/multipage/iana.html`, retrieved 2026-08-10): Type name
`text`; Subtype name `event-stream`; Required parameters: none; Optional parameters:
`charset` (value must be "utf-8"; "allowed for legacy server compatibility only");
Encoding considerations "8bit (always UTF-8)"; Published specification: "This document is
the relevant specification"; Contact: Ian Hickson; Change controller: **W3C**. The
template is prefaced: "This registration is for community review and will be submitted
to the IESG for review, approval, and registration with IANA."

**[FACT] `application/x-ndjson` and `application/jsonl` — ABSENT.** `application.csv`
(1,776 entries) and the full index both return **0** occurrences of `ndjson`;
`https://www.iana.org/assignments/media-types/application/x-ndjson` returns **HTTP 404**.

**[FACT] `application/json-seq` — PRESENT.** Registry record retrieved 2026-08-10:
reference **RFC 7464**, registered 2015-01-08 (last updated 2015-01-12), registrant
Nicolas Williams, change controller **IETF**, intended usage COMMON. RFC 7464 abstract:
"A JSON text sequence consists of any number of JSON texts, all encoded in UTF-8, each
prefixed by an ASCII Record Separator (0x1E), and each ending with an ASCII Line Feed
character (0x0A)." Named implementations in the registration are command-line tools
(`jq`, `cligj`, `json-text-sequence`), not HTTP APIs.

**[FACT] Related sequence types in the registry:** `application/cbor-seq` (RFC 8742) —
used by Kubernetes for CBOR-encoded watch responses; `application/geo+json-seq`
(RFC 8142); `application/city+json-seq` (OGC).

**[FACT] Vendor stream media types — ABSENT from the registry:**
`application/vnd.amazon.eventstream`, `application/vnd.docker.multiplexed-stream`,
`application/vnd.docker.raw-stream`. Grep of `application.csv` for `vnd.amazon` returns
only `vnd.amazon.mobi8-ebook`; for `vnd.docker`, nothing.

### 3.3 RFC 9112 §7.1 — chunked coding is HTTP/1.1-only

Source: `https://www.rfc-editor.org/rfc/rfc9112.html`, retrieved 2026-08-10. Authority
class: IETF Standards Track (STD 99).

**[FACT]** "The chunked transfer coding wraps content in order to transfer it as a series
of chunks, each with its own size indicator, followed by an OPTIONAL trailer section
containing trailer fields."

**[FACT]** "A sender **MUST NOT** send a Content-Length header field in any message that
contains a Transfer-Encoding header field."

**[FACT]** "Transfer-Encoding was added in HTTP/1.1."

**[INFERENCE]** Three consequences for a version-neutral statement about incremental
bodies. (a) `Transfer-Encoding: chunked` cannot appear in an HTTP/2 or HTTP/3 message at
all, so any guidance phrased in those terms is silently inapplicable over the transports
most of these APIs actually run on. (b) The *observable, version-neutral* property is the
**absence of `Content-Length`** plus a media type whose grammar is self-delimiting —
which is what SSE, NDJSON, and `application/json-seq` each supply in different ways.
(c) Because trailer sections in HTTP/1.1 exist only inside chunked coding, a rule that
depends on trailers inherits the same version-specificity (see §3.4).

### 3.4 RFC 9110 — trailers, and the finality of the status code

Source: raw text at `https://www.rfc-editor.org/rfc/rfc9110.txt` (502,941 bytes),
downloaded and read 2026-08-10. Authority class: IETF Internet Standard (STD 97).

**[FACT] §6.5, the definition:** "Fields (Section 5) that are located within a 'trailer
section' are referred to as 'trailer fields' (or just 'trailers', colloquially). Trailer
fields can be useful for supplying message integrity checks, digital signatures,
delivery metrics, or **post-processing status information**."

**[FACT] §6.5, the constraint:** "Trailer fields ought to be processed and stored
separately from the fields in the header section **to avoid contradicting message
semantics known at the time the header section was complete**."

**[FACT] §6.5.1, the limitations, in full force:**

> A trailer section is only possible when supported by the version of HTTP in use and
> enabled by an explicit framing mechanism. For example, the chunked transfer coding in
> HTTP/1.1 allows a trailer section to be sent after the content …
>
> A sender MUST NOT generate a trailer field unless the sender knows the corresponding
> header field name's definition permits the field to be sent in trailers.
>
> Trailer fields can be difficult to process by intermediaries that forward messages from
> one protocol version to another. If the entire message can be buffered in transit, some
> intermediaries could merge trailer fields into the header section (as appropriate)
> before it is forwarded. **However, in most cases, the trailers are simply discarded.**
>
> Because of the potential for trailer fields to be discarded in transit, **a server
> SHOULD NOT generate trailer fields that it believes are necessary for the user agent to
> receive.**

**[INFERENCE]** The last sentence is the operative one for this survey: the RFC that
defines trailers rules out relying on them for anything the client must see. An error
raised after `200 OK` is by definition something the client must see. This is the
protocol-level explanation for the [COMPARATIVE] observation that no reference uses
trailers for streaming errors — it is not an oversight, it is what the spec advises.

**[FACT] §15 opening:** "The status code of a response is a three-digit integer code that
describes the result of the request and the semantics of the response, including whether
the request was successful and what content is enclosed (if any)."

**[INFERENCE]** RFC 9110 contains no sentence of the form "a sent status code cannot be
revised" — the finality is *structural*, arising from the message grammar in RFC 9112
(status line first, then fields, then content) rather than from a prohibition. Stating
the constraint as "the status line is the first thing on the wire and the message grammar
provides no way to replace it" is defensible from the specs; stating it as "RFC 9110
forbids revising the status" is not, and would repeat the over-reading pattern that
`baseline-02i` had to correct for RFC 6648.

### 3.5 Fetch Standard — what a browser can see

Source: `https://fetch.spec.whatwg.org/`, retrieved 2026-08-10. Authority class: WHATWG
Living Standard.

**[FACT]** The CORS-safelisted response-header names are exactly: `Cache-Control`,
`Content-Language`, `Content-Length`, `Content-Type`, `Expires`, `Last-Modified`,
`Pragma`. **[FACT]** "A response will typically get its CORS-exposed header-name list set
by extracting header values from the `Access-Control-Expose-Headers` header."

**[INFERENCE]** For streaming specifically this bites twice. First, any per-stream
metadata a server puts in a response header (a stream id, a model name, a request id) is
invisible to a cross-origin browser client unless explicitly exposed — which is the same
mechanism `R4.17` already binds for this standard's other headers. Second, `Content-Type`
*is* safelisted, so the media type is always readable; **[INFERENCE]** a client that
distinguishes a stream from a non-stream by media type works cross-origin with no server
opt-in, whereas one that keys off a custom header does not.

**[FACT] Verified negative:** the Fetch Standard text retrieved contains no
streaming-specific CORS carve-out — a `text/event-stream` response is subject to exactly
the same CORS checks as any other response.

---

## 4. Side-by-side comparison tables

Abbreviations: **CC** = OpenAI Chat Completions; **Resp** = OpenAI Responses;
**Asst** = OpenAI Assistants (threads/runs); **GL** = Gemini Generative Language API
(`generateContent` family); **Int** = Gemini Interactions API. All rows verified
2026-08-10 against the source cited in §5.

### 4.1 Negotiation

| Reference | Mechanism | Exact form | Non-streaming default |
|---|---|---|---|
| OpenAI CC | request-body flag | `"stream": true` | one `chat.completion` object |
| OpenAI Resp | request-body flag | `"stream": true`; also `"background": true` for detached runs | one `response` object |
| OpenAI Resp (resume) | **query parameter on GET** | `GET /v1/responses/{id}?stream=true&starting_after=<int>` | the stored `response` object |
| OpenAI Asst | request-body flag | `"stream": true` | run object, client polls |
| Anthropic | request-body flag | `"stream": true` | one `message` object |
| Gemini GL | **distinct method name** plus query parameter | `POST …/models/{model}:streamGenerateContent?alt=sse` | `:generateContent` returns one object; `:streamGenerateContent` **without** `alt=sse` returns a JSON array |
| Gemini Int | request-body flag (query parameter shown inconsistently) | `POST /v1beta/interactions` with `"stream": true`; one docs page adds `?alt=sse`, another omits it | one interaction object |
| AWS Bedrock | **distinct method name** | `POST /model/{modelId}/invoke-with-response-stream` | `POST /model/{modelId}/invoke` |
| Kubernetes | **query parameter** | `GET …/pods?watch=1&resourceVersion=<rv>` | a `PodList` collection |
| Docker | **distinct endpoint** plus query parameters | `GET /events?since=<ts>&until=<ts>` | n/a — `/events` always streams |
| Elasticsearch | n/a for responses | `application/x-ndjson` is a **request-body** media type on `_bulk` | n/a |
| Shopify | n/a — file handoff | GraphQL `bulkOperationRunQuery`, result fetched from a `url` | n/a |
| **`Accept: text/event-stream`** | **used by no reference as primary negotiation** | — | — |

**[FACT]** What happens when streaming is asked for on an endpoint that does not offer it
is documented by **none** of the deep-dive references. See §11.

### 4.2 Framing and media type

| Reference | Response `Content-Type` | Body grammar | `id:` emitted | `retry:` emitted |
|---|---|---|---|---|
| OpenAI CC | not stated in the OpenAPI spec; described as "data-only server-sent events" | SSE, `data:` only, no `event:` | no | no |
| OpenAI Resp | not stated in the OpenAPI spec | SSE with `event:` **and** `data:` | no | no |
| Anthropic | not stated on the streaming or Messages reference page | SSE with `event:` **and** `data:` | no | no |
| Gemini GL (`alt=sse`) | not stated in the reference | SSE `data:` frames | no | no |
| Gemini GL (no `alt=sse`) | not stated | JSON **array** of `GenerateContentResponse` | n/a | n/a |
| Gemini Int | not stated | SSE with `event:` **and** `data:` | no | no |
| AWS Bedrock | **`application/vnd.amazon.eventstream`** (stated: "always set to") | binary framed — 8-byte prelude, prelude CRC, headers, payload, message CRC; 16 bytes overhead | n/a | n/a |
| Kubernetes watch (JSON) | **`application/json`** with `Transfer-Encoding: chunked` | concatenated JSON documents, one per event | n/a (uses `resourceVersion`) | n/a |
| Kubernetes watch (CBOR) | **`application/cbor-seq`** | RFC 8742 CBOR Sequence, one event per entry | n/a | n/a |
| Docker `/events` | **`application/json`** (`produces` in the spec) | stream of JSON objects | n/a (uses `since`) | n/a |
| Docker attach/logs | **`application/vnd.docker.multiplexed-stream`** | 8-byte binary frame header + payload | n/a | n/a |
| Elasticsearch `_bulk` | request side: `application/x-ndjson` **or** `application/json` | NDJSON; "The final line of data must end with a newline character (`\n`)" | n/a | n/a |
| Shopify bulk | file, not a response stream | JSON Lines (JSONL) | n/a | n/a |

**[INFERENCE]** Two labelling patterns coexist and conflict. SSE users label the stream
with a media type that describes the *framing* (`text/event-stream`); Kubernetes and
Docker label a stream of JSON documents `application/json`, which describes the framing of
one *element* and is not a valid parse of the whole body. The second pattern is what
`application/x-ndjson` and `application/json-seq` exist to fix, and neither Kubernetes nor
Docker uses either.

### 4.3 Event typing

| Reference | Typing style | Where the type lives | Full type set |
|---|---|---|---|
| OpenAI CC | **untyped frames, implicit discriminator** | `object: "chat.completion.chunk"` inside the payload | one frame shape |
| OpenAI Resp | **named, dual-carried** | SSE `event:` **and** payload `type` | 55+ variants in `ResponseStreamEvent`; core set `response.created`, `response.in_progress`, `response.queued`, `response.output_item.added`, `response.content_part.added`, `response.output_text.delta`, `response.output_text.done`, `response.content_part.done`, `response.output_item.done`, `response.completed`, `response.failed`, `response.incomplete`, `error` |
| Anthropic | **named, dual-carried** | SSE `event:` **and** payload `type` | `message_start`, `content_block_start`, `content_block_delta`, `content_block_stop`, `message_delta`, `message_stop`, `ping`, `error`. Delta sub-types: `text_delta`, `input_json_delta`, `thinking_delta`, `signature_delta` |
| Gemini Int | **named, dual-carried** | SSE `event:` and payload `event_type` | `interaction.created`, `interaction.status_update`, `step.start`, `step.delta`, `step.stop`, `interaction.completed`, `error`, `done` |
| Gemini GL | **untyped frames** | discriminated positionally by `candidates[].finishReason` and presence of `usageMetadata` | one frame shape |
| AWS Bedrock | **named, out-of-band** | event-stream message headers, not the JSON payload | `chunk` plus six exception names |
| Kubernetes | **named, in-payload** | `"type"` member | `ADDED`, `MODIFIED`, `DELETED`, `BOOKMARK`, `ERROR` |
| Docker `/events` | **in-payload** | `Type` and `Action` members | container/image/volume/network/daemon/service/node/secret/config/builder actions |

**[FACT] Forward-compatibility statements.** Anthropic: "In accordance with the versioning
policy, new event types may be added, and your code should handle unknown event types
gracefully." Kubernetes on `BOOKMARK`: "you shouldn't assume bookmarks are returned at any
specific interval, nor can clients assume that the API server will send any `BOOKMARK`
event even when requested." **[COMPARATIVE]** OpenAI and Gemini publish no equivalent
statement, though OpenAI's spec declares `ResponseStreamEvent` as an open `anyOf` with a
`type` discriminator.

### 4.4 Termination

| Reference | Normal end | Distinguishable from truncation? |
|---|---|---|
| OpenAI CC | **sentinel** `data: [DONE]` | Yes — sentinel absent means truncated |
| OpenAI Asst | **sentinel** `event: done` + `data: [DONE]` | Yes |
| OpenAI Resp | **terminal typed event** `event: response.completed` (also `response.failed`, `response.incomplete`); **no `[DONE]`** | Yes — plus `sequence_number` gaps are detectable |
| Anthropic | **terminal typed event** `event: message_stop` | Yes |
| Gemini Int | **sentinel** `event: done` + `data: [DONE]`, after `interaction.completed` | Yes — two independent signals |
| Gemini GL | **connection close alone** — no sentinel documented; last chunk carries `finishReason` and `usageMetadata` | Weakly — via `finishReason` presence |
| AWS Bedrock | **connection close** after the last `chunk` event | Weakly |
| Kubernetes watch | **never ends normally** — the watch runs until timeout, disconnect, or `410 Gone` | n/a by design |
| Docker `/events` | ends when `until` is reached; otherwise runs indefinitely | n/a |

**[INFERENCE]** The two designs answer "did I get everything?" differently. A sentinel is
a *positive* end-of-stream token that a naive line reader can match without parsing JSON;
a terminal typed event requires the client to know the vocabulary but carries the final
state (`completed` vs `failed` vs `incomplete`) in the same frame. OpenAI Responses and
Gemini Interactions ship examples of each design decision made in opposite directions by
teams that ship the other design elsewhere in the same product.

### 4.5 Post-commit errors — errors raised after `200 OK` is on the wire

| Reference | Mechanism | Shape | RFC 9457 `problem+json`? | Trailers? |
|---|---|---|---|---|
| Anthropic | **named error event** `event: error` | `{"type":"error","error":{"type":"overloaded_error","message":"Overloaded"}}` — the same envelope as its non-streaming errors, minus `request_id` | **No** — private schema | No |
| OpenAI Resp | **named error event** `event: error` | `{"type":"error","code":"…","message":"…","param":null,"sequence_number":N}`; also `response.failed` and `response.incomplete` carry a `response.error` object | **No** — private schema | No |
| OpenAI CC | not documented; **[COMPARATIVE]** community reports of an error object in a data frame | not documented | **No** | No |
| Gemini Int | **named error event** `event: error` | `{"error":{"message":"Deadline expired before operation could complete.","code":"gateway_timeout"},"event_type":"error"}` | **No** — private schema | No |
| Azure OpenAI | **field on a data frame** — the stream keeps its 200 and ends normally | `"finish_reason":"content_filter"` plus `content_filter_results`, then `data: [DONE]` | **No** | No |
| AWS Bedrock | **typed exception members in the response union**, carrying the HTTP status they would have used | `modelStreamErrorException` (424), `throttlingException` (429), `modelTimeoutException` (408), `internalServerException` (500), `serviceUnavailableException` (503), `validationException` (400) | **No** | No |
| Kubernetes | **`ERROR` watch event**, or `410 Gone` **before** the stream starts | `{"type":"ERROR","object":{…Status…}}` | **No** — a `Status` object | No |

**[FACT] Errors before the first byte use a different shape from errors after it, in every
case examined.** Anthropic states this explicitly: "an error can occur after the API
returns a 200 response. In that case, error handling doesn't follow these standard
mechanisms." Kubernetes is the cleanest illustration: a stale `resourceVersion` detected
*before* the stream opens yields a real `410 Gone` status; detected *after* it opens, the
same condition yields an in-band `ERROR` event inside a 200.

**[FACT] Client obligation on a truncated stream.** Anthropic is the only deep-dive
reference that documents one: capture the partial response, then construct a continuation
request — for Claude 4.5 and earlier by prefilling the partial text as an assistant
message, for Claude 4.6 and later by adding a user message that instructs the model to
continue. It also states the limit: "Tool use and extended thinking blocks cannot be
partially recovered. You can resume streaming from the most recent text block." OpenAI's
documented obligation is different in kind — resume the *same* response by sequence
number (§4.6). Gemini documents none.

### 4.6 Resumption and replay

| Reference | Implemented? | Mechanism | Cursor | Guarantee | Window |
|---|---|---|---|---|---|
| SSE spec baseline | defines it | `id:` field, `Last-Event-ID` request header | server-chosen opaque string | none specified | none specified |
| OpenAI Resp | **Yes** | `GET /v1/responses/{id}?stream=true&starting_after=<int>`, with `background: true` for detached runs | `sequence_number` (integer, required on every event) | not stated | "Response data is temporarily stored to disk for roughly 10 minutes" |
| OpenAI CC | **No** | — | — | — | — |
| Anthropic | **No** — documents re-prompting instead | continuation request with the partial text | none | none — the continuation is a **new** generation | n/a |
| Gemini (both APIs) | **No** | — | — | — | — |
| Kubernetes watch | **Yes** — the field's reference implementation | `?watch=1&resourceVersion=<rv>`; `BOOKMARK` events advance the cursor cheaply | `resourceVersion` | at-least-once for changes within the history window; **`410 Gone`** outside it, requiring a full re-list | etcd3 default **~5 minutes** of change history |
| Kubernetes list chunking | **Yes** | `limit` + `continue` | `continue` token | consistent snapshot across chunks | "expire after a short amount of time (**by default 5 minutes**) and return a `410 Gone`" |
| Docker `/events` | **Partially** | `?since=<timestamp>` | wall-clock timestamp | not stated | not stated |
| AWS Bedrock | **No** | — | — | — | — |

**[FACT] Verified negative on `Last-Event-ID`:** no reference in this survey documents
emitting `id:` or honouring `Last-Event-ID`. Every implemented resumption uses a private
parameter and a private cursor.

**[INFERENCE]** The two implemented mechanisms differ in what they resume. OpenAI resumes
*delivery of a stored artifact* — the response is generated once and the stream is a view
over it, which is why `background: true` and a disk-retention window are part of the same
feature. Kubernetes resumes *a position in a change log*, which is why the failure mode is
`410 Gone` rather than a partial replay. **[INFERENCE]** A resumption rule written for one
model does not describe the other.

### 4.7 Final-chunk metadata

| Reference | Metadata | Delivery | Opt-in? |
|---|---|---|---|
| OpenAI CC | token usage for the whole request | "an additional chunk will be streamed before the `data: [DONE]` message. The `usage` field on this chunk shows the token usage statistics for the entire request, and the `choices` field will always be an empty array" | **Yes** — `stream_options.include_usage` |
| OpenAI Resp | full `response` object including `usage` (`input_tokens`, `output_tokens`, `output_tokens_details.reasoning_tokens`, `total_tokens`) | field on the terminal `response.completed` event | No — always present |
| Anthropic | `usage` split across two events | `message_start` carries `input_tokens` and an initial `output_tokens`; `message_delta` carries the running `output_tokens`. **"The token counts shown in the `usage` field of the `message_delta` event are *cumulative*."** Server-tool counts appear as `usage.server_tool_use` | No |
| Gemini Int | `usage` with `total_tokens`, `total_input_tokens`, `input_tokens_by_modality`, `total_cached_tokens`, `total_output_tokens`, `total_tool_use_tokens`, `total_thought_tokens` | field on `interaction.completed` | No |
| Gemini GL | `usageMetadata` | field on the final chunk | No |
| Azure OpenAI | content-filter annotations with character offsets | separate annotation frames interleaved through the stream, with `"choices"` carrying no delta | mode-dependent (Asynchronous Filter) |
| Kubernetes | `resourceVersion` | on every event's object; `BOOKMARK` events carry only `.metadata.resourceVersion` | `allowWatchBookmarks=true` for bookmarks |

**[INFERENCE]** Three shapes are in play — a dedicated extra frame (OpenAI CC), a field on
the terminal event (Responses, Interactions, Gemini GL), and a running total spread across
frames (Anthropic). Only the first is opt-in, and only the first has a documented
interaction with the sentinel ("before the `data: [DONE]` message").

### 4.8 Browser and CORS behaviour

| Reference | Auth credential | Settable by `EventSource`? | Documented browser path | CORS / `Access-Control-Expose-Headers` guidance |
|---|---|---|---|---|
| OpenAI | `Authorization: Bearer <access-token>` header | **No** | JS SDK over `fetch` | none found |
| Anthropic | `x-api-key` header plus `anthropic-version` header | **No** | TypeScript SDK over `fetch` | none found |
| Gemini | `x-goog-api-key` header, or `key=` query parameter on the older surface | header no; query parameter yes | JS SDK | none found |
| AWS Bedrock | SigV4 signature over multiple headers | **No** | SDK only | none found |
| Kubernetes | `Authorization: Bearer <access-token>` header | **No** | not a browser API | none found |

**[FACT] Verified negative:** none of the five references' streaming documentation
mentions `EventSource`, CORS, or `Access-Control-Expose-Headers`. Anthropic and OpenAI
both link to MDN's `EventSource` article as *format* documentation while showing only
`curl` and SDK integrations.

**[INFERENCE]** The practical consequence is uniform across the field: a browser page
cannot call any of these APIs directly with `EventSource`, both because the credential is
a header and because sending a long-lived API key to a browser is a separate problem. The
observed pattern is a server-side relay. **[INFERENCE]** Per-origin connection limits
therefore bind the relay's own origin, not the vendor's: under HTTP/1.1 browsers cap
concurrent connections per origin (commonly 6), so several simultaneous `EventSource`
connections to one origin can starve ordinary requests; under HTTP/2 they multiplex over
one connection and the cap does not apply the same way. **[COMPARATIVE] No reference
surveyed documents this**; it is a browser-platform property, and the survey found no
vendor statement of it to cite.

---

## 5. Per-reference notes

### 5.1 OpenAI — two generations of streaming shipping side by side

Primary sources, all retrieved 2026-08-10:
`https://raw.githubusercontent.com/openai/openai-openapi/master/openapi.yaml` (OpenAI's
own OpenAPI specification, 2,845,739 bytes, downloaded and grepped locally);
`https://developers.openai.com/api/docs/guides/streaming-responses`;
`https://developers.openai.com/api/docs/guides/background`. Note: `platform.openai.com`
301-redirects to `developers.openai.com`; the prior finding in `baseline-02i` that
`platform.openai.com` returns 403 to automated fetch still held for the un-redirected
host.

**Negotiation.** `"stream": true` in the request body, on both Chat Completions and
Responses. Responses additionally accepts `"background": true`, and its *resume* path is a
`GET` with query parameters (below).

**Chat Completions framing.** **[FACT]** The `stream` parameter's own description in the
spec: "Whether to stream back partial progress. If set, tokens will be sent as
**data-only** server-sent events as they become available, with the stream terminated by a
`data: [DONE]` message." Each frame's payload is a `chat.completion.chunk` object; there
is no `event:` line, so the SSE event type is the default `message` for every frame.

**Responses framing.** **[FACT]** Named events with dual carriage. The spec's worked
example runs `event: response.created` → `response.in_progress` →
`response.output_item.added` → `response.content_part.added` →
`response.output_text.delta` (repeated) → `response.output_text.done` →
`response.content_part.done` → `response.output_item.done` → **`event:
response.completed`**, and stops. There is no `[DONE]` frame in that example. The
`DoneEvent` schema that does define `event: done` / `data: [DONE]` ("Occurs when a stream
ends") is used by the Assistants `threads/runs` streams, where the spec's examples end
`event: thread.run.completed` … `event: done` / `data: [DONE]`.

**[FACT] Every Responses stream event carries `sequence_number`**, described as "A
sequence number for this chunk of the stream response" and marked `required` on the event
schemas examined.

**Post-commit errors.** **[FACT]** `ResponseErrorEvent`: `type` is always `error`;
required members `code`, `message`, `param`, `sequence_number`. Spec example:

```json
{ "type": "error", "code": "ERR_SOMETHING", "message": "Something went wrong",
  "param": null, "sequence_number": 1 }
```

Additionally `response.failed` and `response.incomplete` are distinct terminal events
carrying the full `response` object with its own `error` / `incomplete_details` members.
**[INFERENCE]** OpenAI therefore has two post-commit error channels — a frame-level
`error` event for transport-ish failures and a terminal state event for generation-level
failures — and the spec does not state which applies when.

**Resumption.** **[FACT]** From `GET /v1/responses/{response_id}` in the spec: query
parameter `stream` ("If set to true, the model response data will be streamed to the
client as it is generated using server-sent events") and query parameter `starting_after`,
type integer, description "**The sequence number of the event after which to start
streaming.**" From the background-mode guide: "You will want to keep track of a 'cursor'
corresponding to the `sequence_number` you receive in each streaming event," with the
literal resume URL `https://api.openai.com/v1/responses/resp_123?stream=true&starting_after=42`.
The same guide notes "Response data is temporarily stored to disk for roughly 10 minutes
to enable asynchronous execution and polling" and that "SDK support for resuming the
stream is coming soon."

**Streaming-specific security control.** **[FACT]** `ResponseStreamOptions.include_obfuscation`:
"When true, stream obfuscation will be enabled. Stream obfuscation adds random characters
to an `obfuscation` field on streaming delta events to normalize payload sizes as a
mitigation to certain side-channel attacks. These obfuscation fields are **included by
default**, but add a small amount of overhead to the data stream. You can set
`include_obfuscation` to false to optimize for bandwidth if you trust the network links
between your application and the OpenAI API." **[INFERENCE]** This is a threat unique to
incremental delivery — frame sizes leak token lengths to an on-path observer even under
TLS — and OpenAI is the only reference in this survey that addresses it. It is not
mentioned by any standards document examined.

**Confidence: high** on framing, termination, and resumption (vendor's own machine-readable
spec plus a prose guide, two independent artifacts). **Moderate** on Chat Completions
post-commit error shape, which the spec does not define.

### 5.2 Anthropic — named events, explicit about the post-commit problem

Primary sources, retrieved 2026-08-10:
`https://platform.claude.com/docs/en/docs/build-with-claude/streaming`;
`https://platform.claude.com/docs/en/api/errors`;
`https://platform.claude.com/docs/en/api/messages`. (`docs.claude.com` 301-redirects to
`platform.claude.com`.)

**Negotiation.** **[FACT]** "When creating a Message, you can set `"stream": true` to
incrementally stream the response using server-sent events (SSE)." The Messages reference
gives the parameter as "Whether to incrementally stream the response using server-sent
events."

**Framing and typing.** **[FACT]** "Each server-sent event includes a named event type and
associated JSON data. Each event uses an SSE event name (for example, `event:
message_stop`), and includes the matching event `type` in its data." The documented flow:
`message_start` (a `Message` with empty `content`) → per content block
`content_block_start` / one or more `content_block_delta` / `content_block_stop` → one or
more `message_delta` → a final `message_stop`, with `ping` events interspersed.

**[FACT] Delta sub-types:** `text_delta`; `input_json_delta` (whose `partial_json` member
carries *partial JSON strings* that must be accumulated and parsed at
`content_block_stop`); `thinking_delta`; and `signature_delta`, "sent just before the
`content_block_stop` event. This signature is used to verify the integrity of the thinking
block."

**Keep-alive.** **[FACT]** "Event streams may also include any number of `ping` events."
No interval is published. **[INFERENCE]** `ping` is a *named event*, not the SSE spec's
comment-line keep-alive (`: …`), so it reaches an application-level handler rather than
being silently discarded by an SSE parser — a deliberate divergence from the WHATWG
authoring note.

**Post-commit errors.** **[FACT], from the errors reference:** "When receiving a streaming
response over server-sent events (SSE), **an error can occur after the API returns a 200
response. In that case, error handling doesn't follow these standard mechanisms.**" And
from the streaming page: "The API may occasionally send errors in the event stream. For
example, during periods of high usage, you may receive an `overloaded_error`, which would
normally correspond to an HTTP 529 in a non-streaming context," with the wire example
reproduced in §10.2. **[INFERENCE]** The in-band envelope is the vendor's own
non-streaming error envelope minus its `request_id` member — so the shape is consistent
within the vendor and is not RFC 9457.

**Forward compatibility.** **[FACT]** "In accordance with the versioning policy, new event
types may be added, and your code should handle unknown event types gracefully."

**Resumption.** **[FACT] Not implemented.** The docs' "Error recovery" section describes
re-prompting rather than resumption: capture the partial response, construct a
continuation request, resume streaming. The construction differs by model generation — for
Claude 4.5 and earlier the partial text goes back as an assistant message; for Claude 4.6
and later as a user message instructing the model to continue. Stated limit: "Tool use and
extended thinking blocks cannot be partially recovered. You can resume streaming from the
most recent text block." **[INFERENCE]** This is not resumption in the `Last-Event-ID`
sense — it is a fresh generation seeded with the prefix, so byte-for-byte continuity is
not claimed.

**Why streaming is load-bearing here.** **[FACT]** "Consider using the streaming Messages
API or Message Batches API for long-running requests, especially those over 10 minutes."
"The SDKs validate that your non-streaming Messages API requests are not expected to
exceed a 10-minute timeout. They also set a socket option for TCP keep-alive." The errors
page lists `504 - timeout_error` with the advice "Consider using the streaming Messages API
for long-running requests." **[INFERENCE]** Streaming is here functioning as an idle-timeout
mitigation, not only as a latency feature — the same role Kubernetes' watch plays.

**Confidence: high.** Two independent vendor pages (streaming guide and errors reference)
corroborate the event vocabulary, the post-commit statement, and the 10-minute framing.

### 5.3 Google Gemini — three negotiation mechanisms and two source conflicts

Primary sources, retrieved 2026-08-10: the machine-readable discovery document
`https://generativelanguage.googleapis.com/$discovery/rest?version=v1beta` (365,610 bytes,
**revision 20260806**, downloaded and parsed locally); `https://ai.google.dev/api/generate-content`;
`https://ai.google.dev/gemini-api/docs/text-generation`;
`https://ai.google.dev/gemini-api/docs/interactions/streaming`.

**The older surface — Generative Language API.** **[FACT]** The discovery document confirms
`models.streamGenerateContent`, `POST v1beta/{+model}:streamGenerateContent`, described as
"Generates a streamed response from the model given an input `GenerateContentRequest`."
This is a **distinct method name** sitting beside `models.generateContent`. The human
reference adds the query parameter `alt=sse` and shows:

```
POST https://generativelanguage.googleapis.com/v1beta/models/{model}:streamGenerateContent?alt=sse&key=$GEMINI_API_KEY
```

**[FACT] Without `alt=sse`, `:streamGenerateContent` returns a JSON array of
`GenerateContentResponse` objects rather than SSE frames.** **[INFERENCE]** Gemini is
therefore the only reference surveyed where the *same* streaming method serves two
different body grammars selected by a query parameter — a chunked JSON array and an SSE
stream.

**[FACT] Source conflict A — `alt=sse` is not in the machine-readable contract.** The
discovery document's global `alt` parameter declares `enum: ["json", "media", "proto"]`
with `default: "json"` and the enum descriptions "Responses with Content-Type of
application/json", "Media download with context-dependent Content-Type", "Responses with
Content-Type of application/x-protobuf". A grep of the whole 365,610-byte document for the
quoted token `"sse"` returns **zero** hits, and for `event-stream` **zero** hits.
**Which source should govern: the human reference**, because it is the document a client
developer reads, it carries a runnable `curl` example, and the discovery document is
generated from a protobuf service definition that describes the RPC surface rather than the
REST transcoding layer where `alt=sse` is implemented. **[INFERENCE]** The practical
consequence is that any tooling generated from the discovery document — client generators,
API gateways, contract tests — will not know `alt=sse` exists.

**The newer surface — Interactions API.** **[FACT]** A second, newer generation uses a
**request-body flag**: `POST https://generativelanguage.googleapis.com/v1beta/interactions`
with `"stream": true` in the body. Its SSE event vocabulary is `interaction.created`,
`interaction.status_update`, `step.start`, `step.delta`, `step.stop`,
`interaction.completed`, `error`, `done`.

**[FACT] Source conflict B — the `alt=sse` query parameter is shown inconsistently for
Interactions.** The text-generation page shows
`POST "https://generativelanguage.googleapis.com/v1beta/interactions?alt=sse"` with
`"stream": true` in the body; the dedicated streaming page shows
`POST "https://generativelanguage.googleapis.com/v1beta/interactions"` with no query
parameter and the same body flag. **Which source should govern: the dedicated streaming
page**, as the more specific document for this question — but the disagreement is
unresolved and is recorded in §11 rather than smoothed over. **[FACT]** The
`interactions` resource is also absent from the v1beta discovery document, which lists
only `auth_tokens`, `batches`, `cachedContents`, `corpora`, `dynamic`, `environments`,
`fileSearchStores`, `files`, `generatedFiles`, `media`, `models`, `tunedModels`.

**Termination.** **[FACT]** The Interactions stream ends `event: done` / `data: [DONE]`
after `event: interaction.completed`. **[INFERENCE]** Google adopted OpenAI's Chat
Completions sentinel for a brand-new API in the same period that OpenAI dropped it from
*its* newest API — a convergence and a divergence pointing in opposite directions, on the
same axis, in the same year.

**Post-commit errors.** **[FACT]** `event: error` with
`data: {"error":{"message":"Deadline expired before operation could complete.","code":"gateway_timeout"},"event_type":"error"}`.
The `code` member carries a string token, not an integer status.

**Resumption.** **[FACT] Not implemented.** `previous_interaction_id` exists but chains
*conversations*, not interrupted streams.

**Transport hygiene.** **[FACT]** Every Gemini `curl` example passes `--no-buffer`.
**[INFERENCE]** That is client-side; the survey found no Gemini statement about proxy or
CDN buffering, `X-Accel-Buffering`, or keep-alive frames.

**Confidence: high** on the existence of all three negotiation mechanisms (discovery
document plus two human pages). **Moderate** on the exact current status of `alt=sse` for
the Interactions API, because the vendor's own two pages disagree.

### 5.4 Microsoft / Azure — no guideline, a detailed product behaviour

**[FACT] Verified negative on the guideline.** The Azure REST API Guidelines
(`https://raw.githubusercontent.com/microsoft/api-guidelines/vNext/azure/Guidelines.md`,
retrieved 2026-08-10) contain **no** guidance on streaming responses, server-sent events,
`text/event-stream`, NDJSON, long-polling, or incremental delivery. Its long-running-operation
material is entirely poll-based: `202 Accepted`, an `operation-location` header, a status
monitor, and `retry-after`.

**[FACT] The product, by contrast, documents post-commit behaviour in unusual detail.**
Source: `https://learn.microsoft.com/en-us/azure/ai-foundry/openai/concepts/content-streaming`,
page `ms.date` 2026-07-31, updated 2026-08-03, retrieved 2026-08-10.

- *Default mode:* "completion content buffers, the content guardrail system runs on the
  buffered content, and … content is returned to the user if it doesn't violate the
  guardrail policy … or it is immediately blocked and a guardrail error is returned
  instead. … Content isn't returned token-by-token in this case, but in 'content chunks'
  of the respective buffer size."
- *Asynchronous Filter* (API version **2024-02-01 and later**): "content filters run
  asynchronously, and completion content returns immediately with a smooth token-by-token
  streaming experience. No content is buffered."
- *The post-commit signal:* "The content filtering error signal is delayed. If there is a
  policy violation, it's returned as soon as it's available, and the stream stops. **The
  content filtering signal is guaranteed within a ~1,000-character window** of the
  policy-violating content."
- *The status stays 200:* "**Status 200 with `finish_reason: "content_filter"`**: Charged
  for both prompt and completion tokens generated before filtering."
- *In-band annotation channel:* frames whose `choices[].delta` is absent and which instead
  carry `content_filter_results` and `content_filter_offsets` with `check_offset`,
  `start_offset`, `end_offset`. "`check_offset` shows how much text is fully moderated.
  It's an exclusive lower bound on the `end_offset` values of future annotations. It never
  decreases."

**[INFERENCE]** This is the most explicit worked example in the survey of a stream where
the *content already delivered to the user is retroactively invalidated* while the HTTP
status remains 200 — and the vendor's own advice is client-side redaction. Any rule about
post-commit errors that assumes the error arrives before the client has acted on the data
does not cover this case.

**[FACT]** The historical Microsoft REST API Guidelines were not re-examined in this run
beyond what `baseline-02i` recorded; the Azure `vNext` file is the current artifact.

**Confidence: high** on both halves. The guideline negative rests on a full read of the
vendor's own current guideline file, not on a search; the product behaviour rests on a
vendor concept page with an explicit `ms.date` of 2026-07-31 that carries its own worked
wire samples. **Moderate** on whether the Default (buffered) mode ever emits an error frame
rather than terminating with a status — the page describes "a guardrail error is returned"
without showing that frame.

### 5.5 Kubernetes — the field's reference implementation of resumable streaming

Primary source: `https://raw.githubusercontent.com/kubernetes/website/main/content/en/docs/reference/using-api/api-concepts.md`
(79,388 bytes, downloaded and read 2026-08-10), the source of
`https://kubernetes.io/docs/reference/using-api/api-concepts/`.

**[FACT] Negotiation and framing.** "If you sent an HTTP GET request with the `?watch`
query parameter, Kubernetes calls this a **watch** and not a **get**." "Each change
notification is a JSON document. The HTTP response body (served as `application/json`)
consists a series of JSON documents." The worked example (§10.5) shows `200 OK` /
`Transfer-Encoding: chunked` / `Content-Type: application/json`.

**[FACT] CBOR alternative.** "If an API server encodes its response to a watch request
using CBOR, the response body will be a **CBOR Sequence** and the `Content-Type` HTTP
response header will use the IANA media type `application/cbor-seq`. Each entry of the
sequence (if any) is a single CBOR-encoded watch event." **[INFERENCE]** This is the only
instance in the survey of a reference labelling a stream with a *registered* sequence media
type, and it appears on the CBOR path only — the JSON path keeps `application/json`.

**[FACT] Resumption and its failure mode.** "A given Kubernetes server will only preserve a
historical record of changes for a limited time. Clusters using etcd 3 preserve changes in
the last **5 minutes** by default. When the requested watch operations fail because the
historical version of that resource is not available, clients **must handle the case by
recognizing the status code `410 Gone`, clearing their local cache, performing a new get or
list operation, and starting the watch from the `resourceVersion` that was returned**."

**[FACT] Bookmarks.** "the Kubernetes API provides a watch event named `BOOKMARK`. It is a
special kind of event to mark that all changes up to a given `resourceVersion` the client
is requesting have already been sent. The document representing the `BOOKMARK` event is of
the type requested by the request, but only includes a `.metadata.resourceVersion` field."
Requested via `allowWatchBookmarks=true`, with the caveat quoted in §4.3.
**[INFERENCE]** A bookmark is a cursor-advance frame carrying no payload — a design that
lets a client stay resumable during quiet periods without the server replaying data, and
which has no counterpart in any AI-provider stream surveyed.

**[FACT] Streaming lists.** `sendInitialEvents=true` combined with
`resourceVersionMatch=NotOlderThan` starts the watch with synthetic `ADDED` events for the
current state, then a `BOOKMARK` marking the sync point, then the ordinary watch stream —
collapsing list-then-watch into one request.

**[FACT] Chunked collection retrieval (distinct from watch).** `limit` and `continue` query
parameters with a `continue` token in `metadata`; "Like a watch operation, a `continue`
token will expire after a short amount of time (**by default 5 minutes**) and return a
`410 Gone` if more results cannot be returned." `remainingItemCount` gives an estimate.

**[FACT] Compression is documented and orthogonal:** `Accept-Encoding: gzip` on list
responses, with `content-encoding: gzip` in the response.

**Long-poll timeouts.** **[COMPARATIVE, not primary-verified]** Widely-reported behaviour is
that `timeoutSeconds` sets a server-side watch duration and that, when unset, the timeout is
randomised between the API server's `--min-request-timeout` (default 1800 s) and twice that,
to spread reconnection load. **This could not be confirmed in `api-concepts.md`** — the file
contains zero occurrences of `timeoutSeconds`. Recorded as unverified in §11.

**Confidence: high** on the watch wire format, `resourceVersion` semantics, the `410 Gone`
contract, `BOOKMARK` events, the CBOR sequence media type, and the two 5-minute windows —
all read verbatim from the upstream source file of the vendor's own reference page, with the
`410 Gone` behaviour stated twice in that file (once for watch, once for `continue` tokens).
**Low — do not rely on it** for the `timeoutSeconds` row in §6, which failed verification
against that file.

### 5.6 AWS as contrast — a binary framing outside the SSE/NDJSON axis

Primary source: `https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_InvokeModelWithResponseStream.html`,
retrieved 2026-08-10. Corroborating source for the framing: the Amazon Event Stream
specification published under Smithy 2.0 (`https://smithy.io/2.0/aws/amazon-eventstream.html`)
and `https://github.com/awslabs/aws-c-event-stream`, "C99 implementation of the
vnd.amazon.eventstream content-type".

**[FACT] Negotiation is a distinct method name:**
`POST /model/{modelId}/invoke-with-response-stream HTTP/1.1`, beside the non-streaming
`/invoke`. The request sends `accept: application/vnd.amazon.eventstream`.

**[FACT] Response.** "If the action is successful, the service sends back an **HTTP 200**
response." "For streaming, the content type in the response is **always set to
`application/vnd.amazon.eventstream`**. The response includes an additional header
(`x-amzn-bedrock-content-type`), which contains the actual content type of the response."

**[FACT] Framing.** Binary. Each message has an 8-byte prelude — a 4-byte big-endian total
message length and a 4-byte big-endian headers length — followed by a 4-byte prelude CRC,
the headers, the payload, and a 4-byte message CRC; "Total message overhead, including the
prelude and both checksums, is 16 bytes." Event *types* live in the message headers rather
than the payload.

**[FACT] In-band errors carrying HTTP status codes.** The documented response union is
`chunk` plus `internalServerException` (500), `modelStreamErrorException` (424),
`modelTimeoutException` (408), `serviceUnavailableException` (503), `throttlingException`
(429), `validationException` (400).

**[INFERENCE]** AWS is the field's clearest statement that once the status is committed,
the status code becomes *content*. It is also the only reference surveyed that provides
per-message integrity checks (the two CRCs), which makes normal end distinguishable from
corruption independently of any sentinel.

**Confidence: high.** Two independent artifacts agree: the vendor's own API reference (which
states the media type, the 200, and the exception-to-status mapping outright) and the Smithy
2.0 Amazon Event Stream specification plus the `aws-c-event-stream` reference implementation
for the binary framing. The IANA negative for `application/vnd.amazon.eventstream` was
grepped directly from the registry CSV.

### 5.7 Docker — NDJSON-shaped streams labelled `application/json`, plus a binary frame

Primary source: the Docker Engine API v1.51 OpenAPI document
(`https://docs.docker.com/reference/api/engine/version/v1.51.yaml`, 457,984 bytes,
downloaded and read 2026-08-10).

**[FACT] `GET /events`** — "Stream real-time events from the server." `produces:
application/json`. Query parameters `since` ("Show events created since this timestamp then
stream new events"), `until` ("Show events created until this timestamp then stop
streaming"), and `filters`. Responses `200`, `400`, `500` are all declared against a single
schema, so the streaming nature is documented in prose only.

**[FACT] Attach and logs use a private binary framing.** "When the TTY setting is disabled
in `POST /containers/create`, the HTTP `Content-Type` header is set to
`application/vnd.docker.multiplexed-stream` and the stream over the hijacked connected is
multiplexed to separate out `stdout` and `stderr`. The stream consists of a series of
frames, each containing a header and a payload." The header is
`header := [8]byte{STREAM_TYPE, 0, 0, 0, SIZE1, SIZE2, SIZE3, SIZE4}` with `STREAM_TYPE`
0 = `stdin`, 1 = `stdout`, 2 = `stderr`, and the size as a big-endian `uint32`. The
upgraded variant responds `HTTP/1.1 101 UPGRADED` with `Content-Type:
application/vnd.docker.raw-stream`, `Connection: Upgrade`, `Upgrade: tcp`.

**[INFERENCE]** The `101` path leaves HTTP request/response semantics in the same way a
WebSocket upgrade does, and is therefore a boundary case for this survey's scope rather
than an HTTP streaming mechanism.

**Cursor:** `since` is a wall-clock timestamp, so resumption after a disconnect is
best-effort and vulnerable to clock skew and to events sharing a timestamp.
**[INFERENCE]** This is the weakest resumption cursor in the survey — a timestamp is not
monotonic per-event the way `sequence_number` and `resourceVersion` are.

**Confidence: high** on the media types, the frame header layout, and the `/events`
parameters — all read from the vendor's own versioned OpenAPI document rather than from
rendered prose. **Moderate** on `/events` being a genuine stream rather than a bounded
collection, because the specification declares a single non-array response schema and
carries the streaming behaviour only in its prose description.

### 5.8 Elasticsearch and Shopify — NDJSON in the *other* direction, and a file handoff

**[FACT] Elasticsearch `_bulk` is request-side NDJSON, not a response stream.** Source:
`https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-bulk`, retrieved
2026-08-10: "use a `Content-Type` header of `application/json` or
`application/x-ndjson`"; "The final line of data must end with a newline character
(`\n`). Each newline character may be preceded by a carriage return (`\r`)"; and "make
sure that the JSON actions and sources are not pretty printed." **[INFERENCE]** This is
the single most-cited `application/x-ndjson` deployment in the field and it is a *request*
body — evidence that the media type's real-world weight is in upload, not in incremental
response delivery. Note also that the media type is **recommended, not required**:
`application/json` is accepted for the same body.

**[FACT] Shopify bulk operations produce a JSONL file, not a stream.** Source:
`https://shopify.dev/docs/api/usage/bulk-operations/queries`, retrieved 2026-08-10: "we've
chosen the JSON Lines (JSONL) format for the response data." The API call returns operation
metadata; when the operation finishes, "the `url` field will contain a URL where you can
download the data." Completion is signalled by the `bulk_operations/finish` webhook
("Subscribing to the webhook topic is recommended over polling") or by polling
`bulkOperation(id:)`. Ordering is specified: "The order of each connection type is
preserved and all nested connections appear after their parents in the file," with a
`__parentId` member linking children to parents.

**[INFERENCE]** Shopify is out of scope for incremental delivery over one request — it is
the `202`-style operation-resource pattern with a file artifact — but it is the survey's
clearest evidence that JSON Lines is chosen for *bulk result sets* where SSE is chosen for
*incremental generation*. The two format families are not competing for the same job.

**Confidence: high** on both vendors' facts, each read from its own reference page, and on
the classification of both as *not* incremental delivery over one HTTP request.
**Moderate** on the [INFERENCE] that the two format families never compete: it rests on the
absence of a counter-example across thirteen references rather than on any vendor stating
it, and one reference streaming JSON Lines as an HTTP response body would weaken it.

### 5.9 Non-participants — verified negatives, one line and a method each

| Reference | Finding | Verification method, 2026-08-10 |
|---|---|---|
| **Stripe** | No public HTTP response-streaming surface; no SSE, no NDJSON | Stripe's own OpenAPI specification `https://raw.githubusercontent.com/stripe/openapi/master/openapi/spec3.json` (7,967,776 bytes) downloaded and grepped: **0** occurrences of `event-stream`, **0** of `ndjson`. Stripe's event delivery is webhooks plus a polled `/v2/core/events` list |
| **Twilio** | No HTTP response-streaming surface | Twilio Event Streams delivers to outbound sinks (Amazon Kinesis, webhooks), not to an HTTP response body; Twilio Media Streams is WebSocket (§7). No `text/event-stream` surface located |
| **Zalando RESTful API Guidelines** | No streaming guidance | `https://opensource.zalando.com/restful-api-guidelines/` (672,729 bytes) downloaded and grepped: **0** occurrences of `event-stream`, `server-sent`, or `ndjson`; **1** of `streaming`, and not in a rule about HTTP response streaming |
| **Google AIP** | Streaming explicitly excluded from the long-running-operation pattern | AIP-151 (`https://google.aip.dev/151`), verbatim: "The response type **must** be `google.longrunning.Operation`" and "The response **must not** be a streaming response" |
| **Azure REST API Guidelines** | No streaming guidance | Full `vNext/azure/Guidelines.md` examined; no rule on streaming, SSE, NDJSON, long-polling, or incremental delivery |
| **GitHub** | No documented HTTP streaming wire format | `https://docs.github.com/en/copilot/how-tos/copilot-sdk/features/streaming-events` documents ~50 event names as an **SDK subscription model** with no HTTP endpoint, no `text/event-stream`, and no wire-level termination or error semantics. Enterprise "Copilot agent session streaming" (public preview, announced 2026-07-02) pushes audit records to a customer-supplied collector or SIEM — outbound delivery, not an HTTP response stream |

**[INFERENCE]** Five of the eight charter references and two of the four guideline
documents publish nothing on this surface. That is itself the strongest single
[COMPARATIVE] finding in the report: HTTP response streaming has not entered general REST
guidance, and the practice documented in §5.1–§5.7 is almost entirely vendor convention
formed inside one product category.

---

## 6. Long-polling as a distinct mechanism

Long-polling is a different mechanism with a different failure mode: the response is a
single complete message whose *arrival* is deferred, rather than a single message delivered
in pieces. Its characteristic failure is an expired hold returning nothing useful, not a
truncated body.

| Reference | Parameters | Hold duration | What an expired hold returns | Cursor across polls |
|---|---|---|---|---|
| **HashiCorp Consul** blocking queries | `index=<n>`, `wait=<duration>` (e.g. `10s`, `5m`) | **default 5 minutes, maximum 10 minutes**; plus jitter of up to `wait / 16` added "to spread out the wake up time of any concurrent requests" | the current value with the same `X-Consul-Index`; "a blocking request is **no guarantee** of a change" | **`X-Consul-Index` response header**, fed back as `index` |
| **AWS SQS** `ReceiveMessage` (contrast) | `WaitTimeSeconds` on the call, or the `ReceiveMessageWaitTimeSeconds` queue attribute | 0 s = short polling (default); **maximum 20 seconds** | "An empty response is sent only if the polling wait time expires" | receipt handles, not a stream cursor |
| **Kubernetes** watch | `?watch=1`, `resourceVersion`, `timeoutSeconds` | see §5.5 — the documented artifact does not state it | connection close; the client re-watches from its last `resourceVersion` | `resourceVersion` |
| **Docker** `/events` | `since`, `until` | until `until` is reached, else indefinite | n/a | `since` timestamp |

**[FACT] Consul's client obligations, verbatim in substance:** reset the index to `0` if it
ever decreases (which can follow snapshot restores, list operations, or upgrades);
sanity-check that `X-Consul-Index` is at least `1` to prevent a busy loop; and rate-limit
rather than sleep-loop when updates are rapid. **[FACT]** Consul 1.10 added a **streaming
backend** that activates when `index` or `cached` is set and publishes changes as events;
the response header `X-Consul-Query-Backend: streaming` indicates the mode.
**[INFERENCE]** Consul is the survey's one example of a reference that offers long-polling
and event streaming *behind the same request shape*, with the mode reported back in a
header rather than negotiated by the client.

**[FACT] Verified negative:** none of OpenAI, Anthropic, or Gemini documents a long-polling
endpoint. Their non-streaming path is a plain synchronous request, and their deferred path
is a batch job resource.

**[INFERENCE] On the empty-response convention:** the survey found no `204 No Content` used
for an expired hold by any reference. Consul returns a full `200` with the unchanged value;
SQS returns an empty message list inside a `200`. The `204` convention that the WHATWG SSE
spec reserves for "stop reconnecting" is therefore *not* the same `204` an expired long-poll
might use — **[INFERENCE]** a server that served both mechanisms on one path would have
these two meanings collide.

---

## 7. WebSockets instead of HTTP streaming — boundary note

Per the owner's ruling, WebSockets are out of scope. Recorded here only as the boundary
fact: which references route this capability to WebSockets rather than to HTTP streaming.
No WebSocket protocol detail follows.

- **OpenAI Realtime API** — real-time bidirectional audio and text over WebSocket and
  WebRTC, not over SSE. Verified 2026-08-10.
- **Google Gemini Live API** — bidirectional real-time sessions over WebSocket, using the
  `BidiGenerateContent` message pair; documented at `https://ai.google.dev/api/live`.
  Verified 2026-08-10.
- **Twilio Media Streams** — real-time call audio delivered over WebSocket; Twilio's only
  real-time delivery mechanism found. Verified 2026-08-10.
- **Docker** — `/containers/{id}/attach` upgrades via `101 UPGRADED` to a raw or multiplexed
  TCP stream, and a `…/attach/ws` variant exists. **[INFERENCE]** The `101` path leaves HTTP
  request/response semantics whether or not the upgrade target is WebSocket.
- **Kubernetes** — `exec`, `attach`, and `port-forward` use connection upgrade rather than a
  streamed HTTP body; `watch` (the mechanism surveyed here) does not.

**[INFERENCE]** The split is consistent across the field: where the interaction is
*bidirectional and real-time*, references leave HTTP; where it is *unidirectional
server-to-client over one request*, they stay on HTTP and reach for SSE. No reference
surveyed offers WebSockets as an alternative encoding of an otherwise identical
unidirectional streaming capability.

---

## 8. Agreements versus divergences

### 8.1 Where the field agrees

1. **[COMPARATIVE] SSE is the framing for incremental generated content.** All three
   deep-dive providers, plus Azure's hosting of OpenAI, emit SSE-shaped frames. No
   reference uses NDJSON for this job, and no reference uses SSE for bulk result sets.
2. **[COMPARATIVE] The initial status is `200`, always.** No reference uses `202` for a
   stream, and none uses a `1xx` informational response to defer the status.
3. **[COMPARATIVE] Post-commit errors go in the body, never in trailers, and never as
   `problem+json`.** Unanimous across seven references. RFC 9110 §6.5.1 supplies the
   protocol-level reason (§3.4).
4. **[COMPARATIVE] The SSE `id:` / `retry:` fields are unused.** Zero of seven emitters
   use either, despite both being in the framing they have adopted.
5. **[COMPARATIVE] The negotiation is always explicit and always client-driven.** No
   reference streams by default; every one requires the client to ask.

### 8.2 Where it diverges, with the trade-off on both sides

**Sentinel versus terminal typed event.** A sentinel (`data: [DONE]`) is matchable by a
line reader with no schema knowledge and survives an unknown-event-type vocabulary; it also
carries no information, so a client wanting the final state must have kept it from earlier
frames, and the sentinel is not valid JSON, so a naive `JSON.parse` over every `data:` line
throws on the last one. A terminal typed event (`response.completed`, `message_stop`)
carries the final state and parses uniformly; it requires the client to know which event
names are terminal, and a vocabulary that grows (as both OpenAI and Anthropic say theirs
will) can add terminal names a deployed client does not recognise.

**Named `event:` types versus data-only frames with an in-payload discriminator.** Named
types let a browser `EventSource` client register per-type listeners and let an intermediary
route without parsing the body; they also mean the type exists in two places (SSE metadata
and payload) which can disagree, and they are invisible to a `fetch`-based reader that does
not implement SSE parsing. Data-only frames keep exactly one source of truth and work
identically under any reader; they force every client to parse every payload to learn what
it received, and they cannot express a keep-alive or a control frame without inventing a
payload shape for it.

**Private in-band error object versus RFC 9457 `problem+json`.** A private object matches
the vendor's non-streaming error envelope, so one client error type handles both paths
(Anthropic's explicit design); it is not machine-interpretable by a generic client and
carries no registered `type` URI. A `problem+json` object inside a `data:` frame would be
interpretable by generic tooling and would satisfy a `problem+json` obligation formally; it
would be the *only* place in such an API where a media type appears inside a body framed as
a different media type, since the response's `Content-Type` is already `text/event-stream`
— **[INFERENCE]** the `Content-Type` cannot be `application/problem+json` for a frame, so
the media type's own identifying mechanism is unavailable and the shape would travel
unlabelled.

**Resumption by private cursor versus `Last-Event-ID`.** A private cursor
(`starting_after`, `resourceVersion`) can be an integer or a version token with defined
ordering semantics, can be validated server-side, and works over `fetch`; it is invisible to
`EventSource`, which will re-send `Last-Event-ID` and be ignored. `id:` /`Last-Event-ID`
works automatically in every browser with no client code; it is an opaque string with no
ordering semantics, the reconnection interval is implementation-defined, and the mechanism
cannot express "I want the response, not the changes since".

**Keep-alive as a named event versus an SSE comment.** A named `ping` event reaches the
application, which can use it to reset an application-level idle timer and to distinguish
"server alive, still working" from "connection idle"; it also enters the event vocabulary,
so every client must know to ignore it, and it consumes an event-type name. A comment line
(`: keepalive`) is discarded by any conforming SSE parser at no application cost; it is
therefore invisible to the application, which cannot use it, and it is inaccessible to a
`fetch`-based reader that splits on `data:` only.

**Media type that names the framing versus one that names the element.**
`text/event-stream` and `application/x-ndjson` tell a generic client how to find record
boundaries without knowing the schema; the first is unregistered and the second is
unregistered *and* `x-`-prefixed. `application/json` on a stream of concatenated JSON
documents (Kubernetes, Docker) uses only registered names; it is a false statement about
the body — the body is not a JSON text — and a conforming JSON parser fed the whole body
fails.

**Streaming versus an operation resource for the same capability.** **[COMPARATIVE]** Only
OpenAI ships both for one capability, and it relates them precisely: `background: true`
creates a retrievable `response` resource, the `id` addresses it, `GET /v1/responses/{id}`
polls it, and `GET /v1/responses/{id}?stream=true&starting_after=<n>` streams it — one
resource, two access modes, the same terminal state either way. Google states the opposite
rule in AIP-151 (an LRO method's response "must not be a streaming response"), and Gemini
ships the two shapes on different methods with no documented relationship. Anthropic offers
streaming and Message Batches as separate products with no shared identifier.
**[INFERENCE]** The trade-off: unifying them means the stream must be a *view over a stored
artifact*, which forces a retention window (OpenAI's "roughly 10 minutes") and a storage
cost; keeping them separate avoids that but leaves a client that loses a stream with no way
to recover the work already done.

---

## 9. CONTESTED AXES REGISTER (scoped to survey-08)

Each row is self-contained. "How contested" is a measured rating of the field's spread, not
a lean toward any option.

| Axis | Options observed | Who does what | Trade-off in one line | How contested |
|---|---|---|---|---|
| **S1. Negotiation mechanism** | (a) request-body flag `"stream": true`; (b) distinct method or endpoint name; (c) query parameter; (d) `Accept: text/event-stream` | (a) OpenAI CC + Responses, Anthropic, Gemini Interactions; (b) Gemini `:streamGenerateContent`, AWS Bedrock `/invoke-with-response-stream`; (c) Gemini `alt=sse`, Kubernetes `?watch=1`, OpenAI resume `?stream=true`; (d) **nobody** | A body flag keeps one URL and one cache key but makes the response media type depend on the request body, which breaks content negotiation and GET-ability; a distinct name makes the two contracts separately describable in OpenAPI but doubles the surface | **wide-open** — all four in live use, and one vendor uses three |
| **S2. Framing and media type** | (a) `text/event-stream`; (b) `application/x-ndjson`; (c) `application/json` over concatenated documents; (d) `application/json-seq`; (e) vendor binary | (a) all three AI providers + Azure; (b) Elasticsearch, but request-side only; (c) Kubernetes JSON watch, Docker `/events`; (d) **nobody over HTTP**; (e) AWS `vnd.amazon.eventstream`, Docker `vnd.docker.multiplexed-stream` | SSE is what everyone ships and what browsers parse natively, but its media type is unregistered; `application/json-seq` is the only registered option and has zero HTTP adoption; `application/json` on a stream is registered and inaccurate | **split** — near-consensus on SSE for generated content, genuine split for everything else |
| **S3. IANA registration of the chosen media type** | (a) registered; (b) unregistered but universal; (c) unregistered and `x-`-prefixed | (a) `application/json-seq`, `application/cbor-seq`; (b) `text/event-stream`; (c) `application/x-ndjson` | Registration is the test this repository already applied to `Operation-Location`, and the media type the whole field uses fails it; the registered alternative has no adoption to point at | **near-consensus on practice, unresolved on principle** — nobody disputes what is shipped, and the registry disagrees with all of it |
| **S4. Event typing** | (a) named type in SSE `event:` **and** in the payload; (b) named type in the payload only; (c) untyped frames with an implicit discriminator; (d) type in transport headers | (a) OpenAI Responses, Anthropic, Gemini Interactions; (b) Kubernetes (`"type"`), Docker; (c) OpenAI Chat Completions, Gemini `generateContent`; (d) AWS Bedrock | Dual carriage serves both `EventSource` and `fetch` readers at the cost of two places that can disagree; single carriage has one truth and forces every client to parse every frame | **split**, with a clear direction of travel — every newest-generation API surveyed chose (a) |
| **S5. Termination signalling** | (a) sentinel `data: [DONE]`; (b) terminal typed event; (c) both; (d) connection close alone | (a) OpenAI CC, OpenAI Assistants; (b) OpenAI Responses, Anthropic; (c) Gemini Interactions (`interaction.completed` then `done`/`[DONE]`); (d) Gemini `generateContent`, AWS Bedrock | A sentinel needs no schema knowledge but is not valid JSON and carries no state; a terminal event carries the outcome but requires a known vocabulary; close-alone cannot distinguish success from truncation | **wide-open** — one vendor uses both patterns simultaneously in different APIs |
| **S6. Post-commit error shape** | (a) named `error` event with a private object; (b) an error field on an ordinary data frame; (c) typed exception carrying the HTTP status it would have used; (d) RFC 9457 `problem+json` | (a) Anthropic, OpenAI Responses, Gemini Interactions; (b) Azure OpenAI `finish_reason: "content_filter"`; (c) AWS Bedrock; (d) **nobody** | A private object matches the vendor's own non-streaming envelope so one client handler covers both, at the cost of being uninterpretable to generic tooling; `problem+json` inside an SSE frame is interpretable but travels unlabelled, since the response `Content-Type` is already `text/event-stream` | **split on shape, near-consensus on the negative** — every reference uses a private schema and none uses `problem+json` or trailers |
| **S7. Errors before versus after the first byte** | (a) different shapes, stated; (b) different shapes, unstated; (c) same shape | (a) Anthropic states it explicitly; (b) OpenAI, Gemini, AWS in practice; (c) **nobody** | Two error shapes means clients need two handlers and error taxonomies can drift apart; one shape is impossible while the pre-commit path carries a status code and the post-commit path cannot | **near-consensus on the fact, split on whether it is documented** |
| **S8. Resumption obligation** | (a) resumable by private cursor over a stored artifact; (b) resumable by private cursor over a change log; (c) re-prompt / re-request; (d) not addressed | (a) OpenAI Responses (`starting_after` over `sequence_number`, ~10-minute retention); (b) Kubernetes (`resourceVersion`, ~5-minute history, `410 Gone`); (c) Anthropic (continuation request, explicitly not byte-continuous); (d) Gemini, AWS Bedrock, Azure | Resumability requires server-side retention and a stable ordering, and buys a client that survives a dropped connection; no resumption is free at the server and forces the client to discard partial work or re-prompt | **wide-open** — and the field's own framing (`id:` / `Last-Event-ID`) is used by nobody |
| **S9. Keep-alive discipline** | (a) named `ping` event, no interval published; (b) SSE comment line, spec-suggested ~15 s; (c) `BOOKMARK`-style cursor-advance frame; (d) nothing documented | (a) Anthropic; (b) the WHATWG authoring note, shipped by nobody surveyed; (c) Kubernetes, opt-in and explicitly not guaranteed; (d) OpenAI, Gemini, AWS | A named event is usable by the application but must be ignored by every client; a comment is free and invisible; no keep-alive means idle proxies close working streams | **wide-open** — three mechanisms, one per reference, and most references document none |
| **S10. Browser authentication under `EventSource`** | (a) header credential, so `EventSource` is unusable and only `fetch` works; (b) query-parameter token; (c) cookie; (d) not addressed | (a) OpenAI, Anthropic, AWS, Kubernetes; (b) Gemini's older `key=` query parameter; (c) nobody surveyed; (d) all five decline to discuss browser clients at all | A header credential is the only shape that keeps the secret out of URLs and logs and rules out the browser-native API; a query-parameter token restores `EventSource` and leaks into logs, history, and referrers | **near-consensus in practice, unaddressed in documentation** — nobody documents a browser-direct path |
| **S11. CORS exposure for streams** | (a) documented; (b) undocumented | (a) **nobody**; (b) every reference surveyed | Streams carry their discriminating metadata in the body, so the `Content-Type`-only safelist is often sufficient; any per-stream header (request id, model, stream id) is invisible cross-origin without an explicit `Access-Control-Expose-Headers` | **uncontested by silence** — the survey found no reference that addresses it either way |
| **S12. Final-chunk metadata** | (a) dedicated extra frame, opt-in; (b) field on the terminal event, always present; (c) cumulative field spread across frames; (d) interleaved annotation frames | (a) OpenAI CC `stream_options.include_usage`; (b) OpenAI Responses, Gemini Interactions, Gemini `generateContent`; (c) Anthropic (`message_delta.usage`, documented cumulative); (d) Azure content-filter annotations | An extra frame keeps the terminal event's shape stable and adds a frame a client must expect; a field on the terminal event needs no extra parsing and grows that event; a cumulative running field lets a client abort early with usable numbers and invites double-counting | **split** — four shapes across six references |
| **S13. Streaming versus an operation resource for the same capability** | (a) one resource, two access modes, shared identifier; (b) separate methods, no documented relationship; (c) mutually exclusive by rule; (d) separate products | (a) OpenAI Responses with `background: true`; (b) Gemini (`:streamGenerateContent` versus `batchGenerateContent`); (c) Google AIP-151, verbatim "The response must not be a streaming response"; (d) Anthropic (streaming versus Message Batches) | Unifying them lets a client that loses the stream recover the work, and forces server-side retention with a documented expiry; separating them costs nothing at the server and strands a disconnected client | **split**, and Google's own guideline contradicts Google's own product |
| **S14. HTTP-version phrasing of the framing rule** | (a) documented in `Transfer-Encoding: chunked` terms; (b) documented in media-type terms only; (c) not documented | (a) Kubernetes; (b) all SSE emitters; (c) Docker, Gemini | Chunked phrasing is concrete and testable and is silently inapplicable over HTTP/2 and HTTP/3, where no chunked coding exists; media-type phrasing is version-neutral and says nothing about framing guarantees | **near-consensus** — one reference of thirteen uses chunked phrasing, and RFC 9112 confines it to HTTP/1.1 |

---

## 10. EXAMPLES APPENDIX

Verbatim wire excerpts, grouped by reference. Blank-line record separation is
semantically load-bearing in every SSE excerpt and is preserved exactly. Credentials
appear only as the environment-variable placeholders the vendors themselves publish.

### 10.1 WHATWG SSE — multi-line `data` concatenation

Source: `https://html.spec.whatwg.org/multipage/server-sent-events.html`, retrieved
2026-08-10. The spec's own example. "The following event stream, once followed by a blank
line:"

```
data: YHOO
data: +2
data: 10

```

"…would cause an event message with the interface `MessageEvent` to be dispatched on the
`EventSource` object. The event's `data` attribute would contain the string
`"YHOO\n+2\n10"` (where `"\n"` represents a newline)."

Keep-alive comment form implied by the authoring note ("a comment line (one starting with a
':' character) every 15 seconds or so"):

```
: keep-alive

```

Concrete numbers from the specification, with source: reconnection time —
**implementation-defined, no numeric default**, "probably in the region of a few seconds";
keep-alive comment interval — **~15 seconds**, advisory; status that stops reconnection
permanently — **any status other than 200**, plus **`204`** as the explicit "stop
reconnecting" signal.

### 10.2 Anthropic

Request (from the vendor's own `curl` example; the API key is an environment-variable
reference in the original):

```
POST https://api.anthropic.com/v1/messages
anthropic-version: 2023-06-01
content-type: application/json
x-api-key: $ANTHROPIC_API_KEY

{"model":"claude-opus-5","messages":[{"role":"user","content":"Hello"}],"max_tokens":256,"stream":true}
```

Response stream, verbatim and complete:

```
event: message_start
data: {"type": "message_start", "message": {"id": "msg_1nZdL29xx5MUA1yADyHTEsnR8uuvGzszyY", "type": "message", "role": "assistant", "content": [], "model": "claude-opus-5", "stop_reason": null, "stop_sequence": null, "usage": {"input_tokens": 25, "output_tokens": 1}}}

event: content_block_start
data: {"type": "content_block_start", "index": 0, "content_block": {"type": "text", "text": ""}}

event: ping
data: {"type": "ping"}

event: content_block_delta
data: {"type": "content_block_delta", "index": 0, "delta": {"type": "text_delta", "text": "Hello"}}

event: content_block_delta
data: {"type": "content_block_delta", "index": 0, "delta": {"type": "text_delta", "text": "!"}}

event: content_block_stop
data: {"type": "content_block_stop", "index": 0}

event: message_delta
data: {"type": "message_delta", "delta": {"stop_reason": "end_turn", "stop_sequence":null}, "usage": {"output_tokens": 15}}

event: message_stop
data: {"type": "message_stop"}
```

Post-commit error frame, verbatim:

```
event: error
data: {"type": "error", "error": {"type": "overloaded_error", "message": "Overloaded"}}
```

Tool-input partial-JSON delta, verbatim:

```
event: content_block_delta
data: {"type": "content_block_delta","index": 1,"delta": {"type": "input_json_delta","partial_json": "{\"location\": \"San Fra"}}
```

The non-streaming error envelope, for comparison with the in-band shape above:

```json
{
  "type": "error",
  "error": {
    "type": "not_found_error",
    "message": "The requested resource could not be found."
  },
  "request_id": "req_011CSHoEeqs5C35K2UUqR7Fy"
}
```

Concrete numbers: non-streaming request timeout guard — **10 minutes**, SDK-enforced;
Messages API request size limit — **32 MB**; Batch API — **256 MB**; Files API — **500 MB**;
`ping` interval — **not published**; overload status in the non-streaming path — **529**;
streaming-recommended timeout status — **504 `timeout_error`**.

### 10.3 OpenAI

Chat Completions chunk payload (from the vendor's own OpenAPI `x-oaiMeta` example; each
would appear on a `data:` line):

```json
{"id":"chatcmpl-123","object":"chat.completion.chunk","created":1694268190,"model":"gpt-4o-mini","system_fingerprint":"fp_44709d6fcb","choices":[{"index":0,"delta":{"content":"Hello"},"logprobs":null,"finish_reason":null}]}
```

Chat Completions terminator, verbatim from the spec text: `data: [DONE]`. The
`stream_options.include_usage` behaviour, verbatim: "If set, an additional chunk will be
streamed before the `data: [DONE]` message. The `usage` field on this chunk shows the token
usage statistics for the entire request, and the `choices` field will always be an empty
array. All other chunks will also include a `usage` field, but with a null value."

Responses API stream, verbatim (payloads elided in the middle, frame boundaries preserved):

```
event: response.created
data: {"type":"response.created","response":{"id":"resp_67c9fdcecf488190bdd9a0409de3a1ec07b8b0ad4e5eb654","object":"response","created_at":1741290958,"status":"in_progress", ... }}

event: response.in_progress
data: {"type":"response.in_progress","response":{ ... }}

event: response.output_item.added
data: {"type":"response.output_item.added","output_index":0,"item":{"id":"msg_67c9fdcf37fc8190ba82116e33fb28c507b8b0ad4e5eb654","type":"message","status":"in_progress","role":"assistant","content":[]}}

event: response.content_part.added
data: {"type":"response.content_part.added","item_id":"msg_67c9fdcf37fc8190ba82116e33fb28c507b8b0ad4e5eb654","output_index":0,"content_index":0,"part":{"type":"output_text","text":"","annotations":[]}}

event: response.output_text.delta
data: {"type":"response.output_text.delta","item_id":"msg_67c9fdcf37fc8190ba82116e33fb28c507b8b0ad4e5eb654","output_index":0,"content_index":0,"delta":"Hi"}

event: response.output_text.done
data: {"type":"response.output_text.done","item_id":"msg_67c9fdcf37fc8190ba82116e33fb28c507b8b0ad4e5eb654","output_index":0,"content_index":0,"text":"Hi there! How can I assist you today?"}

event: response.content_part.done
data: {"type":"response.content_part.done", ... }

event: response.output_item.done
data: {"type":"response.output_item.done","output_index":0,"item":{ ... "status":"completed" ... }}

event: response.completed
data: {"type":"response.completed","response":{ ... "status":"completed", ... "usage":{"input_tokens":37,"output_tokens":11,"output_tokens_details":{"reasoning_tokens":0},"total_tokens":48} ... }}
```

There is no `data: [DONE]` frame after `response.completed` in the vendor's example.

Responses error event payload, verbatim from the spec's example:

```json
{ "type": "error", "code": "ERR_SOMETHING", "message": "Something went wrong", "param": null, "sequence_number": 1 }
```

Assistants (threads/runs) terminator, verbatim:

```
event: thread.run.completed
data: {"id":"run_123","object":"thread.run", ... "status":"completed", ... }

event: done
data: [DONE]
```

Resume request, verbatim from the background-mode guide:

```
GET https://api.openai.com/v1/responses/resp_123?stream=true&starting_after=42
Authorization: Bearer <access-token>
```

Concrete numbers: `starting_after` — an **integer** matching a `sequence_number`; stored
response retention for resumption — "**roughly 10 minutes**"; obfuscation padding — on by
default via `include_obfuscation`.

### 10.4 Google Gemini

Older surface, verbatim `curl` from the vendor's text-generation guide:

```bash
curl "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:streamGenerateContent?alt=sse&key=$GEMINI_API_KEY" \
    -H 'Content-Type: application/json' \
    --no-buffer \
    -d '{ "contents":[{"parts":[{"text": "Write a story about a magic backpack."}]}]}'
```

Newer surface, verbatim `curl` from the vendor's streaming-interactions page (note the
absence of `alt=sse` here and its presence on the text-generation page — the conflict
recorded in §5.3):

```bash
curl -X POST "https://generativelanguage.googleapis.com/v1beta/interactions" \
      -H "x-goog-api-key: $GEMINI_API_KEY" \
      -H "Content-Type: application/json" \
      --no-buffer \
      -d '{
        "model": "gemini-3.6-flash",
        "input": "Count from 1 to 25.",
        "stream": true
      }'
```

Interactions frame pattern and terminator:

```
event: interaction.created
data: { ... }

event: step.delta
data: { ... }

event: interaction.completed
data: { ... "usage": {"total_tokens": ..., "total_input_tokens": ..., "total_output_tokens": ..., "total_cached_tokens": ..., "total_tool_use_tokens": ..., "total_thought_tokens": ...} ... }

event: done
data: [DONE]
```

Interactions error frame, verbatim:

```
event: error
data: {"error":{"message":"Deadline expired before operation could complete.","code":"gateway_timeout"},"event_type":"error"}
```

Discovery-document evidence for the `alt` conflict, verbatim from
`https://generativelanguage.googleapis.com/$discovery/rest?version=v1beta`
(revision 20260806):

```json
"alt": {
  "description": "Data format for response.",
  "type": "string",
  "enum": ["json", "media", "proto"],
  "enumDescriptions": [
    "Responses with Content-Type of application/json",
    "Media download with context-dependent Content-Type",
    "Responses with Content-Type of application/x-protobuf"
  ],
  "location": "query",
  "default": "json"
}
```

### 10.5 Kubernetes

Watch request and response, verbatim from `api-concepts.md`:

```http
GET /api/v1/namespaces/test/pods?watch=1&resourceVersion=10245
---
200 OK
Transfer-Encoding: chunked
Content-Type: application/json

{
  "type": "ADDED",
  "object": {"kind": "Pod", "apiVersion": "v1", "metadata": {"resourceVersion": "10596", ...}, ...}
}
{
  "type": "MODIFIED",
  "object": {"kind": "Pod", "apiVersion": "v1", "metadata": {"resourceVersion": "11020", ...}, ...}
}
...
```

Bookmark request and event, verbatim:

```http
GET /api/v1/namespaces/test/pods?watch=1&resourceVersion=10245&allowWatchBookmarks=true
---
200 OK
Transfer-Encoding: chunked
Content-Type: application/json

{
  "type": "ADDED",
  "object": {"kind": "Pod", "apiVersion": "v1", "metadata": {"resourceVersion": "10596", ...}, ...}
}
...
{
  "type": "BOOKMARK",
  "object": {"kind": "Pod", "apiVersion": "v1", "metadata": {"resourceVersion": "12746"} }
}
```

Streaming-lists request, verbatim:

```http
GET /api/v1/namespaces/test/pods?watch=1&sendInitialEvents=true&allowWatchBookmarks=true&resourceVersion=&resourceVersionMatch=NotOlderThan
```

Concrete numbers: etcd3 change-history window — **5 minutes** by default; `continue` token
expiry — **5 minutes** by default, then `410 Gone`; etcd watch-cache snapshot retention —
events older than **75 seconds** may be dropped when the cache is full.

### 10.6 AWS Bedrock

Request, verbatim from the API reference:

```
POST https://bedrock-runtime.us-east-1.amazonaws.com/model/amazon.titan-text-express-v1/invoke-with-response-stream

-H accept: application/vnd.amazon.eventstream
-H content-type: application/json
-H x-amzn-bedrock-accept: */*

Payload
{"inputText": "Hello world"}
```

Response headers, verbatim:

```
-H content-type: application/vnd.amazon.eventstream
-H x-amzn-bedrock-content-type: application/json

Payload (stream events)
<response chunk>
```

Concrete numbers: prelude — **8 bytes** (two big-endian `uint32` fields); prelude CRC —
**4 bytes**; message CRC — **4 bytes**; total per-message overhead — **16 bytes**;
maximum request body — **25,000,000 bytes**. In-band exception-to-status mapping:
`validationException` 400, `modelTimeoutException` 408, `modelStreamErrorException` 424,
`throttlingException` 429, `internalServerException` 500, `serviceUnavailableException` 503
— all carried inside a response whose status line reads `HTTP/1.1 200`.

### 10.7 Azure OpenAI

Blocked-stream tail, verbatim from the content-streaming concept page (payloads
abbreviated, frame boundaries preserved):

```
data: {"id":"chatcmpl-8JCbt5d4luUIhYCI7YH4dQK7hnHx2","object":"chat.completion.chunk","created":1699587397,"model":"gpt-35-turbo","choices":[{"index":0,"finish_reason":null,"delta":{"content":" better"}}],"usage":null} 

data: {"id":"","object":"","created":0,"model":"","choices":[{"index":0,"finish_reason":null,"content_filter_results":{...},"content_filter_offsets":{"check_offset":65,"start_offset":65,"end_offset":1056}}],"usage":null} 

data: {"id":"","object":"","created":0,"model":"","choices":[{"index":0,"finish_reason":"content_filter","content_filter_results":{"protected_material_text":{"detected":true,"filtered":true}},"content_filter_offsets":{"check_offset":65,"start_offset":65,"end_offset":1056}}],"usage":null} 

data: [DONE] 
```

Concrete numbers: Asynchronous Filter availability — API version **2024-02-01 and later**;
filtering-signal guarantee — within a **~1,000-character** window; offsets counted from
**0 at the beginning of the prompt**.

### 10.8 Docker

Frame header for `application/vnd.docker.multiplexed-stream`, verbatim from the Engine API
specification:

```go
header := [8]byte{STREAM_TYPE, 0, 0, 0, SIZE1, SIZE2, SIZE3, SIZE4}
```

with `STREAM_TYPE` 0 = `stdin`, 1 = `stdout`, 2 = `stderr`, and `SIZE1..SIZE4` "the four
bytes of the `uint32` size encoded as big endian."

Upgraded raw-stream response, verbatim:

```
HTTP/1.1 101 UPGRADED
Content-Type: application/vnd.docker.raw-stream
Connection: Upgrade
Upgrade: tcp

[STREAM]
```

### 10.9 Long-polling references

Consul blocking query shape (parameters and cursor header as documented):

```
GET /v1/health/service/<service>?index=<X-Consul-Index value>&wait=10s
---
200 OK
X-Consul-Index: <new value>
X-Consul-Query-Backend: blocking-query
```

Concrete numbers: `wait` default **5 minutes**, maximum **10 minutes**, plus jitter of up
to **`wait / 16`**; a decreasing index must be reset to **0**; `X-Consul-Index` must be
sanity-checked to be at least **1**.

AWS SQS long polling: `WaitTimeSeconds` — **0** means short polling (the default),
**maximum 20 seconds**; "An empty response is sent only if the polling wait time expires."

---

## 11. Open questions and caveats

### 11.1 Named unverified items

1. **Response `Content-Type` is undocumented by all three deep-dive providers.** Neither
   OpenAI's OpenAPI specification, Anthropic's streaming or Messages reference pages, nor
   Gemini's reference states the media type of a streaming response. All three describe the
   body as "server-sent events" and link to MDN. **[INFERENCE]** it is `text/event-stream`,
   because that is the only media type an SSE parser accepts and the WHATWG spec makes any
   other media type fatal to an `EventSource` client — but this is an inference, not a
   quoted fact, and only AWS Bedrock ("always set to `application/vnd.amazon.eventstream`")
   and Kubernetes state their streaming media type outright. Confirming it would require a
   live authenticated request, which was out of scope for this run.

2. **What happens when a client asks for streaming on an endpoint that does not offer
   it** is documented by no reference surveyed. Unverified.

3. **Kubernetes `timeoutSeconds` semantics and the randomised default.** The commonly
   repeated behaviour (server-side watch timeout; when unset, randomised between
   `--min-request-timeout`, default 1800 s, and twice that) could **not** be confirmed
   against `api-concepts.md`, which contains zero occurrences of `timeoutSeconds`. The
   number appears here only in §6 as a **[COMPARATIVE, not primary-verified]** row and
   should not be cited at the gate without a fresh check of the Kubernetes API reference.

4. **OpenAI's five-minute resume window.** Community reports describe a
   `BadRequestError` reading "This response can no longer be streamed because it is more
   than 5 minutes old." The vendor's own background-mode guide states only "Response data
   is temporarily stored to disk for **roughly 10 minutes**." The two numbers describe
   different things and the shorter one is **unverified against a vendor source**. Only the
   10-minute figure is used in this report.

5. **OpenAI Chat Completions post-commit error shape** is not defined in OpenAI's OpenAPI
   specification. The Responses API's `ResponseErrorEvent` is defined; the Chat Completions
   equivalent is not. Unverified.

6. **`X-Accel-Buffering: no`.** The header is documented by **nginx** (buffering "can also
   be enabled or disabled by passing 'yes' or 'no' in the 'X-Accel-Buffering' response
   header field") and is widely used for SSE, but **no reference API in this survey
   documents emitting it**, and it is not in the IANA HTTP Field Name Registry. Recorded
   as infrastructure practice with no vendor-documented adoption among the references.

7. **Compression interaction with streaming.** Only Kubernetes documents response
   compression at all (`Accept-Encoding: gzip` on list responses), and it does so for
   collection reads, not for watch. No reference states whether its streaming responses are
   compressed. Unverified.

8. **Twilio and Stripe negatives rest on different evidence strengths.** Stripe's negative
   is strong: its own machine-readable OpenAPI specification was downloaded and grepped.
   Twilio's negative rests on documentation review and search, not on a machine-readable
   artifact, because Twilio publishes per-product OpenAPI specs rather than one. Twilio's
   row is therefore **weaker evidence** than Stripe's and is labelled accordingly in §5.9.

9. **Internet-Drafts on this surface were surveyed shallowly.** Two were located —
   `draft-gupta-httpapi-events-query` (individual submission, published July 2025, expiry
   2026-01-05, therefore **expired** as of the retrieval date unless revised) and
   `draft-ietf-alto-incr-update-sse` (an ALTO working-group document using SSE for a
   domain-specific purpose). Neither datatracker entry was fetched directly in this run, so
   **the status, revision, and expiry of both are recorded from search results and are
   unverified**. Neither may be cited as a published standard, and the httpapi working
   group's current docket was not enumerated. A targeted datatracker sweep is the obvious
   follow-up if the gate needs standards-track trajectory on this surface.

### 11.2 Source conflicts surfaced, not averaged

**Conflict A — Gemini `alt=sse` exists in the human documentation and not in the
machine-readable discovery document.** Evidence in §5.3. **Which should govern: the human
reference**, because it carries a runnable example and the discovery document describes the
RPC surface rather than the REST transcoding layer where `alt=sse` lives. **[INFERENCE]**
The unresolved consequence is that code generators and gateways built from the discovery
document cannot know the parameter exists — a real interoperability gap, not a
documentation nit.

**Conflict B — Gemini's own two pages disagree on whether `alt=sse` is needed for the
Interactions API.** Evidence in §5.3. **Which should govern: the dedicated streaming page**,
as the more specific document. The disagreement is unresolved.

**Conflict C — the IANA registry disagrees with universal practice on
`text/event-stream`.** Evidence in §3.2. **Which should govern for the question "is this
media type registered": the registry, unambiguously** — it is the authority for that
question and the answer is no. The registry does **not** govern the separate question of
what the field ships, which is equally unambiguously `text/event-stream`. These are two
different questions and the survey records both answers rather than reconciling them.

**Conflict D — Google's guideline contradicts Google's product.** AIP-151: an LRO method's
response "must not be a streaming response." Gemini ships `:streamGenerateContent` and
`batchGenerateContent` as separate methods, which is technically compliant, while OpenAI
unifies the two access modes on one resource. **Which should govern: neither for this
standard's purposes** — AIP-151 governs Google's own protobuf-first API surface and its
rule is about *one method's response type*, not about whether a product may offer both
shapes. Recorded as a divergence in S13, not as a rule.

### 11.3 What would change these findings

1. **Completion of the `text/event-stream` IANA registration.** The WHATWG template has
   sat un-submitted; if it were registered, finding 1 and axis S3 would change materially,
   and the parallel with `Operation-Location` would dissolve.
2. **Any reference shipping `application/json-seq` over HTTP.** It would convert the
   registered option from theoretical to demonstrated and change axis S2's shape.
3. **Any reference emitting `id:` and honouring `Last-Event-ID`.** It would move axis S8
   from "the spec's own mechanism is unused by everyone" to a genuine two-sided split.
4. **Anthropic or Gemini shipping cursor-based resumption.** OpenAI's `starting_after` is
   currently a single data point among the three deep-dive providers; a second would make
   it a pattern.
5. **A vendor emitting `application/problem+json` inside a stream frame.** Axis S6's
   unanimous negative is the strongest single finding in this report and rests on seven
   references; one counter-example would weaken it, though the labelling problem noted in
   §8.2 would remain.

**Non-invalidating:** further AI-provider examples of SSE. The SSE-for-generated-content
finding is already unanimous across every reference that streams generated content, and no
additional vendor changes it.

### 11.4 Source inventory

Distinct primary and corroborating sources cited in this report, all retrieved
2026-08-10: **41**. Of these, **11** are standards or registry artifacts (WHATWG HTML SSE,
WHATWG HTML IANA-considerations, WHATWG Fetch, IANA `text.csv`, IANA `application.csv`,
IANA media-types index, IANA per-type probes, IANA provisional registry, RFC 9110, RFC 9112,
RFC 7464); **9** are machine-readable vendor artifacts downloaded and inspected locally
(OpenAI OpenAPI specification, Gemini v1beta discovery document, Stripe OpenAPI
specification, Docker Engine API v1.51 specification, Kubernetes `api-concepts.md` source,
Zalando guidelines, NDJSON specification, Azure REST API Guidelines, jsonlines.org); the
remainder are vendor reference pages. Six searches were used only to locate primary sources
and none is cited as evidence for a load-bearing claim.
