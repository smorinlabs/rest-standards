# baseline-04d — Caching posture for streams (prompt)

*Phase 8 research batch, dispatched 2026-08-10 as one of five narrow leaves
under `baseline-04`, each testing one interaction that §13.4 of
`rest-api-standard.md` records as recognized and not yet ruled. Run by an
opus research subagent with the repository evidence rules embedded. Filed
after the run, reconstructed from the dispatch.*

## Question

What caching posture should this standard require for a **streaming
response**?

## The concrete problem

`R7.1` requires an explicit `Cache-Control` on every response. `R7.3`'s
default tier for authenticated or mutable resources is `private, no-cache`
revalidating via the strong `ETag` machinery of `R3.10` — machinery a stream
appears unable to supply, because the body does not exist when headers are
sent. `R7.3` also calls blanket `no-store` a named anti-pattern, yet the
standard's own worked example E.11 sends `private, no-store` for a
non-sensitive export stream.

**Check first:** whether `R7.3` contains any BCP 14 keyword at all. That
determines whether this is a binding conflict or an unpicked default.

## Required coverage

1. **Mandatory deep-dive: OpenAI, Anthropic, Google Gemini.** What
   `Cache-Control` their streaming endpoints actually send. Say so plainly if
   it cannot be verified directly rather than guessing.
2. **The eight standard references** — Stripe, GitHub, Google/AIP,
   Microsoft/Azure, Twilio, Shopify, Zalando, AWS — wherever they stream.
   Non-participation is a dated finding.
3. **Comparable precedent:** Kubernetes watch, Elasticsearch, Docker events,
   SSE reference implementations; what server frameworks and reverse proxies
   (nginx, Envoy) default to or recommend for SSE and chunked responses.
4. **Standards layer:** RFC 9111 — is an incrementally-generated response
   addressed? What does it say about storing a response whose body is still
   arriving, and about `no-store` versus `no-cache`? Quote operative text.
   Also: can an `ETag` legitimately be a version-based validator computed
   *before* transmission, which matters when the stream views a retained
   artifact?

## Evidence both ways

FOR mandating `no-store` on streams, and AGAINST (streams that are cacheable
or revalidatable, the cost of blanket no-store, whether the existing three-tier
default already covers it).

## Also assess

Whether this needs a rule at all. "The existing rules already cover this, no
amendment needed" is a defensible finding — say so plainly if the evidence
supports it.

## Evidence rules

Direct URL, authority class, and access date on every material claim;
two-source minimum on load-bearing claims; primary sources first;
`[FACT]`/`[COMPARATIVE]`/`[INFERENCE]`/`[POLICY]` labels; conflicts surfaced,
never averaged. Public repo with push protection — placeholder credentials
only.

## Output

TL;DR and recommendation · standards-and-currency matrix · field evidence with
comparison tables · evidence for and against, separately · proposed rule text
or a reasoned "no rule needed", with classification and confidence · declined
alternatives · what could not be verified.
