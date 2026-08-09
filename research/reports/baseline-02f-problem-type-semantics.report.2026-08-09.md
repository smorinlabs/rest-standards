# Baseline 02f — Semantics of the RFC 9457 `type` member

*Narrow leaf under `baseline-02`, executing the Gate-C drafting consequence
(b) recorded in `baseline-02d` (2026-08-09): "rule explicitly on `type`". Run
2026-08-09; all URLs accessed 2026-08-09. Does not re-derive the `AC-003`
mandate case, the CAMARA rejection, or the framework-default survey — those
are in `baseline-02c` / `baseline-02d`. Cloudflare's implementation is
examined in depth in `baseline-02e`.*

**Headline:** the claim that the editors declined to split `type` for RFC
7807 back-compatibility — carried in this repo only as `hdamker`'s secondhand
attribution — is now **corroborated by primary IETF text, in the editors' own
words**, on the httpapi list and in the WG issue tracker. The work also
produced two previously unrecorded, primary-sourced failure modes: ASP.NET
Core **shipped a breaking change to its default `type` values between .NET 7
and .NET 8**, and `httpstatuses.com` — a real shipped `type` value — now
redirects to an unrelated marketing site.

---

## Q1 — What RFC 9457 requires, recommends, and permits for `type`

Source throughout: RFC 9457, https://www.rfc-editor.org/rfc/rfc9457.txt
(Standards Track, July 2023; obsoletes RFC 7807).

### Q1.1 §3.1.1 verbatim `[FACT]`

> The "type" member is a JSON string containing a URI reference [URI] that
> identifies the problem type. **Consumers MUST use the "type" URI (after
> resolution, if necessary) as the problem type's primary identifier.**
>
> When this member is not present, its value is assumed to be "about:blank".
>
> If the type URI is a locator (e.g., those with an "http" or "https"
> scheme), dereferencing it **SHOULD** provide human-readable documentation
> for the problem type (e.g., using HTML [HTML5]). However, consumers
> **SHOULD NOT** automatically dereference the type URI, unless they do so
> when providing information to developers (e.g., when a debugging tool is in
> use).
>
> When "type" contains a relative URI, it is resolved relative to the
> document's base URI […] it is **RECOMMENDED** that absolute URIs be used in
> "type" when possible and that when relative URIs are used, they include the
> full path (e.g., "/types/123").
>
> **The type URI is allowed to be a non-resolvable URI.** For example, the
> tag URI scheme [TAG] can be used to uniquely identify problem types:
> `tag:example@example.org,2021-09-17:OutOfLuck`
>
> However, **resolvable type URIs are encouraged by this specification
> because it might become desirable to resolve the URI in the future.** For
> example, if an API designer used the URI above and later adopted a tool
> that resolves type URIs […] taking advantage of that capability would
> require switching to a resolvable URI, **creating a new identity for the
> problem type and thus introducing a breaking change.**

Emphasis added. Requirement summary:

| Aspect | Strength | Source |
| --- | --- | --- |
| `type` present at all | **Optional** (absent ⇒ `about:blank`) | §3.1.1 |
| Value is a URI **reference** (a bare string is legal) | Definitional | §3.1.1 |
| Consumer uses `type` as the primary identifier | **MUST** | §3.1.1 |
| Absolute rather than relative URI | **RECOMMENDED** | §3.1.1 |
| Relative URI includes the full path if used | **RECOMMENDED** | §3.1.1 |
| Dereferencing an `http(s)` `type` yields human-readable docs | **SHOULD** | §3.1.1 |
| A problem type URI resolves to HTML documentation | **SHOULD** | §4 |
| Consumer auto-dereferences `type` | **SHOULD NOT** (except developer/debug tooling) | §3.1.1 |
| Non-resolvable URI (`tag:`, `urn:`) | **Explicitly allowed**, with a caution | §3.1.1 |
| Type URI stability | *No normative keyword* — stated as consequence: changing it is "a breaking change" | §3.1.1 |

`[INFERENCE]` **The overload is in the text, not only in practice.** §3.1.1
puts a `MUST` on identity and a `SHOULD` on documentation on the same string,
then states that moving between the two modes is a breaking change. The RFC
documents the tension; it never resolves it.

### Q1.2 What §4 asks of API authors `[FACT]`

> New problem type definitions **MUST** document: 1. a type URI (typically,
> with the "http" or "https" scheme) 2. a title that appropriately describes
> it (think short) 3. the HTTP status code for it to be used with
>
> A problem type URI **SHOULD** resolve to HTML [HTML5] documentation that
> explains how to resolve the problem.

§4.1: *"you could **mint and document a new type URI (which ought to be under
your control and stable over time)**, an appropriate title and the HTTP
status code that it will be used with, along with what it means and how it
should be handled."*

So the RFC's composite recipe is: an `https` URI, under your control, stable
over time, resolving to HTML docs, with fixed title and fixed status per
type. "Under your control and stable over time" is prose, not a keyword.

### Q1.3 `about:blank` semantics `[FACT]` — §4.2.1

> The "about:blank" URI [ABOUT], when used as a problem type, indicates that
> the problem has **no additional semantics beyond that of the HTTP status
> code**. When "about:blank" is used, the title SHOULD be the same as the
> recommended HTTP status phrase for that code […] Consequently, any problem
> details object not carrying an explicit "type" member implicitly uses this
> URI.

`[INFERENCE]` **Decisive against candidate policy (iv).** `about:blank` is
not "no type"; it is a registered type whose *definition* is "no semantics
beyond the status code". A required, discriminating `code` extension is by
construction additional semantics, and §3.2 scopes extensions to their type:
*"Problem type definitions MAY extend the problem details object with
additional members that are specific to that problem type."* Carrying a
discriminating `code` under `about:blank` contradicts the registered
definition of the type it is scoped to. It also fails §3.1.1's `MUST`: a
conforming consumer must dispatch on `type`, and every response would present
the same one.

