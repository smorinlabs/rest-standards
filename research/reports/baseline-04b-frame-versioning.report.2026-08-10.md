# baseline-04b — Frame-type vocabulary versioning (report, 2026-08-10)

**Series:** `baseline` (prescriptive). **Parent:** `baseline-04-streaming`.
**Question:** How should a streaming API version its frame-type vocabulary, and
what should this standard require?

**Mandate.** `research/decisions/baseline-04-streaming.decision.md` (Tier B)
recorded frame-vocabulary versioning as a known gap, opened Phase 7 to rule it,
and set an interim posture: *treat frame-type names and terminality as frozen
surface*. That posture is a placeholder, not a ratified rule. This report
supplies the evidence to rule it.

**Scope boundary — two different surfaces, only one is open.** The decision
record already **ratified a declination** of reserving specific frame-type names
(`message_stop`, `response.completed`, and similar) in §1.10, keeping only
`error`. This report does **not** reopen that. It rules a different question:
whether an API's **own published** frame-type names, once documented under
`R13.5`, are frozen within a GA major version. Registration is closed;
published-name immutability is open.

---

## 1. TL;DR and recommendation

**Recommendation (one paragraph).** Amend `R9.4` rather than mint a new rule:
add *documented frame-type names, their meanings, and which types are terminal*
to the frozen surface, and add *adding frame types where the vocabulary is
documented growable (`R13.5`)* to the compatible list — the same shape `R4.9`'s
open enums already have. The field evidence is one-directional on the additive
half and near-one-directional on the destructive half: **every reference that
publishes a compatibility promise classifies adding an event type as
backward-compatible** (Stripe, OpenAI, Anthropic, Gemini, Google AIP, Azure,
Zalando — seven independent sources, all quoted below), and **every
*deliberate* rename observed in the field rode a version mechanism** — Stripe
behind a dated `api_version` with the old name flowing to pinned endpoints
indefinitely, OpenAI behind `OpenAI-Beta: realtime=v1` with an eight-month
sunset, Anthropic behind the `anthropic-version` date header, Gemini behind an
`Api-Revision` date header on a pre-GA surface. That distribution maps exactly
onto `R9.4`'s existing taxonomy, whose breaking bucket already reads "a new
major, or a pre-GA tier", so the amendment adds a row to a table rather than new
machinery. The one genuine counterexample — OpenAI silently renaming
`response.code_interpreter_call.code.delta` on its GA Responses surface, a
change its **own** published backward-compatibility list does not license — is
the argument *for* stating the rule explicitly rather than assuming vendors
will not do it. The one project that ships this exact artifact — Kubernetes,
whose watch stream has carried `ADDED`, `MODIFIED`, `DELETED`, and `ERROR`
unchanged since 2014 — freezes it more strictly than proposed here, ruling that
"a constant value which was supported in API v1 must exist and function until
API v1 is removed" (S36). The cost of the freeze (a typo immortalized;
Kubernetes' own `phase` field is the documented case) is paid by the alias path
that AIP-180, the CloudEvents Primer, Standard Webhooks, and GitHub's shipped
practice all converge on: add the new name, keep emitting the old one, remove
the old one only at a major.

**Two findings sharpen the rule rather than soften it.** Kubernetes dissents
from the additive unanimity — adding a value is *not* compatible unless the
vocabulary declared itself growable at first publication (S37) — which is
exactly the pre-condition `R13.5` already imposes, and is why §6.1's
compatible-list entry is conditional rather than flat. And **there is no
standards-track authority on this question at all**: the three published
standards reached (RFC 6838, RFC 6648, OAS 3.1.0) are silent or speak only by
analogy, and AsyncAPI closed the request for one as not planned. Every claim
here is therefore `[COMPARATIVE]` or `[POLICY]`, never protocol fact.

**Confidence:** moderate-high on the name freeze (`[COMPARATIVE]`, seven vendor
promises plus a normatively-phrased project policy, one contrary shipped
instance); moderate on the terminality clause (`[POLICY]` — this standard's own
construction, no source states it).

---

## 2. Standards-and-currency matrix

Authority classes used: **W** = published standard body specification;
**CNCF** = CNCF-hosted specification; **VD** = vendor documentation;
**VC** = vendor changelog / deprecation notice; **VM** = vendor
machine-readable API specification (primary artifact);
**G** = published design guideline (single-organization, not a standard).

All URLs accessed **2026-08-10** unless stated otherwise.

| # | Source | URL | Class | Pub / rev date | Accessed |
|---|---|---|---|---|---|
| S1 | Anthropic — Versions (`anthropic-version`) | `https://platform.claude.com/docs/en/api/versioning` | VD | version history to `2023-06-01` | 2026-08-10 |
| S2 | Anthropic — Streaming messages | `https://platform.claude.com/docs/en/docs/build-with-claude/streaming` | VD | undated page | 2026-08-10 |
| S3 | OpenAI — Backward compatibility | `https://developers.openai.com/api/docs/api-reference/backward-compatibility` | VD | undated page | 2026-08-10 |
| S4 | OpenAI — Deprecations | `https://developers.openai.com/api/docs/deprecations` | VC | rolling; entries dated | 2026-08-10 |
| S5 | OpenAI — Changelog | `https://developers.openai.com/api/docs/changelog` | VC | rolling; entries dated | 2026-08-10 |
| S6 | OpenAI — OpenAPI specification, repo `openai/openai-openapi` | `https://github.com/openai/openai-openapi` | VM | HEAD `577fa92` dated 2026-08-11; snapshot `498c71d` dated 2025-04-29 | cloned 2026-08-10 |
| S7 | OpenAI — Realtime guide (beta-to-GA) | `https://developers.openai.com/api/docs/guides/realtime` | VD | undated page | 2026-08-10 |
| S8 | Gemini — Interactions breaking-changes migration guide (May 2026) | `https://ai.google.dev/gemini-api/docs/interactions-breaking-changes-may-2026` | VD | dated May 2026 | 2026-08-10 |
| S9 | Gemini — Interactions streaming | `https://ai.google.dev/gemini-api/docs/interactions/streaming` | VD | undated page | 2026-08-10 |
| S10 | Gemini — Changelog | `https://ai.google.dev/gemini-api/docs/changelog` | VC | entries dated 2025-12-11, 2026-05-06 | 2026-08-10 |
| S11 | Gemini — API versions | `https://ai.google.dev/gemini-api/docs/api-versions` | VD | undated page | 2026-08-10 |
| S12 | Gemini — Interactions overview | `https://ai.google.dev/gemini-api/docs/interactions-overview` | VD | states GA "as of June 2026" | 2026-08-10 |
| S13 | Stripe — API upgrades | `https://docs.stripe.com/upgrades` | VD | undated page | 2026-08-10 |
| S14 | Stripe — Changelog entry, event-type rename | `https://docs.stripe.com/changelog/2020-08-27/renames-automatically-updated-event-type` | VC | 2020-08-27 | 2026-08-10 |
| S15 | Stripe — Event destinations (thin vs snapshot events) | `https://docs.stripe.com/event-destinations` | VD | undated page | 2026-08-10 |
| S16 | GitHub — Breaking changes (REST) | `https://docs.github.com/en/rest/about-the-rest-api/breaking-changes` | VD | undated page | 2026-08-10 |
| S17 | GitHub — Webhook best practices | `https://docs.github.com/en/webhooks/using-webhooks/best-practices-for-using-webhooks` | VD | undated page | 2026-08-10 |
| S18 | GitHub — API versions | `https://docs.github.com/rest/overview/api-versions` | VD | default version `2022-11-28` | 2026-08-10 |
| S19 | Google AIP-180 — Backwards compatibility | `https://google.aip.dev/180` | G | undated; living | 2026-08-10 |
| S20 | Google AIP-181 — Stability levels | `https://google.aip.dev/181` | G | undated; living | 2026-08-10 |
| S21 | Google AIP-126 — Enumerations | `https://google.aip.dev/126` | G | undated; living | 2026-08-10 |
| S22 | Microsoft — Azure REST API Guidelines | `https://github.com/microsoft/api-guidelines/blob/vNext/azure/Guidelines.md` | G | living `vNext` branch | 2026-08-10 |
| S23 | Microsoft — Azure service versioning policy | `https://learn.microsoft.com/en-us/azure/developer/intro/azure-service-sdk-tool-versioning` | VD | undated page | 2026-08-10 |
| S24 | Microsoft — Event Grid event schema | `https://learn.microsoft.com/en-us/azure/event-grid/event-schema` | VD | undated page | 2026-08-10 |
| S25 | Twilio — Event types lifecycle | `https://www.twilio.com/docs/events/event-types-lifecycle` | VD | undated page | 2026-08-10 |
| S26 | Twilio — Event Streams overview | `https://www.twilio.com/docs/events` | VD | undated page | 2026-08-10 |
| S27 | Shopify — How Shopify manages API versioning and breaking changes | `https://shopify.engineering/shopify-manages-api-versioning-breaking-changes` | VD | undated engineering post | 2026-08-10 |
| S28 | Shopify — Webhook deprecation changelog entry | `https://shopify.dev/changelog/deprecation-of-checkoutandaccountsconfigurationsupdate-webhook` | VC | posted 2025-08-12 | 2026-08-10 |
| S29 | Shopify — API versioning | `https://shopify.dev/docs/api/usage/versioning` | VD | quarterly cadence | 2026-08-10 |
| S30 | Zalando — RESTful API Guidelines, `chapters/events.adoc` | `https://raw.githubusercontent.com/zalando/restful-api-guidelines/main/chapters/events.adoc` | G | living `main` branch | 2026-08-10 |
| S31 | Zalando — Guidelines rendered site | `https://opensource.zalando.com/restful-api-guidelines/` | G | living | 2026-08-10 |
| S32 | AWS — EventBridge event structure | `https://docs.aws.amazon.com/eventbridge/latest/ref/overiew-event-structure.html` | VD | undated page | 2026-08-10 |
| S33 | CloudEvents — Specification v1.0.2, `type` attribute | `https://github.com/cloudevents/spec/blob/v1.0.2/cloudevents/spec.md` | CNCF | v1.0.2 (released tag) | 2026-08-10 |
| S34 | CloudEvents — Primer, "Versioning of CloudEvents" | `https://github.com/cloudevents/spec/blob/main/cloudevents/primer.md` | CNCF (**non-normative**) | living `main` | 2026-08-10 |
| S35 | Kubernetes — API concepts (watch events, bookmarks) | `https://kubernetes.io/docs/reference/using-api/api-concepts/` | VD (project docs) | living | 2026-08-10 |
| S36 | Kubernetes — API deprecation policy | `https://kubernetes.io/docs/reference/using-api/deprecation-policy/` | VD (project docs) | living | 2026-08-10 |
| S37 | Kubernetes — `sig-architecture/api_changes.md` | `https://raw.githubusercontent.com/kubernetes/community/master/contributors/devel/sig-architecture/api_changes.md` | VD (project docs) | living `master` | 2026-08-10 |
| S38 | Kubernetes — `sig-architecture/api-conventions.md` | `https://raw.githubusercontent.com/kubernetes/community/master/contributors/devel/sig-architecture/api-conventions.md` | VD (project docs) | living `master` | 2026-08-10 |
| S39 | Kubernetes — KEP-956, watch bookmarks | `https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/956-watch-bookmark/README.md` | VD (project proposal) | alpha v1.15 (2019) | 2026-08-10 |
| S40 | Kubernetes — OpenAPI `WatchEvent` definition | `https://raw.githubusercontent.com/kubernetes/kubernetes/master/api/openapi-spec/swagger.json` | VM | living `master` | 2026-08-10 |
| S41 | Standard Webhooks — specification | `https://github.com/standard-webhooks/standard-webhooks/blob/main/spec/standard-webhooks.md` | **Community/industry specification — NOT an SDO standard** | states "Version: 1.0.0" | 2026-08-10 |
| S42 | AsyncAPI — specification v3.0.0, Message Object | `https://www.asyncapi.com/docs/reference/specification/v3.0.0` | Community specification | v3.0.0 | 2026-08-10 |
| S43 | AsyncAPI — spec issue #727 (compatibility mode) | `https://github.com/asyncapi/spec/issues/727` | Community issue tracker | closed as not planned | 2026-08-10 |
| S44 | Protocol Buffers — ProtoJSON format | `https://protobuf.dev/programming-guides/json/` | VD (Google) | living | 2026-08-10 |
| S45 | Protocol Buffers — proto3 guide, enum guide | `https://protobuf.dev/programming-guides/proto3/`, `https://protobuf.dev/programming-guides/enum/` | VD (Google) | living | 2026-08-10 |
| S46 | RFC 6838 — Media Type registration procedures | `https://www.rfc-editor.org/rfc/rfc6838` | **W** (IETF, BCP 13) | January 2013 | 2026-08-10 |
| S47 | RFC 6648 — Deprecating `X-` prefixes | `https://www.rfc-editor.org/rfc/rfc6648` | **W** (IETF, BCP 178) | June 2012 | 2026-08-10 |
| S48 | OpenAPI Specification 3.1.0 — Discriminator Object | `https://spec.openapis.org/oas/v3.1.0.html` | **W** (OpenAPI Initiative) | 3.1.0 | 2026-08-10 |
| S49 | CNCF — CloudEvents graduation announcement | `https://www.cncf.io/announcements/2024/01/25/cloud-native-computing-foundation-announces-the-graduation-of-cloudevents/` | CNCF | 2024-01-25 | 2026-08-10 |
| S50 | GitHub — Dependabot alerts webhook changelog | `https://github.blog/changelog/2022-10-06-new-dependabot-alerts-webhook/` | VC | 2022-10-06 | 2026-08-10 |

