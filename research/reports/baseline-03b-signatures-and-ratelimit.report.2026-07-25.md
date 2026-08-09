# Baseline 03b — RFC 9421 Deployment and RateLimit Draft Trajectory

*Narrow leaf under `baseline-03`. All sources verified 2026-07-25. Tests the
two principles that diverge from universal vendor practice.*

## Verdicts

| Principle | Verdict |
| --- | --- |
| `OP-016` (prefer RFC 9421) | **Keep as "prefer." Raise confidence.** The mechanism is proven at internet scale — but in a different use case than webhooks. |
| `OP-010` (IETF RateLimit fields) | **Keep, but lower confidence and add an explicit expiry contingency.** The draft is technically sound and editorially troubled, with no advancement path in seven years. |

---

## Part 1 — RFC 9421 is deployed at scale, in an adjacent use case

`[FACT]` **Cloudflare has shipped RFC 9421 verification in production.** Its
own engineering blog announces: *"Starting today, Cloudflare is ramping up
verification of cryptographic signatures provided by automated crawlers and
bots."* Available on Free and Pro plans, rolling out to Business and
Enterprise. Signatures are validated at Cloudflare's edge with no site-owner
configuration.

`[FACT]` The deployment is **Web Bot Auth** — an application-layer profile of
RFC 9421 scoped to bot and agent authentication. It adds three things atop the
RFC: a `tag="web-bot-auth"` parameter in `Signature-Input`, a `Signature-Agent`
header pointing at the bot's key directory, and a well-known URL convention at
`/.well-known/http-message-signatures-directory` for key discovery.

`[FACT]` Cloudflare, Amazon, Akamai, and OpenAI back Web Bot Auth, with an IETF
working group chartered in 2026.

> **Scope capped 2026-08-09 by `baseline-03d-webhook-signing-adoption`.** The
> IETF webbotauth WG charter explicitly excludes *"authenticating access to
> content not intended for human consumption (e.g., HTTP APIs, agent-to-agent
> interfaces)"* — so Web Bot Auth will not grow into a webhook profile, and
> this deployment must not be read as a leading indicator for webhook
> signing. It remains valid evidence that RFC 9421 is implementable at scale,
> and nothing more. Direct webhook deployments of RFC 9421 now exist instead
> (UCP, AdCP, Qerko) — see `baseline-03d`.

### The distinction that governs `OP-016`

`[INFERENCE]` This is strong evidence for one claim and no evidence for
another, and the two must not be merged:

- **Proven:** RFC 9421 is implementable, verifiable at CDN edge scale, and
  backed by four major infrastructure vendors. The 2024 concern that it was a
  standard nobody had implemented is **no longer true**.
- **Not proven:** that webhook *consumers* can verify RFC 9421 signatures.
  Cloudflare's deployment verifies signatures on **inbound crawler traffic** at
  the edge. A webhook flows the other way — provider signs, consumer verifies,
  and the consumer is typically an ordinary application server with no CDN in
  the path. Cloudflare's edge doing verification says nothing about a
  consumer's ability to do so.

`OP-016` therefore stays at "prefer RFC 9421, with ad-hoc HMAC-SHA256 as a
documented interim" — but its **confidence rises from moderate**, because the
principal objection (nobody has implemented it) is now answered.

`[FACT]` Library support exists across Go (`yaronf/httpsign`), Python
(`pyauth/http-message-signatures`), .NET (NSign), Kotlin, and others.

---

## Part 2 — The RateLimit draft: sound design, stalled process

### Revision history

| Revision | Date |
| --- | --- |
| draft-11 | 2026-05-23 (current) |
| draft-10 | 2025-09-27 |

`[FACT]` Expires **2026-11-24**. Intended RFC status: **"(None)."** No IETF
Last Call, no working-group Last Call, no IESG submission. The work originated
in **2019** — seven years without advancing.

### The HTTPDIR review — read, not assumed

`[FACT]` The HTTP Directorate early review of draft-10 by **Lucas Pardue**,
dated **2026-01-16**, returned a verdict of **"Not Ready."**

`[FACT]` The substance of that verdict matters more than its label. Pardue
marked it not ready because of *"several minor issues... that add up to a
document lacking precision of core protocol elements while containing
distracting bloat,"* and judged that *"a strong editorial pass through the
document would improve the ability for new implementers to use it."*

Critically, he also wrote that **"the technical design presented seems sound"**
and that the appendix B examples *"suggest several aspects have been
thoughtfully considered."*

> **Falsified 2026-08-09 by `baseline-03f-ratelimit-draft-trajectory`.** The
> inference below does not survive the editors' own response to the review:
> Pardue's largest concern was parameter extensibility (not editorial), and
> the editors' answering PR #166 **renames the wire parameters** (`r`→`a`,
> `t`→`w`) and adds five IANA registries. The wire format is a moving
> target, not "probably stable." The expiry-based re-check trigger below is
> also superseded — the draft has expired and revived three times, so
> expiry is procedural noise; see `baseline-03f` for the re-keyed triggers.

~~`[INFERENCE]` **This is an editorial objection, not a design objection**, and
the two have opposite implications for adoption. A design objection would mean
the wire format may change and adopting it is risky. An editorial objection
means the specification is hard to read but what it specifies is probably
stable. draft-11 landed four months after the review, consistent with the
editorial pass Pardue asked for.~~

A summary of the draft history characterized this as *"substantive challenges"*
with *"no clear resolution pathway."* Reading the review itself does not support
that characterization, and the leaf prompt's instruction to read the review
rather than its summary is what surfaced the difference.

### What this means for `OP-010`

The risk is **not** that the header format is wrong. The risk is that the draft
**expires unadvanced on 2026-11-24**, leaving the standard citing a dead
document — exactly the failure `baseline-02` documented for the Idempotency-Key
draft.

> **Superseded 2026-08-09 by `baseline-03f` and the ratified `OP-010`
> decision.** The expiry-keyed contingency below was re-keyed: the draft has
> expired and revived three times (RFC 2026 §2.2 restarts the clock on any
> revision), so expiry alone must not trigger withdrawal. The ratified
> triggers are IANA registration (upgrade), wire-syntax change (re-pin), and
> 18-month sustained abandonment (withdraw), reviewed semi-annually — next
> 2027-02-09. 2026-11-24 remains only a check for whether a draft-12
> appeared.

~~`OP-010` should therefore carry an explicit contingency: if the draft expires
without renewal or advancement, fall back to a documented proprietary scheme
and stop citing the draft. **This is a calendar item, not a research item.**~~

~~**Re-check trigger: 2026-11-24.**~~

---

## Sources

- https://blog.cloudflare.com/verified-bots-with-cryptography/
- https://github.com/cloudflare/web-bot-auth
- https://datatracker.ietf.org/doc/rfc9421/
- https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/history/
- https://datatracker.ietf.org/doc/review-ietf-httpapi-ratelimit-headers-10-httpdir-early-pardue-2026-01-16
- https://lists.w3.org/Archives/Public/ietf-http-wg/2026JanMar/0017.html
- https://github.com/yaronf/httpsign · https://github.com/pyauth/http-message-signatures
