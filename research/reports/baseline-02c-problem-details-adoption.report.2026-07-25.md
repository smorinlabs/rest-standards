# Baseline 02c — RFC 9457 Adoption Among Greenfield APIs

*Narrow leaf under `baseline-02`. All sources verified 2026-07-25. Tests the
inference behind `AC-003`.*

## Verdict

**The `AC-003` inference holds, and is strengthened materially.**

`baseline-02` §8.1 asserted that the eight surveyed vendors diverge from RFC
9457 for historical reasons rather than on the merits, and that "none of the
eight is a greenfield API that evaluated RFC 9457 and rejected it." That was an
inference about other parties' motives. It now has direct supporting evidence.

**Recommended confidence for `AC-003` at Gate C: raise from moderate to
high-moderate.**

## The framework-default test

`[INFERENCE]` The best available proxy for greenfield behavior is not any
single new API — it is **what a new API emits when its author makes no
deliberate choice**. That is the framework default, and it governs far more new
APIs than any vendor guideline does.

| Framework | Problem Details behavior | RFC cited | Source class |
| --- | --- | --- | --- |
| **ASP.NET Core** (Web API controllers) | ✅ **Enabled by default** | RFC 9457 | **Vendor-primary** (Microsoft Learn, updated 2026-07-22) |
| **Spring Framework / Spring Boot** | ⚙️ **Opt-in**, one property | RFC 9457 | **Vendor-primary** (Spring docs) |

### ASP.NET Core

`[FACT]` Microsoft's own documentation states, of Web API controllers:

> "For web API controllers, MVC transforms an error result to produce a
> `ProblemDetails`. **The automatic creation of a `ProblemDetails` for error
> status codes is enabled by default.**"

It further documents `StatusCodePagesMiddleware` as generating "a problem
details response by default," and `DefaultProblemDetailsWriter` as supporting
`application/problem+json`. The controllers section cites **RFC 9457**
explicitly.

### Spring

`[FACT]` Spring's documentation states:

> "The Spring Framework supports the 'Problem Details for HTTP APIs'
> specification, **RFC 9457**."

It is opt-in — extend `ResponseEntityExceptionHandler`, or set
`spring.mvc.problemdetails.enabled` in Spring Boot — but it is first-class,
current-RFC, and a single switch.

## The decisive case: Microsoft contradicts itself

`[FACT]` Microsoft Graph — one of the eight surveyed references — ships a
proprietary OData-derived error shape
(`error{code,message,target,details,innererror}`), per
`survey-03-representations-errors`.

`[FACT]` Microsoft's own application framework emits RFC 9457 Problem Details
**by default**.

`[INFERENCE]` The same vendor therefore defaults new applications to Problem
Details while its flagship API does not use it. This is very difficult to
explain as a merits-based rejection of RFC 9457 and easy to explain as
incumbency: Graph's error shape predates the relevant RFCs and cannot change
without breaking clients. This is the single strongest piece of evidence
available for `AC-003`'s reasoning, and it comes from vendor-primary
documentation on both sides.

## Disconfirming evidence — sought, not found

> **Superseded 2026-08-09 by `baseline-02d-greenfield-adoption`.** The absence
> claim below is falsified: CAMARA (Linux Foundation) evaluated RFC 9457 in
> minuted issues #133/#157 and its TSC decided "not to be implemented"
> (2024-04-04/15). Two of its three rejection arguments are the incumbency
> mechanism this report identified; one (`type` semantics) is a genuine merits
> objection. The confidence raise this section supported is withdrawn — see
> `baseline-02d` for the re-argued case.

~~`[FACT — absence]` No documented case was located of a greenfield API
evaluating RFC 9457 and choosing a proprietary shape on the merits.~~

`[INFERENCE]` This absence is **weak** evidence and must be labeled as such.
Design rationales for rejecting a standard are rarely published; absence of a
public rejection is not proof no rejection occurred. It is consistent with the
inference rather than confirmation of it. The framework-default evidence above
carries the argument; this does not.

## Sample-bias finding

`[INFERENCE]` The eight surveyed references are large, long-lived, custom-stack
APIs — Stripe, GitHub, Google, Microsoft, Twilio, Shopify, plus two guideline
documents. That population is **structurally the least representative of a new
build**: each has enormous installed-base cost for changing an error format,
and most predate RFC 7807 (2016) entirely.

The "1 of 8" figure is therefore a real measurement of the wrong population for
this decision. It accurately describes what large incumbent APIs do; it does
not describe what a new API would choose. Gate C should weigh it accordingly —
not discount it, but not treat it as a referendum on the merits either.

## Incidental finding — bears on `AC-005`

`[FACT]` Microsoft's error-handling documentation cites **RFC 7807** in its
Minimal APIs sections and **RFC 9457** in its controllers sections, within the
same page. `[INFERENCE]` Even a vendor with excellent documentation practice
carries stale citations three years after RFC 9457 obsoleted RFC 7807. This
corroborates `AC-005` (never cite RFC 7807) as a live hygiene problem rather
than a theoretical one, and suggests the standard should state the supersession
explicitly rather than assume readers know.

## Sources

- https://learn.microsoft.com/en-us/aspnet/core/web-api/handle-errors?view=aspnetcore-10.0
- https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-rest-exceptions.html
- https://datatracker.ietf.org/doc/rfc9457/
- In-repo: `survey-03-representations-errors.report.2026-07-19.md`
