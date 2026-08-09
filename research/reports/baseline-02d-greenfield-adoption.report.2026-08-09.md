# Baseline 02d — RFC 9457 Greenfield Adoption and Criticism

*Narrow leaf under `baseline-02`. Extends `baseline-02c` (2026-07-25). Does not
re-derive ASP.NET/Spring defaults, the Microsoft self-contradiction, or the
8-vendor sample-bias finding. Run 2026-08-09; all URLs accessed 2026-08-09.*

**Headline:** one finding overturns a labeled claim in `baseline-02c`. That
report recorded `[FACT — absence]` "No documented case was located of a
greenfield API evaluating RFC 9457 and choosing a proprietary shape on the
merits." Such a case exists and is fully documented with dates, named
participants, and a formal steering-committee decision: **CAMARA (Linux
Foundation)**. Details in D1 below.

---

## D1 — What new (2023+) APIs and API standards are doing

### 1.1 CAMARA — a post-RFC-9457 standards body that evaluated it and said no `[FACT]`

CAMARA is the Linux Foundation / GSMA telco API standardization project
(started 2022). It ran an explicit, minuted evaluation of RFC 9457 and
rejected it.

| Artifact | Dates | Outcome |
| --- | --- | --- |
| Issue #31 "Adopt RFC 7807 error responses…" | opened 2023-07-12, closed 2024-02-05 | superseded by #133 |
| Issue #133 "Adopt RFC 9457 error response format" | 2024-02-05 → 2024-03-05 | closed; replaced by #156/#157 |
| Issue #156 "Add reference documentation link to error responses" | 2024-03-05 → 2024-09-02 | closed by author: *"Closing as no interest. We will need to rely on good documentation."* |
| Issue #157 "Adopting RFC 9457 for Error Handling" | 2024-03-06 → 2024-04-15 | closed by maintainer `rartych` with the single comment **"not to be implemented"**, citing the TSC decision of 2024-04-04 |

`[FACT]` The **current** CAMARA API Design Guide (repo `main`, release train
through `r4.3`, published 2026-05-22) contains **zero** occurrences of `9457`,
`7807`, or `problem+json`. It specifies a proprietary `ErrorInfo` object: *"The
value of the `status` field is matching the numeric status code… the `code`
field is matching the numeric error code value… the `message` field is a human
understandable description. The `ErrorInfo` object is provided within the HTTP
body."*
- https://github.com/camaraproject/Commonalities/issues/133 · https://github.com/camaraproject/Commonalities/issues/157 · https://github.com/camaraproject/Commonalities/blob/main/documentation/CAMARA-API-Design-Guide.md

`[COMPARATIVE]` Named corporate positions recorded in #133 (Feb 2024) — this
is unusually clean vendor-practice evidence:

| Party | Position | Stated reason |
| --- | --- | --- |
| Vodafone (`eric-murray`) | adopt | wants a standard field for a documentation link |
| Nokia (`uwerauschenbach`) | adopt | *"painful change now but we have a solid foundation for the future"* |
| T-Mobile US (`gmuratk`) | adopt | standards compliance |
| Orange (`bigludo7`) | **reject** | *"as the RFC 9457 is not widely used in the industry we did not see enough value to change all CAMARA assets"* |
| Telefónica (`jlurien`) | **reject** | *"The current format is in line with the format used by big players and familiar to the developers, while the RFC is not widely adopted. The impact of this change at this moment is huge, as there are many integrations going on."* |
| Deutsche Telekom (`hdamker`, personal capacity) | **reject partial adoption** | merits objection, quoted in D4 below |

