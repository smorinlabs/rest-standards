# Streaming profile — companion to §13

**Informative. Nothing here is normative.** The rules are §13 of
[`rest-api-standard.md`](rest-api-standard.md); where this document restates
one, the rule governs. Requirement keywords appear here in code font
(`MUST`, `SHOULD`, `MAY`) to make clear that they are references to
obligations defined elsewhere, never obligations created here. The
authoritative rationale, evidence, and declined options are in
[`research/decisions/baseline-04-streaming.decision.md`](research/decisions/baseline-04-streaming.decision.md).

This document exists so §13 can stay short. Most APIs do not stream; those
that declare the `streaming` applicability switch off never need to read
past §13's first paragraph. Everything an implementer actually needs — how
the mechanisms differ, what the wire looks like, why the rules landed where
they did, and what to do about proxies — is here.

**Version 1.1.0**, added with §13. Evidence current as of **2026-08-10**.

## Contents

1. [Which mechanism, and when](#1-which-mechanism-and-when)
2. [Server-Sent Events on the wire](#2-server-sent-events-on-the-wire)
3. [Newline-delimited JSON and the registered alternative](#3-newline-delimited-json-and-the-registered-alternative)
4. [Framing in media-type terms, not chunked terms](#4-framing-in-media-type-terms-not-chunked-terms)
5. [Why the error rules are shaped the way they are](#5-why-the-error-rules-are-shaped-the-way-they-are)
6. [Keep-alive, and why no interval is required](#6-keep-alive-and-why-no-interval-is-required)
7. [Browser clients, CORS, and the `EventSource` dead end](#7-browser-clients-cors-and-the-eventsource-dead-end)
8. [Metadata placement](#8-metadata-placement)
9. [Deployment: proxies, buffering, compression, timeouts](#9-deployment-proxies-buffering-compression-timeouts)
10. [Frame-size leakage](#10-frame-size-leakage)
11. [What the field actually does](#11-what-the-field-actually-does)
12. [Registration status, stated plainly](#12-registration-status-stated-plainly)

---

## 1. Which mechanism, and when

Three mechanisms are in scope, and they are not interchangeable. Picking by
"what the response is" rather than "what technology is fashionable" resolves
almost every case.

| The response is… | Use | Why |
| --- | --- | --- |
| Generated progressively, and a partial answer is useful | Server-Sent Events (`ST-004`, R13.4) | Native browser parsing, named frame types, an existing ecosystem |
| A large finite record set produced faster than it can be sent | Newline-delimited JSON (`ST-013`) | No per-frame ceremony; the consumer is a program, not a browser |
| Not ready yet, and the client wants to wait rather than poll hot | Long-polling (`ST-011`, R13.11) | One connection per wait, no framing protocol at all |
| Not ready yet, and the wait may be long or the client may disconnect | Not streaming — an operation resource (§10) | The state survives the connection |

**WebSockets are outside the standard entirely** (§1.2). This is not a gap
left for later. After a `101 Switching Protocols` upgrade, the exchange is no
longer HTTP request/response: there is no status code, no response media
type, and no request for a conditional header, a problem document, or an
idempotency key to attach to. Every mechanism the standard uses to make
guarantees is absent. An API may offer a WebSocket surface; the standard
simply says nothing about it.

**Streaming is not a substitute for an operation resource.** If the work
outlives the connection, or the client may reasonably disconnect and come
back, the durable answer is §10's operation resource. Streaming is a delivery
channel over work that is happening now. Where both exist, R13.9 binds them
into one identity — see [§5](#5-why-the-error-rules-are-shaped-the-way-they-are).

## 2. Server-Sent Events on the wire

SSE is defined in the WHATWG HTML Living Standard, not in an RFC. A stream is
a sequence of records separated by blank lines; each record is a set of
`field: value` lines.

```
event: export.chunk
id: 42
data: {"rows":500}

: this is a comment, used as a keep-alive

event: export.completed
data: {"status":"succeeded"}

```

Four fields are defined, and one line form is not a field:

| Line | Meaning |
| --- | --- |
| `event:` | The frame's type. A client using `EventSource` registers listeners per type. Absent means the default type, `message`. |
| `data:` | The payload. **Repeated `data:` lines are concatenated with newlines**, which is how a multi-line JSON body travels. |
| `id:` | The last-event identifier. A browser `EventSource` will send it back as `Last-Event-ID` on reconnect. |
| `retry:` | Reconnection delay in milliseconds, for `EventSource`'s automatic reconnect. |
| `: text` | A comment. Ignored by every parser, which is what makes it usable as a keep-alive. |

Three properties surprise implementers:

- **The blank line is load-bearing.** A record is not dispatched until a blank
  line arrives. A server that forgets the trailing blank line produces a frame
  the client never sees.
- **UTF-8 only.** The specification requires it; there is no charset
  negotiation.
- **A byte-order mark is stripped** if present, once, at the start.

**Why the standard requires typed frames** (`ST-005`, R13.5) rather than
leaving typing to the payload: without a per-frame type, a client must parse
every payload just to learn what it received, and cannot route, ignore, or
count frames. Carrying the type in **both** the `event:` field and a payload
member (`ST-014`) serves both consumer shapes — a browser `EventSource`
registering per-type listeners, and a `fetch`-based reader that splits on
`data:` and never implements SSE parsing at all. The cost is two places that
can disagree, which is why dual carriage is `SHOULD`-strength guidance here
rather than a rule.

**Why a terminal frame is required** (`ST-006`, R13.6): "did I get
everything?" cannot be answered from connection close alone. A terminal typed
frame carries the outcome. A bare sentinel such as `data: [DONE]` carries
none and is not valid JSON, so a uniform `JSON.parse` over `data:` lines
throws on it. Because at least one shipped API emits a terminal event **and
then** a sentinel, R13.6 permits a trailing sentinel and requires clients to
tolerate one — a rule demanding exactly one terminal frame would outlaw a
working design for no benefit.

## 3. Newline-delimited JSON and the registered alternative

For record streams and bulk result sets, one JSON document per line, no
framing beyond the newline:

```
{"id":"ord_1","total":4599}
{"id":"ord_2","total":1250}
```

Served as `application/x-ndjson` (`ST-013`). Two cautions belong in the API's
own documentation:

- **`application/x-ndjson` is not registered with IANA**, and it has no
  standards body of any kind — the NDJSON specification is a GitHub
  repository, last updated 2014.
- **RFC 6838 §3.4** notes that names beginning with `x-` "are no longer
  considered to be members of this tree," and RFC 6648 discourages creating
  new `x-` prefixed names. Neither retroactively prohibits using an existing
  convention, which is why `ST-013` permits it with disclosure rather than
  banning it.

`application/json-seq` (RFC 7464) **is** registered, and is the honest answer
when a registered media type is a hard requirement — for example when a
gateway or contract-validation tool rejects unregistered types. It is not the
default because it has no adoption among surveyed APIs and no client
ecosystem; it uses an ASCII record separator before each document, which most
JSON tooling will not produce or consume without custom code.

**Never label a concatenated-document stream `application/json`** (R13.1). A
conforming JSON parser handed the whole body fails, so the label is a false
statement about the payload — and it is precisely the practice the two
line-delimited media types exist to correct.

## 4. Framing in media-type terms, not chunked terms

It is tempting to specify a streaming contract as "the response uses
`Transfer-Encoding: chunked`." Do not (`ST-015`).

Chunked transfer coding is defined in RFC 9112 §7.1, and RFC 9112 §6.1
scopes the mechanism that carries it: "Transfer-Encoding was added in
HTTP/1.1," and "A server `MUST NOT` send a response containing
Transfer-Encoding unless the corresponding request indicates HTTP/1.1 (or
later minor revisions)."

Over HTTP/2 it is not merely unused but forbidden. RFC 9113 lists
`Transfer-Encoding` among the connection-specific header fields, and "Any
message containing connection-specific header fields `MUST` be treated as
malformed." HTTP/2 and HTTP/3 frame the body themselves, with DATA frames.

Guidance phrased in chunked terms is therefore silently inapplicable — or
actively wrong — over the transports most streaming APIs actually run on.

The version-neutral observable properties are the **media type** and the
**absence of `Content-Length`**. Those hold on every HTTP version, which is
why R13.1 is written about the media type and says nothing about transfer
coding.

## 5. Why the error rules are shaped the way they are

This is the part of §13 most likely to look arbitrary. It is not; it is
squeezed between two published requirements with almost no room left.

**The problem.** `R5.12` requires every application-generated error to be
servable as `application/problem+json`. Once `200 OK` and its headers are on
the wire, the status code cannot be revised — and a generation that fails
halfway has already committed one.

**Why not put the real status in the problem object?** RFC 9457 §3.1: the
`status` member "if present, is only advisory," and "Generators `MUST` use
the same status code in the actual HTTP response, to assure that generic HTTP
software that does not understand this format still behaves correctly." A
problem document carrying `status: 503` inside a `200` response therefore
violates a Standards Track requirement. The same sentence's "if present"
makes **omitting** the member legitimate, which is the option R13.7 takes.

**Why not trailer fields?** They look purpose-built — metadata after the
body — and they do not work. RFC 9110 §6.5.1: "in most cases, the trailers
are simply discarded," and a server "`SHOULD NOT` generate trailer fields that
it believes are necessary for the user agent to receive." Browsers do not
expose them through `fetch`. No surveyed API uses them.

**What is left** is in-band delivery, which is what every surveyed API
independently arrived at — though each with a private schema carrying no
registered identity. R13.7 keeps the industry's mechanism and adds back what
the private schemas gave up: a `type` and `code` bound by `R5.13`'s template,
listed in the `R5.16` catalog, so one client error handler covers both
channels and generic tooling can recognize the failure.

**Two error shapes, and that is correct.** An error *before* the status is
committed is an ordinary problem response — `422`, `404`, whatever
applies — regardless of what the request asked to stream (`ST-008`, R13.8).
An error *after* is an `error` frame. Clients need both paths. The clean
illustration is Kubernetes: a stale cursor detected before the stream opens
yields a real `410 Gone`; the same condition after it opens yields an in-band
event.

**`R5.1` is not violated.** "A failed operation `MUST NOT` return 2xx" binds
the status to the outcome *as known when the status is generated*. That scope
is stated in `R5.1` itself, so a reader of §5 alone sees it.

**Where the full document lives.** When an operation resource also exists,
R13.9 requires it to serve the complete problem document — with `status`, as
a real `application/problem+json` response. That is the out-of-band half of
R13.7's carve-out: the operation resource is the one place a status code is
still available to be generated.

## 6. Keep-alive, and why no interval is required

Idle intermediaries close connections. A stream that produces nothing for
minutes at a time needs traffic to survive.

SSE offers the cheapest mechanism: a comment line, which every parser
discards.

```
: keep-alive

```

The WHATWG text suggests "every 15 seconds or so" — but that is an **authoring
note in a living standard**, not a protocol requirement, and no numeric
default exists anywhere else. Across the surveyed field there are three
different mechanisms, roughly one per API that does anything at all, and most
document none.

That is why `ST-016` asks only that an API **document** whether it emits
keep-alives and in what form, and mandates no interval. It also asks that a
keep-alive frame carry no application state — which is what makes R12.10's
client obligation safe: a client that discards keep-alives must lose nothing
by doing so.

**Clients must not depend on keep-alive timing** (R12.10). No API surveyed
publishes a guaranteed interval, so a client treating a missed keep-alive as
a failure signal is building on something no provider promised.

## 7. Browser clients, CORS, and the `EventSource` dead end

`EventSource` is the browser's native SSE client. It reconnects
automatically, handles `Last-Event-ID`, and dispatches per-type listeners.
It also **cannot set request headers** — no `Authorization`, no API key
header. Its only native credential is a cookie.

The field's workaround is a token in the query string. **The standard already
forbids that**: `R8.2` says access tokens `MUST NOT` be accepted or emitted in
URI query parameters, on BCP 240 grounds. The consequence is stated rather
than engineered around: **a browser-direct `EventSource` connection cannot be
authenticated under this standard.**

A query-string token is not merely inelegant. It lands in server access logs,
browser history, `Referer` headers on any outbound link, and any URL a user
copies and shares.

Two paths remain (`ST-018`):

- **A `fetch`-based reader.** `fetch` sets headers freely, so `Authorization`
  works. The cost is implementing SSE parsing — splitting on blank lines,
  concatenating repeated `data:` lines, handling reconnection — since the
  browser only does that inside `EventSource`.
- **A first-party relay.** The browser connects to your own origin, which
  holds the credential server-side. Note that per-origin connection limits
  then bind at the relay's origin, not the upstream provider's.

**Connection limits.** Under HTTP/1.1 browsers cap concurrent connections per
origin at around six, and a held-open stream occupies one for its lifetime —
a handful of streams can starve ordinary requests. HTTP/2 multiplexes and
largely dissolves the problem.

**CORS.** `Content-Type` is CORS-safelisted, so a cross-origin browser client
can read it with no server opt-in. **Every custom response header is not**:
without an explicit `Access-Control-Expose-Headers` listing, a cross-origin
reader cannot see it at all. That fact drives the next section.

## 8. Metadata placement

Put per-stream metadata — stream identifiers, model or version names, usage
totals, cursors — **in the stream body, not in response headers**
(`ST-017`).

The reason is mechanical, not aesthetic. A body-carried value is readable by
a cross-origin browser client with no server configuration. A header-carried
value is invisible unless the server lists it in
`Access-Control-Expose-Headers` — and `R4.17` already requires exactly that
listing for standard-bound headers, so a header choice pulls the value into
`R4.17`'s scope and adds a configuration step that can be forgotten.

Following the guidance leaves `R4.17`'s list unchanged by streaming. Either
path is conformant; this one is cheaper.

The field offers no dominant shape for *where in the body* — a dedicated
final frame, a field on the terminal frame, or a running cumulative field all
ship. The standard deliberately takes no position, because once frames are
typed (R13.5) and the terminal frame is defined (R13.6), the choice has no
interoperability consequence. Be aware that a cumulative running total
invites double-counting if a client sums frames instead of reading the last
one.

## 9. Deployment: proxies, buffering, compression, timeouts

Streaming is the one part of an HTTP API that infrastructure can silently
break while every test passes (`ST-019`). Three failure modes account for
nearly all of it:

**Buffering.** A reverse proxy or CDN that buffers responses will hold frames
until its buffer fills or the response ends — turning a stream into a slow
non-stream, with no error anywhere. nginx documents `X-Accel-Buffering: no`
to disable this per response, and it is widely used for SSE.

`X-Accel-Buffering` is **absent from the IANA HTTP Field Name Registry** — a
probe on 2026-08-10 found zero matches across 259 rows. It fails the same
registry test that led this standard to decline `Operation-Location`, so it
is described here as infrastructure practice and is deliberately **not**
reserved in §1.10 and not mandated anywhere.

**Compression.** `Content-Encoding: gzip` over a stream can introduce its own
buffering, since a compressor may wait for enough input to emit a block. Test
the deployed path with compression enabled rather than assuming, and note
that no surveyed API documents whether its streams are compressed.

**Idle timeouts.** Load balancers, proxies, and application servers each have
one. Every one of them must exceed the expected keep-alive interval, or
working streams die on a timer. This is usually where "it works locally"
diverges from production.

## 10. Frame-size leakage

A threat unique to incremental delivery: under TLS, an on-path observer
cannot read frame contents but **can** see frame sizes and timing. Where
frames correspond to units of generated content, sizes can leak information
about what was generated.

`ST-020` permits padding frames to normalize sizes, with the behavior and any
opt-out documented. It is `MAY`-strength because exactly one surveyed API
addresses it at all and no standards document examined mentions it — one data
point does not make a default. §8 already governs credential handling and
redaction; this is a streaming-specific consideration on top.

## 11. What the field actually does

Recorded 2026-08-10 from
[`research/reports/survey-08-streaming.report.2026-08-10.md`](research/reports/survey-08-streaming.report.2026-08-10.md),
which carries the citations. Vendor behavior changes; treat this as dated
evidence, not current fact.

**Negotiation is wide open.** Four mechanisms are in live use — a request-body
flag, a distinct method or endpoint name, a query parameter, and the `Accept`
header — and one vendor uses three of them. No surveyed vendor uses `Accept`;
the only thing that does is RFC 8895, a Standards Track RFC. R13.2 chooses
`Accept` anyway, because `R4.10` already governs media-type selection and
already forbids the query-parameter form. An API mirroring a vendor's
body-flag design is nonconformant on R13.2 and records that in its
conformance note under `R1.7`.

**Framing has split cleanly by job.** SSE for generated content, JSON Lines
for bulk result sets. The two do not compete.

**Termination has no consensus.** Sentinel, terminal typed event, both, or
connection close alone — all four ship, and one vendor uses different
patterns in two of its own APIs.

**Post-commit errors are a unanimous negative.** Across seven streaming
references, **none** emits `problem+json` in-band and **none** uses trailer
fields. Every one uses a private schema. One vendor states the situation
outright in its own documentation; another carries HTTP status codes as data
labels inside a `200`.

**Resumption exists, but never through SSE's own mechanism.** Two references
implement it — one over a stored artifact with a required sequence number and
roughly a ten-minute retention window, one over a change log with a history
window and a defined out-of-window error. **No reference emits `id:` or
honors `Last-Event-ID`.** The specification's own mechanism is universally
unused, which is why R13.10 requires a position identifier without requiring
that mechanism.

**Nobody documents browser integration.** Not one surveyed reference
addresses browser-direct consumption or CORS exposure for streams, in either
direction.

## 12. Registration status, stated plainly

Because the standard requires APIs to disclose this (R13.4), here it is in
one place. All probes 2026-08-10.

| Media type | Registered? | Evidence |
| --- | --- | --- |
| `text/event-stream` | **No** | Absent from the 105-entry IANA `text/*` subregistry; per-type registry URL returns `404` while a `text/html` control returns `200`. The WHATWG text carries a registration template prefaced "will be submitted to the IESG for review, approval, and registration with IANA" — the submission has not happened |
| `application/x-ndjson` | **No** | No registry row. Defined only by a GitHub repository, last updated 2014 |
| `application/json-seq` | **Yes** | Registered via RFC 7464 — and used over HTTP by no surveyed API |

Two further facts worth stating together, because they are easy to get
backwards:

- **There is a Standards Track RFC built on SSE.** RFC 8895 (November 2020)
  uses `Accept: text/event-stream` and `Content-Type: text/event-stream`
  directly. So "SSE has no standards-track footing" is too strong.
- **That RFC assumes the media type is registered, and it is not.** RFC 8895
  §12 states that all media types it uses other than the two it defines "have
  already been registered." The IANA registry disagrees. The registry governs
  on registration status: an RFC registers a media type only through its own
  IANA Considerations section, and RFC 8895's registers only the two types it
  defines. The conflict is recorded rather than smoothed over.

A third: RFC 8895's normative reference for SSE is the **W3C Recommendation
of February 2015**, which W3C has since marked obsolete; its URL now redirects
to the WHATWG HTML standard. Any citation of "the SSE specification" should
name which document it means. This one means the WHATWG HTML Living Standard.

**If the registration ever completes**, R13.4's disclosure duty and the
§1.10 row's caveat both dissolve. A dated re-check is registered for
**2027-02-10** in
[`research/README.md`](research/README.md).
