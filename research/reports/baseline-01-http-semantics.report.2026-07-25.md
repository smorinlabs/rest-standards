# Baseline 01 — HTTP Semantics and Resource Modeling

*Prescriptive research. Primary-source status verified live 2026-07-25 against
IETF Datatracker, RFC Editor, and IANA registries. Vendor-practice evidence is
carried from the in-repo `survey` series (retrieved 2026-07-19/20) and labeled
as such. This report **proposes** normative rules; nothing here is project
policy until ratified at Gate C.*

**Label key used throughout:** `[FACT]` sourced to a primary specification ·
`[COMPARATIVE]` deployed vendor practice, evidence of convention only ·
`[INFERENCE]` reasoned from sourced facts · `[RECOMMENDATION]` proposed rule ·
`[POLICY]` a choice this project must make that evidence cannot settle.

---

## 1. Executive recommendation

Design against **HTTP as a coherent application protocol whose generic elements
this standard consumes but never redefines**, and treat the resource — not the
operation — as the unit of design. Concretely, seven commitments:

1. **Bind to the consolidated core.** RFC 9110 (STD 97) and RFC 9111 (STD 98)
   are the sole normative source for method properties, status-code semantics,
   validators, conditional requests, and caching. The RFC 723x family is
   obsolete and must not be cited.
2. **Never overlay generic semantics.** RFC 9205 (BCP 56) forbids applications
   from redefining, refining, or overlaying methods, status codes, or existing
   header fields. This is a binding constraint on the standard itself, not
   merely a style preference, and it settles several contested areas outright.
3. **Own your URIs; do not let a standard own them.** RFC 8820 (BCP 190)
   places URI structure under the authority of the resource owner. This
   standard may specify *how to construct* URIs consistently; it must not
   mandate fixed path prefixes that constrain deployments.
4. **Adopt QUERY for safe, body-carrying reads.** RFC 10008 (June 2026)
   closes the GET-versus-POST gap with a method that is safe, idempotent, and
   cacheable while carrying a request body. This is new enough that it needs a
   migration posture, not immediate mandatory adoption.
5. **Make every mutation conditionally safe.** Strong validators plus
   `If-Match` are the interoperable answer to lost updates; the mechanism is
   near-universal in *form* and inconsistently applied in *practice*.
6. **Cache deliberately, in both directions.** Explicit freshness or explicit
   `no-store` — never silence. Silence delegates the decision to heuristic
   freshness in every intermediary on the path.
7. **Assume intermediaries exist.** Caches, gateways, and proxies are on the
   path whether or not the designer pictured them. `Vary`, `Cache-Control`,
   and authentication interact, and getting that interaction wrong leaks data
   across users.

**Overall confidence: high** for items 1–3 and 5–7, which rest on published
Internet Standards and BCPs. **Moderate** for item 4, which rests on a
three-month-old Proposed Standard with no deployed reference implementations
among the eight surveyed APIs.

---

## 2. Source-and-currency matrix

All rows verified 2026-07-25. Authority classes: **STD** = Internet Standard ·
**BCP** = Best Current Practice · **PS** = Proposed Standard · **REG** = IANA
registry · **ACADEMIC** = foundational non-normative.

