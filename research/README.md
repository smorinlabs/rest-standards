# Research Inventory

Every research artifact in this repository lives in this directory, in a flat
structure. There are no per-topic subdirectories: a prompt, its runs, and its
eventual decision record are linked by a shared filename stem, not by folder.

Shared boundaries are in [`CONSTRAINTS.md`](CONSTRAINTS.md). The repository plan
is in [`../PLAN.md`](../PLAN.md). Agent-facing rules are in
[`CLAUDE.md`](CLAUDE.md).

## Layout

```
research/
  prompts/     the question, as issued
  reports/     what a run of that question returned
  decisions/   the ratified conclusion (Phase 2; not yet created)
```

## Naming convention

```
<series>-<seq>-<topic>.<role>[.<date>[<letter>]].md
```

| Field | Rule |
| --- | --- |
| `series` | `survey` or `baseline` (see below). Fixed vocabulary. |
| `seq` | Two digits. A trailing letter marks a supplement or follow-up leaf (`01b`). |
| `topic` | Lowercase, hyphenated, **stable forever** — this is the pairing key. |
| `role` | `prompt`, `framing`, `report`, or `decision`. |
| `date` | ISO `YYYY-MM-DD`. Reports only. A trailing letter disambiguates same-day runs. |

Three properties follow from this, and they are the reason for the convention:

- **Pairing is a glob.** `ls research/*/survey-05-reliability.*` returns the
  prompt, every run of it, and its decision record. No index lookup required.
- **Gaps are visible.** A stem present in `prompts/` with no match in `reports/`
  has not been run. Coverage is checkable without opening a file.
- **Files self-identify out of context.** A file pasted into a conversation or
  copied elsewhere still states its series, subject, role, and vintage.

## Series

The two series differ in kind, not just in vintage. Do not merge them.

| Series | Mode | Question it answers | Output |
| --- | --- | --- | --- |
| `survey` | Descriptive | What do eight reference APIs and the standards actually do today? | Comparison and a contested-axes register. Makes no recommendations. |
| `baseline` | Prescriptive | What should this standard require? | Proposed normative-principles tables with `MUST`/`SHOULD`/`MAY` strength. |

`baseline` reports propose rules; they do not ratify them. Ratification is
Phase 2 (Gate C) and lands in `decisions/`.

## Inventory — `survey` series

Eight prompts, ten runs. Descriptive comparison across Stripe, GitHub,
Google/AIP, Microsoft, Twilio, Shopify, Zalando, and AWS-as-contrast.
Series framing: [`prompts/survey-00-series.framing.md`](prompts/survey-00-series.framing.md).

| Stem | Covers | Runs | Decision |
| --- | --- | --- | --- |
| `survey-01-foundations` | RFCs 9110/9111/9457/3339/8288, patch formats, Sunset, OpenAPI 3.1, HATEOAS reality | `2026-07-19`, `2026-07-20` | — |
| `survey-01b-foundations-mechanics` | Implementation-grade spec mechanics: grammars, header formats, verbatim payloads | `2026-07-19` | — |
| `survey-02-structure` | Resource modeling, URLs, methods, status codes | `2026-07-19` ⚠ | — |
| `survey-03-representations-errors` | Body conventions (casing, envelopes, timestamps, money, IDs) and error shapes | `2026-07-19` ⚠ | — |
| `survey-04-collections` | Pagination, filtering, sorting, field selection, expansion | `2026-07-19` | — |
| `survey-05-reliability` | Idempotency, concurrency/ETags, async operations, bulk | `2026-07-19` | — |
| `survey-06-lifecycle-operations` | Versioning, deprecation, rate limits, caching, auth surface, docs | `2026-07-19a`, `2026-07-19b` | — |
| `survey-07-webhooks` | Envelopes, signatures, delivery/retries, verification, thin-vs-fat | `2026-07-19` | — |

## Inventory — `baseline` series

Three prompts derived from the original three-thread plan. Each carries a
neighbouring `.framing.md` recording its provenance and shaping decisions.

| Stem | Covers | Runs | Principle IDs | Decision |
| --- | --- | --- | --- | --- |
| `baseline-01-http-semantics` | REST constraints, resource identity, URI modeling, methods, status codes, conditional requests, caching | `2026-07-25` | `HS-001`–`HS-020` | — |
| `baseline-02-api-contracts` | JSON representation, OpenAPI/JSON Schema contracts, Problem Details, collections, compatibility, idempotency contracts, bulk, async | `2026-07-25` | `AC-001`–`AC-021` | — |
| `baseline-03-operational-practice` | Transport security, authn/authz, rate limits, retries, deprecation, observability, webhook delivery and signing | `2026-07-25` | `OP-001`–`OP-025` | — |