`[INFERENCE]` Characterize this precisely. By Feb 2024 CAMARA had already
shipped APIs on the proprietary shape, so two of the three rejection arguments
(migration cost, installed base) are the *same incumbency mechanism*
`baseline-02c` identified — CAMARA is an early-stage incumbent declining to
migrate, not a clean blank-page evaluation. But the third argument
(`hdamker`'s `type`-semantics objection) is a genuine merits criticism, made
with reference to the IETF httpapi mailing list. So: `baseline-02c`'s
`[FACT — absence]` is **falsified**; its underlying *inference* that
incumbency explains most divergence **survives partially**.

### 1.2 New commercial APIs — mixed, trending custom `[COMPARATIVE]`

- **Anthropic Claude API** (launched 2023, post-RFC-9457): proprietary shape
  `{"type":"error","error":{"type":…,"message":…},"request_id":…}`. No mention
  of problem+json. https://platform.claude.com/docs/en/api/errors
- **Cloudflare** (2026-03-11): **adopted**, network-wide. *"Starting today,
  Cloudflare returns RFC 9457-compliant structured Markdown and JSON error
  payloads to AI agents."* Covers all 1xxx error codes. Content-negotiated:
  `Accept: application/problem+json` returns
  `application/problem+json; charset=utf-8`; the payload uses all five
  standard members plus extension members (`error_code`, `ray_id`,
  `retryable`, `retry_after`, …). Browsers still get HTML. Measured for error
  1015: HTML 46,645 bytes / 14,252 tokens vs JSON 970 bytes / 256 tokens.
  https://blog.cloudflare.com/rfc-9457-agent-error-pages/
  - `[INFERENCE]` This is the newest and most operationally significant
    adoption datapoint in the corpus, it postdates all July-2026 research in
    this repo, and its motivation is **agent/LLM consumption**, not human
    developer experience. That is a new argument class for the mandate.
- **ACME / RFC 8555** (Standards Track, March 2019) `[FACT]`: an IETF protocol
  that *requires* problem documents with
  `Content-Type: application/problem+json`, with a protocol-specific
  `subproblems` extension member and `urn:ietf:params:acme:error:*` type URNs.
  Every ACME server (Let's Encrypt, ZeroSSL, Google Trust Services) therefore
  emits problem+json in production. https://www.rfc-editor.org/rfc/rfc8555.txt
  - `[INFERENCE]` Precedent value: the extension mechanism and URN-typed
    problems work at internet scale in a protocol with many independent
    implementations.

---

## D2 — Who mandates / recommends / supports it (government and industry guidelines)

`[FACT]` unless noted. Requirement strength is the load-bearing column.

| Guideline | Strength | RFC cited | Date verified |
| --- | --- | --- | --- |
| **Zalando RESTful API Guidelines**, rule `[#176]` | **MUST** | 9457 | repo `main`, 2026-08-09 |
| **Netherlands NLgov REST API Design Rules** — editor's draft | **MUST** | 9457 | draft dated 2026-07-09 |
| **Netherlands NLgov ADR v2.1.0** — *published, definitive* | **linter warning only** | 9457 | published 2025-08-27 |
| **Italy AgID Modello di Interoperabilità**, `RAC_REST_NAME_008` | **DEVE** (must) | **7807** (stale) | repo `master` |
| **Belgium Belgif REST guide**, rule `[err-problem]` | **SHOULD** | 9457 | page dated 2026-06-12 |
| **Australia api.gov.au** national API design standard | custom format, no mention | — | beta |
| **UK GDS** API technical and data standards | silent on format | — | updated 2024-07-19 |
| **US 18F / GSA** API standards | custom `{message, exception}` | — | repo `master` |
| **Google AIP-193** | **MUST** `google.rpc.Status` | none | last updated 2024-10-18 |

Verbatim quotes for the load-bearing rows:

- **Zalando #176 — `{MUST} support problem JSON`:** *"RFC 9457 defines a
  Problem JSON object using the media type `application/problem+json`… every
  endpoints must be capable of returning a Problem JSON on client usage errors
  (4xx status codes) as well as server side processing errors (5xx status
  codes)."* With an explicit carve-out: *"Clients must be robust and **not
  rely** on a Problem JSON object being returned, since (a) failure responses
  may be created by infrastructure components not aware of this guideline or
  (b) service may be unable to comply with this guideline in certain error
  situations."*
  https://github.com/zalando/restful-api-guidelines/blob/main/chapters/http-status-codes-and-errors.adoc
  - `[INFERENCE]` This is the best available model for *how* to word a
    mandate: the obligation is "capable of returning," and the infrastructure
    carve-out is written into the rule rather than left implicit. It matches
    `AC-003`'s exception clause almost exactly.
- **Netherlands, rule `/core/error-handling/problem-details`, class
  "technical":** *"Error responses with HTTP status codes `4xx` or `5xx` MUST
  use either `application/problem+json` or `application/problem+xml` as the
  `Content-Type` header, and the response body MUST conform to the structure
  defined in [RFC9457]. The following fields MUST be present: `status`,
  `title`, and `detail`."* https://logius-standaarden.github.io/API-Design-Rules/
  - **Do not flatten this to "the Netherlands mandates problem+json."** The
    published, definitive v2.1.0 — the version submitted to Forum
    Standaardisatie for the Dutch comply-or-explain list of mandatory
    standards — carries **no normative error rule**, only a Spectral linter
    rule `use-problem-schema` at **warning** level: *"The content type of an
    error response should be `application/problem+json` or
    `application/problem+xml` to match RFC 9457."*
    https://gitdocumentatie.logius.nl/publicatie/api/adr/2.1.0/
  - `[INFERENCE]` The dated trajectory (warning-level in the 2025-08-27
    published version → `MUST` in the 2026-07-09 draft) is itself the finding:
    a national government guideline is *tightening* toward a mandate during
    exactly the window this project is deciding in.
- **Belgium `[err-problem]`:** *"Any information on a problem SHOULD be
  provided in the Problem Detail format, as specified in Problem Details for
  HTTP APIs (RFC 9457, which obsoletes RFC 7807): the media type for problems
  SHOULD be `application/problem+json`…"* Changelog 2023-10-05: *"Problem
  Details for HTTP APIs is updated to RFC 9457 which obsoletes RFC 7807
  throughout the guide."* https://www.belgif.be/specification/rest/api-guide/
  - `[INFERENCE]` Belgif migrated its citations within ~3 months of RFC
    9457's publication. Contrast with Italy (still 7807) and Microsoft's mixed
    citations found in `baseline-02c` — further support for `AC-005`.
- **Italy `RAC_REST_NAME_008`:** *"In caso di errori si DEVONO ritornare: un
  payload di tipo Problem definito in :rfc:`7807`"* — mandatory, but cites the
  obsoleted RFC.
  https://github.com/italia/lg-modellointeroperabilita-docs/blob/master/doc/04_Raccomandazioni%20di%20implementazione/05_raccomandazioni-tecniche-per-rest/02_progettazione-e-naming.rst
- **Australia:** custom error collection
  `{id, detail, code, source{pointer, parameter}}`, *"The returned error
  objects must be in a collection (array)."* No mention of RFC 7807, RFC 9457,
  or problem+json. https://api.gov.au/sections/error-handling.html —
  `[INFERENCE]` the `source.pointer`/`source.parameter` members are JSON:API's
  error-object shape, not Problem Details.
- **UK GDS:** prescribes only *"Your error codes must be consistent and easy
  to read"* and a security caution about error content; no format, no RFC.
  https://www.gov.uk/guidance/gds-api-technical-and-data-standards
- **US 18F:** *"Handle all errors… and return a data structure in the same
  format as the rest of the API"*, example `{"message": …, "exception": …}`.
  https://github.com/18F/api-standards
- **OTTO (German retailer) guideline `r000040`** —
  `[COMPARATIVE, quoted-within-primary]`: cited by `hdamker` in CAMARA #133 as
  an organization that adopted RFC 9457 *and* defined how to build resolvable
  type URLs. Direct fetch of
  https://api.otto.de/portal/guidelines/r000040 returned no substantive
  content; treat as unverified.

`[INFERENCE]` The pattern is regional, not chronological: continental-EU
government guidelines (BE, NL, IT) converge on Problem Details; Anglosphere
guidelines (UK, AU, US) are silent or custom. A greenfield standard aiming at
broad adoption cannot claim "governments require this" without that qualifier.

---

## D3 — Framework support beyond ASP.NET/Spring

`[FACT]` **FastAPI** default error body is
`{"detail": [ {"loc": …, "msg": …, "type": …} ]}` for validation errors; its
error-handling documentation does not mention problem+json, RFC 7807, or RFC
9457. Support exists only via third-party packages
(`g0di/fastapi-problem-details`, `NRWLDev/fastapi-problem`,
`fastapi-rfc7807`). https://fastapi.tiangolo.com/tutorial/handling-errors/

`[COMPARATIVE, secondary]` For NestJS, Express, Laravel, Rails, Go, and
Quarkus only community packages were found, not framework defaults or
first-party opt-in switches: `@sjfrhafe/nest-problem-details` (npm),
`pedrosalpr/laravel-api-problem`, `t1/problem-details` (Java),
`quarkus-resteasy-problem`, `zalando/problem` (Java). The *specific* packages
were not verified against each framework's primary docs — treat them as
low-confidence and the *pattern* (community package, not default) as
moderate-confidence.

`[INFERENCE]` `baseline-02c` built its case on the framework-default proxy.
That proxy now looks narrower than it did: ASP.NET Core is the only surveyed
framework where Problem Details is the **default**, Spring is the only one
where it is a **first-party one-property switch**, and the fastest-growing
Python API framework emits a different shape by default with no first-party
option. This does not contradict `baseline-02c` (which only claimed ASP.NET
and Spring) but it materially limits how far that argument generalizes.

---

## D4 — Criticisms of the standard

The highest-quality criticism found is not in blogs — it is in the CAMARA
record, from named engineers, with RFC section citations. `[OPINION]`,
attributed.

1. **`type` is overloaded: stable identifier vs. resolvable documentation
   locator.** `hdamker` (Deutsche Telekom), 2024-02-28: *"This parameter is
   the **identifier** of a **problem definition type** — which means that
   every time you are using a different URI here you are defining a new
   problem definition type and you are breaking the API contract… There is a
   requirement within the RFCs that the URI value in `type` SHOULD be
   resolvable and point to some documentation, but seems to be difficult to
   bring together with the requirement that the identifier for a problem type
   must not change."* He adds: *"on the mailing list of RFC9457 there are
   several concerns about using the `type` member at the same time as an
   identifier AND as a resolvable URI pointing to a description. The main
   reasons brought by the editors to not split this into two members… was the
   backward compatibility to RFC 7807."*
   - **This criticism is corroborated by the behavior of two mandating
     adopters** `[FACT]`: Zalando — *"Problem type and instance identifiers
     in our APIs are not meant to be resolved. RFC 9457 encourages that
     problem types are URI references that point to human-readable
     documentation, **but** we deliberately decided against that… URLs tend
     to be fragile and not very stable."* Belgif — introduces a
     **non-standard `href` member** and states *"using `href` instead of
     `type` for documentation intentionally deviates from the recommendation
     in the RFC."* Two independent adopters both refuse the same `SHOULD`,
     and one invents a member to work around it.
2. **Partial adoption is worse than none.** `hdamker`: *"my personal position
   is that adapting the RFC 9457 partially without following the intention of
   the RFC and using 'problem types' as defined is worse than staying with
   our current proprietary structure."*
3. **`status` is redundant and gets misused.** `pjhac`, 2024-02-05: *"the
   'status', as defined in the RFC, has to be redundant of the HTTP Status
   Code, but I know by experience that poorly designed/implemented clients
   and servers are going to misuse it… If the server violates RFC9457, the
   client cannot really know it… which makes this additional 'status'
   meaningless."*
4. **No slot for an application error code.** `pjhac`: *"I'd personally find
   more useful to have an error 'code' (like
   `DEVICE_IDENTIFIER_TYPE_NOT_MANAGED`)… I do not see another parameter in
   the RFC that could fit such 'code'."* `jlurien` (Telefónica): *"renaming
   the keys from code/message to title/detail would [not] solve the problem,
   as long as we keep the same values."* The RFC's `about:blank` rule (title
   SHOULD equal the HTTP status phrase) collides with using `title` to carry
   an app-specific code.
5. **Code generators cannot populate `type`.** Recorded in #133 as a stated
   disadvantage: *"Populating additional fields such as `type` cannot be
   automated using code generators."*
6. **Media-type handling is a live interop hazard.** `[FACT]` Zalando's own
   hint: *"The media type `application/problem+json` is often not implemented
   as a subset of `application/json` by libraries and services! Thus clients
   need to include `application/problem+json` in the `Accept`-Header to
   trigger delivery of the extended failure information."*
7. **`[OPINION]`** HN, `Xelbair`, 2025-03-29, incorrectly believing
   problem+json is *"still just in draft stage"* while linking RFC 9457
   itself — a small but real awareness signal three years post-publication.

**Counter-evidence on criticism volume:** HN engagement is negligible. Algolia
search returns only 6 comments ever containing `problem+json` across all of
HN; the top RFC 9457 story (2023-08-15) has 13 points and 4 comments; a
2025-08-07 submission has 2 points and 0 comments. 2024–2026 blog coverage
(Swagger/SmartBear, Redocly, codecentric, frankel.ch) is uniformly
pro-adoption with no substantive dissent. `[INFERENCE]` Absence of blog
criticism here is weak evidence — it reflects low overall discussion volume,
not settled consensus. The real debate happened inside a standards body's
issue tracker, which is exactly where a blog-level search would miss it.

### Is there a credible alternative?

`[FACT]` **JSON:API v1.1 error objects** — an error object *"MAY have the
following members, and MUST contain at least one of: `id`, `links`, `status`,
`code`, `title`, `detail`, `source`, `meta`"*; errors are returned as an array
keyed by `errors`. The specification **does not reference RFC 7807 or RFC
9457** anywhere. https://jsonapi.org/format/
- `[INFERENCE]` It carries the *same* all-members-optional weakness routinely
  charged against Problem Details, and is not positioned as a competitor.
  Widely-cited claims that JSON:API errors "align with RFC 9457 so a client
  can treat the payload as either format" appear only in AI-generated
  secondary content (`jsonic.io`) that also misdates RFC 9457 to March 2023 —
  **do not use**; the primary spec contradicts the alignment claim by never
  mentioning the RFC.

`[FACT]` **Google AIP-193** mandates `google.rpc.Status` + required
`ErrorInfo` in `details`; no RFC 7807/9457 reference; created 2019-07-26, last
updated 2024-10-18. https://google.aip.dev/193 — `[INFERENCE]`
Google-internal governance; adopted outside Google only by parties copying
Google's style guide wholesale. Not a cross-vendor movement.

`[INFERENCE]` **No credible cross-vendor alternative format is gaining
traction.** The realistic alternative to mandating RFC 9457 is *bespoke
per-API envelopes*, which is what CAMARA, Australia, 18F, and Anthropic each
independently produced — and each produced a *different* one. That is the
strongest argument for a mandate in this report.

---

## D5 — Practical adoption signals

`[FACT]` **The IANA "HTTP Problem Types" registry is nearly empty.**
Established 2023-05-02, last updated 2026-06-26. Verified twice (XHTML page
and `http-problem-type-uris.csv`): **6 entries total** — `about:blank` (RFC
9457), two from RFC 9458 (Oblivious HTTP), and three from
`RFC-ietf-httpapi-digest-fields-problem-types-06`, an **unpublished draft**.
https://www.iana.org/assignments/http-problem-types/http-problem-types.xhtml
- `[INFERENCE]` The common problem-type registry was RFC 9457's headline
  addition over RFC 7807. Three years on it holds two published non-trivial
  entries, both from a single niche protocol. The registry is, empirically,
  not a working ecosystem asset — which weakens "adopt 9457 to get shared
  problem types" and strengthens the Zalando/Belgif approach of
  locally-defined, non-resolvable type URNs.

`[FACT]` **Linter support exists but is thin.** The Dutch ADR ships a Spectral
rule `use-problem-schema` in its published linter configuration.

`[FACT — absence]` Not found:
- default or built-in `application/problem+json` error emission in Kong,
  Apigee, Azure API Management, or AWS API Gateway;
- a problem-details rule in the Stoplight/`philsturgeon` Spectral OWASP API
  Security ruleset;
- OpenAPI code-generator special handling of `application/problem+json`
  (generators treat it as an ordinary media type with a schema);
- any observability vendor treating problem+json as a first-class signal;
- German, Swiss, or European-Commission API guidelines addressing the format.

`[COMPARATIVE]` Client-side parsing is manual almost everywhere. .NET's
`ProblemDetails` type can be deserialized with
`ReadFromJsonAsync<ProblemDetails>` after the caller checks
`Content-Type == "application/problem+json"` — i.e. the type exists, the
automatic dispatch does not. `treve` (HN, 2023-03-30) reports the
`badgateway/ketting` generic REST client extracting problem+json messages:
*"If you don't use `application/problem+json` you just get a HTTP errors, but
with `application/problem+json` we'll extract the human-readable error
message."*

`[INFERENCE]` Net tooling picture: **server-side emission is well supported**
(ASP.NET default, Spring switch, mature libraries in Java/Python/Node/PHP);
**client-side automatic consumption and gateway/observability integration are
not**. The practical benefit of a mandate accrues mainly to human readers, to
hand-written client error handling, and — new in 2026 — to LLM agents.

---

## Net assessment for the mandate decision

**Overall: the case for a `MUST` is roughly unchanged in strength, but its
*justification* must change.** The "everyone is converging on this" argument
is now demonstrably weaker than `baseline-02c` implied; the "there is no
alternative worth converging on instead" argument is now much stronger and
better evidenced. Both point at the same rule, for different reasons.

### Evidence that strengthens the mandate

1. `[FACT]` Three national/industry guidelines mandate or recommend it with
   post-2023 currency (Zalando `MUST`, NL draft `MUST`, Belgif `SHOULD`), and
   the Dutch guideline is actively *tightening* from warning-level (2025-08)
   to `MUST` (2026-07 draft).
2. `[FACT]` RFC 8555 (ACME) proves the format, its extension mechanism, and
   URN-typed problems work in a multi-implementation Standards Track protocol
   at internet scale.
3. `[FACT]` Cloudflare shipped RFC 9457 network-wide on 2026-03-11 with a
   measured 55–64× payload/token reduction for agent consumers. This is the
   newest large-vendor adoption in the corpus and postdates all prior
   research in this repo.
4. `[INFERENCE]` **No credible alternative exists.** JSON:API errors
   (primary-verified: no RFC reference, same all-optional weakness) and
   AIP-193 (Google-internal, no RFC reference) are not cross-vendor
   movements. The observed alternative in practice is *n* mutually
   incompatible bespoke envelopes.
5. `[FACT]` Zalando supplies a proven mandate *wording*: obligation is "must
   be capable of returning," with the infrastructure-generated-error
   carve-out written into the rule. `AC-003` can adopt this shape directly.

### Evidence that weakens the mandate

1. `[FACT]` **CAMARA — a post-9457 standards body — evaluated RFC 9457 twice
   and formally decided "not to be implemented"** (TSC, 2024-04-04/15), and
   its current guide (r4.3, 2026-05) still ships a proprietary `ErrorInfo`.
   This falsifies `baseline-02c`'s `[FACT — absence]`. Two of the three
   rejection arguments are incumbency; one (`type` semantics) is a merits
   objection.
2. `[FACT]` The IANA problem-types registry holds 6 entries after 3 years,
   half of them from an unpublished draft. The shared-vocabulary benefit is
   largely theoretical.
3. `[FACT]` Two mandating adopters (Zalando, Belgif) **both deliberately
   violate the RFC's resolvable-`type` `SHOULD`**, and one invents a
   non-standard `href` member. A mandate must therefore say what `type` is
   *for*, or inherit a known ambiguity.
4. `[FACT]` Framework defaults generalize less than `baseline-02c` implied:
   FastAPI's default is `{"detail": …}` with no first-party option.
5. `[FACT]` Anglosphere government guidelines are silent (UK, US) or use a
   different shape (AU, JSON:API-flavored).
6. `[FACT]` Media-type handling is an interop hazard by the mandating
   adopter's own admission (`application/problem+json` frequently not treated
   as a subset of `application/json`).

