# baseline-04 — Streaming posture (report, 2026-08-10)

Research leaf under Phase 6. Series `baseline` = **prescriptive**: this report
**proposes** normative principles and does not ratify them. Nothing here binds
`rest-api-standard.md`. Ratification is the owner walk landing in
`research/decisions/baseline-04-streaming.decision.md`; §13 prose is a later,
separate step. Per `research/CONSTRAINTS.md` no normative standard prose is
drafted here — the two exceptions are the two artifacts the prompt explicitly
requires as output, the proposed §1.2 boundary text and the proposed
`streaming` switch definition (§6 below), both marked as proposals.

Provisional identifiers are in the **`ST-*` series** (`ST-001`…`ST-020`), a new
series. `R1.3` freezes `HS-*`, `AC-*`, and `OP-*` as 1.0 provenance keys;
minting in those would fabricate lineage.

Label key, per `rest-api-standard.md` §1.6: **[FACT]** protocol requirement
carried by a published standard · **[COMPARATIVE]** selected default, chosen
from legitimate alternatives on surveyed evidence · **[POLICY]** this project's
choice · **[INFERENCE]** reasoning from the above. A principle resting on a
WHATWG Living Standard, an obsoleted W3C Recommendation, an Internet-Draft, or
a convention with no standards body is **not** `[FACT]`, however universal its
adoption. Phase 4 relabelled twelve rules for exactly that error.

Evidence base: `research/reports/survey-08-streaming.report.2026-08-10.md`
(cited as **`survey-08`**, with its Contested Axes Register rows cited as
**S1**–**S14**). Independent verification performed for this leaf on
**2026-08-10** is marked **[verified here]** and listed in §2.

---

## 1. Executive recommendation

**The posture.** Treat incremental delivery as a **delivery mode of an ordinary
`200 OK` response**, not as a new resource model, a new status code, or a new
error framework. A conforming streaming response commits `200` with an accurate,
self-delimiting media type in `Content-Type`; it is selected by content
negotiation on `Accept` wherever one endpoint serves both a streamed and a
non-streamed representation, because the ratified `R4.10` already fixes
media-type selection as the negotiation mechanism and already supplies the
`406` rejection guard that `R1.9` exists to provide for `dry_run`. Its frames
are individually typed, and it ends with a documented terminal event so a client
can tell completion from truncation. An error raised after the status is
committed is delivered in-band, as a **problem details object by data model**
(RFC 9457 §3) bound to this standard's `type`/`code` discipline (`R5.13`) and
listed in the API's problem catalog (`R5.16`) — but **not** as an
`application/problem+json` response, which is structurally unavailable once
`200` is on the wire. Resumption is a conditional SHOULD, not a MUST, because
the field's own mechanism (`id:` / `Last-Event-ID`) is used by nobody and the
two implementations that exist resume different things. Where a capability
offers both a stream and a `R10.9` operation resource, the two are one
capability with one identity: the stream carries the operation identifier and
the two channels must agree on terminal state. Everything about keep-alive
intervals, proxy buffering, browser relays, and long-poll tuning is deployment
guidance and belongs in the informative companion, not in §13.

