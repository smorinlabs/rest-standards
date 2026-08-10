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

### Gate B follow-up leaves

Four narrow leaves run 2026-07-25 to test the weakest links in the baseline
principles, plus one run 2026-08-09 during Gate C. Each verified its claims
against primary sources with a two-source minimum.

| Stem | Question | Runs | Outcome |
| --- | --- | --- | --- |
| `baseline-01b-query-deployment` | Is RFC 10008 QUERY deployed enough to promote `HS-009`? | `2026-07-25` | **No.** `HS-009` stays `MAY` — no CDN does body-keyed caching |
| `baseline-02b-openapi-tooling` | Is OpenAPI 3.2 tooling mature enough for `AC-001`? | `2026-07-25` | **No.** `AC-001` revised to accept 3.1 or 3.2; JSON Schema pin strengthened |
| `baseline-02c-problem-details-adoption` | Does the inference behind `AC-003` hold? | `2026-07-25` | **Yes, strengthened.** Confidence raised — *partially superseded by `baseline-02d`* |
| `baseline-02d-greenfield-adoption` | Do new (2023+) APIs adopt RFC 9457, and does the standard survive its critics? | `2026-08-09` | **Mixed.** CAMARA rejection falsifies `02c`'s absence claim; no credible alternative exists; keep `MUST`, confidence back to moderate, re-argue |
| `baseline-02e-cloudflare-implementation` | What exactly did Cloudflare ship on 2026-03-11, live-verified? | `2026-08-09` | **Strong precedent for the shape, weak for the obligation.** Supports "capable of returning" wording + carve-out; `type` is a docs affordance, not the identity — stability lives in extension members |
| `baseline-02f-problem-type-semantics` | What should a greenfield standard mandate for RFC 9457 `type`? | `2026-08-09` | **Split identity from documentation.** Stable `https` `type` 1:1 with `code`, dereference optional, docs in a separate member; `about:blank` ban proposed; IETF back-compat rationale now primary-sourced |
| `baseline-02g-idempotency-key-practice` | What do OpenAI, Anthropic, and Google do for idempotency keys? | `2026-08-09` | **None of the three AI vendors ships one.** Google's AIP-155 `request_id` is a query param over REST, no fingerprinting; both AI vendors' SDKs carry dormant header-shaped Stainless machinery, switched off |
| `baseline-02h-patch-format` | Which PATCH body format — Merge Patch, JSON Patch, or plain JSON? | `2026-08-09` | **Merge Patch + null-equivalence rule + JSON Patch MAY.** RFC 5789 mandates no format (Content-Type negotiates); Merge Patch alone implements AC-011's wire semantics; Azure and Zalando both mandate it with the same companion rule; plain JSON is the undeclared field plurality |
| `baseline-03b-signatures-and-ratelimit` | Do `OP-016` and `OP-010` survive contact with reality? | `2026-07-25` | **Split.** `OP-016` raised; `OP-010` lowered with an expiry contingency |
| `baseline-03c-webhook-threat-model` | What is webhook signing for, and how do vendor schemes fare against it? | `2026-08-09` | **Purpose confirmed (security).** 13 invariants derived; all documented failures are receiver-side; RFC 9421 alone does not sign the body — needs RFC 9530 pairing |
| `baseline-03d-webhook-signing-adoption` | What do post-2023 webhook implementations actually sign with? | `2026-08-09` | **Topology split.** Standard Webhooks won product webhooks (OpenAI, Anthropic, Gemini + ~9 verified); RFC 9421 won cross-org protocols (UCP MUST, AdCP removing HMAC in 4.0); Web Bot Auth charter excludes webhooks — `baseline-03b` inference capped |
| `baseline-03e-ratelimit-field-survey` | What does the industry actually emit for rate limits? | `2026-08-09` | **No convergence.** X-prefixed family is a plurality (10/31, corrected), not a majority; 11/31 publish no quota state; exactly one draft-11-shaped emitter (Cloudflare, uncited); every IETF citation in the wild is draft-06 or earlier — the superseded trio shape |
| `baseline-03f-ratelimit-draft-trajectory` | Does the RateLimit draft have a path to RFC, and what is normatively available instead? | `2026-08-09` | **MUST rejected on RFC 2026 grounds.** Draft expired+revived 3×, no AD/WGLC/milestone, wire format still moving (PR #166 renames parameters); falsifies `03b`'s "editorial → stable" inference; recommends MUST 429+`Retry-After` (published standards) + SHOULD pinned draft-11 fields, contingency re-keyed off expiry |
| `baseline-03g-risk-axes` | Defaults + flip triggers for the five threat-model-conditional security axes? | `2026-08-09` | **Deployment-profile skeleton produced.** Bearer+TTL/rotation default (BCP 240's MUST satisfiable by rotation; Entra ships neither DPoP nor mTLS; MCP mandates plain Bearer) · opaque-on-the-wire default · multi-dimensional tiered rate posture · 300s/60s asymmetric replay window + dedup · centralize authz decision, enforce in-handler |

**Dated re-check triggers:** `OP-010` semi-annual review, next **2027-02-09**
(upgrade if `RateLimit` lands in the IANA field registry; re-pin on syntax
change — watch draft PR #166 / issue #158; withdraw only on 18-month
abandonment; 2026-11-24 is only a check for whether a draft-12 appeared) ·
Spring Framework 7.1, targeted November 2026 (`HS-009`) · swagger-parser #2248
or openapi-generator #22728 closing (`AC-001`) · AdCP 4.0 release, Standard
Webhooks issue #34, and any UCP revision after 2026-04-08 (`OP-016`) ·
2026-11-15, `draft-knauer-secure-webhook-token` expiry (`OP-016`) ·
five-axes profile watches: Microsoft Entra shipping mTLS PoP (highest-value —
would weaken the bearer default), any MCP proof-of-possession extension,
OpenFGA CNCF graduation, FAPI 2.0 adoption outside finance.

## Decision index — `decisions/` (Gate C, complete 2026-08-09)

The ADR layer. One file per stem, paired with its prompt and reports by the
naming convention; each entry records the ratified rule, its Phase 2
classification, the evidence chain, and the options declined. This table is
the quick-citation index; the files are authoritative.

| ID | Decided | Outcome (one line) | File |
| --- | --- | --- | --- |
| `AC-003` | 2026-08-09 | RFC 9457 `problem+json` ratified **MUST** — "capable of returning" wording, infrastructure carve-out, nothing premised on the IANA registry; re-argued as "no credible alternative" | `decisions/baseline-02-api-contracts.decision.md` |
| `AC-004` (amended) | 2026-08-09 | `type` = stable https URI 1:1 with `code` via standard-fixed template; dereference optional; docs in separate `documentation` member; `about:blank` banned | `decisions/baseline-02-api-contracts.decision.md` |
| `OP-016` | 2026-08-09 | Webhook signing **MUST**, scheme by trust topology — Standard Webhooks for shared-secret; RFC 9421 + RFC 9530 for cross-org; SHA-1 banned; 13 invariants | `decisions/baseline-03-operational-practice.decision.md` |
| Resource orientation | 2026-08-09 | **MUST** be resource-oriented — nouns + standard verbs; non-CRUD ops as POST action sub-resource (syntax deferred to the action-syntax item); no RPC carve-out | `decisions/baseline-01-http-semantics.decision.md` |
| Pluralization | 2026-08-09 | **MUST** plural collections; singleton/config exception (`/user`, `me` pattern); irregulars: one form, consistent | `decisions/baseline-01-http-semantics.decision.md` |
| `OP-010` (+ item 22) | 2026-08-09 | **MUST** 429 + `Retry-After` (published standards); **SHOULD** draft-11 `RateLimit` fields as `[POLICY]`; proprietary headers documented incl. epoch-vs-delta; contingency re-keyed off expiry (IANA upgrade / re-pin / 18-mo withdraw) | `decisions/baseline-03-operational-practice.decision.md` |
| `AC-007` (completed) | 2026-08-09 | **snake_case** for bodies and query params, regex-enforced (`^[a-z_][a-z_0-9]*$`) | `decisions/baseline-02-api-contracts.decision.md` |
| `OP-015` (completed) | 2026-08-09 | **Major version in path** (`/v1`), compatible-evolution-first; majors rare; minor/patch never in URIs | `decisions/baseline-03-operational-practice.decision.md` |
| `AC-016` (completed) | 2026-08-09 | **`Idempotency-Key` header**, Stripe semantics (fingerprint + reject reuse-with-different-payload), `[POLICY]` — draft expired, never cite as standard | `decisions/baseline-02-api-contracts.decision.md` |
| Path depth | 2026-08-09 | Nest 1 level as the norm, **ceiling 3 resources/path**, prefer flat + query filters | `decisions/baseline-01-http-semantics.decision.md` |
| Trailing slash | 2026-08-09 | **None canonical**; trailing-slash request SHOULD 308 to canonical; never semantic | `decisions/baseline-01-http-semantics.decision.md` |
| Action syntax | 2026-08-09 | **Sub-path verb** `POST /{collection}/{id}/{action}` + reserved-word rule; `:verb` declined | `decisions/baseline-01-http-semantics.decision.md` |
| Path casing (rider) | 2026-08-09 | **kebab-case** segments (`^[a-z][a-z\-0-9]*$`) | `decisions/baseline-01-http-semantics.decision.md` |
| DELETE response | 2026-08-09 | **204 No Content**; soft-delete exception returns 200 + tombstone | `decisions/baseline-01-http-semantics.decision.md` |
| Money encoding | 2026-08-09 | **Minor-unit integer + ISO 4217 `currency`** (owner choice over recommended decimal string; exponent-table consequence noted) | `decisions/baseline-02-api-contracts.decision.md` |
| `AC-001` (completed) | 2026-08-09 | **OpenAPI 3.1 floor**; 3.2 gated on verified toolchain; JSON Schema 2020-12 pin stands | `decisions/baseline-02-api-contracts.decision.md` |
| BCP 190 posture | 2026-08-09 | **State the scope reading** — house standard constraining its own URI space is not BCP 190's harm | `decisions/baseline-01-http-semantics.decision.md` |
| HATEOAS posture | 2026-08-09 | **Honest description** — resource-oriented HTTP, not Fielding-complete REST; divergence noted openly | `decisions/baseline-01-http-semantics.decision.md` |
| Pagination links | 2026-08-09 | **Body envelope only**; no RFC 8288 `Link` headers for pagination | `decisions/baseline-02-api-contracts.decision.md` |
| Caching posture | 2026-08-09 | **Three-tier explicit default** — never silent; authenticated → `private, no-cache` + ETag; `no-store` sensitive-only; `public` immutable-only | `decisions/baseline-01-http-semantics.decision.md` |
| Field selection | 2026-08-09 | **MAY**; when offered, `fields` comma-list of snake_case names — no `$select`, no brackets | `decisions/baseline-02-api-contracts.decision.md` |
| Filter grammar | 2026-08-09 | **Per-field + bracket ranges, AND-only** on lists; DSL only as a separate search endpoint | `decisions/baseline-02-api-contracts.decision.md` |
| `AC-017` (completed) | 2026-08-09 | Idempotency-key retention **≥24h** floor | `decisions/baseline-02-api-contracts.decision.md` |
| Deprecation window | 2026-08-09 | Deprecated major supported **≥12 months** after successor; sunset announced at deprecation | `decisions/baseline-03-operational-practice.decision.md` |
| Webhook dead-letter | 2026-08-09 | **≥72h** exponential retries, then **≥30d** dead-letter store + redelivery API | `decisions/baseline-03-operational-practice.decision.md` |
| Support tiers | 2026-08-09 | **AIP-style path tiers** — `/v1alpha…`/`/v1beta…`/`/v1`; GA-only deprecation guarantees | `decisions/baseline-03-operational-practice.decision.md` |
| Auth per client class | 2026-08-09 | **OAuth REQUIRED for delegated authority; API keys acceptable server-to-server single-trust**; BCP 240 wherever OAuth | `decisions/baseline-03-operational-practice.decision.md` |
| Five-axes profile | 2026-08-09 | **Deployment profile ratified** — bearer+rotation, opaque-on-wire, tiered rate posture, 300s/60s replay + dedup, centralized-decision/in-handler authz; each with flip triggers | `decisions/baseline-03-operational-practice.decision.md` |
| HS batch (20) | 2026-08-09 | `HS-001`–`HS-020` ratified en bloc as proposed | `decisions/baseline-01-http-semantics.decision.md` |
| AC batch (15) | 2026-08-09 | Remaining AC principles ratified en bloc as proposed | `decisions/baseline-02-api-contracts.decision.md` |
| OP batch (22) | 2026-08-09 | Remaining OP principles ratified en bloc as proposed — **Gate C pile complete** | `decisions/baseline-03-operational-practice.decision.md` |
| A2 · Sorting cluster (addendum) | 2026-08-09 | **MUST** documented stable default order (ties by `id`); `sort=-field,other` when offered; `cursor`+`limit` request params; `request-id` correlation header | `decisions/baseline-02-api-contracts.decision.md` |
| A1 · PATCH format (addendum) | 2026-08-09 | **RFC 7396 Merge Patch MUST** at `application/merge-patch+json` + null ≡ absent companion rule; RFC 6902 as bounded MAY | `decisions/baseline-02-api-contracts.decision.md` |
| A3 · Status-code rows (addendum) | 2026-08-09 | **201+`Location`** on every create; **200 + envelope** over 207 for partial bulk; **404** masks cross-tenant existence; 405/415/428/401-403/empty-200 rows en bloc | `decisions/baseline-01-http-semantics.decision.md` |
| A4 · Dry-run (addendum) | 2026-08-09 | **`?dry_run=true`**, MAY per endpoint / SHOULD on destructive+bulk; output contract ratified (marker, real outcome shape, validation depth, no key consumption); `Prefer: validate-only` declined on advisory-execution risk | `decisions/baseline-02-api-contracts.decision.md` |
| A5 · Action verbs (addendum) | 2026-08-09 | **Core registry** (cancel, archive/restore, approve/reject, publish/unpublish, duplicate) + AIP-136 discipline + one-verb-per-meaning; no collection-level actions | `decisions/baseline-02-api-contracts.decision.md` |

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
