# Baseline 03f — IETF RateLimit Draft: Trajectory, Ecosystem, and Alternatives

*Narrow leaf under `baseline-03`, companion to `baseline-03e` (field
survey). Establishes the draft's real process trajectory, which revisions
the ecosystem actually implements, and what is normatively available from
published standards instead. Run 2026-08-09; all URLs accessed 2026-08-09.*

**Labels:** `[FACT]` = quoted/observed from a primary source;
`[COMPARATIVE]` = vendor practice as comparative evidence; `[INFERENCE]` =
reading; `[POLICY]` = project choice.

**Evidence protocol note, and a warning.** A web-fetch summarizer
**fabricated a normative sentence** during this run, reporting that RFC 9110
§10.2.3 says *"A server MAY send a Retry-After header field with any
response status code, but SHOULD send it with 503, 429, and 3xx responses."*
**That sentence does not exist in RFC 9110.** It was caught, and thereafter
every load-bearing quote was re-verified by downloading raw `.txt` and
grepping. Every RFC, draft, and registry quote below is raw-text verified.
Treat any summarizer-mediated normative quote in this project as untrusted.

---

## Verdict first

**The evidence supports rule shape (b) — MUST `Retry-After` on 429 per
published standards, SHOULD the pinned draft fields — with the contingency
re-keyed off expiry entirely.**

Rule shape (a) — MUST emit the IETF fields now — is **rejected by the
evidence**, on four independent grounds, any one sufficient:

1. `[FACT]` **RFC 2026 §2.2 forbids exactly this act**, in a boxed callout:
   *"Under no circumstances should an Internet-Draft be referenced by any
   paper, report, or Request-for-Proposal, nor should a vendor claim
   compliance with an Internet-Draft."*
2. `[FACT]` **The wire format is unsettled by the editors' own hand.** Open
   PR #166 renames the wire parameters `r`→`a` and `t`→`w`; open issue #158
   would unquote every policy identifier.
3. `[FACT]` **The draft itself makes emission optional:** *"A server MAY
   return RateLimit header fields independently of the response status
   code."*
4. `[FACT]` **Neither field is in the IANA HTTP Field Name Registry** — not
   even provisionally, in a registry that does list 23 provisional entries.

Rule shape (c) — documented proprietary scheme — is **also rejected**:
`X-RateLimit-Reset` carries three incompatible meanings under a
case-insensitively identical field name (Q4). Mandating it would mandate an
ambiguity.

---

## Corrections to established context

| Claim | Status | Correction |
|---|---|---|
| "draft-9 renamed the triple to two combined structured fields" | **Wrong** | The rename happened in **two steps, neither at -09**: draft-**07** (2023-06-24) dropped the triple for a Dictionary; draft-**08** (2024-10-07) replaced that with quoted-name Item lists. Drafts 09–11 changed no field names. |
| "Intended RFC status: (None)" ⇒ no standards intent | **Nuance, not error** | Datatracker's `intended_std_level` is `null`; the draft's own header reads `Intended status: Standards Track`. The unset datatracker field signals process momentum (an AD would set it), not author intent. |
| `baseline-03b`: "an editorial objection means what it specifies is probably stable" | **Falsified** | The editors' response to the HTTPDIR review (PR #166) renames wire parameters — see Q2. |
| `baseline-03b`: re-check trigger 2026-11-24 (expiry) | **Wrong trigger** | The draft has expired and revived **three times**; RFC 2026 §2.2 restarts the clock on any new revision. See revival mechanics. |
| `baseline-03e` note that the 2026-11-24 trigger "remains the right instrument" | **Overridden** | That leaf did not have the expiry-revival history in view. The revision history governs. Conflict surfaced per project rules, not averaged. |

---

## Q1 — Draft mechanics and the wire-format generations

`[FACT]` There are **three shipped wire formats and a fourth pending**,
verified by downloading each revision's raw text and grepping section
headings.

**Generation A — drafts 00–06** (last: draft-06, 2022-12-22). Section
headings `3.1 RateLimit-Limit`, `3.2 RateLimit-Policy`,
`3.3 RateLimit-Remaining`, `3.4 RateLimit-Reset`. `RateLimit-Policy` was
added in draft-04 (2022-05-30) — changelog verbatim: *"Split policy
informatio in RateLimit-Policy #81"* (typo in original).

