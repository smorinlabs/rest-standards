# Baseline 02 — Representations and API Contracts

*Prescriptive research. Primary-source status verified live 2026-07-25 against
IETF Datatracker, spec.openapis.org, and json-schema.org. Vendor-practice
evidence is carried from the in-repo `survey` series (retrieved 2026-07-19/20)
and labeled as such. This report **proposes** normative rules; nothing here is
project policy until ratified at Gate C.*

**Label key:** `[FACT]` sourced to a primary specification · `[COMPARATIVE]`
deployed vendor practice · `[INFERENCE]` reasoned from sourced facts ·
`[RECOMMENDATION]` proposed rule · `[POLICY]` a choice evidence cannot settle.

---

## 1. Executive recommendation

Design contracts so that **the machine-readable description is the source of
truth, and every convention that cannot be expressed in it is a convention the
standard should not adopt.** Seven commitments:

1. **Baseline on OpenAPI 3.2.0 with the JSON Schema 2020-12 dialect**, while
   accepting 3.1 documents. 3.2 is strictly incremental over 3.1 and tooling
   support landed through Q1 2026.
2. **Mandate RFC 9457 `application/problem+json` for all error responses**,
   with a project-defined extension set. This is a deliberate divergence from
   the field: only one of eight surveyed references mandates it.
3. **Accept that application-level idempotency has no standards backing.** The
   IETF draft expired on 2026-04-18 and never advanced. Any rule here is
   `[POLICY]` grounded in convergent vendor practice, and must be labeled as
   such rather than presented as standards conformance.
4. **Choose one casing and apply it to bodies and query parameters alike.** The
   field is genuinely split; the cost is inconsistency, not incorrectness.
5. **Represent time as RFC 3339 strings and money as minor-unit integers or
   decimal strings — never floats.** The float prohibition is the single
   strongest cross-source consensus in this domain.
6. **Default to cursor pagination with opaque, non-constructable cursors**, and
   permit offset only where data is small or stable.
7. **Make every asynchronous acceptance produce a status resource.** A 202 with
   no addressable operation strands the client.

**Overall confidence: high** for items 1, 5, 6, 7 · **moderate** for 2 (a
defensible divergence from field practice, argued below) · **moderate** for 3
and 4, which are policy choices with evidence but no governing standard.

---

## 2. Source-and-currency matrix

All rows verified 2026-07-25.

| Source | Class | Version / date | Status verified today | URL |
| --- | --- | --- | --- | --- |
| OpenAPI Specification | Industry spec | **3.2.0**, 19 Sep 2025 | Current release. Aligns with JSON Schema draft 2020-12. Spec states tooling supporting 3.1 SHOULD be compatible across 3.1.x, and that minor versions **may** carry non-backwards-compatible changes where impact is low. | https://spec.openapis.org/oas/latest.html |
| JSON Schema | Internet-Draft lineage | **2020-12** | Current stable. Superseded 2019-09. A next draft is work-in-progress with no release date. **Not an RFC** — no IETF standards-track status. | https://json-schema.org/specification |
| RFC 9457 — Problem Details | Proposed Standard | Jul 2023 | Proposed Standard. Obsoletes RFC 7807. **No updated-by.** Defines `application/problem+json` and `application/problem+xml`. | https://datatracker.ietf.org/doc/rfc9457/ |
| RFC 6570 — URI Template | Standards Track | Mar 2012 | Active, not obsoleted. Errata exist. | https://datatracker.ietf.org/doc/rfc6570/ |
| RFC 7240 — Prefer Header | Proposed Standard | Jun 2014 | Proposed Standard. **Updated by RFC 8144.** Defines `respond-async`, `return=minimal`/`return=representation`, `wait`, `handling=strict`/`lenient`. Preferences are non-binding. | https://datatracker.ietf.org/doc/rfc7240/ |
| RFC 8288 — Web Linking | Proposed Standard | Oct 2017 | Proposed Standard. Obsoletes RFC 5988. No updated-by. | https://datatracker.ietf.org/doc/rfc8288/ |
| `draft-ietf-httpapi-idempotency-key-header` | **Expired Internet-Draft** | draft-**07**, revised 2025-10-15 | **EXPIRED 2026-04-18.** Datatracker states the draft "is no longer active" and intended RFC status is "(None)." Never published as an RFC. | https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/ |