### Q1.4 The registry section `[FACT]` — §4.2

- Policy: **Specification Required** (RFC 8126 §4.6).
- *"Vendor-specific, application-specific, and deployment-specific values are
  unable to be registered."*
- *"Registrations MAY use the prefix
  `https://iana.org/assignments/http-problem-types#` for the type URI. **Note
  that those URIs may not be able to be resolved.**"*

`[INFERENCE]` Two usable consequences. The registry is structurally closed to
one organization's problem types, so "register with IANA" is not an available
policy for an ordinary API. And — the strongest single argument that
non-resolving `https` URIs are RFC-legal rather than RFC-breaking — **the
RFC's own registry mints `https` type URIs and says in the same paragraph
they may not resolve.**

### Q1.5 Appendix D — the whole 7807→9457 delta on `type` `[FACT]`

> * Section 4.2 introduces a registry of common problem type URIs
> * Section 3 clarifies how multiple problems should be treated
> * **Section 3.1.1 provides guidance for using type URIs that cannot be
>   dereferenced**

The third item is this leaf's subject: the entire delta is guidance for
non-dereferenceable URIs, added because the field was confused.

### Q1.6 Live registry check `[FACT]`

`https://www.iana.org/assignments/http-problem-types/http-problem-type-uris.csv`,
fetched 2026-08-09 — **6 rows**, unchanged from `baseline-02d`: `about:blank`
(RFC 9457), `#date` and `#ohttp-key` (RFC 9458), three `#digest-*` rows still
referencing `RFC-ietf-httpapi-digest-fields-problem-types-06`, an unpublished
draft. `[FACT — absence]` No entry from any non-IETF party in three years.

---

## Q2 — The IETF debate on splitting identifier from documentation link

`[FACT]` The design record is the WG repository `ietf-wg-httpapi/rfc7807bis` —
75 issues and PRs, highest number #76, spanning 2021-01-28 → 2023-03-27 —
plus two IETF Last Call threads on the `httpapi` list. Relevant items: #11,
#13, #15, #21, #26 (PR), #29 (PR), #62, #63, #64, #65.

### Q2.1 Who raised the split `[FACT]`

| # | Date | Raiser | Proposal |
| --- | --- | --- | --- |
| Issue #11 | 2021-02-02 | Tim Perry (`pimterry`, HTTP Toolkit) | make `type` an opaque string; *"Having a separate optional documentation URL instead would be more useful imo."* |
| Issue #15 | 2021-02-11 | Tim Perry | *"It would be useful to provide an explicit way to link to documentation in these cases, or to make it clear that no documentation is available."* — cites **Zalando** verbatim as the motivating counter-example |
| Issue #13 | 2021-02-11 | Tim Perry | register a `urn:problem:` namespace; recommend e.g. `urn:problem:example.com:out-of-credit` |
| Issue #64 + list | 2022-11-07 / 11-11 | Ben Bucksch | *"Remove the recommendation that the type ID is a resolvable URI with error documentation, and instead add a second 'documentation' field with a link. This removes the coupling between the ID, which needs stability, and hosting of webpages."* |
| httpapi list | 2024-09-04 | Marcus Koch | post-publication: `"type": "Unauthorized"` + `"about": "<doc URL>"` |

Counter-position `[OPINION]`: Asbjørn Ulsberg, 2021-02-02 — *"I think that's
a horrible suggestion. Big 👎🏼 … `type` works perfectly well for
documentation. Decoupling the problem type from the problem type's
documentation when the former can redirect to the latter just adds confusion,
possible inconsistencies, and errors."* His alternative is a controlled
redirect: *"use the same base URI as in the API for `type` URIs … Since
`problems` now is a programmable resource within the API, it's easy to
redirect from the above URI to [the docs URL]. Whenever the URI of the
documentation changes, it's just to change the coded redirection."*
(https://github.com/ietf-wg-httpapi/rfc7807bis/issues/64)

### Q2.2 Why it was rejected — editors, verbatim `[FACT]`

1. **Mark Nottingham, 2021-02-10:** *"that's interesting, but it's
   backwards-incompatible, so we'd need to use a new media type. Is that
   worth it (considering the resulting confusion, etc.)?"* — closing: *"So it
   seems like we have some agreement that we don't want to introduce a new
   mime type, and as a result we shouldn't change the nature of `type` in a
   backwards-incompatible fashion."*
   https://github.com/ietf-wg-httpapi/rfc7807bis/issues/11
2. **Mark Nottingham, 2021-07-26**, specifically on a documentation member:
   *"if we're talking about adding a new member that's valid and standard
   across all problem objects, it violates this statement in 7807: 'Note that
   because extensions are effectively put into a namespace by the problem
   type, it is not possible to define new "standard" members without defining
   a new media type.' So if we decide that this is a must-have, we'll need to
   either define a new media type for problem types, or convince ourselves
   that defining such an extension won't break currently deployed
   extensions."* https://github.com/ietf-wg-httpapi/rfc7807bis/issues/15
3. **Erik Wilde (co-author), 2022-11-14, httpapi list**, answering Bucksch:
   *"first of all, it's really important to always keep in mind that this is
   a revision of an existing spec with the explicit goal to not break
   anything. that limits the maneuvering space. we cannot change the fact
   that type is defined to be as it is, as a URI. … personally, i always have
   been a fan of separating identification and description, but it's a topic
   with no clear decision on what's the better approach; it's a matter of
   taste and of constraints."*
   https://mailarchive.ietf.org/arch/msg/httpapi/3Vk4SjI9jt4IZWSnXTAUPskrdOQ/

Supporting: **Darrel Miller, 2022-11-07** — *"It would also be a breaking
change to RFC 7807 which would fragment the ecosystem and require a new media
type registration."*; **Erik Wilde, 2022-12-22** — *"breaking changes are not
on the table. deprecating existing features is not an option. the revision
has added cautionary text and that's all that possibly can be done."*;
**Mark Nottingham, 2022-11-29**, closing #65 (remove the `about:blank`
default) — *"We can't change this without changing the media type, and it's
too late to do that."*

