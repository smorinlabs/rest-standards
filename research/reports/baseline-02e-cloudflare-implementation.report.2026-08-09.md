# Baseline 02e — Cloudflare's RFC 9457 Implementation, Verified Live

*Narrow leaf under `baseline-02`. Extends `baseline-02d` finding 1.2
(Cloudflare adoption). Run 2026-08-09; all primary sources captured and live
behavior independently verified against Cloudflare's production edge on
2026-08-09. Raw HTTP captures were retained in the session scratchpad
(ephemeral, machine-local); every load-bearing capture is quoted verbatim
below.*

**Evidence labels:** `[FACT]` = primary source (Cloudflare blog/docs/changelog)
or live HTTP capture. `[COMPARATIVE]` = comparison against RFC 9457 text.
`[INFERENCE]` = reading, not stated by Cloudflare. `[ABSENT]` = explicitly
checked, not found.

## 0. Timeline (three ship events, not one)

| Date | Event | Source |
|---|---|---|
| 2026-02-26 | Markdown responses for 1xxx errors (`Accept: text/markdown`). Frontmatter field was `http_status`. | `[FACT]` https://developers.cloudflare.com/changelog/post/2026-02-26-markdown-responses-for-1xxx-errors/ |
| 2026-03-11 | JSON + RFC 9457 for 1xxx errors. Blog post "Slashing agent token costs by 98%…". Breaking rename `http_status` → `status`. | `[FACT]` https://blog.cloudflare.com/rfc-9457-agent-error-pages/ · https://developers.cloudflare.com/changelog/post/2026-03-11-json-rfc9457-responses-for-1xxx-errors/ |
| 2026-04-27 | Extended to Cloudflare-generated 5xx (500, 502, 504, 520–526). | `[FACT]` https://developers.cloudflare.com/changelog/post/2026-04-27-structured-responses-for-5xx-errors/ |
| 2026-05-05 | Consolidated reference page last updated. | `[FACT]` https://developers.cloudflare.com/fundamentals/reference/error-responses/ |

`[FACT]` The 2026-03-11 blog stated the 1xxx-only scope and pre-announced the
extension: *"This covers all 1xxx-class errors today. The same contract will
extend to Cloudflare-generated 4xx and 5xx errors next."* The 4xx half of that
promise has **not** shipped as a separate changelog entry as of 2026-08-09
`[ABSENT]` — though many 1xxx codes already return 4xx statuses (1015→429,
1020→403, 1026→451).

---

## Q1 — Trigger / negotiation

### It is `Accept`-header content negotiation only. No user-agent sniffing.