### Currency corrections this thread produced

- `[FACT]` **OpenAPI 3.2.0 exists and post-dates the survey series.** Published
  19 Sep 2025. No `survey` report mentions it; all baseline on 3.1 (verified by
  search across `research/reports/`).
- `[FACT]` **The Idempotency-Key draft is dead, not merely unfinished.**
  Datatracker records expiry on 2026-04-18 and intended RFC status "(None)."
  The widely-copied Stripe `Idempotency-Key` header therefore has **no**
  standards backing, and this project cannot claim standards conformance for
  any idempotency-key rule it adopts.
- `[FACT]` **RFC 7240 is updated by RFC 8144.** Any citation of the Prefer
  header should account for that update.
- `[FACT]` **RFC 9457 requires no members at all.** Every member including
  `type` is optional; `type` defaults to `about:blank`. A "conforming" problem
  document can therefore be nearly empty, which means conformance to RFC 9457
  alone guarantees very little — the project must specify its own required
  subset.
- `[FACT]` **JSON Schema is not an IETF standard.** It carries Internet-Draft
  lineage with no RFC. Rules depending on it rest on an industry specification,
  not a standards-track document — a materially weaker authority class than
  RFC 9110, and it should be labeled accordingly.

---

## 3. Domain map — which layer governs what

The prompt requires this explicitly, and it is the single most useful artifact
for preventing rule duplication across threads.

| Concern | Governing layer | Consequence |
| --- | --- | --- |
| Method properties, status meaning, validators, caching, conditional requests | **HTTP (RFC 9110/9111)** | Not re-decidable here. See `baseline-01`. |
| Error serialization envelope | **Media type (RFC 9457)** | Standard chooses whether to adopt; shape of extensions is project policy. |
| Structural validation of payloads | **JSON Schema 2020-12** | Expressible and machine-checkable. |
| Endpoint/operation description, examples, parameter binding | **OpenAPI 3.2.0** | The contract artifact. |
| Field casing, envelope presence, ID format, enum evolution | **Shared convention** | Not expressible as a standards requirement; must be project policy, consistently applied and linted. |
| Money and time representation | **Convention + hard prohibitions** | Float prohibition is near-universal; the positive choice is policy. |
| Idempotency-key semantics | **Project policy only** | No live standard exists. |
| Pagination mechanics | **Convention** | RFC 8288 governs `Link` *if* headers are used; body-field alternatives are unstandardized. |

`[INFERENCE]` The practical rule this map yields: **if a convention cannot be
expressed in the OpenAPI document or enforced by a linter, it will drift.**
That is the argument for choosing conventions that are machine-checkable over
conventions that are merely documented.

---

## 4. Conventions, patterns, and the field's actual split

`[COMPARATIVE]` All rows from `survey-03-representations-errors` and
`survey-04-collections` unless noted.

