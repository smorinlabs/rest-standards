# baseline-02i — Operation-resource discovery on 202 (report, 2026-08-10)

Research leaf under `AC-019` / rule `R10.1`. Series `baseline` = prescriptive:
this report proposes; only a ratified record in `research/decisions/` binds.

Label key: **[FACT]** primary-sourced · **[COMPARATIVE]** surveyed practice ·
**[INFERENCE]** reasoning from the above · **[POLICY]** a judgment the project
must make.

---

## 1. TL;DR and recommendation

**Verdict: SHOULD `Location` — but only as the second half of a two-part rule.
The first half is a new MUST that the `202` body identify the operation.**

Pure `keep-permitted` cannot be the answer. `R10.1` already requires the
operation resource to be *addressable*, and Appendix E.7's `202` body carries
only `"id": "op_000example"` — so today the standard mandates addressability
while naming no mechanism by which the address is learned. GitHub's repository-
statistics `202` is the live demonstration of that failure mode: `202` with no
`Location`, no body identity, and no job resource at all **[FACT]**, which is
precisely what `AC-019` was ratified to forbid.

`MUST Location` also fails, on three independent grounds developed in §5:
RFC 9110 defines no `Location` semantics for `202`; `Location` is not
CORS-safelisted, so a browser client cannot read it cross-origin unless the
server also emits `Access-Control-Expose-Headers` (a header this standard
currently never mentions); and **[COMPARATIVE]** 14 of the 17 surveyed running
APIs — including all three AI providers — carry identity in the body and emit no
such header at all.

**Proposed rule sentence** (new `R10.1a`, or an added paragraph on `R10.1`):