`[FACT]` Docs: *"Cloudflare selects the response format based on the client's
`Accept` header, following standard HTTP content negotiation. When multiple
formats are acceptable, quality factors (`q` values) determine precedence. At
the same quality value, the first-listed type wins."*
(https://developers.cloudflare.com/fundamentals/reference/error-responses/)

`[ABSENT]` The words "user-agent"/"user agent" appear in the docs page **only**
inside `curl --user-agent "TestAgent/1.0"` example flags. No bot/agent UA
detection is described anywhere in the blog, the three changelogs, or the
reference page. Caveat: every live request in this run carried
`User-Agent: TestAgent/1.0`, so UA-independence was not empirically proven.

### Negotiation table (verbatim from docs, 2026-05-05)

```
| Accept header sent                    | Response format                                |
| application/json                      | JSON (application/json; charset=utf-8)         |
| application/problem+json              | JSON (application/problem+json; charset=utf-8) |
| application/json, text/markdown;q=0.9 | JSON (higher quality factor)                   |
| text/markdown                         | Markdown (text/markdown; charset=utf-8)        |
| text/markdown, application/json       | Markdown (equal quality, first-listed wins)    |
| text/*                                | Markdown                                       |
| text/html                             | HTML                                           |
| */*                                   | HTML                                           |
| Not set                               | HTML                                           |
```

`[FACT]` Blog adds the decision rule and the browser guarantee: *"The behavior
is deterministic — the first explicit structured type wins"* and
*"Wildcard-only requests (`*/*`) do not signal a structured preference;
clients must explicitly request Markdown or JSON."* and *"Browsers keep
receiving HTML unless clients explicitly ask for Markdown or JSON."*

### Exact Content-Type returned — live-verified

`[FACT]` Captures against `https://blog.cloudflare.com/cdn-cgi/error/1015` on
2026-08-09 16:21 UTC:

| Request `Accept` | Response status | Response `Content-Type` | `Retry-After` |
|---|---|---|---|
| `application/problem+json` | `HTTP/2 429` | `application/problem+json; charset=utf-8` | `30` |
| `application/json` | `HTTP/2 429` | `application/json; charset=utf-8` | `30` |
| `text/markdown` | `HTTP/2 429` | `text/markdown; charset=utf-8` | `30` |

Full header block for the problem+json case:

```
HTTP/2 429
date: Sun, 09 Aug 2026 16:21:01 GMT
content-type: application/problem+json; charset=utf-8
content-length: 981
retry-after: 30
cache-control: private, max-age=0, no-store, no-cache, must-revalidate, post-check=0, pre-check=0
expires: Thu, 01 Jan 1970 00:00:01 GMT
referrer-policy: same-origin
x-frame-options: SAMEORIGIN
access-control-allow-origin: https://dash.cloudflare.com
server: cloudflare
cf-ray: a288176dfd41b701-SJC
```

`[FACT]` Charset is `utf-8`, lowercase, always present on all three structured
types. `[FACT]` `application/json` and `application/problem+json` return a
**byte-identical body** — only the `Content-Type` differs (`content-length:
981` in both captures). Changelog states this explicitly: *"Same body in both
cases."*

`[FACT, docs+blog; not independently verified]` The HTML rows (`*/*`,
`text/html`, no header) are documented in two primary sources but could not be
re-verified live — see the rate-limit finding below.

### The Markdown variant

`[FACT]` Negotiated by `Accept: text/markdown` or `Accept: text/*`. Media type
returned: `text/markdown; charset=utf-8`. It is **not** a `+markdown`
structured-suffix type and has no RFC 9457 relationship — it is a parallel
format carrying the same fields.

`[FACT]` Structure per docs: YAML frontmatter between `---` delimiters, then
three prose sections. Docs: *"The frontmatter omits the RFC 9457 standard
members (`type`, `title`, `instance`) and the `footer` field since these are
either redundant with the prose or not applicable to the Markdown format."*

Live-captured Markdown body (808 bytes):

```markdown
---
error_code: 1015
error_name: rate_limited
error_category: rate_limit
status: 429
ray_id: a28817d9fbb8c65e
timestamp: 2026-08-09T16:21:19Z
zone: blog.cloudflare.com
cloudflare_error: true
retryable: true
retry_after: 30
owner_action_required: false
---

# Error 1015: You are being rate limited

## What Happened

You are being rate-limited by the website owner's configuration.

## What You Should Do

**Wait and retry.** This block is transient. Wait at least 30 seconds, then retry with exponential backoff.

Recommended approach:
1. Wait 30 seconds before your next request
2. If rate-limited again, double the wait time (60s, 120s, etc.)
3. If rate-limiting persists after 5 retries, stop and reassess your request pattern

---

This error was generated by Cloudflare on behalf of the website owner.
```

### Finding: a real enforced error did NOT honor the contract

`[FACT]` After ~25 requests, Cloudflare's edge rate-limited the probing IP on
the `/cdn-cgi/error/` path. The **real, enforced** 1015 came back as:

```
HTTP/2 429
content-type: text/plain; charset=UTF-8
content-length: 17

error code: 1015
```

…despite `Accept: application/problem+json`. This persisted across
`blog.cloudflare.com`, `developers.cloudflare.com`, `www.cloudflare.com`, and
`community.cloudflare.com` (IP-scoped limiter), and across all 16 `Accept`
variants tried.

`[INFERENCE]` Two readings, indistinguishable from outside: (i) a dedicated
abuse-protection path guarding `/cdn-cgi/error/` itself, sitting *in front of*
the structured-error layer; or (ii) some real enforcement paths generally emit
the legacy plain-text body while the documented `/cdn-cgi/error/<code>`
endpoint is a synthetic preview. `[ABSENT]` The docs present
`/cdn-cgi/error/<code>` under a heading "Test structured error responses" but
never state whether it is synthetic or whether all real enforcement paths
route through the same renderer.

This is direct evidence for `AC-003` draft amendment (a) — see the
implications section.

---

## Q2 — Payload shape

### All five RFC 9457 members are populated. None are omitted.

`[FACT]` Docs: *"JSON responses follow RFC 9457 (Problem Details for HTTP
APIs). Any HTTP client that understands Problem Details can parse the five
standard members (`type`, `title`, `status`, `detail`, `instance`) without
Cloudflare-specific code."*

```
| Field    | Type    | Description                                                               |
| type     | string  | URI pointing to Cloudflare documentation for this error code.             |
| title    | string  | Short summary, for example, "Error 522: Connection timed out".            |
| status   | integer | HTTP status code of the response.                                         |
| detail   | string  | Plain-text explanation of what went wrong and which party is responsible. |
| instance | string  | Ray ID identifying this specific error occurrence.                        |
```

### Extension members (12) — verbatim docs table

```
| error_code             | integer         | Cloudflare error code (for example, 522, 1015).
| error_name             | string          | Machine-readable name in snake_case (for example, connection_timeout, rate_limited). Stable — suitable for programmatic matching.
| error_category         | string          | Fault classification. Refer to Error categories. Stable — suitable for programmatic matching.
| ray_id                 | string          | Same value as instance. Included for compatibility with existing Cloudflare tooling.
| timestamp              | string          | ISO 8601 timestamp of when the error was generated.
| zone                   | string          | The requested hostname.
| cloudflare_error       | boolean         | Always true. Confirms this error was generated by Cloudflare, not the origin.
| retryable              | boolean         | Whether the error is transient and the request can be retried.
| retry_after            | integer or null | Seconds to wait before retrying. Present only when retryable is true. Matches the Retry-After HTTP header value.
| owner_action_required  | boolean         | Whether the site operator needs to take action to resolve the error.
| what_you_should_do     | string          | Actionable guidance for the client: what to do next, whether to retry, and who can fix the problem.
| footer                 | string          | Attribution line.
```

