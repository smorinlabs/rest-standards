Establish exactly what Cloudflare implemented when it shipped RFC 9457
Problem Details network-wide ("agent error pages", announced 2026-03-11),
so that `AC-003`'s mandate wording can follow a proven large-scale
implementation rather than the announcement of one.

Motivation: `baseline-02d` identified Cloudflare as the newest and most
operationally significant RFC 9457 adoption. Gate C is weighing three
amendments to `AC-003` — (a) obligation worded as "must be capable of
returning" problem+json with an infrastructure-error carve-out; (b) `type`
semantics: stable non-resolvable identifier vs resolvable URL vs URN; (c)
premising nothing on the IANA problem-types registry — and Cloudflare's
implementation choices bear on all three.

Questions:

1. Trigger/negotiation: what exactly gets a client the problem+json
   response; what browsers get; exact `Content-Type` values; how the
   parallel Markdown variant is negotiated and typed.
2. Payload shape: which of the five RFC members are populated and how; the
   full extension-member list with semantics; verbatim example payloads.
3. The `type` member specifically: URL vs URN vs `about:blank`; resolvable
   or not; stable or not; documented policy or accident.
4. Scope: which errors are covered, defaults, and what operators can
   override.
5. Stated rationale and any measured results beyond the launch post.
6. Deviations from or extensions to RFC 9457, and any published reasoning.

Method requirements: read the launch post and all related changelogs and
developer docs; verify live against Cloudflare's production edge where
possible; label `[FACT]` / `[COMPARATIVE]` / `[INFERENCE]` and report
absences explicitly; URL every claim.

Output: findings per question, then a section mapping the implementation
onto amendments (a), (b), and (c) — where it supports, contradicts, or is
silent.
