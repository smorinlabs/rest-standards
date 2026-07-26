Determine whether RFC 10008 (the HTTP QUERY method) has sufficient deployed
support to justify recommending it in a public REST design standard, and
specifically whether principle `HS-009` should remain `MAY` or be promoted to
`SHOULD`.

Exact scope: support for QUERY in HTTP parsers and runtimes; HTTP client
tooling; server frameworks; browsers and the Fetch API; and — decisively —
whether any production CDN, reverse proxy, or shared cache implements
**body-derived cache keys** for QUERY responses as RFC 10008 requires.

Do not re-decide whether QUERY is semantically correct or desirable; RFC 10008
settles its properties. This leaf answers a deployment question only.

Research requirements:

1. Distinguish three separate capabilities and do not conflate them:
   (a) parsing/accepting the method, (b) passing it through to an origin, and
   (c) caching the response with the request body incorporated into the cache
   key. Only (c) delivers the advantage QUERY has over POST.
2. Prefer primary sources: the implementing project's own repository, release
   notes, issue tracker, or official documentation. A vendor's own engineering
   blog counts as primary for that vendor.
3. Verify every material claim against at least two independent sources.
   Secondary technology blogs do not corroborate one another.
4. Explicitly record the **absence** of evidence where a capability cannot be
   confirmed. An unverifiable vendor claim is a finding, not a gap to be
   filled with a plausible assumption.
5. Note that two of RFC 10008's three authors are affiliated with CDN vendors
   and assess whether that constitutes evidence of deployed support or only of
   design intent.

Output: a capability matrix by implementation and capability class; an
explicit verdict on whether `HS-009` should change strength; and a statement
of what evidence would change the verdict.
