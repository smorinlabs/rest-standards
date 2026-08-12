# baseline-04g — Signalling that delivery ended while the work did not

**Series:** `baseline` (prescriptive — this report *proposes* rules; only a
ratified record in `research/decisions/` binds the standard).
**Parent:** `baseline-04` (streaming), via the Phase 8 proposal set in
`docs/reviews/2026-08-10-phase-8-consolidated-proposals.md`.
**Run date:** 2026-08-10 (assigned pairing key; all source access dates below
are **2026-08-11**, the date the fetches were executed).
**Question:** When a streaming response ends while the underlying work is still
running — or has succeeded but its result is no longer deliverable — how does
the API tell the client, and how does the client tell that apart from "the work
failed"?

Label key used throughout: `[FACT]` sourced statement · `[COMPARATIVE]`
cross-vendor observation · `[INFERENCE]` reasoning from sourced facts ·
`[POLICY]` this project's choice.

---

## 1. TL;DR and recommendation

**Recommendation, in two sentences.** The standard should **scope both `R12.10`
and `R13.9` rather than invent a new terminal frame type**: the terminal frame
should be permitted to carry the operation's *current* state — including a
non-terminal one — together with a documented reason naming why delivery ended,
and an `error` frame should determine the fate of the **delivery**, with the
operation resource (which `R13.9` already declares authoritative and already
makes reachable through `operation_id`/`operation_url`) determining the fate of
the **work**. This is what the two closest shipped precedents do — Temporal
returns the *most advanced lifecycle stage reached* plus a continuation handle
on a success response, and A2A v1.0.1 closes the stream when the task reaches "a
terminal **or interrupted** state" — while the one protocol that shipped a
standalone end-of-stream marker decoupled from state (A2A v0.3.0's `final`)
removed it in v1.0.1 as redundant.

**The four load-bearing findings.**

1. `[FACT]` **No published standard defines the vocabulary.** HTTP semantics and
   framing (RFC 9110, RFC 9112), Problem Details (RFC 9457), SSE (WHATWG HTML),
   CloudEvents v1.0.2 and its Subscriptions API, AsyncAPI v3.0.0, gRPC's
   protocol and status-code documents, and RFC 6202 all lack any status, field,
   or event meaning "the transfer ended, the work did not." The verified
   negative is documented source-by-source in §4. This removes the option of
   *adopting* a vocabulary and makes whatever the standard does a `[POLICY]`
   choice constrained by comparative practice.
2. `[FACT]` **gRPC has this standard's exact constraint and lives with the
   ambiguity, at a documented cost.** `DEADLINE_EXCEEDED` is defined as: "For
   operations that change the state of the system, this error may be returned
   even if the operation has completed successfully." A single status conflates
   transport termination with RPC failure, and gRPC's own text concedes the
   client cannot tell. That concession is the strongest evidence *against*
   leaving the case undefined.
3. `[FACT]` **The error channel is already used in the field for
   delivery-only conditions, and clients branch on the code, not the type.**
   Kubernetes delivers a `410 Gone` mid-watch as an in-band `ERROR` event
   carrying a `Status` object with `reason: Expired`; the reference client
   (`client-go`'s `Reflector`) branches on that reason and logs "Watch closed"
   at verbosity 4, versus "Failed to watch" for anything else. Replicate's Cog
   emits an `error` event when its SSE replay buffer has dropped events, while
   the prediction itself keeps running. Under `R12.10` as written, a conforming
   client must read both as "the operation failed."
4. `[FACT]` **The two systems that model "work with an identity beyond the
   stream" both report the current, possibly non-terminal, state.** Temporal's
   `UpdateWorkflowExecutionResponse.stage` carries "the most advanced lifecycle
   stage that the Update is known to have reached," with `UNSPECIFIED` meaning
   "the server's maximum wait time was reached before the Update reached the
   stage specified in the request WaitPolicy … clients may then retry the call
   as needed" — a non-error response whose meaning is exactly "delivery ended at
   my limit, the work continues." A2A v1.0.1 closes the stream on "a terminal or
   interrupted state," the interrupted states being `TASK_STATE_INPUT_REQUIRED`
   and `TASK_STATE_AUTH_REQUIRED`, neither of which is terminal.

**What this means for the Phase 8 rulings.** The D2 decision ("exempt and
signal": the terminal frame is exempt from carrying `operation_state`, and the
client continues through the operation resource) is *directionally* right — the
operation resource is the continuation — but it declined the alternative the
field actually ships. §6 sets out why the declined alternative's stated
objection does not apply to a relaxation conditioned on the divergence case, and
recommends the walk revisit D2's repair choice. §6 also shows that `ST-025`
(credential expiry ⇒ `error` frame) and the `ST-026` gap signal (undeliverable
replay ⇒ `error` frame) are, as decided, in direct collision with `R12.10`, and
that the collision is not removable by drafting alone.

---

## 2. Standards-and-currency matrix

Authority classes: **PS** protocol standard (IETF Standards Track, WHATWG Living
Standard) · **PD** published draft or expired draft · **PP** published protocol
specification from a foundation or project (not IETF) · **VD** vendor
documentation · **VS** vendor or project source code / machine-readable
specification · **SD** secondary (forum, blog).

Repository sources are pinned to a commit or an annotated tag's commit, per the
prior leaf's citation challenge. Moving references (`/main`, `/master`, living
vendor pages) are marked as such.

| ID | Source | URL | Class | Version / date | Accessed |
| --- | --- | --- | --- | --- | --- |
| S1 | RFC 9112 §8 — Handling Incomplete Messages | `https://www.rfc-editor.org/rfc/rfc9112.html` | PS | STD 99, 2022-06 | 2026-08-11 |
| S2 | RFC 9110 §15.3.3 — 202 Accepted | `https://www.rfc-editor.org/rfc/rfc9110.html` | PS | STD 97, 2022-06 | 2026-08-11 |
| S3 | RFC 9457 — Problem Details for HTTP APIs | `https://www.rfc-editor.org/rfc/rfc9457.html` | PS | 2023-07 | 2026-08-11 |
| S4 | WHATWG HTML — Server-sent events | `https://html.spec.whatwg.org/multipage/server-sent-events.html` | PS | Living Standard (moving URL) | 2026-08-11 |
| S5 | RFC 9113 §6.8 — GOAWAY | `https://www.rfc-editor.org/rfc/rfc9113.txt` | PS | 2022-06 | 2026-08-11 |
| S6 | RFC 6202 — Long Polling and Streaming in Bidirectional HTTP | `https://www.rfc-editor.org/rfc/rfc6202.html` | PS (Informational) | 2011-04 | 2026-08-11 |
| S7 | RFC 2518 §10.1 — 102 Processing; IANA HTTP Status Code Registry | `https://www.rfc-editor.org/rfc/rfc2518.txt` · `https://www.iana.org/assignments/http-status-codes/http-status-codes.xhtml` | PS (obsoleted) + registry | RFC 2518 obsoleted by RFC 4918 (2007-06); registry still cites RFC 2518 | 2026-08-11 |
| S8 | RFC 8297 — 103 Early Hints | `https://www.rfc-editor.org/rfc/rfc8297.html` | PS (Experimental) | 2017-12 | 2026-08-11 |
| S9 | gRPC — `doc/statuscodes.md` | `https://github.com/grpc/grpc/blob/39d7a4eaeb6d2a09feedfbc69f9b29a172938eae/doc/statuscodes.md` | PP (pinned) | commit 39d7a4e, 2023-10-06 | 2026-08-11 |
| S10 | gRPC — `doc/PROTOCOL-HTTP2.md` | `https://github.com/grpc/grpc/blob/cf61c7d62a1a7f43b9d2ea6488186bc14fc41a8c/doc/PROTOCOL-HTTP2.md` | PP (pinned) | commit cf61c7d, 2025-04-17 | 2026-08-11 |
| S11 | CloudEvents core spec v1.0.2 | `https://github.com/cloudevents/spec/blob/v1.0.2/cloudevents/spec.md` | PP (tag-pinned) | v1.0.2 | 2026-08-11 |
| S12 | CloudEvents Subscriptions API | `https://github.com/cloudevents/spec/blob/main/subscriptions/spec.md` | PP (**moving URL** — no independent tag) | main | 2026-08-11 |
| S13 | AsyncAPI specification v3.0.0 | `https://www.asyncapi.com/docs/reference/specification/v3.0.0` | PP | v3.0.0 | 2026-08-11 |
| S14 | Kubernetes — API concepts (docs source) | `https://github.com/kubernetes/website/blob/9d7f0c542630ae53fcc4ef36a221a734205ab58a/content/en/docs/reference/using-api/api-concepts.md` | VD (pinned) | commit 9d7f0c5, 2026-05-17 | 2026-08-11 |
| S15 | Kubernetes — `apimachinery/pkg/watch/watch.go` | `https://github.com/kubernetes/kubernetes/blob/60a317eadfcb839692a68eab88b2096f4d708f4f/staging/src/k8s.io/apimachinery/pkg/watch/watch.go` | VS (pinned) | v1.33.0 → commit 60a317e | 2026-08-11 |
| S16 | Kubernetes — `apiserver/pkg/storage/cacher/cacher.go` | same commit, path `staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go` | VS (pinned) | v1.33.0 | 2026-08-11 |
| S17 | Kubernetes — `apimachinery/pkg/api/errors/errors.go` | same commit, path `staging/src/k8s.io/apimachinery/pkg/api/errors/errors.go` | VS (pinned) | v1.33.0 | 2026-08-11 |
| S18 | Kubernetes — `client-go/tools/cache/reflector.go` | same commit, path `staging/src/k8s.io/client-go/tools/cache/reflector.go` | VS (pinned) | v1.33.0 | 2026-08-11 |
| S19 | Kubernetes — `apiserver/pkg/endpoints/handlers/get.go` | same commit, path `staging/src/k8s.io/apiserver/pkg/endpoints/handlers/get.go` | VS (pinned) | v1.33.0 | 2026-08-11 |
| S20 | Kubernetes — KEP-956 watch bookmarks | `https://github.com/kubernetes/enhancements/blob/b9b8b52c1e8830a9c7d7065315d9675bf93f0d04/keps/sig-api-machinery/956-watch-bookmark/README.md` | VD (pinned) | commit b9b8b52, 2020-12-23 | 2026-08-11 |
| S21 | AWS — Kinesis `SubscribeToShard` | `https://docs.aws.amazon.com/kinesis/latest/APIReference/API_SubscribeToShard.html` | VD (living) | — | 2026-08-11 |
| S22 | AWS — Kinesis `SubscribeToShardEvent` | `https://docs.aws.amazon.com/kinesis/latest/APIReference/API_SubscribeToShardEvent.html` | VD (living) | — | 2026-08-11 |
| S23 | A2A Protocol — `docs/specification.md` | `https://github.com/a2aproject/A2A/blob/3303592588e388e62e0f69f701af531d2f4e3991/docs/specification.md` | PP (tag v1.0.1 → commit) | v1.0.1 | 2026-08-11 |
| S24 | A2A Protocol — `specification/a2a.proto` | same commit as S23 | PP (pinned) | v1.0.1 | 2026-08-11 |
| S25 | A2A Protocol — `specification/grpc/a2a.proto` | `https://github.com/a2aproject/A2A/blob/8d57eba286de756176892518a8fc39b0ac2ccefb/specification/grpc/a2a.proto` | PP (tag v0.3.0 → commit) | v0.3.0, 2025-07-30 | 2026-08-11 |
| S26 | A2A Protocol — v0.3.0 `docs/specification.md` | `https://github.com/a2aproject/A2A/blob/8d57eba286de756176892518a8fc39b0ac2ccefb/docs/specification.md` | PP (pinned) | v0.3.0 | 2026-08-11 |
| S27 | A2A Protocol — CHANGELOG + issue #1308 | `https://github.com/a2aproject/A2A/blob/3303592588e388e62e0f69f701af531d2f4e3991/CHANGELOG.md` · `https://github.com/a2aproject/A2A/issues/1308` | PP (pinned) + project issue | v1.0.1 | 2026-08-11 |
| S28 | Temporal — `workflowservice/v1/request_response.proto` | `https://github.com/temporalio/api/blob/3ebdff42a9f07ac484b415fe8ff0b483b4ce3340/temporal/api/workflowservice/v1/request_response.proto` | VS (pinned) | commit 3ebdff4 | 2026-08-11 |
| S29 | OpenAI — `openapi.yaml` (Responses streaming events) | `https://github.com/openai/openai-openapi/blob/577fa92299e000af1616f3be1fec740e3383a86a/openapi.yaml` | VS (pinned) | commit 577fa92, 2026-08-11 | 2026-08-11 |
| S30 | OpenAI — Background mode guide | `https://developers.openai.com/api/docs/guides/background` | VD (living) | — | 2026-08-11 |
| S31 | Anthropic — Streaming Messages | `https://platform.claude.com/docs/en/docs/build-with-claude/streaming` | VD (living) | — | 2026-08-11 |
| S32 | Google — Gemini Live API session management | `https://ai.google.dev/gemini-api/docs/live-session` | VD (living) | — | 2026-08-11 |
| S33 | Replicate Cog — `docs/http.md` | `https://github.com/replicate/cog/blob/966752e9f5f5c165fc5e9618642fd353f0db0e56/docs/http.md` | VD (pinned) | commit 966752e, 2026-07-06 | 2026-08-11 |
| S34 | Replicate Cog — `integration-tests/tests/sse_stream_history_capacity.txtar` | `https://github.com/replicate/cog/blob/1f6c4cdbd8b93df3bfd47056e891d9ca088dd4d0/integration-tests/tests/sse_stream_history_capacity.txtar` | VS (pinned) | commit 1f6c4cd | 2026-08-11 |
| S35 | Google — AIP-151 Long-running operations | `https://google.aip.dev/151` | VD | Approved, last updated 2019-07-25 | 2026-08-11 |
| S36 | Google — AIP-127 HTTP and gRPC transcoding | `https://google.aip.dev/127` | VD | — | 2026-08-11 |
| S37 | Microsoft — Azure REST API Guidelines | `https://github.com/microsoft/api-guidelines/blob/vNext/azure/Guidelines.md` | VD (**moving URL** — branch `vNext`) | — | 2026-08-11 |
| S38 | Zalando — RESTful API Guidelines, events chapter | `https://github.com/zalando/restful-api-guidelines/blob/main/chapters/events.adoc` | VD (**moving URL** — branch `main`) | — | 2026-08-11 |
| S39 | Stripe — Meter event streams | `https://docs.stripe.com/api/v2/billing-meter-stream` | VD (living) | — | 2026-08-11 |
| S40 | GitHub — REST endpoints for events (`X-Poll-Interval`) | `https://docs.github.com/en/rest/activity/events` | VD (living) | — | 2026-08-11 |
| S41 | Shopify — Bulk operation queries | `https://shopify.dev/docs/api/usage/bulk-operations/queries` | VD (living) | — | 2026-08-11 |
| S42 | Twilio — Media Streams WebSocket messages | `https://www.twilio.com/docs/voice/media-streams/websocket-messages` | VD (living) | — | 2026-08-11 |
| S43 | AWS — Bedrock `InvokeModelWithResponseStream` | `https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_InvokeModelWithResponseStream.html` | VD (living) | — | 2026-08-11 |
| S44 | AWS — Transcribe `TranscriptResultStream` | `https://docs.aws.amazon.com/transcribe/latest/APIReference/API_streaming_TranscriptResultStream.html` | VD (living) | — | 2026-08-11 |
| S45 | AWS — Lambda `InvokeWithResponseStreamCompleteEvent` | `https://docs.aws.amazon.com/lambda/latest/api/API_InvokeWithResponseStreamCompleteEvent.html` | VD (living) | — | 2026-08-11 |
| S46 | AWS — DynamoDB Streams `GetShardIterator` | `https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_streams_GetShardIterator.html` | VD (living) | — | 2026-08-11 |
| S47 | Microsoft — GPT Realtime how-to (`expires_at`) | `https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/realtime-audio` | VD (living) | page dated 2026-05-13 | 2026-08-11 |
| S48 | OpenAI Developer Community — `session_expired` error shape | `https://community.openai.com/t/realtime-api-session-expired/975036` | SD | — | 2026-08-11 |
| S49 | Temporal — Sending Signals, Queries & Updates | `https://docs.temporal.io/sending-messages` | VD (living) | — | 2026-08-11 |

**Currency notes.**

- `[FACT]` S7: the IANA HTTP Status Code Registry still lists **RFC 2518** as the
  reference for `102 Processing`, although RFC 2518 was obsoleted by RFC 4918
  (2007), and RFC 4918 does not define `102`. Any argument built on `102` is
  therefore built on an obsoleted reference; §4 treats it as unusable.
- `[FACT]` S35 (AIP-151) carries a last-updated date of **2019-07-25**, which
  predates the SSE-based agent APIs surveyed here by five to six years. Its
  prohibition on streaming LRO responses is current *as published guidance* but
  is not current *as a description of the field*.
- S12, S37, S38 could not be pinned to a release tag that independently
  versions the document in question. They are cited as moving URLs and are not
  load-bearing for any recommendation.

---

## 3. The taxonomy

Five distinct shapes are shipped. The discriminating question for the last
column — how the client tells "delivery ended" from "the work failed" — is
answered differently in each.

| # | Pattern | Who ships it | Exact wire form | How the client distinguishes it from failure |
| --- | --- | --- | --- | --- |
| **T1** | **Distinct non-failure terminal event type** | OpenAI Responses API (`response.incomplete`, alongside `response.completed`, `response.failed`, `error`) · A2A **v0.3.0** (`final: true` on a status update, decoupled from task state) | S29: `{"type":"response.incomplete","response":{…,"status":"incomplete","incomplete_details":{"reason":"max_tokens"}},"sequence_number":N}` · S25: `bool final = 4; // Whether this is the last status update expected for this task.` | By the event's **type name**. The client dispatches on `response.incomplete` versus `response.failed`. Note: OpenAI's `incomplete` means *the work ended incomplete*, not *the work continues* — the two are different claims. |
| **T2** | **Error-typed message carrying a distinguishing reason code, meaning "delivery ended, state preserved"** | Kubernetes watch (`ERROR` event carrying a `Status` with `code: 410`, `reason: Expired`) · Replicate Cog (`event: error` with `SSE stream replay truncated`) | S15–S17: watch event type `ERROR`, object `metav1.Status{Code: 410, Reason: "Expired"}` · S34: `event: error` … `SSE stream replay truncated` … `"skipped":3` | By the **reason/code inside the error**, not by the message type. `client-go` branches on `IsResourceExpired` and logs "Watch closed" at V(4); everything else goes to "Failed to watch" (S18). |
| **T3** | **Terminal message carrying the work's *current* state plus a continuation handle** | Temporal (`stage` = most advanced lifecycle stage reached; `UNSPECIFIED` = wait limit hit; `update_ref` "Never null") · A2A **v1.0.1** (stream closes on "a terminal **or interrupted** state") | S28: `UpdateWorkflowExecutionResponse{ update_ref, outcome (unset if incomplete), stage }` · S23: "until the task reaches a terminal or interrupted state, at which point the stream closes" | By reading the **state value**, which is drawn from one vocabulary spanning terminal and non-terminal values. A non-terminal value means the work continues; the handle says where. |
| **T4** | **Advance warning message before a close that is not a failure** | Google Gemini Live (`GoAway` with `timeLeft`, plus `SessionResumptionUpdate` handles) — **WebSocket transport** | S32: "The server sends a GoAway message that signals that the current connection will soon be terminated. This message includes the timeLeft, indicating the remaining time." | By receiving a **dedicated pre-close message type**; the disconnection that follows is expected rather than diagnosed. |
| **T5** | **Silent close plus a documented reconnect/continue rule, with a per-frame checkpoint** | AWS Kinesis `SubscribeToShard` (5-minute cap, `ContinuationSequenceNumber` on every event) · Kubernetes watch timeout (randomized close, resume from last `resourceVersion`, `BOOKMARK` to advance it cheaply) · WHATWG SSE (reconnect is the default on any close; `204` is the only way to stop it) | S21: "for up to 5 minutes, after which time you need to call `SubscribeToShard` again to renew the subscription" · S22: "Use this as `SequenceNumber` in the next call … Use `ContinuationSequenceNumber` for checkpointing" · S19: `timeout = minRequestTimeout * (rand.Float64() + 1.0)` · S4: "a client can be told to stop reconnecting using the HTTP 204 No Content response code" | **It does not distinguish them on the wire at all.** The client is told out of band, by documentation, that a close at the limit is normal, and is given a position to resume from. Truncation and normal end are the same bytes. |
| **T0** | **Nothing — the error channel means failure, full stop** | Anthropic Messages API · AWS Bedrock `InvokeModelWithResponseStream` · AWS Transcribe streaming · gRPC status codes | S31: `event: error` / `{"type":"error","error":{"type":"overloaded_error",…}}`; terminal success event is `message_stop` · S43/S44: every non-data member of the event stream is an exception type · S9: `DEADLINE_EXCEEDED` | It cannot. gRPC states the cost explicitly: `DEADLINE_EXCEEDED` "may be returned even if the operation has completed successfully" (S9). |

`[INFERENCE]` **The organizing axis.** T0 and T2 use the *same* wire construct —
an error-typed in-band message — for opposite meanings. What separates them is
not the message but the system: **does the work have an addressable identity
beyond the stream?** Anthropic's message has none (there is no message resource
to poll; recovery is re-prompting, S31), so error-means-failure is correct for
it. Kubernetes' cluster state, Cog's prediction, OpenAI's stored response and
Temporal's workflow all do, so their streams are delivery channels over a
resource that outlives them, and the resource is authoritative. **This standard
already contains that discriminator twice** — `R13.9`'s "one capability, one
identity" applicability condition, and `R13.10`'s "a view over a retained
artifact" test. The fix therefore does not need a new concept; it needs the
existing condition applied to `R12.10`.

