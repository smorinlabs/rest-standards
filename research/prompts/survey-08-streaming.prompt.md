# Deep Research Prompt — REST API Conventions Series (Part 8): Streaming responses

> A **descriptive** research part feeding a follow-up prescriptive leaf
> (`baseline-04-streaming`) and then a ratification gate. Part 8 extends the
> seven-part series charter in `survey-00-series.framing.md`; the scope,
> reference set, and boundaries specific to this part are in
> `survey-08-streaming.framing.md`, which this prompt assumes. This part covers
> ONLY incremental delivery of a response over a single HTTP request —
> asynchronous job resources are Part 5, outbound webhooks are Part 7 — so do
> not expand scope.

## Scope line

**The exact question:** When an API delivers a response incrementally over one
HTTP request — Server-Sent Events, long-polling, or a streaming HTTP body
(chunked NDJSON/JSON Lines) — what does each reference actually do, and on
which axes does the field genuinely split?

## Mandate

- **DESCRIPTIVE ONLY.** Document what each reference does today. No "the
  standard should…" statements, no proposed rules, no `MUST`/`SHOULD`
  recommendations of your own. A survey report that recommends a rule has
  exceeded its mandate. Flag conflicts and trade-offs descriptively; the
  prescriptive leaf and the owner's ratification gate decide.
- **Treat the three AI providers as the deep-dive targets** — OpenAI,
  Anthropic, Google Gemini — because they are where streaming is the default
  API shape, not an accessory. Establish precisely what each does and how the
  rest of the field compares.
- **Non-participation is a finding.** Where a charter reference publishes no
  streaming surface, say so in one line with a verification date rather than
  omitting it.
- **Decision-readiness is the quality bar:** capture each divergence precisely
  enough to decide from without re-research.

## WebSockets are out of scope

The owner ruled WebSockets a non-goal for this phase: after the `101` upgrade
the exchange is no longer HTTP request/response. Do not survey WebSocket APIs.
Do record, in one short subsection, **which references offer WebSockets
*instead of* HTTP streaming for the same capability**, since that is a boundary
fact the standard will state.

## Surface to research

