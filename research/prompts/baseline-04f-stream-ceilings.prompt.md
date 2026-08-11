# baseline-04f — Resource ceilings for streams (prompt)

*Phase 8 research batch, dispatched 2026-08-10 as one of five narrow leaves
under `baseline-04`, each testing one interaction that §13.4 of
`rest-api-standard.md` records as recognized and not yet ruled. Run by an
opus research subagent with the repository evidence rules embedded. Filed
after the run, reconstructed from the dispatch.*

## Question

What **resource ceilings** should a streaming API publish and enforce?

## The concrete problem

`R11.1` requires published and enforced maxima for page size, expansion depth,
and bulk item count. A held-open stream is arguably the largest unbounded
resource commitment an HTTP API makes, and it is in none of those three
dimensions. No required maximum stream duration; no required per-principal
concurrent-stream ceiling.

## Check the claimed gap before concluding anything is missing

A prior review flagged the §13.4 register entry as overstated. Determine, with
quoted rule text, which parts of the gap are real:

- `R8.10`'s rate-limit axis default mentions a published posture with
  "concurrency separate" — does that already require a published concurrency
  ceiling?
- `R6.5` requires each collection to document its default and maximum `limit`,
  and the Phase 6 walk did not scope `R6.5` — does a *streamed collection*
  therefore already owe a published maximum?

## Required coverage

1. **Mandatory deep-dive: OpenAI, Anthropic, Google Gemini.** Published
   maximum request or stream duration; concurrent-connection or
   concurrent-request limits per key or organization; how streaming requests
   are counted against rate limits.
2. **The eight standard references**, for concurrency and duration limits on
   long-lived connections. Non-participation is a dated finding.
3. **Comparable precedent, with real numbers:** Kubernetes watch
   `timeoutSeconds` and its randomization — **verify directly in current
   documentation or source, because a prior leaf recorded this unverified**;
   Consul blocking queries; AWS SQS long polling; SSE per-origin browser
   connection limits under HTTP/1.1 versus HTTP/2.
4. **Standards and security layer:** OWASP API4:2023 unrestricted resource
   consumption — does it address long-lived connections? Any published
   guidance on bounding streaming duration or concurrency.

## Evidence both ways

FOR requiring published maximum duration and concurrency ceilings, and
AGAINST (unbounded streams are legitimate — watches, event tails; a duration
cap may harm valid use; whether the ceiling belongs in the standard or in
deployment configuration).

## Assess separately

A **documentation** duty (publish your limits) versus an **enforcement** duty
(have limits). `R11.1` requires both; `R13.11` requires documentation only for
long-poll hold. Which is right here, and should unbounded streams be exempt
when documented as unbounded per `R13.6`?

## Evidence rules

Direct URL, authority class, and access date on every material claim;
two-source minimum on load-bearing claims; primary sources first;
`[FACT]`/`[COMPARATIVE]`/`[INFERENCE]`/`[POLICY]` labels; conflicts surfaced,
never averaged. Verify rather than inherit a prior leaf's unverified number.
Public repo with push protection — placeholder credentials only.

## Output

TL;DR and recommendation · standards-and-currency matrix · which parts of the
claimed gap are real versus already covered, with quoted rule text · field
evidence with comparison tables and concrete numbers · evidence for and
against, separately · proposed rule text distinguishing documentation from
enforcement, with classification and confidence · declined alternatives ·
what could not be verified.