```
RateLimit-Limit: 5000
RateLimit-Policy: 1000;w=3600, 5000;w=86400
RateLimit-Remaining: 100
RateLimit-Reset: 36000
```

**Generation B7 — draft-07 only** (2023-06-24 → expired 2023-12-29).
Headings collapse to `3.1 RateLimit` and `3.5 RateLimit-Policy`; §3.1
verbatim: *"The field is a Dictionary."*

```
RateLimit-Policy: 100;w=60
RateLimit: limit=100, remaining=50, reset=5
```

**Generation C — drafts 08–11** (draft-11 = 2026-05-23, current). draft-08
changelog verbatim: *"Refactored both fields to lists of Items that
identify policy and use parameters; Added quota unit parameter; Added
partition key parameter."*

```
RateLimit-Policy: "burst";q=100;w=60,"daily";q=1000;w=86400
RateLimit: "default";r=50;t=30
RateLimit-Policy: "peruser";q=65535;qu="content-bytes";w=10;pk=:sdfjLJUOUH==:

HTTP/1.1 429 Too Many Requests
RateLimit: "default";r=0;t=5
```

Parameters: `RateLimit-Policy` takes `q` (required), `w`, `qu` (default
`requests`), `pk`; `RateLimit` takes `r` (required), `t`, `pk`.

**Generation D — pending, not merged.** Open PR #166 ("Address HTTPDIR
review feedback and add structured partition keys", opened 2026-04-20,
untouched since 2026-04-22) proposes, verbatim from its own body:

> **Wire parameter rename**: r to a (available quota), t to w (effective
> window)

plus a new `RateLimit-Partition` header field, a cost parameter `c`, and 5
new IANA registries. Separately, open issue #158 / PR #159 would change
every policy identifier from `"burst"` to `burst` (SF String → SF Token) —
presented by the editor at IETF 124.

`[INFERENCE]` **A config option naming a draft number is only interpretable
against this table.** `draft-6` means Generation A; `draft-7` means the B7
dictionary — a format that existed for six months and appears in no
revision before or since; `draft-8` means Generation C.

### The draft's own accounting of proprietary practice

`[FACT]` draft-11's appendix (marked *"This section is to be removed before
publishing as an RFC"*) documents the epoch-vs-delta ambiguity itself:

> *  X-RateLimit-Remaining references different values, depending on the
>    implementation:
>    -  seconds remaining to the window expiration
>    -  milliseconds remaining to the window expiration
>    -  seconds since UTC, in UNIX Timestamp [UNIX]
>    -  a datetime, either IMF-fixdate [HTTP] or [RFC3339]

`[INFERENCE]` The draft says `X-RateLimit-Remaining` where it plainly means
`-Reset` (a remaining *count* has no datetime form) — an editorial defect
consistent with the HTTPDIR verdict. Two further defects in the current
revision: §5.1 prints `HTTP/1.1 429 Bad Request` (wrong reason phrase), and
§3.1.2 defines the default quota unit as `requests` while the §10.3 IANA
table registers `request` — a wire-value ambiguity in a field the draft
says `MUST be a String`.

---

## Q2 — Process trajectory: active editing, zero advancement

### Revision and expiry history `[FACT]`

| Rev | Date | Event |
|---|---|---|
| 00–06 | 2020-12-18 → 2022-12-22 | seven revisions |
| 07 | 2023-06-24 | → **expired 2023-12-29** |
| 08 | 2024-10-07 | **revival after ~9.3 months expired** |
| 09 | 2025-03-17 | → **expired 2025-09-18** |
| 10 | 2025-09-27 | **revival after 9 days**; HTTPDIR review requested 2025-10-08, returned "Not Ready" 2026-01-16 → **expired 2026-03-31** |
| 11 | 2026-05-23 | **revival after ~7.5 weeks**. Expires 2026-11-24 |

### Revival mechanics — the finding that reshapes the contingency

`[FACT]` RFC 2026 §2.2, raw-text verified: *"At any time, an Internet-Draft
may be replaced by a more recent version of the same specification,
restarting the six-month timeout period."*

`[FACT]` **This draft has expired and revived three times.** `[INFERENCE]`
Expiry is procedural noise for this document, not a death signal. The
maximum observed dormancy between revisions is **~15.5 months**
(draft-07 → draft-08). Any abandonment threshold shorter than that would
have false-fired in mid-2024 — and the draft came back.