`[FACT]` Naming convention is `snake_case` throughout, for both standard and
extension members. No namespacing, no prefix (e.g. no `cf_`), no nesting — the
object is flat. Blog: *"JSON responses carry the same fields as a flat
object."*

`[FACT, live]` `retry_after` is **omitted entirely** when `retryable: false`
(verified on 1020, 1009, 1101, 1016, 1026, 526, 1044) — the docs' "integer or
null" phrasing notwithstanding, an explicit `null` was never observed.

### Verbatim live payloads

**1015 rate limit** (live 2026-08-09, pretty-printed; identical body under
both JSON media types):

```json
{
  "type": "https://developers.cloudflare.com/support/troubleshooting/http-status-codes/cloudflare-1xxx-errors/error-1015/",
  "title": "Error 1015: You are being rate limited",
  "status": 429,
  "detail": "You are being rate-limited by the website owner's configuration.",
  "instance": "a28817ceee58faca",
  "error_code": 1015,
  "error_name": "rate_limited",
  "error_category": "rate_limit",
  "ray_id": "a28817ceee58faca",
  "timestamp": "2026-08-09T16:21:17Z",
  "zone": "blog.cloudflare.com",
  "cloudflare_error": true,
  "retryable": true,
  "retry_after": 30,
  "what_you_should_do": "**Wait and retry.** This block is transient. Wait at least 30 seconds, then retry with exponential backoff.\n\nRecommended approach:\n1. Wait 30 seconds before your next request\n2. If rate-limited again, double the wait time (60s, 120s, etc.)\n3. If rate-limiting persists after 5 retries, stop and reassess your request pattern",
  "owner_action_required": false,
  "footer": "This error was generated by Cloudflare on behalf of the website owner."
}
```