Sixty-six proposed principles in total. **All are proposals carrying research
confidence, not ratified policy** — see the series table above.

## Currency corrections — read before citing a `survey` report

The `baseline` series verified primary-source status live on 2026-07-25 and
found material that the `survey` series does not contain. Where the two differ,
the baseline reports are current.

| Finding | Consequence | Where |
| --- | --- | --- |
| **RFC 10008 — The HTTP QUERY Method**, Proposed Standard, June 2026. Safe, idempotent, cacheable, carries a request body. | Absent from all ten survey reports. Directly addresses the complex-query workarounds documented in `survey-04-collections`. | `baseline-01` |
| **RFC 9205 (BCP 56)** and **RFC 8820 (BCP 190)** | Absent from all ten survey reports. Both are binding BCPs governing URI design and status-code use. | `baseline-01` |
| **OpenAPI 3.2.0**, released 19 Sep 2025 | Survey series baselines on 3.1 throughout. | `baseline-02` |
| **Idempotency-Key draft expired 2026-04-18**, intended RFC status "(None)" | The widely-copied `Idempotency-Key` header has no standards backing. Any rule is project policy. | `baseline-02` |
| **RateLimit draft-11 is active**, expires 2026-11-24 | Different posture from the dead idempotency draft — adoptable as a forward bet. | `baseline-03` |
| **RFC 8594 (Sunset) is Informational**, while RFC 9745 (Deprecation) is Standards Track; the two use deliberately different date formats | The pair is routinely described as equivalent. It is not. | `baseline-03` |
| **RFC 9421, RFC 9700, RFC 9325, W3C Trace Context, and OWASP appear in no survey report** | The survey read vendor API documentation, where security and telemetry posture is largely not published. | `baseline-03` |

`[Note]` These gaps reflect the survey's descriptive mandate rather than a
defect in it: a comparison of what vendors shipped cannot surface a standard
nobody has shipped yet, nor a BCP about how to design protocols.

## Provenance notes

Read these before treating a filename date as fact.

- **⚠ Inferred dates.** `survey-02-structure` and `survey-03-representations-errors`
  state only "Jul 2026" / "July 2026" in their headers, with no day. Both are
  dated `2026-07-19` by inference from the surrounding batch, in which every
  other report states that day. The filename date is an inference, not a claim
  from the report.
- **Two runs of `survey-01-foundations`.** Independent executions dated
  2026-07-19 and 2026-07-20. Both are retained: where two runs of one prompt
  diverge, the divergence is a confidence signal for the decision phase.
- **Two runs of `survey-06-lifecycle-operations`.** Both ran 2026-07-19, so the
  `a`/`b` suffixes are arbitrary disambiguators and carry no ranking. They
  differ in retrieval window — `a` states 2026-07-19; `b` states a
  July 12–19 window — and in emphasis. **Checked 2026-07-25: they do not
  contradict each other on versioning transport.** Both place Google and
  Microsoft Graph on coarse major-version path tokens and Azure on a dated
  query parameter; the runs foreground different vendors in their summaries,
  which reads as disagreement until the bodies are compared. Two independent
  runs agreeing is a confidence signal in its own right.
- **One redaction, recorded.** `survey-06-lifecycle-operations.report.2026-07-19a.md`
  line 168 originally quoted Stripe's public documentation example test-mode
  key verbatim to illustrate secret-key-as-Basic-username auth. It is a
  long-published example from Stripe's own docs and test-mode only, but it is
  credential-shaped, this repository is public, and GitHub push protection
  blocked it. The token was replaced with `sk_test_<redacted-see-README>`;
  the finding it supports — secret key as Basic username with an empty
  password, Bearer also accepted — is unchanged. **This is the only
  modification made to any `survey` report's content.**
- **Original filenames.** The ten `survey` reports arrived as
  `compass_artifact_wf-<uuid>_text_markdown.md`. That name carried no link to
  its prompt; the mapping in this document was recovered by reading each
  report's title and retrieval date. Rename on arrival so this is never
  necessary again.