| Axis | The split |
| --- | --- |
| **Casing** | `snake_case`: Stripe, GitHub, Twilio (bodies), Shopify, Zalando (mandated). `camelCase`: Microsoft Graph, Google (proto3-derived — `.proto` defines `lower_snake_case`, the JSON mapping emits `lowerCamelCase`, and parsers accept both). Twilio is internally inconsistent: snake_case bodies, PascalCase query params. |
| **Timestamps** | RFC 3339 strings: GitHub, Google, Microsoft, Zalando. Unix epoch **seconds** integers: Stripe. RFC 2822 strings: Twilio. |
| **Money** | Minor-unit integers: Stripe. Decimal numbers: Zalando. Decimal strings: Shopify. **No reputable reference uses floats.** |
| **Envelopes** | Typed list envelope + `object` discriminator: Stripe. Bare JSON arrays for collections: GitHub. Lightly wrapped: Twilio, Graph, Google, Shopify. Formalized envelope: JSON:API. Top-level object mandated, bare array forbidden: Zalando. |
| **Errors** | Proprietary at every commercial API. RFC 9457 mandated only by Zalando. Google `google.rpc.Status` and Microsoft OData-derived `error{code,message,target,details,innererror}` are the most fully specified proprietary models. |
| **Pagination** | Object-ID cursors: Stripe (`starting_after`/`ending_before`). Opaque server tokens: Google AIP-158 (`page_token`), Microsoft (`$skiptoken`), AWS (`NextToken`). Opaque `page_info` in a `Link` header: Shopify. Offset/page-number: GitHub, Twilio, Microsoft `$skip`, Zalando (permitted, discouraged). |
| **"Next" signal** | Body field: Stripe, Google, Microsoft, AWS, Twilio. `Link` header (RFC 8288): GitHub, Shopify. Zalando permits both. |
| **Idempotency** | Header: Stripe `Idempotency-Key`, Shopify. Body field: Google AIP-155 `request_id`, AWS `ClientToken`. Header pair: OASIS/Azure `Repeatability-*`. Natural via PUT/client IDs: GitHub. |
| **Enum evolution** | New value is breaking unless opted in: Stripe. `x-extensible-enum`: Zalando. Additive within major version: Google AIP. |
| **Bulk** | `$batch` (JSON, 20-item cap, 4 MB): Microsoft Graph. `207 Multi-Status`: Zalando. Deprecated global HTTP batch: Google. No generic batch endpoint: Stripe. |
| **Async** | Operation resource polled at `operations/{id}`: Google AIP-151. Header-driven `202` + `Azure-AsyncOperation`/`Location` + `Retry-After`: Azure. Largely synchronous, completion pushed by webhook: Stripe. |

---

## 5. Anti-patterns

| Anti-pattern | Concrete failure |
| --- | --- |
| **Inconsistent envelopes** | Some endpoints wrap, some do not. Every client needs per-endpoint special-casing; generated clients cannot share a deserializer. |
| **Ambiguous null vs omission** | `{"x": null}` and `{}` mean different things to PATCH (clear the field vs leave alone) but are conflated by most serializers. Silent data loss on round-trip. |
| **Floating-point money** | `0.1 + 0.2 ≠ 0.3` in IEEE 754. Produces off-by-a-cent errors that reconcile inconsistently across languages. Strongest consensus prohibition in this domain. |
| **Local-time timestamps** | A timestamp without an offset is unresolvable across zones and ambiguous across DST transitions. |
| **Unstable enum handling** | Clients that reject unknown enum values break whenever the server adds one, making every addition a breaking change. |
| **Duplicated error formats** | Multiple error shapes across one API force clients to try-parse. Common when gateway errors differ from application errors. |
| **Citing RFC 7807 as current** | Obsoleted by RFC 9457 (Jul 2023). Wire format is unchanged, so the error is citation hygiene rather than incompatibility — but a standard that cites an obsolete RFC undermines its own authority. |
| **Offset pagination on mutable high-volume data** | Concurrent inserts and deletes cause items to be skipped or repeated across pages. Not wrong for small stable sets; wrong silently for large mutable ones. |
| **Leaking storage or query syntax** | Exposing SQL fragments, ORM operators, or index names couples the contract to the implementation and becomes an injection surface. |
| **Arbitrary version placement** | Version markers scattered across path, header, and query within one API make routing and client configuration unpredictable. |
| **Permanent parallel versions** | Every retained major version multiplies test matrix and support burden without bound. |
| **Undocumented breaking changes** | Changes that violate the compatibility contract without notice; the failure surfaces in the client's production, not the provider's. |
| **Reused idempotency keys across differing payloads** | Without fingerprinting the request body, a reused key can replay a response for a *different* request. `[COMPARATIVE]` Stripe returns 400 `idempotency_error` on same-key/different-params, and 409 `idempotency_key_in_use` for a concurrent in-flight request — two distinct behaviors widely conflated into "409 idempotency_error" (`survey-05-reliability`). |
| **Unbounded deduplication** | Retaining every idempotency key forever is unimplementable. `[COMPARATIVE]` Stripe prunes at ≥24 hours. A retention window must be stated. |
| **Non-atomic bulk ambiguity** | A bulk endpoint that does not state whether it is all-or-nothing leaves the client unable to know what to retry after partial failure. |
| **202 with no status resource** | The client is told "accepted" and given no way to learn the outcome. |