**522 origin timeout** — docs' published example
(https://developers.cloudflare.com/fundamentals/reference/error-responses/),
which the live capture reproduced field-for-field with only
`instance`/`ray_id`/`timestamp`/`zone` differing:

```json
{
	"type": "https://developers.cloudflare.com/support/troubleshooting/http-status-codes/cloudflare-5xx-errors/error-522/",
	"title": "Error 522: Connection timed out",
	"status": 522,
	"detail": "Cloudflare could not establish a TCP connection to the origin server. The TCP handshake timed out, which may indicate the origin is overloaded, firewalling Cloudflare, or unreachable at the network level.",
	"instance": "9f140b785e57c458",
	"error_code": 522,
	"error_name": "connection_timeout",
	"error_category": "origin",
	"ray_id": "9f140b785e57c458",
	"timestamp": "2026-04-24T09:22:40Z",
	"zone": "example.com",
	"cloudflare_error": true,
	"retryable": true,
	"retry_after": 120,
	"owner_action_required": true,
	"what_you_should_do": "**Wait and retry.** Back off for at least 120 seconds. If the error persists, the website operator should verify firewall rules and ensure the origin accepts connections from Cloudflare IP ranges.",
	"footer": "This error was generated by Cloudflare on behalf of the website owner."
}
```

**1020 WAF block, non-retryable** (live) — note absent `retry_after`:

```json
{"type":"https://developers.cloudflare.com/support/troubleshooting/http-status-codes/cloudflare-1xxx-errors/error-1020/","title":"Error 1020: Access denied","status":403,"detail":"The request was blocked by a Cloudflare firewall rule configured by the site owner.","instance":"a28818bd8e015024","error_code":1020,"error_name":"firewall_rule_blocked","error_category":"access_denied","ray_id":"a28818bd8e015024","timestamp":"2026-08-09T16:21:55Z","zone":"blog.cloudflare.com","cloudflare_error":true,"retryable":false,"owner_action_required":true,"what_you_should_do":"**Do not retry.** This block is intentional. Contact the site owner if you believe this is an error.","footer":"This error was generated by Cloudflare on behalf of the website owner."}
```

### `Retry-After` header (belt-and-braces with the body field)

`[FACT]` Docs: *"Retryable error codes include a standard `Retry-After` HTTP
response header. The header value in seconds matches the `retry_after` field
in the response body."* Live-verified on 1015 (30), 522 (120), 500 (30), 1200
(60); absent on 526, 1020, 1009, 1101, 1016, 1026, 1044.

Published values — 5xx: 500→30, 502→60, 504→120, 520→60, 521→120, 522→120,
523→120, 524→120, 525/526→not retryable. 1xxx (only six are retryable):
1004→120, 1015→30, 1033→120, 1038→60, 1200→60, 1205→5. `[FACT]` *"If a WAF
rate limiting rule has already set a dynamic `Retry-After` value on the
response, that value takes precedence over the default."*

---

## Q3 — The `type` member (load-bearing)

### Answer in one line: resolvable https documentation URLs, per-error-code where a doc page exists, silently degrading to a **category index URL** where it does not.

**(a) `[FACT]` Values are resolvable https URLs.** Never `about:blank`
(grep-verified absent from blog and docs; never observed in 11 live
payloads). Never URNs. Never relative.

**(b) `[FACT, live-verified]` Per-code deep links resolve HTTP 200:**

| error_code | `type` value | HTTP status of that URL |
|---|---|---|
| 1015 | `…/cloudflare-1xxx-errors/error-1015/` | 200 |
| 1020 | `…/cloudflare-1xxx-errors/error-1020/` | 200 |
| 1009 | `…/cloudflare-1xxx-errors/error-1009/` | 200 |
| 1016 | `…/cloudflare-1xxx-errors/error-1016/` | 200 |
| 1101 | `…/cloudflare-1xxx-errors/error-1101/` | 200 |
| 1200 | `…/cloudflare-1xxx-errors/error-1200/` | 200 |
| 522 | `…/cloudflare-5xx-errors/error-522/` | 200 |
| 526 | `…/cloudflare-5xx-errors/error-526/` | 200 |

(Prefix elided is
`https://developers.cloudflare.com/support/troubleshooting/http-status-codes/`.)

**(c) `[FACT, live-verified — the important one]` For error codes with no
per-code doc page, `type` falls back to the category index, losing per-error
identity:**

| error_code | `type` emitted | would-be per-code URL | index URL |
|---|---|---|---|
| 1026 | `…/cloudflare-1xxx-errors/` | `…/error-1026/` → **404** | 200 |
| 1044 | `…/cloudflare-1xxx-errors/` | `…/error-1044/` → **404** | 200 |
| 500 | `…/cloudflare-5xx-errors/` | (not probed) | 200 |

So `type` for 1026 and `type` for 1044 are **the same string**. `type` is
therefore **not a unique discriminator** across Cloudflare's error space. An
RFC 9457 client that dispatches solely on `type` — the exact pattern RFC 9457
§3.1.1 designates `type` for — cannot tell a legal-takedown 451 from a
FIPS-cipher 403.

`[ABSENT]` Neither the docs table (*"URI pointing to Cloudflare documentation
for this error code"*), the changelog (*"URI pointing to Cloudflare
documentation for the specific error code"*), nor the blog (*"A URI pointing
to Cloudflare's documentation for the specific error code"*) mentions this
fallback. All three assert per-code granularity that the implementation does
not always deliver.

**(d) `[FACT]` The URLs are documented but never declared stable.** The docs
annotate exactly two fields with a stability guarantee — `error_name`:
*"Stable — suitable for programmatic matching"* and `error_category`:
*"Stable — suitable for programmatic matching."* No such annotation appears on
`type`, or on any other field. `[INFERENCE]` Cloudflare positions the
extension members, not `type`, as the durable programmatic identity. The blog
corroborates this rhetorically: every worked example, the ten-category routing
table, and the Python reference implementation branch on `error_code` /
`error_category` / `retryable` / `owner_action_required`. `type` is never used
for dispatch in any Cloudflare-authored example.

**(e) `[FACT]` These are documentation-site URLs on a product docs host,
subject to that site's information architecture.** The URLs live under
`developers.cloudflare.com/support/troubleshooting/http-status-codes/` — a
path segment that encodes a site-navigation taxonomy ("support",
"troubleshooting"), not an error-identity namespace. `[INFERENCE]` Coupling
`type` identity to a docs-site URL structure means a docs reorganization is a
wire-protocol change; the 1026/1044 index fallback is already evidence that
page existence, not error identity, drives the value.

**(f) `[ABSENT]` The IANA problem-types registry is never mentioned.** Grep
for `iana` and `registr*` across the full blog text and the full docs page:
zero hits (the single `registr` hit is Cloudflare's "Registrar" product in
site navigation). Cloudflare registered nothing and referenced nothing. Same
for all three changelogs.

---

## Q4 — Scope

### Covered

`[FACT]` Docs: *"This machine-readable response covers all 1xxx error codes
(which return HTTP 4xx or 5xx status codes depending on the error) and
Cloudflare-generated 5xx errors (500, 502, 504, 520-526)."*

`[FACT]` Blog: *"Cloudflare now returns RFC 9457-compliant structured
responses for all 1xxx-class error paths."*

Ten `error_category` values for 1xxx, with published code lists:

```
| access_denied | IP blocks, country blocks, firewall rules        | 1005, 1006, 1007, 1008, 1010, 1012, 1106-1109 |
| rate_limit    | Rate limiting                                    | 1015, 1025, 1027, 1200                        |
| dns           | DNS resolution errors                            | 1001, 1016                                    |
| config        | Zone or origin configuration errors              | 1004, 1014, 1033, 1043, 1047, 1049            |
| tls           | Client TLS errors (version, cipher, certificate) | 1017, 1028, 1029, 1044                        |
| legal         | Legal restrictions (DMCA, country blocks)        | 1026, 1039                                    |
| worker        | Worker script errors                             | 1042, 1100, 1101, 1102, 1103, 1104, 1105      |
| rewrite       | URL rewrite rule errors                          | 1036, 1037                                    |
| snippet       | Snippet configuration errors                     | 1201, 1202, 1203, 1204, 1205, 1206            |
| unsupported   | Unsupported features or protocols                | 1045                                          |
```

Three for 5xx: `origin` (502, 504, 520–524, retryable), `cloudflare` (500,
retryable, 30s), `ssl` (525, 526, **not** retryable).

### Not covered

`[FACT]` **Origin-generated errors pass through untouched.** Docs:
*"Responses for 5xx errors generated by the origin server are passed through
by Cloudflare to the client and are not affected."* Changelog:
*"Origin-generated 5xx responses that Cloudflare passes through are not
affected."* `[INFERENCE]` Cloudflare only guarantees the shape for errors
**it** synthesizes — it does not rewrite or wrap the customer's own error
bodies.

`[FACT]` Unknown codes are not synthesized: `GET /cdn-cgi/error/9999` returned
`HTTP/2 404`, `content-type: text/html; charset=UTF-8`, an nginx-style HTML
404 body — no problem details.

`[FACT]` Cloudflare-generated **4xx** errors as a class (distinct from 1xxx
codes that happen to carry 4xx statuses) were pre-announced 2026-03-11 as
"next" but have no shipping changelog as of 2026-08-09.

`[ABSENT]` Nothing in any source describes behavior for WAF managed-challenge
/ interstitial pages, `/cdn-cgi/challenge-platform`, or Turnstile — the docs
defer to *"the Custom Errors documentation"* for *"WAF custom block responses,
and security challenge pages."*

### On by default, network-wide, all plans, zero configuration — with an override hierarchy

`[FACT]` Blog: *"This is live across the Cloudflare network, automatically.
Site owners do not need to configure anything."* and *"This is network-wide
and additive."*

`[FACT]` Docs: *"Structured error responses are available on all plans,
including the Free plan. Custom Error Rules for overriding these responses
require a Cloudflare paid plan."* Changelog (5xx): *"Available now for all
zones on all plans."*

**But customers can silently break it.** `[FACT]` Published priority order:

```
1. Custom Error Rules — If a rule matches the error and request conditions, the rule's content is served.
2. Error Pages — If an Error Page is configured for the error type and no Custom Error Rule matched,
   the Error Page is served as HTML regardless of the Accept header.
3. Structured error responses — If no Custom Error Rule matched and no Error Page is configured,
   Cloudflare serves its default response in the format the client requested (JSON, Markdown, or HTML).
```

`[FACT]` The consequence is documented bluntly: with a dashboard Error Page
configured, `Accept: application/json` gets *"Your custom HTML error page"* —
*"Error Pages do not perform content negotiation."* Cloudflare's suggested
remedy is to **remove** the Error Page, or (paid plans only) write a Custom
Error Rule matching on the header, e.g.:

```
(http.response.code eq 522) and (any(http.request.headers["accept"][*] contains "application/json"))
```

`[INFERENCE]` So the "network-wide default machine contract" is a default that
any zone owner can override into HTML, and a Custom Error Rule serving JSON
may serve a **customer-defined** shape rather than the Cloudflare/RFC 9457
one — the docs say the rule can *"Serve a custom JSON response with your own
error format."* A client cannot assume a Cloudflare-shaped body just because
it got JSON from a Cloudflare-fronted host.

---

## Q5 — Stated rationale and measured results

### Rationale (blog, verbatim)

- *"But when these agents hit an error, they still receive the same HTML
  error pages we built for browsers: hundreds of lines of markup, CSS, and
  copy designed for human eyes. Those pages give agents clues, not
  instructions, and waste time and tokens."*
- *"To an agent, this is garbage. It cannot determine what error occurred, why
  it was blocked, or whether retrying will help. Even if it parses the HTML,
  the content describes the error but doesn't tell the agent — or the human,
  for that matter — what to do next."*
- On why per-site config was insufficient: *"Custom Error Rules can customize
  many Cloudflare errors, including some 1xxx cases. But they depend on
  per-site configuration, so they cannot serve as a universal agent contract
  across the web."*
- On what the standard buys them: *"That stability is not a Cloudflare
  invention. RFC 9457 — Problem Details for HTTP APIs defines a standard JSON
  shape for reporting errors over HTTP, so clients can parse error responses
  without knowing the specific API in advance."*
- On format vs. semantics: *"The core value is not format choice. It is
  semantic stability."*

### Measured results — the only numbers Cloudflare published

`[FACT]` The 55–64× figures are from this table (blog, error 1015,
`cl100k_base` tokenizer):

| Payload | Bytes | Tokens (cl100k_base) | Size vs HTML | Token vs HTML |
|---|---|---|---|---|
| HTML response | 46,645 | 14,252 | — | — |
| Markdown response | 798 | 221 | 58.5x less | 64.5x less |
| JSON response | 970 | 256 | 48.1x less | 55.7x less |

`[FACT]` Live 1015 bodies match the published sizes closely: Markdown
`content-length: 808` (vs 798 published), JSON `content-length: 981` (vs
970) — differences attributable to hostname length (`blog.cloudflare.com` vs
the docs' placeholder domain).

`[FACT]` The headline "98%" is the same measurement expressed as a percentage,
on **one error code** (1015), on **one metric pair** (bytes and tokens).
Cloudflare characterizes it as *"measured against a live 1015 ('rate-limit')
error response."*

### Follow-ups, adoption numbers, ecosystem reaction

`[ABSENT]` **No adoption numbers of any kind.** No follow-up post reporting
request volumes, percentage of traffic sending structured `Accept` headers, or
named framework integrations, in any Cloudflare property searched (blog,
changelogs, docs) as of 2026-08-09.

`[ABSENT]` **No named agent framework, SDK, or crawler is cited as consuming
it.** The blog's "Real-world use cases" section is prospective and
hypothetical throughout — describing what agents *could* do, not what any
named product does.

`[FACT]` What Cloudflare did instead was issue a call to action: *"To make
this work across the web, agent runtimes should default to explicit
structured Accept headers, not bare `*/*`. Use `Accept: text/markdown, */*`
for model-first workflows and `Accept: application/json, */*` for typed
control flow. If you maintain an agent framework, SDK, or browser automation
stack, ship this default and treat bare `*/*` as legacy fallback."*

Third-party coverage found was aggregation/commentary, not primary evidence of
adoption. **Caution:** a search-engine summary asserted that "Spring Framework
and ASP.NET Core include native support" (generic RFC 9457 library support,
unrelated to Cloudflare) and that "The Agent SDK now supports structured error
responses" (untraceable to any primary source). Both treated as unverified.

---

## Q6 — Deviations from and extensions to RFC 9457

### Deviations

**1. `detail` is static per error code, not occurrence-specific.**
`[COMPARATIVE]` RFC 9457 §3.1.4 specifies `detail` as *"a human-readable
explanation specific to this occurrence of the problem"*; Cloudflare's own
changelog repeats that wording verbatim. `[FACT]` But the live 1015 `detail` —
`"You are being rate-limited by the website owner's configuration."` — is
**byte-identical** to the blog's March 2026 example, and the live 522 `detail`
is byte-identical to the docs' 522 example. Occurrence-specific data lives in
`instance`/`ray_id`/`timestamp`/`zone`, not `detail`. The docs' own field
table quietly downgrades the promise to *"Plain-text explanation of what went
wrong and which party is responsible"* — dropping "specific to this
occurrence."

**2. `instance` is a bare Ray ID, not a dereferenceable URI.** `[FACT]` Live
value: `"a28817ceee58faca"`. `[COMPARATIVE]` RFC 9457 §3.1.5 defines
`instance` as *"A URI reference that identifies the specific occurrence of the
problem"*, resolved against the document's base URI. A bare hex string is
technically a valid relative URI reference per RFC 3986, so this is not
strictly non-conforming, but it resolves to a nonexistent path and carries no
dereference value. Cloudflare made no attempt to hide the choice — it
duplicates the same value into the `ray_id` extension *"for compatibility with
existing Cloudflare tooling."*

**3. `type` degrades to a non-unique category URL for undocumented codes.**
Covered in Q3(c). `[COMPARATIVE]` This weakens `type` below the RFC 9457
§3.1.1 role of *"the problem type"* identifier.

**4. Duplication across members.** `[FACT]` `status` duplicates the HTTP
status line; `ray_id` duplicates `instance`; `retry_after` duplicates the
`Retry-After` header; `error_code` is embedded in `title` as a prefix ("Error
1015: …"); for 5xx, `error_code` and `status` are the same number (522/522).
`[INFERENCE]` This is deliberate belt-and-braces for consumers reading only
the body, or only the headers, or only the frontmatter — the same information
reachable three ways rather than one canonical location. RFC 9457 §3.1.3
permits `status` but calls it advisory.

**5. `what_you_should_do` embeds Markdown inside a JSON string.** `[FACT]` The
value contains `**Wait and retry.**`, `\n\n`, and a numbered list.
`[COMPARATIVE]` RFC 9457 §3.1 warns against putting markup in Problem Details
fields for security reasons and notes clients should not render them as
markup. Cloudflare ships Markdown in this extension member deliberately —
`[INFERENCE]` because the intended consumer is a language model, for which
Markdown is native, not a browser DOM.

### Extensions and design choices

**6. Extension members as the primary payload.** `[FACT]` Blog: *"The
operational fields — `error_code`, `error_category`, `retryable`,
`retry_after`, `owner_action_required`, and more — are RFC 9457 extension
members. Clients that don't recognize them simply ignore them."*
`[INFERENCE]` The five standard members are the interoperability veneer; all
the decision-driving content is in extensions. Every Cloudflare-authored code
example branches on extension members only.

**7. Legacy numeric codes mapped as an integer extension, not folded into
`type` or `status`.** `[FACT]` `error_code` carries the raw legacy integer
(1015, 522), `error_name` carries a new snake_case symbolic name
(`rate_limited`, `connection_timeout`, `firewall_rule_blocked`,
`country_banned`, `worker_threw_exception`, `origin_dns_error`, `legal_block`,
`cache_connection_limit`, `tls_not_fips_compliant`,
`invalid_ssl_certificate`, `internal_server_error`), and `error_category`
carries a coarse 10-value (1xxx) / 3-value (5xx) routing taxonomy. `status`
carries the orthogonal HTTP status (1015→429, 1020→403, 1016→530, 1026→451,
1200→503, 1101→500). `[INFERENCE]` This is a three-level identity ladder —
category for routing, name for symbolic matching, code for exact identity —
where `type` sits *outside* the ladder as a documentation pointer.

**8. Explicit machine-actionability fields with no RFC analogue.** `retryable`
(boolean), `retry_after` (integer seconds), `owner_action_required`
(boolean), `cloudflare_error` (always `true`, a provenance sentinel
distinguishing Cloudflare-synthesized from origin-generated). Blog: *"You can
replace brittle 'if status == 429 then maybe retry' heuristics with explicit
control flow."*

**9. A parallel non-JSON serialization sharing one semantic contract.**
`[FACT]` Blog: *"Cloudflare exposes one policy contract across two wire
formats. Whether a client consumes Markdown or JSON, the operational meaning
is identical: same error identity, same retry/backoff signals, same
escalation guidance."* `[COMPARATIVE]` RFC 9457 defines
`application/problem+json` and `application/problem+xml`; Cloudflare
implements the JSON one, skips XML entirely, and adds a Markdown format that
is outside the RFC.

**10. Deliberate media-type mirroring rather than always returning
`problem+json`.** `[FACT]` Blog: *"Clients that send
`Accept: application/problem+json` get `application/problem+json;
charset=utf-8` back — useful for HTTP client libraries that dispatch on media
type. Clients that send `Accept: application/json` get `application/json;
charset=utf-8` — same body, safe default for existing consumers."*
`[INFERENCE]` A conformance-vs-compatibility hedge: RFC 9457 §3 says a
Problem Details response *should* use `application/problem+json`, but
Cloudflare prioritizes not surprising clients that asked for plain JSON.

**11. A breaking rename executed to move toward the RFC.** `[FACT]`
`http_status` → `status` (both formats) and `what_happened` → `detail` (JSON
only) on 2026-03-11, flagged under a "Breaking change" heading with *"Agents
consuming Markdown frontmatter should update parsers accordingly."*
`[INFERENCE]` Cloudflare accepted a breaking change to its 13-day-old
Markdown format in order to align field names with RFC 9457 vocabulary —
evidence they treated RFC alignment as worth a compatibility cost.

**12. `[ABSENT]` No design-notes artifact.** There is no RFC/ADR/spec
document, no published JSON Schema, and no versioning statement for the
payload shape. The blog and the docs page are the entirety of the design
rationale, and neither discusses *why* `type` points at docs rather than at an
abstract identifier, nor why the IANA registry was skipped.

---

## Implications for the `AC-003` amendments

Cloudflare is a **strong precedent for the shape and a weak precedent for the
obligation** — they shipped the largest-scale RFC 9457 deployment documented
in this corpus (network-wide, every plan) while quietly declining the parts
of the RFC that a naive mandate would enforce hardest.

### (a) "Must be capable of returning" problem+json, with an infrastructure-error carve-out

**Verdict: strongly supported, including by Cloudflare's own failure modes.**

| Evidence | Bearing |
|---|---|
| `[FACT]` The negotiated default is HTML; problem+json is served only on explicit `Accept`. Blog: *"clients must explicitly request Markdown or JSON."* | **Supports** capability-based wording over "must always return." Cloudflare's mandate on itself is exactly "capable of returning on request," never "returns unconditionally." |
| `[FACT]` Priority order puts Custom Error Rules and Error Pages *above* structured responses; a configured Error Page defeats `Accept` entirely — *"Error Pages do not perform content negotiation."* | **Supports** a carve-out. Even the vendor lets an operator turn the guarantee off, and documents it rather than treating it as a bug. |
| `[FACT, live]` A real enforced 1015 returned `text/plain; charset=UTF-8` / `error code: 1015` despite `Accept: application/problem+json`. | **Supports** the carve-out most directly. An infrastructure protection layer sitting in front of the API surface emitted a legacy body. If Cloudflare cannot guarantee it on every enforcement path, a greenfield standard that mandates it unconditionally will be violated by its adopters' load balancers, WAFs, and rate limiters on day one. |
| `[FACT]` `GET /cdn-cgi/error/9999` → nginx-style HTML 404. | **Supports.** Unrouted/unknown requests never reach the layer that knows how to build problem details. |
| `[FACT]` Origin-generated 5xx *"are passed through by Cloudflare to the client and are not affected."* | **Supports** scoping the obligation to errors the component itself generates. Cloudflare draws exactly this boundary. |

**Wording consequence:** phrase the obligation as a capability of the
*application*, scoped to responses the application itself generates, and name
the carve-out concretely — reverse proxies, CDNs, WAFs, rate limiters, and
load balancers that terminate the request before it reaches application code.
Cloudflare's implementation is the citable existence proof that this carve-out
is real rather than an escape hatch.

### (b) `type` semantics — stable non-resolvable identifier vs. resolvable URL vs. URN

**Verdict: Cloudflare chose resolvable URLs, and their implementation is the
best available argument that resolvable URLs alone are insufficient as the
identity mechanism.**

*Supports mandating resolvable URLs:*
- `[FACT]` Cloudflare emits resolvable `https://` URLs universally; never
  `about:blank`, never a URN, never a relative reference.
- `[FACT]` Eight of eight per-code URLs probed return 200 and land on
  human-readable troubleshooting documentation — the RFC's stated aspiration
  for `type`.
- `[INFERENCE]` For a debugging human or a model reading the payload, a URL
  that opens documentation has obvious value that an opaque identifier lacks.

*Contradicts relying on `type` as the stable identifier:*
- `[FACT]` `type` collides across distinct errors when no per-code page
  exists: 1026 (legal takedown, 451) and 1044 (FIPS cipher, 403) emit the
  **identical** `type` string. Dispatching on `type` cannot distinguish them.
- `[FACT]` `type` values are tied to a documentation site's URL taxonomy, so
  page existence and site IA — not error identity — determine the value.
- `[FACT]` Cloudflare marks **only** `error_name` and `error_category` as
  *"Stable — suitable for programmatic matching."* They pointedly do not make
  that promise about `type`.
- `[FACT]` Zero Cloudflare-authored examples branch on `type`.

*Silent on:* URNs (never mentioned, never used), and any rationale for
choosing docs URLs over abstract identifiers.

**Reading for the amendment.** Cloudflare's design is effectively *"`type` is
a documentation affordance; a separate stable extension member is the
identity."* Three ways a greenfield standard can absorb that lesson:

1. **Mandate resolvable URLs as the identity.** The 1026/1044 collision is a
   live counterexample — when documentation coverage lags error coverage, the
   identity silently degrades and nobody notices.
2. **Mandate stable non-resolvable identifiers.** Consistent with where
   Cloudflare actually put stability, but forgoes the human/model-facing
   value that Cloudflare clearly wanted from `type`.
3. **Mandate a stable identifier and treat resolvability as a separate,
   non-load-bearing quality.** The closest match to what Cloudflare shipped
   once the docs' claims are distinguished from the wire behavior — `type`
   for humans, `error_name`/`error_category` for machines. Also the only
   option under which the 1026/1044 case is not a conformance defect.

If the project does mandate resolvable URLs, Cloudflare's implementation
argues for an explicit requirement that **the URL be distinct per problem type
even when no documentation page exists yet** — precisely the requirement
Cloudflare's docs assert and its edge does not keep.

### (c) Not premising anything on the IANA problem-types registry

**Verdict: supported by total silence, from the largest deployment in
existence.**

- `[ABSENT]` Zero occurrences of "IANA" or "registry" across the full blog
  post text, the full docs page, and all three changelogs (grep-verified).
- `[FACT]` Cloudflare registered no problem types. Every `type` value is a
  private URL under their own domain.
- `[FACT]` Their entire interoperability claim rests on the *shape*, not on a
  shared vocabulary: *"any HTTP client that understands Problem Details can
  parse the five standard members without Cloudflare-specific code."*
- `[FACT]` Semantic interoperability, where Cloudflare wanted it, was built by
  publishing a **private taxonomy** — the ten `error_category` values — and
  documenting per-category agent behavior in a table, rather than by
  registering anything centrally.

`[INFERENCE]` The strongest form of this evidence: the most visible RFC 9457
rollout of 2026, shipped by an infrastructure vendor whose explicit goal was
*"a universal agent contract across the web,"* achieved that goal without
touching the registry and without appearing to consider it. A greenfield
standard that premised conformance on registry participation would classify
Cloudflare's implementation as non-conforming — which is a reductio.

**Caveat:** this is an argument from absence. Cloudflare never states a
position *against* the registry; they simply never mention it. The finding
supports "do not premise anything on it," not "the registry is a bad idea."

---

## Explicitly reported absences

- `[ABSENT]` Whether `/cdn-cgi/error/<code>` is a synthetic preview or the
  same code path as real enforcement — docs call it a test, never say.
- `[ABSENT]` Why the real, IP-enforced 1015 returned legacy `text/plain`
  despite a structured `Accept` header.
- `[ABSENT]` Any statement that `type` URLs are stable, versioned, or
  guaranteed to resolve.
- `[ABSENT]` Any mention of the IANA problem-types registry, or of
  `application/problem+xml`.
- `[ABSENT]` Any published JSON Schema, payload version field, or deprecation
  policy for the extension members.
- `[ABSENT]` Behavior for WAF managed challenges, Turnstile interstitials, and
  `/cdn-cgi/challenge-platform`.
- `[ABSENT]` Cloudflare-generated 4xx errors as a distinct class — announced
  2026-03-11 as "next," no shipping changelog as of 2026-08-09.
- `[ABSENT]` Any adoption metric, traffic percentage, or named third-party
  consumer.
- **Not tested:** whether User-Agent influences negotiation (all requests sent
  `TestAgent/1.0`); the HTML-default rows of the negotiation table live
  (blocked by the rate limiter — documented in two primary sources, not
  independently confirmed).