| Source | Class | Date | Status verified today | URL |
| --- | --- | --- | --- | --- |
| RFC 9110 — HTTP Semantics | STD 97 | Jun 2022 | Internet Standard. Obsoletes RFC 2818, 7230 (in part), 7231, 7232, 7233, 7235, 7538, 7615, 7694. Updates RFC 3864. Errata exist. | https://datatracker.ietf.org/doc/rfc9110/ |
| RFC 9111 — HTTP Caching | STD 98 | Jun 2022 | Internet Standard. Obsoletes RFC 7234. **No updated-by documents.** | https://datatracker.ietf.org/doc/rfc9111/ |
| RFC 3986 — URI Generic Syntax | STD 66 | Jan 2005 | Internet Standard. Updated by RFC 8820. No bis draft replacing the core grammar. | https://datatracker.ietf.org/doc/rfc3986/ |
| RFC 9205 — Building Protocols with HTTP | BCP 56 | Jun 2022 | Best Current Practice. Obsoletes RFC 3205. | https://datatracker.ietf.org/doc/rfc9205/ |
| RFC 8820 — URI Design and Ownership | BCP 190 | Jun 2020 | Best Current Practice. Obsoletes RFC 7320. Updates RFC 3986. | https://datatracker.ietf.org/doc/rfc8820/ |
| RFC 10008 — The HTTP QUERY Method | PS | Jun 2026 | Proposed Standard. Authors Reschke, Snell, Bishop. **Three months old at research time.** | https://datatracker.ietf.org/doc/rfc10008/ |
| RFC 8288 — Web Linking | PS | Oct 2017 | Proposed Standard — *not* an Internet Standard. Obsoletes RFC 5988. No updated-by. | https://datatracker.ietf.org/doc/rfc8288/ |
| RFC 5789 — PATCH Method | PS | Mar 2010 | Proposed Standard. Not obsoleted. Defines PATCH as neither safe nor idempotent. | https://datatracker.ietf.org/doc/rfc5789/ |
| RFC 9651 — Structured Field Values | PS | Sep 2024 | Proposed Standard. Obsoletes RFC 8941. Adds Date and Display String types. | https://datatracker.ietf.org/doc/rfc9651/ |
| IANA HTTP Method Registry | REG | Updated 2026-06-17 | Current. QUERY registered (safe, idempotent) citing RFC 10008 §2. | https://www.iana.org/assignments/http-methods/http-methods.xhtml |
| IANA HTTP Status Code Registry | REG | Current | 422 now defined by RFC 9110 §15.5.21 (moved out of WebDAV). 207 remains RFC 4918. 428/429 remain RFC 6585. | https://www.iana.org/assignments/http-status-codes/http-status-codes.xhtml |
| Fielding, *Architectural Styles*, ch. 5 | ACADEMIC | 2000 | Foundational. Non-normative — defines REST as a style, imposes no wire requirements. | https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm |

### Currency corrections this thread produced

- `[FACT]` **422 Unprocessable Content is core HTTP.** The IANA registry cites
  RFC 9110 §15.5.21. It is no longer a WebDAV-only code, which removes the
  historical objection to using it in non-WebDAV APIs.
- `[FACT]` **`immutable` and `stale-while-revalidate` are not in RFC 9111.**
  Confirmed absent from its table of contents; they are separate extensions
  (RFC 8246 and RFC 5861 respectively). Any rule relying on them must cite the
  extension and accept narrower support.
- `[FACT]` **RFC 8288 is a Proposed Standard, not an Internet Standard.** It is
  routinely described as settled; its maturity level is one step below RFC
  9110/9111. This matters when weighing `Link`-header pagination against
  body-field alternatives.
- `[FACT]` **RFC 9110 obsoletes RFC 2818.** HTTPS guidance that cites RFC 2818
  is citing an obsolete document.

### Gap this thread closes against the `survey` series

`[FACT]` RFC 10008, RFC 9205, and RFC 8820 appear in **none** of the ten
`survey` reports (verified by search across `research/reports/`). This is a
consequence of the survey's descriptive mandate rather than a defect: no
surveyed vendor has shipped QUERY, and BCPs describing how to *design* a
protocol do not surface in a comparison of what vendors *shipped*. The
practical consequence is that the survey's treatment of complex-query design —
Stripe's separate Search language, the vendor split on filter grammars
(`survey-04-collections`) — documents a set of workarounds for a gap that
RFC 10008 now addresses directly.

---

## 3. REST constraints, and what they do and do not bind

`[FACT]` Fielding defines six constraints: client-server, stateless, cache,
uniform interface, layered system, and code-on-demand (optional). The uniform
interface has four sub-constraints: identification of resources; manipulation
of resources through representations; self-descriptive messages; and hypermedia
as the engine of application state.

`[FACT]` Fielding defines a resource as "any information that can be named" and
a representation as "a sequence of bytes, plus representation metadata to
describe those bytes."

`[INFERENCE]` The resource/representation distinction is the one piece of the
dissertation that carries direct wire consequences, because RFC 9110 §3.1 and
§3.2 restate it normatively. The remaining constraints are architectural
properties, not protocol requirements — a compliant HTTP API can violate
hypermedia entirely without violating any RFC.