**Currency and status warnings.**

- **S33/S34 are not the same authority.** The CloudEvents *specification* v1.0.2
  is normative; the *Primer* states in its own front matter that it is
  non-normative. Every versioning statement quoted in §4 comes from the Primer.
  The `main`-branch spec is labelled **1.0.3-wip** and MUST NOT be cited as a
  released version.
- **S19–S22, S30–S31 are single-organization design guidelines, not standards.**
  Google AIP, the Azure guidelines, and the Zalando guidelines are comparative
  evidence of established practice only, per `CONSTRAINTS.md`.
- **S41 (Standard Webhooks) is not a published standard** despite the project
  name. Its own text describes it as "a design document [that] outlines our
  proposal", governed by a vendor technical steering committee. It carries the
  string "Version: 1.0.0" but has no SDO (IETF, W3C, ISO) backing. It MUST NOT
  be cited as a standard; the name is branding.
- **S46–S48 are the only standards-body publications in this report** (IETF ×2,
  OpenAPI Initiative ×1), and none of them addresses frame-type versioning; they
  are cited for adjacent principles only, which is stated at each use. **S33
  (CloudEvents v1.0.2) is a ratified specification of a graduated CNCF project**
  (S49) — a real specification, but a foundation's rather than a standards
  body's, and it too is silent on this question: its versioning guidance lives
  only in the non-normative Primer (S34).
- **S6 has a one-year hole.** The `openai/openai-openapi` repository deleted
  `openapi.yaml` at commit `e1cb7a8` (2025-06-17) and re-added it at `5162af9`
  (2026-05-13). Any change dated inside that window can only be bounded, not
  pinpointed. This is stated wherever it bites.

---

## 3. Field evidence

### 3.1 Anthropic — frame types are declared enum-like output values, and a stream-format change was version-gated

**[FACT]** The versioning page (S1) states what is preserved within a version:

> "For any given version with the Messages API, Anthropic preserves: Existing
> input parameters · Existing output parameters"

and what may change:

> "However, Anthropic may do the following: Add additional optional inputs · Add
> additional values to the output · Change conditions for specific error types ·
> **Add new variants to enum-like output values (for example, streaming event
> types)**"

**[FACT]** The streaming page (S2) carries the matching client obligation:

> "In accordance with the versioning policy, new event types may be added, and
> your code should handle unknown event types gracefully."

**[FACT] The load-bearing datum — a stream-shape change was gated behind the
version header.** S1's version history for `2023-06-01`:

> "New format for streaming server-sent events (SSE): … All events are named
> events, rather than data-only events. **Removed unnecessary `data: [DONE]`
> event.**"

**[INFERENCE]** Anthropic is the clearest instance in this survey of a vendor
treating the *shape and vocabulary* of a stream as versioned contract: changing
from data-only to named events, and removing the terminal sentinel, required a
new `anthropic-version` date rather than shipping in place. Read against S1's
"Anthropic preserves … existing output parameters", the additive-only promise
for event types is explicit and the destructive direction is handled by the
version header.

**Two-source:** S1 and S2 are independent pages that corroborate the additive
promise; the version-gating datum is single-source (S1) but is the vendor's own
version history, the authoritative artifact for that claim.

**Sub-vocabulary note.** Anthropic's `content_block_delta` frames carry a
second-level discriminator (`text_delta`, `input_json_delta`, `thinking_delta`,
`signature_delta`) (S2). S1's phrase "enum-like output values" covers these as
much as the top-level event names — the vendor draws no line between the two
levels.

### 3.2 OpenAI — the published promise says additions only; the shipped spec shows two gated renames and one silent one

**[FACT] The published compatibility list** (S3) enumerates backwards-compatible
changes:

> "Adding new resources (URLs) to the REST API and client libraries · Adding new
> optional API parameters · Adding new properties to JSON response objects or
> event data · Changing the order of properties in a JSON response object ·
> Changing the length or format of opaque strings, like resource identifiers ·
> **Adding new event types in streaming APIs**"

**[FACT]** S3 does **not** enumerate breaking changes. It states only that
OpenAI avoids them "whenever reasonably possible". So OpenAI publishes the
additive half of the promise and leaves the destructive half unstated —
a gap this standard's `R9.4` exists to close.

**[FACT] Gated rename #1 — the Realtime beta-to-GA vocabulary change.**
Verified from OpenAI's own machine-readable specification (S6), not from prose:
the schema `RealtimeServerEventResponseAudioDelta` at HEAD (`577fa92`,
2026-08-11) declares its wire value as

