Companion leaf to `baseline-03c` (threat model): establish what NEWER
webhook implementations (post-2023, big providers and major independents)
actually use for signing, and compare the candidate schemes, to drive
`OP-016`'s mechanism recommendation at Gate C.

Context: `survey-07` documented the 8 legacy references and noted the
Standard Webhooks spec with OpenAI/Anthropic/Gemini as claimed adopters,
unverified. The current proposal reads "Prefer RFC 9421; otherwise
HMAC-SHA256 over delivery ID + timestamp + raw body" and does not mention
Standard Webhooks at all. RFC 9421's only deployment evidence so far is
Cloudflare's Web Bot Auth — inbound bot auth, not webhooks.

Questions:

1. Standard Webhooks: governance (who maintains it, spec version, activity)
   and adoption depth — verify every claimed adopter against their OWN docs
   (full-spec vs variant vs delivered-by-Svix), and find more 2023+
   adopters.
2. RFC 9421 for webhooks specifically: any production outbound-webhook
   deployments, IETF drafts profiling it, whether the Web Bot Auth
   trajectory extends to webhooks, consumer-side tooling. Report absences
   honestly.
3. What brand-new (2024–2026) webhook implementations chose, including
   webhook-as-a-service platforms (Svix, Hookdeck, Convoy) that set defaults
   for downstream companies.
4. Comparative assessment of the three candidates — RFC 9421, Standard
   Webhooks, bespoke Stripe-style HMAC — across security properties,
   consumer burden, library ecosystem, interop leverage, governance risk,
   and migration paths between them.
5. Relationship between Standard Webhooks and RFC 9421: has either camp
   addressed the other; any convergence plan.

Evidence rules as in `CONSTRAINTS.md`: 2+ sources per material claim,
primary preferred; never accept a claimed adoption without checking the
adopter's own docs; label and date claims; verbatim quotes for load-bearing
text; report absences explicitly.

Output: findings per question; a verified adopter table; the comparative
table; then "Recommended shape for OP-016" — which candidate(s) to name, in
what precedence, with proposed rule wording and per-claim confidence.
