Survey what the industry actually emits for rate-limit communication,
before ratifying `OP-010` (IETF `RateLimit`/`RateLimit-Policy` fields) at
Gate C. Commissioned by the project owner alongside `baseline-03f` (draft
trajectory): "see what's actually adopted in industry, and if it's this
convention or another one."

Populations, each verified against the vendor's own docs:

1. The modern AI platforms — OpenAI, Anthropic, Google Gemini — mandatory
   in this and all future vendor evaluations: exact header names, reset
   semantics, scope dimensions (requests vs tokens, per-model, org/project
   tiers), 429 behavior and `Retry-After`.
2. The legacy 8: confirm current header sets from primary docs; note format
   divergences precisely (epoch vs delta reset, casing).
3. Newer/dev-tools APIs (2023+): Vercel, Linear, Notion, Slack, Discord,
   Cloudflare API, Supabase, Resend, Polar, xAI, Mistral, Cohere, Groq,
   Together AI — any adopting `RateLimit`/`RateLimit-Policy`?
4. Gateways and infrastructure (the default-setters): Kong, Envoy, Apigee,
   AWS API Gateway, Azure API Management, nginx, Traefik, Tyk, Zuplo,
   HAProxy, Cloudflare's own products — native IETF-field support, and
   which draft revision.
5. The de-facto convention characterized: is there a dominant
   `X-RateLimit-{Limit,Remaining,Reset}` trio; its known ambiguities; does
   anyone document their headers as following the IETF draft.

Critical distinction to resolve per implementation: the draft renamed its
fields mid-life (trio → combined structured fields at draft-07), so every
adoption claim must be pinned to a revision and a verbatim header line.

Evidence rules as in `CONSTRAINTS.md`: 2+ sources per material claim,
primary preferred; label and date claims; verbatim header names and example
values; report absences explicitly.

Output: a comprehensive header table (provider | headers | reset semantics |
scope | Retry-After | IETF? | category | source); findings per population;
a closing "Convention landscape" section quantifying the categories and
stating what the wild-adoption picture implies for mandating the fields.