`[COMPARATIVE]` All eight surveyed references operate at Richardson Level 2;
none implements Level 3 hypermedia as its primary contract
(`survey-01-foundations`, both runs, high confidence — the two independent runs
agree on this point).

`[RECOMMENDATION]` Treat the resource/representation distinction and the
stateless constraint as normative; treat hypermedia as an optional capability
with documented benefits rather than a conformance requirement. Claiming
"RESTful" as a marketing term while operating at Level 2 is the field norm; the
standard should describe what it actually requires rather than gesture at
Fielding.

---

## 4. Conventions and canonical patterns

### 4.1 Method selection

`[FACT]` Registered method properties, from the IANA registry (verified
2026-06-17 update):

| Method | Safe | Idempotent | Reference |
| --- | --- | --- | --- |
| GET | yes | yes | RFC 9110 §9.3.1 |
| HEAD | yes | yes | RFC 9110 §9.3.2 |
| POST | no | no | RFC 9110 §9.3.3 |
| PUT | no | yes | RFC 9110 §9.3.4 |
| DELETE | no | yes | RFC 9110 §9.3.5 |
| OPTIONS | yes | yes | RFC 9110 §9.3.7 |
| TRACE | yes | yes | RFC 9110 §9.3.8 |
| PATCH | no | no | RFC 5789 §2 |
| QUERY | yes | yes | RFC 10008 §2 |

`[FACT]` Safe is defined at RFC 9110 §9.2.1, idempotent at §9.2.2. Safety is a
statement about *requested* semantics, not a guarantee that no state changes —
logging and metrics do not make GET unsafe.

`[FACT]` RFC 10008 states QUERY responses are cacheable and that "a cache MAY
use it to satisfy subsequent QUERY requests," with the cache key incorporating
request content.

### 4.2 Conditional requests and concurrency

`[FACT]` ETag is defined at RFC 9110 §8.8.3, Last-Modified at §8.8.2,
conditional requests at §13, `If-Match` at §13.1.1, `If-None-Match` at §13.1.2,
and precondition precedence at §13.2.2.

`[COMPARATIVE]` The mechanism is near-consensus in form and split on placement
and obligation: Google AIP-154 puts `etag` in the body; Stripe offers no
optimistic-concurrency mechanism at all (`survey-05-reliability`).

### 4.3 Caching

`[FACT]` Storage rules at RFC 9111 §3, freshness at §4.2, freshness-lifetime
calculation at §4.2.1, heuristic freshness at §4.2.2, validation at §4.3,
invalidation at §4.4, response directives at §5.2.2.

`[INFERENCE]` Heuristic freshness (§4.2.2) is the reason silence is dangerous.
A response with no explicit freshness information is not thereby uncacheable —
a cache may apply a heuristic. Omitting `Cache-Control` is a decision to let
every intermediary choose for you.

---

## 5. Anti-patterns

Each entry states the failure mode concretely. Items the prompt required are
all addressed; two are qualified rather than endorsed.

| Anti-pattern | Concrete failure | Governing source |
| --- | --- | --- |
| **RPC-shaped action proliferation** | `/createUser`, `/getUserById` defeat caching, method-based routing, and intermediary retry logic. Every operation becomes opaque POST. | RFC 9205 §4.4 forbids overlaying generic semantics |
| **Method tunneling** | `POST /resource?_method=DELETE` hides the real operation. Caches and proxies see POST and cannot apply DELETE's invalidation (RFC 9111 §4.4). | RFC 9111 §4.4 |
| **Unsafe behavior behind safe methods** | A GET that mutates gets replayed by prefetchers, crawlers, and retrying proxies. Safety is a contract with *intermediaries*, not just clients. | RFC 9110 §9.2.1 |
| **Indiscriminate 200** | `200 OK` carrying `{"error": ...}` breaks every generic client, cache, and monitor that reads the status line. | RFC 9205 §4.6 |
| **Invented status semantics** | Custom codes, or redefining registered ones, break intermediaries that act on class. RFC 9205 explicitly prohibits both. | RFC 9205 §4.6 |
| **Identifier leakage into mutable paths** | Embedding a mutable attribute (email, slug, tenant name) in the URI means identity changes when the attribute changes, breaking links and caches. | RFC 8820; RFC 9110 §3.1 |
| **Weak or missing validators** | Without a strong validator, `If-Match` cannot protect a mutation; weak validators are valid for caching but not for range or update preconditions. | RFC 9110 §8.8.1, §13.1.1 |
| **Lost updates** | Read-modify-write with no precondition silently discards a concurrent write. The client that wrote second wins and nobody is told. | RFC 9110 §13.1.1 |
| **Cache disabling by default** | Blanket `no-store` on everything discards the single largest available performance and availability lever, usually to avoid thinking about §4.2. | RFC 9111 §5.2.2.5 |
| **Incorrect `Vary` use** | Omitting `Vary` on a negotiated response serves one client's representation to another. Including `Vary: *` or over-broad values makes the response effectively uncacheable. | RFC 9110 §12.5.5 |
| **Assuming intermediaries are absent** | Authenticated responses without `private` or `no-store` can be stored by a shared cache and served to a different user. This is a data-leak class, not a performance class. | RFC 9111 §3, §5.2.2.7 |