> `enum: [response.output_audio.delta]` — "The event type, must be
> `response.output_audio.delta`."

while the same wire name `response.audio.delta` was the Realtime value in the
2025-04-29 snapshot (`498c71d`). Likewise `RealtimeServerEventResponseTextDelta`
now declares `response.output_text.delta`. **The schema identifiers still carry
the old names** — a fossil that makes the rename unambiguous rather than a
coincidence of two unrelated schemas.

Set difference of event-type-shaped string literals between the 2025-04-29
snapshot and HEAD — names present then, absent now:

| Name present 2025-04-29, absent at HEAD | Replacement at HEAD | Surface |
|---|---|---|
| `response.text.delta` / `response.text.done` | `response.output_text.delta` / `.done` | Realtime (beta → GA) |
| `response.audio_transcript.delta` / `.done` | `response.output_audio_transcript.delta` / `.done` | Realtime (beta → GA) |
| `response.code_interpreter_call.code.delta` / `.done` | `response.code_interpreter_call_code.delta` / `.done` | **Responses (GA)** |
| `response.code_interpreter_call.in.progress` | `response.code_interpreter_call.in_progress` | **Responses (GA)** |

**[FACT] How the Realtime rename was gated.** S7: "Remove the
`OpenAI-Beta: realtime=v1` header when calling the GA interface." S4 records
"Realtime API Beta", announced 2025-09-15, shutdown 2026-05-12. S5 confirms:
"The Realtime API Beta was deprecated and removed from the API on May 12,
2026." That is a **beta-gated rename with roughly an eight-month sunset** —
three independent vendor artifacts.

**[FACT] Silent rename #2 — the genuine counterexample.** The
`response.code_interpreter_call.code.delta` → `response.code_interpreter_call_code.delta`
change happened on the **GA Responses surface**. Evidence that it is GA and not
beta: in the 2025-04-29 snapshot the schema
`ResponseCodeInterpreterCallCodeDeltaEvent` carries
`x-oaiMeta: {name: response.code_interpreter_call.code.delta, group: responses}`
with **no** `beta`, `preview`, `experimental`, or `deprecated` marker anywhere in
the schema block (grepped 2026-08-10). The schema identifier is byte-identical
at HEAD; only the wire value moved. **No changelog entry for this rename was
found** — S5 was fetched and searched on 2026-08-10 and returned no entry
mentioning renamed events. Under S6's repository gap, the change can only be
bounded to **between 2025-06-17 and 2026-05-13**.

**[INFERENCE]** This rename is not licensed by OpenAI's own published
compatibility list (S3), which permits additions and says nothing about renames.
A vendor that publishes a compatibility contract still shipped an unannounced
rename on a GA surface. `response.code_interpreter_call.in.progress` →
`.in_progress` in the same window reads as a typo fix.

**[FACT] Terminal types were stable.** `response.completed`, `response.failed`,
`response.incomplete`, `response.created`, and `response.in_progress` are present
in both the 2025-04-29 snapshot and HEAD. Across sixteen months of OpenAI's
Responses vocabulary, the frames that end a stream did not move.

**[FACT] Whole-vocabulary retirement is handled by deprecation machinery, not
rename.** S4: "On August 26th, 2025, we notified developers using the Assistants
API of its deprecation and removal from the API one year later, on August 26,
2026." The Assistants `thread.run.*` vocabulary plus its `event: done` /
`data: [DONE]` sentinel retires wholesale on a twelve-month notice, replaced by
the Responses API — not renamed in place.

### 3.3 Google Gemini — a documented breaking rename of six event types, including the terminal one

**[FACT]** The Interactions API's current streaming page (S9) states:

> "In accordance with the API's versioning policy, new event types and delta
> types may be added over time."

and the client obligation:

> "Your code should handle unknown event types gracefully—log and skip any
> events you don't recognize rather than throwing an error."

**[FACT] The rename.** S8 is a dedicated migration guide. Its own framing:

> "The changes described in this guide are breaking changes to the Interactions
> API. The legacy schema will be removed on **June 8, 2026**. Use the
> `Api-Revision` request header to manage your migration."

The event-type mapping it publishes:

| Legacy event type | Replacement |
|---|---|
| `interaction.start` | `interaction.created` |
| `content.start` | `step.start` |
| `content.delta` | `step.delta` |
| `content.stop` | `step.stop` |
| `interaction.complete` | `interaction.completed` |
| `interaction.status_update` | `interaction.in_progress`, `interaction.requires_action`, and others |

**This is the exact failure the register named.** `interaction.complete` is the
terminal frame. Under `R12.10` a deployed client that ignores the unrecognized
`interaction.completed` would see no terminal frame and report truncation on
every success.

**[FACT] The migration mechanism**, quoted from S8's timeline table:

| Date | Phase | REST API users |
|---|---|---|
| May 7 | Opt-in | "Add `Api-Revision: 2026-05-20` header to opt in. Default remains legacy." |
| May 26 | Default flip | "New schema is now the default. Send `Api-Revision: 2026-05-07` header to opt out." |
| June 8 | Sunset | "Legacy schema removed for Interactions API. `Api-Revision` header ignored." |

S8 also states: "New features shipped after May 7 will only appear in `steps`
responses. Users on the legacy `outputs` schema will not receive new
capabilities until they migrate."

**Two-source:** S10 (the changelog) independently corroborates, dated
2026-05-06: "**Upcoming breaking change**: The Interactions API request and
response schema (`outputs` → `steps`) and output format configuration
(`response_format`) are changing. The new schema becomes the default on **May
26** and the legacy schema will be removed on **June 8**."

**[FACT] Stability tier — this rename was pre-GA.** S10 dates the Interactions
API launch to 2025-12-11 ("Launched the Interactions API"). S12 states: "The
Interactions API is the best way to build with Gemini models and agents. As of
June 2026, it is Generally Available and recommended for all new projects." The
rename ran 2026-05-07 to 2026-06-08 — **inside the API's first six months and
concluding at GA**.

**[INFERENCE]** Classified against `R9.4`'s taxonomy, this is a breaking change
made on a **pre-GA tier**, which `R9.4` already permits. It is therefore not a
counterexample to the freeze; it is a confirming instance, and it demonstrates
the failure mode this report is about being taken seriously enough to warrant a
dedicated migration guide, a request header, and a dated sunset.

**Source conflicts, surfaced not averaged.**

1. **`interaction.status_update` is simultaneously retired and current.** S8
   lists it as replaced; S9, the live streaming reference, still lists it among
   the current event types. Both accessed 2026-08-10. Which governs is not
   determinable from the published pages.
2. **Policy versus practice.** S11 states the general Gemini policy: "Features
   in the stable version are fully supported over the lifetime of the major
   version. If there are any breaking changes, a new major version of the API
   will be created and the existing version will be deprecated after a
   reasonable period of time." The Interactions rename instead used a dated
   `Api-Revision` header with a thirteen-day gap between default flip and hard
   removal — no new major version, and a migration window an order of magnitude
   shorter than Stripe's or OpenAI's. **The practice governs as evidence of what
   happened; the policy governs as evidence of what the vendor says it will
   do.** This standard should follow neither uncritically.

### 3.4 The eight standard references

Sources S13–S32, plus S30 for Zalando. Non-participation is recorded as a
one-line verified negative with the URL checked and the date.

**Stripe.** **[FACT]** S13 lists, under "Backward-compatible changes":
"Adding new event types. Make sure that your webhook listener gracefully
handles unfamiliar event types." **[FACT]** S14 is a shipped rename:
"Renames the `payment_method.card_automatically_updated` event type to
`payment_method.automatically_updated`", with the upgrade instruction "update
your API requests to include `Stripe-Version: 2020-08-27`" and "Upgrade the API
version used for webhook endpoints", plus a 72-hour rollback window. **[FACT]**
S13 on the mechanism: "Each major release … includes changes that aren't
backward-compatible with previous releases", and a webhook endpoint with "an
explicit version set … always uses that version". **[INFERENCE]** Stripe's
rename is therefore version-gated with **no forced sunset**: an endpoint pinned
to a pre-2020-08-27 version keeps receiving the old name. This is the most
consumer-protective rename mechanism observed. **[FACT]** S15 records that v2
"thin events" are "Unversioned, allowing you to upgrade your integration without
changing your webhook endpoint configuration", in contrast to v1 snapshot events
which are "Versioned by API version".

**GitHub.** **[FACT]** S16's breaking-change list includes "Removing an entire
operation", "Removing or renaming a parameter", "Removing or renaming a response
field", "Removing enum values"; its compatible list includes "Adding enum
values". **[FACT — verified negative]** S16 is written for REST operations,
parameters, and response fields and **does not mention webhooks, webhook event
names, or event payloads** (checked 2026-08-10). S17 gives the nearest client
obligation: "Your application should check the event type and action of a
webhook payload before processing the payload" and "GitHub continues to add new
event types and new actions to existing event types." **[FACT]** S18: date-based
`X-GitHub-Api-Version`, default `2022-11-28`, prior version supported "for at
least 24 more months". **[INFERENCE]** GitHub is the clearest case of a
compatibility contract whose own wording leaves the event-name vocabulary
outside its scope — precisely the `R9.4` gap this report closes.

