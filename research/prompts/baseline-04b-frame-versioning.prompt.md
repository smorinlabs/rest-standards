# baseline-04b — Frame-vocabulary versioning (prompt)

*Phase 8 research batch, dispatched 2026-08-10 as one of five narrow leaves
under `baseline-04`, each testing one interaction that §13.4 of
`rest-api-standard.md` records as recognized and not yet ruled. Run by an
opus research subagent with the repository evidence rules embedded. Filed
after the run, reconstructed from the dispatch.*

## Question

How should a streaming API version its **frame-type vocabulary**, and what
should this standard require?

## The concrete problem

`R13.5` requires an API to document its frame-type vocabulary and state that
it may grow. `R9.4`'s frozen-surface list does not mention frame-type names.
`R12.10` requires clients to ignore unrecognized frame types **and** to treat
a stream closing without its documented terminal frame as truncated. So
renaming a terminal frame appears *compatible* under §9.3 while breaking every
deployed client — which reports truncation on every successful stream.

`R9.4` already freezes "reserved-name semantics (§1.10)" and "problem
`type`/`code` pairs (R5.13.2)". Both are precedent for freezing a name that
clients dispatch on.

## Required coverage

1. **Mandatory deep-dive: OpenAI, Anthropic, Google Gemini.** How each versions
   its streaming event-type vocabulary; any rename or removal that shipped;
   the compatibility promise published about event types; whether the
   vocabulary is documented as growable.
2. **The eight standard references** — Stripe, GitHub, Google/AIP,
   Microsoft/Azure, Twilio, Shopify, Zalando, AWS — wherever they have an
   event-type surface. Non-participation is a one-line dated finding.
3. **Comparable precedent:** webhook event-type versioning (Stripe's
   `api_version`, Standard Webhooks), Kubernetes watch event types, AsyncAPI
   on message-type evolution, published guidance on evolving an open enum in a
   wire protocol.
4. **Standards layer:** AsyncAPI, CloudEvents, or IETF work classifying
   event-type renaming as breaking or compatible. A verified negative is a
   finding.

## Evidence both ways

Gather evidence FOR freezing frame-type names (who does it, what breaks
without it, documented incidents) and AGAINST (who renames freely and
survives, what freezing costs).

## Evidence rules

Direct URL, authority class, and access date on every material claim;
two-source minimum on load-bearing claims; primary sources first; never
present a draft or vendor convention as a published standard;
`[FACT]`/`[COMPARATIVE]`/`[INFERENCE]`/`[POLICY]` labels; conflicts surfaced,
never averaged. Public repo with push protection — placeholder credentials
only.

## Output

TL;DR and recommendation · standards-and-currency matrix · field evidence with
comparison tables · evidence for and against, separately · proposed rule text
with classification and confidence · declined alternatives · what could not be
verified, named.