---

## 4. Standards layer — the verified negative

Each row records what was searched for, the most relevant verbatim text, and a
verdict. A negative here is load-bearing, so the search targets are recorded,
not only the conclusion.

| Source | Searched for | Verbatim | Verdict |
| --- | --- | --- | --- |
| RFC 9112 §8 (S1) | A client-facing reading of an incomplete transfer | "A message body that uses the chunked transfer coding is incomplete if the zero-sized chunk that terminates the encoding has not been received." — "A client that receives an incomplete response message, which can occur when a connection is closed prematurely or when decoding a supposedly chunked transfer coding fails, MUST record the message as incomplete." | **Does not define it, and points the other way.** The protocol's own reading of a premature close is "incomplete", which is the posture `R12.10` already takes. This *supports* the truncation half of `R12.10` that this report proposes to keep. |
| RFC 9110 §15.3.3 (S2) | Whether `202` covers the case | "the request has been accepted for processing, but the processing has not been completed. The request might or might not eventually be acted upon" | **Silent on the case.** `202` is a complete response to a request whose *work* has not finished; it says nothing about a transfer ending mid-flight. `R13.1` already forbids a streaming `202`. |
| RFC 9457 (S3) | A non-failure problem document | "This document defines a 'problem detail' to carry machine-readable details of **errors** in HTTP response content." | **Explicitly does not.** Problem details are scoped to errors. A "delivery ended normally" problem document would be an off-label use. |
| WHATWG HTML SSE (S4) | Whether a server close can mean "reconnect, nothing is wrong" | "Clients will reconnect if the connection is closed; a client can be told to stop reconnecting using the HTTP 204 No Content response code." — reconnection is the default path; "Once the user agent has failed the connection, it does not attempt to reconnect." | **Closest positive analogue, and still not a vocabulary.** SSE's *default* reading of a close is benign-and-retry, which is the opposite of `R12.10`'s default reading. But the format carries no field distinguishing "ended at a limit" from "died", and `retry:` only sets a timer. The transport says nothing about server-side work. |
| RFC 9113 §6.8 (S5) | Whether `GOAWAY` carries application meaning | "Activity on streams numbered lower than or equal to the last stream identifier might still complete successfully." Reattempting such requests "is not possible, except for idempotent actions." | **Transport-level only.** Real "might still be running" language exists, but it is guidance about connection draining and retry safety, not a signal an application client reads to classify an outcome. |
| gRPC `PROTOCOL-HTTP2.md` (S10) | Application-visible drain semantics | "Clients are free to continue working with the already accepted streams until they complete or the connection is terminated." | **Does not define it.** Same conclusion as S5 at the RPC layer. |
| gRPC `statuscodes.md` (S9) | A status meaning "transport ended, work continues" | `DEADLINE_EXCEEDED`: "The deadline expired before the operation could complete. For operations that change the state of the system, this error may be returned even if the operation has completed successfully." `UNAVAILABLE`: "most likely a transient condition, which can be corrected by retrying with a backoff." `OK`: "Not an error; returned on success." | **Explicitly does not — and documents the resulting ambiguity.** The nearest code concedes it cannot distinguish the two. There is no `OK`-with-incomplete-results status: a server-streaming RPC that ends early ends with a status like any other, and nothing in the status set says "partial". |
| CloudEvents v1.0.2 core (S11) | Stream/subscription termination attribute or type | Terms "terminat", "disconnect", "end of stream" do not occur in the core specification. | **Explicitly does not.** |
| CloudEvents Subscriptions (S12) | Termination semantics for a live subscription | Only a **Delete** operation ("removes the identified subscription") and transport-specific session-lifetime notes. | **Explicitly does not.** Deleting a subscription is not a soft delivery-ended signal. |
| AsyncAPI v3.0.0 (S13) | Channel/stream lifecycle vocabulary | Only the Operation Reply Object for request/reply is defined. | **Explicitly does not.** |
| RFC 6202 §5.5 / §2.1 (S6) | What a server sends when a long poll times out with no data | Warns the client "might receive a 408 Request Timeout answer from the server or a 504 Gateway Timeout answer from a proxy"; the server "responds to a request only when a particular event, status, or timeout has occurred." | **Explicitly does not.** It names the practitioner hazard — the timeout surfaces as an HTTP *error* code — and standardizes no alternative. |
| RFC 2518 §10.1 + IANA registry (S7) | `102 Processing` as an in-flight signal | "An interim response used to inform the client that the server has accepted the complete request, but has not yet completed it." | **Not usable.** `102` is a `1xx` interim response sent *before* the final response on a still-open transfer, so it cannot end one; and its registry reference is an obsoleted RFC that its own successor dropped. |
| RFC 8297 (S8) | `103 Early Hints` | "indicates to the client that the server is likely to send a final response with the header fields included in the informational response." | **Not usable**, same structural reason as `102`. |

