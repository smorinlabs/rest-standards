# baseline-04e — Idempotency-key replay of a streaming request (prompt)

*Phase 8 research batch, dispatched 2026-08-10 as one of five narrow leaves
under `baseline-04`, each testing one interaction that §13.4 of
`rest-api-standard.md` records as recognized and not yet ruled. Run by an
opus research subagent with the repository evidence rules embedded. Filed
after the run, reconstructed from the dispatch.*

## Question

What should an **idempotency-key replay of a streaming request** deliver, and
must it avoid re-executing the underlying work?

## The concrete problem — the sharpest of the five

`R3.9` requires the server to fingerprint the payload and "replay the stored
response for a genuine retry." `R12.10` explicitly directs a client whose
stream truncated **not** to retry without an idempotency key — routing clients
into `R3.9`'s path. But "the stored response" has no defined meaning when the
response was a stream that ended at frame 47 of 200. One conforming server may
treat an incomplete stream as having no stored response and re-execute;
another may replay from frame 1. On a payment endpoint the first behavior
executes twice. Both conform today.

**The overlap that must be addressed:** `R13.10` resumption is a *different*
mechanism — a SHOULD, conditioned on a retained artifact, using
`stream_position`. State clearly how replay and resumption relate and whether
one satisfies the other.

## Required coverage

1. **Mandatory deep-dive: OpenAI, Anthropic, Google Gemini.** Prior repo
   research (`baseline-02g`) found none ships a true idempotency-key header —
   re-verify, and determine what each does when a streaming request is
   retried. OpenAI's stored-response, `background: true`, and `starting_after`
   mechanics are directly relevant.
2. **The eight standard references**, with Stripe as the idempotency
   deep-dive: what happens when a keyed request is retried while the original
   response is still in flight? Does any vendor document idempotent replay of
   an incomplete or streamed response?
3. **Comparable precedent:** gRPC retry semantics for streaming RPCs;
   Kubernetes watch resumption versus retry; published guidance on
   at-most-once execution for long-running or streamed mutations; the expired
   IETF Idempotency-Key draft — does it address incomplete responses?
4. **Standards layer:** RFC 9110 on retry safety and method idempotence;
   anything addressing a retry whose original response was partially
   delivered.

## Evidence both ways

FOR a strict rule (replay must not re-execute; document what a replay
delivers) and AGAINST (over-specification; whether real systems solve it
differently; whether streaming plus non-idempotent mutation is simply a shape
to avoid).

## Also assess seriously

Whether the standard should instead **forbid or discourage** streaming
responses on non-idempotent mutations, rather than defining replay semantics
for them.

## Evidence rules

Direct URL, authority class, and access date on every material claim;
two-source minimum on load-bearing claims; primary sources first; never
present a draft or vendor convention as a published standard;
`[FACT]`/`[COMPARATIVE]`/`[INFERENCE]`/`[POLICY]` labels; conflicts surfaced,
never averaged. Public repo with push protection — placeholder credentials
only.

## Output

TL;DR and recommendation · standards-and-currency matrix · field evidence with
comparison tables · the replay-versus-resumption relationship stated
explicitly · evidence for and against, separately · proposed rule text with
classification and confidence · declined alternatives · what could not be
verified.
