# CLI Design Standard → REST Standard: Gap Review

*Run 2026-08-09, immediately after Gate C closed (PR #3). Compares the CLI
Design Standard v1.4.14 (the `cli-design-standard.md` document in the
maintainer's separate `cli-standards` repository — Parts I–III +
Appendices A–E, plus its skill apparatus) against this repo's
ratified decision index, decision files, and `PLAN.md` Phase 3 outline.
Nothing here re-litigates a ratified decision; every item is additive.
Produced by a commissioned review agent; sources read in full are listed in
the agent record. Disposition: the Tier 1 content gaps route to the Gate C
addendum walk; apparatus items and Tier 2/3 fold into the Phase 3 drafting
outline.*

## A. Coverage table — content plane

| CLI area | REST analogue | Status | Value of closing |
|---|---|---|---|
| R1.1 noun-verb ordering | Resource orientation, POST action sub-resource | **COVERED** — "Resource orientation" decision (MUST, no RPC carve-out) | — |
| R1.2 nesting depth ≤3 | Path depth | **COVERED** — 1 level norm, ceiling 3 | — |
| R1.3 singular nouns | Pluralization | **COVERED** — plural + singleton/`me` exception | — |
| R1.4 kebab-case casing | Path + body/query casing | **COVERED** — path kebab rider + `AC-007` snake_case | — |
| R1.5 binary naming | API base URL / host + base-path shape, environment hosts | **NOT COVERED** | Moderate |
| R1.6 aliases | Alternate identifiers (ID vs slug), canonical-URI aliasing | **NOT COVERED** (trailing-slash 308 is the only canonicalization rule) | Low-moderate |
| **R2.1 canonical verb table** | **Blessed vocabulary for `{action}` in `POST /{collection}/{id}/{action}`** | **NOT COVERED** — syntax ratified, vocabulary unconstrained | **High** |
| R2.2 resource naming | Noun naming: domain term, no abbreviations, one noun per concept | **PARTIAL** — casing/plural only | Moderate |
| R2.3 positionals=identity, flags=modifiers | Path = identity, query = modifiers | **PARTIAL** — `HS-004` + filter grammar; no explicit "no modifiers in path" rule | Moderate |
| R3.1/R3.5 argv parsing mechanics | — | **N/A** | — |
| R3.6 boolean negation | Boolean query-parameter convention (`?active=true`; absent ≠ false) | **NOT COVERED** | Low-moderate |
| R3.7 repeatable/list flags | Repeated param vs comma-list for multi-valued filters | **PARTIAL** — `fields` is comma-list; filter multi-value unspecified | Moderate |
| R3.8 flag/env/config name parity | Same concept, same name across path, query, body, header | **PARTIAL** — `AC-007` fixes casing, not names | Moderate |
| R3.9 `--file`/stdin input | File upload: multipart, binary bodies, resumable/chunked | **NOT COVERED** | Moderate-high |
| R3.10 no prefix abbreviation | Unknown query parameter: reject 400 vs ignore silently | **NOT COVERED** (unknown *body* fields ≈ `OP-008` allow-list) | Moderate-high |
| **R4.1 standard-option tiers table** | **Reserved query-parameter + reserved-header inventory** | **NOT COVERED** as a table; pieces scattered | **High** |
| R4.2 `-o`/`--output` formats | Content negotiation: default media type absent `Accept`, 406 vs fallback, `?format=` ban, compression, charset | **PARTIAL** — `HS-018` `Vary` + `problem+json` only | Moderate-high |
| **R4.3 `--dry-run`** | **Validate-only / dry-run mode** | **NOT COVERED** | **High** |
| R4.4 verbosity ladder | Response verbosity | **COVERED** — `AC-020` `Prefer: return=minimal\|representation` (MAY) | — |
| R4.6 `--version` output | Version / capability discovery endpoint (root doc, `/version`) | **NOT COVERED** | Moderate |
| R4.7 `--concurrency` | Server-side concurrency limits | **COVERED** — five-axes rate posture (concurrency separate) | — |
| R5.1 precedence chain | Client/SDK base-URL + credential configuration conventions | **NOT COVERED** — scope tension with `PLAN.md` ("does not prescribe client libraries") | Owner scope call |
| R5.2/R5.3 TOML + XDG paths | — | **N/A** | — |
| R5.5 no secrets in argv | No tokens in query strings | **COVERED** — `OP-002` | — |
| R5.6 secret redaction in output | Credential/PII redaction in problem `detail`, echoed input, logs | **PARTIAL** — `OP-020` bans internal detail, not echoed secrets | Moderate |
| R5.7 `config view --show-origin` | Capability/quota introspection endpoint (`GET /rate_limit`, `/me`) | **NOT COVERED** | Moderate |
| R5.8 config validation | Contract/impl drift check | **COVERED** — `AC-002` automated compatibility gate | — |
| R5.9 cache/offline controls | Caching | **COVERED** — three-tier posture + `HS-016/017/018` | — |
| **R6.1 closed exit-code table** | **Closed status-code subset table** | **NOT COVERED** — `HS-003/010/011` give rules, no house subset | **High** |
| R6.2 empty result = exit 0 | Empty collection = 200 + empty array, never 404 | **NOT COVERED** | Moderate (cheap) |
| R6.3 partial failure | Bulk per-item outcomes | **PARTIAL** — `AC-018` requires outcomes; **status code** (200 vs 207) unfixed | Moderate-high |
| R7.1 stdout/stderr split | Never 2xx on failure | **COVERED** — `HS-010` | — |
| **R7.2 machine-output stability rules** | **Breaking-change taxonomy** (what is/isn't compatible) | **NOT COVERED** — "compatible-evolution-first" asserted in `OP-015`, never enumerated | **High** |
| R7.3 TTY detection | Content negotiation | see R4.2 | — |
| R7.4 progress indicators | Progress representation + polling guidance (`Retry-After` on 202) | **PARTIAL** — `AC-019` fixes the operation resource, not polling | Moderate-high |
| R7.5 help text format | Documentation obligations (example per operation, problem-type docs) | **PARTIAL** — `AC-001` requires the doc; no content obligations | Moderate |
| R7.6 error message style | `title`/`detail` writing rules; **field-level validation error array** | **PARTIAL** — envelope fixed; multi-field validation shape unspecified | Moderate-high |
| R7.8 machine error schema | Problem Details | **COVERED** — `AC-003` + `AC-004` amended (strongest area) | — |
| R7.10 `--watch`/`--follow`, JSONL | SSE / streaming / long-poll | **NOT COVERED** — scope tension with `PLAN.md` streaming boundary | Owner scope call |
| **R7.11 deterministic list ordering** | **Documented default sort order** | **NOT COVERED** — and `AC-013` opaque cursors are unsound without it | **High** |
| R7.12 error-code catalog | Published catalog of every problem `type`/`code` | **PARTIAL** — stable codes required, catalog not | Moderate (cheap) |
| R8.1 destructive confirmation | Mandatory `If-Match` on DELETE, collection-DELETE ban, bulk-destructive filter requirement | **PARTIAL** — `HS-015` is a general SHOULD | Moderate-high |
| R8.1 (soft-delete) | Soft-delete/restore/purge lifecycle | **PARTIAL** — named as DELETE exception; lifecycle unspecified | Moderate |
| R8.3 idempotency/declarative | Idempotency keys, PUT/PATCH split | **COVERED** — `AC-016/017`, `HS-008` | — |
| R8.6 dry-run output contract | see R4.3 | **NOT COVERED** | High |
| R9.1 completion/man pages | Where the OpenAPI doc is served, `/.well-known/` | **PARTIAL** — publication required, location unspecified | Moderate (cheap) |
| R9.2 deprecation policy | Deprecation + sunset | **COVERED**; *field-level* deprecation within a major is the one hole | Low-moderate |
| **R9.3 "interface is the public contract"** | **What exactly is frozen within a major** | **NOT COVERED** | **High** (merges with R7.2) |
| R9.4 UTF-8 + locale independence | Charset mandate, `Accept-Language`, localized `title`/`detail` | **PARTIAL** — `AC-009` timestamps only | Moderate |
| R9.6 signals | — | **N/A** | — |
| R9.7 telemetry consent | **PII must not appear in URIs/paths/query** | **NOT COVERED** — adjacent to `OP-002`, unstated | Moderate-high (cheap) |
| R9.8 empty state / first run | Sandbox/test-mode environment | **NOT COVERED** | Low |
| R9.9 release integrity | API changelog obligation for conforming APIs | **PARTIAL** — Phase 5 covers the standard's own changelog only | Low-moderate |
| R9.10 `doctor` | Health/status endpoint | **NOT COVERED** | Moderate |
| R9.11 plugins | Customer-defined `metadata` bags | **NOT COVERED** | Low-moderate |
| R9.12 experimental features | Support tiers | **COVERED** — `/v1alpha`, `/v1beta`, `/v1` | — |
| R9.13 command metadata | Machine-readable contract | **COVERED** — `AC-001` | — |
| **§10 as a whole** | **Consolidated client-obligations section** | **NOT COVERED as a section** — obligations scattered (`OP-012`, `OP-017`, `AC-012`, `AC-013`) | **High** |
| R10.2 `--profile/--context` | Multi-tenancy addressing: tenant in path vs header vs token claim | **NOT COVERED** | Moderate-high |
| R10.3 pagination | Pagination | **PARTIAL** — response ratified; **request parameter names** unfixed | Moderate-high |
| R10.4 long-running ops | 202 operation resource | **PARTIAL** — resource covered; polling, cancellation, progress not | Moderate-high |
| R10.5 idempotency keys | `Idempotency-Key` | **COVERED** | — |
| R10.6 TLS | TLS 1.2+/1.3 | **COVERED** — `OP-001` | — |
| R10.7 rate limits | 429 + `Retry-After` + draft-11 fields | **COVERED** — `OP-010` | — |
| *(no CLI analogue)* | **PATCH document format (RFC 7396 vs RFC 6902 vs plain)** | **NOT COVERED — dropped handoff** (`baseline-01` §9 → `baseline-02`, never decided) | **Highest** |
| *(no CLI analogue)* | Sorting parameter and syntax | **NOT COVERED** — in `PLAN.md` §6 scope, no decision | **High** |
| *(no CLI analogue)* | Expansion/embedding | **NOT COVERED** — explicitly deferred by field-selection decision | Moderate |
| *(no CLI analogue)* | Search-endpoint shape | **NOT COVERED** — named but undefined by the filter-grammar decision | Low-moderate |

## B. Recommended additions, ranked

### Tier 1 — close before or during Phase 3 drafting

1. **PATCH document format** — the dropped handoff. New addendum decision +
   small research leaf (`baseline-02h`): RFC 7396's null-means-remove
   collides with `AC-011` where null is meaningful; RFC 6902 is precise but
   verbose; most surveyed vendors ship plain partial JSON.
2. **Deterministic default collection ordering** — opaque cursors over an
   unordered set silently skip/duplicate. One-sentence rule under §6.
3. **Sorting** — the only `PLAN.md`-scoped topic with no Gate C entry.
   Small addendum decision; evidence in `survey-04`.
4. **Closed status-code subset table** — drafting table + decisions on the
   contested rows: 200/201/204 on create, `Location` on 201, **207 vs 200
   for `AC-018` partial bulk**, 401-vs-403, 403-vs-404 existence masking
   (pairs with `OP-006`), 405+`Allow`, 415, 428 (pairs with `HS-015`).
5. **Reserved query-parameter and header inventory** — the R4.1 analogue.
   Empty cells found while drafting it: pagination *request* params
   (`cursor`/`limit` names), the correlation-ID header name (`OP-018`
   mandates one without naming it), sort param, dry-run param.
6. **Dry-run / validate-only** — addendum decision; `Prefer: validate-only`
   vs `?dry_run=` (Kubernetes `?dryRun=All`, AIP-163 `validate_only`), plus
   the R8.6 output contract (what a dry-run response must state).
7. **Breaking-change taxonomy + frozen-surface enumeration** — drafting
   section; AIP-180/Zalando supply language. Enumerate the frozen surface
   (paths, method/status mapping, field names/types, enum additions per
   `AC-012`, problem `type`/`code` pairs, header names, default sort
   order), then the allowed/forbidden change table.
8. **Tier + applicability system** — apparatus; `internal`/`partner`/
   `public` tiers, applicability switches (webhooks, async, bulk,
   multi-tenant, public-internet, PII, third-party clients, file upload),
   the CLI's N/A-with-reason discipline verbatim.
9. **Consolidated client-obligations section** — mostly consolidation of
   `OP-012`/`OP-017`/`AC-012`/`AC-013` + the `AC-003` robustness note; new
   content small (client timeouts, honor `Retry-After`, TLS verification
   on, unknown-field tolerance).

### Tier 2 — worth doing, lower urgency

10. **Blessed action-verb vocabulary** (addendum decision; `survey-02`
    actions column has the raw material; constrains the ratified syntax,
    does not reopen it).
11. **In-document Decision Log** (Part II of the drafted standard, one row
    per ratified decision linking to its `.decision.md`; SemVer + amendment
    rule).
12. **Conformance-note template + no-silent-deviation clause** (copy the
    CLI template; one normative paragraph).
13. **Rule-ID mapping policy** — settle provenance IDs (`HS-*`) vs
    section-shaped IDs before section 1 is drafted; a mapping table keeps
    every decision record and trigger citation alive.
14. **Executable conformance fixtures** — a Spectral ruleset (the ratified
    regexes are machine-checkable) + a live-probe table.
15. **Framework and gateway mapping appendix** — evidence already in
    `baseline-02b/02c/02e/03g`; no new research.
16. **Unknown query-parameter handling** (reject-vs-ignore; light research).
17. **Content-negotiation defaults** (default media type, 406-vs-fallback,
    `?format=` ban, charset, compression).
18. **PII must never appear in a URI** — one MUST NOT; `OP-002` bans only
    tokens today.
19. **Multi-tenancy addressing** (path vs header vs token claim; needs
    research; interacts with in-handler object authorization).
20. **Destructive-operation guards** (mandatory `If-Match` on DELETE for
    concurrent resources; unfiltered collection-DELETE ban; bulk-destructive
    filter requirement).
21. **Field-level validation error shape** — `errors[]` extension member
    with JSON Pointer targets (Zalando/Belgif shapes).

### Tier 3 — cheap folds (one line each)

Empty collection → 200 + empty array, never 404 · problem-type/code catalog
published · OpenAPI doc served at a documented stable location · UTF-8 +
`Accept-Language` posture + locale-independent machine values · polling and
cancellation guidance for `AC-019` operations (`Retry-After` on 202) ·
health/status endpoint · version/capability discovery · quota introspection
endpoint · boolean query-param convention · repeated-vs-comma multi-value
form · same-concept-same-name parity across surfaces · noun naming rules ·
secret/PII redaction in problem `detail` and echoed input · field-level
deprecation within a major · API changelog obligation · base-URL/environment
host shape (licensed by the BCP 190 scope reading + `HS-005`'s exception) ·
alternate identifiers/aliasing · customer `metadata` bags · search-endpoint
shape (pairs with `HS-009` QUERY) · request body size limits (adjacent to
`OP-009`) · file upload/multipart conventions · expansion/embedding · a
small-API profile (fold into tiers) · worked example annotated with rule
IDs.

### Already-recorded Phase 3 items (not gaps; listed to avoid double-count)

Webhook consumer-obligation placement · `AC-003` client-robustness note +
the problem+json-is-not-json interop hazard · ISO 4217 exponent-table
pointer for minor-unit money.

## C. Genuinely N/A, with reasons

argv tokenization mechanics (R3.1/R3.5) · client filesystem config layout
(R5.2/R5.3) · stdout/stderr separation (transfers only as `HS-010`, covered)
· pager · shell completion/man pages (replaced by `AC-001`; only serving
location transfers) · process signals · self-update/signed binaries
(changelog obligation is the residue) · telemetry *consent* (a provider
necessarily logs; PII-in-URIs is the residue) · verb-first small-CLI profile
as a structural alternative (resource orientation is MUST; only the tier
idea transfers) · verbosity ladder (covered by `AC-020`).

## D. Two scope questions for the owner

1. **Client/SDK configuration conventions** (base URLs, sandbox hosts,
   credential env-var names) — `CONSTRAINTS.md` excludes client libraries;
   an informative appendix tests that intent.
2. **SSE and streaming responses** — the exclusion predates streaming
   becoming the default AI-API shape. Reaffirm the boundary in the
   standard's non-goals, or open a decision item.

## E. Reverse gap

The REST project's evidence apparatus (per-rule confidence, evidence-class
labels, options-declined records, fork honesty, dated re-check triggers)
exceeds the CLI standard's. Copy the CLI's *structure* — tiers, decision
log, conformance note, fixtures — not its thinner one-line rationale style.