### Q2.3 The one recorded WG-consensus decision `[FACT]`

**Nottingham, 2021-07-27**, reporting IETF 111: *"Discussed in 111: default
resolvable is good. Not a strong motivation for a 'doesn't resolve' flag;
close with no action (except perhaps prose)."*
https://github.com/ietf-wg-httpapi/rfc7807bis/issues/15

What shipped instead of a split:
- **PR #26** (merged 2021-09-08, fixes #15) added the `MUST … primary
  identifier` sentence, the `SHOULD` on dereferencing a locator, and the
  debugging-tool carve-out on `SHOULD NOT` auto-dereference.
- **PR #29** (merged 2021-09-23, fixes #13) added the `tag:` URI example and
  the breaking-change warning — the "recommend a URN namespace" proposal was
  answered with *an example*, not a registration.

### Q2.4 Reconciling the `hdamker` claim `[FACT]`

`hdamker` (Deutsche Telekom), CAMARA #133, 2024-02-11: *"on the mailing list
of RFC9457 there are several concerns about using the `type` member at the
same time as an identifier AND as a resolvable URI pointing to a description.
The main reasons brought by the editors to not split this into two members
(e.g. introducing a descriptionURL) was the backward compatibility to RFC
7807."* https://github.com/camaraproject/Commonalities/issues/133

**Verdict: substantively accurate, now primary-sourced. Upgrade from
`[COMPARATIVE, secondhand]` to `[FACT]`.** "Several concerns on the mailing
list" is verified by the 2022-11 Last Call thread (Bucksch ↔ Wilde) and the
2024-09 thread (Koch ↔ Wilde, Salz, Maton). "Back-compat was the main reason"
is verified verbatim in Q2.2 items 1–3.

Two nuances to carry with it:
- Back-compat was the *procedural* reason, not the only one. The substantive
  WG position (Q2.3) was that a URI already does both jobs, and a
  resolvability flag would carry too little information to justify the cost —
  Nottingham, 2021-07-26: *"an often under-appreciated aspect of APIs is the
  cognitive load they place upon users."*
- The editors did **not** endorse the resolvable model on the merits. Wilde,
  unprompted, 2024-09-04: *"personally, i am not a fan of the 'use resolvable
  URIs' model because i do think it creates more brittleness than value. but
  the semweb vibe of the last 20 years have meant that this has become sort
  of a dogma."*
  https://mailarchive.ietf.org/arch/msg/httpapi/i3HzIvcAa2xBVCwzpt_4XIzWIoM/

`[FACT — absence]` No pre-2022 `httpapi` list thread proposing a `type`/`href`
split was located; the pre-Last-Call design argument happened in the GitHub
tracker. Search is partly obstructed: `mailarchive.ietf.org` returns HTTP 403
to scripted search requests, so the list was searched through the rendered
`/arch/browse/` interface with messages fetched individually. Treat this as
"not found under the queries used", not "does not exist".

---

## Q3 — Field practice: how adopters populate `type`

### Q3.1 Zalando — non-resolvable relative paths, documented policy `[FACT]`

`https://github.com/zalando/restful-api-guidelines/blob/main/chapters/http-status-codes-and-errors.adoc`
(repo `main`), rule `[#176] {MUST} support problem JSON`:

> **Note:** Problem `type` and `instance` identifiers in our APIs are not
> meant to be resolved. RFC 9457 encourages that problem types are URI
> references that point to human-readable documentation, **but** we
> deliberately decided against that, as all important parts of the API must
> be documented using OpenAPI anyway. In addition, URLs tend to be fragile
> and not very stable over longer periods because of organizational and
> documentation changes and descriptions might easily get out of sync.
>
> In order to stay compatible with RFC 9457 we proposed to use relative URI
> references usually defined by `absolute-path [ '?' query ] [ '#' fragment ]`
> as simplified identifiers in `type` and `instance` fields:
> `/problems/out-of-stock` · `/problems/insufficient-funds` ·
> `/problems/user-deactivated` · `/problems/connection-error#read-timeout`
>
> **Hint:** The use of absolute URIs is not forbidden but strongly
> discouraged.

`[INFERENCE]` Zalando is the strongest *policy* datapoint and the weakest
*mechanical* one. Its rationale (URL fragility, docs drift) is exactly the
greenfield concern; its encoding is the one shape §3.1.1 warns against, and
is self-defeating for its own stability goal — `/problems/out-of-stock`
resolves against the response's base URI, so the identifier differs per host
(Q4.3). **Adopt the rationale; reject the encoding.**

### Q3.2 Belgif (Belgium) — stable URN in `type`, doc URL in `href` `[FACT]`

`https://github.com/belgif/rest-guide/blob/main/guide/src/main/asciidoc/errorhandling.adoc`
(repo `main`, pushed 2026-07-27):

> This guide applies following additional restrictions on the Problem Details
> structure:
> * `type` and `instance` **SHOULD** be specified as absolute URIs …
> * The `type` property, a stable identifier for the type of problem, is
>   **REQUIRED** instead of optional
> * a new optional `href` property **SHOULD** be used to provide a URI to
>   human-readable documentation on the problem type instead of dereferencing
>   the `type` value
>
> Note that using `href` instead of `type` for documentation **intentionally
> deviates from the recommendation in the RFC**. `href` allows use of a URL
> for documentation purposes that may change over time, while `type` can be
> specified as a URN that must remain stable. **This is especially useful for
> API-specific problem types for which the documentation URL may depend on
> technical aspects, like deployment environment.**