`[FACT]` **Verdict: verified negative.** No published standard defines a
client-facing vocabulary meaning "the transfer ended but the work did not."

`[INFERENCE]` This is decisive in a specific way. It removes "adopt the existing
standard vocabulary" from the option set, so every remaining option is a
`[POLICY]` choice; and it means the standard cannot be accused of departing from
a protocol requirement whichever shape it picks. What remains to adjudicate is
comparative practice and internal consistency.

---

## 5. Per-system findings, with exact wire forms

### 5.1 Kubernetes watch — the clearest "delivery ended, state preserved" signal

`[FACT]` **Event vocabulary** (S15): `ADDED`, `MODIFIED`, `DELETED`, `BOOKMARK`,
`ERROR`. The `Event.Object` comment states what each carries, including:
"If Type is Bookmark: the object (instance of a type being watched) where only
ResourceVersion field is set. On successful restart of watch from a bookmark
resourceVersion, client is guaranteed to not get repeat event nor miss any
events."

`[FACT]` **The after-commit rule, shipped.** `cacher.go` (S16) carries this
comment at the point where a watch that cannot be served from history is turned
into a watcher that emits one event:

> To match the uncached watch implementation, once we have passed
> authn/authz/admission, and successfully parsed a resource version, other
> errors must fail with a watch event of type ERROR, rather than a directly
> returned error.

`[COMPARATIVE]` That is `R13.7` — an error raised after the response status is
committed is delivered in-band — arrived at independently by the field's
reference implementation. It is corroboration for `R13.7`, not a problem for it.