### Is there any dated path to RFC? `[FACT]` **No.**

- **Datatracker state:** `I-D Exists`, `WG Document`. Responsible AD:
  `(None)`. Shepherd: `null`. IESG state description: *"The IESG has not
  started processing this draft, or has stopped processing it without
  publication."*
- **Charter milestones:** the httpapi charter has **no milestone for this
  draft**.
- **No WGLC.** By contrast `rest-api-mediatypes-09` is `In WG Last Call`,
  so the WG does run them. This is the only active httpapi WG draft that
  has not reached Last Call.
- **Mailing list, 2026:** only I-D Action notices for -10 and -11, the
  Pardue review posting, and bot digests. No Last Call, WGLC, or IESG
  subject lines.
- **IETF 124 (2025-11-04)** — the last session with materials — carried a
  3-page deck "RateLimiting Update" by Darrel Miller covering **only**
  issue #149 (reset wording) and issue #158 (String vs Token). No
  advancement slide, no timeline.
- `[INFERENCE]` **The WG appears not to have met in 2026** — sessions for
  IETF 125 (2026-03-14) and IETF 126 (2026-07-18) show `materials: []`
  against a control of `materials: 7` for IETF 124. Caveat: slots were
  requested (`on_agenda: true`) but never scheduled.
- **Repo activity:** last commit to `main` 2026-04-18 — ~3.7 months of
  silence.

`[FACT]` **The WG itself is productive** — 6 published RFCs, 2 in the RFC
Editor queue in 2026. `[INFERENCE]` The draft is not stalled because the WG
is moribund; it is specifically not advancing.

`[FACT]` Darrel Miller is **both an httpapi WG co-chair and the draft's
uploading editor** (revisions 08–10).

### The HTTPDIR review response falsifies `baseline-03b`'s key inference

`[FACT]` Pardue's largest concern was **not** editorial: *"Parameter
Extensibility — the largest concern that does significant work"*,
recommending *"an IANA registry and more policy about use of parameters."*

`[FACT]` The editors' response, PR #166, adds 5 IANA registries **and
renames the wire parameters** `r`→`a`, `t`→`w`.

`[INFERENCE]` `baseline-03b` reasoned that an editorial objection implies
"what it specifies is probably stable." **That inference is falsified by
the editors' own response.** The correction inverts the risk model: the
danger to `OP-010` is not a dead citation, it is a **live citation to a
moving target**.

---

## Q3 — Implementations by draft revision

Classification: **A** = proprietary `X-RateLimit-*`; **B** = Gen A IETF
triple; **B7** = draft-07 dictionary; **C** = Gen C item-list (draft-08–11);
**D** = nothing or `Retry-After` only.

### Libraries and frameworks