**The single hardest call, and why.** The post-commit error. `R5.12` requires
every application-generated error to be *servable as* `application/problem+json`
when the client asks. Once `200 OK` and its headers are on the wire the status
line cannot be replaced (structurally — RFC 9110 contains no prohibition; the
finality comes from the message grammar, `survey-08` §3.4), trailer fields are
ruled out by the RFC that defines them (RFC 9110 §6.5.1: "a server SHOULD NOT
generate trailer fields that it believes are necessary for the user agent to
receive" **[verified here]**), and the response `Content-Type` is already the
stream's media type, so a frame cannot be labelled `application/problem+json`.
Two facts verified for this leaf decide the shape of the answer, and they point
in opposite directions:

1. **RFC 9457 Appendix C anticipates exactly this embedding.** Verbatim:
   "Problem details can be embedded in other formats either by encapsulating one
   of the existing serializations (JSON or XML) into that format or by
   translating the model of a problem detail (as specified in Section 3) into
   the format's conventions," illustrated with a problem document inside an HTML
   `script` element **[verified here]**. An SSE `data:` field carrying a
   problem+json serialization is that pattern. So the *data model* travels
   legitimately in-band.
2. **RFC 9457 §3.1 forbids the `status` member from disagreeing with the
   response.** Verbatim: "The 'status' member, if present, is only advisory; it
   conveys the HTTP status code used for the convenience of the consumer.
   **Generators MUST use the same status code in the actual HTTP response**, to
   assure that generic HTTP software that does not understand this format still
   behaves correctly" **[verified here]**. So an in-band object carrying
   `"status": 503` inside a `200` response violates a Standards Track MUST —
   which is precisely what AWS Bedrock does in substance, carrying 424/429/500
   as data labels inside a `200` (`survey-08` §5.6, S6 option (c)).

The proposal therefore **carves out from `R5.12` and requires a scoped
relaxation of `R5.13`**, both enacted as a MINOR amendment: the mid-stream error
carries the problem details object with `type`, `title`, `code`, `detail`, and
**`status` omitted** — RFC 9457 permits its absence, `R5.13` currently does not.
The carve-out's scope is exactly the two things that become structurally
unavailable at the commit point: the media-type label and the `status` member.
Everything `R5.12` exists to buy — a stable machine-readable identity per error,
a published catalog, one client handler — survives intact. Where the capability
also exposes an operation resource, the full problem document *with* `status` is
required from that resource, which restores the `R5.12` capability obligation
out-of-band. Full analysis in §3.1; the five-surface amendment impact is
enumerated there.

**Second-hardest, and the one most likely to be contested at the gate:**
requiring `Accept`-based negotiation rules against unanimous vendor practice.
Every deep-dive provider uses a request-body flag (S1). The narrow derivation is
composition, not preference — where one endpoint serves both shapes the choice
*is* media-type selection, and `R4.10` already governs it and already forbids
the query-parameter alternative — but the owner should rule on it explicitly
rather than have it arrive as a consequence. It is listed as an owner decision
in §7.

---

## 2. Standards-and-currency matrix

Every source relied on by a proposed principle. "Access date" is the date this
leaf retrieved it; rows marked **[verified here]** were fetched or probed by
this leaf independently of `survey-08`.

| Source | URL | Authority class | Version / date | Access date |
| --- | --- | --- | --- | --- |
| RFC 9457, *Problem Details for HTTP APIs* — §3 model, §3.1 `status`, Appendix C embedding **[verified here]** | `https://www.rfc-editor.org/rfc/rfc9457.html` and `.txt` | IETF **Standards Track** | Published July 2023 | 2026-08-10 |
| RFC 9110, *HTTP Semantics* — §6.5.1 trailer limitations **[verified here]** | `https://www.rfc-editor.org/rfc/rfc9110.txt` | IETF **Internet Standard**, STD 97 | June 2022 | 2026-08-10 |
| RFC 9112, *HTTP/1.1* — §7.1 chunked coding is HTTP/1.1-only | `https://www.rfc-editor.org/rfc/rfc9112.html` | IETF **Internet Standard**, STD 99 | June 2022 | via `survey-08` §3.3, 2026-08-10 |
| RFC 8895, *ALTO Incremental Updates Using SSE* **[verified here]** | `https://www.rfc-editor.org/rfc/rfc8895.txt` | IETF **Standards Track** (Datatracker: Proposed Standard, WG `alto`) | November 2020 | 2026-08-10 |
| RFC 8895 status corroboration **[verified here]** | `https://datatracker.ietf.org/doc/rfc8895/` | IETF Datatracker | Proposed Standard; supersedes `draft-ietf-alto-incr-update-sse-22` | 2026-08-10 |
| RFC 6838, *Media Type Specifications and Registration Procedures* — §3.4 **[verified here]** | `https://www.rfc-editor.org/rfc/rfc6838.html` | IETF **BCP 13** | January 2013 | 2026-08-10 |
| RFC 6648, *Deprecating the "X-" Prefix* — scope includes media types **[verified here]** | `https://www.rfc-editor.org/rfc/rfc6648.txt` | IETF **BCP 178** | June 2012 | 2026-08-10 |
| RFC 7464, *JSON Text Sequences* — registers `application/json-seq` | `https://www.rfc-editor.org/rfc/rfc7464.html` | IETF **Standards Track** | February 2015 | via `survey-08` §3.2, 2026-08-10 |
| WHATWG HTML Living Standard §9.2, *Server-sent events* **[verified here]** | `https://html.spec.whatwg.org/multipage/server-sent-events.html` | **WHATWG Living Standard — not an RFC**, no IETF consensus, continuously revised, no version number | Living; retrieved 2026-08-10 | 2026-08-10 |
| W3C *Server-Sent Events* Recommendation — **obsolete**, redirects to WHATWG | `https://www.w3.org/TR/eventsource/` | **W3C Recommendation, obsoleted** | February 2015; marked obsolete | via `survey-08` §3.0a, 2026-08-10 |
| WHATWG Fetch Standard — CORS-safelisted response headers | `https://fetch.spec.whatwg.org/` | **WHATWG Living Standard** | Living | via `survey-08` §3.5, 2026-08-10 |
| IANA media types — `text/event-stream` per-type probe: **HTTP 404** **[verified here]** | `https://www.iana.org/assignments/media-types/text/event-stream` | **IANA registry** | probe | 2026-08-10 |
| IANA media types — control probe `text/html`: **HTTP 200** **[verified here]** | `https://www.iana.org/assignments/media-types/text/html` | **IANA registry** | probe | 2026-08-10 |
| IANA media types — `application/x-ndjson` probe: **HTTP 404** **[verified here]** | `https://www.iana.org/assignments/media-types/application/x-ndjson` | **IANA registry** | probe | 2026-08-10 |
| IANA media types — `application/json-seq` probe: **HTTP 200 (registered)** **[verified here]** | `https://www.iana.org/assignments/media-types/application/json-seq` | **IANA registry** | probe | 2026-08-10 |
| IANA HTTP Field Name Registry — `X-Accel-Buffering` **absent** (0 matches across 259 rows) **[verified here]** | `https://www.iana.org/assignments/http-fields/field-names.csv` | **IANA registry** | 259 rows at retrieval | 2026-08-10 |
| IETF `httpapi` WG document list — **no streaming, SSE, or stream media-type work item** **[verified here]** | `https://datatracker.ietf.org/wg/httpapi/documents/` | IETF Datatracker | 3 active WG drafts, 2 with IESG, 6 RFCs at retrieval | 2026-08-10 |
| `draft-ietf-httpapi-rest-api-mediatypes` — registers OpenAPI media types only; no streaming content **[verified here]** | `https://datatracker.ietf.org/doc/draft-ietf-httpapi-rest-api-mediatypes/` | **Internet-Draft**, WG document, In WG Last Call | rev **09**, expires **2026-11-24** | 2026-08-10 |
| `draft-gupta-httpapi-events-query` — HTTP Events Query | `https://datatracker.ietf.org/doc/draft-gupta-httpapi-events-query/` | **Internet-Draft**, individual submission, **not WG-adopted** | rev **03**, expires **2027-01-06** | via `survey-08` §3.0a, 2026-08-10 |
| NDJSON specification | `https://github.com/ndjson/ndjson-spec` | **No standards body** — a GitHub repository | v1.0.0, last update 2014-10-19 | via `survey-08` §3.2 |
| jsonlines.org | `https://jsonlines.org/` | **No standards body** — a project website | retrieved 2026-08-10 | via `survey-08` §3.2 |
| Vendor evidence: OpenAI, Anthropic, Gemini, Azure OpenAI, AWS Bedrock, Kubernetes, Docker, Elasticsearch, Shopify, Consul, SQS, Stripe, Twilio, Zalando, Google AIP-151, Azure REST API Guidelines | see `survey-08` §5, §6, §9 for the 41-source inventory with per-row URLs | **Vendor documentation — comparative evidence only, never protocol authority** (`research/CONSTRAINTS.md`) | as recorded per row in `survey-08` | 2026-08-10 |

**Two currency notes that bear on classification.**

1. **The SSE authority chain contains an obsoleted link.** RFC 8895 (Standards
   Track) normatively references `[SSE]` as the **W3C Recommendation of
   February 2015**, which is now obsolete and redirects to the WHATWG HTML
   Living Standard (`survey-08` §3.0a point 3). Consequence for drafting: no
   proposed principle may say "the SSE specification" without naming which
   document it means. Every principle below names the WHATWG HTML Living
   Standard §9.2 explicitly, with its access date.
2. **`text/event-stream` is used on the wire by a Standards Track RFC and is
   still unregistered.** RFC 8895 §8.2 shows `Accept: text/event-stream,
   application/alto-error+json` in a request and `Content-Type:
   text/event-stream` in the matching response **[verified here]**, and its §12
   asserts that "all other media types used in this document have already been
   registered." The IANA per-type probe returns **404** against a `text/html`
   control returning **200** **[verified here]**. The registry governs the
   registration question; the RFC's sentence is an assumption, not a
   registration event, because an RFC registers a media type only through its
   own IANA Considerations, and RFC 8895's registers only its two `alto-*`
   types. **This conflict is surfaced, not averaged** (`research/CONSTRAINTS.md`),
   and it is the crux of §3.2 below.

---

## 3. Composition analysis

One subsection per ratified rule the proposal touches, stating **composes**,
**amends**, or **carves out**. The amendment rule (Part II, ratified Gate D
2026-08-09) makes every rule change atomic across five surfaces: **(1)** the
rule text, **(2)** its decision record as a dated annotation, never a silent
edit, **(3)** its Part II provenance row, **(4)** its Appendix A checklist row,
**(5)** the Appendix E worked example where it appears. Each amending subsection
below enumerates all five.

### 3.1 `R5.12` and `R5.13` — the problem+json obligation: **CARVES OUT (amendment required)**

**What the ratified rules say.** `R5.12`: every API MUST be *capable of
returning* every error response the application itself generates as
`application/problem+json` when the client requests it, with one named
carve-out for infrastructure-generated errors. `R5.13`: every problem document
MUST carry `type`, `title`, `status`, and a stable `code`, with the fixed
`type`/`code` binding template.

**Why a carve-out and not composition.** Three structural facts, each
primary-sourced:

- **The status line is not revisable.** RFC 9110 contains no sentence forbidding
  revision; the finality arises from the message grammar — status line first,
  then fields, then content (`survey-08` §3.4, which explicitly warns against
  over-reading RFC 9110 here, the same over-reading `baseline-02i` had to
  correct for RFC 6648). **[FACT]** about the grammar; **[INFERENCE]** about the
  consequence.
- **Trailers cannot carry it.** RFC 9110 §6.5.1: "Because of the potential for
  trailer fields to be discarded in transit, a server SHOULD NOT generate
  trailer fields that it believes are necessary for the user agent to receive"
  **[verified here, verbatim]**. An error the client must see is by definition
  necessary for the user agent to receive. Zero of thirteen references use
  trailers (S6). **[FACT]**
- **A frame cannot be labelled.** The response `Content-Type` is already the
  stream's media type, so the `application/problem+json` label — the mechanism
  by which a problem document identifies itself — is unavailable for an
  individual frame (`survey-08` §8.2). **[INFERENCE]**

**What the carve-out is, exactly.** The proposal (ST-007) requires the in-band
error frame's payload to be a **problem details object by the RFC 9457 §3 data
model**, carrying `R5.13`'s required members other than `status` — `type`, `title`, and
`code` — bound by its 1:1 `type`/`code` template and listed in the `R5.16`
catalog, and to **omit `status`**. `detail` and extension members remain
permitted on the usual terms; the carve-out subtracts, and adds nothing, so no
member is required in-band that an ordinary error response does not already
carry. RFC 9457 §3.1 makes `status` optional ("if present") but binds a
present `status` to the actual response status by a MUST **[verified here]**; a
`200` response carrying `"status": 503` in the body violates it. RFC 9457
Appendix C independently blesses embedding the model in another format
**[verified here]**, which is the authority for carrying the object in a frame
at all. Note Appendix C is an appendix and therefore informative; it is cited as
evidence that the embedding is anticipated, not as a normative permission.

So the carve-out is scoped to exactly two things, and no more: **the media-type
label** and **the `status` member**.

**Which rule each half hits.**

| Half of the carve-out | Rule affected | Character | Amendment character |
| --- | --- | --- | --- |
| The mid-stream error is not servable as an `application/problem+json` **response** | `R5.12` | A second named exception, alongside the existing infrastructure carve-out | **Relaxation ⇒ MINOR** version bump |
| The in-band object omits the `status` member `R5.13` requires | `R5.13` | Scopes the required member set to problem documents carried as a response body | **Relaxation ⇒ MINOR** version bump |

**The five surfaces, enumerated.** For `R5.12`: (1) rule text in §5.3 gains a
second named exception scoped to the `streaming` switch; (2) `AC-003` in
`research/decisions/baseline-02-api-contracts.decision.md` gains a **dated
annotation** recording the Phase 6 carve-out — never a silent edit; (3) the
Part II §II.1 row `AC-003 → R5.12, R12.6, R12.7` gains the `ST-*` key, and a new
row maps the streaming decision record to its rules; (4) the Appendix A
checklist row for `R5.12` gains the streaming condition; (5) Appendix E has no
streaming worked example today, so this surface is **satisfied by adding one**
rather than by editing one — the drafting step should treat the absence as a
task, not as a no-op. For `R5.13`: the same five, with the §5.3 member-set
sentence scoped and the Appendix A row conditioned.

**What is preserved, and this is the argument for the shape.** The obligation
`R5.12` exists to buy is not the media type — it is that every application error
has a stable, machine-readable, catalogued identity a generic client can branch
on. All of it survives: `type` and `code` unchanged, the catalog (`R5.16`)
covering mid-stream codes, one client error handler across both paths. What is
surrendered is the label and the advisory status number, and both are
surrendered because the protocol makes them unavailable, not because they are
inconvenient. **The field's alternative is strictly worse**: seven of seven
references ship a private schema with no registered identity (S6), and AWS
demotes real status codes to data labels in a way RFC 9457 §3.1 forbids for a
problem document.

**A third, out-of-band restoration.** Where the `async-operations` switch is on
and a `R10.9` operation resource exists for the same capability (ST-009), the
full problem document — **with** `status`, under `application/problem+json`, in
an ordinary error response — MUST be retrievable from the operation resource.
That restores `R5.12`'s capability obligation in the only place where a status
code is still available to be generated.

**Flagged for the owner, and this is a genuine conflict with a ratified rule:**
`R5.1` — "The status code MUST match the registered semantics of the outcome. A
failed operation MUST NOT return 2xx" (`HS-010`, protocol requirement, RFC
9205, confidence high). A stream whose generation fails after the commit point
returns `200` for an operation that failed. This is not a technicality: it is
AWS Bedrock's documented behaviour, Azure's documented `Status 200 with
finish_reason: "content_filter"`, and it is unavoidable for every reference in
the survey. The proposal does **not** quietly work around it. The reading
offered — and it is an owner call, not a research finding — is that `R5.1` binds
the status to the outcome **as known at the moment the status is generated**,
and a streaming response generates its status before any outcome exists. That
reading needs to be stated in writing wherever `R5.1` or ST-001 is drafted, or
`R5.1` needs an explicit streaming scope. Raised in §7 as an owner decision.

