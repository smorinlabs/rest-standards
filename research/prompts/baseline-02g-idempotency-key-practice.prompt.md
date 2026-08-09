Before completing `AC-016`'s placement fork (header vs body field) at Gate
C, establish what OpenAI, Anthropic, and Google (Gemini and Cloud/AIP)
actually do for idempotency keys — the modern-AI-vendor evaluation the
project requires in every vendor-practice question.

Questions, each verified against the vendor's own docs and SDK source:

1. OpenAI: any idempotency mechanism in the published OpenAPI spec, API
   reference, or production-best-practices docs? What is
   `X-Client-Request-Id` for?
2. Anthropic: any mechanism? What are `request-id` and the Batches
   `custom_id` for? What do the SDKs do on automatic retries?
3. Google Gemini: any mechanism in the discovery document?
4. Google Cloud / AIP-155: current state of `request_id` — field or
   header, exact semantics (UUID, fingerprinting, retention), still active
   guidance, any movement toward headers? How does it transcode over REST?
5. The Stainless SDK layer both AI vendors use: does it ship idempotency
   machinery, is it enabled, and what wire shape would it use if enabled?

Evidence rules: primary sources only (published specs, discovery docs,
AIP repo, SDK source); label `[FACT]`/`[INFERENCE]`; verbatim quotes; date
findings; report absences explicitly.

Output: per surface — mechanism (header/body/query/none), exact name,
semantics, sources; then a net paragraph on which placement the modern AI
vendors' practice supports, if any.
