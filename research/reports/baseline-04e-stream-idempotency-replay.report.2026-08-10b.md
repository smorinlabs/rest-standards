# Baseline 04e — Idempotency-key replay of a streaming request (run **b**)

*Research leaf under `baseline-04` (streaming), answering the fourth row of
`rest-api-standard.md` §13.4 "Known unresolved interactions": **what an
`Idempotency-Key` replay of a streaming request delivers, and whether it may
re-execute the underlying work.** Series `baseline` = prescriptive: this report
**proposes** rule text; it does not ratify it. Ratification is a `decisions/`
record.*

*Second run, 2026-08-10 (suffix `b`). The first run of the same date is
retained unchanged at
`research/reports/baseline-04e-stream-idempotency-replay.report.2026-08-10.md`.
All web access dates are 2026-08-10 unless a row states otherwise. The ratified
record `research/decisions/baseline-04-streaming.decision.md` is treated as
settled and is not reopened.*

*Rules touched: `R3.9` (idempotency keys), `R12.10` (client obligations on a
truncated stream), `R13.2` (streamed/non-streamed negotiation on `Accept`),
`R13.5`/`R13.6` (frame typing and terminal frames), `R13.7` (post-commit error
frames), `R13.9` (stream ↔ operation-resource identity), `R13.10`
(resumption), `R10.9` (`202` operation discovery).*

---

## 0. This run supersedes the first run's central negative

**Read this section before relying on anything in the first run.**

The first run's stated "load-bearing claim" was a universal negative: that no
published API both accepts an idempotency key and streams, and that `R3.9` plus
§13 therefore "manufactures an intersection with zero published precedent."

**That negative is false, and this run withdraws it.** `[FACT]` Replicate's Cog
HTTP API documents both capabilities on one endpoint, in a capability table
whose row reads `PUT /predictions/<prediction_id>` with
`Accept: text/event-stream` → "Streaming, idempotent", and describes that
endpoint as "the idempotent version of the `POST /predictions` endpoint"
(https://raw.githubusercontent.com/replicate/cog/main/docs/http.md, pinned at
commit `966752e9f5f5c165fc5e9618642fd353f0db0e56`, file last revised
2026-07-06; accessed 2026-08-10).

**What follows from the withdrawal, item by item:**

| First-run claim | Status after this run |
| --- | --- |
| "No reference in the field ships the combination" (§1, §3.3 `Both?` column empty in fourteen rows) | **Withdrawn.** The column was empty because the candidate set excluded the software layer where the combination lives. |
| "Nobody has solved this because nobody has built it. That is the reason the standard must rule it rather than cite it." | **Withdrawn.** It has been built. The standard should now rule it *with* a citable implementation, which changes the rule's shape. |
| `ST-E01` clause 1 — a keyed repeat arriving while the original is still executing MUST return `409 Conflict` | **Falsified as a MUST.** The only shipped implementation of the intersection does the opposite: it attaches the caller to the in-flight stream. Revised in §7 to a documented choice among defined behaviors. |
| `ST-E01` clause 2 — a repeat after a terminal state MUST NOT re-execute and MUST deliver the recorded outcome | **Survives, and is now the rule's strongest clause** — but see §5.8: Cog *does* re-execute here, because it retains nothing after terminal, which is evidence about cost rather than about desirability. |
| `ST-E01` clause 3 — documented behavior for an abandoned execution | **Survives unchanged.** Still undocumented everywhere. |
| The `R3.9` "stored response" amendment | **Survives, and is corroborated** (§8). Cog's own behavior instantiates the outcome-not-frames reading. |
| The ≥24 h floor versus the field's minutes-long artifacts | **Survives and strengthens** (§8). Cog's buffer is not merely shorter — it is denominated in *events*, not time. |
| `ST-E02` / `ST-027` — prefer splitting execution from delivery | **Survives as one option, but is no longer the only structural answer** (§7.4). Cog demonstrates a second one. |

**What is built on the falsified negative, and therefore needs revisiting.**
`ST-026` in `docs/reviews/2026-08-10-phase-8-consolidated-proposals.md` (target:
a new `R13.15`) carries the negative as its "Why this is not over-specification"
justification and the `409` as its highest-confidence clause. That entry is
already annotated **NOT RATIFIABLE AS WRITTEN** in the consolidated document.
This report supplies its replacement (§7). **No other file is edited by this
run.** Nothing in `rest-api-standard.md` depends on the negative, because §13.4
records this interaction as *unresolved* rather than resolved — the standard's
released text is unaffected either way.

**One important qualification, developed in §5.6.** The falsifying instance
sits inside a carve-out `R3.9` already grants. `R3.9`'s obligation is the
`Idempotency-Key` **header** on non-idempotent requests, and its stated
exception is "naturally idempotent operations (PUT with a client-supplied ID)"
— which is exactly Cog's design. So the negative is false **as the first run
stated it**, and the first run stated it about "an idempotency key," not about
the header. The narrower negative — that no API carries an `Idempotency-Key`
*header* on a streaming endpoint — was **not** falsified by this run and is
restated with its remaining scope in §4.8. Both facts matter, and the report
keeps them apart rather than collapsing them in either direction.

**A second, independent weakening of the same claim.** `[FACT]` The Agent2Agent
(A2A) Protocol specification v1.0.0 places a required client-minted
`message_id`, a deduplication allowance ("Send Message operations MAY be
idempotent. Agents may utilize the messageId to detect duplicate messages"), and
an SSE response on the single operation `POST /message:stream` (§4.3). `[INFERENCE]`
Even setting Cog aside, a released specification contemplates the combination,
so "an intersection with zero published precedent" was never sustainable.

---

## 1. TL;DR and revised recommendation

**The finding that replaces run a's negative.** `[COMPARATIVE]` As of
2026-08-10, Replicate's **Cog** is the only surveyed implementation where one
endpoint both treats a client-supplied identifier as a deduplication key and
delivers incremental output on that same request. A repeat arriving while the
original runs **attaches to the in-flight stream**, replaying a bounded buffer
and then continuing live. The **A2A Protocol v1.0.0** specifies the same
combination on `POST /message:stream` but leaves deduplication at `MAY`.
Client-supplied deduplication keys are otherwise common — Step Functions, Azure
Durable Functions, Temporal, Orkes, Inngest, Trigger.dev, ComfyUI, AIP-155 — and
appear **exclusively on non-streaming endpoints**, while Google's own guidance
forbids the pairing outright for long-running-operation methods.

**So the shape of the gap changed, not its existence.** `[INFERENCE]` Run a said
the standard must rule this interaction because nobody had built it. The better
argument, and the one this run makes, is that **two independent specifications
have now reached the same gap and left it open** — this standard at §13.4, and
A2A at `MAY` — while the one shipped implementation resolves it in a way the
first run's proposed rule would have forbidden.

**The revised recommendation, in three moves.**

1. **Keep the invariant, drop the mandated status code.** A keyed repeat MUST
   NOT start a second execution while the identifier is inside its stated
   window — now supported by seven sources including the streaming one. But the
   `409` MUST becomes a documented choice between **serving the in-flight work**
   (Cog; Temporal's `USE_EXISTING`) and **rejecting with `409`** (Stripe;
   Shopify). Two independent grounds force this, and either alone would suffice:
   the only implementation at the intersection attaches, and the expired IETF
   draft everyone cites says `409` at **SHOULD**, twice — run a escalated it to
   a MUST.
2. **Add the obligation nobody has written down: an attached stream must not
   hide a gap.** `[FACT]` Cog in one shipped configuration delivers a terminal
   frame with every intermediate frame missing and no error. `[INFERENCE]`
   `R12.10`'s only integrity test is that a *missing terminal frame* means
   truncation, so a conforming client reads that response as a complete result.
   Truncation is a missing suffix; attachment loss is a missing prefix, and no
   rule in §12 or §13 reaches it. This is run b's genuinely new contribution.
3. **Fix `R3.9`'s exception, which is where the falsifying instance actually
   lives.** `[FACT]` `R3.9` exempts "naturally idempotent operations (PUT with a
   client-supplied ID)" — which is exactly Cog's design — and the exemption is
   written as a blanket exit from the whole rule. `[INFERENCE]` So an API taking
   that route inherits no payload-fingerprint duty and no retention duty, and
   Cog demonstrates the result: it compares no payload, so a repeat carrying
   *different* input is silently served the original's work. The exception should
   exempt the **header**, not the **guarantees**, and only for genuine
   state-replacement `PUT` rather than for execution-shaped `PUT`.

**Recommendation on the second structural route.** `[COMPARATIVE]` Splitting
execution from delivery — mutation returns an operation resource, stream is a
safe `GET` — is retained at SHOULD and is now much better evidenced: run a had
one example (OpenAI), run b has five, three of which carry an idempotency key on
the mutation half (ComfyUI, Trigger.dev, Inngest). Cog demonstrates a second
route — name the work in the request target — and `ST-E02b` names both.

**Confidence.** High on the non-re-execution invariant and on the falsification
itself; moderate-high on the documented-menu shape, the gap-disclosure clause,
and the `R3.9` exception split; moderate on the abandoned-execution clause and
on the boundary test between state-replacement and execution-shaped `PUT`.

**What this run explicitly does not do.** `[POLICY]` It does not promote
attachment to a MUST. One implementation is an existence proof, not a practice —
and Replicate's own hosted API does not expose it (§4.6). Inverting run a's
error would be repeating it.

---

## 2. What the first run got wrong, and why the search missed it

`[FACT]` The first run's own §8.6 conceded the bound: its negatives "rest on the
specific pages and machine-readable specifications listed in §2," and it
inherited `baseline-02g`'s caveat that absences come "from published
specifications and fetched pages rather than exhaustive site crawls." The
concession was accurate. The error was promoting a bounded negative to a
universal one, and then making it load-bearing.

**Four mechanisms produced the miss, in decreasing order of contribution.**

1. **The candidate set was a set of *businesses*, not a set of *software
   layers*.** `[INFERENCE]` Every one of the fourteen rows in the first run's
   §3.3 is a hosted commercial API or a corporate design guideline. Cog is
   neither: it is the open-source HTTP server that Replicate's model containers
   serve, shipped as a Docker image that anyone can run. An API-design survey
   scoped to vendors' *public product* documentation cannot see it, because the
   intersection lives in the serving layer underneath the product.

2. **The question was asked as "who has an `Idempotency-Key` header?" rather
   than "who deduplicates a client-supplied identifier?"** `[INFERENCE]` The
   first run searched for the Stripe-shaped mechanism it already knew. Cog's
   identifier is a path segment in a `PUT`, and the word "idempotent" appears in
   its docs attached to an endpoint, not to a header. A header-name search
   returns nothing on that page.

3. **Recency.** `[FACT]` The SSE capability landed in Cog on 2026-05-22 in pull
   request #3019, "feat: Expose prediction SSE streams"
   (https://github.com/replicate/cog/pull/3019, merged 2026-05-22T19:50:57Z,
   accessed 2026-08-10) — roughly eleven weeks before the first run. The
   idempotent `PUT` predates it; the *intersection* is new.

