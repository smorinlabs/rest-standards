# Framing: Streaming responses (Phase 6)

Date framed: 2026-08-10

This document frames **both** Phase 6 research leaves — the descriptive
`survey-08-streaming` and the prescriptive `baseline-04-streaming` — because
they share one scope ruling, one reference set, and one set of boundaries.
`baseline-04-streaming.framing.md` records only what is specific to the
prescriptive leaf and refers here for the rest.

## Trigger and question

`rest-api-standard.md` version 1.0.0 excludes streaming by owner ruling at
Gate D (2026-08-09), recorded in §1.2 and in `PLAN.md` Phase 6. The exclusion
predates streaming becoming the default shape of AI APIs. Phase 6 closes that
gap under the Part II amendment rule.

The research question: **when a conforming API delivers a response
incrementally over a single HTTP request, what does the field actually do —
and on which axes does it genuinely split?**

## Owner rulings that scope this phase (2026-08-10)

1. **Coverage.** Server-Sent Events, long-polling, and streaming HTTP bodies
   (chunked NDJSON/JSON Lines). **WebSockets are an explicit non-goal**: after
   the `101` upgrade the exchange leaves HTTP, so no status code, media type,
   conditional-request, `Problem Details`, or applicability-switch machinery in
   this standard can bind it. Phase 6 states that boundary rather than leaving
   §1.2's open deferral in place.
2. **Deliverable shape.** The owner ruled §13-in-document with the body held in
   a separate referenced document, for progressive disclosure. **The reading
   recorded here — and awaiting explicit confirmation before Step 3 drafts
   anything — is the detail split:** a **compact normative §13** in
   `rest-api-standard.md` carrying the `R13.x` rules, paired with a separate
   **informative companion document** holding the explanatory body — mechanism
   comparison, wire examples, vendor evidence, client guidance. Progressive
   disclosure: an API that does not stream declares the `streaming`
   applicability switch off and never opens the companion. The rules stay in
   the standard so the conformance surface (Appendix A checklist, Part II
   provenance rows, `conformance/spectral.yaml`) remains single. The exact
   normative/informative line is set at drafting, once the rule count is known.

## Prior-work check (`CLAUDE.md` rule 1)

Checked against the `research/README.md` inventory on 2026-08-10: **no existing
prompt covers streaming.** `survey-05-reliability` covers asynchronous
operations as *job resources with polling*, not incremental delivery;
`survey-07-webhooks` covers *outbound* server-to-server delivery, which is a
different direction and a different trust boundary. Neither is a reusable
prompt for this question, and neither report can be re-read to answer it. These
are new top-level stems in each series, not follow-up leaves.

## Reference set — and its deliberate divergence from the series charter

`survey-00-series.framing.md` fixes the survey series on eight references
(Stripe, GitHub, Google/AIP, Microsoft, Twilio, Shopify, Zalando,
AWS-as-contrast). **`survey-08` deliberately departs from that set**, and the
departure is the finding that motivates the phase: most of the eight publish no
streaming surface at all. Stripe and Zalando are non-participants here; keeping
them as primary references would produce eight columns of "not applicable."

The reference set for this surface is therefore chosen by *who actually ships
streaming*:

- **Mandatory (standing project rule, and the heart of this domain):** OpenAI,
  Anthropic, Google Gemini. All three stream; all three negotiate it
  differently, which is itself the primary contested axis.
- **Carried over from the charter where they have a real streaming surface:**
  Google (AIP plus the Gemini `alt=sse` mechanism), Microsoft/Azure, GitHub,
  Shopify (bulk operations return JSON Lines), AWS-as-contrast (event-stream
  encoding over HTTP).
- **Added because they are the field's reference implementations of
  non-SSE streaming:** Kubernetes watch (long-poll and chunked JSON),
  Elasticsearch and Docker (NDJSON), plus any post-2023 API with a documented
  resumption mechanism.
- **Recorded as non-participants, briefly:** Stripe, Twilio, Zalando. A
  one-line "no public streaming surface, verified <date>" row is a real
  finding, not a gap.

The charter's other series-wide conventions — descriptive-only mandate,
primary-sources-first, per-finding confidence, surfaced conflicts, a scoped
Contested Axes Register, and the specification-grade detail requirement — apply
to `survey-08` unchanged.

## Framing-level orientation (unverified; the run must verify every item)

These were noted to shape the prompt, not established. Each is a claim the run
must confirm against primary sources or correct.

- Server-Sent Events is a **WHATWG Living Standard section, not an RFC**. Its
  authority class differs from the RFCs this standard cites elsewhere, and the
  run must state that class explicitly rather than calling SSE "the SSE spec."
  The registration status of `text/event-stream` in the IANA media-type
  registry needs checking separately from the WHATWG text.
