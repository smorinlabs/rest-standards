Determine the real-world standing of the two `baseline-03` principles that
deliberately diverge from universal vendor practice: `OP-016` (prefer RFC 9421
HTTP Message Signatures for webhooks) and `OP-010` (emit the IETF `RateLimit`
header fields). Both were adopted against zero adoption among the eight
surveyed references, and both carry documented fallbacks.

Exact scope: production deployments and library support for RFC 9421; and the
standardization trajectory of `draft-ietf-httpapi-ratelimit-headers` ahead of
its 2026-11-24 expiry.

Research requirements:

1. For RFC 9421, separate **the mechanism working at scale** from **the
   mechanism being used for webhook signing**. A large deployment in an
   adjacent use case proves implementability but does not prove that webhook
   consumers can verify signatures. Do not let the first stand in for the
   second.
2. Identify the use case of any production deployment precisely. Record who
   deployed it, on what date, at what scale, and for what purpose, from the
   deploying vendor's own announcement.
3. For the RateLimit draft, obtain the full revision history and any directorate
   or working-group review. Where a review returns a negative verdict, **read
   the review itself** — distinguish an editorial objection from a design
   objection, because the two have opposite implications for adopting the draft.
4. Assess trajectory honestly: elapsed time since the work began, interval
   between recent revisions, and whether Last Call or IESG submission has
   occurred.
5. Verify every material claim against at least two independent sources,
   preferring IETF Datatracker and the deploying vendor's own engineering blog.

Output: a deployment-evidence table for RFC 9421 distinguishing use cases; a
trajectory assessment for the RateLimit draft including the substance of any
review verdict; and explicit verdicts on whether `OP-016` and `OP-010` should
change strength or confidence going into Gate C.