### 3.2 `R10.9` and §10.1 — the operation resource: **COMPOSES (no amendment)**

`R10.9` requires a `202 Accepted` to identify its operation resource in the body
and SHOULD carry `Location` denoting the operation, never the result; `R10.1`
requires every `202` to return an addressable operation resource with terminal
states, an expiry, and a failure representation.

**The proposal composes, and the composition runs in one direction only:
streaming is an access mode over a capability, never a substitute for the
operation resource.** Three consequences, all in ST-001 and ST-009:

1. **A streaming response is never a `202`.** `R10.1` binds `202` to an
   operation resource, and the WHATWG HTML Living Standard §9.2 makes any
   non-`200` status fatal to an `EventSource` client — "If res's status is not
   200, or if res's `Content-Type` is not `text/event-stream`, then fail the
   connection," and "Once the user agent has failed the connection, it does not
   attempt to reconnect" **[verified here, verbatim]**. A `202` streaming
   response would therefore both fork `R10.1` and permanently kill browser
   clients. No reference in the survey uses `202` for a stream (S5, §8.1 finding
   2). This is the cleanest composition result in the leaf.
2. **Shared identity.** Where a capability offers both channels, the stream's
   first frame carries the same operation identifier the `202` body carries per
   `R10.9`. This is not invented: it is OpenAI's shipped design — `background:
   true` creates a retrievable `response` resource, `GET /v1/responses/{id}`
   polls it, and `GET /v1/responses/{id}?stream=true&starting_after=<n>` streams
   it, one resource with two access modes (S13 option (a), `survey-08` §5.1).
3. **Terminal-state agreement, with the operation resource authoritative.** A
   stream that ends `failed` and an operation resource that reads `succeeded`
   is a contract violation. Naming the operation resource as authoritative
   follows from `R10.1`, which already makes it the defined-terminal-state
   channel; the stream is a view.

**AIP-151 is explicitly not adopted.** Google's guideline says an LRO method's
response "must not be a streaming response" (S13 option (c)). It is declined for
three stated reasons (§5, Declined options): it governs a protobuf-first RPC
surface with different apparatus; its rule is about *one method's response
type*, not about whether a capability may offer both shapes; and Google's own
Gemini product ships both shapes, so the guideline does not describe even its
own author's practice. `survey-08` §11.2 Conflict D reaches the same
adjudication from the descriptive side.

### 3.3 `R4.17` — CORS header exposure: **COMPOSES UNCHANGED (no new header)**

`R4.17` requires standard-bound response headers to be listed in
`Access-Control-Expose-Headers` for cross-origin browser clients.

**The proposal adds no standard-bound response header, so `R4.17`'s list is
unchanged.** That is a deliberate design choice, not an accident: ST-017
(Companion) recommends carrying per-stream metadata in the stream body rather
than in response headers. The mechanism reason is verified: `Content-Type` is
CORS-safelisted and every custom header is not (`survey-08` §3.5, WHATWG Fetch),
so body-carried metadata is readable cross-origin with no server opt-in while a
header-carried stream id is invisible without one. `request-id` is already
required on every response by §1.10 and is therefore already inside `R4.17`'s
scope; a streaming response inherits that with no new rule.

**Consequence worth recording:** if a provider does put per-stream metadata in a
response header, `R4.17` already binds it. No new rule is needed for that case
either. `R4.17` is untouched by Phase 6.

### 3.4 `R1.8`, `R1.9`, §1.10 reserved names, `R1.6` switches: **COMPOSES, plus registry additions**

- **`R1.9`'s rejection-guard model is adopted, not extended.** ST-003 proposes
  that an endpoint that does not implement streaming, receiving a `stream`
  request modifier with any value, MUST be rejected with `400` rather than
  silently returning a non-streamed response. This is `R1.9`'s structure applied
  to a second modifier, for the same documented reason: silently ignoring an
  unimplemented request modifier is the hazard that disqualified `Prefer:
  validate-only` (§1.10 provenance, `R4.15`).
- **Where `Accept` is the negotiation mechanism the guard already exists.**
  `R4.10` already says a request whose `Accept` excludes every supported
  representation SHOULD receive `406 Not Acceptable` rather than a silently
  substituted type. That is the same guard, already ratified, at SHOULD
  strength. **This is the decisive argument for `Accept` over a body flag**: one
  mechanism inherits its rejection guard from a ratified rule; the other needs a
  new one minted for it. Recorded in ST-002 and §5.
- **§1.10 additions proposed** (three registry rows, no rule change): the media
  type `text/event-stream` with its registration gap disclosed in the row; the
  request modifier name `stream`; and the SSE event-type name `error` for the
  in-band problem frame. Rationale for the third: all three deep-dive providers
  independently chose `event: error` (S6 option (a)), and §1.10's governing
  principle is same concept, same name, everywhere. **Note that the third is
  not merely a row:** §1.10 today has tables for query parameters, headers,
  media types, and action verbs, and a reserved *frame-type name* fits none of
  them, so admitting it opens a fifth reserved-name category. That is part of
  Decision 4 in §7, not a drafting detail.
- **`R1.6`**: one new switch, `streaming`. Scope in §6. `R1.6` requires every
  switch to control at least one rule and requires a stated reason when
  declared off; both are satisfiable.

**Flagged for the owner:** the §1.10 reserved-media-type table currently holds
three entries, all RFC-backed and registered (`application/problem+json`,
`application/merge-patch+json`, `application/json-patch+json`). Adding an
**unregistered** media type changes that table's character. The precedent for
doing it with disclosure rather than declining exists in the same section — the
reserved-header table already carries `Idempotency-Key` (whose standardizing
draft expired 2026-04-18, marked "never cite it as a standard") and `RateLimit`
(a pinned unpublished draft revision, marked "MUST NOT be described as
standards-compliant"). The proposed row follows that pattern exactly. Put to
the owner as Decision 4 in §7.

### 3.5 `R4.10` and `R4.11` — content negotiation: **COMPOSES, and decides the negotiation question**

`R4.10`: "Media-type selection MUST use HTTP content negotiation, never a
`format` query parameter." `R4.11`: every response whose content was selected by
a request header MUST send `Vary` listing each header that influenced selection.

Streaming changes the response's media type. Where one endpoint serves both a
streamed and a non-streamed representation, choosing between them **is**
media-type selection, so `R4.10` governs it already and the two shipped
alternatives fail it:

- a **query parameter** (`alt=sse`, `?watch=1`) is the mechanism `R4.10`
  forbids by name for media-type selection;
- a **request-body flag** (`"stream": true`) makes the response media type
  depend on the request body, which is not content negotiation at all — S1's own
  trade-off column records that it "breaks content negotiation and GET-ability."

`R4.11` then requires `Vary: Accept` on any endpoint that negotiates the two
shapes. Both results are composition, not new policy. **What is new policy** is
the decision to let a ratified rule overrule unanimous vendor practice, and that
is put to the owner in §7 rather than assumed.

**The escape hatch is preserved:** a distinct resource that streams
unconditionally (S1 option (b) — AWS `invoke-with-response-stream`, Docker
`/events`) performs no media-type selection at all, so `R4.10` does not bind it.
ST-002 permits it explicitly.

### 3.6 `R8.2` — credentials in query parameters: **COMPOSES; it forecloses the field's browser answer**

`R8.2`: "Access tokens MUST NOT be accepted or emitted in URI query parameters"
(`OP-002`, protocol requirement, BCP 240 / RFC 9700, confidence high).

The browser `EventSource` constructor cannot set request headers, so it cannot
send `Authorization` or `x-api-key` (`survey-08` §3.1, §4.8; WHATWG HTML Living
Standard §9.2 — the only `EventSource`-native credential is a cookie, via
`withCredentials`). The field's one workaround is Gemini's older `key=` query
parameter (S10 option (b)). **`R8.2` already forbids it.** No new rule is
needed and none is proposed; the consequence is stated instead, in the
companion (ST-018): under this standard a browser page cannot authenticate a
direct `EventSource` connection, so browser clients use a `fetch`-based reader
(which can set headers) or a first-party relay that holds the credential
server-side. That matches every documented path in the field — none of the five
references documents a browser-direct `EventSource` integration (S10).

**Security consequence stated rather than blessed, as the prompt requires:** a
query-parameter token leaks into access logs, browser history, `Referer`
headers, and shared URLs; BCP 240 is the authority and `R8.2` is this standard's
enactment of it. A provider wanting browser-native `EventSource` must either
relax `R8.2` — a MUST deviation, nonconformance unless recorded as an approved
exception under `R1.7` — or use a cookie, which introduces CSRF surface and
`withCredentials` CORS handling that §8 governs. Neither is recommended here.

### 3.7 §12 — client obligations: **COMPOSES, with one extension**

`R12.4` requires clients to tolerate unknown response fields and unknown enum
values. Streaming adds a third unknown: **unknown event types**. Both Anthropic
("new event types may be added, and your code should handle unknown event types
gracefully") and OpenAI's open `anyOf` discriminator make this a live obligation
(S4, §4.3). ST-012 extends the tolerant-reading obligation to event types; it is
an extension of `R12.4`'s principle to a new surface, not a change to `R12.4`.

`R12.1` (retry only documented-retryable failures; never retry a non-idempotent
request without an idempotency key) composes directly with truncation recovery:
a client that reconnects after a truncated stream is re-issuing a request, and
if the underlying operation is non-idempotent, `R12.1` and `R3.9` already
govern. ST-012 states the connection rather than re-deriving it.

### 3.8 Summary table

| Ratified rule | Verdict | Version impact |
| --- | --- | --- |
| `R5.12` | **Carves out** — second named exception | MINOR (relaxation) |
| `R5.13` | **Amends** — required member set scoped to response-carried documents | MINOR (relaxation) |
| `R5.1` | **Flagged conflict** — needs an owner reading or an explicit streaming scope | MINOR if scoped; none if the reading is merely recorded |
| `R10.1`, `R10.9` | **Composes** | none |
| `R4.17` | **Composes unchanged** — no new standard-bound header | none |
| `R4.10`, `R4.11` | **Composes** — and decides the negotiation question | none |
| `R1.6`, `R1.8`, `R1.9`, §1.10 | **Composes** — one new switch, three registry rows | MINOR (additions) |
| `R8.2` | **Composes** — forecloses query-parameter tokens for `EventSource` | none |
| `R12.1`, `R12.4`, `R12.5` | **Composes**, with tolerant-reading extended to event types | MINOR (added §12 rule) |
| `R5.16` | **Composes** — the catalog covers mid-stream problem codes | none |

Net: **MINOR version bump** (v1.1.0), consistent with `PLAN.md` Phase 6's
expectation. No rule is strengthened, removed, or re-meant, so no MAJOR trigger
fires.

---

## 4. Proposed normative-principles table

Twenty principles. **Twelve** are proposed for the compact normative §13;
**eight** for the informative companion. Strength distribution: **10 MUST**,
**8 SHOULD**, **2 MAY**. Every principle is gated by the `streaming` switch
(§6) unless its Applicability column says otherwise.

Reading the columns: **Class** is the §1.6 classification; **Conf** is
confidence (high / moderate / low); **Home** is `§13` for the compact normative
section or `Comp` for the companion. Register rows are `survey-08` §9 axes.

### 4.1 Proposed for the normative §13

| ID | Strength | Proposed rule sentence | Rationale | Class | Applicability | Evidence and register rows | Conf | Home |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| **ST-001** | **MUST** | A streaming response MUST be a `200 OK` whose `Content-Type` names a self-delimiting stream media type; `202 Accepted` MUST NOT be used for a streaming response, and a stream of concatenated JSON documents MUST NOT be labelled `application/json`. | Three joined facts. `R10.1` binds `202` to an operation resource, so a `202` stream forks a ratified rule. The WHATWG HTML Living Standard §9.2 makes a non-`200` status or a mismatched `Content-Type` fatal and non-retryable for an `EventSource` client. `application/json` on a concatenated-document body is a false statement about the body — a conforming JSON parser fed the whole body fails — and it is the practice `application/x-ndjson` and `application/json-seq` exist to fix. Declaring the media type also closes the survey's own §11.1 gap: all three deep-dive providers leave their stream `Content-Type` undocumented. | `[COMPARATIVE]` on the `200` and the declaration; `[POLICY]` on the `202` prohibition and the mislabel prohibition | `streaming` on | WHATWG HTML LS §9.2 (2026-08-10, verified here); `survey-08` §4.2, §8.1 finding 2, §11.1 item 1; **S2**, **S5**, **S14** | high | §13 |
| **ST-002** | **MUST** | Where one endpoint serves both a streamed and a non-streamed representation, the choice MUST be made by content negotiation on `Accept`, and the response MUST carry `Vary: Accept`; a query parameter MUST NOT select between them. An API MAY instead expose a distinct resource that streams unconditionally, in which case no negotiation applies. | Streaming changes the response media type, so the choice is media-type selection, which `R4.10` already governs and for which it already forbids the query-parameter form; `R4.11` then requires `Vary`. Choosing `Accept` also inherits `R4.10`'s `406` rejection guard for free, where a body flag would require a new guard to be minted (ST-003). A Standards Track RFC uses exactly this mechanism: RFC 8895 §8.2 sends `Accept: text/event-stream`. The distinct-resource escape hatch performs no selection, so `R4.10` does not reach it. | `[POLICY]` — derived from ratified `R4.10`/`R4.11`, against unanimous vendor practice | `streaming` on; the `Vary` clause is `R4.11`'s existing obligation restated for clarity | `R4.10`, `R4.11`; RFC 8895 §8.2 (verified here); `survey-08` §4.1; **S1** | moderate — the rule is composition, the departure from practice is a policy call (§7) | §13 |
| **ST-003** | **MUST** | `stream` is a reserved request-modifier name meaning "deliver this response incrementally". An endpoint that does not implement streaming and receives `stream` — as a query parameter or a body member, with any value — MUST be rejected with `400`, never silently answered with a non-streamed response. | Directly modelled on `R1.9`'s `dry_run` guard, for the identical hazard: silently ignoring an unimplemented request modifier lets a client believe it got what it asked for. The survey found this behaviour documented by **no** reference (§11.1 item 2), so there is no field practice to defer to and the standard must supply one. Reserving the name also prevents two APIs meaning different things by `stream`, which is §1.10's purpose. | `[POLICY]` | `streaming` off as well as on — the guard binds precisely the endpoints that do not stream | `R1.9`, §1.10; `survey-08` §11.1 item 2; **S1** | high | §13 |
| **ST-004** | **SHOULD** | Server-Sent Events framing, served as `text/event-stream`, SHOULD be the default for incrementally generated content. An API adopting it MUST document that the media type has no IANA registration, and MUST NOT describe it as a registered or standardized media type. | The field is unanimous for this job — all three deep-dive providers and Azure (S2 option (a)) — browsers parse it natively, and an IETF Standards Track RFC (RFC 8895) uses it on the wire, which no alternative can claim. The registration gap is real and is disclosed rather than worked around: the per-type IANA probe returns 404 against a 200 control, verified here. The disclosure duty mirrors §1.10's existing treatment of `Idempotency-Key` and `RateLimit`. See §5 for why the IANA-registry test declines `Operation-Location` but admits this. | `[COMPARATIVE]` — a selected default resting on surveyed practice plus a WHATWG Living Standard, explicitly **not** `[FACT]` | `streaming` on; incremental generated content. Record streams and bulk result sets fall to ST-013 | IANA probes (verified here); WHATWG HTML LS §9.2; RFC 8895 §8.2, §12; `survey-08` §2 finding 1, §3.2; **S2**, **S3** | high on the practice, moderate on the default (the registration gap is a standing weakness, §8) | §13 |
| **ST-005** | **MUST** | Every frame in a stream MUST carry a documented type that identifies what the frame is, and the API MUST document its full frame-type vocabulary and state that the vocabulary may grow. | Without a per-frame type a client must parse every payload to learn what it received, and cannot route, ignore, or count frames. The direction of travel is unambiguous: every newest-generation API surveyed chose named types (S4), and both vendors that publish a forward-compatibility statement pair typing with "new event types may be added." The growth statement is what makes ST-012's client tolerance obligation discharge-able. | `[COMPARATIVE]` | `streaming` on | `survey-08` §4.3; **S4** | high | §13 |
| **ST-006** | **MUST** | A stream MUST end with a documented terminal frame carrying the final outcome, so that a client can distinguish normal completion from a truncated connection. A trailing sentinel frame MAY follow the terminal frame, and clients MUST tolerate one. | "Did I get everything?" is unanswerable from connection close alone, and two references end that way (S5 option (d)). A terminal typed event carries the outcome — completed, failed, incomplete — in the same frame, where a bare sentinel carries none and is not valid JSON, so a uniform `JSON.parse` over `data:` lines throws on it. The trailing-sentinel allowance is required by observed practice: Gemini Interactions emits `interaction.completed` **and then** `done` / `[DONE]`, so a rule demanding exactly one terminal frame would outlaw a shipped design for no benefit. | `[COMPARATIVE]` on the requirement; `[POLICY]` on preferring a typed terminal frame over a bare sentinel | `streaming` on. Endless streams — a watch, an event tail — have no normal end; for those the terminal-frame obligation applies to the server-initiated close case only, and the API MUST document that the stream is unbounded | `survey-08` §4.4, §8.2; **S5** | high | §13 |
| **ST-007** | **MUST** | An error raised after the response status is committed MUST be delivered in-band, in a frame of the reserved `error` type, whose payload is a problem details object per RFC 9457 §3 carrying `R5.13`'s required members other than `status` — `type`, `title`, and `code` — bound by `R5.13`'s `type`/`code` template and listed in the `R5.16` catalog, with `detail` and extension members permitted exactly as on any other problem document. The object MUST omit the `status` member. The frame MUST NOT be described as an `application/problem+json` response. | The hardest call; full argument in §3.1. RFC 9457 Appendix C anticipates embedding the model in another format; RFC 9457 §3.1 requires a present `status` to equal the actual response status, which is `200`, so `status` is omitted rather than falsified. Trailers are excluded by RFC 9110 §6.5.1. Preserves everything `R5.12` exists to buy — stable identity, published catalog, one client handler — and surrenders only the label and the advisory number, both structurally unavailable. Strictly better than the field's unanimous private-schema answer, which carries no registered identity at all. **Carves out from `R5.12`; requires `R5.13`'s member set to be scoped.** | `[POLICY]` on the carve-out and the omission; the two constraints it obeys are `[FACT]` (RFC 9457 §3.1, RFC 9110 §6.5.1) | `streaming` on | RFC 9457 §3, §3.1, Appendix C (verified here); RFC 9110 §6.5.1 (verified here); `survey-08` §4.5, §3.4, §8.2; **S6**, **S7** | moderate-high — the constraints are certain, the resolution among them is this project's choice | §13 |
| **ST-008** | **MUST** | An error detected before the response status is committed MUST follow `R5.12` unchanged and be servable as an `application/problem+json` error response, whatever the request's streaming `Accept` or `stream` modifier asked for. A streaming request modifier governs the success representation only. | Closes the hole a gate reviewer will find: a request carrying `Accept: text/event-stream` that fails validation must still produce a `422` with a real problem document, and nothing in the streaming rules may be read as licensing a `200`-plus-error-frame answer to a request that never began succeeding. It is also the field's universal behaviour, stated by Anthropic outright and illustrated cleanly by Kubernetes, where a stale cursor detected before the stream opens yields a real `410 Gone` and the same condition after it opens yields an in-band event. | `[POLICY]` — a scope clarification of `R5.12` | `streaming` on | `R5.12`; `survey-08` §4.5 (Anthropic, Kubernetes rows); **S7** | high | §13 |
| **ST-009** | **MUST** | Where one capability is exposed both as a stream and as a `R10.9` operation resource, the two MUST be one capability with one identity: the stream MUST carry the operation identifier that `R10.9` binds into the `202` body, both channels MUST report the same terminal state with the operation resource authoritative, and the full problem document for a failed operation — carrying `status`, as `application/problem+json` — MUST be retrievable from the operation resource. | Composes with `R10.9` instead of forking it (§3.2), and is the out-of-band half of ST-007's carve-out: the one place a status code is still available to be generated is the operation resource. The shape is not invented — it is OpenAI's shipped `background: true` design, one resource with two access modes and one terminal state (S13 option (a)). AIP-151's mutual-exclusivity rule is declined with reasons in §5. | `[COMPARATIVE]` on the unified shape; `[POLICY]` on making the operation resource authoritative | `streaming` **and** `async-operations` both on | `R10.1`, `R10.9`; `survey-08` §5.1, §8.2, §11.2 Conflict D; **S13** | moderate-high — one shipped exemplar, one contrary guideline, adjudicated | §13 |
| **ST-010** | **SHOULD** | Where a stream is a view over a retained artifact, the API SHOULD offer resumption. An API that offers resumption MUST carry a monotonically ordered position identifier on every frame, MUST document the retention window, and MUST reject a resumption request whose position lies outside that window with a defined error rather than silently restarting the stream. | A MUST fails the evidence test the prompt sets: only two of thirteen references implement resumption, and the SSE specification's own mechanism (`id:` / `Last-Event-ID`) is implemented by **nobody** (S8, verified negative). Conditioning the SHOULD on "a view over a retained artifact" is what makes it decidable — OpenAI can resume because `background: true` stores the response for roughly ten minutes; Anthropic cannot, and re-prompts instead, producing a new generation rather than a continuation. Requiring retention and a defined out-of-window failure is Kubernetes' decade-old contract (`410 Gone` after the history window), the field's only battle-tested version. | `[COMPARATIVE]` | `streaming` on; the SHOULD binds only where a retained artifact exists | `survey-08` §4.6, §8.2; **S8** | moderate — two implementations, resuming two different things | §13 |
| **ST-011** | **MUST** | A long-polling endpoint MUST document its maximum hold duration, and an expired hold MUST return `200` with a well-formed empty-result representation carrying the cursor for the next poll. `204 No Content` MUST NOT be used for an expired hold. | Long-polling needs one rule of its own and no more; everything else is covered by the general rules plus §10 (§5, Declined options). The `204` prohibition has two independent grounds: `R5.7` already binds `204` to a successful DELETE, and the WHATWG HTML Living Standard §9.2 gives `204` a conflicting reserved meaning on this exact surface — "a client can be told to stop reconnecting using the HTTP 204 No Content response code" **[verified here]** — so a server offering both mechanisms on one path would have the two meanings collide. The field agrees by practice: Consul returns a full `200` with the unchanged value, SQS an empty list inside a `200`, and no reference uses `204`. | `[COMPARATIVE]` on the `200`-with-empty-result shape; `[POLICY]` on the `204` prohibition | `streaming` on, long-polling endpoints only | `R5.7`; WHATWG HTML LS §9.2 (verified here); `survey-08` §6 | high | §13 |
| **ST-012** | **MUST** | A client consuming a stream MUST treat a connection that closes without the documented terminal frame as truncated and MUST NOT treat partial content as complete; MUST ignore frame types it does not recognize; MUST NOT depend on keep-alive frames arriving on any schedule; and MUST NOT recover from truncation by replaying a non-idempotent request without an idempotency key. | The client half, belonging in §12 alongside `R12.1`–`R12.9`. Truncation is the failure mode streaming adds and the one clients get wrong; the unknown-type clause extends `R12.4`'s tolerant reading to the surface streaming introduces (§3.7); the keep-alive clause is grounded in the fact that no reference publishes an interval, including the only vendor that ships a keep-alive frame at all; the replay clause connects to `R12.1` and `R3.9` rather than restating them. | `[COMPARATIVE]` on truncation and tolerance; `[POLICY]` on the replay clause | `streaming` on, client side | `R12.1`, `R12.4`, `R3.9`; `survey-08` §4.4, §4.3, §3.1; **S5**, **S4**, **S9** | high | §13 (drafting may place it in §12 with the other client obligations) |

### 4.2 Proposed for the informative companion

| ID | Strength | Proposed guidance sentence | Rationale | Class | Evidence and register rows | Conf | Home |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **ST-013** | **MAY** | An API MAY use newline-delimited JSON for record streams and bulk result sets, served as `application/x-ndjson`, provided it documents that the media type is unregistered and that RFC 6648 discourages new `x-` prefixed names. `application/json-seq` (RFC 7464) MAY be used where a registered media type is required, at the cost of no client ecosystem. | The two format families do not compete: SSE is chosen for incremental generation, JSON Lines for bulk result sets, across every reference examined. NDJSON's real weight is on the request side (Elasticsearch `_bulk`). The `x-` caution is precise and not over-read: RFC 6838 §3.4 says names beginning with `x-` "are no longer considered to be members of this tree (see RFC 6648)," and RFC 6648's SHOULD NOT binds *creators of new parameters*, with media types inside its stated scope — it does not retroactively prohibit using an existing convention. | `[COMPARATIVE]`, with an `[FACT]` component on the registration status | RFC 6838 §3.4, RFC 6648 §3 (both verified here); IANA probe 404 for `application/x-ndjson`, 200 for `application/json-seq` (verified here); `survey-08` §5.8; **S2**, **S3** | high | Comp |
| **ST-014** | **SHOULD** | An SSE stream SHOULD carry each frame's type in both the `event:` field and a `type` member of the payload. | Dual carriage serves both consumers: a browser `EventSource` client registering per-type listeners, and a `fetch`-based reader that splits on `data:` without implementing SSE parsing. Every newest-generation API surveyed chose it. The cost — two places that can disagree — is real, which is why this is guidance rather than a §13 MUST; ST-005's typing requirement already carries the normative weight. | `[COMPARATIVE]` | `survey-08` §4.3; **S4** | high | Comp |
| **ST-015** | **SHOULD** | A streaming contract SHOULD be documented in media-type terms. It SHOULD NOT be phrased as a requirement to use `Transfer-Encoding: chunked`, which exists only in HTTP/1.1. | RFC 9112 §7.1 scopes chunked coding to HTTP/1.1; HTTP/2 and HTTP/3 have no such coding, so chunked-phrased guidance is silently inapplicable over the transports most streaming APIs actually run on. The version-neutral observable properties are the media type and the absence of `Content-Length`. One reference of thirteen uses chunked phrasing. | `[FACT]` on the version scoping (RFC 9112, Internet Standard); `[COMPARATIVE]` on the phrasing preference | RFC 9112 §7.1; `survey-08` §3.3; **S14** | high | Comp |
| **ST-016** | **SHOULD** | An API SHOULD document whether it emits keep-alive frames and in what form. A keep-alive frame SHOULD carry no application state, so that a client that discards it loses nothing. | There is no protocol mechanism to point at: the WHATWG HTML Living Standard §9.2 offers only an authoring note — a comment line "every 15 seconds or so" **[verified here]** — and no numeric default anywhere. Three mechanisms exist across the field, one per reference, and most references document none. That spread is why this is companion guidance with no interval mandated. The stateless-frame recommendation is what makes ST-012's client clause safe. | `[COMPARATIVE]` — an authoring note in a Living Standard is explicitly not a protocol requirement | WHATWG HTML LS §9.2 (verified here); `survey-08` §4.3, §5.2; **S9** | moderate | Comp |
| **ST-017** | **SHOULD** | Per-stream metadata — stream identifiers, model or version names, usage totals, cursors — SHOULD be carried in the stream body rather than in response headers. | `Content-Type` is CORS-safelisted; every custom response header is not, so body-carried metadata is readable by a cross-origin browser client with no server opt-in while a header-carried value is invisible without an explicit `Access-Control-Expose-Headers` listing. Following this keeps `R4.17`'s list unchanged by Phase 6; not following it puts the header inside `R4.17`, which already binds. Either way the standard is consistent — this is the cheaper path. | `[INFERENCE]` from the WHATWG Fetch safelist, stated as guidance | WHATWG Fetch (`survey-08` §3.5); `R4.17`; **S11**, **S12** | moderate | Comp |
| **ST-018** | **SHOULD** | Browser clients SHOULD consume streams with a `fetch`-based reader or through a first-party relay that holds the credential server-side. `R8.2` forbids passing an access token in a query parameter, so a browser-direct `EventSource` connection cannot be authenticated under this standard. | `EventSource` cannot set request headers, so it cannot send `Authorization` or an API key; its only native credential is a cookie. The field's one workaround is a query-parameter key, which `R8.2` already forbids on BCP 240 grounds — the security consequence is stated, not blessed: such a token leaks into access logs, browser history, `Referer` headers, and shared URLs. No reference documents a browser-direct integration. The relay's own origin, not the vendor's, is where per-origin connection limits bind — a companion-level deployment fact with no vendor statement to cite. | `[POLICY]` — the consequence of ratified `R8.2`; the `EventSource` limitation itself is `[FACT]` about the WHATWG Living Standard's API | `R8.2`; WHATWG HTML LS §9.2; `survey-08` §4.8, §3.5; **S10**, **S11** | high | Comp |
| **ST-019** | **SHOULD** | Deployment guidance: intermediaries and CDNs that buffer responses defeat incremental delivery; response compression can add its own buffering; idle-timeout settings on proxies and load balancers must exceed the expected keep-alive interval. | Real and load-bearing operationally, and entirely outside the standard's normative surface: `X-Accel-Buffering`, the header commonly used to disable proxy buffering, is documented by nginx, is used widely for SSE, and is **absent from the IANA HTTP Field Name Registry** — 0 matches across 259 rows **[verified here]**. It fails the same registry test the standard applied to `Operation-Location`, so it may be described as infrastructure practice but must never be reserved or mandated. No reference API documents emitting it, and no reference states whether its streams are compressed. | `[COMPARATIVE]` at best; the registry absence is `[FACT]` | IANA HTTP field-name registry probe (verified here); `survey-08` §11.1 items 6 and 7 | moderate on practice, high on the registry fact | Comp |
| **ST-020** | **MAY** | Where frame sizes could reveal sensitive information to an on-path observer, an API MAY pad frames to normalize payload sizes, and SHOULD document the behaviour and any opt-out. | A threat unique to incremental delivery: under TLS, frame sizes still leak token lengths to an observer. One reference addresses it — OpenAI's `include_obfuscation`, on by default — and no standards document examined mentions it. One data point is not a default, so this is a MAY with the mechanism described; §8 already governs credential handling and redaction, and this adds a streaming-specific consideration rather than a rule. | `[COMPARATIVE]` — single-reference practice, labelled as such | `survey-08` §5.1 (streaming-specific security control) | low-moderate — one reference | Comp |

### 4.3 Coverage of the Contested Axes Register

Every `survey-08` axis maps to a principle or carries a stated reason for
carrying none.

| Axis | Disposition |
| --- | --- |
| **S1** Negotiation mechanism | ST-002 (`Accept`, or a distinct always-streaming resource), ST-003 (`stream` reservation and rejection guard). Body flag declined, §5. |
| **S2** Framing and media type | ST-001 (accurate self-delimiting media type declared; no `application/json` mislabel), ST-004 (SSE default), ST-013 (NDJSON as MAY). |
| **S3** IANA registration | ST-004's disclosure duty and the §1.10 row proposed in §3.4; the registry test is worked through in §5. |
| **S4** Event typing | ST-005 (normative typing), ST-014 (dual carriage, companion). |
| **S5** Termination | ST-006, including the trailing-sentinel allowance. |
| **S6** Post-commit error shape | ST-007. The whole of §3.1. |
| **S7** Errors before versus after the first byte | ST-008 makes the two-shape reality explicit rather than leaving it undocumented — the axis's own finding was that only one reference states it. |
| **S8** Resumption | ST-010. `Last-Event-ID` declined, §5. |
| **S9** Keep-alive | ST-016 (companion) plus ST-012's client clause. **No §13 rule**, because the only available authority is an authoring note in a Living Standard and the field shows three mechanisms across three references — there is nothing to make normative. |
| **S10** Browser authentication | ST-018 (companion). **No new §13 rule**, because ratified `R8.2` already decides it (§3.6). |
| **S11** CORS exposure | ST-017 (companion). **No §13 rule**, because `R4.17` already binds any header a provider chooses to emit (§3.3). |
| **S12** Final-chunk metadata | ST-017 covers placement. **No §13 rule on shape**: four shapes across six references with no dominant pattern, and the choice has no interoperability consequence once frames are typed (ST-005) and the terminal frame is defined (ST-006). |
| **S13** Streaming versus an operation resource | ST-009. AIP-151 declined, §5. |
| **S14** HTTP-version phrasing | ST-015 (companion), and ST-001's media-type phrasing is the version-neutral form in the normative text. |

---

## 5. Declined options

Written in the shape the decision layer can cite directly: option, reason, and
what would reopen it.

**Request-body flag (`"stream": true`) as the primary negotiation mechanism.**
Declined although it is what every deep-dive provider ships (S1 option (a)). It
makes the response's media type depend on the request body, which is not content
negotiation, so it collides with ratified `R4.10`; S1's own trade-off column
records that it "breaks content negotiation and GET-ability"; and it has no
rejection guard, so an endpoint that ignores an unimplemented flag reproduces
exactly the silent-real-execution hazard that disqualified `Prefer:
validate-only`. Permitted as an **additional** mechanism under ST-003's guard,
never as the only one. *Reopens if:* the owner rules field compatibility above
`R4.10` composition (§7, decision 1).

**Query parameter (`alt=sse`, `?stream=true`) as the negotiation mechanism.**
Declined on the same rule: `R4.10` forbids a query parameter for media-type
selection by name. Gemini's own machine-readable discovery document does not
contain `alt=sse` at all, so tooling generated from the contract cannot know the
parameter exists — a live interoperability failure, not a style objection.
Query parameters remain correct for things that are *not* media-type selection,
such as a resumption position (ST-010).

**`Accept: text/event-stream` — declined as a *sufficient* mechanism on its
own.** The WHATWG HTML Living Standard makes it optional for `EventSource`:
"User agents **may** set (`Accept`, `text/event-stream`)" **[verified here]**.
A server therefore cannot rely on its presence to detect a browser client. This
is why ST-002 requires `Accept` only where one endpoint serves both shapes, and
permits a distinct always-streaming resource, which needs no header at all.

**`application/json-seq` (RFC 7464) as the required media type.** Declined
despite being the only IANA-registered option — probe returns 200 **[verified
here]**. Zero HTTP adoption across thirteen references; its registration names
command-line tools as implementations, not HTTP APIs; no browser parses it; and
requiring it would make this standard prescribe a wire format nobody ships,
which is the failure mode the repository's descriptive discipline exists to
avoid. Left available as a MAY (ST-013) for providers who need a registered
name. *Reopens if:* any reference ships it over HTTP (`survey-08` §11.3 item 2).

**`application/json` for a stream of concatenated JSON documents.** Declined and
prohibited (ST-001). It uses only registered names and is a false statement
about the body — the body is not a JSON text, and a conforming parser fed the
whole body fails. Kubernetes and Docker both do this; it is the clearest case in
the survey of registered-but-wrong beating unregistered-but-accurate, and the
standard picks accuracy.

**Trailer fields for the post-commit error.** Declined on the authority of the
RFC that defines them: RFC 9110 §6.5.1, "a server SHOULD NOT generate trailer
fields that it believes are necessary for the user agent to receive"
**[verified here]**, plus "in most cases, the trailers are simply discarded" and
the requirement for an explicit framing mechanism, which in HTTP/1.1 means
chunked coding and therefore reintroduces the version-specificity ST-015
avoids. Zero of thirteen references use trailers. This is not an oversight in
the field; it is what the specification advises.

**An in-band problem document carrying the would-be status (the AWS shape).**
Declined. RFC 9457 §3.1: "Generators MUST use the same status code in the actual
HTTP response" **[verified here]**. An object carrying `"status": 503` inside a
`200` violates a Standards Track MUST the moment it is called a problem
document. AWS Bedrock ships exactly this (424, 429, 500, 503 as in-band data
labels), which is why the survey called it the axis in its purest form — and why
this standard cannot copy it. ST-007 omits `status` instead. An API that needs
the coarse retryability signal has it in the `R5.16` catalog, where retryability
is documented per code.

**Amending `R5.12` to permit a private in-band error schema.** Declined. It
would surrender the identity machinery (`type`/`code`, the catalog, one client
handler) that `R5.12` exists to buy, in exchange for matching a field practice
that is unanimous only in *not* having solved the problem — seven of seven
references ship a private schema with no registered identity. The carve-out in
ST-007 is narrower and keeps the machinery.

**`Operation-Location`-style new headers for stream metadata.** Declined before
being proposed. The IANA-registry test applies, `baseline-02i` declined
`Operation-Location` on exactly those grounds, and ST-017 removes the need by
putting metadata in the body. `X-Accel-Buffering` is declined as a reserved name
for the same reason plus RFC 6648: absent from the IANA HTTP Field Name Registry
across 259 rows **[verified here]**. It may be described in the companion as
infrastructure practice; it may not be reserved or mandated.

**SSE `id:` and `Last-Event-ID` as the required resumption mechanism.**
Declined. **No reference in the survey emits `id:` or honours `Last-Event-ID`**
(S8, verified negative) — the mechanism's own framing is unused by everyone who
has adopted the framing. It is an opaque string with no ordering semantics, the
reconnection interval is implementation-defined with no numeric default in the
specification **[verified here]**, and it cannot express "resume this stored
response" as distinct from "send me changes since." ST-010 requires a
monotonically ordered position instead. An API MAY additionally mirror its
position into `id:` for browser clients; if it does, honouring `Last-Event-ID`
must be equivalent to the documented parameter, or the two paths diverge.
*Reopens if:* any reference implements it (`survey-08` §11.3 item 3).

**A `MUST` on resumption.** Declined for want of an argument. Two of thirteen
references implement it, and they resume different things — OpenAI a stored
artifact, Kubernetes a position in a change log — so a single mandatory
mechanism would describe neither. The prompt's own test applies: a MUST needs an
argument stronger than the mechanism's existence, and there isn't one.

**AIP-151's mutual exclusivity ("the response must not be a streaming
response").** Declined as a rule for this standard. It governs a protobuf-first
RPC surface with different apparatus; its prohibition is about *one method's
response type*, not about whether a capability may offer both shapes; and
Google's own Gemini product ships both, so it does not describe its author's
practice. `survey-08` §11.2 Conflict D adjudicates it the same way from the
descriptive side. ST-009 unifies instead, following the one shipped exemplar.

**A separate rule set for long-polling.** Declined. Long-polling differs from
incremental delivery in exactly one observable respect — an expired hold rather
than a truncated body — so it needs one rule (ST-011) and inherits everything
else from the general rules and §10. Its cursor discipline is `R6.3`/`R12.5`'s
already; its `202`/operation-resource relationship is `R10.1`'s already.

**A `204 No Content` convention for an expired long-poll hold.** Declined on two
independent grounds: `R5.7` already binds `204` to a successful DELETE, and the
WHATWG HTML Living Standard reserves `204` on this exact surface to mean "stop
reconnecting permanently" **[verified here]**. No reference uses it; Consul and
SQS both return `200`.

**Reserving a frame-type vocabulary (`message_stop`, `response.completed`, and
similar).** Declined as over-reach. The field's vocabularies are product-shaped
and diverge for good reasons; §1.10 reserves names where two APIs would
otherwise mean different things by the same word, and ST-005's requirement that
each API document its own vocabulary carries the interoperability weight. The
one exception proposed is the `error` frame type (§3.4), where all three
deep-dive providers independently converged.

---

## 6. Proposed §1.2 boundary text and `streaming` switch definition

Both artifacts are **proposals for the owner walk**, supplied because the prompt
requires them as output. They are not drafted standard prose in the sense
`research/CONSTRAINTS.md` forbids; the §13 rules themselves remain unwritten.

### 6.1 Proposed replacement for the §1.2 streaming deferral

Replacing the current sentence beginning "Streaming responses — SSE,
long-polling, and similar — are out of scope for this version by owner ruling at
Gate D."

> **Proposed text.** Responses delivered incrementally over a single HTTP
> request — Server-Sent Events, long-polling, and streaming HTTP bodies — are in
> scope and are governed by §13, which binds only while the `streaming`
> applicability switch is on. **WebSockets are an explicit non-goal.** After a
> `101` upgrade the exchange is no longer HTTP request/response, so no status
> code, media type, conditional request, problem document, or applicability
> switch defined by this standard can bind it; a standard that claimed otherwise
> would be describing a protocol it does not govern. The same reasoning excludes
> any other connection-upgrade mechanism, including raw upgraded TCP streams.
> Where an interaction is bidirectional and real-time, the field consistently
> leaves HTTP; this standard follows that boundary rather than legislating
> across it.

> Proposed provenance line: Gate D deferral closed by Phase 6 · WebSockets
> non-goal ruled by the owner (2026-08-10) · boundary reasoning from
> `survey-08` §7 · project policy.

The final sentence is grounded, not rhetorical: `survey-08` §7 records that no
reference offers WebSockets as an alternative encoding of an otherwise identical
unidirectional streaming capability — the split between HTTP streaming and
WebSockets tracks unidirectional versus bidirectional, consistently, across the
field.

### 6.2 Proposed `streaming` switch definition

Per `R1.6`, added to the §1.8 vocabulary, which becomes `webhooks` ·
`async-operations` · `bulk-operations` · `streaming`.

> **Proposed definition.** `streaming` is on for an API that delivers any
> response incrementally over a single HTTP request — Server-Sent Events, a
> long-poll hold, or a streaming body. It gates **ST-001, ST-002, ST-004
> through ST-012** (using provisional IDs until §13 rules are numbered).
> **ST-003 is deliberately outside the switch**: the rejection guard binds
> precisely the endpoints that do **not** stream, so gating it on `streaming`
> would disable it exactly where it is needed — the same structure as `R1.9`,
> which binds standard-wide rather than under a switch. **ST-009 additionally
> requires `async-operations` to be on**; with `async-operations` off there is
> no operation resource to compose with and the rule is vacuous. An API
> declaring `streaming` off carries the `R1.6` one-line reason and never opens
> the companion document.

`R1.6`'s "every switch controls at least one rule" test is satisfied (eleven
rules), and the vocabulary grows only because new rules need it.

---

## 7. Conflicts and open questions

Separated as the prompt requires: what more research could settle, versus what
is an owner policy decision. Only the latter goes to the ratification walk.

### 7.1 Owner decisions — these belong at the walk

**Decision 1 — Negotiation: ratified `R4.10` composition, or field
compatibility?** ST-002 requires `Accept`-based selection where one endpoint
serves both shapes. Every deep-dive provider uses a request-body flag, so an API
that wraps OpenAI, Anthropic, or Gemini semantics literally would be
nonconformant on this rule. *Options:* **(a)** adopt ST-002 as proposed —
`R4.10` composition wins, body flag permitted only as an additional mechanism
under ST-003's guard; **(b)** permit a body flag as a primary mechanism with an
explicit named deviation from `R4.10`, requiring the guard; **(c)** scope
`R4.10` so that streaming selection is exempt from it. *Recommendation:* **(a)**,
because it costs a conformance note entry for wrapper APIs and buys the `406`
guard, `Vary` correctness, and GET-ability for everyone else, while (c) weakens
a ratified rule for one case. *Precedent:* `R8.2` already overrules Gemini's
`key=` query parameter without anyone proposing to relax it.

**Decision 2 — the `R5.12` / `R5.13` carve-out.** §3.1 sets out the amendment in
full. *Options:* **(a)** carve out as proposed, with `status` omitted in-band;
**(b)** carve out but require the would-be status in a reserved extension member
instead — preserves AWS's demonstrated usefulness at the cost of a new reserved
name and a member whose relationship to RFC 9457's `status` a reader must be
told; **(c)** decline the carve-out and permit a private in-band schema —
declined in §5, listed here for completeness. *Recommendation:* **(a)**; the
retryability signal it gives up is already in the `R5.16` catalog. This decision
also settles whether `R5.13`'s member set is scoped in the same amendment.

**Decision 3 — the `R5.1` reading.** A stream that fails after commit returns
`200` for a failed operation; `R5.1` says a failed operation MUST NOT return
2xx. *Options:* **(a)** record the reading that `R5.1` binds the status to the
outcome as known when the status is generated, in the §13 preamble; **(b)** add
an explicit streaming scope to `R5.1`, which is a MINOR amendment across five
surfaces; **(c)** leave it and accept that a reviewer will find the collision.
*Recommendation:* **(b)** — this is precisely the "no silent deviation"
discipline `R1.7` enforces, and the reading is load-bearing enough to deserve
rule text rather than a preamble sentence. (c) is not viable.

**Decision 4 — the §1.10 reserved-media-type row for `text/event-stream`.**
Admitting an unregistered media type to a table whose three existing entries are
all RFC-registered. *Options:* **(a)** admit it with the disclosure footnote
pattern already used for `Idempotency-Key` and `RateLimit`; **(b)** keep the
table registered-only and carry `text/event-stream` in §13's rule text with the
disclosure. *Recommendation:* **(a)** — the reserved-name registry's purpose is
"same concept, same name, everywhere," which applies with equal force to an
unregistered name, and the disclosure pattern is established in the same
section. **Carried with this decision:** reserving the `error` frame-type name
(§3.4) opens a **fifth reserved-name category** in §1.10, alongside query
parameters, headers, media types, and action verbs. Recommended, on the same
"same concept, same name" ground and on three independent vendor convergences —
but it is a structural addition to §1.10 rather than a row in an existing
table, so it is surfaced here rather than assumed at drafting.

**Decision 5 — the resumption cursor's name and opacity.** ST-010 requires a
monotonically ordered position identifier but deliberately does not name the
request parameter. *Options:* **(a)** reuse the reserved `cursor` (§1.10),
inheriting `AC-013`'s opacity and `R12.5`'s "never construct, modify, or
persist" client obligation; **(b)** reserve a new name for stream position.
*The fork matters:* OpenAI's mechanism is a readable integer the client echoes
from `sequence_number`, and echoing is not constructing — but a monotonic
integer invites arithmetic, which `R12.5` forbids for cursors, while Kubernetes'
`resourceVersion` is a version token that behaves like a cursor. Reusing
`cursor` imports an opacity discipline that ST-010's own "monotonically ordered"
requirement partly contradicts. *Tentative lean:* **(b)**, a new reserved name,
because the contradiction is in the rule text rather than in anyone's judgment —
`R12.5` says a client must never construct or modify a cursor, and ST-010
requires a position with visible ordering, which is exactly the property that
makes construction tempting and detectable. A distinct name lets both
disciplines stay true, where reuse forces one of them to be qualified.
*Counterargument, and it is not weak:* two names for two nearly identical
client-side obligations is precisely the divergence §1.10 exists to prevent, and
a resumption position is opaque in every respect that matters — the client
echoes it and never interprets it. This remains an owner call on the §1.10
registry's shape, not a research finding; it is flagged rather than absorbed.

### 7.2 Flagged conflicts with ratified 1.0 rules — consolidated

| Ratified rule | Nature of the conflict | Where resolved |
| --- | --- | --- |
| `R5.12` | A mid-stream error cannot be served as an `application/problem+json` response | §3.1; Decision 2 — carve-out, MINOR |
| `R5.13` | Required member set includes `status`, which RFC 9457 §3.1 forbids in-band | §3.1; Decision 2 — scoped, MINOR |
| `R5.1` | A failed generation returns `200`, and `R5.1` forbids 2xx for a failed operation | §3.1 closing note; Decision 3 |
| `R4.10` | The rule composes cleanly but overrules unanimous vendor practice | §3.5; Decision 1 |
| §1.10 media-type table | Would admit its first unregistered entry | §3.4; Decision 4 |
| `R12.5` | Cursor opacity versus a monotonically ordered stream position | Decision 5 |
| `R8.2` | No conflict — but it forecloses browser-native `EventSource` authentication entirely, which is a capability consequence the owner should see stated | §3.6; ST-018 |

### 7.3 Questions further research could settle

1. **The actual response `Content-Type` of the three deep-dive providers'
   streams.** Undocumented by all three (`survey-08` §11.1 item 1); confirming
   it needs a live authenticated request. It affects nothing in ST-001, which
   states what a conforming API must do rather than what vendors do, but it
   would upgrade the survey's inference to a fact.
2. **Whether a second provider ships cursor resumption.** OpenAI is currently
   one data point among three deep-dive providers; a second would move ST-010
   from a conditional SHOULD toward a default.
3. **Whether the `text/event-stream` IANA registration is ever completed.** The
   WHATWG registration template has sat un-submitted, prefaced "will be
   submitted to the IESG for review, approval, and registration with IANA." The
   `httpapi` working group's docket was enumerated for this leaf and contains
   **no** streaming, SSE, or stream-media-type work item **[verified here]**, so
   there is no visible in-flight effort. Completion would dissolve ST-004's
   disclosure duty and the `Operation-Location` parallel.
4. **Long-poll hold-duration conventions beyond Consul and SQS.** The survey's
   Kubernetes `timeoutSeconds` row failed verification and is not relied on
   here; ST-011 requires the maximum hold to be *documented* rather than
   prescribing a number, which is what the thin evidence supports.

---

## 8. Overall confidence, invalidation conditions, and re-check trigger

**Overall confidence: moderate-high.** The composition analysis is the strongest
part — `R4.10`, `R8.2`, `R10.1`, and `R4.11` decide four questions by
themselves, and those results are as firm as the ratified rules. The
protocol-level constraints on the post-commit error are certain and
independently verified for this leaf: RFC 9457 §3.1, RFC 9457 Appendix C, and
RFC 9110 §6.5.1. What is genuinely this project's judgment, and should be read
as such, is the *resolution* among those constraints (ST-007), the decision to
let `R4.10` overrule unanimous practice (ST-002), and the conditional shape of
the resumption SHOULD (ST-010).

**Weakest links, named.**

- **ST-004 rests on an unregistered media type.** No amount of adoption converts
  a WHATWG Living Standard section plus a missing IANA row into a protocol
  requirement, which is why it is classified `[COMPARATIVE]` and carries a
  disclosure duty. This is the leaf's standing weakness and it cannot be
  engineered away.
- **ST-009 rests on one shipped exemplar.** OpenAI's unified design is the only
  reference that relates a stream and an operation resource at all; the contrary
  guideline (AIP-151) is declined with reasons, but a second exemplar would
  materially strengthen it.
- **ST-020 rests on one reference** and is a MAY for that reason.
- **ST-016 has no citable authority** beyond an authoring note, which is why no
  interval is proposed and the material is companion-only.

**Conditions that would invalidate the recommendation.**

1. **Completion of the `text/event-stream` IANA registration.** ST-004's
   disclosure duty and Decision 4 both dissolve; the `Operation-Location`
   parallel dissolves with them.
2. **A vendor emitting `application/problem+json` content inside a stream
   frame.** It would not change ST-007 — the `status` constraint is a protocol
   fact, not a practice observation — but it would move the axis from a
   unanimous negative to a two-sided split and would strengthen the proposal's
   evidence base considerably.
3. **Any reference shipping `application/json-seq` over HTTP.** It would convert
   the registered option from theoretical to demonstrated and would reopen the
   media-type call.
4. **Any reference emitting `id:` and honouring `Last-Event-ID`.** It would
   reopen ST-010's mechanism choice.
5. **An IETF working-group item on HTTP response streaming.** The `httpapi`
   docket was enumerated on 2026-08-10 and has none; an adopted draft would
   change the authority class available to §13 and could convert several
   `[COMPARATIVE]` principles toward `[FACT]`.
6. **A ruling on Decision 1 that reverses ST-002.** ST-003's guard becomes
   load-bearing rather than supplementary, and the §13 negotiation rule needs
   rewriting rather than adjusting.

**Non-invalidating:** further AI-provider examples of SSE framing, or further
examples of private in-band error schemas. Both findings are already unanimous;
additional instances change no proposed principle.

**Proposed re-check trigger for the register in `research/README.md`:**

> **2027-02-10 — streaming standards-footing re-check.** Re-probe the IANA
> media-type registry for `text/event-stream` (per-type URL, against a
> `text/html` control) and re-enumerate the IETF `httpapi` working-group docket
> for any HTTP response-streaming or stream-media-type work item. Fires the
> §13 review if either has changed. Rationale: ST-004 is the only proposed
> principle whose classification would change outright, and both probes are
> cheap and unambiguous. Baseline recorded 2026-08-10: registry probe 404
> against a 200 control; `httpapi` docket carries three active working-group
> drafts, none on streaming.

---

*End of report. Proposals only — nothing here binds `rest-api-standard.md`
until ratified in `research/decisions/baseline-04-streaming.decision.md`.*