| Package | Version | Option | Default behavior | Class |
|---|---|---|---|---|
| **express-rate-limit** | 8.6.2 | `standardHeaders: false \| true \| 'draft-6' \| 'draft-7' \| 'draft-8'` | **A** (`standardHeaders: false`, `legacyHeaders: true`) | A default; B/B7/**C** opt-in |
| `@fastify/rate-limit` | 11.2.0 | `enableDraftSpec` (default `false`) | A | A; draft-**00-era** opt-in |
| `@nestjs/throttler` | 6.5.0 | `setHeaders` (on/off only) | A | A |
| Laravel `ThrottleRequests` | 13.24.0 | none (unconditional) | A | A |
| Flask-Limiter | 4.1.1 | `RATELIMIT_HEADERS_ENABLED` (default `False`) | none | A |
| Spring Cloud Gateway `RedisRateLimiter` | 5.0.2 | `include-headers` | A | A |
| `bucket4j-spring-boot-starter` | 0.14.0 | `hide-http-response-headers` | A | A |
| `didip/tollbooth` | v7.0.2 / v8 | none | A + B | A + B |
| `ulule/limiter`, `sethvargo/go-limiter` | v3.11.2 / v1.2.0 | none | A | A |
| `tower-governor`, `actix-governor` | 0.8.0 / 0.10.0 | `use_headers()` | A | A |
| Django REST Framework | 3.18.0 | none | `Retry-After` on 429 only | **D** |
| **ASP.NET Core** `Microsoft.AspNetCore.RateLimiting` | v7.0.20 → main | **none exists** | **nothing at all** | **D** |
| **trillium-ratelimit** (Rust) | **0.0.2** | none (always on) | **C** | **C** |

### Gateways, proxies, CDNs

| Product | Emits | Class |
|---|---|---|
| **Kong** rate-limiting | `X-RateLimit-{Limit,Remaining}-{Second…Year}` **plus** bare `RateLimit-Limit`/`-Remaining`/`-Reset`; no `RateLimit-Policy`. Provenance PR #5335 (2019-12-19) cites `draft-polli-ratelimit-headers-01` | A + B |
| **Envoy** | `enable_x_ratelimit_headers` accepts only `OFF` and `DRAFT_VERSION_03` — the **2020 pre-WG individual draft**. Off by default | A |
| **Traefik** | `Retry-After` + proprietary `X-Retry-In` | D |
| **Atlassian** Jira Cloud | `X-RateLimit-*` enforcing **plus `Beta-RateLimit`/`Beta-RateLimit-Policy`** in draft-11 syntax, informational only | A + **C (prefixed)** |
| Tyk, Apache APISIX | `X-RateLimit-*` | A |
| NGINX, HAProxy, Apigee, AWS API Gateway, Azure APIM, Cloudflare, Zuplo | nothing, or `Retry-After` only | D |

### The three sharpest measurements

`[FACT]` **1. Exactly two Generation C emitters exist anywhere.**
`express-rate-limit` at `standardHeaders: 'draft-8'` (58.2M npm
downloads/week — but **opt-in and off by default**), and
`trillium-ratelimit` v**0.0.2**, published 2026-07-15, **2,178 all-time
downloads, 0 GitHub stars**.

`[FACT]` **2. `standardHeaders: true` resolves to Generation A.** Verbatim
from `source/rate-limit.ts` lines 197–201 — typos and the unbalanced
backtick are preserved from the upstream source comment:

```ts
// The default value for the `standardHeaders` option is `false`. If set to
// `true`, it resolve to `draft-6`. `draft-7` and draft-8` (recommended) are
// used only if explicitly set.
let standardHeaders = notUndefinedOptions.standardHeaders ?? false
if (standardHeaders === true) standardHeaders = 'draft-6'
```

`[INFERENCE]` The obvious way to "turn on the standard headers" in the
most-used Node rate limiter yields the **2022 triple**, five revisions
stale — while the library's own comment marks `draft-8` "(recommended)."

`[FACT]` **3. Client tooling for Generation C does not exist even where the
emitter does.** `ratelimit-header-parser` 0.1.0, written **by the
express-rate-limit team**, advertises support *"including the combined form
from draft 7"* and **cannot parse the draft-8 headers its own sibling
library recommends emitting.**

`[FACT]` **A semantic incompatibility, not merely a naming one.**
Flask-Limiter and Laravel emit `*-Reset` as an **absolute Unix timestamp**
(Flask-Limiter's own test asserts `str(int(time.time() + 61))`; Laravel
calls `$this->availableAt($retryAfter)`). **Every draft revision uses
delta-seconds.** Flask-Limiter's `RATELIMIT_HEADER_RESET` lets an operator
rename `X-RateLimit-Reset` → `RateLimit-Reset` while keeping timestamp
semantics — producing a draft-noncompliant value under a draft-reserved
field name.

`[INFERENCE]` **The fragmentation the "standard" label hides is worse than
an old/new split.** Five classes; installed base concentrated in A; the
only high-volume Gen C path off by default; one vendor shipping Gen C
behind a `Beta-` prefix; Envoy pinned to a pre-WG individual draft from
2020.

---

## Q4 — Alternatives

### `X-RateLimit-*`: the same field name, three meanings

`[FACT]` `[COMPARATIVE]`

| Vendor | Header | Meaning |
|---|---|---|
| Discord | `X-RateLimit-Reset` | Unix epoch seconds (float-capable) |
| Reddit | `X-Ratelimit-Reset` | Delta seconds — *"Approximate number of seconds to end of period"* |
| Atlassian | `X-RateLimit-Reset` | **ISO 8601 timestamp** (`2026-01-01T01:01:01Z`) |

`[INFERENCE]` HTTP field names are case-insensitive per RFC 9110, so
**Discord's epoch and Reddit's delta are literally the same field with
opposite semantics**. Across eight vendors there are **six distinct
spellings** and six wire encodings.

### `Retry-After` alone: what is actually normative

`[FACT]` **429 is defined in RFC 6585, not RFC 9110.** Grepping the
complete 10,785-line RFC 9110 text for `429` returns **zero occurrences**.
IANA maps `429,Too Many Requests,[RFC6585]`. RFC 6585 is Proposed Standard
and **not obsoleted**.

`[FACT]` **RFC 6585 §4 is the only normative link between `Retry-After`
and 429, and it is `MAY`:**

> The response representations SHOULD include details explaining the
> condition, and MAY include a Retry-After header indicating how long
> to wait before making a new request.

`[FACT]` RFC 9110 §10.2.3 defines `Retry-After` for **503 and 3xx only**
and never mentions 429. `delay-seconds = 1*DIGIT` — a non-negative
**integer** (note Shopify prints `Retry-After: 2.0`, non-conformant).

`[FACT]` `Retry-After` is registered `permanent` in the IANA HTTP Field
Name Registry. `RateLimit`, `RateLimit-Policy`, and every `X-RateLimit-*`
variant are **absent** — verified two ways, and the absence is not an
export artifact (23 provisional entries are visible).

### Is `Retry-After` + documented proprietary detail the honest position? `[INFERENCE]` **Yes — and the draft agrees.**

`[FACT]` draft-11 §7: *"If a response contains both the RateLimit and
Retry-After fields, the Retry-After field MUST take precedence and the
effective window MAY be ignored."*

`[FACT]` `[COMPARATIVE]` Three of the largest API-guideline authorities sit
at exactly this tier: Microsoft Azure REST Guidelines (108,399 bytes,
**zero** occurrences of `ratelimit` or `429`; *"the `Retry-After` should be
what developers rely on to back off"*), Microsoft Graph (*"MUST return a
429 … and a 503 …"*, no headers), Google AIP (**zero** files matching
`ratelimit` or `throttl`).

### Multi-dimensional headers, and whether `qu` covers them

`[FACT]` OpenAI uses `x-ratelimit-<measure>-<dimension>` with Go duration
resets; Anthropic uses `anthropic-ratelimit-<dimension>-<measure>` with
RFC 3339 resets. **Segment order is reversed**; no single positional parse
covers both.

`[FACT]` The draft's `qu` parameter covers this case **poorly**: the word
"token" appears **zero times** in the 35-page draft; `qu` exists only on
`RateLimit-Policy`, not on the `RateLimit` field carrying the live
remaining count; the Quota Units registry is Specification Required **and
does not exist yet**; exactly one `qu=` example appears in the whole
document. `[INFERENCE]` `qu` is a design sketch for the token case, not a
solution to it.

### Everything else

`[FACT]` **No published RFC of any status defines HTTP rate-limit or quota
response fields** — full RFC index scan returns only storage quotas and
TCP. W3C: one **Retired** browser storage API. OpenAPI: `X-Rate-Limit-*`
appears in OAS 3.1.1/3.2.0 **as illustrative examples with no assigned
semantics**, using the extra-hyphen spelling the draft's own appendix names
as a defect. GraphQL: cost lives in the response body; no standards-body
work item.

`[FACT]` **A trap worth naming:** `draft-ietf-ccwg-ratelimited-increase-08`
(2026-08-05) is in `IESG Evaluation::AD Followup` — far more momentum than
the HTTPAPI draft. It is **TCP congestion control**, not HTTP quota
signalling. A search for "IETF rate limit standardization" lands on it and
overestimates maturity.

---

## Q5 — Recommendation for OP-010

**Rule shape (b), with the contingency re-keyed.** Confidence:
**moderate-high** for the mandate clause, **moderate** for the advisory
clause.

### Proposed wording

> **OP-010** — **MUST** · Apply rate limits, and communicate exhaustion
> using the published HTTP mechanism: return `429` with `Retry-After`, per
> **OP-011**.
>
> Services **SHOULD** additionally advertise quota state using the
> `RateLimit` and `RateLimit-Policy` fields **in the syntax of
> `draft-ietf-httpapi-ratelimit-headers-11`**.
>
> `[POLICY]` These two fields are an **unpublished IETF Internet-Draft**,
> not a published standard. They MUST NOT be described as
> standards-compliant in API documentation, and the pinned revision number
> MUST be cited wherever they are referenced.
>
> A service that does not emit them **MUST** document its proprietary quota
> headers explicitly, including — for any reset field — whether the value
> is an **absolute epoch timestamp or a delta in seconds**.

Drafting notes: **(1)** `OP-010` must *cross-reference* `OP-011` rather
than restate it (`OP-011` already requires 429/503 + `Retry-After`).
**(2)** The decision record should note that `OP-011`'s MUST is a
deliberate tightening of RFC 6585 §4's MAY — legitimate house policy over a
**published** standard, categorically different from claiming compliance
with a draft. **(3)** The epoch-vs-delta disclosure requirement earns its
place: it is the single documented ambiguity that breaks real clients.

### The contingency, re-keyed

`[INFERENCE]` **Expiry is the wrong trigger.** The draft has expired and
revived three times; RFC 2026 §2.2 restarts the clock on any new revision.
A 2026-11-24 fallback would very likely fire on a document that returns in
weeks — as it did in 2026, 2025, and 2024.

> **Conflict surfaced.** `baseline-03e` concluded the 2026-11-24 trigger
> "remains the right instrument." **The revision history governs and
> overrides it** — that leaf did not have the three expiry-revival cycles
> or the RFC 2026 restart clause in view.

Three triggers replace the single date:

| Trigger | Condition | Action |
|---|---|---|
| **Upgrade** (SHOULD → MUST) | `RateLimit` appears in the IANA HTTP Field Name Registry with status `permanent`. Check: `curl -s https://www.iana.org/assignments/http-fields/field-names.csv \| grep -i ratelimit` | Raise to MUST; drop the `[POLICY]` label |
| **Re-pin** | Any new revision changes the wire syntax — watch **PR #166** (`r`→`a`, `t`→`w`) and **issue #158** (String→Token) | Update the pinned revision; drop to MAY if the format churns again before publication |
| **Withdraw** (drop the SHOULD; keep 429 + `Retry-After`) | **Sustained abandonment**: *either* (a) **18 months** with no new revision **and** no WGLC, *or* (b) the draft leaves the httpapi WG's active document list unrevived, *or* (c) the WG concludes | Fall back to the documented-proprietary clause the rule already carries |

The 18-month threshold is calibrated above the **15.5-month** maximum
observed dormancy, so it cannot false-fire on the pattern this document has
actually exhibited. **Review cadence: semi-annual, next 2027-02-09.**
2026-11-24 remains worth a glance, but **only to check whether a draft-12
appeared** — expiry alone must not trigger the fallback.

### Why not the stronger or weaker rule

`[INFERENCE]` The SHOULD is not timidity. Generation C has exactly two
emitters, one opt-in-and-off-by-default and one a 0-star crate; the
reference client parser cannot read it; and the first production deployment
(Atlassian) ships it **behind a `Beta-` prefix and explicitly
non-enforcing** — *"At enforcement, these headers will drop the `Beta-`
prefix… Format and behavior are identical, only the prefix differs"*
(independently verified against the live page). A prefixed field is not the
draft's field. Atlassian is real evidence that the format is implementable
and that a major vendor is betting on it — which is why the rule points at
the fields at all — but it does not make them interoperable today.

---

## Method limitations, stated explicitly

- A web-fetch summarizer **fabricated a normative RFC 9110 sentence**
  during this run; all RFC/draft/registry quotes were re-verified from raw
  `.txt`.
- The session's search budget was exhausted early; discovery ran through
  direct URL fetches and registry APIs. "What is specified" is well
  covered; "what is being discussed" (mailing-list threads beyond subject
  lines) is under-covered.
- GitHub's code-search API was unusable during the run; an obscure
  Generation C emitter in a less-probed language cannot be ruled out.
- Cloudflare and AWS API Gateway findings are absence-of-documentation, not
  documented absence.
- The "httpapi did not meet in 2026" claim is an `[INFERENCE]` from
  `materials: []` against a control, not a stated cancellation.
- Reddit's current numeric limits are unverified (help-center pages 403);
  the header semantics rest on Reddit's own archived wiki.
