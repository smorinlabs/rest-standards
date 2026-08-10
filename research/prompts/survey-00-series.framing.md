# REST API Conventions Research Series — Index

Seven self-contained deep-research prompts. Each covers all eight references (Stripe, GitHub, Google/AIP, Microsoft, Twilio, Shopify, Zalando, AWS-as-contrast) on one coherent slice of the convention surface, and each returns its own scoped **Contested Axes Register**. The seven registers merge into one master register that drives the decision phase.

| # | Stem | Covers | Notes |
|---|------|--------|-------|
| P1 | `survey-01-foundations` | RFCs 9110/9111/9457/3339/8288, JSON Patch/Merge Patch, Sunset, IETF `httpapi` statuses, JSON:API, OpenAPI 3.1, HATEOAS adoption reality | Lightest run; read first — frames the rest |
| P1b | `survey-01b-foundations-mechanics` | Implementation-grade *mechanics* of the P1 standards: exact grammars, header formats, verbatim example payloads | Supplement — P1 settled status/adoption; P1b captures spec mechanics |
| P2 | `survey-02-structure` | Resource modeling, URLs, methods, status codes | AWS gets its full contrast treatment here |
| P3 | `survey-03-representations-errors` | Body conventions (casing, envelopes, timestamps, money, IDs) + error shapes | "What the JSON looks like" |
| P4 | `survey-04-collections` | Pagination, filtering, sorting, field selection, expansion | Shopify is a key reference here |
| P5 | `survey-05-reliability` | Idempotency (Stripe deep-dive), concurrency/ETags, async/LRO, bulk | Highest-priority part |
| P6 | `survey-06-lifecycle-operations` | Versioning, deprecation, rate limits, caching, auth surface, docs practice | SigV4 contrast lives here |
| P7 | `survey-07-webhooks` | Envelopes, signatures, delivery/retries, verification, thin-vs-fat | Adds the Standard Webhooks spec as a baseline |

The original `P#` numbering is retained above as historical lineage. Files were
renamed to the repository naming convention in
[`../README.md`](../README.md); add `.prompt.md` to a stem for the prompt and
look in `../reports/` for its runs.

## How to run

- Submit each file as its own deep-research session, verbatim. They are **independent — run them in parallel** if you like.
- Suggested reading order once reports return: P1 → P2 → P3 → P4 → P5 → P6 → P7 (foundations first; the rest in structural order).

## After the reports return

1. Paste the reports back into the working conversation (or their registers, at minimum).
2. **Step 3 — decision elicitation:** the seven registers are merged into one master Contested Axes Register; decisions proceed one-by-one (context → options → recommendation → confidence → genuine-fork-or-not), structural locks first (resource orientation, noun number, casing, versioning scheme), then the rest batched by confidence.
3. **Step 4 —** the generation prompt is drafted once with all locked decisions baked in (plus a built-in self-review pass), then run to produce the REST Design Standard v1.0.0.
4. **Step 5 —** one consolidated hardening pass (ambiguities/conflicts/gaps) → v1.1.0 → scripted consistency sweep.

## Addendum 2026-08-10 — Part 8 extends the series

The seven parts above (plus the P1b supplement) were the version 1.0 series and
are complete. **Phase 6 adds an eighth part, `survey-08-streaming`**, covering
Server-Sent Events, long-polling, and streaming HTTP bodies — a surface ruled
out of scope at Gate D and reopened after 1.0 shipped. It is a genuine series
member and carries every series-wide convention listed below, with one recorded
exception: **its reference set is not the eight above**, because most of the
eight publish no streaming surface. The substitution and its reasons are in
[`survey-08-streaming.framing.md`](survey-08-streaming.framing.md). The "seven"
counts in the table and prose above are the 1.0 series and are left as written
rather than silently restated.

## Series-wide conventions (already embedded in every prompt)

Descriptive-only mandate · Stripe as key-reference-not-canonical · the same out-of-scope guard (OAuth internals, GraphQL/gRPC, infra, SDKs, event-streaming platforms) · primary-sources-first, currency checks, per-finding confidence, surfaced conflicts · a scoped Contested Axes Register as a required output of every part (except P1b) · **a specification-grade detail requirement in every part**: exact names/formats, at least one verbatim example per mechanism, concrete numbers with sources, collected in an Examples Appendix — the completion test is "implementable without opening the reference's docs."