- `Transfer-Encoding: chunked` is **HTTP/1.1 framing (RFC 9112)**. HTTP/2 and
  HTTP/3 have no chunked coding. Any rule phrased in terms of chunked encoding
  would be version-specific — the run must establish what is actually
  version-neutral.
- **NDJSON / JSON Lines has no standards body.** `application/x-ndjson` appears
  widely; its IANA registration status must be checked. This connects directly
  to the precedent already set in this repository: `Operation-Location` was
  declined on IANA-registry grounds (`baseline-02i`), and the same test applies
  to any media type or header this phase would bless.
- Once a `200` and its headers are on the wire, **the status code cannot be
  revised**. How each reference signals an error that occurs after that commit
  point is expected to be the sharpest divergence, and it is the axis on which
  this standard's RFC 9457 `problem+json` obligation (`R9.x`, riding `AC-003`)
  is hardest to satisfy.
- `EventSource` **cannot set request headers**, including `Authorization`.
  Whether references respond with query-parameter tokens, cookies, or
  fetch-based readers is both a practice question and a security question, and
  it touches the ratified `R4.17` CORS header-exposure rule.

## Boundaries

In scope:

- negotiation of incremental delivery (request body flag, `Accept`, query
  parameter, distinct endpoint or method name);
- framing and media types: `text/event-stream` field grammar, NDJSON/JSON
  Lines, chunked JSON arrays, and any vendor-specific encoding;
- event typing, ordering, sequence identifiers, and terminal signaling;
- errors raised after the response status is committed, including trailer
  fields and in-band error objects;
- resumption and replay: `Last-Event-ID`, `id:`, cursors, and what each
  reference guarantees on reconnect;
- keep-alive, idle timeout, proxy buffering, and compression interactions;
- browser-client constraints: `EventSource` versus `fetch`, CORS behavior,
  header visibility, per-origin connection limits;
- long-polling as a distinct mechanism: hold times, empty-response
  conventions, and cursor handoff;
- the boundary with this standard's existing `202` + operation-resource
  contract (`R10.9`) — documented as a question, not resolved;
- metadata delivered in a final chunk (usage, totals, cursors).

Out of scope:

- **WebSockets** (owner ruling above) — recorded only as a boundary statement;
- WebTransport, gRPC streaming, Server-Sent-Events-over-HTTP/3 exotica beyond
  what a reference actually ships;
- message brokers and event-streaming platforms (Kafka, Pub/Sub, SNS,
  EventBridge) except as contrast where a reference offers one instead of
  streaming;
- outbound webhooks — settled in `survey-07` and `OP-016`;
- client SDK design, server frameworks, gateways, and load-balancer tuning
  beyond the observable wire behavior they force;
- re-litigating any 1.0 decision.

## Source seeds

Checked only to frame the field. The run must verify currency, record access
dates, and expand the set.

- WHATWG HTML Living Standard, Server-Sent Events: https://html.spec.whatwg.org/multipage/server-sent-events.html
- IANA media types registry: https://www.iana.org/assignments/media-types/media-types.xhtml
- RFC 9112, HTTP/1.1 (chunked transfer coding): https://www.rfc-editor.org/rfc/rfc9112.html
- RFC 9110, HTTP Semantics (status codes, trailer fields): https://www.rfc-editor.org/rfc/rfc9110.html
- RFC 9457, Problem Details for HTTP APIs: https://www.rfc-editor.org/rfc/rfc9457.html
- JSON Lines: https://jsonlines.org/
- Fetch Standard (CORS, streams): https://fetch.spec.whatwg.org/
- OpenAI streaming: https://platform.openai.com/docs/api-reference/streaming
- Anthropic streaming: https://docs.claude.com/en/docs/build-with-claude/streaming
- Google Gemini streaming: https://ai.google.dev/api/generate-content
- Kubernetes API concepts, watch and chunking: https://kubernetes.io/docs/reference/using-api/api-concepts/
- Shopify bulk operations (JSON Lines): https://shopify.dev/docs/api/usage/bulk-operations/queries
- Microsoft REST API guidelines: https://github.com/microsoft/api-guidelines
- Google AIP-151, long-running operations: https://google.aip.dev/151

## Why two leaves rather than one

Owner-selected Option A over a single lighter leaf (Option B, declined: thin
evidence on genuine forks invites re-litigation at the ratification gate) and
over a separate extension document (Option C, declined: splits the conformance
surface). The split preserves the series boundary that the repository depends
on — `survey` documents the field and makes no recommendations; `baseline`
proposes rules and cites the survey's evidence. Collapsing them would produce a
report that recommends from its own summary, which is the failure mode
`CLAUDE.md` names explicitly.