> **R10.1a** A `202 Accepted` MUST identify its operation resource in the
> response body — either the operation's `id`, where the operation resource's
> URI template is documented in the description document (R4.1), or an absolute
> `url` member. The `202` SHOULD additionally carry a `Location` header whose
> value is the absolute URI **of the operation resource**, never of the eventual
> result; where both are present they MUST denote the same resource. A `202`
> that carries neither strands the client and violates R10.1.
>
> > Provenance: research leaf `baseline-02i`, riding `AC-019` · body clause is
> > protocol-grounded (RFC 9110 §15.3.3: the representation "ought to ... point
> > to (or embed) a status monitor") · the `Location` SHOULD is `[POLICY]` —
> > RFC 9110 defines no `Location` semantics for `202` · confidence
> > moderate-high (body) / moderate (header).

Strength rationale, in one line each:

| Clause | Strength | Why not stronger | Why not weaker |
|---|---|---|---|
| body identity | **MUST** | — | `R10.1`'s "addressable" is unenforceable without it; the header alone is unreadable to browser clients |
| `Location` | **SHOULD** | no RFC semantics for `202`; CORS-invisible; 0/3 AI providers emit it | it is the only carrier a generic client can follow with zero vendor knowledge, and the only way to point at a monitor that is not template-constructable or not same-origin (Microsoft Graph) |
| `Location` = the **operation**, not the result | **MUST** (definitional) | — | ARM and Zalando's fallback both use `Location` for the eventual *result*; leaving it undefined imports that ambiguity |

**Two consequential edits the decision must carry with it** (otherwise the fix
is half-applied):

1. **§1.10 register, `Location` row** (rest-api-standard.md:268) currently reads
   that the `202` operation-resource pointer is "permitted, not restricted."
   A SHOULD binding must be recorded there, exactly as the `201` binding is.
2. **Appendix F/OpenAPI row**: `202` responses declare a `Location` header
   object plus the operation schema's identity member — both are lintable, in
   the same way `R5.6`'s `201` + `Location` header object already is
   (rest-api-standard.md:1942).

**Open `[POLICY]` sub-question the owner must settle:** whether the body member
is a bare `id` (OpenAI / Gemini / Stripe / Shopify pattern) or an absolute
`url` (Adobe Substance 3D / GitHub migrations / Anthropic `results_url`
pattern). A `url` member is strictly more robust and is *required* if the
monitor can live off-origin — but it edges toward a self-link, and §1.2 declines
hypermedia controls. My reading: a plain absolute `url` scalar is not a
hypermedia control (no link-relation graph, no runtime-discovered state
machine), so it is compatible; if the project disagrees, `id` + a documented
template is sufficient and is what the majority of surveyed vendors do. If `url`
is chosen, `R1.8` reservation discipline requires registering the name.

---

## 2. Standards layer

**RFC 9110, *HTTP Semantics*** — Internet Standard, STD 97 (IETF standard,
authority class: IETF standards-track). Source: raw text at
https://www.rfc-editor.org/rfc/rfc9110.txt, downloaded and read 2026-08-10.

**[FACT] §10.2.2 `Location` — the operative sentences:**

> The "Location" header field is used in some responses to refer to a
> specific resource in relation to the response.  The type of
> relationship is defined by the combination of request method and
> status code semantics.

> For 201 (Created) responses, the Location value refers to the primary
> resource created by the request.  For 3xx (Redirection) responses,
> the Location value refers to the preferred target resource for
> automatically redirecting the request.

**[INFERENCE]** Do **not** state that RFC 9110 forbids `Location` outside
201/3xx. It does not. It defines the relationship only for those two, and says
the relationship is a function of method + status. For `202` the relationship is
**undefined, not prohibited** — which is exactly the space a house standard (or
ARM, or Microsoft Graph) may legitimately define. Writing the rule *is* the act
of supplying the missing definition, which is why the clause "of the operation
resource, never of the eventual result" is load-bearing rather than decorative.

**[FACT] §15.3.3 `202 Accepted` — the only 202-specific guidance in the RFC:**

> The 202 response is intentionally noncommittal.  Its purpose is to
> allow a server to accept a request for some other process ... without
> requiring that the user agent's connection to the server persist
> until the process is completed.  The representation sent with this
> response ought to describe the request's current status and point to
> (or embed) a status monitor that can provide the user with an
> estimate of when the request will be fulfilled.

**[INFERENCE]** These two sections pull in opposite directions and that tension
is the honest framing of this whole leaf. §10.2.2 leaves the header undefined
for `202`; §15.3.3 names *the representation* — the body — as the thing that
points to the status monitor. The RFC's own preferred carrier is therefore the
body. That is the single strongest argument for making the body clause the MUST
and the header clause the SHOULD, and it aligns the standard with the RFC rather
than against it.

**[FACT] Contrast — the `201` texts that `R5.6` rests on** (same source, same
access date). §15.3.2: "The primary resource created by the request is
identified by either a Location header field in the response or, if no Location
header field is received, by the target URI." §9.3.3 (POST): "the origin server
SHOULD send a 201 (Created) response containing a Location header field that
provides an identifier for the primary resource created ... **and** a
representation that describes the status of the request while referring to the
new resource(s)." `R5.6`'s provenance line is accurate: the RFC-level strength is
SHOULD, and the standard's unconditional MUST is a `[POLICY]` tightening.

**[FACT] IANA HTTP Field Name Registry**
(https://www.iana.org/assignments/http-fields/http-fields.xhtml, accessed
2026-08-10; authority class: IANA registry): `Location` — status **permanent**,
reference RFC 9110 §10.2.2. `Operation-Location` — **absent**.
`Azure-AsyncOperation` — **absent**.

**[INFERENCE]** This settles the header-*name* question independently of the
header-*strength* question. If this standard emits any header for operation
discovery it must be `Location`. Adopting Azure's `operation-location` would
mean this standard mandating an unregistered vendor field — the same defect the
project already flagged for `Idempotency-Key` and `RateLimit`, and it is
additionally discouraged by RFC 6648, which the project already cites in the
`request-id` register row.

**[FACT] RFC 9110 §7.6.1** (same source): intermediaries "SHOULD remove or
replace fields that are known to require removal before forwarding" and
enumerates them — `Proxy-Connection`, `Keep-Alive`, `TE`, `Transfer-Encoding`,
`Upgrade`. **[INFERENCE]** There is therefore no RFC basis for the folk claim
that intermediaries strip arbitrary end-to-end response headers. The real
header-visibility exposure is CORS (§5), not proxies. Do not argue the
header-stripping point in the decision record; it does not survive contact with
the spec.

**[FACT] IANA Link Relation Types** (accessed 2026-08-10): `status` —
"Identifies a resource that represents the context's status." [RFC 8631];
`monitor` — "Refers to a resource that can be used to monitor changes in an
HTTP resource." [RFC 5989]. A registered vocabulary for this exists and is
unused by every vendor surveyed.

### Standards-track work in flight — neither is a standard

| Draft | Status (verified 2026-08-10) | What it says about discovery |
|---|---|---|
| `draft-wright-http-progress-02`, *Reporting Progress of Long-Running Operations in HTTP* | **EXPIRED**, individual submission; rev 02 dated 2019-11-21, expired 2020-05-24, never adopted, no replacement. https://datatracker.ietf.org/doc/draft-wright-http-progress/ | `202` + `Location` identifying the status document |
| `draft-ratnawat-httpapi-async-problem-details-00`, *Problem Details for Asynchronous Job Failures* | **ACTIVE but individual** (not WG-adopted); published 2026-02-26, **expires 2026-08-30**, intended Standards Track. https://datatracker.ietf.org/doc/draft-ratnawat-httpapi-async-problem-details/ | §3.2 [FACT, verbatim]: "If the job was submitted via HTTP and the server returned a 'Location' header pointing to a job status resource, the 'jobId' value SHOULD match the identifier embedded in that URI." §6.5 [FACT, verbatim]: "Servers that return 202 Accepted SHOULD include a 'Link' header with a relation type that points to the job status resource." |

**[FACT]** Verbatim quotes above are from
https://www.ietf.org/archive/id/draft-ratnawat-httpapi-async-problem-details-00.txt,
accessed 2026-08-10, which also carries "This Internet-Draft will expire on
30 August 2026."

**[POLICY]** Neither may be cited as normative. Note the second one's §6.5
recommends the RFC 8288 `Link` header — the exact mechanism `R6.4` bans for
pagination. Surfaced as a conflict in §5, not averaged away.

**[FACT] Kubernetes API conventions** — verified negative. The full
`api-conventions.md` (2231 lines,
https://raw.githubusercontent.com/kubernetes/community/master/contributors/devel/sig-architecture/api-conventions.md,
accessed 2026-08-10) contains zero occurrences of `202` and zero of `Location`
in any casing; the status-code enumeration omits `202` entirely. Kubernetes
expresses asynchrony through the `status` subresource and `observedGeneration`,
not an operation URI. This is an architectural absence, not a search failure —
and it means k8s supplies no evidence either way.

---

## 3. Vendor survey

All rows verified 2026-08-10 against the vendor's own document. Authority class
is "guideline doc" for the four guideline rows and "vendor doc" for the rest.

### 3a. Guideline documents

| Guideline | Mechanism | Strength | Primary source |
|---|---|---|---|
| **Azure REST API Guidelines** | `operation-location` **response header**, absolute URL; plus `Operation-Id` header | **DO** (its top tier) for the `202` LRO path; **YOU SHOULD** for the `201`-PUT variant. **Verified negative:** the file contains no `Location` LRO rule and never mentions `Azure-AsyncOperation` | `https://raw.githubusercontent.com/microsoft/api-guidelines/vNext/azure/Guidelines.md` |
| **Microsoft REST API Guidelines §13.2** (historical) | Examples emit `Operation-Location`; the **only** normative sentence says "the location header". §13.2.5 puts `resourceLocation` in the completed operation's **body** | **SHOULD** — and it names `location`, not `Operation-Location`. Internally inconsistent (see §5) | `https://raw.githubusercontent.com/microsoft/api-guidelines/28bc3ef56742/Guidelines.md` (commit `28bc3ef56742`, 2023-03-24; the `vNext` root file is now a redirect stub) |
| **Zalando RESTful API Guidelines, rule 253** | Recommended: `201` + "the `job-id` passed in the response payload **and/or** via the URL of the `Location` header". Fallback: `202` + `Location` | **MAY** ("MAY support asynchronous request processing") — the whole rule | `https://opensource.zalando.com/restful-api-guidelines/#253` |
| **Google AIP-151** | **Body field only** — `Operation.name`; HTTP mapping `GET /v1/{name=operations/**}`. **Verified negative:** AIP-151 mentions no header and no `202` at all | methods "**should** return a `google.longrunning.Operation`"; response type "**must** be" that type | `https://google.aip.dev/151` + `https://raw.githubusercontent.com/googleapis/googleapis/master/google/longrunning/operations.proto` |

**[FACT] Azure, verbatim:** ":white_check_mark: **DO** include an
`operation-location` response header with the absolute URL of the status monitor
for the operation." and ":white_check_mark: **DO** return a `202-Accepted`
status code from the request that initiates an LRO action on a resource".

**[FACT] Microsoft REST API Guidelines §13.2.7, verbatim:** "The server MUST
indicate the request has been started by responding with a 202 Accepted status
code. The response SHOULD include the location header containing a URL that the
client should poll for the results after waiting the number of seconds specified
in the Retry-After header."

**[FACT] Zalando rule 253, verbatim:** "`POST /report-jobs` returns HTTP status
code `201` to indicate successful initiation of asynchronous processing together
with the `job-id` passed in the response payload and/or via the URL of the
`Location` header." — Zalando is the only guideline surveyed that explicitly
blesses **both** carriers.

**[FACT] ARM RPC Addendum — flagged evidence, fork-mirror only.** The canonical
`https://github.com/Azure/azure-resource-manager-rpc` returns **HTTP 404** as of
2026-08-10 (verified via WebFetch and the GitHub API). Quotes come from a public
fork at
`https://raw.githubusercontent.com/AzureExpert/azure-resource-manager-rpc/master/v1.0/Addendum.md`:
"The response headers **MUST** include a Location header that points to a URL
where the ongoing operation can be monitored"; `Azure-AsyncOperation` is
"**Optional**". **This row does not meet the two-source bar** and must not be
cited as a MUST-level precedent. The commonly repeated precedence rule
("`Azure-AsyncOperation` wins over `Location`") is **not in the RPC text** — it
appears only as descriptive client prose on Microsoft Learn
(`https://learn.microsoft.com/en-us/azure/event-grid/async-operations`, accessed
2026-08-10: "If your operation returns this value, always use it (instead of
Location) to track the status of the operation"), with no RFC 2119 keywords.

### 3b. Running APIs

| Vendor / API | Create status | Mechanism | Primary source (accessed 2026-08-10) |
|---|---|---|---|
| **Microsoft Graph** (driveItem copy) | `202` | **`Location` header only — empty body** | `https://learn.microsoft.com/en-us/graph/long-running-actions-overview` (ms.date 2024-11-07) |
| **Adobe PDF Services** | `201` | **`location` response header** | `https://developer.adobe.com/document-services/docs/overview/pdf-services-api/gettingstarted` |
| **Adobe Substance 3D** | `202` | **Body** `url` + `id` (self URL); poll URL expires 2 min after completion | `https://developer.adobe.com/firefly-services/docs/s3dapi/getting-started/asynchronous-jobs/` |
| **Adobe Firefly** | not documented (flagged) | **Body** `jobId`, `statusUrl`, `cancelUrl` (absolute URLs) | `https://developer.adobe.com/firefly-services/docs/firefly-api/guides/how-tos/using-async-apis` |
| **Stripe** (Reporting Report Runs) | `200` | **Body** `id` + `status`; completion also pushed by `reporting.report_run.succeeded` webhook | `https://docs.stripe.com/api/reporting/report_run/create.md` + `https://docs.stripe.com/api/errors.md` |
| **GitHub** repo statistics | `202` | **NONE** — no header, no body identity, no job resource; client retries the same URL | `https://docs.github.com/en/rest/metrics/statistics` |
| **GitHub** org migrations | `201` | **Body** `id`, `url`, `state` | `https://docs.github.com/en/rest/migrations/orgs?apiVersion=2022-11-28` |
| **Twilio** BulkExport custom job | `201` (from Twilio's own OpenAPI spec) | **Body `job_sid` only** — the create-response schema has **no** `url`; `url` appears only on the later GET | `https://www.twilio.com/docs/usage/bulkexport/export-custom-job` + `https://raw.githubusercontent.com/twilio/twilio-oai/main/spec/json/twilio_bulkexports_v1.json` |
| **Shopify** GraphQL bulk operations | HTTP 200 (GraphQL) | **Body** `bulkOperation.id` (a GID, not a URL) | `https://shopify.dev/docs/api/usage/bulk-operations/queries` |
| **Shopify** REST Checkout (deprecated as of API version 2024-07) | **`202`** | **Body `token`**; no `Location` in the documented examples. The later `GET …/checkouts/{token}.json` also returns `202` while still processing | `https://shopify.dev/docs/api/admin-rest/unstable/resources/checkout` |
| **Atlassian** CSM REST | not stated | **Body** task ID → `GET /api/v1/tasks/{taskId}` | `https://developer.atlassian.com/cloud/customer-service-management/rest/v1/intro/` |
| **AWS Textract** (contrast) | `200` | **Body `JobId`** — an opaque ID with no URI at all; consumed by a differently-named action | `https://docs.aws.amazon.com/textract/latest/dg/API_StartDocumentTextDetection.html` |

**[FACT] Microsoft Graph, verbatim:** "The API accepts the action and returns a
`202 Accepted` response along with a `Location` header for the API URL to
retrieve action status reports." The worked example is `HTTP/1.1 202 Accepted` +
`Location:` and **no body whatsoever**. Two notes on that page are load-bearing
for §5: "The location URL returned might not be on the Microsoft Graph API
endpoint," and the monitor "doesn't require authentication, because the URL is
short-lived and unique to the original caller."

**[FACT] GitHub statistics, verbatim:** "If the data hasn't been cached when you
query a repository's statistics, you'll receive a `202` response; a background
job is also fired to start compiling these statistics." followed by "You should
allow the job a short time to complete, and then submit the request again."
No `Location`, no body identity, no operation resource.

**[COMPARATIVE] Two flagged non-verifications, restated plainly:** (a) Adobe
Firefly's status code is not stated on its own how-to page — the body mechanism
is confirmed, the `202` is not; (b) Atlassian Jira Cloud's widely-paraphrased
"`303` + `Location`" could not be retrieved verbatim from the vendor domain
across four fetches and is **unverified** — do not cite it.

**[COMPARATIVE] Prior in-repo corroboration.** `survey-05` (2026-07-19,
`research/reports/survey-05-reliability.report.2026-07-19.md`) independently
recorded the same split: "Long-running work splits between an **operation
resource** you poll (Google AIP-151 `operations/{id}`; Microsoft Graph
RELO/stepwise) and **header-driven polling** (Azure `202` +
`Azure-AsyncOperation`/`Location` + `Retry-After`); Stripe stays largely
synchronous and pushes completion via webhook events." That satisfies the
two-source bar for the header/body split itself.

---

## 4. The AI-provider row set

Mandatory per project rule. All accessed 2026-08-10.

| Provider | Create status | Identity carrier | Poll URL | Primary source |
|---|---|---|---|---|
| **OpenAI** Batch — `POST /v1/batches` | **`200`** ("Batch created successfully.") | **Body** `id` = `"batch_abc123"`, plus `status` | `GET https://api.openai.com/v1/batches/{batch_id}` | `https://raw.githubusercontent.com/openai/openai-openapi/master/openapi.yaml` (OpenAI's own spec; commit `c309ca176bc22c6075a0c2c2543f2ac4f307c447`, 2026-08-08). The rendered reference at `platform.openai.com` returns **403** to automated fetch |
| **OpenAI** fine-tuning — `POST /v1/fine_tuning/jobs` | **`200`** ("OK") | **Body** `id` = `"ftjob-abc123"` + `status` | `GET /v1/fine_tuning/jobs/{fine_tuning_job_id}` | same spec |
| **Anthropic** Message Batches — `POST /v1/messages/batches` | **not documented** (flagged) | **Body** `id` = `"msgbatch_…"`, `processing_status`, `expires_at`; and `results_url`, a body-carried **absolute URL**, populated only once processing ends | `GET https://api.anthropic.com/v1/messages/batches/{id}` | `https://platform.claude.com/docs/en/api/creating-message-batches` (`docs.claude.com` 301-redirects here) — **verified twice, independently** |
| **Google Gemini** batch mode — `POST …:batchGenerateContent` | **not documented** (flagged) | **Body** — a `google.longrunning.Operation` with `name: "batches/123456789"` | `GET https://generativelanguage.googleapis.com/v1beta/{name=batches/*}` | `https://ai.google.dev/api/batch-mode` + Google's discovery doc `https://generativelanguage.googleapis.com/$discovery/rest?version=v1beta` — **verified twice** |
| **Google Vertex AI** batch prediction (contrast) | not stated | **Body** — the `BatchPredictionJob` resource itself (`name` + `state`), **not** an `Operation` | `GET …/batchPredictionJobs/{id}` | `https://aiplatform.googleapis.com/$discovery/rest?version=v1` (the rendered HTML reference 404s / renders nav-only) |

**[FACT] Anthropic create response, verbatim** (abridged to the operative
members; I retrieved this page directly as well as via the survey agent):

```json
{
  "id": "msgbatch_013Zva2CMHLNnXjNJJKqJ2EF",
  "created_at": "2024-08-20T18:37:24.100435Z",
  "expires_at": "2024-08-20T18:37:24.100435Z",
  "processing_status": "in_progress",
  "request_counts": { "canceled": 10, "errored": 30, "expired": 10,
                      "processing": 100, "succeeded": 50 },
  "results_url": "https://api.anthropic.com/v1/messages/batches/msgbatch_013Zva2CMHLNnXjNJJKqJ2EF/results",
  "type": "message_batch"
}
```

and the operative prose: "To poll a Message Batch, you'll need its `id`, which
is provided in the response when creating a batch or by listing batches."

**[FACT]** The OpenAI spec's omission of `Location` on `/batches` is deliberate,
not stylistic: the same 84,016-line file **does** declare a `Location` response
header — on a `201` ("Realtime call created successfully"), with the description
"Relative URL containing the call ID for subsequent control requests." A grep of
the whole file for `operation-location` returns zero hits.

**[FACT] Findings that bear directly on the decision:**

1. **Zero of the three AI providers emits any discovery header.** All three use
   body-carried identity.
2. **None of the three uses `202`.** Two do not document a success code at all;
   OpenAI documents `200`. The AI-provider evidence is therefore evidence about
   *carrier*, and near-silent about `202` specifically. **[INFERENCE]** Do not
   over-read it as "the industry rejects `Location` on `202`" — it is "the
   industry does not use `202` for batch job creation."
3. **Two body-carried-URL precedents exist**: Anthropic's `results_url` and
   GitHub migrations' `url`. Both are absolute URLs handed over verbatim rather
   than constructed by the client — the same affordance a `Location` header
   provides, delivered in the body.
4. **Google is internally split on shape** (Gemini wraps in `Operation`; Vertex
   returns the job resource directly) while agreeing on mechanism (body `name`).

---

## 5. Conflict and trade-off analysis

### 5.1 Conflicts, surfaced rather than averaged

**Conflict A — Microsoft contradicts itself three ways.** Azure guidelines:
`operation-location`, DO-level, no `Location` rule at all. ARM RPC: `Location`
MUST, `Azure-AsyncOperation` optional. Microsoft Graph and the historical
Microsoft REST API Guidelines §13.2.7: `Location`, SHOULD — while §13.2's own
examples emit `Operation-Location`. **Which governs for this standard: none of
them on header *name*; the IANA registry does.** `Location` is permanently
registered (RFC 9110 §10.2.2); `Operation-Location` and `Azure-AsyncOperation`
are unregistered, and RFC 6648 (already cited in this standard's `request-id`
register row) discourages minting new non-standard fields. A house standard that
mandated `operation-location` would be repeating the defect the project already
flagged for the expired `Idempotency-Key` draft. **[POLICY]** If a header is
used, it is `Location`.

**Conflict B — RFC 9110 §15.3.3 vs §10.2.2.** §15.3.3 names the *representation*
as the thing that points to the status monitor; §10.2.2 leaves `Location`'s
meaning on `202` undefined. **Which governs: §15.3.3**, because it is the only
`202`-specific text in the RFC and it speaks to exactly this question. That
makes the body the RFC-preferred carrier and the header a legitimate but
extra-spec house extension — hence MUST body, SHOULD header, and not the
reverse.

**Conflict C — `draft-ratnawat` §6.5 recommends an RFC 8288 `Link` header** for
job-status discovery, which is the precise mechanism `R6.4` bans for pagination.
**Which governs: the project's ratified posture.** The draft is an individual
submission expiring 2026-08-30 and is not adoptable evidence; and adopting a
`Link` header here after banning it for pagination would be an internal
inconsistency far larger than the one this leaf is trying to close.

**Conflict D — `Location` means two different things in the wild.** In ARM and
in Zalando's `202` fallback, the `Location` URL eventually yields *the final
result*. In Microsoft's §13.2.3 example and in Graph, `Location`/
`Operation-Location` points at *the operation/monitor* while the result is
identified separately (`resourceLocation`, `resourceId`). **[INFERENCE]** A bare
"SHOULD send `Location`" would import this ambiguity wholesale. The proposed rule
resolves it explicitly, and that clause is the most valuable part of the rule —
more valuable than the strength keyword itself.

### 5.2 Header vs body, on the merits

| Axis | Header (`Location`) | Body member |
|---|---|---|
| Generic client can follow with zero vendor knowledge | **Yes** — one registered field, one meaning | No — the client must know the member name and, for a bare `id`, the URI template |
| Readable by a browser JS client cross-origin | **No**, unless the server also sends `Access-Control-Expose-Headers: Location` | **Yes** — the body is always readable once the CORS check passes |
| Can point at a monitor that is off-origin / non-constructable / separately authenticated | **Yes** (Microsoft Graph does exactly this) | Only if the member is an absolute `url`, not a bare `id` |
| Survives intermediaries | Yes — **[FACT]** RFC 9110 §7.6.1 lists only connection-specific fields for removal; no RFC basis for a general stripping fear | Yes |
| Expressible and lintable in OpenAPI 3.1 | Yes (header object — `R5.6` already does this for `201`) | Yes (schema member) |
| Survives a `HEAD`-like or empty-body response | Yes | No |
| Matches surveyed practice **[COMPARATIVE]** | **2** of the 17 running APIs in §3b/§4 (Microsoft Graph, Adobe PDF Services) — plus 3 of the 4 guideline docs | **14** of 17, including 3 of 3 AI providers. The 17th, GitHub statistics, has neither |

**[FACT] The CORS asymmetry, primary-sourced.** WHATWG Fetch Standard (Living
Standard, https://fetch.spec.whatwg.org/, accessed 2026-08-10): the
CORS-safelisted response-header names are `Cache-Control`, `Content-Language`,
`Content-Length`, `Content-Type`, `Expires`, `Last-Modified`, `Pragma`.
`Location` is **not** among them; a response's exposed headers "will typically
get its CORS-exposed header-name list set by extracting header values from the
`Access-Control-Expose-Headers` header." Corroborated (second source, authority
class: vendor/reference documentation) by MDN,
https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Access-Control-Expose-Headers,
accessed 2026-08-10, verbatim: "Only the CORS-safelisted response headers are
exposed by default. For clients to be able to access other headers, the server
must list them using the `Access-Control-Expose-Headers` header." — followed by
the same seven-name list. **[INFERENCE]** A browser client
calling a cross-origin API therefore cannot read `Location` on a `202` unless
the server opts in with a header this standard never mentions — a grep of
`rest-api-standard.md` for "CORS" and "Access-Control" returns nothing. This is
the single strongest argument against `MUST Location` as the *sole* mechanism,
and it is a latent gap in the standard regardless of how this leaf lands: the
same reasoning applies to `ETag`, `request-id`, `Retry-After`, and `RateLimit`,
all of which this standard already mandates. **[POLICY]** Worth spinning out as
its own gap-review item; it is out of scope here.

### 5.3 Does the pagination posture cut the same way?

**No — and the standard's own `R5.6` proves it.** The ratified pagination
justification is: "one source of truth — dual emission creates two places a
cursor can live, which drift under maintenance"
(`research/decisions/baseline-02-api-contracts.decision.md:299`). Three
disanalogies, each independently sufficient:

1. **The standard already practises dual emission for exactly this shape.**
   `R5.6` (rest-api-standard.md:787) requires `201 Created` with a `Location`
   header; the standard's own worked example E.2 (rest-api-standard.md:1999)
   then emits **both** that `Location` and a full body representation carrying
   `"id": "ord_000example"`. A conforming `201` therefore carries the new
   resource's identity twice — header URI and body `id` — and the project
   ratified that without treating it as a one-source-of-truth violation. An
   operation URI on a `202` is the same shape, one status code over.
   **Precision note:** `R5.6` itself mandates only the header, not the body
   representation — the §5.2 quick map row reads "Create (single resource) |
   201 + `Location`" with no body clause, and no other rule mandates a create
   response body (`R4.3` governs its *shape* if present, not its presence). So
   this disanalogy rests on the standard's ratified example and normal practice,
   not on a second MUST. It is still decisive: whatever the strength, the
   project plainly does not read "identity in a header and in the body" as the
   defect `R6.4` names.
2. **Cursors are volatile; an operation URI is minted once.** A cursor is
   recomputed on every page and its encoding changes under maintenance — which
   is what "drift" names. An operation resource's URI is assigned at creation
   and is immutable for the operation's lifetime (bounded by `R10.1`'s expiry).
   There is no second place for it to drift *to*: the header and the body member
   are two renderings of one immutable value, and the proposed rule makes their
   equality a MUST.
3. **Pagination's `Link` header was strictly redundant; here neither carrier is
   guaranteed.** `R6.1` already mandates a body envelope carrying continuation
   state, so RFC 8288 `Link` added a second copy of information the body was
   *required* to have. `R10.1` mandates no identity member at all, and E.7's
   `202` body carries only `id` with no stated URI relationship. Rejecting the
   header here would not be "keeping one source of truth" — it would be leaving
   zero *specified* sources and relying on an unstated convention.

**[INFERENCE]** The correct reading of the one-source-of-truth principle is
therefore *one normative source plus consistent redundancy*, not *one carrier*.
The proposed rule follows it precisely: the body member is the normative source
(so a client that ignores headers is never stranded), `Location` is a redundant
convenience hint, and the MUST-denote-the-same-resource clause makes divergence
a conformance failure rather than a maintenance hazard.

---

## 6. Confidence, and what would invalidate this

**Confidence by clause:**

| Clause | Confidence | Basis |
|---|---|---|
| Body MUST identify the operation | **high** | RFC 9110 §15.3.3 names the representation; **[COMPARATIVE]** 14/17 running APIs, 3/3 AI providers; closes `R10.1`'s enforceability gap |
| `Location` at SHOULD (not MUST, not merely permitted) | **moderate** | No RFC semantics for `202`; CORS-invisible; but it is the only zero-knowledge carrier and Graph proves it is sometimes the *only* possible one. This is the judgment call in the leaf |
| `Location` denotes the operation, not the result | **high** | Conflict D is documented across four sources; leaving it open is a known defect |
| Header name must be `Location`, never `Operation-Location` | **high** | IANA registry (permanent vs absent) + RFC 6648, corroborating |
| ARM RPC "`Location` MUST" as precedent | **low — do not rely on it** | Fork-mirror only; canonical repo 404s; fails the two-source bar |

**What would change the recommendation:**

1. **A ratified scope decision that browser/JS clients are out of scope**, *and*
   a guarantee that the operation resource is always same-origin and
   template-constructable, would remove both objections to `MUST Location`.
   Alternatively, adding a CORS rule that mandates
   `Access-Control-Expose-Headers` for every header this standard binds would
   neutralize the CORS objection on its own — and should be considered
   regardless of this leaf's outcome.
2. **WG adoption of `draft-ratnawat-httpapi-async-problem-details`** (currently
   individual, expires 2026-08-30) would introduce standards-track text
   recommending a `Link` header for job status. That would force an explicit
   reconciliation with `R6.4` rather than the current clean dismissal. Re-check
   the datatracker entry after 2026-08-30: expiry is the likely outcome, but
   adoption would be material.
3. **Republication of the canonical ARM RPC repo** would let the `Location` MUST
   row be verified and would strengthen — though not by itself carry — the case
   for a stronger keyword.
4. **A project decision that a `url` member is a hypermedia control** barred by
   §1.2 would not change the verdict, but would force the body clause to the
   `id` + documented-template form and would make the `Location` SHOULD more
   load-bearing (since a bare `id` cannot address an off-origin monitor).

**Non-invalidating:** further vendor examples either way. The survey is already
lopsided toward body-carried identity, and the recommendation does not rest on
the count — it rests on RFC 9110 §15.3.3 for the body MUST and on the
zero-knowledge-client argument for the `Location` SHOULD.