1. **Negotiation** — how a client asks for incremental delivery, verbatim per
   reference. Expect at least four distinct mechanisms and confirm which each
   uses: a request-body flag (`"stream": true`); an `Accept: text/event-stream`
   request header; a query parameter (Gemini's `alt=sse` — verify); a distinct
   endpoint or method name (`streamGenerateContent` — verify). Record what
   happens when a client asks for streaming on an endpoint that does not offer
   it, and what the non-streaming default returns.
2. **Framing and media type** — the exact response `Content-Type` per
   reference; whether the body is SSE, NDJSON/JSON Lines, a chunked JSON array,
   or a vendor encoding (AWS `application/vnd.amazon.eventstream` — verify).
   For SSE, the field grammar actually emitted: `event:`, `data:`, `id:`,
   `retry:`, comment lines, blank-line record separation, multi-line `data`
   handling. For NDJSON, the exact media-type string and delimiter discipline.
3. **Event typing** — named event types (Anthropic's `message_start`,
   `content_block_delta`, `message_delta`, `message_stop`, `ping` — verify the
   full set) versus untyped `data:`-only frames with a discriminator inside the
   JSON, versus positional/implicit typing. Whether the type appears in the SSE
   `event:` field, inside the payload, or both.
4. **Termination signaling** — how a client knows the stream ended normally: a
   sentinel frame (OpenAI's `data: [DONE]` — verify current status across the
   Chat Completions and Responses APIs), a terminal named event, or connection
   close alone. Whether a normal end is distinguishable from a truncated
   connection, and how.
5. **Errors after the status is committed** — the sharpest expected axis. Once
   `200 OK` and its headers are sent the status cannot be revised. Per
   reference: an error event in-band, an error object inside a data frame,
   HTTP trailer fields, abrupt close, or a partial-result marker. Record
   whether the in-band error shape resembles RFC 9457 `problem+json` or a
   private schema, and whether errors *before* the first byte use a different
   shape from errors after it. Also record what each says about the client's
   obligation on a truncated stream.
6. **Resumption and replay** — whether `id:` is emitted and `Last-Event-ID`
   honored on reconnect; any cursor, sequence-number, or offset mechanism;
   what is guaranteed on resume (exactly-once, at-least-once, nothing);
   Kubernetes `resourceVersion` watch semantics as the reference
   implementation. Explicitly record which of the AI providers implement
   resumption at all, since the expectation is that most do not — verify
   rather than assume.
7. **Metadata in the final chunk** — token usage, totals, cursors, or billing
   data delivered at stream end. OpenAI's `stream_options.include_usage` and
   Anthropic's usage in `message_delta` (verify both); whether metadata is a
   separate frame, a field on a terminal event, or absent.
8. **Long-polling as a distinct mechanism** — which references offer it, hold
   durations and their limits, what an expired hold returns (empty `200`, `204`,
   a timeout marker), how the client carries a cursor across polls, and the
   documented relationship to the same API's streaming or webhook channels.
9. **Transport and infrastructure behavior** — keep-alive frames or SSE comment
   pings and their documented interval; documented idle timeouts; proxy and
   CDN buffering (including any documented `X-Accel-Buffering: no` guidance);
   whether responses are compressed; `Content-Length` absence; whether guidance
   is phrased in HTTP/1.1 chunked terms and, if so, what it says about
   HTTP/2 and HTTP/3.
10. **Browser-client constraints** — `EventSource` versus a `fetch`-based
    reader; the `EventSource` inability to set request headers (including
    `Authorization`) and how each reference tells browser clients to
    authenticate; CORS behavior for streaming responses, including which
    response headers are exposed (`Access-Control-Expose-Headers`); per-origin
    connection limits under HTTP/1.1 versus HTTP/2.
11. **Boundary with asynchronous job resources** — where a reference offers
    *both* streaming and a `202`-style operation resource for the same
    capability, document how they relate: does the stream carry the operation
    identifier, do both channels report the same terminal state, and is one
    documented as primary. Document this as observed practice; do **not**
    recommend how they should relate.

## Standards layer to establish (authority class matters)

For each item, state what the document actually says, its **authority class**
(published standard / living standard / expired draft / vendor convention /
no standards body), its publication or access date, and its registration
status where applicable.

- **WHATWG HTML Living Standard, Server-Sent Events** — the field grammar,
  UTF-8 requirement, comment lines, the `EventSource` reconnection algorithm
  and default retry, `Last-Event-ID` semantics. Note explicitly that this is a
  living standard, not an RFC.
- **`text/event-stream`** — its IANA media-type registry entry, checked in the
  registry itself rather than inferred from the WHATWG text.
- **RFC 9112 §7** — chunked transfer coding, and the fact that HTTP/2 and
  HTTP/3 do not use it. Establish what a version-neutral statement about
  incremental bodies can actually say.
- **RFC 9110** — trailer-field semantics and the constraint that a sent status
  code is final; quote the operative sentences from raw RFC text.
- **NDJSON / JSON Lines** — whether any standards body defines it; the IANA
  registration status of `application/x-ndjson` and of any `application/jsonl`
  or `application/json-seq` alternative (RFC 7464 is a published RFC — verify
  what it defines and whether anyone ships it).
- **Fetch Standard** — CORS handling of streaming responses and header
  exposure.
- Any Internet-Draft touching this surface: record draft number, status, and
  expiry date, and never present it as a published standard.

## Out of scope (entire series)

OAuth/OIDC internals; GraphQL/gRPC; gateway and infrastructure product
selection; SDK design; event-streaming platforms as such (Kafka, Pub/Sub, SNS,
EventBridge appear only as contrast where a reference offers one instead of
HTTP streaming). Plus, for this part: WebSockets, per the ruling above.

## Quality bar

- Primary sources first (official API reference documentation, the WHATWG and
  IETF texts, IANA registries); secondary only as corroboration.
- **Two-source minimum on every load-bearing claim.**
- **Currency:** record retrieval dates. Streaming surfaces change fast — an
  undated vendor claim is not usable at the ratification gate.
- **Confidence** per non-obvious finding, with its basis.
- **Surface disagreements** between sources rather than silently picking one;
  state which source should govern and why.
- Label every claim `[FACT]`, `[COMPARATIVE]`, or `[INFERENCE]`. Do not emit
  `[POLICY]` labels — this report makes no policy.

## Specification-grade detail requirement

A finding is complete only when someone could implement or emulate the
mechanism from this report **without opening the reference's own docs**. For
every mechanism documented:

- **Exact names and formats** — media types, header names with value syntax,
  field and parameter names verbatim, event-type strings, sentinel values.
- **Verbatim examples** — at least one real wire example per major mechanism: a
  request line plus relevant headers, and a captured stream excerpt showing
  frame boundaries (minimally elided for length). Show blank-line separation
  explicitly where it is semantically load-bearing.
- **Concrete numbers** — hold durations, keep-alive intervals, timeouts,
  retry defaults, connection limits — each with its source and retrieval date.

Summaries without these artifacts do not satisfy the deliverable.

## Required deliverable structure

1. **TL;DR**
2. **Key findings** (numbered)
3. **Standards layer** — the authority-class table described above, before any
   vendor material, so that vendor practice is read against it
4. **Side-by-side comparison tables** — negotiation; framing and media type;
   event typing; termination; post-commit errors; resumption; final-chunk
   metadata; browser/CORS behavior
5. **Per-reference notes** — a streaming character sketch of each, with OpenAI,
   Anthropic, and Gemini as their own subsections, and a short
   non-participants subsection (one line and a verification date each)
6. **Long-polling subsection** — treated separately, since it is a different
   mechanism with a different failure mode
7. **WebSockets-instead-of-streaming boundary note** — which references route
   this capability to WebSockets, one line each; no WebSocket protocol detail
8. **Agreements versus divergences** — each divergence with its trade-offs,
   descriptively
9. **CONTESTED AXES REGISTER (scoped to this part)** — one row per contested
   axis. Expect at least: negotiation mechanism; framing/media type; event
   typing; termination signaling; post-commit error shape; resumption
   obligation; keep-alive discipline; browser authentication under
   `EventSource`; final-chunk metadata; the streaming-versus-operation-resource
   relationship. Columns: **Axis · Options observed · Who does what ·
   Trade-off in one line · How contested** (near-consensus / split /
   wide-open). Each row must be self-contained enough to become a decision item
   directly.
10. **EXAMPLES APPENDIX** — every verbatim wire excerpt, header line, and
    concrete number collected under the specification-grade requirement,
    grouped by reference
11. **Open questions and caveats** — including anything you could not verify,
    named as unverified rather than smoothed over

## Credential hygiene

This repository is public and has GitHub push protection enabled. Never
reproduce a real or documentation-example API key, token, or secret, even one
published by a vendor. Use placeholder forms only — `Bearer <access-token>`,
`sk_test_<redacted>`. A prior report required redaction for exactly this
reason.