---

## 6. Contested choices

| Choice | Recommendation | Alternatives and tradeoffs | Exceptions | Conf. |
| --- | --- | --- | --- | --- |
| **camelCase vs snake_case** | `[POLICY]` Pick one; apply to bodies **and** query params. Slight lean to `snake_case` — the larger camp, and Zalando's mandate gives body/query consistency. | camelCase suits JS/.NET-first consumers; Google's is a derived proto3 mapping, not a free choice. | A domain format that mandates its own casing. | Moderate |
| **Top-level envelope** | Always a top-level **object**; never a bare array. | Bare arrays (GitHub) cannot gain metadata without a breaking change. Full JSON:API envelope buys formalism at high verbosity. | None. | High |
| **String identifiers** | Strings, always. | Numeric IDs invite arithmetic, break past 2^53 in JS, and leak volume. `[COMPARATIVE]` typed prefixes (`cus_`, `evt_`) aid debugging. | Genuinely numeric domain quantities. | High |
| **Date/time precision** | RFC 3339 strings, UTC, explicit offset. State sub-second precision in the schema. | Epoch integers (Stripe) are compact but opaque and lose sub-second precision. RFC 2822 (Twilio) is legacy. | Date-only values where a time would be false precision. | High |
| **Unknown enum values** | Clients MUST tolerate unknown values. Document an extensibility marker. | Treating additions as breaking (Stripe) is safer for clients but makes evolution expensive. | Closed enums that are genuinely fixed (ISO country codes). | Moderate |
| **Link placement** | Body fields for pagination; `Link` header additionally where it aids generic clients. | `Link` (RFC 8288) is a Proposed Standard but a minority practice (GitHub, Shopify). Body fields dominate. | None. | Moderate |
| **Cursor vs offset** | Opaque, non-constructable cursors by default. | Offset is simpler and supports jump-to-page; it is incorrect on mutable high-volume data. | Small, stable, or user-facing paged UI. | High |
| **Filter grammars** | A constrained per-field parameter set. Avoid a general query DSL in v1. | A DSL (OData `$filter`) is expressive but a large surface, hard to lint, and an injection risk. | A documented search sub-resource. | Moderate |
| **Field masks / selection** | Support explicit field selection; keep it simple. | `[COMPARATIVE]` Stripe uses `expand[]` in place of selection — idiosyncratic. Google field masks are more complete. | None. | Low-moderate |
| **Version placement** | Handed to `baseline-03` for rollout; contract rule here is that placement must be **uniform** across the API. | See `survey-06`, whose two runs frame this differently — reconcile before ratifying. | None. | Moderate |
| **OpenAPI baseline** | **3.2.0**, accepting 3.1 documents. | 3.0 lacks JSON Schema 2020-12 alignment. Note the spec permits low-impact breaking changes in minor versions — so "3.x compatible" is not a safe blanket claim. | Tooling genuinely stuck on 3.0. | High |
| **Design-first vs code-first** | Design-first, with the OpenAPI document as source of truth and CI compatibility checks. | Code-first drifts silently and encodes implementation detail in the contract. | None. | Moderate |
| **Problem Details extensions** | Adopt RFC 9457 and define a required project subset plus a stable machine-readable `code`. | RFC 9457 requires **no** members, so bare conformance is nearly vacuous. | None. | High |
| **Atomic vs partial bulk** | State the mode explicitly per endpoint. Prefer atomic; use `207` only when partial success is genuinely meaningful. | `207` is RFC 4918 (WebDAV), not core HTTP — a currency caveat. | None. | Moderate |
| **Operation-resource lifecycle** | Every 202 returns an addressable operation resource with terminal states, expiry, and a failure representation. | Header-only polling (Azure) works but is less discoverable. | None. | High |