### Left unchanged

- Practitioner sentiment. Public discussion volume is too low to move
  confidence in either direction; the pro-adoption blog consensus (Swagger,
  Redocly, codecentric) is real but is secondary content with no dissenting
  counterpart, and the substantive debate happened where blogs did not look.
- Client-library and gateway support: thin before, thin now.

### Recommendation for `AC-003` at Gate C

`[RECOMMENDATION]` **Keep the `MUST`; lower the confidence from
`high-moderate` back toward `moderate`; and re-argue it.**

- `baseline-02c` raised confidence partly on an absence that no longer holds.
  That specific support must be withdrawn and the report annotated.
- The mandate should now be justified as *"no credible alternative exists and
  bespoke envelopes do not converge"* rather than *"the field is converging
  on 9457."* The first claim is well evidenced here; the second is not.
- Three drafting consequences follow directly from primary sources: (a) adopt
  Zalando's "capable of returning" + infrastructure carve-out wording; (b)
  **rule explicitly on `type`** — mandate stable identifiers (URN or stable
  URI), state whether they must be dereferenceable, and if not, provide a
  documentation member, because every mandating adopter examined had to solve
  this and two solved it by deviating from the RFC; (c) do not premise
  anything on the IANA registry.
- New consideration not present in the July research: `[INFERENCE]` the 2026
  agent-consumption argument (Cloudflare's measurement; CAMARA's open issue
  #587 reopening design guidelines for MCP/agent readiness, 2026-02-16) is a
  fresh and quantified benefit for machine-parseable error bodies. It
  deserves its own line in the rationale.

---

## Strongest sources

| # | Source | Class | Why it matters |
| --- | --- | --- | --- |
| 1 | https://github.com/camaraproject/Commonalities/issues/133 | primary (project record) | Named multi-operator deliberation, Feb 2024, with arguments on both sides |
| 2 | https://github.com/camaraproject/Commonalities/issues/157 | primary | Formal TSC decision "not to be implemented", 2024-04-15 |
| 3 | https://github.com/camaraproject/Commonalities/blob/main/documentation/CAMARA-API-Design-Guide.md | primary | Current guide (r4.3, 2026-05) still proprietary — the rejection held |
| 4 | https://github.com/zalando/restful-api-guidelines/blob/main/chapters/http-status-codes-and-errors.adoc | primary | The only surveyed `MUST`, verbatim, with carve-out wording and interop caveats |
| 5 | https://logius-standaarden.github.io/API-Design-Rules/ + https://gitdocumentatie.logius.nl/publicatie/api/adr/2.1.0/ | primary (pair) | Government guideline tightening warning→MUST, dated on both sides |
| 6 | https://www.belgif.be/specification/rest/api-guide/ | primary | `SHOULD`; fast 7807→9457 migration; documented deviation from the `type` SHOULD |
| 7 | https://www.iana.org/assignments/http-problem-types/http-problem-types.xhtml (+ CSV) | primary registry | 6 entries, 3 unpublished-draft; registry benefit is theoretical |
| 8 | https://blog.cloudflare.com/rfc-9457-agent-error-pages/ | vendor-primary | 2026-03-11 network-wide adoption + quantified agent-token benefit |
| 9 | https://www.rfc-editor.org/rfc/rfc8555.txt | IETF primary | Standards Track protocol mandating problem+json at internet scale |
| 10 | https://jsonapi.org/format/ and https://google.aip.dev/193 | primary (pair) | The two candidate alternatives; neither references RFC 9457; neither is cross-vendor |