**[COMPARATIVE] GitHub has replaced webhook event names, always by
dual-delivery, never in place.** Two instances were located:
`integration_installation` / `integration_installation_repositories` →
`installation` / `installation_repositories`, and
`repository_vulnerability_alert` → `dependabot_alert` (S50, dated 2022-10-06,
announced with advance notice). In the first, GitHub ran both names
concurrently — "When you subscribe to the new replacement webhook events, you
will also receive the deprecated events until they are removed permanently."
**Neither is a rename in the strict sense**: a new name was introduced, the old
name kept flowing for a transition window, and the old name was removed only at
the end of it. **No instance was found, at GitHub or anywhere in this survey, of
an event-type name being reused for a different meaning, or swapped with no
transition period.** That absence is itself a finding. **Confidence caveat:** the
exact removal date for the Integrations events rests on secondary discussion —
GitHub retired the `developer.github.com/changes/` changelog path and the
primary announcement URL now returns 404 (checked 2026-08-10) — so the
dual-delivery *pattern* is the claim made here, not any specific date.

**Google AIP.** **[FACT]** AIP-180 (S19): "Existing components (interfaces,
methods, messages, fields, enums, or enum values) **must not** be removed from
existing APIs in the same major version", and — the load-bearing sentence —
"Renaming a component is semantically equivalent to 'remove and add'. In cases
where these sorts of changes are desirable, a service **may** add the new
component, but **must not** remove the existing one." **[FACT]** On additions:
"Enum values **may** be freely added to enums which are only used in request
messages. Enums that are used in response messages or resources and which are
expected to receive new values **should** document this." **[FACT]** AIP-181
(S20) on stable components: "There **must** be no breaking changes to these
components", and on beta: "Backwards-incompatible changes **must** be made only
after a reasonable deprecation period." **[FACT — verified negative]** AIP-126
(S21) contains enum naming and design guidance (the `_UNSPECIFIED` zero-value
convention) but no client-tolerance instruction for unrecognized values, checked
2026-08-10.

**Microsoft / Azure.** **[FACT]** S22: "☑️ **YOU SHOULD** use extensible
enumerations unless you are positive that the symbol set will NEVER change over
time"; "✅ **DO** document to customers that new values may appear in the future
so that customers write their code today expecting these new values tomorrow";
"⛔ **DO NOT** remove values from your enumeration list as this breaks customer
code." **[FACT]** S23: "Stable versions are backwards compatible"; breaking
versions "are rare", are announced and preceded by a preview, and prior stable
versions "remain available for at least three years after the breaking change
version releases." **[FACT — verified negative]** S24 defines Event Grid's
`eventType` as "One of the registered event types for this event source" and
`dataVersion` as publisher-defined, with **no compatibility contract published
for the `eventType` string vocabulary itself** (checked 2026-08-10). No Event
Grid `eventType` rename instance was found.

**Twilio.** **[FACT]** S25 publishes a four-stage lifecycle for event type
names: "Available event types are ready to use"; "You can still use deprecated
event types, but you should start looking for alternatives"; "You should
immediately replace restricted event types with alternatives"; "You may not
receive events for discontinued event types"; and "In most cases, an event type
is deprecated to give notice to users before being restricted or discontinued."
**[FACT]** S26: event types follow `com.twilio.<product>.<resource>.<action>`
and "one event type can have multiple versions". **[FACT — verified negative]**
No page fetched contains an "ignore unrecognized event types" instruction, and
no specific historical discontinuation instance was located (checked
2026-08-10). **[INFERENCE]** Twilio is the only reference with a *named lifecycle
state machine* for event type names — evidence that retirement is treated as a
process with notice, not an edit.

**Shopify.** **[FACT]** S27's breaking-change definition: "Ultimately, a
breaking change is any change that requires a third-party developer to do any
migration work to maintain the existing functionality of their application", and
its examples explicitly include "modifying the expected payload of webhooks and
async callbacks". **[FACT]** S28 is a shipped webhook-topic removal: "Apps still
subscribed to the `checkout_and_accounts_configurations/update` webhook should
unsubscribe before the removal date", posted 2025-08-12 with removal effective
2026-01-01. **[FACT]** S29: quarterly date-named versions; "Each stable version
is supported for a minimum of 12 months, with at least nine months of overlap
between consecutive versions." **[FACT — verified negative]** No
ignore-unknown-topics instruction found (checked 2026-08-10).

**Zalando.** The only reference with normative rules that treat an event type as
a first-class versioned artifact. **[FACT]** Rule **209** *{MUST} Maintain
backwards compatibility for events* (S30): "Changes to events must be based
around making additive and backward compatible changes." Its
backward-compatible list includes "Adding new optional fields to JSON objects"
and "Adding new values to extensible enum fields"; its incompatible list
includes "Changing the type of a field, object, enum or array" and "Adding a
value to an `enum` enumeration" (i.e. a *closed* enum). **[FACT]** Rule **213**
*{MUST} Follow naming convention for event type names*: the pattern is
`<functional-name>.<event-name>[.<version>]`, with the published example
`customer-personal-data.email-changed.v2`. **[FACT]** Rule **197** *{MUST}
Specify and register events as event types*: an EventType declaration carries
"The name of the event type", "A schema defining the event payload", and "The
compatibility mode for the type". **[FACT]** Rule **245** *{MUST} Carefully
define the compatibility mode*: modes are `none`, `forward`, `compatible`, where
`compatible` means "Only the addition of new optional properties and definitions
to an existing schema is allowed." **[FACT]** Rule **246** *{MUST} Use semantic
versioning of event type schemas*: "Event schemas must be versioned", and
"Changing an event type with compatibility mode `compatible` or `forward` can
lead to a PATCH or MINOR version revision. MAJOR breaking changes are not
allowed."

**[INFERENCE]** Zalando answers the rename problem structurally rather than by
prohibition: rule 213 puts the version **inside the event type name**, so a
breaking revision produces `…email-changed.v2` — a *new name*, never a mutated
one. Under that convention the freeze is automatic, because no name is ever
reused with different meaning.

**AWS.** **[FACT]** S32 defines the envelope: "The combination of the **source**
and **detail-type** fields identify the service that has generated the event,
and the type of event, respectively", with `version` describing the envelope
schema. **[FACT — verified negative, four ways]** Checked 2026-08-10: no
published statement that adding a `detail-type` is compatible; none that
renaming or removing one is breaking; no changelog instance of an AWS service
renaming a `detail-type`; no instruction to consumers to ignore unrecognized
`detail-type` values. **[INFERENCE]** AWS is the one reference in the set with
no centralized API compatibility policy page comparable to Stripe's, GitHub's,
or Azure's; its commitments are per-service.

### 3.5 Comparison table

Read "adding compatible?" as *does the source publish that adding an event type
is a backward-compatible change*. Read "rename gated?" as *did an observed
rename ride a version mechanism*.

| Reference | Event/frame-type surface | Adding compatible, published? | Rename or removal classified breaking? | Observed rename or removal | Rename gated by | Ignore-unknown instruction |
|---|---|---|---|---|---|---|
| Anthropic | SSE named events, Messages API | Yes — "new variants to enum-like output values (for example, streaming event types)" | Not stated for names; existing output parameters preserved per version | Stream format changed at `2023-06-01` (named events; `[DONE]` removed) | `anthropic-version` date header | Yes, explicit |
| OpenAI | SSE named events, Responses + Realtime + Assistants | Yes — "Adding new event types in streaming APIs" | **Not published** — no breaking list at all | Realtime beta→GA renames; **one silent GA rename** in Responses | Realtime: `OpenAI-Beta: realtime=v1`, 8-month sunset. Responses rename: **nothing** | No |
| Google Gemini | SSE named events, Interactions API | Yes — "new event types and delta types may be added over time" | Yes — dedicated guide titled as breaking changes | Six event types renamed, incl. the terminal `interaction.complete` | `Api-Revision` date header; 13 days from default flip to sunset; pre-GA | Yes, explicit ("log and skip") |
| Stripe | Webhook event types | Yes — "Adding new event types" | Implied structurally by dated `api_version` pinning, not one sentence | `payment_method.card_automatically_updated` → `payment_method.automatically_updated` | `Stripe-Version: 2020-08-27`; old name flows to pinned endpoints indefinitely | Yes, twice |
| GitHub | Webhook events | Yes for REST enums; **webhooks out of the policy's scope** | Yes for REST params/fields/enums; **not for webhook names** | None found for event names; field-level payload changes only | n/a | Partial — "check the event type … before processing" |
| Google AIP | Design guideline (enums, components) | Yes, with caution for response-side enums | Yes — "Renaming a component is semantically equivalent to 'remove and add'" | n/a — a guideline, no shipped changelog | n/a | Not explicit |
| Microsoft / Azure | Event Grid `eventType`; extensible enums | Yes, as general REST guidance | Yes, as general REST guidance — "DO NOT remove values" | None found for `eventType` | n/a | Yes, as general guidance |
| Twilio | Event Streams event types | Not stated | Yes — four-stage lifecycle with advance notice | Lifecycle documented; no dated instance located | Lifecycle stages, not a version header | No |
| Shopify | Webhook topics | Not stated for topics; general forward-compatibility test only | Yes — webhook payload changes named as breaking | `checkout_and_accounts_configurations/update` removed, ~5 months' notice | Quarterly date-named API version | No |
| Zalando | Registered Event Types | Yes — rule 209, additive changes | Yes — rule 209; rule 246 makes rename MAJOR | n/a — a guideline | Version embedded in the event type name (rule 213) | Yes — robustness principle, extensible enums |
| AWS | EventBridge `detail-type` | **No — verified negative** | **No — verified negative** | **None found — verified negative** | n/a | **No — verified negative** |