`[FACT]` **The wire construct.** `newErrWatcher` (S16) builds
`watch.Event{Type: watch.Error}` whose `Object` is the `*metav1.Status` from the
underlying `StatusError`. For a history-window miss the underlying error is
`errors.NewResourceExpired(…)`, which S17 defines as:

```go
func NewResourceExpired(message string) *StatusError {
	return &StatusError{metav1.Status{
		Status:  metav1.StatusFailure,
		Code:    http.StatusGone,      // 410
		Reason:  metav1.StatusReasonExpired,
		Message: message,
	}}
}
```

So the client receives, inside a `200 OK` stream, a JSON document of the form
`{"type":"ERROR","object":{"kind":"Status","status":"Failure","code":410,"reason":"Expired","message":"too old resource version: …"}}`.
`[INFERENCE]` The literal JSON above is assembled from S15's event encoding and
S17's `Status` construction rather than quoted from a single document; the field
values are quoted, the assembly is mine.

`[FACT]` **How the client distinguishes it from failure** (S18). `client-go`'s
`Reflector` converts the `ERROR` event's object back into an error
(`apierrors.FromObject(event.Object)`), then branches:

```go
switch {
case isExpiredError(err):
	// … first try to LIST with setting RV to resource version of last observed object.
	logger.V(4).Info("Watch closed", …)
case apierrors.IsTooManyRequests(err):
	logger.V(2).Info("Watch returned 429 - backing off", …)
default:
	utilruntime.HandleErrorWithContext(ctx, err, "Failed to watch", …)
}
```

`[COMPARATIVE]` This is the whole finding in one construct: an **error-typed
message whose code makes it a non-failure**. `Expired` is logged as "Watch
closed" at verbosity 4 and triggers a re-list; anything unrecognized is "Failed
to watch". A client that treated every `ERROR` event as failure would still be
*correct about the stream* and *wrong about the world* — the cluster state is
intact and the correct action is to re-list, not to report an error.

`[FACT]` **The documented client contract** (S14): "When the requested **watch**
operations fail because the historical version of that resource is not
available, clients must handle the case by recognizing the status code `410
Gone`, clearing their local cache, performing a new **get** or **list**
operation, and starting the **watch** from the `resourceVersion` that was
returned." Retention: "Clusters using etcd 3 preserve changes in the last 5
minutes by default."

`[FACT]` **`BOOKMARK` is a checkpoint, not a terminal signal** (S14, S20): "It is
a special kind of event to mark that all changes up to a given `resourceVersion`
the client is requesting have already been sent." It is opt-in
(`allowWatchBookmarks=true`) and unreliable by contract: "you shouldn't assume
bookmarks are returned at any specific interval, nor can clients assume that the
API server will send any `BOOKMARK` event even when requested."

`[FACT]` **The duration cap is a silent close** (S19). The watch handler
computes `timeout = time.Duration(float64(minRequestTimeout) * (rand.Float64() +
1.0))` when the client supplied no `timeoutSeconds`, and the stream simply ends.
No terminal event marks it. `client-go` treats `io.EOF` as "watch closed
normally" (S18) with no log at all.

`[COMPARATIVE]` So Kubernetes ships **two different endings for two different
reasons**: the duration cap is T5 (silent close, resume from last
`resourceVersion`), and the history-window miss is T2 (error-typed message with
a distinguishing reason). It does not use one mechanism for both.

### 5.2 AWS Kinesis `SubscribeToShard` — the hard cap with a per-frame checkpoint

`[FACT]` (S21) "your consumer starts receiving events of type
`SubscribeToShardEvent` over the HTTP/2 connection for up to 5 minutes, after
which time you need to call `SubscribeToShard` again to renew the subscription
if you want to continue to receive records."

`[FACT]` (S21) The response `EventStream` union contains exactly one data member
(`SubscribeToShardEvent`) and eight exception members
(`InternalFailureException`, `KMSAccessDeniedException`, `KMSDisabledException`,
`KMSInvalidStateException`, `KMSNotFoundException`, `KMSOptInRequired`,
`KMSThrottlingException`, `ResourceInUseException`, `ResourceNotFoundException`).
`[FACT]` **None of them is emitted at the 5-minute limit** — the limit is not
listed as an error condition anywhere on S21.

`[FACT]` (S22) `ContinuationSequenceNumber` is **required on every event**: "Use
this as `SequenceNumber` in the next call to `SubscribeToShard`, with
`StartingPosition` set to `AT_SEQUENCE_NUMBER` or `AFTER_SEQUENCE_NUMBER`. Use
`ContinuationSequenceNumber` for checkpointing because it captures your shard
progress even when no data is written to the shard."

`[COMPARATIVE]` Pure T5. The signal is the *absence* of a signal plus a
documented rule and a continuation position on every frame. `[INFERENCE]` This
is the shape `R13.10`'s `stream_position` already models; what Kinesis adds is
that the position is mandatory on every event precisely so that a close at the
cap needs no message.

### 5.3 A2A Protocol — the one protocol that shipped a standalone end marker and then removed it

`[FACT]` **v0.3.0 had a `final` flag decoupled from state** (S25):

```proto
message TaskStatusUpdateEvent {
  string task_id = 1;
  string context_id = 2;
  TaskStatus status = 3;
  // Whether this is the last status update expected for this task.
  bool final = 4;
  google.protobuf.Struct metadata = 5;
}
```

`[FACT]` The v0.3.0 wire example (S26) ends a stream with
`"status":{"state":"completed",…},"final":true,"kind":"status-update"` followed
by "_(Server closes the SSE connection after the `final:true` event)._"

`[FACT]` **v0.3.0 defined a reattach method** (S26 §7.9, `tasks/resubscribe`):
"Allows a client to reconnect to an SSE stream for an ongoing task after a
previous connection (from `message/stream` or an earlier `tasks/resubscribe`)
was interrupted." And, importantly for `ST-026`: "The server's behavior
regarding events missed during the disconnection period (e.g., whether it
attempts to backfill some missed events or only sends new ones from the point of
resubscription) is implementation-dependent and not strictly defined by this
specification."