Rule `[prb-type]`: *"Problem `type` SHOULD be specified as a URN in one of
following formats: `urn:problem-type:<org>:<api>:<type>` ·
`urn:problem-type:<org>:<type>`"*, `<type>` in lowerCamelCase; *"Reverse FQDN
notation for `<org>` is not recommended in order to keep the identifiers
short and readable."*

```json
{ "type": "urn:problem-type:belgif:resourceNotFound",
  "href": "https://www.belgif.be/specification/rest/api-guide/problems/resourceNotFound.html",
  "instance": "urn:uuid:d9e35127-e9b1-4201-a211-2b52e52508df",
  "status": 404, "title": "Resource is not found",
  "detail": "There is no enterprise in CBE with enterprise number 0206731645" }
```
```json
{ "type": "urn:problem-type:cbss:socialStatus:searchCriteriaTooWide",
  "href": "https://api.ksz-bcss.fgov.be/socialStatus/v2/refData/problemTypes/urn:problem-type:cbss:socialStatus:searchCriteriaTooWide",
  "status": 400, "title": "Search criteria should be more specific" }
```

Three corrections to how this repo has characterized Belgif:
- `[FACT]` **`href` is not "non-standard" in the RFC's frame.** It is a legal
  §3.2 extension member. The only deviation is declining the §3.1.1 `SHOULD`
  on `type` resolvability — which the RFC permits, and which Belgif justifies
  in writing.
- `[FACT]` `urn:problem-type:` is **not a registered IANA URN namespace**.
  The IANA URN Namespaces registry
  (https://www.iana.org/assignments/urn-namespaces/urn-namespaces.xhtml,
  fetched 2026-08-09) contains no `problem-type` NID. Under RFC 8141 the NID
  should be registered; Belgif's URNs are well-formed but formally
  unregistered. That is a real cost of candidate policy (iii).
- `[FACT]` The guide records the doc-URL churn it designed around:
  *"WARNING: `href` links to descriptions of standardized problem types will
  change in the future to recommended format containing full problem type
  identifier without HTML extension. Current URLs will redirect."* The
  documentation URL is moving; the `type` URN is not. **The split working as
  intended, observed in the wild.**

### Q3.3 ACME / RFC 8555 — formal IETF URN sub-namespace `[FACT]`

RFC 8555 §6.7: *"To facilitate automatic response to errors, this document
defines the following standard tokens for use in the 'type' field (within the
ACME URN namespace `urn:ietf:params:acme:error:`)"* — 21 tokens (`badNonce`,
`badCSR`, `caa`, `rateLimited`, `compound`, `userActionRequired`, …). §9.7.4
creates an IANA **"ACME Error Types"** registry: *"Type: The label to be
included in the URN for this error, following `urn:ietf:params:acme:error:`"*.