`[INFERENCE]` **Two required items need qualification rather than endorsement.**
"Trailing slashes" and "path depth" are frequently listed as anti-patterns but
have no primary-source basis. RFC 3986 treats `/x` and `/x/` as distinct URIs
and expresses no preference; RFC 8820 actively discourages a standard from
constraining path structure at all. Both are consistency policy, not protocol
correctness, and are routed to §7 accordingly.

---

## 6. Contested areas

The prompt required each of these be addressed without forcing false consensus.
Verdict column: **SETTLED** by a primary source · **DEFAULT** where evidence
supports a default with documented exceptions · **POLICY** where evidence
cannot settle it.

| Area | Verdict | Position and basis |
| --- | --- | --- |
| **Nouns / pluralization** | POLICY | No RFC addresses it. `[COMPARATIVE]` plural is near-universal: AIP mandates it, Zalando mandates it, GitHub/Stripe/Twilio/Shopify use it (`survey-02-structure`). Adopt plural for consistency, not correctness. |
| **Path depth** | POLICY | Unaddressed by any source. RFC 8820 discourages standards constraining structure. Deep nesting couples identity to hierarchy, which hurts when the hierarchy changes — an `[INFERENCE]`, not a rule. |
| **Trailing slashes** | POLICY | `[FACT]` RFC 3986 makes `/x` and `/x/` distinct. Pick one and redirect the other with 308; the choice itself is arbitrary. |
| **Actions that resist CRUD** | DEFAULT | `[FACT]` RFC 9205 §4.4 forbids overlaying method semantics, which rules out inventing verbs *as methods*. It does not forbid an action sub-resource. `[COMPARATIVE]` the field splits three ways: `:verb` (Google), sub-path (GitHub/Stripe), body flag (`survey-02-structure`). Default to a POST action sub-resource; the syntax choice is POLICY. |
| **PUT vs PATCH** | SETTLED (properties) / POLICY (choice) | `[FACT]` PUT is idempotent, PATCH is not (RFC 5789 §2; IANA registry). A client may make a PATCH conditionally safe with `If-Match`. Use PUT for full replacement, PATCH for partial; the *patch format* is `baseline-02`'s call. |
| **DELETE response** | DEFAULT | `[FACT]` 204 and 200 are both valid (RFC 9110 §15.3.5, §15.3.1). `[COMPARATIVE]` Stripe returns a `deleted: true` body with 200 (`survey-02-structure`). Default 204 with no body; 200 with a body is a documented exception where the client needs final state. |
| **404 vs 410** | SETTLED | `[FACT]` RFC 9110 §15.5.11: 410 means the condition is "likely to be permanent" and the server knows it. 404 makes no such claim. Use 410 only when permanence is actually known and recorded; otherwise 404. Most APIs cannot honestly assert permanence. |
| **409 vs 422** | SETTLED (semantics) | `[FACT]` 409 (§15.5.10) is a conflict with the *current state of the resource*; 422 (§15.5.21) is a request that is syntactically valid but semantically unprocessable. `[FACT]` 422 is now core HTTP, not WebDAV. Validation failures → 422; state conflicts (version mismatch, duplicate, illegal transition) → 409. |
| **202 semantics** | SETTLED (meaning) / POLICY (shape) | `[FACT]` 202 means accepted, processing not complete, and the outcome may still fail (§15.3.3). `[INFERENCE]` 202 without a status resource strands the client. The status-resource shape belongs to `baseline-02`. |
| **Redirects** | SETTLED | `[FACT]` 307 and 308 preserve method and body; 301/302 have a documented history of method rewriting to GET; 303 explicitly directs a GET to another resource (§15.4.4, §15.4.8, §15.4.9). For APIs use 307/308; use 303 deliberately for post-action redirection. |
| **Link relations** | DEFAULT | `[FACT]` RFC 8288 is a Proposed Standard, one maturity step below RFC 9110. `[COMPARATIVE]` `Link`-header pagination is a minority: GitHub and Shopify use it; Stripe, Google, Microsoft, AWS, Twilio use body fields (`survey-04-collections`). Registered relation names SHOULD be used where one exists; header-vs-body placement is `baseline-02`'s call. |
| **Caching authenticated / mutable resources** | SETTLED (mechanism) / POLICY (aggression) | `[FACT]` shared caches storing authenticated responses without `private`/`no-store` is a cross-user leak (RFC 9111 §3, §5.2.2.7). Mechanism is settled; how aggressively to cache mutable data is a risk decision. |