4. **Replicate was already inside this project's evidence base, on a different
   axis.** `[FACT]` `baseline-03d` (2026-08-09) cites
   `replicate.com/docs/topics/webhooks/verify-webhook` and lists Replicate among
   verified Standard-Webhooks implementers. `[INFERENCE]` The vendor was
   reachable and had already been read once, for webhooks. Nothing carried that
   familiarity across to the streaming question, because the two leaves searched
   by topic and never by vendor.

**The transferable lesson, stated as method rather than blame.** `[POLICY]` A
universal negative over "the field" is only as strong as the definition of the
field, and this project's default field — eight commercial references plus
three AI providers plus the standards bodies — systematically excludes
self-hosted and open-source infrastructure. Where a future leaf's conclusion
*depends* on a negative, the negative should be stated with its scope attached
("no *commercial hosted API in the reference set*…"), and at least one search
should be run against the mechanism rather than the mechanism's usual name.

---

## 3. Standards-and-currency matrix

Authority classes: **A** = published IETF standards-track RFC or Internet
Standard; **B** = expired or in-flight Internet-Draft, or a non-IETF committee
specification; **C** = vendor or consortium protocol specification; **D** =
vendor API-design guideline; **E** = shipped API documentation or implementation
source; **F** = engineering design document with no standards status.

**Sources carried forward from run a and re-verified in run b:**