`[INFERENCE]` The existence proof for candidate policy (iii): a
non-resolvable URN namespace with a registry and many interoperating
implementations (Let's Encrypt, ZeroSSL, Google Trust Services). Cost:
`urn:ietf:params:…` is available only to an IETF specification (RFC 3553). A
private organization needs its own registered NID — the step Belgif skipped.

### Q3.4 Cloudflare — resolvable https docs URL + machine `error_code` `[FACT]`

https://blog.cloudflare.com/rfc-9457-agent-error-pages/ (2026-03-11). The
`type` value below was confirmed verbatim in the page source by direct fetch,
not only through a summarizer. Full implementation detail, including the
category-index fallback that makes `type` non-unique for undocumented codes,
is in `baseline-02e`.

```json
{ "type": "https://developers.cloudflare.com/support/troubleshooting/http-status-codes/cloudflare-1xxx-errors/error-1015/",
  "title": "Error 1015: You are being rate limited", "status": 429,
  "detail": "You are being rate-limited by the website owner's configuration.",
  "instance": "9d99a4434fz2d168",
  "error_code": 1015, "error_name": "rate_limited", "error_category": "rate_limit",
  "ray_id": "9d99a4434fz2d168", "retryable": true, "retry_after": 30 }
```

`[INFERENCE]` The closest existing analogue to `AC-003` + `AC-004` combined,
and the newest large-vendor datapoint: a **resolvable https `type` under a
controlled documentation domain** carried alongside a **stable machine key**
(`error_code` 1015 / `error_name` `rate_limited`) that is not the `type` URL.
The stability promise is made about the schema, not the URL: *"The schema is
stable by design, so agents can implement durable control flow without
chasing presentation changes."* The durable dispatch key Cloudflare offers
consumers is `error_code`.

### Q3.5 ASP.NET Core — RFC section anchors on a deprecated host, and a shipped identity break `[FACT]`

Primary source: `dotnet/aspnetcore`,
`src/Shared/ProblemDetails/ProblemDetailsDefaults.cs`:

```csharp
public static readonly Dictionary<int, (string Type, string Title)> Defaults = new()
{
    [400] = ( "https://tools.ietf.org/html/rfc9110#section-15.5.1", "Bad Request" ),
    [401] = ( "https://tools.ietf.org/html/rfc9110#section-15.5.2", "Unauthorized" ), …
```

Applied unconditionally by `DefaultProblemDetailsWriter.WriteAsync` via
`ProblemDetailsDefaults.Apply(...)`, and mirrored into MVC's
`options.ClientErrorMapping[statusCode].Link` by
`ApiBehaviorOptionsSetup.ConfigureClientErrorMapping`.

**Version drift, verified by git tag** `[FACT]`:

| Tag | `type` for 400 | `type` for 401 |
| --- | --- | --- |
| `v7.0.0` | `https://tools.ietf.org/html/rfc7231#section-6.5.1` | `https://tools.ietf.org/html/rfc7235#section-3.1` |
| `v8.0.0`, `v9.0.0`, `main` | `https://tools.ietf.org/html/rfc9110#section-15.5.1` | `https://tools.ietf.org/html/rfc9110#section-15.5.2` |

Changed by commit `77c461e9` (#43232, 2022-08-26); more types added by
`44a9f8a8` (#58101, 2024-10-09); a 500-title fix in `813b5e76` (#65590,
2026-07-01).

`[INFERENCE]` Under §3.1.1's `MUST`, a conforming client dispatching on
`type` saw **every problem type identity in the platform change** when a
service upgraded .NET 7 → .NET 8. The most widely deployed Problem Details
implementation shipped exactly the breaking change the RFC warns about, as a
routine framework upgrade, with no deprecation path — and it did so because
the identifier tracked *a third party renumbering its own documents* (RFC
7231/7235 → RFC 9110). That is the single best-evidenced argument against
letting a documentation URL double as the identifier.

### Q3.6 Spring — `about:blank`, i.e. nothing to dispatch on `[FACT]`

`org.springframework.http.ProblemDetail`:

| Tag | Declaration | Effect |
| --- | --- | --- |
| `v6.0.0` (Nov 2022) – `v6.2.0` | `private static final URI BLANK_TYPE = URI.create("about:blank"); private URI type = BLANK_TYPE;` | emits `"type":"about:blank"` |
| `v7.0.0` (Nov 2025), `main` | `private @Nullable URI type;` — javadoc: *"By default, this is not set. According to the spec, when not present, the type is assumed to be 'about:blank'"* | omits `type` |

`[INFERENCE]` Semantically identical under §3.1.1, so not a breaking identity
change — but the Spring default gives a client **nothing to dispatch on**,
which is precisely why `AC-004` must require `type` explicitly rather than
inherit a framework default.

### Q3.7 Others

- `[FACT]` **Adyen** runs two formats. Classic Payments API: proprietary
  `status`/`errorCode`/`message`/`errorType`
  (https://docs.adyen.com/development-resources/error-codes/).
  Configuration, Balance Platform and Management APIs: *"The error responses
  use the RFC 7807 format"*, with `type` documented as *"A URL of a
  documentation page where you can find more information about the error"*
  (https://docs.adyen.com/errors/). `[INFERENCE]` Adyen's prose redefines
  `type` as a documentation locator and drops the identifier role — the
  overload resolved opposite to Belgif. `[FACT — absence]` No concrete
  example `type` value published.
- `[FACT]` **SmartBear** operates a third-party registry at
  https://problems-registry.smartbear.com/ (20 types; source
  https://github.com/SmartBear-DevRel/problems-registry) with resolvable
  values such as `https://problems-registry.smartbear.com/missing-request-parameter`,
  promoted in https://swagger.io/blog/problem-details-rfc9457-api-error-handling/.
  `[INFERENCE]` A vendor registry filling the gap left by an IANA registry
  closed to application-specific values (Q1.4) — the shared-vocabulary demand
  is real and IANA is not meeting it.
- `[FACT]` **`khellang/Middleware`** (third-party ASP.NET Core middleware)
  shipped `"type": "https://httpstatuses.com/404"` as a default; its own
  issue #7 argues this violates RFC 7807 §4's advice that generic problems
  are better expressed as plain status codes.
  https://github.com/khellang/Middleware/issues/7
- `[FACT — absence]` No Problem Details adoption or `type` guidance found for
  **GitLab** (`message` envelope), **Microsoft REST API Guidelines**
  (`error.code`/`message`), **Twilio** (`code`/`message`/`more_info`),
  **PayPal** (`name`/`details`/`links[rel=information_link]`), **Ping
  Identity** (`id`/`code`/`message`). **Atlassian, Stoplight, Apiwiz**:
  nothing located — absence under the queries used, not a confirmed negative.
- `[FACT — absence]` No problem types defined by RFC 9530 (Digest Fields),
  RFC 9728 (OAuth Protected Resource Metadata), or
  `draft-ietf-httpapi-ratelimit-headers`. The only non-ACME IETF problem
  types are RFC 9458's two plus three from the unpublished
  `digest-fields-problem-types` draft, all using the
  `https://iana.org/assignments/http-problem-types#…` fragment form.

### Q3.8 Comparison table

| Adopter | Example `type` value | URL or URN | Resolvable? | Stable by promise? | Policy or accident? |
| --- | --- | --- | --- | --- | --- |
| **Zalando** `[#176]` | `/problems/out-of-stock` | relative URI reference (rooted path) | No — deliberately | Intended; **undermined by base-URI resolution** | **Documented policy**, written rationale |
| **Belgif** `[err-problem]`/`[prb-type]` | `urn:problem-type:belgif:resourceNotFound` (+ `href`) | URN, **unregistered NID** | No — by construction | Yes — *"MUST remain stable over time"* | **Documented policy**, deviation stated |
| **ACME / RFC 8555** | `urn:ietf:params:acme:error:badNonce` | URN, IETF namespace + IANA sub-registry | No | Yes — Standards Track + registry | **Documented policy**, protocol-level |
| **Cloudflare** (2026-03-11) | `https://developers.cloudflare.com/support/troubleshooting/http-status-codes/cloudflare-1xxx-errors/error-1015/` | https URL, controlled domain | **Yes** | Schema stability promised; **URL stability not promised** | **Documented policy**; dispatch key is `error_code` |
| **ASP.NET Core ≥8** | `https://tools.ietf.org/html/rfc9110#section-15.5.1` | https URL, **third-party host** | Yes, via 301 to `datatracker.ietf.org` | **No — changed in .NET 8** | **Framework default (accident for the API author)** |
| **ASP.NET Core 7** | `https://tools.ietf.org/html/rfc7231#section-6.5.1` | https URL, third-party host | Yes, via 301 | superseded | Framework default |
| **Spring ≥6.0** | `about:blank` (≤6.2 explicit, ≥7.0 omitted) | special scheme | n/a | n/a | Framework default — **no discriminating type at all** |
| **Adyen** (Balance Platform / Management) | not published; documented as *"A URL of a documentation page"* | https URL | Intended yes | Not stated | Documented policy; identifier role dropped |
| **SmartBear registry** | `https://problems-registry.smartbear.com/missing-request-parameter` | https URL, vendor registry | Yes | Not stated | Documented policy (registry) |
| **IANA registry** (RFC 9457 §4.2) | `https://iana.org/assignments/http-problem-types#date` | https URL + fragment | **No** — RFC: *"may not be able to be resolved"* | Yes | **Documented policy — the RFC's own** |
| **RFC 9457 §3.1.1 example** | `tag:example@example.org,2021-09-17:OutOfLuck` | `tag:` URI | No | Yes | Spec example |
| **`khellang/Middleware`** | `https://httpstatuses.com/404` | https URL, third-party host | **No — domain repurposed** | No | Library default; flagged in its own tracker |

---

## Q4 — Failure modes, with primary evidence

### Q4.1 The RFC's own statement of the trap `[FACT]`

§3.1.1: switching a `type` from non-resolvable to resolvable is *"creating a
new identity for the problem type and thus introducing a breaking change."*
The trap is symmetric — any change to the identifier, including one made to
fix a documentation URL, breaks the API contract.

### Q4.2 Type-URL rot, observed `[FACT]`

| Case | Status on 2026-08-09 | Consequence |
| --- | --- | --- |
| `https://tools.ietf.org/html/rfc9110#section-15.5.1` (ASP.NET Core default) | HTTP **301** → `https://datatracker.ietf.org/doc/html/rfc9110`. The target page **does** carry `id="section-15.5.1"`, and per RFC 9110 §10.2.2 a client inherits the original fragment when `Location` has none — so the deep link still lands correctly. | The documentation promise currently holds, **but only through a redirector the IETF has deprecated** (migration completed 2022-06-17, https://www.ietf.org/blog/finalizing-ietf-tools-transition/). The identifier's fate is bound to a third party's hostname policy — and the .NET 7→8 identity break (Q3.5) happened precisely because the identifier tracked that third party renumbering its documents. |
| `https://httpstatuses.com/404` (`khellang/Middleware` default) | HTTP **301** → `https://www.webfx.com/web-development/glossary/` | **Worse than a 404: a live, plausible, unrelated page.** A developer or agent following the link is silently misinformed. |

`[INFERENCE]` The rot case rests on the second row plus Q3.5; the first row
is a *dependency* finding, not a breakage finding. Neither is a small shop's
neglect — one is a domain that changed hands, the other the IETF's own
hostname policy. **A stability promise about a `type` URL is a promise about
DNS and a documentation site over a decade; the corpus contains an
implementation that made it and did not keep it.**

### Q4.3 Environment leakage / relative-URI drift `[FACT]`

Ben Bucksch, issue #64, 2022-11-07:

> APIs may be available on different endpoints (URLs). Depending on the
> endpoint, that would create different IDs. … He and QA are testing against
> a dev or staging server and they do not know or realize that the production
> server will have a different API endpoint URL. **So, it works in testing,
> but fails in production.**

and 2022-11-08: *"Common practice is that API endpoint URLs are configurable,
and to have different endpoints for testing and production environments, so
using relative URIs pretty much **guarantees** that the `type` IDs are not
stable, and even change between testing and production environment."*
https://github.com/ietf-wg-httpapi/rfc7807bis/issues/64

`[FACT]` The WG declined to act — Nottingham, 2022-11-29: *"the specification
already warns about the issues involved when relative URIs are used. Given
the constraints that others have mentioned, I don't think we can do much
more."* Counter-position, Austin Wright, 2022-12-22: *"The scenario you
outlined is one where the application should use a constant, full URI and not
a URI reference."*

`[FACT]` Belgif independently confirms the documentation-side version of the
same hazard as its reason for `href`: *"especially useful for API-specific
problem types for which the documentation URL may depend on technical
aspects, **like deployment environment**."*

`[INFERENCE]` Zalando's `/problems/out-of-stock` sits squarely in this
failure mode. It is stable only because implementations — including Zalando's
own `zalando/problem` Java library, per Tim Perry's 2021-02-02 analysis in
issue #11 — compare the raw string and never perform the base-URI resolution
§3.1.1 requires. **Zalando's scheme works because clients violate the RFC.**

### Q4.4 Client coupling `[FACT]`

§3.1.1's `MUST` makes coupling mandatory: a conforming consumer is *required*
to key on `type`. There is no conforming way to make anything else the
primary identifier. Every property the identifier lacks — stability,
environment-independence, host-independence — is a defect the client absorbs.
This is why policy (iv) is not merely unconventional but non-conforming
(Q1.3).

### Q4.5 Versioning of problem types `[FACT]`

One observed break and one observed avoidance:
- **Break:** ASP.NET Core .NET 7 → .NET 8 (Q3.5) — every default type
  identity changed, tracking an RFC renumbering unrelated to the API's
  semantics.
- **Avoidance:** Belgif's `href` warning (Q3.2) — documentation URLs are
  announced as moving, `type` URNs are not, and the announcement is
  *possible because they are separate fields*.

`[FACT — absence]` No adopter surveyed publishes a versioning or deprecation
policy for problem types themselves (how to retire a type, whether a type may
be narrowed). Nothing for Zalando, Belgif, Cloudflare, ACME, or the IANA
registry.

---

## Q5 — The five candidate policies

`AC-004` already requires `type`, `title`, `status`, and a stable
machine-readable `code` extension on every problem document. That changes the
calculus for every option below, because a discriminator of last resort
already exists.

### (i) Resolvable `https` under a controlled domain, with a stability promise

**For** `[FACT]`: follows §3.1.1 `SHOULD`, §4 `SHOULD`, §4.1's "under your
control and stable over time"; matches the IETF 111 consensus (*"default
resolvable is good"*); implemented at scale by Cloudflare; Ulsberg's redirect
pattern decouples the docs URL from the identifier while keeping one field.
**Against** `[FACT]`: the promise is about DNS plus a docs site over a
decade. ASP.NET Core broke type identity across a major version;
`httpstatuses.com` changed hands; the ASP.NET values today depend on a
redirector the IETF has deprecated. Zalando and Belgif each refused this
`SHOULD` in writing, and a co-author calls the model *"more brittleness than
value"*.

### (ii) Non-resolvable stable URIs (`tag:`, `urn:`, or `https` that need not resolve)

**For** `[FACT]`: explicitly permitted — *"The type URI is allowed to be a
non-resolvable URI"* — with a `tag:` example; the RFC's own IANA registry
mints `https` URIs that *"may not be able to be resolved"*; two independent
mandating adopters chose it. Deviating from a `SHOULD` with documented reason
is conformant behavior.
**Against** `[FACT]`: §3.1.1's caution — you cannot later make it resolvable
without a breaking change; and a bare 404 disappoints every tool and human
that follows it (Perry's tooling argument, #15). *This objection is answered
in the recommendation by permitting an optional redirect.*

### (iii) A URN scheme, formally

**For** `[FACT]`: ACME proves it at internet scale with a registry; Belgif
proves it for a national guideline; a URN cannot be mistaken for a locator.
**Against** `[FACT]`: `urn:ietf:params:…` is reserved to IETF specifications
(RFC 3553), and `urn:problem-type:` is **not a registered IANA NID**.
Recommending a URN means recommending either an NID registration most
adopters will not perform, or a technically unregistered namespace.

### (iv) `about:blank` + rely on the `code` extension

**Against, decisively** `[FACT]`: §4.2.1 defines `about:blank` as *"no
additional semantics beyond that of the HTTP status code"*, and §3.2 scopes
extension members to the type that defines them — so a discriminating `code`
under `about:blank` contradicts its registered definition. §3.1.1's `MUST`
makes `code`-only dispatch non-conforming, and §4.2.1's
`title`-equals-status-phrase rule blocks carrying meaning in `title` instead.
This is the defect `hdamker` identified in CAMARA's proposal and called
*"worse than staying with our current proprietary structure"*.
**For:** nothing beyond "it is the Spring default". No surveyed guideline
recommends it.

### (v) Split: stable `type` + separate documentation member

**For** `[FACT]`: proposed independently over four years (Perry 2021, Bucksch
2022, Koch 2024); shipped and running in Belgif; the co-author's personal
position (*"i always have been a fan of separating identification and
description"*); the only option under which a documentation URL can move
without a breaking change — and Belgif's live `href` warning shows that
happening.
**Against** `[FACT]`: rejected at IETF as a *standard* member because it
would require a new media type — a constraint binding the RFC, **not** a
private API standard, since §3.2 permits any extension member. Cost: not
interoperable, so generic tooling will not know it.

`[INFERENCE]` **(i) and (v) are not exclusive, and the field's two
best-evidenced adopters each implement half of the combination.** Cloudflare
puts a resolvable docs URL in `type` and a stable machine key in
`error_code`; Belgif puts a stable machine key in `type` and a movable docs
URL in `href`. Both separate the durable identifier from the movable
documentation locator, differing only in which member carries which. Given
`AC-004` already mandates a stable `code`, the greenfield standard is one
decision away from the same architecture — and the ASP.NET evidence settles
which member should carry the movable thing.

---

## Recommendation for the greenfield standard

**Adopt (ii) + (v): a stable, absolute `type` URI under a controlled domain
that clients must not depend on dereferencing, plus a separate documentation
member — with `code` as the ergonomic dispatch key and `type` as its
normative URI form.**

**Why not (i).** `[INFERENCE]` (i) is the RFC's preference and is defensible,
but it requires a multi-decade promise about a documentation URL, and the
corpus contains a case where the most-deployed implementation in the world
broke every type identity as a routine upgrade, plus a case where a shipped
`type` URL now serves unrelated content. A standard should not require a
guarantee its own reference implementations demonstrably fail to keep. (v)
removes the need for the guarantee; (ii) makes the removal honest by not
*contracting* the identifier as a locator.

**Why not a URN.** `[FACT]` `urn:problem-type:` is unregistered and
`urn:ietf:params:` is closed to non-IETF parties. An `https` URI under a
domain the API owner controls gives the same stability with no registration
step and a built-in uniqueness guarantee (DNS), and RFC 9457 §4.2 blesses
non-resolving `https` type URIs for its own registry. `[POLICY]` A URN SHOULD
remain permitted for organizations already operating a registered NID, so
Belgif- and ACME-style APIs stay conformant.

### Exact interaction with the required `code` extension

This is the crux, because §3.1.1 makes `code`-only dispatch non-conforming.
Bind the two members by construction rather than leaving them independent:

1. **`type` is the normative identifier; `code` is its short form.** Every
   problem type has exactly one `code` (a stable machine token) and exactly
   one `type`, and `type` is derived from `code` by a fixed published
   template — `code: "out_of_credit"` ⇒
   `type: "https://problems.example.com/out-of-credit"`. One value, two
   encodings; they cannot drift, and a reviewer can check the pair
   mechanically.
2. `type` therefore satisfies §3.1.1's `MUST` for conforming consumers, while
   `code` supplies the ergonomic switch key that every surveyed proprietary
   format provides and that `pjhac` said RFC 9457 lacks.
3. **Neither may change once published.** A new meaning is a new `code` and a
   new `type`, never an edit to either.
4. The documentation link lives in a **third member** (`documentation`), is
   not an identifier, and MAY change or vary by environment.
5. **A provider MAY serve an HTTP redirect at the `type` URI to current
   documentation** (Ulsberg's pattern) — but resolvability is a courtesy,
   never a contract, and clients MUST NOT depend on it. This answers the
   "https that 404s is the worst variant" objection without reintroducing the
   URL-stability promise.
6. `about:blank` is permitted by the RFC only where the problem adds nothing
   to the status code — in which case `code` must be absent too, since
   §4.2.1 admits no additional semantics. `[POLICY]` **Recommend banning
   `about:blank` outright.** A standard requiring `type` and `code` on every
   document has no use for a type meaning "no type", and the ban removes the
   §4.2.1 conflict rather than managing it. This is stricter than `AC-004` as
   currently worded and needs a Gate-C decision.

### Proposed normative wording (core, 3 sentences)

> The `type` member **MUST** be an absolute URI under a domain the API
> provider controls, **MUST** identify exactly one problem type, **MUST**
> correspond one-to-one with the document's `code` member by the provider's
> published template, and **MUST NOT** change once published — a change of
> meaning is published as a new problem type with a new `type` and a new
> `code`. A `type` URI is **not required to be dereferenceable** and clients
> **MUST NOT** depend on dereferencing it; where human-readable documentation
> exists it **MUST** be supplied in the separate `documentation` member,
> whose value **MAY** change over time and **MAY** vary by deployment
> environment. The `type` member **MUST** be present on every problem
> document, and `about:blank` **MUST NOT** be used.

Supplementary rules, if wanted alongside the core: a provider **MAY** serve a
redirect from the `type` URI to current documentation, provided no client
behavior depends on it; a `urn:` `type` is **permitted** for providers
operating an IANA-registered URN namespace identifier.

`[POLICY]` Three deliberate deviations from RFC 9457, all permitted by it,
all to be recorded as project policy rather than protocol law: (a) declining
the §3.1.1/§4 `SHOULD` on resolvability, on the Q4.2/Q3.5 evidence — the same
deviation Zalando and Belgif each made in writing; (b) requiring `type`,
which the RFC makes optional; (c) forbidding `about:blank`, which the RFC
registers and defaults to. Only (c) is stricter than any surveyed adopter.

`[POLICY]` Name the member `documentation`, not `href`. Belgif's `href` is a
legal §3.2 extension, but the name collides conceptually with hypermedia link
members and nothing is gained by copying it — there is no interoperating
consumer of `href` to be compatible with.

### Open questions for Gate C

1. **`about:blank` ban vs. carve-out in `AC-004`** — owner decision;
   recommendation above is "ban".
2. **Does the `code` ⇒ `type` template belong in the standard or in each
   API's description?** Recommend: the standard fixes the shape
   (`<https base>/<code, underscores to hyphens>`), each API fixes the base
   URI.
3. **`https` vs `tag:` for the non-dereferenced identifier** — `tag:`
   signals non-resolvability most honestly, but `[FACT — absence]` no
   surveyed adopter uses `tag:` in production; only the RFC's own example
   does. Recommend `https`, matching IANA's own practice, with the
   optional-redirect rule (supplementary rule above) as the reason the choice
   is safe.

---

## Strongest sources

| # | Source | Class | Why load-bearing |
| --- | --- | --- | --- |
| 1 | https://www.rfc-editor.org/rfc/rfc9457.txt §3.1.1, §3.2, §4, §4.1, §4.2, §4.2.1, App. D | primary standard | Exact requirement strengths, `about:blank` definition, the RFC's own breaking-change warning, IANA's non-resolving https precedent |
| 2 | https://mailarchive.ietf.org/arch/msg/httpapi/3Vk4SjI9jt4IZWSnXTAUPskrdOQ/ | primary (IETF list) | Co-author Wilde stating the back-compat constraint at Last Call — corroborates `hdamker` |
| 3 | https://github.com/ietf-wg-httpapi/rfc7807bis/issues/15 | primary (WG tracker) | Nottingham's new-media-type argument against a standard documentation member; IETF 111 outcome |
| 4 | https://github.com/ietf-wg-httpapi/rfc7807bis/issues/11 | primary (WG tracker) | Full identifier-vs-locator argument; `revisit-on-breaking-change` disposition |
| 5 | https://github.com/ietf-wg-httpapi/rfc7807bis/issues/64 | primary (WG tracker) | Staging-vs-production relative-URI failure mode, argued and declined |
| 6 | https://mailarchive.ietf.org/arch/msg/httpapi/i3HzIvcAa2xBVCwzpt_4XIzWIoM/ | primary (IETF list) | Wilde post-publication: *"i am not a fan of the 'use resolvable URIs' model"* |
| 7 | `dotnet/aspnetcore` `src/Shared/ProblemDetails/ProblemDetailsDefaults.cs` at tags `v7.0.0`/`v8.0.0`/`v9.0.0` | primary (source) | The .NET 7→8 breaking change to default `type` identities |
| 8 | https://github.com/belgif/rest-guide/blob/main/guide/src/main/asciidoc/errorhandling.adoc | primary (guideline source) | The `type`/`href` split, the stability rule, the live doc-URL-churn warning |
| 9 | https://github.com/zalando/restful-api-guidelines/blob/main/chapters/http-status-codes-and-errors.adoc | primary (guideline source) | Refusal of the resolvability `SHOULD`, with rationale and concrete values |
| 10 | https://blog.cloudflare.com/rfc-9457-agent-error-pages/ | vendor-primary | Resolvable `type` + separate stable `error_code` at network scale, 2026-03-11 |
| 11 | https://www.rfc-editor.org/rfc/rfc8555.txt §6.7, §9.7.4 | primary standard | The only registry-backed non-resolvable URN problem-type namespace in production |
| 12 | https://www.iana.org/assignments/http-problem-types/http-problem-type-uris.csv + https://www.iana.org/assignments/urn-namespaces/urn-namespaces.xhtml | primary registries | 6 entries after 3 years; `urn:problem-type` unregistered |