---

## 7. Proposed normative principles

Provisional IDs use the `HS-` prefix (HTTP Semantics). **These are proposals
carrying research confidence, not ratified policy.** Strength follows RFC
2119/8174.

| ID | Str. | Rule | Rationale | Exceptions | Evidence | Conf. |
| --- | --- | --- | --- | --- | --- | --- |
| HS-001 | MUST | Cite RFC 9110/9111 as the normative source for method, status, validator, and caching semantics. Do not cite the RFC 723x family. | 723x is obsoleted by STD 97/98. | None. | [9110](https://datatracker.ietf.org/doc/rfc9110/), [9111](https://datatracker.ietf.org/doc/rfc9111/) | High |
| HS-002 | MUST NOT | Redefine, refine, or overlay the semantics of any registered method, status code, or header field. | RFC 9205 §4.4 states this as a binding requirement on applications using HTTP. | None. | [9205](https://datatracker.ietf.org/doc/rfc9205/) | High |
| HS-003 | MUST NOT | Define or use unregistered status codes. | RFC 9205 §4.6; intermediaries act on class and registered meaning. | None. | [9205](https://datatracker.ietf.org/doc/rfc9205/) | High |
| HS-004 | MUST | Identify each resource by a URI that remains stable across changes to its mutable attributes. | Identity coupled to mutable data breaks links, caches, and references when the attribute changes. | Deliberately versioned or dated resources. | [9110 §3.1](https://www.rfc-editor.org/rfc/rfc9110.html#section-3.1), [8820](https://datatracker.ietf.org/doc/rfc8820/) | High |
| HS-005 | MUST NOT | Mandate a fixed URI path prefix that constrains a deployment's URI space. | RFC 8820 (BCP 190) places URI structure under the owner's authority. | The standard may specify *construction rules* applied within an owner's own space. | [8820](https://datatracker.ietf.org/doc/rfc8820/) | High |
| HS-006 | MUST | Use methods per their registered safety and idempotence. Never perform requested state change behind a safe method. | Safety is a contract with intermediaries, which prefetch and retry. | Logging/metrics side effects do not violate safety. | [9110 §9.2.1](https://www.rfc-editor.org/rfc/rfc9110.html#section-9.2.1), [IANA](https://www.iana.org/assignments/http-methods/http-methods.xhtml) | High |
| HS-007 | MUST NOT | Tunnel a method through another method (`?_method=`, action-in-body dispatch). | Defeats cache invalidation (RFC 9111 §4.4) and intermediary handling. | None. | [9111 §4.4](https://www.rfc-editor.org/rfc/rfc9111.html#section-4.4) | High |
| HS-008 | SHOULD | Use PUT for full replacement and PATCH for partial modification. | PUT is idempotent; PATCH is not. | Patch document format is out of scope here — see `baseline-02`. | [5789](https://datatracker.ietf.org/doc/rfc5789/) | High |
| HS-009 | MAY | Use QUERY (RFC 10008) for safe, idempotent, body-carrying reads instead of overloading POST. | Closes the GET/POST gap; safe, idempotent, and cacheable with a body. | **Confirmed MAY by `baseline-01b` (2026-07-25).** Parsers accept it (llhttp `HTTP_QUERY = 46`) and curl works via `-X`, but MDN documents no browser support, Spring's PR is open and blocked targeting Nov 2026, and **no CDN has announced body-keyed caching** — the one capability that makes QUERY preferable to POST. | [10008](https://datatracker.ietf.org/doc/rfc10008/); leaf `baseline-01b` | Moderate |
| HS-010 | MUST | Return a status code whose registered semantics match the outcome. Never return 2xx for a failed operation. | Generic clients, caches, and monitors act on the status line. | None. | [9205](https://datatracker.ietf.org/doc/rfc9205/) | High |
| HS-011 | SHOULD | Use 422 for semantically invalid but well-formed requests; 409 for conflicts with current resource state. | 422 is now core HTTP (§15.5.21), removing the WebDAV objection. | 400 remains correct for malformed syntax. | [IANA](https://www.iana.org/assignments/http-status-codes/http-status-codes.xhtml), [9110 §15.5.21](https://www.rfc-editor.org/rfc/rfc9110.html#section-15.5.21) | High |
| HS-012 | SHOULD NOT | Return 410 unless permanence is actually known and recorded. | 410 asserts a condition "likely to be permanent"; most systems cannot honestly assert it. | Tombstoned resources with retained deletion records. | [9110 §15.5.11](https://www.rfc-editor.org/rfc/rfc9110.html#section-15.5.11) | High |
| HS-013 | MUST | Use 307/308 where method and body must be preserved across a redirect. | 301/302 have a documented history of rewriting the method to GET. | 303 where a GET on a different resource is genuinely intended. | [9110 §15.4](https://www.rfc-editor.org/rfc/rfc9110.html#section-15.4) | High |
| HS-014 | MUST | Emit a strong `ETag` on any resource that supports conditional update. | Weak validators are valid for caching but insufficient for `If-Match`. | Resources that are never updated by clients. | [9110 §8.8.3](https://www.rfc-editor.org/rfc/rfc9110.html#section-8.8.3), [§13.1.1](https://www.rfc-editor.org/rfc/rfc9110.html#section-13.1.1) | High |
| HS-015 | SHOULD | Require `If-Match` on unsafe requests to resources exposed to concurrent modification, returning 412 on mismatch. | The interoperable defence against lost updates. | Single-writer resources; append-only collections. | [9110 §13.1.1](https://www.rfc-editor.org/rfc/rfc9110.html#section-13.1.1) | High |
| HS-016 | MUST | Send explicit freshness information or explicit `no-store` on every response. Silence is not a decision. | Heuristic freshness (§4.2.2) means an unlabeled response may still be cached by intermediaries. | None. | [9111 §4.2.2](https://www.rfc-editor.org/rfc/rfc9111.html#section-4.2.2) | High |
| HS-017 | MUST | Mark responses carrying user-specific or authenticated data `private` or `no-store`. | A shared cache may otherwise serve one user's data to another. This is a data-leak class. | Genuinely public responses. | [9111 §3](https://www.rfc-editor.org/rfc/rfc9111.html#section-3), [§5.2.2.7](https://www.rfc-editor.org/rfc/rfc9111.html#section-5.2.2.7) | High |
| HS-018 | MUST | Send `Vary` listing every request header that influenced content selection. | Omission serves the wrong representation to the wrong client. | Responses with no negotiation. | [9110 §12.5.5](https://www.rfc-editor.org/rfc/rfc9110.html#section-12.5.5) | High |
| HS-019 | SHOULD | Define new header fields using RFC 9651 structured field types. | Avoids each field inventing parsing rules; RFC 9651 obsoletes RFC 8941. | Fields that must match an existing deployed non-structured syntax. | [9651](https://datatracker.ietf.org/doc/rfc9651/) | Moderate |
| HS-020 | SHOULD | Use registered link relation types where one exists for the semantic. | Registered relations are interoperable; invented ones are not. | No registered relation fits. | [8288](https://datatracker.ietf.org/doc/rfc8288/) | Moderate — RFC 8288 is PS, and body-field links dominate in practice |

---

## 8. Conflicts and open questions

### 8.1 Research-resolvable

- **QUERY intermediary support.** RFC 10008 is normatively clear; what is
  unknown is whether deployed caches, gateways, and CDNs handle a
  content-keyed cache correctly. This is an empirical question that a narrow
  follow-up leaf could answer, and it is the sole reason HS-009 is MAY rather
  than SHOULD. Suggested leaf: `baseline-01b-query-deployment`.
- **Strong-validator generation cost.** Whether a strong ETag can be produced
  cheaply depends on storage design. Evidence here is thin and mostly vendor
  anecdote.

### 8.2 Genuine standards tension

- **RFC 8820 versus the purpose of this document.** BCP 190 says specifications
  must not constrain URI structure; a design standard's value lies partly in
  prescribing consistent URI construction. `[INFERENCE]` The resolution is
  scope: BCP 190 restrains *interoperability standards* from constraining
  *other parties'* URI spaces. A house standard constraining its own
  deployments is not the harm BCP 190 addresses. This reading should be stated
  explicitly in the standard rather than left implicit, because a reader who
  knows BCP 190 will otherwise see a conflict.
- **Hypermedia.** Fielding's uniform interface requires HATEOAS; every surveyed
  API declines it. The standard cannot both claim Fielding conformance and
  describe the field. Recommend describing what is required and noting the
  divergence openly.

### 8.3 Organization policy — not research-resolvable

Pluralization · path depth limits · trailing-slash convention · custom-action
syntax (`:verb` vs sub-path) · DELETE response body · caching aggressiveness
for mutable data. Evidence describes practice; none of it establishes
correctness. These belong in the Gate C decision pass.

---

## 9. Dependency handoff

**To `baseline-02` (API contracts)** — not answered here: patch document format
(JSON Patch vs Merge Patch); error body shape for 4xx, including whether RFC
9457 is mandated; the async operation-resource model behind 202; pagination
cursor mechanics and whether links go in `Link` or the body; representation of
`etag` if a body copy is also carried.

**To `baseline-03` (operational practice)** — not answered here: retry and
backoff policy given which methods are idempotent; overload behaviour and 429
versus 503; rate-limit header policy; deprecation and sunset signaling;
authentication scheme choice and its interaction with HS-017.

---

## 10. Confidence and invalidating assumptions

**Overall confidence: high** for the protocol-binding rules (HS-001 through
HS-008, HS-010 through HS-018), which rest on Internet Standards and BCPs
verified live today. **Moderate** for HS-009 (RFC 10008 adoption), HS-019
(structured fields), and HS-020 (link relations), each for a stated reason.

Assumptions that would materially change these recommendations:

1. **That intermediaries exist on the path.** If every deployment were strictly
   point-to-point with TLS termination at the origin and no shared cache,
   HS-016 through HS-018 would drop from MUST to SHOULD. This assumption holds
   for public APIs and should be stated rather than relied on silently.
2. **That clients are heterogeneous and not centrally controlled.** If every
   client were a first-party SDK shipped in lockstep with the server, the cost
   of non-standard semantics would fall sharply and most of §6 would become
   pure preference.
3. **That the API is public or crosses a trust boundary.** Internal APIs behind
   a single trust boundary can rationally accept weaker cache-safety rules.
4. **That resources have meaningful concurrent-write exposure.** HS-015 is
   motivated by lost updates; single-writer systems can justify omitting it.
5. **That QUERY adoption proceeds.** If RFC 10008 fails to gain intermediary
   support over the next year, HS-009 should be withdrawn rather than promoted.
   If it gains rapid support, HS-009 should be reconsidered as SHOULD.

**Research completeness note.** This report recommends an actionable baseline
rather than cataloguing alternatives, as the prompt requires. Where it declines
to recommend — pluralization, path depth, trailing slashes, action syntax — it
does so because no primary source addresses the question, and it routes those
to Gate C as explicit policy choices rather than manufacturing a rule and
presenting it as protocol law.