---

## 4. Standards layer

**CloudEvents (CNCF).** **[FACT]** The normative specification v1.0.2 (S33)
defines `type` as: "This attribute contains a value describing the type of event
related to the originating occurrence. Often this attribute is used for routing,
observability, policy enforcement, etc. The format of this is producer defined
and might include information such as the version of the `type`". Constraints:
REQUIRED, non-empty string, "SHOULD be prefixed with a reverse-DNS name."
**[FACT — verified negative]** The normative spec gives **no** guidance on
consumer handling of unrecognized `type` values (checked 2026-08-10).

**[FACT] The Primer (S34), explicitly non-normative**, is where CloudEvents
addresses evolution: "The CloudEvents specification does not mandate any
particular pattern to be used, or even the use or consideration of versioning at
all." Then, the two sentences that matter here:

> "When a CloudEvent's data changes in a backwardly-compatible way, the value of
> the `type` attribute should generally stay the same."
>
> "When a CloudEvent's data changes in a backwardly-incompatible way, the value
> of the `type` attribute should generally change."

and the migration path:

> "The event producer is encouraged to produce both the old event and the new
> event for some time (potentially forever) in order to avoid disrupting
> consumers."

**[INFERENCE]** CloudEvents' Primer prescribes the same shape this report
recommends, from the opposite direction: a type name is bound to a meaning, so a
meaning change produces a *new* name, and the transition is a dual-publish
window rather than a mutation. It must not be cited as a normative requirement —
it is a primer.

**Kubernetes — the strongest precedent in the report, and the source of its one
real conflict.**

**[FACT] The vocabulary.** S35 documents the watch event stream and its types;
`ADDED`, `MODIFIED`, `DELETED`, `ERROR` come from the `WatchEvent` type, and
`BOOKMARK` from the bookmarks feature: "the Kubernetes API provides a watch
event named BOOKMARK. It is a special kind of event to mark that all changes up
to a given resourceVersion the client is requesting have already been sent."

**[FACT] The vocabulary is an open string at the schema level.** In the
Kubernetes OpenAPI specification (S40), `WatchEvent`'s `type` member is declared
as a bare `"type": "string"` **with no `enum` constraint**. The five names are
documented in prose and enforced by policy, not by the schema.

**[FACT] The deprecation policy covers enum-like values by name.** S36 lists
what the deprecation rules govern: "REST resources (aka API objects) · Fields of
REST resources · Annotations on REST resources … · **Enumerated or constant
values** · Component config structures." Its Rule #1: "API elements may only be
removed by incrementing the version of the API group. Once an API element has
been added to an API group at a particular version, it can not be removed from
that version or have its behavior significantly changed, regardless of track."
And the passage specific to this class: "As with whole REST resources and fields
thereof, **a constant value which was supported in API v1 must exist and
function until API v1 is removed**."

**[COMPARATIVE]** This is the closest thing in the survey to the rule this
report proposes, written by a project that ships the exact artifact under
discussion — a typed event stream — and it freezes constant values to the life
of the API version. It is project documentation, not a standard, but it is
normatively phrased and enforced.

**[FACT] `BOOKMARK` was added purely additively, behind a request opt-in.**
S39 (KEP-956) extends `ListOptions` with `AllowWatchBookmarks`. S35's client
guidance: "you can request BOOKMARK events by setting the
`allowWatchBookmarks=true` query parameter to a watch request, but you shouldn't
assume bookmarks are returned at any specific interval, nor can clients assume
that the API server will send any BOOKMARK event even when requested."

**[FACT] No watch event type has ever been renamed or removed.**
`ADDED`/`MODIFIED`/`DELETED`/`ERROR` date to the earliest history of
`pkg/watch/watch.go` (2014); `BOOKMARK` was added once, in 2019. No rename or
removal was located.

**[FACT] The source conflict — Kubernetes says adding is *not* compatible.**
S37 states, in direct contradiction of the seven vendor promises in §3:

> "Adding a new value to an enumerated set is *not* a compatible change. Clients
> which assume they know how to handle all possible values of a given field will
> not be able to handle the new values."
>
> "However, removing a value from an enumerated set *can* be a compatible
> change, if handled properly (treat the removed value as deprecated but
> allowed)."

**This conflict is not averaged; it is resolved by a condition S37 itself
supplies** in the very next passage:

> "For enumeration-like fields that expect to add new values in the future, such
> as `reason` fields, **document that expectation clearly in the API field
> description in the first release the field is made available, and describe how
> clients should treat an unknown value.** Clients should treat such sets of
> values as potentially open-ended."