---

## 7. Proposed normative principles

Provisional IDs use the `AC-` prefix. **Proposals, not ratified policy.**

| ID | Str. | Rule | Rationale | Exceptions | Evidence | Conf. |
| --- | --- | --- | --- | --- | --- | --- |
| AC-001 | MUST | Publish an OpenAPI document (**3.1 or 3.2**) as the contract source of truth; validate payload structure with JSON Schema 2020-12. | Machine-checkable contract. | **Revised by `baseline-02b` (2026-07-25).** The original 3.2-only mandate is withdrawn: swagger-parser, Redoc, and openapi-generator have open unaddressed 3.2 issues, and Spectral **silently ignores** 3.2 constructs. Only Redocly CLI has full support. The JSON Schema 2020-12 pin is **strengthened** — it is now WG-adopted IETF work targeting Proposed Standard. | [OAS](https://spec.openapis.org/oas/latest.html), [JSON Schema](https://datatracker.ietf.org/doc/draft-ietf-jsonschema-json-schema/); leaf `baseline-02b` | Moderate (version) / High (dialect) |
| AC-002 | MUST | Treat the description document as authoritative and gate changes on an automated compatibility check. | Prevents silent drift between contract and implementation. | None. | [OAS](https://spec.openapis.org/oas/latest.html) | Moderate |
| AC-003 | MUST | Serialize all error responses as `application/problem+json` per RFC 9457. | One error shape across the API; a live standard rather than a proprietary format. | Errors emitted by infrastructure outside application control — which MUST be documented. **Strengthened by `baseline-02c` (2026-07-25):** ASP.NET Core emits Problem Details **by default**, Spring supports RFC 9457 via one property, and Microsoft's own framework default contradicts Microsoft Graph's proprietary shape — direct support for the historical-inertia reading. | [9457](https://datatracker.ietf.org/doc/rfc9457/); leaf `baseline-02c` | **High-moderate** (raised from moderate) |
| AC-004 | MUST | Require `type`, `title`, `status`, and a stable machine-readable `code` extension on every problem document. | RFC 9457 makes every member optional, so conformance alone guarantees almost nothing. | None. | [9457](https://datatracker.ietf.org/doc/rfc9457/) | High |
| AC-005 | MUST NOT | Cite RFC 7807. | Obsoleted by RFC 9457 (Jul 2023). | Historical notes explicitly labeled as such. | [9457](https://datatracker.ietf.org/doc/rfc9457/) | High |
| AC-006 | MUST | Return a JSON object at the top level of every response body. Never a bare array. | A bare array cannot gain pagination or metadata without a breaking change. | Non-JSON media types. | `survey-03` `[COMPARATIVE]` | High |
| AC-007 | MUST | Use one field-casing convention across bodies and query parameters. | Mixed casing (Twilio) forces per-surface handling. | None. | `survey-03` `[COMPARATIVE]` | Moderate — the casing itself is `[POLICY]` |
| AC-008 | MUST | Represent identifiers as strings. | Avoids arithmetic, JS 2^53 truncation, and volume leakage. | Genuinely numeric domain quantities. | `survey-03` `[COMPARATIVE]` | High |
| AC-009 | MUST | Represent timestamps as RFC 3339 strings with an explicit offset. | Local-time and epoch-integer forms are ambiguous or lossy. | Date-only fields. | [RFC 3339](https://datatracker.ietf.org/doc/rfc3339/) | High |
| AC-010 | MUST NOT | Represent monetary amounts as floating-point numbers. | IEEE 754 cannot represent decimal fractions exactly. | None. | `survey-03` `[COMPARATIVE]` — no reputable reference uses floats | High |
| AC-011 | MUST | Distinguish null from omission explicitly in PATCH semantics and state the rule in the contract. | The single most common source of silent data loss on partial update. | None. | `[INFERENCE]` | High |
| AC-012 | MUST | Require clients to tolerate unknown enum values; document additions as non-breaking. | Otherwise every enum addition is a breaking change. | Genuinely closed enumerations. | `survey-06` `[COMPARATIVE]` | Moderate |
| AC-013 | SHOULD | Paginate with opaque, non-constructable cursors. | Offset is incorrect under concurrent mutation. | Small or stable collections; UI needing jump-to-page. | `survey-04` `[COMPARATIVE]` | High |
| AC-014 | MUST | Return a top-level object carrying both items and continuation state for every collection. | Enables adding metadata without breaking clients. | None. | `survey-04` `[COMPARATIVE]` | High |
| AC-015 | MUST NOT | Expose storage or query-engine syntax in filter parameters. | Couples contract to implementation; injection surface. | None. | `[INFERENCE]` | High |
| AC-016 | MUST | Accept an idempotency key on non-idempotent state-changing requests, fingerprint the request payload, and reject a reused key carrying a different payload. | Prevents replaying a stored response for a different request. | Naturally idempotent operations (PUT with client-supplied ID). | **`[POLICY]`** — draft expired 2026-04-18; convergent vendor practice only | Moderate |
| AC-017 | MUST | State an explicit idempotency-key retention window. | Unbounded retention is unimplementable. | None. | `survey-05` `[COMPARATIVE]` (Stripe ≥24 h) | High |
| AC-018 | MUST | State explicitly whether each bulk endpoint is atomic or partial, and represent per-item outcomes when partial. | Otherwise the client cannot know what to retry. | None. | `survey-05` `[COMPARATIVE]` | High |
| AC-019 | MUST | Return an addressable operation resource from every 202, with terminal states, an expiry, and a failure representation. | A 202 with no status resource strands the client. | None. | `survey-05` `[COMPARATIVE]`; [9110 §15.3.3](https://www.rfc-editor.org/rfc/rfc9110.html#section-15.3.3) | High |
| AC-020 | MAY | Support RFC 7240 `Prefer: respond-async` and `return=minimal`/`representation`. | Standardized rather than proprietary toggles. | Preferences are non-binding; servers may ignore them. | [7240](https://datatracker.ietf.org/doc/rfc7240/) (updated by RFC 8144) | Moderate |
| AC-021 | SHOULD | Use RFC 6570 URI Templates where the contract must describe a URI family. | Avoids prose descriptions of URI construction; pairs with BCP 190 (see `baseline-01` HS-005). | None. | [6570](https://datatracker.ietf.org/doc/rfc6570/) | Moderate |

---

## 8. Conflicts and open questions

### 8.1 The RFC 9457 divergence — argued, not assumed

`[COMPARATIVE]` Seven of eight surveyed references ship proprietary error
shapes; only Zalando mandates RFC 9457. AC-003 therefore recommends the
**minority** practice, which requires justification rather than assertion.

`[INFERENCE]` The argument: vendor divergence here is explained by history, not
by a defect in RFC 9457. Stripe, GitHub, AWS, and Twilio all shipped error
formats before RFC 7807 existed (2016) and cannot change them without breaking
clients. Google's and Microsoft's shapes are derived from `google.rpc.Status`
and OData respectively — inherited from adjacent ecosystems, not chosen on the
merits. **None of the eight is a greenfield API that evaluated RFC 9457 and
rejected it.** A new standard carries no such legacy, so vendor prevalence is
weak evidence here.

`[FACT]` The genuine weakness is different and worth stating plainly: RFC 9457
requires *no* members, so conformance alone guarantees almost nothing. That is
what AC-004 addresses.

**Confidence: moderate.** The reasoning is sound but rests on an inference
about *why* vendors diverge. A reviewer who weighs field prevalence more
heavily could legitimately reach the opposite conclusion, and this belongs on
the Gate C agenda rather than being settled here.

### 8.2 Research-resolvable

- **OpenAPI 3.2 tooling maturity.** Secondary sources report support landing
  Q4 2025–Q1 2026; this was not verified against individual tool releases and
  is the weakest link in AC-001. A narrow leaf could settle it.
- **JSON Schema next draft.** Work-in-progress with no announced date. If it
  lands during drafting, AC-001's dialect pin needs revisiting.
- **`x-extensible-enum` interoperability.** Zalando's marker is a convention,
  not a standard; whether generators honor it is untested here.

### 8.3 Standards-level gap

`[FACT]` **Idempotency keys have no live standard.** The IETF draft expired
2026-04-18 with intended RFC status "(None)." Every rule in this area is
project policy grounded in convergent vendor practice. AC-016 is labeled
`[POLICY]` for exactly this reason, and the standard must not present it as
standards conformance. `[INFERENCE]` The header-vs-body split (Stripe header
vs Google `request_id` body field) has no authority to appeal to, so it is a
pure policy fork.

### 8.4 Organization policy

Casing choice · money representation (minor-unit integer vs decimal string) ·
field-selection syntax · filter grammar surface · retention window length ·
whether to publish a `Link` header alongside body pagination fields.

---

## 9. Dependency handoff

**To `baseline-01` (HTTP semantics)** — consumed, not re-decided: method
idempotence as a protocol property; status-code selection for validation
versus conflict (HS-011); validator and `If-Match` mechanics; cache directives.

**To `baseline-03` (operational practice)** — not answered here: where the
version marker lives and how rollout is communicated; deprecation and sunset
signaling; retry and backoff policy consuming AC-016's idempotency contract;
rate-limit representation; webhook envelope and signing, which consumes the
event representation but not the delivery mechanics.

---

## 10. Confidence and invalidating assumptions

**Overall confidence: moderate-to-high.** High for representation rules with
hard technical grounding (AC-006, AC-008 through AC-011, AC-013 through
AC-019). Moderate for AC-003 (argued divergence), AC-007 and AC-016 (policy),
AC-001 (tooling maturity unverified at tool level).

Assumptions that would materially change these recommendations:

1. **That the API is public and clients are not centrally controlled.** With
   first-party SDKs shipped in lockstep, AC-012's enum tolerance and AC-013's
   cursor requirement both weaken considerably.
2. **That collections are large and mutable.** AC-013 is motivated by
   concurrent mutation; a small stable catalogue can use offset safely.
3. **That clients generate code from the contract.** AC-001 and AC-002 assume
   generated clients. Hand-written-only consumers reduce the cost of contract
   drift and weaken the design-first argument.
4. **That backward compatibility matters over a multi-year horizon.** A short
   horizon makes AC-012 and the compatibility gate far less valuable.
5. **That no adjacent ecosystem constrains the format.** An organization
   already committed to OData or JSON:API inherits its conventions, and several
   rules here would yield.
6. **That idempotency standardization stays dead.** If the IETF work is revived
   and published, AC-016 should be re-grounded on the RFC and its `[POLICY]`
   label removed.

**Research completeness note.** This report recommends an actionable contract
baseline covering error, collection, compatibility, idempotency, bulk, and
asynchronous-operation patterns, with a version-aware OpenAPI policy, as the
prompt requires. Where it recommends against field practice (AC-003) it argues
the case and records its own confidence honestly rather than asserting
consensus that does not exist.