`[FACT]` **v1.0.1 removed `final`** (S24 — the field is gone from
`TaskStatusUpdateEvent`; S27 CHANGELOG entry "Remove redundant `final` field
from `TaskStatusUpdateEvent`"). The issue text (S27, issue #1308) states the
reasoning:

> Terminal states (COMPLETED, FAILED, CANCELLED, REJECTED) already indicate task
> completion, making the final field redundant. This creates consistency across
> streaming, polling, and push notification communication patterns.

`[FACT]` **v1.0.1 closes the stream on non-terminal states too** (S23, §"Design
principles"): "Streaming responses are simple, linearly ordered sequences: first
a `Task` (or single `Message`), then zero or more status or artifact update
events until the task reaches a terminal **or interrupted** state, at which point
the stream closes." The interrupted states are named elsewhere in S23 as
`TASK_STATE_INPUT_REQUIRED` and `TASK_STATE_AUTH_REQUIRED`.

`[FACT]` **v1.0.1's reattach carries a full snapshot first** (S23, §3.1.6
Subscribe to Task): "The operation MUST return a `Task` object as the first
event in the stream, representing the current state of the task at the time of
subscription. This prevents a potential loss of information between a call to
`GetTask` and calling `SubscribeToTask`." Subscribing to an already-terminal
task is an error: `UnsupportedOperationError`.

`[FACT]` (S23) Multiple concurrent streams per task are permitted: "An agent MAY
serve multiple concurrent streams to one or more clients for the same task."

`[COMPARATIVE]` A2A is the single most informative system in this survey because
it ran **both** candidate shapes and chose between them: T1 (`final`) in v0.3.0,
T3 (state-driven closure plus snapshot-on-reattach) in v1.0.1, with the removal
justified as redundancy once the state vocabulary carries the meaning.

`[INFERENCE]` **Scope of that inference.** A2A removed a *second* end marker
sitting on top of a state value that already implied the end. It did **not**
remove the terminal message itself, and it did not have `R13.6`'s other job —
distinguishing a normal end from a truncated connection — to discharge, because
A2A's closure condition is a state value the client can check. So S27 is
evidence against **adding a new frame type whose only content is "this is the
end"**; it is not evidence against ratified `R13.6`.

### 5.4 Temporal — "the most advanced stage reached", returned on success

`[FACT]` (S28) `UpdateWorkflowExecutionResponse`:

```proto
message UpdateWorkflowExecutionResponse {
    // Enough information for subsequent poll calls if needed. Never null.
    temporal.api.update.v1.UpdateRef update_ref = 1;
    // The outcome of the Update if and only if the Workflow Update
    // has completed. If this response is being returned before the Update has
    // completed then this field will not be set.
    temporal.api.update.v1.Outcome outcome = 2;
    // The most advanced lifecycle stage that the Update is known to have
    // reached … UNSPECIFIED will be returned if and only if the server's maximum wait
    // time was reached before the Update reached the stage specified in the
    // request WaitPolicy, and before the context deadline expired; clients may
    // may then retry the call as needed.
    temporal.api.enums.v1.UpdateWorkflowExecutionLifecycleStage stage = 3;
    …
}
```

The stage vocabulary is ordered `UNSPECIFIED < ADMITTED < ACCEPTED < COMPLETED`.
`PollWorkflowExecutionUpdateResponse` (S28) carries the identical three members
with the identical comments.

`[FACT]` (S28) Temporal's history long-poll uses the same "wait, then return
what you have" shape: `GetWorkflowExecutionHistoryRequest.wait_new_event` — "If
set to true, the RPC call will not resolve until there is a new event which
matches the `history_event_filter_type`, or a timeout is hit."

`[FACT]` **Second source, product documentation** (S49). The user-facing docs
describe the same model in the same terms: the stages a caller may wait for are
"Accepted - wait until the Worker is contacted, which ensures that the Update is
persisted" and "Completed - wait until the handler finishes and returns a
result", and starting an update with a desired stage "will give you a handle you
can use to track the Update, determine whether it was Accepted, and ultimately
get its result or an error." `[FACT]` S49 does **not** restate the
maximum-wait-time clause; that clause is carried by S28 alone (see §10).

`[COMPARATIVE]` This is the clearest T3 instance, and it is worth being precise
about *why* it is strong evidence. The wait limit being reached is reported:

- on a **success** response, not an error;
- by a **state value from the work's own lifecycle vocabulary**, which includes
  non-terminal members;
- alongside a **mandatory continuation handle** (`update_ref`, "Never null");
- with an explicit client instruction ("clients may then retry the call as
  needed").

`[INFERENCE]` Map that onto §13: the terminal frame is the success response, the
state value is `operation_state`, and the continuation handle is
`operation_id`/`operation_url`, which `R13.9` already requires on every frame.
The only thing this standard lacks is permission for `operation_state` to hold a
non-terminal value, plus a reason naming why delivery stopped.

### 5.5 OpenAI — four terminal event types, and a real detach/reattach

`[FACT]` **Four terminal-ish stream events** (S29, `openapi.yaml`):

| Schema | `type` value | Description (verbatim) |
| --- | --- | --- |
| `ResponseCompletedEvent` | `response.completed` | "Emitted when the model response is complete." |
| `ResponseIncompleteEvent` | `response.incomplete` | "An event that is emitted when a response finishes as incomplete." |
| `ResponseFailedEvent` | `response.failed` | "An event that is emitted when a response fails." |
| `ResponseErrorEvent` | `error` | "Emitted when an error occurs." |

Every one of them carries a required `sequence_number` ("The sequence number for
this event"). The first three embed the full `Response` object; the fourth
carries `code`, `message`, `param`.

`[FACT]` (S29) The `Response.status` enum is `completed`, `failed`,
`in_progress`, `cancelled`, `queued`, `incomplete` — **six values, three of them
non-terminal** — and the `response.incomplete` example carries
`"status":"incomplete"` with `"incomplete_details":{"reason":"max_tokens"}`.

`[FACT]` **Background mode and reattachment** (S30): "To start response
generation in the background, make an API request with `background` set to
`true`"; "Keep polling while the request is in the queued or in_progress state.
When it leaves these states, it has reached a final (terminal) state"; the
response "continues running and you can reconnect" if the connection drops; and
resumption uses "a 'cursor' corresponding to the `sequence_number`" from each
event. `[FACT]` (S29) The retrieve-a-response operation takes query parameters
`stream` and `starting_after` — the latter documented as "The sequence number of
the event after which to start streaming."

`[FACT]` (S30) Retention: "Response data is temporarily stored to disk for
roughly 10 minutes to enable asynchronous execution and polling." Constraint:
"You can only start a new stream from a background response if you created it
with `stream=true`."

`[COMPARATIVE]` OpenAI is the closest thing in the survey to §13-plus-§10 as
this standard composes them: a stream and an addressable response object with
one identity, a per-frame monotonic position, and a documented resume. `[FACT]`
What it does **not** have is any event emitted when the *delivery* ends while
the response is still running — a dropped connection produces nothing, and the
client learns the state by re-fetching. So OpenAI answers the resumption half of
the question and leaves the signalling half unanswered.

`[INFERENCE]` `response.incomplete` is worth separating carefully from this
report's question. It reports that *the work* finished in a third way (hit the
token ceiling, hit a content filter) — not that delivery stopped while the work
continued. It is evidence that the field is willing to ship **more than three
terminal outcomes**, which bears on whether `R10.1`'s terminal vocabulary should
be treated as closed at `succeeded`/`failed`/`canceled`; it is not evidence for
T1 as an answer to this question.

### 5.6 Anthropic — the pure T0 case, and why it is correct there

`[FACT]` (S31) The stream's event flow is `message_start`, then per-content-block
`content_block_start` / `content_block_delta` / `content_block_stop`, then one or
more `message_delta`, then "A final `message_stop` event." `ping` events "may
also" appear anywhere.

`[FACT]` (S31) The `error` event: "The API may occasionally send errors in the
event stream. For example, during periods of high usage, you may receive an
`overloaded_error`, which would normally correspond to an HTTP 529 in a
non-streaming context":

```
event: error
data: {"type": "error", "error": {"type": "overloaded_error", "message": "Overloaded"}}
```

`[FACT]` (S31) Unknown-event tolerance is documented: "new event types may be
added, and your code should handle unknown event types gracefully."

`[FACT]` (S31) There is **no server-side resumption**. Recovery is described as
"Error recovery": capture the partial response, construct a *new* request that
continues from it, and resume streaming — a prompt-level technique, not a
protocol one. `[FACT]` No maximum stream duration and no in-flight revocation
posture is published on this page.

`[COMPARATIVE]` `[INFERENCE]` Anthropic is the case where `R12.10` as written is
exactly right: there is no message resource to consult, an `error` frame is the
work failing, and no other reading is available. This is why the recommendation
in §8 **scopes** `R12.10` on the existence of an operation resource rather than
weakening it globally.

### 5.7 Google Gemini Live — advance warning plus resumption handle (WebSocket)

`[FACT]` (S32) "The server sends a GoAway message that signals that the current
connection will soon be terminated. This message includes the timeLeft,
indicating the remaining time." Session resumption uses
`SessionResumptionUpdate` messages carrying `new_handle` and `resumable`, passed
back as `SessionResumptionConfig.handle` on the next connection. Duration:
"Without compression, audio-only sessions are limited to 15 minutes, and
audio-video sessions are limited to 2 minutes."

`[POLICY]` **Caveat, applied deliberately.** The Live API is a **WebSocket**
surface, which §1.2 puts outside this standard. It is cited here as
comparative-with-caveat: it demonstrates that a vendor facing a hard session
ceiling chose to send an advance warning plus a resumption handle rather than
either an error or a silent close. It cannot ground an HTTP rule on its own, and
no recommendation below rests on it.

`[COMPARATIVE]` The same caveat applies to the OpenAI/Azure Realtime
`session_expired` case (S47, S48): also WebSocket, and see §10 for why its exact
error string is recorded as unverified.

### 5.8 Replicate Cog — an `error` frame for a delivery-only condition

`[FACT]` (S33) Frame vocabulary: `start`, `output`, `log`, `metric`, and
`completed` — "The prediction reached a terminal state. The payload is the final
prediction object, with `status` set to `succeeded`, `failed`, or `canceled`."

`[COMPARATIVE]` `completed` is `R13.6`'s terminal frame and its `status` member
is `R13.9`'s `operation_state`, with the same three-value vocabulary the
standard names.

`[FACT]` (S33) Attach-to-in-flight and the gap signal: "If the prediction is
still running, the server returns a stream for the existing prediction instead
of creating a duplicate prediction. If earlier events have been dropped from the
replay buffer, the stream emits an `error` event and closes. The replay buffer
keeps the most recent 1024 events by default. Set
`COG_STREAM_HISTORY_CAPACITY` to change this limit, or set it to `0` to disable
replay history while keeping live streaming enabled."

`[FACT]` (S34) The integration test fixes the wire form and the intent. Its
comment reads "Capacity 2 should drop older replay events and close with an
error for late subscribers", and the assertions are `event: error`, the message
`SSE stream replay truncated`, and a payload member `"skipped":3`.

`[COMPARATIVE]` This is T2 in a system with an operation resource. The
prediction is *not* failed — it is running, and it will reach `succeeded` — but
the client's *delivery* cannot be made whole, so the API ends the stream with an
`error` frame carrying a code that names the delivery condition and a count of
what was skipped.

`[INFERENCE]` Under `R12.10` a conforming client receiving Cog's frame must
"treat the operation as failed." The prediction then succeeds. This is not a
hypothetical collision; it is the only shipped implementation of the
stream-plus-idempotency intersection that `ST-026` run b identified.

### 5.9 The eight reference APIs

Non-participation is a dated finding and is recorded as one.

| Reference | Long-lived HTTP response surface? | Delivery-ended-but-work-continues signal | Evidence |
| --- | --- | --- | --- |
| **Stripe** | `[FACT]` None located | — | S39: the only "stream" surface is `POST /v2/billing/meter_event_stream`, a high-throughput POST with short-lived session tokens, not a held-open response. Searched `docs.stripe.com` for SSE / streaming / long-lived connections, 2026-08-11 |
| **GitHub** | `[FACT]` None located on the public REST/GraphQL surface | Client-driven polling cadence only | S40: `X-Poll-Interval: 60` on the events API. Actions logs are a 302 to a 1-minute presigned archive URL, not a stream. GraphQL subscriptions are not offered |
| **Google / AIP** | `[FACT]` Explicitly excluded for LRO | — | S35 (AIP-151): "The response **must not** be a streaming response." S36 (AIP-127) describes server-streaming RPCs as a separate mechanism, unrelated to LRO continuation |
| **Microsoft / Azure** | `[FACT]` Not addressed in the guidelines | — | S37: searched `azure/Guidelines.md` for "streaming", "server-sent", "SSE", "chunked" — no matches; the document covers LRO only, and motivates it by "services not wanting to maintain long-lived connections". Azure's Realtime surface is WebSocket (S47) |
| **Twilio** | `[FACT]` None (WebSocket and webhook only) | Media Streams `stop` message (WebSocket) | S42 |
| **Shopify** | `[FACT]` None held open | Terminal `status` plus `partialDataUrl` on a finished bulk operation | S41 — assigned after the job ends, not mid-flight |
| **Zalando** | `[FACT]` Not addressed in the guidelines | — | S38: searched the guidelines for "streaming", "server-sent events", "long polling" — no rule found |
| **AWS** | Yes, several | `[FACT]` No non-failure ending in any of them except Kinesis' silent cap | S43 (Bedrock: every non-`chunk` member is an exception — `modelStreamErrorException`, `modelTimeoutException`, `throttlingException`, …); S44 (Transcribe: every non-`TranscriptEvent` member is an exception, including `LimitExceededException` for exceeding the maximum session duration); S45 (Lambda `InvokeWithResponseStreamCompleteEvent` carries `ErrorCode`/`ErrorDetails`); S46 (DynamoDB Streams is poll-plus-shard-iterator, the iterator being the continuation position); S21/S22 (Kinesis, §5.2) |

`[COMPARATIVE]` **Six of the eight do not participate at all.** The two that do
— AWS and, marginally, Google via AIP — split: AWS routes every named ending
except the Kinesis cap through an exception type, and AIP forbids the overlap
this standard's `R13.9` exists to govern.

`[INFERENCE]` **AIP-151 is the standing dissent against `R13.9`'s premise and
should be recorded as such rather than argued away.** `R13.9`'s provenance
already notes "one shipped exemplar against one contrary guideline"; S35 is that
guideline, and it is unchanged since 2019. Its practical effect is that Google's
own answer to this report's question is "do not create the situation" — which
the field has since ignored, since OpenAI, Replicate, A2A and Temporal all
expose exactly this overlap.

---

## 6. The `R12.10` / `R13.9` constraint analysis

### 6.1 What the two rules say, and precisely where each breaks

`R12.10` (client obligations): "On an `error` frame it MUST treat the operation
as failed and MUST branch on the frame's `code` and `type` rather than on the
response status."

`R13.9` (one identity): the terminal frame carries `operation_state`, "drawn
from the vocabulary `R10.1` requires the operation resource to document" — and
§1.10 defines that member as "The operation's **terminal-state** value … the
member `R13.9` compares across the two channels."

| Case | What ends | Operation's state at that moment | Blocked by |
| --- | --- | --- | --- |
| Documented duration cap reached (`ST-024`) | Delivery | running / in progress | `R13.9` — no terminal state exists to report; `R13.7`+`R12.10` — routing it through `error` would call a published, expected limit a failure |
| Authorizing credential expired (`ST-025`) | Delivery | running | `R12.10` — the decided shape *is* an `error` frame, so the client must read the still-running operation as failed |
| Replay/resumption cannot deliver missed frames (`ST-026`) | Delivery of a *complete* result | possibly already `succeeded` | `R12.10` — same; the decided shape is an `error` frame, and the operation may already have succeeded |

`[INFERENCE]` The three Phase 8 rulings currently resolve one problem **three
different ways**: D2 exempts the terminal frame from `operation_state`;
`ST-025` sends an `error` frame; `ST-026` sends an `error` frame. Two of the
three then collide with `R12.10` and were not recorded as doing so. The
inconsistency is internal to the decided set, not something this report
introduces.

### 6.2 Does the field suggest `R12.10` should be scoped? Yes, and narrowly

`[COMPARATIVE]` Two shipped systems use an error-typed in-band message to mean
"delivery ended; the underlying state is intact":

- Kubernetes: `ERROR` + `Status{code:410, reason:Expired}`, and the reference
  client's *documented action* is to re-list and continue, not to report failure
  (S14, S16–S18).
- Replicate Cog: `event: error` with `SSE stream replay truncated` while the
  prediction continues to a terminal state (S33, S34).

`[COMPARATIVE]` And one shipped system uses it to mean "the work failed", with
no other reading available: Anthropic (S31), together with AWS Bedrock and
Transcribe (S43, S44), where every non-data member of the event stream really is
a failure of the work.

`[INFERENCE]` The two populations are separated by the condition `R13.9` already
uses: whether the work has an addressable identity beyond the stream. So the
scoping writes itself and does not require a new concept:

> Where the stream has an operation resource, an `error` frame ends the
> **delivery**; what happened to the **work** is read from `operation_state` and,
> authoritatively, from the operation resource. Where there is no operation
> resource, an `error` frame means the work failed, unchanged.

`[POLICY]` This keeps every clause of `R12.10` that RFC 9112 §8 grounds — a
close without the terminal frame is truncation, partial content is not a
complete result — and changes only the sentence that asserts a fact about the
*work* from evidence about the *stream*.

### 6.3 Does the field suggest `R13.9` should be scoped? Yes — toward current state, not exemption

`[COMPARATIVE]` The two systems whose streams sit over identified work both
report the state actually reached, from a vocabulary that spans terminal and
non-terminal values:

- Temporal: `stage` = "the most advanced lifecycle stage that the Update is known
  to have reached", `UNSPECIFIED` when the server's wait limit came first, plus
  `update_ref` "Never null" (S28).
- A2A v1.0.1: the stream closes when the task "reaches a terminal **or
  interrupted** state" (S23), the interrupted states being non-terminal.

`[COMPARATIVE]` And OpenAI's stored `Response.status` enum mixes terminal and
non-terminal values in one member for exactly this reason (S29).

`[INFERENCE]` **Reconciling with the D2 decision.** D2 declined "relaxing
`R13.9` to carry the operation's *current* state" because it "would weaken the
cross-channel comparison in every case to accommodate one, and that comparison
being between **terminal** states is the whole point of it." That objection is
sound against an *unconditional* relaxation. It does not reach a relaxation
conditioned on the divergence:

- when the operation **is** terminal when the stream ends — the overwhelmingly
  common case, and every case `R13.9` was written for — the terminal frame still
  MUST carry a terminal value, and the comparison is exactly as strong as today;
- only when the stream ends **first** does the frame carry a current value, and
  in that case there is no terminal state in existence to compare against, so
  nothing is weakened. The alternative on the table is D2's exemption, which
  removes the member entirely and therefore carries strictly *less* information
  than a current-state value would.

`[INFERENCE]` The comparison D2 protects is also protected by a mechanical test
the client can still run: a value from the documented vocabulary that is not in
the documented terminal subset means "not finished — go to the operation
resource." That is a stronger client contract than an absent member, which is
indistinguishable from a non-conforming server that forgot to send it.

`[POLICY]` **Recommendation to the walk: revisit D2's repair choice.** Both
halves of D2 survive — the stream ends with `R13.6`'s terminal frame, and the
client continues through the operation resource — but the terminal frame should
carry the current state and a reason rather than be exempted. The task brief
authorizes this conclusion, and the version consequence is the same as the one
D2 already accepted: "scoping `R13.9` in a named case is a relaxation, which the
amendment rule puts at MINOR." The `R12.10` scoping is the same class for the
same reason.

### 6.4 Has any system had this standard's exact self-imposed constraint?

`[FACT]` **Yes: gRPC**, and it did not work around it. A gRPC RPC's status is
the outcome of the RPC, so `DEADLINE_EXCEEDED` is both "the transport gave up"
and "the call failed" — and the specification concedes the consequence rather
than fixing it: "For operations that change the state of the system, this error
may be returned even if the operation has completed successfully" (S9). No
status distinguishes the two; the retry guidance explicitly refuses to supply a
list, saying "individual applications must make their own determination as to
which status codes should cause an RPC to be retried."

`[FACT]` **Partially: A2A**, which had a marker whose meaning was "the stream is
over" independent of task state, and removed it because state made it redundant
(S27) — a resolution in the opposite direction from gRPC's, available only
because A2A's state vocabulary contains non-terminal values that a closing
stream can legitimately report.

`[INFERENCE]` The pair is the whole argument. Where the ending signal is welded
to the work's outcome (gRPC status, and `R12.10`+`R13.9` as ratified), the
ambiguity is permanent and is paid by every client. Where the ending signal
reports a state from a vocabulary that includes "not finished" (A2A, Temporal),
the ambiguity does not arise and no extra signal is needed.

---

## 7. Evidence for and against each candidate shape

### Shape A — a distinct non-failure terminal frame type

*Example: a reserved frame type such as `incomplete` or `detached`, sibling to
`error`, ending the stream without asserting failure.*

**For.**

- `[COMPARATIVE]` OpenAI ships a distinct terminal event for a third outcome:
  `response.incomplete` alongside `response.completed` and `response.failed`
  (S29). The field is demonstrably willing to have more than two terminal event
  types.
- `[COMPARATIVE]` A2A v0.3.0 shipped exactly this (`final`, S25), and Gemini
  Live ships a dedicated pre-close message (`GoAway`, S32, WebSocket).
- `[INFERENCE]` It is the only shape that needs no change to any ratified rule's
  meaning — a new type is additive.

**Against.**

- `[FACT]` A2A **removed** its instance as "redundant" once terminal states
  carried the meaning (S27). The one protocol that ran the experiment reversed
  it.
- `[INFERENCE]` It collides with the standard's own compatibility machinery.
  §13.4 already records that renaming or adding terminal frame types is
  unclassified by `R9.4`, and that "every deployed client would ignore the
  unrecognized frame under `R12.10`, see no terminal frame, and report
  truncation on every success." A new terminal type introduced into a deployed
  API produces precisely that: existing clients ignore it (`R12.10`'s
  unknown-type tolerance), then report truncation. `ST-021`'s terminality freeze
  is the rule that names this hazard.
- `[INFERENCE]` `response.incomplete` is not actually this shape's evidence: it
  reports that the *work* ended a third way, not that the work continues (§5.5).
  Once that is subtracted, the shape's shipped support is one removed field and
  one WebSocket message.

**Verdict:** `[POLICY]` Decline. Weakest evidence, and it is the shape the
standard's own compatibility rules punish hardest.

### Shape B — reuse the `error` channel with a distinguishing `code`, and scope `R12.10`

**For.**

- `[FACT]` Two shipped implementations do exactly this — Kubernetes (S16–S18)
  and Cog (S33, S34) — and in the Kubernetes case the reference client's
  branch-on-reason is visible in source.
- `[INFERENCE]` It requires no new frame type, so nothing about terminality
  changes and `ST-021` is untouched. `R13.7` already mandates a `code`, and
  `R5.16` already requires it to be cataloged, so the distinguishing mechanism
  exists.
- `[INFERENCE]` It covers the two cases that genuinely *are* abnormal —
  credential expiry and an undeliverable replay — without stretching the word
  "error".

**Against.**

- `[INFERENCE]` It cannot cover the duration cap without contradicting
  `ST-024`'s own reasoning that "a close at a published limit is a normal end".
  Neither Kubernetes nor Kinesis uses an error for its cap; both close silently
  (S19, S21).
- `[INFERENCE]` `R13.7` requires the `error` frame's payload to be an RFC 9457
  problem object, and RFC 9457 defines problem details as carrying "details of
  **errors**" (S3). Using it for a normal end is an off-label use of a protocol
  document this standard cites as authority elsewhere.
- `[INFERENCE]` Scoping `R12.10` is necessary under this shape but is not
  *sufficient*: a client that learns "not a failure" still needs to know the
  state and where to continue, which the error channel does not carry
  (`R13.7` forbids `status`, and there is no state member on a problem object).

**Verdict:** `[POLICY]` Adopt the `R12.10` scoping this shape requires — it is
needed regardless, because `ST-025` and `ST-026` already decided on `error`
frames — but do not make the error channel the general answer.

### Shape C — the terminal frame carries the operation's *current* state plus a reason (scope `R13.9`)

**For.**

- `[FACT]` Temporal ships it precisely: current stage, non-error response,
  mandatory continuation handle, explicit retry instruction (S28).
- `[FACT]` A2A v1.0.1 ships the closure half: the stream closes on a terminal
  **or interrupted** state (S23), and reattachment leads with a full current-state
  snapshot (S23 §3.1.6).
- `[FACT]` A2A's removal of `final` is a positive argument for this shape, not
  just a negative one against Shape A: the stated reason is that state already
  carries the information (S27).
- `[COMPARATIVE]` OpenAI's stored-response `status` enum mixes terminal and
  non-terminal values in the single member a client reads (S29).
- `[INFERENCE]` It composes with what §13 already has: `operation_id` /
  `operation_url` on every frame make the continuation reachable, and `R13.6`'s
  terminal frame is already the carrier.
- `[INFERENCE]` It preserves truncation detection. A terminal frame still
  arrives, so `R12.10`'s suffix test still discriminates a normal end from a
  dropped connection — which the silent-close pattern (T5) does not.

**Against.**

- `[INFERENCE]` It re-means `operation_state`, which §1.10 currently defines as
  a terminal-state value. That is a real amendment to a ratified definition, not
  a clarification.
- `[INFERENCE]` D2 already declined it once, on the cross-channel-comparison
  argument (answered in §6.3, but the walk may weigh it differently).
- `[FACT]` No HTTP-native precedent. Temporal is gRPC, A2A is a protocol layered
  over SSE and gRPC. `[INFERENCE]` The shape is transport-independent, but the
  evidence base for it is not drawn from the HTTP APIs the standard governs.
- `[INFERENCE]` Clients must now handle a member that can hold a value outside
  the terminal subset. That is a real client-side cost, and it is the cost
  `R9.4`'s newly generalized open-enum classification exists to bound.

**Verdict:** `[POLICY]` Adopt as the primary answer, conditioned on the
divergence case.

### Shape D — leave it undefined

**For.**

- `[FACT]` Every published standard leaves it undefined (§4), and six of the
  eight reference APIs never encounter it (§5.9).
- `[FACT]` AIP-151 resolves it by prohibition — "The response must not be a
  streaming response" (S35) — which is a coherent position a standard may take.
- `[INFERENCE]` The standard has an established mechanism for deliberate
  silence: §13.4's register of recognized-and-not-yet-ruled interactions.

**Against.**

- `[FACT]` gRPC is the counterfactual and documents the cost in its own
  specification: the client cannot tell, and the retry decision is pushed onto
  every application (S9).
- `[INFERENCE]` This standard's position is **worse than silence**. `R12.10`
  does not decline to say what an `error` frame means; it *requires* the client
  to conclude the operation failed. Against Cog's shipped behavior that
  conclusion is simply false. Silence lets a client be careful; a rule makes it
  confidently wrong.
- `[INFERENCE]` Three ratified-or-decided rulings (`ST-024`, `ST-025`,
  `ST-026`) each create the situation. Leaving it undefined would mean shipping
  three rules whose conforming implementations cannot be described by the rest
  of the standard.

**Verdict:** `[POLICY]` Decline for the `R12.10` half — the collision is live
and must be resolved either way. `[POLICY]` Leaving the *reason vocabulary*
undefined (per-API, documented) is however correct: see §8.

---

## 8. Proposed rule text

Proposed identifier: **`ST-030`**, continuing Phase 8's sequence. It is a
two-part amendment plus one reserved member, not a new standalone rule.

### `ST-030a` — scope `R12.10`'s failure inference to the no-operation-resource case

**Target:** amendment to ratified `R12.10`. **Class:** `[POLICY]`, grounded in
`[COMPARATIVE]` evidence (S16–S18, S33, S34 for the delivery-only reading; S31,
S43, S44 for the failure reading). **Confidence: moderate-high** — the two
populations are cleanly separated by a condition the standard already uses;
what is uncertain is only the drafting, not the split.

> Replace `R12.10`'s sentence "On an `error` frame it MUST treat the operation
> as failed" with:
>
> On an `error` frame a client MUST treat the **stream** as ended and MUST NOT
> treat partial content as a complete result. Where the stream carries no
> operation identity (§1.10 `operation_id` / `operation_url`), the client MUST
> treat the operation as failed. Where the stream carries an operation identity,
> the client MUST determine the operation's state from the terminal frame's
> `operation_state` member and, authoritatively, from the operation resource
> (`R13.9`), and MUST NOT infer the operation's fate from the frame type alone.
> In both cases it MUST branch on the frame's `code` and `type` rather than on
> the response status, which is `200`.

Everything else in `R12.10` is unchanged, including the truncation obligation
that RFC 9112 §8 grounds (S1), the trailing-sentinel tolerance, the
unknown-type tolerance, the keep-alive prohibition, and the non-idempotent
replay prohibition.

### `ST-030b` — scope `R13.9` so the terminal frame may carry a current state when the stream ends first

**Target:** amendment to ratified `R13.9` and to §1.10's `operation_state` row.
**Class:** `[POLICY]`, grounded in `[COMPARATIVE]` evidence (S28, S23, S29).
**Confidence: moderate** — two strong precedents, neither of them an HTTP API,
and the amendment re-means a ratified member definition.

> Add to `R13.9`:
>
> Where the operation has reached a terminal state when the stream ends, the
> terminal frame MUST carry that terminal state in `operation_state`, unchanged.
> Where the **stream ends before the operation reaches a terminal state**, the
> terminal frame MUST instead carry the operation's current state in
> `operation_state` — drawn from the same vocabulary `R10.1` requires the
> operation resource to document — and MUST carry a documented
> `stream_end_reason` (§1.10) naming why delivery ended. The API MUST document
> which values of its `operation_state` vocabulary are terminal, so that a
> client can decide mechanically whether the operation finished. The operation
> resource remains authoritative in both cases.

> Amend §1.10's `operation_state` row: it is the operation's state value, drawn
> from the vocabulary `R10.1` requires the operation resource to document; on a
> terminal frame it is the terminal value where one exists, and the current
> value where the stream ended first.

### `ST-030c` — reserve `stream_end_reason`

**Target:** new §1.10 reserved-member row. **Class:** `[POLICY]`.
**Confidence: moderate** on requiring the member, **high** on *not*
standardizing its values.

> `stream_end_reason` — a terminal frame whose `operation_state` is non-terminal.
> A documented, machine-readable value naming why delivery ended while the
> operation continued. The API MUST document its full vocabulary; the vocabulary
> is an open enum under `R9.4`'s enum classification, so removing or renaming a
> value is a breaking change. This standard defines no values.

`[INFERENCE]` **Why no standard value set.** The field ships reason members but
no shared vocabulary: OpenAI's `incomplete_details.reason` takes
`max_output_tokens` / `content_filter` (S29); Kubernetes uses `Status.reason`
values such as `Expired` (S17); Gemini signals through a distinct message type
entirely (S32). Three sources, three vocabularies, zero overlap. Standardizing
values here would be this project's invention presented as practice, which is
the failure mode `ST-026` run a demonstrated.

### What deliberately does **not** change

- `[POLICY]` **`R13.6` stands.** A stream still ends with a terminal frame. The
  silent-close pattern (T5) is shipped by Kubernetes and Kinesis, but both pair
  it with a checkpoint on every frame and an out-of-band documented rule, and
  neither offers the client any way to distinguish a normal end from truncation
  — which is the property `R13.6` exists to provide and RFC 9112 §8 shows the
  transport does not always provide by itself.
- `[POLICY]` **`R13.7` stands.** An `error` frame remains the carrier for errors
  raised after commit, and remains terminal. `ST-030a` changes what the client
  *concludes* from one, not what the server *sends*.
- `[POLICY]` **`R13.1`'s prohibition on a streaming `202` stands.** AIP-151's
  contrary rule (S35) forbids the overlap altogether rather than proposing a
  streaming `202`, so it is not evidence against `R13.1`.

### Consequences for the decided Phase 8 items

| Item | Consequence |
| --- | --- |
| `ST-024` (duration bound) | D2's "exempt from `operation_state`" repair is replaced by `ST-030b`: the terminal frame carries the current state plus `stream_end_reason`. The rest of `ST-024` is unaffected |
| `ST-025` (credential expiry) | Its `error` frame becomes conforming for clients under `ST-030a`, which is what the decision assumed but could not state. The frame still needs an `R5.16` catalog entry (D5) |
| `ST-026` (keyed repeat / replay gap) | Its gap signal — Cog's shipped `error` event — becomes readable as "delivery incomplete" rather than "the work failed", which is the behavior the decision described in prose |
| D6 (cancellation, registered unruled) | `ST-030b` supplies the mechanism for half of it: a stream whose operation is canceled through the other channel ends with a terminal frame carrying `operation_state: canceled`, which is already terminal and needs no new rule. Whether a client disconnect cancels the operation remains unruled |

---

## 9. Declined alternatives

`[POLICY]` Each was considered against the evidence and rejected for a stated
reason.

1. **A new terminal frame type (`incomplete` / `detached`).** Declined. §7 Shape
   A. Decisive: A2A removed its own instance as redundant (S27), and the
   standard's own §13.4 register documents that adding a terminal frame type to
   a deployed API makes every existing client report truncation on success.
2. **A new HTTP status code or header for the condition.** Declined without
   extended analysis: the response status is committed as `200` before the
   condition arises (`R13.1`, `R13.8`), RFC 9110 defines nothing suitable
   (§4), and RFC 9110 §6.5.1 already rules out trailers as a channel — a
   constraint `R13.7` records.
3. **Repealing `R12.10`'s failure inference outright.** Declined. Anthropic,
   Bedrock and Transcribe (S31, S43, S44) are real populations where an error
   frame genuinely is the work failing and no resource exists to consult. An
   unconditional repeal would leave those clients with no rule at all.
4. **Requiring a `GoAway`-style advance warning before a capped close.**
   Declined. One implementation (S32), on a WebSocket transport §1.2 excludes,
   and it presumes a resumption facility that `R13.10` only makes a conditional
   `SHOULD`. It is a candidate for `streaming-profile.md` guidance, not a rule.
5. **Permitting a silent close at a documented maximum (adopting T5).**
   Declined. Kinesis and Kubernetes both ship it (S19, S21), but both pair it
   with a mandatory per-frame checkpoint that §13 makes optional (`R13.10` is a
   conditional `SHOULD`), and neither lets a client distinguish the cap from a
   truncation. `ST-024` already ruled the other way for that reason; this report
   found no evidence to reopen it.
6. **Standardizing the `stream_end_reason` value vocabulary.** Declined. Three
   sources, three disjoint vocabularies (§8).
7. **Requiring frame-for-frame replay so the case cannot arise.** Declined,
   consistent with `ST-026`: retained artifacts in the field run about 10
   minutes (OpenAI, S30) and about 5 minutes (Kubernetes, S14), and Cog's buffer
   is denominated in events and configurable to zero (S33). The requirement
   would exceed everything shipped.
8. **Following AIP-151 and forbidding the stream/operation overlap.** Declined.
   S35 is unchanged since 2019 and the field has moved past it: OpenAI (S29,
   S30), Replicate (S33), A2A (S23) and Temporal (S28) all expose the overlap.
   Recorded as the standing dissent rather than adopted.
9. **A documentation-only duty ("document what you send at the limit").**
   Declined for the reason the Phase 8 walk declined it elsewhere: it leaves the
   client unable to rely on anything, and the specific hazard here is that a
   conforming client is *required* to reach a false conclusion, which
   documentation does not prevent.

---

## 10. What could not be verified

1. `[FACT — negative]` **No published maximum stream duration or in-flight
   revocation posture was located for Anthropic** on S31. This matters because
   Ruling 1's re-check register fires on "any of OpenAI, Anthropic, or Google
   Gemini publishing a maximum stream duration or an in-flight revocation
   posture." As of 2026-08-11 that trigger has **not** fired for Anthropic on
   the streaming documentation page. Other pages were not exhaustively searched.
2. **OpenAI/Azure Realtime `session_expired`.** The exact error payload
   (`{"type":"error","error":{"type":"invalid_request_error","code":"session_expired",…}}`)
   is attested only by a community forum thread (S48) and by Azure's how-to page
   documenting `expires_at` and a maximum session duration (S47). The primary
   OpenAI Realtime reference could not be fetched with the session-lifecycle
   section intact. Treat the code string as reported, not verified. It is also a
   WebSocket surface and grounds nothing here.
3. **What OpenAI sends when a background stream's replay window has expired.**
   S29 documents `starting_after` and S30 documents roughly ten minutes of
   retention, but no error shape for "you asked to resume past what I kept" was
   located. This is the exact analogue of Cog's replay-truncation error and of
   `R13.10`'s out-of-window rejection; a second source would have strengthened
   `ST-030a`.
4. **Whether OpenAI emits anything at all when a background stream's delivery
   ends while the response runs.** S30 says the response "continues running and
   you can reconnect", which implies nothing is sent, but no statement to that
   effect was found. Recorded as unverified rather than as a negative.
5. **The literal Kubernetes `ERROR`-event JSON.** Assembled from S15's event
   encoding and S17's `Status` construction (§5.1); the field values are quoted
   from source, the concatenation is inferred. No single document in the pinned
   set prints the composed wire form.
6. **GitHub Actions Data Stream** (enterprise preview) — no public documentation
   of its termination semantics was located, so GitHub's row in §5.9 covers the
   generally available surface only.
7. **CloudEvents Subscriptions, Microsoft Azure guidelines, and Zalando
   guidelines** could only be cited at moving URLs (S12, S37, S38). Their
   findings are negatives and nothing rests on them, but a reviewer
   re-running this in future may see different text at the same URL.
8. **Twilio Media Streams' `stop` semantics for unidirectional streams** — the
   docs do not state whether the call continues after `stop`; inferred from the
   control surface, not asserted.
9. **Temporal's maximum-wait-time clause is single-sourced.** The sentence that
   makes Temporal the strongest `ST-030b` precedent — "UNSPECIFIED will be
   returned if and only if the server's maximum wait time was reached before the
   Update reached the stage specified in the request WaitPolicy … clients may
   then retry the call as needed" — appears only in S28, the machine-readable
   API definition, where it is repeated on two messages within one file. S49
   corroborates the surrounding model (wait stages, a handle used to retrieve the
   result later) but not that clause. The two-source floor is met for the model
   and **not** for the specific wait-limit sentence; §11 records the consequence.

---

## 11. Confidence summary

| Claim | Confidence | Basis |
| --- | --- | --- |
| No published standard defines the vocabulary | **High** | Twelve sources checked with recorded search targets (§4); two independent passes |
| The error channel is used in the field for delivery-only conditions | **High** | Two shipped systems, one with reference-client source showing the branch (S16–S18, S33, S34) |
| `R12.10` as written forces a false conclusion against Cog's shipped behavior | **High** | Direct reading of the rule against S33/S34 |
| Reporting the current state plus a continuation handle is the field's answer where work has its own identity | **Moderate-high** | Two strong precedents (S28 with S49, and S23 with S24 and S27) plus a supporting enum (S29); neither precedent is an HTTP API, and Temporal's wait-limit sentence is single-sourced (§10 item 9) |
| A distinct non-failure terminal frame type should be declined | **Moderate-high** | One protocol adopted then removed it (S25, S27); the standard's own compatibility register names the hazard |
| `ST-030a` (scope `R12.10`) | **Moderate-high** | The two populations are cleanly separated by an existing rule condition |
| `ST-030b` (scope `R13.9`) | **Moderate** | Re-means a ratified member; reverses a decided repair; evidence is non-HTTP |
| `ST-030c` (reserve `stream_end_reason`, define no values) | **Moderate** on the member, **high** on declining a value set | Three sources, three disjoint vocabularies |

---

## 12. Unresolved questions for the ratification walk

1. **Does the walk accept reversing D2's repair choice** (exempt from
   `operation_state` → carry the current state plus a reason)? §6.3 argues the
   objection D2 recorded does not reach the conditioned form; the walk owns the
   judgment.
2. **Is `stream_end_reason` a new reserved member, or should the reason ride an
   existing one?** No existing §1.10 member fits: `operation_state` carries a
   state, and `R13.7` keeps problem members on the `error` frame only.
3. **Should `R10.1`'s state vocabulary be explicitly required to name its
   non-terminal values?** `ST-030b` requires the API to document which values are
   terminal. Whether that belongs on `R10.1` (all operation resources) or on
   `R13.9` (the streaming intersection) is a placement question with the same
   shape as the `ST-026` / `R3.9` placement question the walk already faces.
4. **Does the Ruling 1 re-check register need a row for this report?** The
   Anthropic/OpenAI/Gemini duration-and-revocation trigger was checked here for
   Anthropic (§10 item 1) and found unfired. A dated negative may be worth
   recording so the next leaf does not re-run it blind.