**[INFERENCE]** Kubernetes and the seven vendors are not disagreeing about
facts; they are describing two different starting conditions. Adding to a
vocabulary that was *never declared growable* breaks clients that reasonably
assumed exhaustiveness. Adding to a vocabulary that *was* declared growable at
first publication does not. That is precisely the pre-condition `R13.5` already
imposes, and it is why the `R9.4` compatible-list wording proposed in §6.1 is
conditional ("where the API has documented its frame-type vocabulary as
growable") rather than unconditional. **This report treats S37 as governing on
the mechanism and the vendor promises as governing on the practice.** An
independent project, reasoning from first principles, arrived at `R13.5`'s
growth-statement requirement as the thing that makes additions safe.

**[FACT] Kubernetes' own published cost case for enum permanence.** S38, on the
`phase` field: "The pattern of using `phase` is deprecated. Newer API types
should use conditions instead. Phase was essentially a state-machine enumeration
field, that contradicted system-design principles and hampered evolution, since
adding new enum values breaks backward compatibility." **[INFERENCE]** This is
the field's clearest documented instance of an enumeration design that could not
be corrected in place and had to be superseded wholesale — the cost of
permanence, stated by the project that paid it.

**CloudEvents status.** **[FACT]** CloudEvents is a CNCF **graduated** project as
of 2024-01-25 (S49), which makes v1.0.2 a ratified specification of a recognized
foundation — but the versioning guidance quoted above still lives only in the
non-normative Primer.

**Standard Webhooks.** **[FACT]** The specification (S41) — a community
document, **not a standard**, see §2's warning — states: "Event types indicate
the type of the event being sent in the webhook and the schema of the payload.
**A payload associated with an event type should always have the same schema.**"
Its published migration strategy for an incompatible payload change: "The last
alternative is to create a new event type for each of the existing ones, and
have them conform to the new format." **[INFERENCE]** Standard Webhooks does not
prohibit renaming; it makes renaming unnecessary by binding one name to one
schema and minting a new name for a new schema — the same shape as CloudEvents'
Primer and AIP-180.

**AsyncAPI — verified negative.** **[FACT]** The v3.0.0 Message Object (S42)
defines `name` only as "A machine-friendly name for the message", with no
stability, versioning, or compatibility guidance attached. **[FACT]** Spec issue
#727, "Is there a way to document the guaranteed compatibility mode for produced
messages?", is **closed as not planned** (S43). **AsyncAPI publishes no
normative stance on message-type name evolution as of 2026-08-10.** The only
compatibility tooling located is the third-party `asyncapi/diff` package, which
is not part of the specification.

**Protocol Buffers — the mechanism that explains the whole pattern.**
**[FACT]** S44, in Google's own words: ProtoJSON "**does not support unknown
fields, and it puts field and enum value names into encoded messages which makes
it much harder to change those names later**." Enum JSON representation: "The
name of the enum value as specified in proto is used." **[FACT]** By contrast,
S45 states that for the binary wire format "Adding additional values to an enum
is safe", and that it is *renumbering* — not renaming — that "result[s] in
incompatible wire-format changes", because binary protobuf encodes the number.
**[FACT]** S45 on open enums: "When a proto3 file imports an enum defined in a
proto3 file, that enum should be treated as open", with unrecognized values
preserved rather than dropped.

**[INFERENCE]** This settles the mechanical question. A name is safe to change
only where an integer indirection carries identity on the wire. A stream frame
type is a string on the wire with no indirection — structurally the ProtoJSON
case, which Google's own documentation says makes names "much harder to change
later". No encoding trick makes a frame-type rename compatible; only
dual-publishing both names does.

**Google AIP-126 already uses the word "frozen".** **[FACT]** S21: "Enums
**should** document whether the enum is **frozen** or they expect to add values
in the future", and "Enums **should** receive new values infrequently … a good
rule of thumb is no more than once a year."

**IETF.** **[FACT — verified negative]** No IETF RFC or active Internet-Draft
addressing SSE event-name (`event:` field) evolution or event-type compatibility
was located (searched 2026-08-10; the one SSE-adjacent active draft,
`draft-ietf-alto-incr-update-sse`, defines ALTO-specific control events and
generalizes to nothing here — and per `research/CLAUDE.md` an Internet-Draft
must never be cited as a published standard in any case).

**[FACT]** The nearest published-standard principle is registry permanence. RFC
6838 §5.5 (S46, BCP 13, January 2013): "Media type registrations may not be
deleted; media types that are no longer believed appropriate for use can be
declared OBSOLETE by a change to their 'intended use' field." **[INFERENCE]**
IANA's model for a registered wire name is *permanence plus a status flag*,
never reuse or rename — the same shape as the dual-emit path in §6.2. This is an
analogy across registries, not authority over frame types, and is labelled as
such.

**[FACT]** RFC 6648 (S47, BCP 178) is about name *form*, not name *stability*,
but its rationale for deprecating `X-` prefixes is the migration cost of a
renamed wire identifier: migration "introduces interoperability issues …
because older implementations will support only the 'X-' name and newer
implementations might support only the standardized name." **[INFERENCE]** The
IETF's stated reason for avoiding a name change is exactly the failure mode this
report addresses.

**OpenAPI / JSON Schema — verified negative.** **[FACT]** The OpenAPI
Specification 3.1.0 Discriminator Object (S48) defines only the mechanics of
selecting a schema from a discriminator value and its mapping. It contains **no
guidance on adding, removing, or renaming discriminator values over time**, and
no compatibility or versioning language (checked 2026-08-10). JSON Schema has no
native discriminator keyword at all.

**The standards-layer bottom line.** `[FACT]` **There is no standards-track
authority on frame-type versioning.** The three standards-body publications
reached (RFC 6838, RFC 6648, OAS 3.1.0) are silent on it or speak only by
analogy; the one ratified foundation specification (CloudEvents v1.0.2) is
likewise silent, with its versioning guidance confined to a non-normative
Primer; and the one community specification asked to supply such a rule
(AsyncAPI, issue #727) declined. Every substantive claim in §3 and §5 is
therefore `[COMPARATIVE]` or `[POLICY]`, never protocol fact — which is why §6
proposes the rule at the strength the evidence supports and no higher.

---

## 5. Evidence for and against, stated separately

### 5.1 For requiring frame-type names to be frozen

1. **The additive promise is near-unanimous, and it is only half a contract.**
   Seven independent sources publish "adding is compatible" (Anthropic S1,
   OpenAI S3, Gemini S9, Stripe S13, GitHub S16 for enums, Azure S22, Zalando
   rule 209). None of them publishes the companion statement that renaming is
   not. A client told "the vocabulary may grow" reasonably infers "and does not
   shrink"; nobody has written that down. `[COMPARATIVE]`

   **Dissent, recorded not averaged:** Kubernetes (S37) holds that "Adding a new
   value to an enumerated set is *not* a compatible change", and — inverting the
   usual polarity — that *removing* one "*can* be a compatible change, if handled
   properly (treat the removed value as deprecated but allowed)." §4 resolves
   this: S37 is describing a vocabulary that never declared itself growable, and
   its own remedy is to "document that expectation clearly … in the first release
   the field is made available". Under `R13.5`, which mandates exactly that
   statement, the two positions agree. The dissent is the reason §6.1's
   compatible-list entry is conditional rather than flat.

2. **The catastrophic case is not hypothetical — Gemini shipped it.**
   `interaction.complete` → `interaction.completed` (S8) renames the terminal
   frame. Under `R12.10` every deployed client ignores the unrecognized name,
   observes no terminal frame, and reports truncation **on every successful
   stream**. Gemini treated it as breaking, wrote a migration guide, and gated
   it behind a request header — the strongest available demonstration that the
   industry regards this specific change as breaking. `[FACT]` on the rename and
   its classification; `[INFERENCE]` on the client-side consequence, which
   follows from `R12.10` by construction.

3. **Every deliberate rename in the field rode a version mechanism.** Stripe a
   dated `api_version` (S14), OpenAI a beta header with an eight-month sunset
   (S4, S5, S7), Anthropic a version date (S1), Gemini a dated `Api-Revision`
   header on a pre-GA surface (S8). Four vendors, four mechanisms, zero
   in-place renames on a GA surface *that anyone announced*. `[COMPARATIVE]`

4. **The design-guideline layer already says it, in the general case.**
   AIP-180: "Renaming a component is semantically equivalent to 'remove and
   add'" and removal is forbidden within a major (S19). Azure: "DO NOT remove
   values from your enumeration list as this breaks customer code" (S22).
   Zalando rule 209 requires event changes to be additive (S30). The proposed
   amendment applies an established general rule to a surface `R9.4` currently
   omits. `[COMPARATIVE]`

5. **CloudEvents' Primer independently prescribes name-binding.** A meaning
   change should change the `type`; the old event should keep being produced
   "for some time (potentially forever)" (S34). `[COMPARATIVE]`, non-normative.

6. **The one project that ships this exact artifact freezes it explicitly.**
   Kubernetes' deprecation policy (S36) names "Enumerated or constant values"
   among the elements it governs and rules that "a constant value which was
   supported in API v1 must exist and function until API v1 is removed", under a
   Rule #1 forbidding an element from being removed "or hav[ing] its behavior
   significantly changed". Its five watch event types have not been renamed or
   removed since 2014, and the one addition since (`BOOKMARK`, 2019) arrived
   behind a request opt-in. This is the closest available analogue to the
   proposed rule, applied to a real typed event stream, and it is *stricter*
   than what §6 proposes. `[COMPARATIVE]`

7. **The mechanism forecloses the alternative.** Google's own ProtoJSON
   documentation (S44) states that putting "enum value names into encoded
   messages … makes it much harder to change those names later", where the
   binary format — which encodes numbers — permits renaming freely (S45). A
   frame type is a bare string on the wire with no indirection. There is no
   encoding-level fix; dual-publishing is the only compatible path. AIP-126
   (S21) already frames the choice in the report's own vocabulary: "Enums
   **should** document whether the enum is **frozen** or they expect to add
   values in the future." `[COMPARATIVE]`

8. **Registry practice at the IETF is permanence plus a status flag.**
   RFC 6838 §5.5 (S46): "Media type registrations may not be deleted; media
   types that are no longer believed appropriate for use can be declared
   OBSOLETE". `[INFERENCE]` — an analogy across registries, not authority over
   frame types, but the shape matches §6.2's dual-emit path exactly.

9. **The standard's own precedent points the same way.** `R9.4` already freezes
   problem `type`/`code` pairs (`R5.13.2`) and reserved-name semantics (§1.10).
   A frame type is the same kind of object: a short machine-readable string on
   the wire that a client branches on, with no indirection layer. The standard
   already records a real failure of exactly this class — the .NET 7→8 problem
   `type` identity change (`baseline-02c`, `baseline-02f`) — as the reason
   `R5.13.2` exists. `[POLICY]`

### 5.2 Against — stated at full strength

1. **A major vendor renames on a GA surface, silently, and nothing broke
   loudly.** OpenAI's `response.code_interpreter_call.code.delta` →
   `response.code_interpreter_call_code.delta` shipped on the GA Responses
   surface with no beta marker, no version gate, and no changelog entry found
   (S6, S5). If a freeze were as load-bearing as §5.1 claims, this should have
   produced visible fallout; none was located. `[FACT]` on the rename;
   `[INFERENCE]` on the absence of fallout, which is an absence of evidence, not
   evidence of absence.

   **Weight.** Real but bounded. The renamed frame is a **non-terminal,
   payload-bearing** type. A client keyed on the old name silently loses code
   deltas — indistinguishable from "this run emitted no code" — so the failure
   is quiet data loss, not a visible error. This is an argument that the blast
   radius is *graded*, not that the freeze is wrong.

2. **A freeze immortalizes mistakes.** OpenAI's
   `response.code_interpreter_call.in.progress` — a name with a dot where an
   underscore belongs, inconsistent with every sibling — was fixed by renaming.
   Under a strict freeze that fix requires dual-publishing both names until a
   major version. The standard would be preserving a typo for the life of a
   major. `[FACT]` on the typo and its fix; `[POLICY]` on the cost assessment.

   **The cost is documented, by the project most committed to the freeze.**
   Kubernetes on its own `phase` field (S38): "The pattern of using `phase` is
   deprecated. Newer API types should use conditions instead. Phase was
   essentially a state-machine enumeration field, that contradicted
   system-design principles and hampered evolution, **since adding new enum
   values breaks backward compatibility**." An enumeration that could not be
   corrected in place had to be superseded wholesale. `[FACT]` This is the
   strongest published cost case located, and it is about an enumeration that
   was *not* declared growable — which is why `R13.5`'s growth statement is
   load-bearing rather than ceremonial.

3. **A vocabulary can be wrong in ways worse than a typo.** Gemini's
   `interaction.status_update` was replaced by a *set* of specific states
   (`interaction.in_progress`, `interaction.requires_action`, …) (S8). That is
   not cosmetic; it is a modeling correction that a freeze converts into
   permanent debt plus a parallel vocabulary. `[FACT]` on the change;
   `[INFERENCE]` on the characterization.

4. **Two references decline to make the promise at all.** AWS publishes nothing
   about `detail-type` compatibility in any of the four directions checked
   (S32). GitHub's breaking-change policy is written so that webhook event names
   fall outside it (S16). A rule the two largest platforms in the set do not
   have is a rule this standard is inventing, not codifying. `[FACT — verified
   negative]`

   **The specification layer declines too.** AsyncAPI's Message Object defines
   `name` as "A machine-friendly name for the message" and nothing more, and the
   spec issue asking for a declarable compatibility mode was closed as not
   planned (S42, S43). The OpenAPI 3.1.0 Discriminator Object carries no
   guidance on evolving a discriminator value set (S48). CloudEvents' *normative*
   spec is silent; only its non-normative Primer speaks (S33, S34). Four
   specifications that could have ruled this, and none did. `[FACT — verified
   negative]`

5. **Sentinel-based designs are indifferent to it.** An API terminating with a
   bare `data: [DONE]` sentinel (permitted under `R13.6`) has no terminal
   *name* to freeze; the catastrophic case in §5.1.2 does not arise. The rule's
   value is concentrated in typed-terminal designs. `[INFERENCE]`

6. **Naming conventions can obviate the rule.** Zalando rule 213 embeds the
   version in the name (`…email-changed.v2`), making reuse-with-new-meaning
   impossible by construction (S30). A standard could mandate the convention
   instead of the freeze. `[COMPARATIVE]`

### 5.3 How the two bodies of evidence reconcile

The against-evidence does not contradict the for-evidence; it refines it in
three specific ways, each of which the proposed rule text absorbs:

- **Blast radius is graded.** Renaming a *terminal* type breaks every stream's
  success path (Gemini, §5.1.2). Renaming a *non-terminal* type causes silent
  data loss (OpenAI, §5.2.1). Both are breaking; only the first is loud. The
  rule freezes both and does not pretend they are equally urgent.
- **A rename must remain *possible*, not free.** Four independent sources supply
  the same escape hatch — AIP-180 ("a service **may** add the new component, but
  **must not** remove the existing one", S19), the CloudEvents Primer ("produce
  both the old event and the new event for some time (potentially forever)",
  S34), Standard Webhooks ("create a new event type for each of the existing
  ones", S41), and GitHub's shipped practice of dual-delivering both names
  through a transition window (S50). That answers §5.2.2 and §5.2.3 without
  weakening the freeze, and it is the only path in the survey that anyone has
  actually run in production.
- **Silence is the defect, not the rename.** OpenAI's silent GA rename is not
  an argument that renames are safe; it is an argument that a vendor's own
  published contract (S3, which lists only additions) is too thin to prevent
  one. This standard should say the part OpenAI left unsaid.

---

## 5.4 Anti-patterns

Four failure shapes this evidence names. Each is sourced above; they are
collected here because `CONSTRAINTS.md` requires an anti-patterns section and a
reviewer should be able to read them without reconstructing §5.

| # | Anti-pattern | Why it fails | Evidence |
|---|---|---|---|
| AP-1 | **Renaming a frame type in place on a GA surface, unannounced** | Clients keyed on the old name silently stop matching. If the renamed type is terminal, `R12.10` turns every success into a reported truncation; if it is payload-bearing, the loss is silent and indistinguishable from "this stream emitted none of those". | OpenAI `response.code_interpreter_call.code.delta` → `..._call_code.delta` on the GA Responses surface, no changelog entry found (S6, S5); §3.2, §5.2.1 |
| AP-2 | **Reusing a retired frame-type name for a different meaning** | A client that was never updated keeps parsing the name and now misinterprets the payload — worse than ignoring it, because the failure is silent and semantic rather than structural. | No instance found anywhere in the survey; AIP-180 forbids it as remove-and-add (S19); RFC 6838 §5.5 shows the registry alternative — obsolete the name, never reassign it (S46); §3.4, §5.1.8 |
| AP-3 | **Publishing a frame-type vocabulary without declaring it growable** | Clients reasonably infer the set is exhaustive and branch exhaustively; the first addition then breaks them. This is the mechanism behind Kubernetes' position that adding a value is *not* compatible. | Kubernetes `api_changes.md` (S37) and the `phase` cost case (S38); the remedy is `R13.5`'s growth statement, which every deep-dive vendor also publishes (S1, S3, S9); §4, §5.1.1 |
| AP-4 | **Demoting a type from terminal, then renaming it** | Two changes that a name-only freeze permits individually compose into exactly the break the freeze exists to prevent. This is why §6.1 freezes terminality as well as names. | Composition hazard identified in the declined "terminal-only freeze" alternative, §7; no shipped instance located |
| AP-5 | **Relying on `Deprecation` and `Sunset` headers to retire one frame type** | Those are response headers, emitted once before any frame. They cannot scope to a single type inside a stream, so a provider that reaches for them has no working channel and effectively announces nothing. | `R9.5` binds response headers; §6.2 |

---

## 6. Proposed rule text

Two changes, both amendments to existing rules. No new rule identifier is
minted, matching the `baseline-04` decision record's practice of scoping
existing rules rather than growing the rule count.

### 6.1 Amendment to `R9.4` (§9.3, the frozen surface)

Add one item to the **Frozen** list:

> · documented stream frame-type names, their meanings, and which of them are
> terminal (`R13.5`, `R13.6`)

Add one item to the **Compatible** list:

> · adding stream frame types, where the API has documented its frame-type
> vocabulary as growable (`R13.5`)

Add one item to the **Breaking** list, since the generic entry "removing or
renaming any frozen element" does not by its wording catch a meaning change
that leaves the name alone:

> · changing whether a documented frame type is terminal, or changing what a
> documented frame type means, with or without a change of name

**Classification and confidence.**

| Clause | Classification | Confidence | Basis |
|---|---|---|---|
| Adding a frame type is compatible **where the vocabulary was documented growable** | `[COMPARATIVE]` | **high** | Seven independent published promises (S1, S3, S9, S13, S16, S22, S30). The one dissent (S37) attaches to vocabularies that never declared growth and prescribes the declaration as its own remedy, so the condition reconciles it. Identical in shape to `R4.9`'s open-enum rule already in `R9.4`. |
| Frame-type **names** are frozen within a GA major | `[COMPARATIVE]` | **moderate-high** | Four vendors gated every announced rename behind a version mechanism (S1, S8, S14, S4/S5/S7); GitHub dual-delivered rather than renamed in place (S50); Kubernetes freezes constant values to the API version (S36) with a vocabulary unchanged since 2014; AIP-180 and Azure state the general rule (S19, S22); CloudEvents' Primer and Standard Webhooks prescribe the same shape (S34, S41); protobuf explains why no encoding fix exists (S44). One contrary shipped instance on a GA surface (S6), unannounced and therefore not a published counter-policy. |
| Frame-type **meanings** are frozen | `[COMPARATIVE]` | **moderate-high** | Follows `R9.4`'s existing "request and response field names, types, and **meanings**" clause; AIP-180 treats a meaning change as remove-and-add (S19). |
| **Terminality** is frozen | `[POLICY]` | **moderate** | **This standard's own construction.** No source in the survey states it. Derived from the `R12.10`/`R13.6` interaction: demoting a terminal type without renaming it produces the same false-truncation failure as renaming it, and a pure name freeze would not catch it. |

### 6.2 Amendment to `R13.5` (§13.2, frame typing and vocabulary)

Append to the rule:

> The vocabulary documentation MUST mark, for each documented frame type,
> whether it is a terminal frame (`R13.6`). An API that must retire a documented
> frame type MUST NOT reuse its name for a different meaning, and MUST either
> defer the retirement to a new major version (`R9.4`) or emit both the retired
> and the replacement type for a documented overlap period, retiring the old
> name only at a major version. The overlap period MUST be stated in the
> vocabulary documentation, because `R9.5`'s `Deprecation` and `Sunset` headers
> are response headers and cannot deprecate one frame type within a stream.

**Classification and confidence.**

| Clause | Classification | Confidence | Basis |
|---|---|---|---|
| Terminality must be documented per type | `[POLICY]` | **moderate-high** | Makes the `R9.4` freeze checkable and discharges `R12.10`, which already requires a client to recognize "the documented terminal frame". Currently `R13.6` requires *a* documented terminal frame without requiring the vocabulary to mark which types those are. |
| A retired name MUST NOT be reused for a different meaning | `[COMPARATIVE]` | **high** | AIP-180: rename equals remove-and-add (S19). CloudEvents Primer: an incompatible data change "should generally change" the `type` (S34). Zalando rule 213 makes reuse impossible by convention (S30). |
| Dual-emit overlap as the in-major migration path | `[COMPARATIVE]` | **moderate-high** | AIP-180: "a service **may** add the new component, but **must not** remove the existing one" (S19). CloudEvents Primer: "produce both the old event and the new event for some time (potentially forever)" (S34). Standard Webhooks: mint a new event type for the new schema (S41). **GitHub has actually shipped it** — both names delivered concurrently until the old one was removed (S50, §3.4). Stripe's pinned-version mechanism achieves the same effect operationally (S13, S14). |
| Header channel unavailable; document the window | `[FACT]` on the mechanism, `[POLICY]` on the remedy | **high** | `R9.5` binds response headers; a stream's headers are sent once, before any frame, so they cannot carry per-frame-type deprecation. |

### 6.3 Scope decisions stated explicitly

- **Top-level frame types only.** Discriminator *sub*-vocabularies — Anthropic's
  `text_delta` / `input_json_delta` / `thinking_delta` / `signature_delta` inside
  a `content_block_delta` frame (S2) — are **already** covered by `R9.4`'s
  existing "request and response field names, types, and meanings" clause and by
  `R4.9`, because a sub-type is a field value inside a frame payload, and
  `R13.5` already binds frame payloads to §4's representation rules. The new
  clause therefore needs to reach only the frame's own type. `[POLICY]`
- **No new §1.10 registrations.** This report does not propose reserving any
  frame-type name. The `baseline-04` decision record ratified that declination;
  `error` remains the sole registered frame type. `[POLICY]`
- **Unbounded streams.** `R13.6` already carves out streams that are unbounded
  by design. Where an API documents no terminal frame, the terminality half of
  the freeze has nothing to bind and the name half still binds. `[POLICY]`

---

## 7. Declined alternatives

**A new standalone rule `R13.12` carrying the whole freeze.** Declined.
`R9.4` is the standard's single breaking-change taxonomy, and its frozen list
already names other rules parenthetically (`R5.13.2`, §1.10, `R6.6`). Putting
the freeze anywhere else would create a second place a reviewer must check to
answer "is this change breaking?", which is the exact defect the taxonomy exists
to remove. The only obligation `R9.4` genuinely cannot carry — per-type
terminality marking and the dual-emit path, because a taxonomy classifies
changes and does not impose documentation duties — is placed on `R13.5`, which
already owns the vocabulary-documentation duty.

**Reserving a canonical frame-type vocabulary in §1.10.** Declined, and out of
scope: already ratified as declined in `research/decisions/baseline-04-streaming.decision.md`.
Nothing found in this leaf disturbs that reasoning — if anything, the
divergence between Anthropic's `message_stop`, OpenAI's `response.completed`,
and Gemini's `interaction.completed` (all shipping simultaneously, S2, S6, S9)
confirms that the field's vocabularies are product-shaped.

**Mandating a version suffix inside frame-type names (the Zalando rule 213
convention).** Declined. It solves the problem elegantly (S30) but only one of
the eleven references uses it, and none of the three deep-dive providers does.
Mandating a naming convention that the entire streaming field ignores would
make the standard unimplementable against every shipped exemplar, and `R13.5`
deliberately leaves each API's vocabulary its own. Recorded as a permitted
option, not a requirement.

**Requiring an opt-in request flag before emitting any new frame type
(the Kubernetes `allowWatchBookmarks` pattern, S35).** Declined as
disproportionate. It is the safest possible way to add a type, but it converts
every vocabulary addition into an API change with a request-side parameter,
which contradicts the unanimous field position that additions are compatible
(§5.1.1) and would make `R13.5`'s growth statement meaningless. Recorded as a
practice an API MAY adopt for a type whose arrival changes client control flow.

**Freezing terminal frame names only, leaving non-terminal names free.**
Declined, but it is the closest genuine alternative and deserves its reasons.
For: it targets the catastrophic case exactly (§5.1.2) and licenses OpenAI's
observed non-terminal rename (§5.2.1) rather than declaring the field
non-conformant. Against: it makes the frozen surface depend on a property the
API itself assigns, so an API could demote a type from terminal and then rename
it — two individually-permitted steps composing into the exact break; it leaves
silent data loss unaddressed; and it splits one vocabulary into two
compatibility regimes, which no source in the survey does. The dual-emit path in
§6.2 delivers most of the flexibility without the composition hazard.

**Ruling nothing and leaving the §13.4 interim posture in place.** Declined.
The register's own text calls this "the sharpest known failure", and this leaf
now has eleven sourced references plus a shipped instance of the exact failure
(S8). Leaving a placeholder where evidence exists is the thin-evidence position
in reverse.

---

## 8. What could not be verified

Named explicitly, with the method used and the date.

1. **The exact date of OpenAI's GA Responses rename.** The
   `openai/openai-openapi` repository deleted `openapi.yaml` at `e1cb7a8`
   (2025-06-17) and re-added it at `5162af9` (2026-05-13), so `git log -S`
   cannot pinpoint the change. It is bounded to that window. Whether an
   announcement exists outside the changelog (an email, a forum post) was not
   determined; only S5 was searched, on 2026-08-10.
2. **Whether OpenAI's GA rename caused any client breakage.** No incident
   report, status-page entry, or issue thread was located. Absence of located
   evidence is not evidence of absence, and §5.2.1 is written to say so.
3. **The complete OpenAI Realtime beta-to-GA rename list.** Four renames were
   verified from the vendor's own specification (S6). The full mapping appears
   in a beta-to-GA migration section that S7's fetch did not return in full, and
   `https://developers.openai.com/api/docs/guides/realtime-beta-migration`
   returned HTTP 404 on 2026-08-10. Search-result summaries describing the full
   list were **not** used as sources.
4. **Which Gemini page governs on `interaction.status_update`.** S8 lists it as
   replaced; S9 still lists it as current. Both accessed 2026-08-10. Surfaced in
   §3.3 rather than resolved.
5. **Whether Gemini's Interactions API was formally labelled preview or beta
   during the May 2026 rename.** S10 dates its launch to 2025-12-11 and S12
   states GA "as of June 2026", which brackets the rename inside the pre-GA
   period; but no page fetched carries an explicit preview or beta label on the
   Interactions surface during May 2026, and no versioned URL path (`v1` versus
   `v1beta`) was obtainable from S12. The pre-GA classification in §3.3 rests on
   the two dates, not on a label.
6. **A specific Twilio event-type discontinuation instance.** The lifecycle
   (S25) is published; no dated example of a discontinued event type was located
   in the Twilio changelog, which was not exhaustively searched (2026-08-10).
7. **Any Azure Event Grid `eventType` rename.** Searched 2026-08-10; none found.
   The absence may reflect Event Grid's publisher-defined model rather than a
   stability guarantee.
8. **Any IETF standards-track or draft treatment of event-type versioning.**
   Searched 2026-08-10; none found. This is the single largest gap: the entire
   recommendation rests on `[COMPARATIVE]` vendor evidence and `[POLICY]`, with
   no protocol authority behind it. Stated in §4 and repeated here so it cannot
   be missed.
9. **Whether any vendor has ever *removed* a frame type from a GA streaming
   vocabulary without replacement.** Every removal located was either a rename
   with a replacement or a whole-API retirement (OpenAI Assistants, S4). No bare
   removal instance was found.
10. **The exact date GitHub removed the `integration_installation` webhook
    events.** GitHub retired the `developer.github.com/changes/` changelog path
    and the primary announcement URL returns 404 (checked 2026-08-10). Only the
    dual-delivery *pattern* is claimed in §3.4; no date is.
11. **Whether Shopify's webhook *topic names* are stable across API versions.**
    Secondary sources indicate the `api_version` governs payload structure while
    topic names do not move, which would make Shopify a clean instance of
    "freeze the name, version the payload". `https://shopify.dev/docs/apps/webhooks/versioning`
    was not directly fetched and quoted, so this is **excluded from the evidence
    in §5** rather than counted.
12. **Whether any published guidance argues affirmatively that frame-type names
    *should* be renamed rather than frozen.** None was found. The nearest is the
    CloudEvents Primer's "put a version in the type name" option (S34) and
    Zalando rule 213 (S30) — both of which produce a *new* name rather than
    mutating an existing one, so neither is genuine contrary guidance. Searched
    2026-08-10. §5.2 therefore rests on one shipped instance (S6) and on cost
    arguments, not on any source advocating renaming.
13. **A specific event-type-name typo publicly regretted by its vendor.** The
    OpenAI `.in.progress` case (S6) is inferred from the spec diff, not from any
    vendor statement about it. No vendor post-mortem on a misnamed event type
    was located (searched 2026-08-10).

---

## 9. Provisional rule identifiers

Per `CONSTRAINTS.md`, a baseline report proposes; it does not ratify. Nothing
here is binding until a record lands in `research/decisions/`.

| Provisional ID | Target | Strength | Classification | Confidence |
|---|---|---|---|---|
| `FV-001` | `R9.4` Frozen list: frame-type names, meanings, terminality | MUST (by inclusion in the frozen surface) | `[COMPARATIVE]` on names and meanings, `[POLICY]` on terminality | moderate-high / moderate |
| `FV-002` | `R9.4` Compatible list: adding frame types where documented growable | permitted | `[COMPARATIVE]` | high |
| `FV-003` | `R9.4` Breaking list: changing terminality or meaning with or without a rename | MUST NOT within a major | `[POLICY]` | moderate |
| `FV-004` | `R13.5`: vocabulary documentation marks terminality per type | MUST | `[POLICY]` | moderate-high |
| `FV-005` | `R13.5`: no name reuse; dual-emit overlap or defer to a major | MUST NOT / MUST | `[COMPARATIVE]` | moderate-high |