| Source | URL | Class | Published / revised | Re-verified | Bears on |
| --- | --- | --- | --- | --- | --- |
| RFC 9110, *HTTP Semantics* §9.2.2, §9.3.4, §6.1 | https://www.rfc-editor.org/rfc/rfc9110.txt | **A** — Internet Standard, STD 97 | June 2022 | 2026-08-10, full text fetched | Retry safety; the definition of `PUT`; message completeness |
| `draft-ietf-httpapi-idempotency-key-header-07` | https://www.ietf.org/archive/id/draft-ietf-httpapi-idempotency-key-header-07.txt | **B** — **EXPIRED** Internet-Draft, never an RFC | Rev. 07 published 2025-10-15; header reads "Expires 18 April 2026" | 2026-08-10, full text fetched | §2.6 concurrent request; the `409` is **SHOULD**, twice |
| Stripe, *Idempotent requests* | https://docs.stripe.com/api/idempotent_requests | **E** | Living documentation | 2026-08-10, re-fetched | ≥24 h then pruned; **"We generate a new request if a key is reused after the original is pruned"**; parameter comparison; no streaming |
| OpenAI, *Background mode* | https://developers.openai.com/api/docs/guides/background | **E** | Living documentation | 2026-08-10, re-fetched | "roughly 10 minutes"; `starting_after` resumption; **no** idempotency or dedup statement |
| RFC 9457 §3.1 · OASIS *Repeatable Requests* v1.0 CS01 · MCP *Transports* rev. 2025-06-18 · gRPC A6 · Google AIP-151/AIP-155 · Azure guidelines · Zalando rule 230 · Shopify idempotency pages · AWS Bedrock `InvokeModelWithResponseStream` | as listed in run a §2 | **A**/**B**/**C**/**D**/**E**/**F** | as listed in run a | Carried forward from run a, 2026-08-10; not re-fetched in run b | Unchanged; run b relies on run a's readings and does not restate them |

**Sources new in run b:**

| Source | URL | Class | Version | Accessed | Bears on |
| --- | --- | --- | --- | --- | --- |
| Cog, *HTTP API* documentation | https://raw.githubusercontent.com/replicate/cog/main/docs/http.md | **E** | commit `966752e9f5f5c165fc5e9618642fd353f0db0e56`, file revised 2026-07-06 | 2026-08-10 | The falsifying evidence; attach-on-repeat; replay buffer |
| Cog, implementation source (`routes.rs`, `prediction.rs`, `service.rs`) | https://github.com/replicate/cog/tree/e456e1924c6051d22008761f4457a008a5f5d7b7/crates/coglet/src | **E** | HEAD `e456e192…`, 2026-08-10 | 2026-08-10 | No payload fingerprint; retention ends at terminal state; the `409` is capacity |
| Cog, integration tests (`sse_stream_history_capacity.txtar`, `…_disabled.txtar`) | https://github.com/replicate/cog/tree/e456e1924c6051d22008761f4457a008a5f5d7b7/integration-tests/tests | **E** | same HEAD | 2026-08-10 | Buffer-exhaustion error; the **silent-loss** configuration |
| Cog, pull request #3019 | https://github.com/replicate/cog/pull/3019 | **E** | merged 2026-05-22T19:50:57Z | 2026-08-10 | Design intent; dates the intersection |
| Replicate hosted API, OpenAPI document | https://api.replicate.com/openapi.json | **E** | `info.version` `1.0.0-a1` | 2026-08-10 | **Verified negative** — zero `idempot`; no `PUT` on `/predictions/{id}` |
| A2A Protocol specification and `a2a.proto` | https://raw.githubusercontent.com/a2aproject/A2A/main/docs/specification.md · https://raw.githubusercontent.com/a2aproject/A2A/main/specification/a2a.proto | **C** — consortium protocol specification, not IETF | Latest released version **1.0.0** | 2026-08-10 | Client-minted required `message_id`; dedup at **MAY**; SSE on `POST /message:stream`; `subscribe` refuses terminal tasks |
| Temporal, Workflow Id and Run Id | https://docs.temporal.io/workflow-execution/workflowid-runid | **E** | Living documentation | 2026-08-10 | `WorkflowIdConflictPolicy` — Fail / Use Existing / Terminate Existing |
| ComfyUI Cloud API v2, OpenAPI document | https://docs.comfy.org/openapi-v2.yaml | **E** | v2, auto-synced from the service repository | 2026-08-10 | `Idempotency-Key` single-use, `422 idempotency_key_reuse`, 24 h; SSE on a separate `GET` |
| Trigger.dev, *Idempotency* | https://trigger.dev/docs/idempotency | **E** | Living documentation | 2026-08-10 | Repeat returns "the original run's handle"; 30-day default key TTL |
| Cloudflare AI Gateway, caching and troubleshooting | https://developers.cloudflare.com/ai-gateway/features/caching/ · https://developers.cloudflare.com/ai-gateway/reference/troubleshooting/ | **E** | Living documentation | 2026-08-10 | `cf-aig-cache-key` is cacheability control; "Streaming responses are not cached by default" |
| AWS Bedrock AgentCore service model and botocore handlers | https://github.com/boto/botocore (service model `bedrock-agentcore`, `botocore/handlers.py`) | **E** | fetched 2026-08-10 | 2026-08-10 | The **false-falsifier** methodology warning (§4.5) |
| vLLM Responses API source (`api_router.py`, `protocol.py`) | https://github.com/vllm-project/vllm/tree/main/vllm/entrypoints/openai/responses | **E** | `main`, fetched 2026-08-10 | 2026-08-10 | Client-settable `request_id` becomes the response `id`; `GET …?stream&starting_after` |
| Triton *generate* extension · KServe Open Inference Protocol v2 | https://raw.githubusercontent.com/triton-inference-server/server/main/docs/protocol/extension_generate.md · https://raw.githubusercontent.com/kserve/open-inference-protocol/main/specification/protocol/inference_rest.md | **C** | `main`, fetched 2026-08-10 | 2026-08-10 | The canonical correlation-ID sentence; SSE on `generate_stream` |
| TensorRT-LLM Triton backend, README | https://raw.githubusercontent.com/triton-inference-server/tensorrtllm_backend/main/README.md | **E** | `main`, fetched 2026-08-10 | 2026-08-10 | A keyed repeat means **cancel**, not deduplicate |
| BentoML async task queues | https://docs.bentoml.com/en/latest/get-started/async-task-queues.html | **E** | Living documentation | 2026-08-10 | Server-generated task id; split design; no key |

**Currency warnings, restated.** `[FACT]` The IETF `Idempotency-Key` draft is
expired and was never an RFC; it MUST NOT be cited as a standard, exactly as
`R3.9` already states — and run b adds that even at its most complete it says
`409` at **SHOULD**. `[FACT]` Cog is read at a moving `main`; every claim is
pinned to a commit. `[FACT]` Implementation source is evidence of behavior only,
never of protocol authority.

---

## 4. The intersection implementations found

### 4.1 How the search was run, so the negatives have a stated scope

`[POLICY]` Run a's error was an unscoped negative, so run b states its scope
before its results. Three sweeps were run over disjoint candidate sets — (i)
self-hosted and open-source model-serving APIs; (ii) LLM gateways and proxies,
workflow and job-runner APIs, and event-sourcing HTTP APIs; (iii) a re-check of
the three AI providers plus durable-execution and serverless-inference hosts.
Each candidate was checked against a named URL, and the decisive question was
asked as a **mechanism** question rather than a header-name question:

> Is the identifier **client-supplied**, and does a repeat carrying it
> **prevent a second execution**?

`[POLICY]` Two distinctions did most of the work and are applied throughout the
taxonomy. A **correlation identifier** is echoed back and never consulted;
it is not the intersection. A **cached response** replayed by a gateway is
caching, not idempotency; it is recorded separately where found.

### 4.2 The taxonomy

`[COMPARATIVE]` **One implementation of the intersection was found: Cog.** The
remaining candidates fall into four other classes, and the classes themselves
are the finding — the field has several defined answers to "what does a repeated
client-supplied identifier mean on a streaming endpoint," and only one of them
is deduplication.

| Behavior on a keyed repeat of a streaming request | Who | Evidence |
| --- | --- | --- |
| **Attach to the running stream** — replay the retained prefix, then continue live; no second execution | **Cog** (`PUT /predictions/{id}` + `Accept: text/event-stream`) | "the server returns a stream for the existing prediction instead of creating a duplicate prediction"; `create_prediction_idempotent` in `routes.rs` |
| **Serve a non-streamed representation of the running work** — same endpoint, non-streaming repeat | **Cog** (`PUT /predictions/{id}` without SSE) | `202 Accepted` plus the current prediction state; "the client receives the response type requested by the retry" |
| **Reject with a conflict status** | **Stripe** (`409`, `idempotency_key_in_use`); **Shopify** (`IDEMPOTENCY_CONCURRENT_REQUEST`); **expired IETF draft** §2.6 (`409`, at **SHOULD**) | All three are **non-streaming** APIs; none has ever answered this for a stream |
| **Cancel the original** — the repeat is a control message, not a retry | **TensorRT-LLM Triton backend** | "Send a second request with `\"stop\": true` and the same `request_id` you used in the original request" |
| **Reject any reuse of the key outright** — not only while executing | **ComfyUI Cloud API v2** (`POST /api/v2/jobs`, `Idempotency-Key`) | "Single-use: the first request to present a key is processed; any later request with the same key is rejected `422` `idempotency_key_reuse` (reject-on-duplicate, no response replay). Keys expire after 24h." |
| **Attach permitted but not required** — a released specification contemplating the combination | **A2A Protocol v1.0.0** (`POST /message:stream`) | Dedup at **MAY**; see §4.3 |
| **Undefined — the identifier is correlation only** | vLLM, LocalAI, Triton core, KServe, Ray Serve, Ollama (no identifier at all), TGI (none), OpenLLM (none), Helicone, Portkey, Modal | KServe/Triton canonical wording: "An identifier for this request. Optional, but if specified this identifier must be returned in the response" |
| **Cached, not deduplicated** — a gateway replaying a body, and excluding streams from it | Cloudflare AI Gateway; LiteLLM, Kong, Portkey, Helicone (caching present, stream interaction undocumented) | Cloudflare, verbatim: "Streaming responses are not cached by default" |

### 4.3 A2A Protocol v1.0.0 — a released specification that contemplates the combination

**This is the second most important finding in the report, and run a had no
equivalent.** `[FACT]` The Agent2Agent (A2A) Protocol specification v1.0.0
places all three properties on one operation, verified directly against the
specification repository on 2026-08-10:

- **A client-minted, required identifier.** `specification/a2a.proto`: "The
  unique identifier (e.g. UUID) of the message. This is created by the message
  creator." — `string message_id = 1 [(google.api.field_behavior) = REQUIRED];`
  (https://raw.githubusercontent.com/a2aproject/A2A/main/specification/a2a.proto).
  The neighbouring `task_id` is by contrast server-generated.
- **A streaming operation.** `POST /message:stream` — "Send message with
  streaming (SSE response)"
  (https://raw.githubusercontent.com/a2aproject/A2A/main/docs/specification.md,
  §11.3.1).
- **Deduplication, at MAY.** §3.3.1, verbatim: "**Send Message** operations MAY
  be idempotent. Agents may utilize the messageId to detect duplicate messages."

`[INFERENCE]` **A2A is not a second implementation of the intersection; it is
something arguably more useful — a standards-shaped document that permits it and
declines to specify it.** Because deduplication is `MAY`, a conforming agent may
re-execute, and a client cannot rely on either behavior. That is precisely the
under-specification `R3.9` plus §13 produces, reached independently by another
specification body. `[POLICY]` It is therefore evidence *for* ruling the
interaction rather than against: run a argued the standard must rule it because
nobody had built it; the better argument is that a second specification has
reached the same gap and left it open.

`[FACT]` **A2A also ships a separate attach channel with a documented terminal
answer.** `POST /tasks/{id}:subscribe` — "Subscribe to task updates (SSE
response, returns error for terminal tasks)"; §10.4.6 states it returns
`UnsupportedOperationError` if the task is in a terminal state.

`[INFERENCE]` **This is a genuine conflict with `ST-E01b` clause 2 and is
surfaced rather than averaged.** `R3.9` requires a repeat to receive the stored
response once the work is done; A2A's attach channel *refuses* a terminal task
outright. `[POLICY]` The two are reconcilable and this report treats them as
addressing different questions: A2A's `subscribe` is a *subscription to future
updates*, which is genuinely meaningless once there are none, whereas `R3.9`'s
replay is a *delivery of a recorded outcome*, which remains meaningful. The
lesson carried into §7 is that an attach channel and a replay channel are not
the same thing, and an API that offers only the first has not discharged `R3.9`.

### 4.4 Temporal — the attach answer, shipped, without the stream

`[FACT]` Temporal's `WorkflowIdConflictPolicy` makes the in-flight answer an
explicit, caller-selected policy with three values
(https://docs.temporal.io/workflow-execution/workflowid-runid, accessed
2026-08-10):

| Policy | Documented behavior |
| --- | --- |
| **Fail** | "Prevents the Workflow Execution from spawning and returns a `Workflow execution already started` error. **This is the default policy, if one isn't specified.**" |
| **Use Existing** | "Prevents the Workflow Execution from spawning and returns a successful response with the Open Workflow Execution's Run Id." |
| **Terminate Existing** | "Terminates the Open Workflow Execution then spawns the new Workflow Execution with the same Workflow Id." |

`[FACT]` The identifier is client-supplied in the request target, exactly Cog's
shape: `POST /namespaces/{namespace}/workflows/{workflow_id}`. `[FACT]` Temporal
does not stream — its HTTP surface exposes no server-streaming operation, and
`GetWorkflowExecutionHistory`'s `wait_new_event` is a long poll returning one
page.

`[COMPARATIVE]` **So the in-flight menu of `ST-E01b` clause 1 is not invented by
this report.** Temporal ships it as a first-class, documented, per-call choice,
and its `USE_EXISTING` is the attach answer under a different name. Two
independent designs — Cog for streams, Temporal for workflows — arrive at
"return the running work rather than an error," and one of them makes the
alternative a documented policy rather than a hidden behavior. `[INFERENCE]` A
third value exists that this report does **not** adopt: `TERMINATE_EXISTING`
would be catastrophic under `R3.9`, since a retry would kill the work it was
trying to recover.

### 4.5 A false falsifier, recorded as a methodology warning

`[POLICY]` **A re-run that greps for the string `idempotencyToken` will find an
apparent second intersection that is not one.** AWS Bedrock AgentCore's
`InvokeAgentRuntime` carries `runtimeSessionId` marked `"idempotencyToken": true`
in the botocore service model, is client-supplied as the header
`X-Amzn-Bedrock-AgentCore-Runtime-Session-Id`, **and** streams. It looks like the
intersection and is not.

`[FACT]` Two independent disproofs. First, the trait is an SDK code-generation
marker, not a server contract: botocore's `generate_idempotent_uuid` handler
fills the member with a random UUID when the caller omits it — it means "the SDK
invents one if you don't," not "the server deduplicates." Second, the AWS API
reference for `InvokeAgentRuntime` contains zero occurrences of
idempotent/idempotency/duplicate/deduplication, and documents the member as
session continuation — "used to maintain state and context across **multiple**
command invocations" — which is the opposite of deduplication.

`[FACT]` The discriminator is the prose, not the trait: the same trait on
`StartAsyncInvoke.clientRequestToken` carries "Specify idempotency token to
ensure that requests are not duplicated," and `StartAsyncInvoke` does not
stream. `[POLICY]` Recorded here because run a's failure was a search-method
failure, and this is the symmetric hazard — a mechanical search producing a
false positive rather than a false negative.

### 4.6 The near misses, which are decision-relevant

`[FACT]` **vLLM implements the split design with a client-nameable identifier.**
Its Responses API accepts a body `request_id` — "The request_id related to this
request. If the caller does not set it, a random_uuid will be generated" — and
that value becomes the response object's `id`. Recovery is then
`GET /v1/responses/{response_id}` with `starting_after: int | None` and
`stream: bool | None`, returning `text/event-stream`.

`[INFERENCE]` **This matters for `ST-E02b`.** It is OpenAI's split — mutation
first, stream over a safe `GET` — but with the client naming the work rather
than the server. That is the two structures of §7.4 composed, and it shows they
are not alternatives so much as separable choices: *who names the work* and
*whether delivery is a separate safe request* are independent axes. `[FACT]`
vLLM documents **no** deduplication on a repeated `POST` carrying the same
`request_id`, so it is not the intersection; the resumption is a different
request against a stored response, gated on `store=true`.

`[FACT]` **Replicate's own hosted product does not expose Cog's idempotent
create.** Independently verified against `https://api.replicate.com/openapi.json`
(`info.version` `1.0.0-a1`, accessed 2026-08-10): zero occurrences of `idempot`;
`/predictions` offers `get` and `post` only; `/predictions/{prediction_id}`
offers `get` only — there is no `PUT`. Streaming is decoupled, reached through
the prediction object's `urls.stream`.

`[INFERENCE]` **This is the single most important qualifier on the falsification,
and it cuts both ways.** The intersection is real and shipped, so run a's
negative is dead. But the vendor that built it kept it in the serving layer and
declined to expose it in its commercial API, where it would face untrusted
clients, multi-tenancy, and billing. `[POLICY]` That is a reason to permit
attachment rather than mandate it — §9.2 — and it is evidence run a could not
have had, because run a never looked at either layer.

### 4.7 Affirmative evidence for the split design, which run a lacked

`[COMPARATIVE]` Run a recommended splitting execution from delivery on the
strength of **one** shipped example (OpenAI's `background: true`). Run b finds
the split shipped repeatedly, and — the part that matters — shipped **with an
idempotency key on the mutation half**, which is the configuration run a's
`ST-E02` assumed without evidence.

| API | The key, on the mutation | The stream, on a separate request |
| --- | --- | --- |
| **ComfyUI Cloud API v2** | `Idempotency-Key` on `POST /api/v2/jobs`; "any later request with the same key is rejected `422` `idempotency_key_reuse` (reject-on-duplicate, no response replay). Keys expire after 24h." | `GET /api/v2/jobs/{id}/events`, SSE; self-described as "Live push only — NOT a replayable log: events carry no `id`, there is no `Last-Event-ID` resume." |
| **Trigger.dev** | `idempotencyKey` (body member, 30-day default TTL); "the second request will not create a new task run. Instead, the original run's handle is returned" | SSE subscribe on a separate `GET`, with `Last-Event-ID` as a resume position |
| **Inngest** | event `id`, 24-hour window; a duplicate "will *not* trigger any functions" | Realtime, a separate WebSocket channel |
| **OpenAI** | none — the split is present without a key | `GET /v1/responses/{id}?stream=true&starting_after=N` |
| **vLLM** | none — but the identifier is client-nameable (§4.6) | `GET /v1/responses/{response_id}` with `starting_after`, `stream` |

`[FACT]` **ComfyUI is the sharpest of these because it answers `R3.9` the
opposite way and says so.** Independently verified against
https://docs.comfy.org/openapi-v2.yaml (HTTP 200, accessed 2026-08-10), the
specification states it twice. On the `Idempotency-Key` parameter: "Client-generated
UUID (recommended). Single-use: the first request to present a key is processed;
any later request with the same key is rejected `422` `idempotency_key_reuse`
(reject-on-duplicate, no response replay). Keys expire after 24h." And in the
operation description, more explicitly still:

> "`Idempotency-Key` is single-use (reject-on-duplicate, **NOT
> record-and-replay**): the first request to present a given key is processed
> normally; ANY later request presenting the same key — a retry, a concurrent
> duplicate, or a same-key request with a different body — is rejected `422`
> `idempotency_key_reuse` and is **never re-executed**."

`[INFERENCE]` **Three things follow, and the third is the most useful.** First,
this is a fourth distinct answer to the in-flight question and an outright
rejection of `R3.9`'s replay model — replay is not the only defensible reading
of an idempotency key, merely the one `R3.9` adopted. Second, it nonetheless
agrees with §6.1's invariant: the repeat is "never re-executed." Third, and
directly supporting §7.2: ComfyUI explicitly rejects "a same-key request with a
different body," which is `R3.9`'s reject-clause implemented on a header-carried
key — the very obligation Cog discards by moving the identifier into the URI.
`[COMPARATIVE]` So the payload-reject duty is not a theoretical nicety this
report invented; one of the two shipped `Idempotency-Key` header implementations
found in run b states it as a headline property.

`[FACT]` ComfyUI also parallels Stripe's execution-boundary rule: "The key is
claimed only for a request that actually reaches submission and is released if
that submission definitively fails without creating a job."

`[INFERENCE]` **Under every row of this table the leaf's question dissolves**,
because the recovery request is a safe `GET` (or a separate subscription), which
RFC 9110 §9.2.2 already makes idempotent. This is why `ST-E02b` keeps the split
as a recommendation at SHOULD, and it now rests on five examples rather than one.

### 4.8 The classes that produced no intersection, with their scope stated

`[FACT]` Checked and negative for the intersection, each against the URL named
in §3 or in the sweep record: Ollama (no request identifier of any kind in
`docs/api.md`), TGI (its `openapi.json` documents no headers at all; the
`x-compute-*` family is response-only and server-generated), OpenLLM, Ray Serve,
KServe, Triton core, LocalAI (`X-Correlation-ID`, named as correlation in
source), BentoML (server-generated task id, split retrieval), and Replicate
hosted.

`[FACT]` Checked and negative across the workflow, job-runner, gateway, and
event-sourcing classes: LiteLLM, Portkey, Helicone, Kong (caching, no key);
Temporal, Argo Workflows, Kubernetes, GitHub Actions, Modal, Inngest,
Trigger.dev, Hatchet, Orkes Conductor, Camunda, Azure Durable Functions, AWS
Step Functions (keys present, never on a streaming response);
EventStoreDB/KurrentDB (`Kurrent-EventId` for idempotency and `Kurrent-LongPoll`
for reads — different methods on different resources), Axon Server, Marten; tus
and the RUFH draft (the draft "Remove[d] Upload-Token and instead use[s]
Server-generated upload URL for upload identification", and in any case governs
*request-body* streaming, not response streaming).

`[FACT]` **The three AI providers re-verified clean**, closing run a's readings
rather than merely repeating them: OpenAI's published OpenAPI has exactly seven
`in: header` parameters and all seven are `openai-beta`; Anthropic's `custom_id`
is documented as a within-batch correlation label — "Useful for matching results
to requests… Must be unique for each request within the Message Batch" — not a
dedup key; the Gemini discovery document at revision **20260810** (run a saw
20260806) still has zero `idempot`, `requestId`, and `clientId`; and AWS Bedrock
Runtime's only `idempotencyToken` is `StartAsyncInvoke.clientRequestToken`,
which does not stream, while all three eventstream operations carry no client
token.

`[FACT]` **The Azure surface run a never checked is now closed.** Azure's
guidelines mandate `Repeatability-Request-ID` and Azure OpenAI streams, making
it the most likely second falsifier. Across `Azure/azure-rest-api-specs`,
`Repeatability-Request-ID` appears in 90 files and **co-occurs with
`text/event-stream` in none**, with zero `Repeatability` hits under
`path:cognitiveservices`.

`[POLICY]` **These negatives are bounded exactly as run a's were**, and are
stated as such: they hold for the pages and machine-readable specifications
named, at the commits named, on 2026-08-10. They are not claims about every
deployed API, and no conclusion in this report rests on their universality. Run
b's own phrasing throughout is "not documented as of 2026-08-10," never "does
not exist."

---

## 5. Cog design study

Cog is the closest thing the field has to a reference implementation of the
intersection, so this section reads it in detail. **Two independent artifact
classes are used and cited separately:** the published documentation, and the
implementation source plus its integration tests. Where they diverge, §5.4 says
so explicitly.

**Sources for this section.**

| Artifact | URL | Version pinned | Accessed |
| --- | --- | --- | --- |
| `docs/http.md` (documentation) | https://raw.githubusercontent.com/replicate/cog/main/docs/http.md | commit `966752e9f5f5c165fc5e9618642fd353f0db0e56`, file last revised 2026-07-06 | 2026-08-10 |
| `crates/coglet/src/transport/http/routes.rs` (HTTP routing and SSE assembly) | https://github.com/replicate/cog/blob/e456e1924c6051d22008761f4457a008a5f5d7b7/crates/coglet/src/transport/http/routes.rs | repository HEAD `e456e1924c6051d22008761f4457a008a5f5d7b7`, 2026-08-10 | 2026-08-10 |
| `crates/coglet/src/prediction.rs` (replay buffer) | https://github.com/replicate/cog/blob/e456e1924c6051d22008761f4457a008a5f5d7b7/crates/coglet/src/prediction.rs | same HEAD | 2026-08-10 |
| `crates/coglet/src/service.rs` (prediction registry and lifetime) | https://github.com/replicate/cog/blob/e456e1924c6051d22008761f4457a008a5f5d7b7/crates/coglet/src/service.rs | same HEAD | 2026-08-10 |
| `integration-tests/tests/sse_stream_history_capacity.txtar`, `…_disabled.txtar` | https://github.com/replicate/cog/tree/e456e1924c6051d22008761f4457a008a5f5d7b7/integration-tests/tests | same HEAD | 2026-08-10 |
| Pull request #3019, "feat: Expose prediction SSE streams" | https://github.com/replicate/cog/pull/3019 | merged 2026-05-22T19:50:57Z | 2026-08-10 |

Authority class for all six: **E** — shipped implementation documentation and
source. Source code is used here as evidence of *behavior*, never as protocol
authority.

### 5.1 How the prediction is identified: a URI path segment, not a header

`[FACT]` The identifier is the last path segment of the request target:
`PUT /predictions/<prediction_id>`. The docs place the responsibility on the
caller — "Clients are responsible for providing unique prediction IDs" — and
recommend "generating a UUIDv4 or UUIDv7, base32-encoding that value, and
removing padding characters," yielding "a random identifier that is 26 ASCII
characters long."

`[FACT]` The server advertises the shape in its root discovery document as
`"predictions_idempotent_url": "/predictions/{prediction_id}"`, alongside
`predictions_cancel_url`; trainings expose the parallel
`trainings_idempotent_url`.

`[FACT]` The identifier may also appear in the request body, and the **only**
body check performed is agreement with the URI. `create_prediction_idempotent`
in `routes.rs` returns `422 Unprocessable Entity` with the message "prediction
ID must match the ID supplied in the URL" when `body.id` is present and differs
from the path segment. There is no other comparison against the body.

### 5.2 What a repeat does while the original is running

`[FACT]` Documentation: "if a client calls this endpoint more than once with the
same ID (for example, due to a network interruption) while the prediction is
still running, no new prediction is created. Instead, the client receives the
response type requested by the retry: JSON for regular requests or a server-sent
event stream for streaming requests." And, for the streaming case specifically:
"If the prediction is still running, the server returns a stream for the
existing prediction instead of creating a duplicate prediction."

`[FACT]` Source, `create_prediction_idempotent`: the handler looks the ID up in
the live prediction registry. On a hit it branches on the *repeat's* negotiated
response mode — a streaming repeat is answered by subscribing to the existing
prediction's event stream; a non-streaming repeat is answered `202 Accepted`
carrying the prediction's current state snapshot. On a miss it falls through to
ordinary creation.

`[FACT]` Pull request #3019 states the intent in the same terms: "Replay
retained events for clients that reconnect to an in-flight prediction with
`PUT /predictions/{id}` and `Accept: text/event-stream`."

**Three consequences worth naming.** `[INFERENCE]`

- **The idempotency unit is the *work*, not the *response*.** The repeat may ask
  for a different representation than the original request did, and gets it.
  The integration tests exercise exactly this: the original is issued with
  `Prefer: respond-async` (a `202` JSON response) and the repeat with
  `Accept: text/event-stream` (a stream over the same running prediction).
- **This composes with content negotiation rather than fighting it.** `R13.2`
  already puts the streamed/non-streamed choice on `Accept`; Cog applies the
  same rule to a repeat. A standard that mandated "a replay delivers what the
  original delivered" would contradict its own negotiation rule.
- **There is no `409` for a keyed repeat.** Cog does return `409 Conflict`, but
  for an unrelated condition — see §5.5.

### 5.3 The replay buffer: what it retains, and for how long

`[FACT]` Retention is **denominated in events, not in time.** `prediction.rs`
defines `DEFAULT_STREAM_HISTORY_CAPACITY: usize = 1024` and reads the override
from the environment variable `COG_STREAM_HISTORY_CAPACITY`. The documentation
states it plainly: "The replay buffer keeps the most recent 1024 events by
default. Set `COG_STREAM_HISTORY_CAPACITY` to change this limit, or set it to
`0` to disable replay history while keeping live streaming enabled."

`[FACT]` The buffer is a per-prediction ring: on each emitted event, if the
history is at capacity the oldest event is dropped and a `skipped` counter is
incremented, then the new event is appended and also broadcast live. A
subscriber receives a snapshot of the retained history, the current `skipped`
count, and a live receiver.

`[FACT]` **The buffer's lifetime is the prediction's lifetime, and nothing
longer.** The prediction registry is documented in `service.rs` as the "Active
predictions — single source of truth for prediction state," and it is the only
store; `remove_prediction` deletes the entry, and `routes.rs` calls it from the
spawned task immediately after the prediction function returns. A comment in the
stream guard records the resulting race directly: "Prediction cleanup may remove
the service entry before the SSE response finishes draining."

`[FACT]` Live subscribers are separately capped: `MAX_STREAM_SUBSCRIBERS` equals
the channel capacity of 1024, and exceeding it yields `429 Too Many Requests`
with `{"error": "Too many stream subscribers"}`.

`[FACT]` Keep-alives are SSE comment lines on a 15-second interval, emitted by
the framework's `KeepAlive` with the text `keep-alive` — transport filler, not
typed frames, which is the shape `R13.5` already contemplates.

### 5.4 What happens after the buffer is exhausted — and the silent-loss case

This is the most decision-relevant part of the design, and the documentation
tells only half of it.

`[FACT]` **Buffer overflowed (capacity > 0, events dropped).** The stream emits
an `error` event and closes. Documentation: "If earlier events have been dropped
from the replay buffer, the stream emits an `error` event and closes." Source:
when the retained `skipped` count is non-zero the stream yields a single event
named `error` with payload
`{"error": "SSE stream replay truncated; events were dropped", "skipped": N}`
and terminates — the retained prefix is **not** delivered first. The integration
test `sse_stream_history_capacity.txtar` pins this behavior at
`COG_STREAM_HISTORY_CAPACITY=2`, asserting `event: error`, the message text, and
`"skipped":3`.

`[FACT]` **A separate live-lag path** produces the same shape when a connected
subscriber falls behind the broadcast channel: `error` with "SSE stream lagged;
events were dropped" and a `skipped` count, then close. A source comment records
that the fix is known and unbuilt: "In the future, this could become
backpressure or cursor-based replay."

`[FACT]` **Buffer disabled (`COG_STREAM_HISTORY_CAPACITY=0`) — loss is
silent.** With capacity zero the history block is skipped entirely, so the
`skipped` counter is never incremented and the error path never fires. The
integration test `sse_stream_history_disabled.txtar` asserts precisely this: a
repeat that attaches mid-prediction receives `event: completed` with
`"status":"succeeded"`, and the test asserts the **absence** of both
`event: output` and the truncation message. The attaching client is handed a
well-formed terminal frame and never learns that every intermediate frame was
lost.

`[INFERENCE]` **This is a gap `R12.10` cannot currently detect, and it is a
finding this leaf did not previously have.** `R12.10` gives a client exactly one
integrity test: a connection that closes *without* the documented terminal frame
is truncated. An attached stream fails the opposite way — the terminal frame
arrives, so the client concludes "complete result," while an arbitrary number of
intermediate frames were never delivered. Truncation is a missing *suffix*;
attachment produces a missing *prefix or middle*. A client that acted on frames
incrementally — the case `R13.4` says SSE exists to serve — has silently
processed a gapped stream.

`[FACT]` Cog partially mitigates this by accident of payload design rather than
by rule: `output` events carry an `index` member, and the terminal `completed`
event carries the full accumulated `output` array, so a client that compares can
notice. `[FACT]` `log` and `metric` events carry no position of any kind
(`{source, data}` and `{name, value, mode}` respectively), so gaps in those are
undetectable. `[FACT]` And **no** Cog event carries an SSE `id:` field — the
source constructs every event with a name and JSON data only — so
`Last-Event-ID` resumption is not available and replay is keyed entirely by the
request's own identity. `[INFERENCE]` This leaves `survey-08`'s verified
negative on `id:`/`Last-Event-ID` intact and gives no reason to reopen the
ratified `R13.10`.

### 5.5 The `409` Cog does return is not an idempotency conflict

`[FACT]` Cog's documented `409 Conflict` is concurrency-slot exhaustion: "By
default, the server runs one prediction at a time, but this can be increased
with `@cog.concurrent(max=N)`. When all prediction slots are in use, the server
returns `409 Conflict`." The source confirms the semantics — the only `409` in
the routing layer is raised on a `CreatePredictionError::AtCapacity` with the
body `{"error": "At capacity - all prediction slots busy", "status": "failed"}`.

`[POLICY]` **This must not be counted as field support for the Stripe-shaped
`409`.** It is a capacity signal on a *new* prediction, not a conflict signal on
a *repeated* one; a keyed repeat of a running prediction never reaches that
branch, because the registry hit returns first. The first run drew the same
distinction for Twilio's `409`, and it is preserved here.

### 5.6 How Cog measures against `R3.9`, clause by clause

This is the comparison that determines what the falsification actually costs the
standard.

| `R3.9` obligation | Cog | Verdict |
| --- | --- | --- |
| Key carried in the `Idempotency-Key` request header | Client-supplied ID as a URI path segment on a `PUT`; no `Idempotency-Key` header anywhere in the docs or source | **Different mechanism** |
| Server fingerprints the request payload | No payload comparison; only `body.id` versus URI id, else `422` | **Not implemented** |
| Rejects a reused key carrying a different payload | A repeat with the same ID and a *different* `input` attaches to the original and the new `input` is ignored | **Contradicts** |
| Replays the stored response for a genuine retry | Only while the work is in flight | **Partial** |
| Stated retention window at least 24 hours | Retention ends when the prediction reaches a terminal state; nothing is kept | **Contradicts** |
| Exception for "naturally idempotent operations (PUT with a client-supplied ID)" | Exactly this | **Cog is the exception, not the rule** |

`[INFERENCE]` **Cog is therefore not a conforming implementation of `R3.9`; it
is an implementation of the carve-out `R3.9` already grants.** Two things follow,
and they pull in opposite directions, so both are stated.

- *Against the first run:* its negative was asserted about "an idempotency key,"
  and Cog is an idempotency key by any ordinary reading — the docs call the
  endpoint idempotent, the discovery document calls the URL
  `predictions_idempotent_url`, and its purpose is stated as safe retry. The
  negative as written is false and the first run's reasoning from it collapses.
- *For the first run's conclusion, partially:* the narrower claim that no
  surveyed API carries an `Idempotency-Key` **header** on a streaming endpoint
  survives this run's search. `R3.9`'s specific mandate still has no shipped
  streaming exemplar.

`[POLICY]` The honest synthesis is that the standard's gap is real but its shape
was wrong. The field has not declined to solve the problem; it has solved it by
**moving the identifier into the request target and making the method
idempotent** — which is the one route `R3.9` explicitly blesses and then stops
describing. §7 builds the revised rule on that observation.

### 5.7 Relationship to `Prefer: respond-async`, the `202` path, and `R10.9`

`[FACT]` Cog exposes three response modes on both create endpoints, selected by
request headers: no header → synchronous JSON; `Prefer: respond-async` → `202
Accepted` with a prediction object in status `starting`;
`Accept: text/event-stream` → `200 OK` held open as an SSE stream.

`[FACT]` **The `202` path has no pollable operation resource.** The
documentation states it as a limitation: "For JSON responses, the only supported
way to receive updates on the status of predictions started asynchronously is
using webhooks. Polling for prediction status is not currently supported." There
is no `GET /predictions/<id>`.

`[INFERENCE]` **This inverts the usual relationship, and it is why Cog reached
attach-to-stream rather than the split the first run recommended.** In OpenAI's
`background: true` design the operation resource is authoritative and the stream
is a view over it, so recovery is a safe `GET`. Cog has no such resource, so the
*only* way back to a running prediction is to re-issue the idempotent `PUT` —
attachment is not a stylistic preference but the sole available channel. `[FACT]`
Cancellation is the one exception: `POST /predictions/<id>/cancel` addresses the
prediction by ID and answers `200` or `404`, which is `R10.9`'s `cancel` action
without the operation resource around it.

`[INFERENCE]` Measured against `R13.9`, Cog is out of scope rather than
non-conforming: `R13.9` binds only where a capability is exposed *both* as a
stream and as an operation resource, and Cog exposes no operation resource. It
is, however, a clean demonstration that the two channels the standard assumes
are not always both present.

### 5.8 Does Cog re-execute in any case? Yes — and the case matters

`[FACT]` After a prediction reaches a terminal state — `succeeded`, `failed`, or
`canceled` — its registry entry is removed immediately, and the registry is the
only store. A subsequent `PUT /predictions/<same-id>` therefore finds nothing,
falls through the registry check, and **creates and executes a new prediction
under the same ID.**

`[INFERENCE]` So Cog's guarantee is narrower than "idempotent" suggests, and the
documentation's own wording is careful about it in a way a reader may miss: the
promise is scoped "**while the prediction is still running**." Cog deduplicates
concurrent retries; it does not deduplicate a retry that arrives after the work
finished. `R3.9`'s ≥24-hour floor asks for the second guarantee, and Cog
provides zero seconds of it.

`[POLICY]` **This is evidence about cost, not about desirability**, and the
report declines to read it as either an endorsement of re-execution or a refutation of
`R3.9`'s floor. What it shows is that the only implementation to reach the
intersection stopped at the boundary where retention becomes an obligation
outliving the request — the same boundary §8 identifies from the other side.

---

## 6. Evidence for and against each candidate rule shape

The question splits into four independent decisions. Treating them as one rule
is what produced the first run's error: it bundled a well-evidenced invariant
with a poorly evidenced response shape and inherited the bundle's confidence
from the stronger half.

### 6.1 Shape A — "a keyed repeat MUST NOT start a second execution"

**FOR.** `[COMPARATIVE]` This is the one proposition every source agrees on,
and the falsification *strengthened* it rather than weakening it. Stripe replays
rather than re-runs; Zalando rule 230 serves from the key cache "instead of
re-executing the request"; AIP-155 returns the prior successful response; OASIS
returns the recorded response; Temporal's `USE_EXISTING` "prevents the Workflow
Execution from spawning"; Step Functions' `StartExecution` "succeeds and return
the same response as the original request"; and Cog — the only implementation at
the intersection — exists precisely so that "no new prediction is created."
Seven sources, including the one that streams.

**FOR.** `[FACT]` Even ComfyUI, which rejects rather than replays, does not
re-execute: its repeat is a `422 idempotency_key_reuse`. `[INFERENCE]` The field
divides on *what to return*, never on *whether to re-run*.

**FOR.** `[FACT]` RFC 9110 §9.2.2 tolerates an automatic `POST` retry only
"before any part of a response is received"
(https://www.rfc-editor.org/rfc/rfc9110.txt, Internet Standard STD 97, June
2022, accessed 2026-08-10). A truncated stream is by definition the case where
part of a response *was* received, so HTTP's own text declines to sanction
re-execution there.

**AGAINST.** `[FACT]` Every retention-bounded implementation re-executes
eventually, and says so. Stripe: "We generate a new request if a key is reused
after the original is pruned." Cog: the registry entry is deleted at terminal
state, so a later repeat re-executes. `[INFERENCE]` So the invariant is only
ever true *within a stated window*, and a rule that omits the window is
unimplementable as written. This is a drafting correction, not a refutation.

**Verdict: adopt, scoped to the retention window.** Confidence **high**.

### 6.2 Shape B — "while the original is executing, the server MUST return `409`"

This is the first run's highest-confidence clause and it does not survive.

**AGAINST — the implementation at the intersection does the opposite.**
`[FACT]` Cog attaches: "the server returns a stream for the existing prediction
instead of creating a duplicate prediction." `[INFERENCE]` A rule stated as a
MUST would make the field's only shipped implementation of the intersection
non-conformant on its central behavior, in favor of an answer no streaming API
has ever shipped.

**AGAINST — attachment is not unique to Cog once you look past streaming.**
`[FACT]` Temporal's `WORKFLOW_ID_CONFLICT_POLICY_USE_EXISTING` "prevents the
Workflow Execution from spawning and returns a successful response with the Open
Workflow Execution's Run Id," on a client-supplied identifier carried in the
request target. `[COMPARATIVE]` Two independent designs answer the in-flight case
by returning the running work rather than an error, so the `409` was never the
field's consensus — it was the consensus of the subset run a happened to survey.

**AGAINST — attaching is better for the client, on the standard's own
reasoning.** `[INFERENCE]` A `409` tells a client that lost its connection to
go away; attachment gives it the output it asked for. `R12.10` forbids the
client from recovering by replaying a non-idempotent request without a key, so
under a `409` rule a client whose stream dropped mid-generation has *no*
recovery at all — it must discard completed, billable work. Attachment is the
recovery `R12.10` presumes exists.

**AGAINST — the nearest-to-standard text says SHOULD, not MUST.** `[FACT]`
`draft-ietf-httpapi-idempotency-key-header-07` §2.6: "The request was retried
before the original request completed. The resource SHOULD respond with a
resource conflict error." Its error section repeats the strength: "If the
request is retried, while the original request is still being processed, the
resource SHOULD reply with an HTTP 409 status code with body containing problem
description"
(https://www.ietf.org/archive/id/draft-ietf-httpapi-idempotency-key-header-07.txt,
**expired** Internet-Draft, "Expires 18 April 2026", accessed 2026-08-10).
`[INFERENCE]` The first run escalated a SHOULD from an expired draft into a MUST
in a published standard. That is an error independent of the Cog evidence, and
it would have been visible without it.

**FOR — the three-source agreement is real, but its scope is not what was
claimed.** `[FACT]` Stripe (`409`, `idempotency_key_in_use`), Shopify
(`IDEMPOTENCY_CONCURRENT_REQUEST`), and the expired draft do agree. `[FACT]`
All three are non-streaming APIs, and Stripe's own rationale is retention-shaped
rather than semantics-shaped: it returns `409` because it "save[s] results only
after the execution of an endpoint begins," so there is nothing recorded to
return. `[INFERENCE]` For a stream there *is* something to return — the live
stream itself — so the premise that produced `409` does not hold at the
intersection.

**Verdict: reject as a MUST; retain as one permitted, documented answer.**

### 6.3 Shape C — "the API MUST document which behavior it implements"

**FOR.** `[COMPARATIVE]` The field genuinely diverges, and the divergence is
correlated with whether the API streams — which means it is a design choice, not
an error by one side. A standard that picks a winner here picks against either
every keyed non-streaming API or the only keyed streaming one.

**FOR — one mature system already models this exact menu as a documented
choice.** `[FACT]` Temporal's `WorkflowIdConflictPolicy` is a caller-selected
enumeration over Fail, Use Existing, and Terminate Existing, with the default
stated — independently verified against
https://docs.temporal.io/workflow-execution/workflowid-runid on 2026-08-10.
`[COMPARATIVE]` Orkes Conductor ships the same idea: an `idempotencyKey`
accompanied by an `idempotencyStrategy` taking `RETURN_EXISTING`, `Fail`, or
`Fail on Running`, the last of which "fails if another workflow with the same
idempotency key is currently in a Running or Paused state." *Attribution note:
the strategy enumeration is corroborated across Orkes' own documentation pages,
but the request-header spelling reported by the sweep
(`X-Idempotency-key` / `X-on-conflict`) was **not** independently confirmed and
is therefore not asserted here; only the body-field form is cited.*
`[INFERENCE]` A rule that requires an API to name its behavior is therefore not
a novel compliance burden; it is the shape two independent workflow platforms
already chose, and both make it per-request rather than per-API.

**FOR — four distinct answers were found, which settles that no single one can
be mandated.** `[COMPARATIVE]` Attach (Cog, Temporal), reject-while-executing
(Stripe, Shopify, expired draft), reject-any-reuse (ComfyUI, `422`), and
cancel-the-original (TensorRT-LLM). `[INFERENCE]` The last is a warning as much
as a data point: a client-supplied identifier on a streaming endpoint already
means "cancel" somewhere in the field, so a standard that assigns it replay
semantics without requiring disclosure invites a genuinely dangerous collision.

**FOR.** `[FACT]` A client cannot discover the behavior by probing without
executing the very operation whose duplication it fears. `[INFERENCE]` Where a
behavior is undiscoverable and consequential, disclosure is the minimum a
standard can require, and `R1.7`'s conformance note is the existing carrier.

**AGAINST.** `[POLICY]` A rule that permits two documented answers (three
observable behaviors, since serving the in-flight work covers both a stream and
a state representation) and requires only
documentation buys interoperability of *understanding*, not of *code*: a generic
client still cannot be written against it. `[INFERENCE]` This is a real cost and
the report accepts it, because the alternative is mandating an answer against
the evidence.

**Verdict: adopt.** Confidence **moderate-high**.

### 6.4 Shape D — the gap-disclosure obligation on an attached stream

This is new in this run and comes from reading the implementation rather than
the documentation.

**FOR — the hazard is demonstrated, not argued.** `[FACT]` With
`COG_STREAM_HISTORY_CAPACITY=0`, an attaching client receives `event: completed`
with `"status":"succeeded"` and **no** `output` events and **no** error, as
`sse_stream_history_disabled.txtar` asserts by testing for their absence.
`[INFERENCE]` The client sees a well-formed terminal frame and concludes it has
a complete result, having been delivered none of the incremental content.

**FOR — `R12.10` cannot catch it.** `[FACT]` `R12.10`'s sole integrity test is
that a connection closing *without* the documented terminal frame is truncated.
`[INFERENCE]` Attachment produces the inverse failure — terminal frame present,
prefix or middle missing — so the client obligation the standard already ships
returns "complete" on a gapped stream. Truncation is a missing suffix;
attachment loss is a missing prefix. No rule in §12 or §13 addresses the second.

**FOR — the rule is non-vacuous and already implementable.** `[FACT]` Cog's
*default* configuration satisfies it: it delivers the retained history from the
first event, and when events have been dropped it emits an `error` event with a
`skipped` count and closes. `[FACT]` Cog's *zero-capacity* configuration
violates it. `[INFERENCE]` A proposed rule whose exemplar satisfies it by
default and fails it in one shipped configuration is in the strongest available
evidentiary position: demonstrably achievable and demonstrably not automatic.

**AGAINST.** `[INFERENCE]` It adds a fourth obligation to an already dense
section, and the cheapest conforming implementation — always emit an error when
anything was dropped — degrades the very recovery attachment exists to provide.
`[POLICY]` The rule is therefore written to permit three discharges (deliver the
prefix, signal the loss, or number the frames) rather than mandating one.

**Verdict: adopt.** Confidence **moderate-high** on the obligation, **moderate**
on the strength.

---

## 7. Revised proposed rule text

Provisional identifiers remain **leaf-scoped** for the reason the first run
gave — `baseline-04c` independently claimed `ST-021` against the same §13.4 row,
and the ratifying decision record assigns final `ST-*` numbers. This run uses
`ST-E01b`, `ST-E02b`, `ST-E03b` to keep both runs' proposals distinguishable in
review.

### 7.1 Amendment riding `R3.9` — define "the stored response" (unchanged in substance)

> **`R3.9`, added clause.** For the purpose of this rule, **the stored
> response** of a request is the outcome the server recorded for that request:
> its status code, its response headers, and — where the response was a
> complete non-streaming body — that body. Where the response was a **stream**
> (§13), the stored response is the stream's **terminal state** together with
> whatever representation of the result the API documents as replayable. An API
> MUST document what a replay of a streaming request delivers.

**Classification:** `[POLICY]`. **Confidence: moderate-high**, raised from the
first run by corroboration rather than by argument. `[FACT]` Cog's terminal
`completed` event carries the full accumulated `output` array, so a client that
missed every incremental frame still receives the complete outcome in the
terminal frame. `[INFERENCE]` The only implementation at the intersection
independently arrived at "the outcome is the durable thing, the frame sequence
is not," which is exactly what this clause says.

### 7.2 Amendment riding `R3.9` — the exception should exempt the header, not the guarantees

**This is new in run b, and it is the most consequential of the three.**

`[FACT]` `R3.9` reads, in part: "**Exception:** naturally idempotent operations
(PUT with a client-supplied ID)." `[FACT]` Cog is precisely that construction —
a `PUT` whose last path segment is a client-supplied identifier. `[INFERENCE]`
Because the exception is written as a blanket exemption from the whole rule, an
API that adopts it inherits **none** of `R3.9`'s obligations: not the payload
fingerprint, not the ≥24-hour retention floor, not the duty to state a window.
Cog demonstrates the outcome exactly — it fingerprints nothing and retains
nothing past terminal — while remaining, on the standard's own text, outside the
rule rather than in breach of it.

**The exception must be split, because it currently covers two unlike things.**
`[INFERENCE]` A plain state-replacement `PUT` — "set this resource's
representation to this" — needs no guarantees at all: repeating it is safe by
method semantics, and a later `PUT` carrying a *different* body is a legitimate
update that the server neither can nor should distinguish from a retry. An
**execution-shaped `PUT`** is a different animal: the request names a resource,
but what it does is *start work whose repetition has an external effect*. Cog's
prediction is the case — a second run is a second GPU-seconds charge — and the
tell is visible in Cog's own behavior: `[FACT]` its `PUT` ignores the enclosed
representation when the resource already exists, which means it is not
performing state replacement in the first place.

> **`R3.9`, revised exception.** The exception for naturally idempotent
> operations (a `PUT` to a client-named resource) exempts such an operation
> from the **carriage** requirement in every case — it need not accept the
> `Idempotency-Key` header, because the request target already carries the
> identifier.
>
> Where the `PUT` performs state replacement (RFC 9110 §9.3.4), the exemption
> is complete and no further obligation attaches.
>
> Where the `PUT` instead **initiates work whose repetition would have an
> external effect** — a charge, a dispatch, a send — the operation is exempt
> from the header only. The server MUST still reject a repeat whose payload
> differs from the original's rather than silently serving the original's work,
> and MUST still state its deduplication window.

**Classification:** `[POLICY]`. **Confidence: moderate-high** on the split,
moderate on the boundary test.

**How to tell the two apart, since the rule turns on it.** `[POLICY]` The test
is what the server does with the enclosed representation on a repeat. A
state-replacement `PUT` applies it. An execution-shaped `PUT` cannot apply it —
the work is already running — so it must either reject the repeat or ignore the
body. An API whose `PUT` ignores a differing body is in the second class by
construction, and owes the guarantees.

**Why the fingerprint obligation must follow the identifier into the URI.**
`[FACT]` Cog performs no payload comparison: the sole body check in
`create_prediction_idempotent` is that a body `id`, where present, equals the
path `id`, yielding `422` otherwise; the `input` member is never compared
against the original. `[INFERENCE]` So a repeat carrying the same ID and
*different* input silently attaches to the original and the new input is
discarded — the client believes it submitted new work and receives the old
work's output. That is the exact failure `R3.9`'s reject-clause exists to
prevent, and moving the identifier from a header to a path segment does nothing
to make it less likely; if anything a path segment is easier to reuse by
accident, since it is a resource name a client may legitimately want to address
twice.

**A second, protocol-level ground.** `[FACT]` RFC 9110 §9.3.4, verbatim: "The
PUT method requests that the state of the target resource be created or replaced
with the state defined by the representation enclosed in the request message
content" (https://www.rfc-editor.org/rfc/rfc9110.txt, STD 97, June 2022,
accessed 2026-08-10).
`[INFERENCE]` A `PUT` that ignores its enclosed representation when the resource
already exists is not implementing `PUT` semantics; it is implementing
create-or-attach under a `PUT` spelling. The standard should adopt Cog's
attachment behavior and decline Cog's payload-blindness, and it can say so with
RFC 9110 rather than with preference.

### 7.3 `ST-E01b` — a keyed repeat of a streaming request never re-executes, and what it may return

> **`ST-E01b`** Where an API deduplicates a request by a client-supplied
> identifier (`R3.9`, in either carriage) and that request's success response is
> a stream (§13), a repeat carrying the same identifier and a matching payload
> MUST NOT cause the underlying work to be executed a second time while the
> identifier remains within the API's stated deduplication window. The server
> MUST answer according to the state of the original execution.
>
> 1. **Original still executing.** The server MUST NOT begin a second
>    execution, and MUST do one of the following two things, which the API MUST
>    document in advance:
>    - **serve the in-flight work** — deliver the running operation in the
>      representation negotiated by the **repeat's** `Accept` (`R13.2`), not by
>      what the original request received: as the in-flight stream where a
>      stream is negotiated, subject to clause 4, or as a non-streamed
>      representation of the operation's current state where one is not;
>    - **reject with conflict** — `409 Conflict`, served as
>      `application/problem+json` per `R5.12` with a `type` and `code` from the
>      `R5.16` catalog. The server MUST NOT record that response as the stored
>      response for the identifier.
> 2. **Original reached a terminal state** — success or failure, including a
>    stream ended by an `error` frame (`R13.7`). The server MUST NOT re-execute
>    while the identifier is within the stated window, and MUST deliver the
>    stored response as defined in `R3.9`. It MAY deliver it as a stream
>    replayed from the first frame or as a non-streamed representation of the
>    same outcome; whichever it does, the API MUST document it, and a replayed
>    response MUST be distinguishable from a first response.
> 3. **Original's execution began and reached no terminal state** — the
>    connection closed and the work was abandoned, cancelled, or lost. The API
>    MUST document its behavior for this case. If it re-executes, it MUST
>    document that it does, because the client cannot otherwise know that its
>    keyed retry can charge twice.
> 4. **Attached and replayed streams MUST make omitted frames detectable.** A
>    server that delivers a stream which does not begin at the original
>    stream's first frame MUST discharge this by at least one of: delivering
>    the retained frames from the first frame onward; ending the stream with a
>    defined `error` frame (`R13.7`) stating that frames were dropped; or
>    carrying a position on every frame whose numbering makes an omission
>    visible — a documented origin and a documented increment, so that a gap is
>    distinguishable from sparse numbering. (`R13.10`'s `stream_position` is
>    strictly increasing but not necessarily dense, so `stream_position` alone
>    discharges this clause only where the API documents it as dense from a
>    stated origin.) The server MUST NOT deliver a terminal frame (`R13.6`) on a
>    stream from which earlier frames were silently omitted.
>
> A stream that ends without its terminal frame (`R12.10`) is **not** a terminal
> state and MUST NOT be treated as one. Beyond the stated window an API MAY
> re-execute, and `R3.9` already requires the window to be stated.

**Classification.** Clause 1's non-re-execution obligation is `[COMPARATIVE]`
(Stripe, Shopify, Zalando 230, AIP-155, OASIS, Cog). Clause 1's menu is
`[POLICY]`, adjudicating a genuine field conflict rather than averaging it —
§6.2 states which source governs where and why. Clause 2 is `[COMPARATIVE]` on
"do not re-execute" and `[POLICY]` on permitting a non-streamed replay. Clause 3
is `[POLICY]` — a documentation duty standing in for a rule the evidence cannot
support. Clause 4 is `[POLICY]`, grounded on a demonstrated shipped hazard
(§6.4).

**Confidence:** high (clause 1's non-re-execution), moderate-high (clause 1's
menu, clause 2, clause 4), moderate (clause 3).

**What changed from `ST-E01`, in one line each.** The `409` moved from mandatory
to one of two documented answers; serving the in-flight work was added and is now the
evidence-backed option; negotiation of the repeat's representation was made
explicit under `R13.2`; the non-re-execution guarantee was scoped to the stated
window, which every implementation requires and neither run's predecessor said;
and clause 4 is entirely new.

### 7.4 `ST-E02b` — two structural alternatives, not one

> **`ST-E02b`** An API SHOULD NOT stream the response to a non-idempotent
> mutation whose repeated execution has an irreversible external effect — a
> payment, a disbursement, a message send, a metered charge. Where such a
> capability needs incremental delivery, the API SHOULD adopt one of two
> structures:
>
> - **Split execution from delivery.** The mutation is a non-streaming request
>   returning an operation resource (`R10.9`, `R13.9`), and the incremental
>   delivery is a **safe** request over that resource, resumable under
>   `R13.10`.
> - **Name the work in the request target.** The mutation is a `PUT` to a
>   client-named resource, so the method is idempotent by RFC 9110 §9.2.2, and
>   a repeat addresses the same work rather than creating new work.
>
> Method idempotence alone does not discharge `ST-E01b`: an API adopting the
> second structure still owes clause 4, and still owes the revised `R3.9`
> exception's payload and window obligations (§7.2). An API that streams such a
> mutation under neither structure MUST comply with `ST-E01b` and MUST state in
> its conformance note (`R1.7`) which of clause 1's three answers it
> implements.

**Classification:** `[COMPARATIVE]` on both structures — the first is OpenAI's
shipped `background: true` design `[FACT]` and the architecture gRPC A6 and MCP
independently reach `[FACT]`; the second is Cog's shipped design `[FACT]` and is
the construction `R3.9` already names as its exception `[FACT]`. `[POLICY]` on
the SHOULD-NOT and on the scoping to irreversible effects.

**Confidence: moderate-high.** Raised from the first run, because the
recommendation now rests on two independently shipped structures rather than
one, and because naming the second closes the hole §7.2 identifies.

**Why the second structure was invisible to the first run.** `[INFERENCE]` It
searched for a header. The second structure has no header — its identifier is
the resource name — which is also why `R3.9` exempts it and then stops
describing it.

### 7.5 The question the first run did not ask: an identical-payload repeat that wants to *attach*

**The question.** `R3.9` requires the server to reject a reused key carrying a
*different* payload. What should happen when a keyed repeat carries an
**identical** payload but the client wants to attach to the running work rather
than receive a replay?

**The answer, in one sentence.** `[POLICY]` The client does not choose; three
independent mechanisms decide in sequence, and the standard should keep them
separate rather than adding a fourth.

| Stage | Decided by | Governs |
| --- | --- | --- |
| **Admission** | `R3.9`'s payload fingerprint | Whether the repeat is a genuine retry at all. A differing payload is rejected here and never reaches the delivery decision. |
| **Delivery** | The server's documented `ST-E01b` clause 1 policy | Whether a matching repeat is served the in-flight work or a `409`. |
| **Representation** | The repeat's `Accept` (`R13.2`) | Whether the served work arrives as a stream or as a state representation. |

`[INFERENCE]` **The fingerprint gate and the attach decision are orthogonal, and
this is why the question has a clean answer.** The fingerprint exists to catch a
client that accidentally reused a key for *different* work — a safety check on
what was requested. It says nothing about what a *matching* repeat receives.
`R3.9` therefore does not forbid attachment, and no change to its reject-clause
is needed to permit it.

`[POLICY]` **The standard should not add a client-side "attach" signal**, and
this run declines to propose one. Three reasons. (a) It would be a fourth
request modifier interacting with `Accept`, `Prefer`, and `stream`, and `R13.3`
already had to rule the `stream`-versus-`Accept` conflict; adding another
negotiation axis multiplies that problem. (b) The client cannot know which
answer is available — whether the work is still in flight is server state — so a
client preference would frequently be unsatisfiable and would need its own
failure mode. (c) `R13.2` already carries the only choice the client is entitled
to make: streamed or not. Cog reaches the same design without arguing for it —
`[FACT]` a repeat gets "the response type requested by the retry," which is
content negotiation and nothing more.

**Does Cog's URI-keyed design sidestepping the fingerprint matter for this
standard? Yes, in one direction only.** `[INFERENCE]` It does **not** matter for
*where the identifier rides*: a path segment and a header identify a request
equally well, and `R3.9` already blesses the path-segment form. It **does**
matter for *what obligations follow the identifier*, because `R3.9`'s exception
is written as a blanket exemption, so an API taking the path-segment route
inherits no fingerprint duty at all. `[FACT]` Cog is the demonstration: it
compares no payload, and a repeat carrying different `input` is silently served
the original's work. `[POLICY]` That is a real defect and the standard should not
inherit it — which is exactly what §7.2's split exception fixes. The header
mandate itself is not what protects the client; the fingerprint is, and the
fingerprint is what the current exception accidentally discards.

### 7.6 `ST-E03b` — the client-side half, riding `R12.10`

> **`R12.10`, added clause.** A client that receives a stream in response to a
> repeated keyed request MUST NOT infer from the presence of a terminal frame
> (`R13.6`) that it received every frame of the original stream. Where the API
> documents attachment (`ST-E01b` clause 1), the client MUST treat the stream as
> potentially beginning after the original's first frame, and MUST rely on the
> terminal frame's outcome rather than on the accumulated incremental content
> unless the API documents that replay begins at the first frame.

**Classification:** `[POLICY]`. **Confidence: moderate-high.** `[INFERENCE]`
This exists because `R12.10`'s only integrity test is suffix-shaped and the
hazard demonstrated in §6.4 is prefix-shaped; without this clause a conforming
client draws a false conclusion from a conforming server's response.

---

## 8. What survived from the first run

Both findings the task flagged as plausibly independent of the falsified
negative were re-verified. Both hold, and both are now better evidenced.

### 8.1 `R3.9`'s "stored response" is genuinely undefined for a stream — **survives**

`[FACT]` `R3.9` requires the server to "replay the stored response for a genuine
retry" and defines the term nowhere. `[FACT]` The standard's own §13.4 table
records the consequence in released text: "`R3.9` replays 'the stored response'
for a genuine retry. For a stream that is undefined: replay from the first
frame, serve a non-streamed representation, or resume — which is `R13.10`, a
different mechanism with different preconditions."

`[INFERENCE]` This is self-evidencing and needs no field support: the ambiguity
is a property of the standard's text, not a claim about the world, so the
falsification cannot touch it. **New corroboration:** Cog resolves the same
ambiguity three different ways depending on configuration and on the repeat's
`Accept` — full prefix replay then live tail, an `error` frame and close, a
`202` status representation, or (at zero capacity) the terminal frame alone.
`[INFERENCE]` One implementation producing four distinguishable answers to
"what does a replay deliver" is the strongest possible demonstration that the
phrase does not determine a behavior.

### 8.2 The retention-floor mismatch forbids mandating frame-for-frame replay — **survives, and strengthens**

`[FACT]` `R3.9` sets the floor: "The stated retention window MUST be at least 24
hours."

`[FACT]` The field's retained streaming artifacts, restated with their units:

| Implementation | What is retained | Denomination | Magnitude |
| --- | --- | --- | --- |
| Cog replay buffer | SSE events for one in-flight prediction | **event count** | 1024 events by default; `0` disables |
| Cog prediction registry | The prediction itself | execution lifetime | deleted at terminal state |
| OpenAI Responses `background` | Response data for polling and resumption | time | "roughly 10 minutes" |
| Kubernetes watch | Change history | time | roughly 5 minutes (`survey-08` §4.6) |
| Stripe idempotency key | Status code and body (not a stream) | time | ≥24 h, then pruned |
| Cloudflare AI Gateway cache | Response bodies | time (TTL) | **streams excluded**: "Streaming responses are not cached by default" |
| ComfyUI Cloud API v2 | The key only — explicitly "no response replay" | time | 24 h |

`[INFERENCE]` **Cog's buffer is not merely shorter than 24 hours — it is not
denominated in time at all.** A capacity of 1024 events is minutes for a slow
generation and seconds for a fast one; the same configuration yields different
windows for different models on the same server. A rule requiring frame-for-frame
replay "for at least 24 hours" cannot be satisfied by, or even meaningfully
compared against, a buffer of this shape. The first run inferred this gap from a
two-order-of-magnitude difference in time; run b finds a unit mismatch beneath
it, which is a stronger form of the same argument.

`[FACT]` A second, independent corroboration: the buffer's lifetime is bounded
by the prediction's, because the registry entry is removed at terminal state and
is the only store. `[INFERENCE]` So the retained frames are gone at exactly the
moment `R3.9`'s window would begin to matter.

**Conclusion, unchanged:** `[POLICY]` frame-for-frame replay is permitted by
`ST-E01b` clause 2 and never required. Confidence **high**, raised from the
first run's moderate.

### 8.3 What else survived, briefly

- `[COMPARATIVE]` **Replay and resumption are distinct mechanisms.** The first
  run's §4 table stands. Cog supplies an empirical instance of the one
  configuration where they converge — attaching to a running prediction with a
  full buffer *is* replay from position zero followed by live resumption on one
  connection — which the first run predicted analytically and could not cite.
- `[FACT]` **The abandoned-execution case remains undocumented everywhere**,
  including in Cog, which simply deletes the prediction and re-executes.
- `[POLICY]` **Stripe governs over OASIS on replaying recorded failures.** The
  first run's §6.4 adjudication is untouched by this run's evidence and is not
  re-litigated here.
- `[FACT]` **The `Idempotency-Key` header still has no shipped streaming
  exemplar**, and the IETF draft that standardized it is still expired. `R3.9`
  remains correctly labelled `[POLICY]`.

---

## 9. Declined alternatives

**9.1 Keep `409` as a MUST for the in-flight case.** Declined. `[FACT]` The only
implementation at the intersection attaches instead; the nearest-to-standard
source says SHOULD, not MUST; and Stripe's own reason for `409` — nothing
recorded to return — does not hold when a live stream exists to return (§6.2).

**9.2 Mandate attachment as a MUST, inverting the first run.** Declined, and
this is the discipline the falsification demands rather than a hedge.
`[INFERENCE]` One implementation is an existence proof, not a practice. Cog is
a single vendor's serving layer, roughly eleven weeks old at the intersection,
non-conformant with `R3.9` on four of six clauses (§5.6), and its attachment is
partly forced by an absent operation resource (§5.7) rather than chosen on the
merits. Promoting it to a MUST would repeat the first run's error with the sign
flipped — deriving a universal from a bounded observation.

**9.3 Mandate frame-for-frame replay from frame 1.** Declined, as in the first
run, now with a stronger ground: the unit mismatch of §8.2.

**9.4 Forbid streaming responses on non-idempotent mutations outright.**
Declined, as in the first run. `[FACT]` AIP-151's "The response **must not** be
a streaming response" binds long-running-operation methods, not all mutations.
`[INFERENCE]` A flat ban would forbid the pattern §13 exists to serve, and Cog
now demonstrates that the pattern can be made safe rather than merely tolerated.

**9.5 Rule that `R13.10` resumption satisfies `R3.9` for streams.** Declined, as
in the first run: a SHOULD conditioned on a retained artifact cannot discharge
an unconditional MUST, and Cog's stream is a live generation with no
independently addressable backing store, so `R13.10` does not even reach it.

**9.6 Copy Cog's payload handling.** Declined explicitly, because a reader could
otherwise take §7 as an endorsement of Cog in whole. `[FACT]` Cog compares no
payload and discards a repeat's differing `input`. `[INFERENCE]` Adopting that
would delete `R3.9`'s reject-clause for exactly the endpoints where the work is
most expensive to duplicate. §7.2 adopts Cog's attachment and declines its
payload-blindness, on RFC 9110 §9.3.4 grounds rather than on preference.

**9.7 Require `Idempotent-Replayed: true` on the wire.** Declined, as in the
first run — Stripe-only, and §1.10's IANA-registry test for reserved header
names does not admit a vendor-invented header. `ST-E01b` clause 2 requires only
that a replay be distinguishable.

**9.8 Mandate a specific `409` problem `type` URI or `code` string.** Declined,
as in the first run: `R5.13` and `R5.16` already bind the template and the
catalog.

---

## 10. What could not be verified

1. **Whether Replicate's hosted product exposes Cog's idempotent streaming
   semantics.** Cog is the open-source serving layer; the hosted API is a
   separate surface with separate documentation, and the two are not
   interchangeable evidence. Treated throughout as one implementation, not two.
2. **The abandoned-execution case remains undocumented everywhere**, in run b as
   in run a. Cog's behavior — delete and re-execute — is observable in source
   but is not stated in its documentation, so it is implementation behavior
   rather than a documented contract, and it is cited as such.
3. **Cog's behavior under concurrent attachment at scale.** The subscriber cap
   (`429` beyond 1024 live subscribers) and the lag path are read from source;
   no deployment evidence was available on how often either fires.
4. **Whether any private or unpublished API carries an `Idempotency-Key`
   *header* on a streaming endpoint.** Run a's negative on this narrower claim
   was not falsified by run b's search, but a search cannot establish
   non-existence — a lesson this run exists to record.
5. **Whether the AI providers will activate the dormant Stainless
   `idempotencyKey` machinery** (`baseline-02g`, 2026-08-09). Unchanged.
6. **OpenAI's real resumption window** — vendor "roughly 10 minutes" against
   community reports of five (`survey-08` §11.1). Unchanged, and immaterial: the
   §8.2 argument holds at either figure and now rests on a unit mismatch
   regardless.
7. **Source-code behavior is pinned to one commit.** All `crates/coglet` claims
   are read at repository HEAD `e456e1924c6051d22008761f4457a008a5f5d7b7`
   (2026-08-10) and the documentation at commit
   `966752e9f5f5c165fc5e9618642fd353f0db0e56`. Cog is actively developed; a
   re-run should re-pin.
8. **Negatives in §4 are bounded by the pages named there.** This run's own
   negatives carry the same limitation that produced run a's error, and are
   stated with their scope attached for that reason.
9. **Whether any A2A agent actually implements the `MAY` deduplication.** The
   specification permits it; no conforming implementation was examined, so A2A
   is cited as specification evidence only and never as shipped behavior.
10. **Azure Durable Functions' `409` on a duplicate `instanceId` is
    source-grade only.** The behavior is visible in the runtime's request
    handler; the current Microsoft Learn HTTP-API page documents only `202` and
    `400`. Recorded as undocumented rather than as a documented contract, and
    not used to support any clause.
11. **EventStoreDB/KurrentDB's wire response to a repeated `Kurrent-EventId`.**
    The mechanism and its purpose are documented — "KurrentDB uses EventId for
    impotency" (typo present in the source) — but no fetched page shows the
    status and body returned for the repeat, so the row is recorded as a
    documented mechanism with an unverified response.
12. **ComfyUI Cloud API v2's currency.** The OpenAPI document is published and
    was fetched and quoted directly (HTTP 200, 2026-08-10), so the *text* is
    verified. What is not verified is deployment: the rendered public reference
    still documents the v1 API, and the SSE endpoint itself documents a `501
    not_implemented` for deployments not yet serving it. Treated as a published
    specification, not as observed shipped behavior.
13. **Orkes Conductor's request-header spelling.** The idempotency-strategy
    enumeration is corroborated across its documentation, but the header names
    reported by the sweep were not independently confirmed and are not asserted
    (§6.3).

---

## 11. Sources

**The falsifying implementation (new in run b), all accessed 2026-08-10:**

- https://raw.githubusercontent.com/replicate/cog/main/docs/http.md — pinned at commit `966752e9f5f5c165fc5e9618642fd353f0db0e56`
- https://github.com/replicate/cog/tree/e456e1924c6051d22008761f4457a008a5f5d7b7/crates/coglet/src — `transport/http/routes.rs`, `prediction.rs`, `service.rs`
- https://github.com/replicate/cog/tree/e456e1924c6051d22008761f4457a008a5f5d7b7/integration-tests/tests — `sse_stream_history_capacity.txtar`, `sse_stream_history_disabled.txtar`
- https://github.com/replicate/cog/pull/3019 — merged 2026-05-22
- https://api.replicate.com/openapi.json — the hosted product's verified negative

**Primary standards and specifications:**

- https://www.rfc-editor.org/rfc/rfc9110.txt (RFC 9110 §6.1, §9.2.2, §9.3.4)
- https://www.rfc-editor.org/rfc/rfc9457.html (RFC 9457 §3.1)
- https://www.ietf.org/archive/id/draft-ietf-httpapi-idempotency-key-header-07.txt (**expired**)
- https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/ (status: Expired)
- https://docs.oasis-open.org/odata/repeatable-requests/v1.0/cs01/repeatable-requests-v1.0-cs01.html
- https://modelcontextprotocol.io/specification/2025-06-18/basic/transports
- https://raw.githubusercontent.com/grpc/proposal/master/A6-client-retries.md
- https://raw.githubusercontent.com/kserve/open-inference-protocol/main/specification/protocol/inference_rest.md
- https://raw.githubusercontent.com/triton-inference-server/server/main/docs/protocol/extension_generate.md
- https://raw.githubusercontent.com/a2aproject/A2A/main/docs/specification.md — A2A Protocol v1.0.0
- https://raw.githubusercontent.com/a2aproject/A2A/main/specification/a2a.proto

**Guidelines:**

- https://google.aip.dev/151 · https://google.aip.dev/155
- https://raw.githubusercontent.com/microsoft/api-guidelines/vNext/azure/Guidelines.md
- https://raw.githubusercontent.com/zalando/restful-api-guidelines/main/chapters/http-headers.adoc

**Shipped API documentation and implementation source:**

- https://docs.stripe.com/api/idempotent_requests · https://docs.stripe.com/error-low-level
- https://shopify.dev/docs/api/usage/implementing-idempotency · https://shopify.dev/docs/api/usage/idempotent-requests
- https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_InvokeModelWithResponseStream.html
- https://developers.openai.com/api/docs/guides/background
- https://raw.githubusercontent.com/openai/openai-openapi/master/openapi.yaml
- https://platform.claude.com/docs/en/api/errors
- https://generativelanguage.googleapis.com/$discovery/rest?version=v1beta
- https://github.com/vllm-project/vllm/tree/main/vllm/entrypoints/openai/responses
- https://raw.githubusercontent.com/triton-inference-server/tensorrtllm_backend/main/README.md
- https://raw.githubusercontent.com/ollama/ollama/main/docs/api.md
- https://raw.githubusercontent.com/huggingface/text-generation-inference/main/docs/openapi.json
- https://raw.githubusercontent.com/mudler/LocalAI/master/core/http/middleware/request.go
- https://docs.bentoml.com/en/latest/get-started/async-task-queues.html
- https://docs.temporal.io/workflow-execution/workflowid-runid
- https://trigger.dev/docs/idempotency · https://trigger.dev/docs/management/sessions/channels
- https://docs.comfy.org/openapi-v2.yaml
- https://developers.cloudflare.com/ai-gateway/features/caching/ · https://developers.cloudflare.com/ai-gateway/reference/troubleshooting/
- https://www.inngest.com/docs-markdown/guides/handling-idempotency
- https://docs.kurrent.io/server/v25.0/http-api/optional-http-headers/
- https://kubernetes.io/docs/reference/using-api/api-concepts/
- https://platform.claude.com/docs/en/api/messages/batches/create

**Prior repo research relied on for previously verified detail:**

- `research/reports/baseline-04e-stream-idempotency-replay.report.2026-08-10.md` (run a — the report this run revises)
- `research/reports/baseline-02g-idempotency-key-practice.report.2026-08-09.md`
- `research/reports/survey-05-reliability.report.2026-07-19.md`
- `research/reports/survey-08-streaming.report.2026-08-10.md`
- `research/reports/baseline-04-streaming.report.2026-08-10.md`
- `research/decisions/baseline-04-streaming.decision.md` (ratified; not reopened)
