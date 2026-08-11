# baseline-04c — Authorization over a stream's lifetime (prompt)

*Phase 8 research batch, dispatched 2026-08-10 as one of five narrow leaves
under `baseline-04`, each testing one interaction that §13.4 of
`rest-api-standard.md` records as recognized and not yet ruled. Run by an
opus research subagent with the repository evidence rules embedded. Filed
after the run, reconstructed from the dispatch.*

## Question

What should this standard require about **authorization over the lifetime of
a long-lived stream**?

## The concrete problem

`R8.6` authorizes a *request*. A stream is one request that can run for tens
of minutes and outlive the credential that opened it. `R8.5` wants expiring
credentials, and `R8.10`'s token-format axis assumes a revocation posture. No
rule says whether a server must re-evaluate authorization mid-stream, bound
stream lifetime by credential lifetime, or terminate on revocation. A revoked
principal can keep receiving data until the stream ends.

## Required coverage

1. **Mandatory deep-dive: OpenAI, Anthropic, Google Gemini.** Published
   maximum stream or request duration; what happens to an in-flight stream
   when a key is revoked or a token expires; any documented mid-stream
   authorization behavior.
2. **The eight standard references** — Stripe, GitHub, Google/AIP,
   Microsoft/Azure, Twilio, Shopify, Zalando, AWS — wherever they hold
   long-lived connections. Non-participation is a dated finding.
3. **Comparable precedent:** Kubernetes watch and its token-expiry behavior;
   gRPC long-lived stream auth; WebSocket auth-expiry guidance as contrast
   (WebSockets are out of scope for the standard); published guidance on OAuth
   token expiry during a long-lived HTTP response.
4. **Standards layer:** RFC 9700, BCP 240, RFC 7009, RFC 9068 — does any
   published standard address authorization for a request already in
   progress? A verified negative is a finding.

## Evidence both ways

FOR requiring mid-stream re-evaluation or lifetime binding (incidents,
security guidance, who implements it) and AGAINST (cost, complexity, who
deliberately does not, whether the threat is real in practice).

## The narrower option to assess explicitly

Requiring only that an API **document** its posture, without mandating a
mechanism. The standard already did exactly this for keep-alive frames. Is
disclosure-only sufficient here, or does a security-relevant item need more?

## Evidence rules

Direct URL, authority class, and access date on every material claim;
two-source minimum on load-bearing claims; primary sources first; never
present a draft as a published standard;
`[FACT]`/`[COMPARATIVE]`/`[INFERENCE]`/`[POLICY]` labels; conflicts surfaced,
never averaged. Public repo with push protection — placeholder credentials
only.

## Output

TL;DR and recommendation · standards-and-currency matrix · field evidence with
comparison tables · evidence for and against, separately · proposed rule text
with classification and confidence, including whether disclosure-only
suffices · declined alternatives · what could not be verified.
