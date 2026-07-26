# Baseline 01b — HTTP QUERY Deployment Reality

*Narrow leaf under `baseline-01`. All sources verified 2026-07-25. Answers one
question: should `HS-009` remain `MAY`?*

## Verdict

**`HS-009` stays `MAY`. Do not promote to `SHOULD`.**

The reason is precise: protocol-level plumbing exists, but **the one capability
that makes QUERY worth choosing over POST — cacheable responses keyed on the
request body — has no confirmable production implementation anywhere.**

## Capability matrix

RFC 10008 requires three distinct things, routinely conflated. Only (c)
delivers QUERY's advantage.

| Implementation | (a) Parses | (b) Passes through | (c) **Body-keyed caching** | Source class |
| --- | --- | --- | --- | --- |
| llhttp (Node.js parser) | ✅ `HTTP_QUERY = 46` | n/a | n/a | **Primary** — enum in `nodejs/node` header |
| curl | ✅ via `-X QUERY` | n/a | n/a | **Primary** — curl author's blog |
| Spring Framework | ❌ PR open, blocked | ❌ | ❌ | **Primary** — PR #34993 |
| Browsers / Fetch | ❌ not documented | ❌ | ❌ | **Primary** — MDN omits QUERY entirely |
| Cloudflare | unconfirmed | unconfirmed | **❌ no announcement found** | absence of primary evidence |
| Akamai | unconfirmed | unconfirmed | **❌ no announcement found** | absence of primary evidence |
| Fastly / CloudFront | unconfirmed | unconfirmed | unconfirmed | no evidence located |

## Findings

1. **`[FACT]` llhttp defines `HTTP_QUERY = 46`.** Verified directly in
   `nodejs/node/deps/llhttp/include/llhttp.h`. Node's parser accepts the
   method. Corroborated by llhttp PR #265 and node issue #51562 (closed).
2. **`[FACT]` curl supports QUERY only through the generic `-X` flag.** Daniel
   Stenberg (curl's author) documents `curl -d "data" -X QUERY <url>` and
   describes no dedicated option. He notes a redirect caveat: use `--follow`
   rather than legacy `-L`, which "changes the HTTP method on all subsequent
   requests independently of what the server responds."
   Source: https://daniel.haxx.se/blog/2026/06/21/query-with-curl/
3. **`[FACT]` Spring Framework does not support QUERY.** PR #34993 is **open
   and labeled `status: blocked`**, submitted 2025-06-03, marked ready for
   review 2026-06-15 after RFC approval, locked to collaborators 2026-07-08.
   Maintainer `bclozel`: *"we would like to support this in time for Spring
   Framework 7.1 in November."* Target is therefore **November 2026** — a
   stated intent, not shipped code.
4. **`[FACT]` MDN does not document a QUERY method.** Its HTTP methods
   reference lists GET, HEAD, POST, PUT, DELETE, CONNECT, OPTIONS, TRACE, and
   PATCH — and nothing else. No browser exposes QUERY via `fetch()`.
5. **`[FACT — absence]` No CDN announcement of QUERY caching could be located.**
   Two independent searches against Cloudflare, Akamai, Fastly, and CloudFront
   produced no vendor announcement that any edge cache handles QUERY today.
   A secondary source independently reached the same conclusion, noting that
   despite two of the three authors' CDN affiliations, no announcement from
   either could be found.

## On the authorship signal

`[INFERENCE]` RFC 10008's authors are J. Reschke (greenbytes), J. M. Snell
(**Cloudflare**), and M. Bishop (**Akamai**) — verified from IETF Datatracker.
This is frequently cited as evidence that CDN support is imminent.

It is not evidence of deployed support. It is evidence of **design intent**:
the organizations most exposed to cache behavior helped shape the spec. Those
are different claims, and conflating them would put a `SHOULD` on infrastructure
that demonstrably does not exist yet. The authorship signal is a reason to
expect support, and a reason to re-check — not a reason to recommend.

## What would change the verdict

A single primary announcement from any major CDN that its edge cache serves
QUERY responses using a body-derived cache key. That would move `HS-009` to
`SHOULD` immediately, since every other layer is either ready or trivially
ready.

**Re-check trigger:** Spring Framework 7.1 (targeted November 2026) is the
nearest dated milestone. Re-run this leaf then.

## Sources

- https://datatracker.ietf.org/doc/rfc10008/
- https://github.com/nodejs/node/blob/main/deps/llhttp/include/llhttp.h
- https://github.com/nodejs/node/issues/51562
- https://daniel.haxx.se/blog/2026/06/21/query-with-curl/
- https://github.com/spring-projects/spring-framework/pull/34993
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods
