# Baseline 02h — PATCH Document Format

*Gate C addendum leaf under `baseline-02`, closing the dropped handoff the
CLI-standards gap review surfaced: `baseline-01` §9 handed "patch document
format" to `baseline-02`, which decided only the null-vs-omission half
(`AC-011`). Run 2026-08-09; all RFC text read from `rfc-editor.org` raw
`.txt`; Azure and Kubernetes quotes re-verified against raw Markdown bytes.*

**Evidence classes:** `[FACT]` primary spec · `[COMPARATIVE]` vendor
practice · `[INFERENCE]` · `[RECOMMENDATION]`.

## Q1 — Format semantics from raw RFC text

### RFC 5789 (PATCH method) — Standards Track, March 2010

The governing constraint, verbatim (§2):

> "There is no guarantee that a resource can be modified with PATCH.
> Further, it is expected that different patch document formats will be
> appropriate for different types of resources and that no single format
> will be appropriate for all types of resources. Therefore, **there is no
> single default patch document format that implementations are required to
> support.** Servers MUST ensure that a received patch document is
> appropriate for the type of resource identified by the Request-URI."

`[FACT]` RFC 5789 mandates **no** format. It defines the negotiation
surface instead: the patch document is "identified by a media type";
discovery is `Accept-Patch` ("SHOULD appear in the OPTIONS response for any
resource that supports the use of the PATCH method", §3.1), and a 415
response "SHOULD include an Accept-Patch response header … to notify the
client what patch document media types are supported" (§2.2).

Two requirements the standard inherits:

- **Atomicity (§2):** "The server MUST apply the entire set of changes
  atomically and never provide … a partially modified representation. If
  the entire patch document cannot be successfully applied, then the
  server MUST NOT apply any of the changes."
- **Concurrency (§2):** "Collisions from multiple PATCH requests may be
  more dangerous than PUT collisions because some patch formats need to
  operate from a known base-point or else they will corrupt the resource.
  Clients using this kind of patch application SHOULD use a conditional
  request … a strong ETag … in an If-Match header." PATCH "is neither safe
  nor idempotent."

### RFC 7396 (JSON Merge Patch) — Standards Track, October 2014, `application/merge-patch+json`

Header states `Obsoletes: 7386`. Null-means-remove, from the normative
pseudocode (§2):

```
define MergePatch(Target, Patch):
  if Patch is an Object:
    if Target is not an Object:
      Target = {} # Ignore the contents and set it to an empty Object
    for each Name/Value pair in Patch:
      if Value is null:
        if Name exists in Target:
          remove the Name/Value pair from Target
      else:
        Target[Name] = MergePatch(Target[Name], Value)
    return Target
  else:
    return Patch
```

The RFC names its own limitation (§1): merge patch documents are "suitable
for describing modifications to JSON documents that primarily use objects
for their structure **and do not make use of explicit null values**."

Arrays (§2): "it is not possible to patch part of a target that is not an
object, such as to replace just some of the values in an array."

`[FACT]` Precision point from Appendix A: the test case `{"e":null}` +
`{"a":1}` → `{"e":null,"a":1}` shows a merge-patch **target may hold** a
null value — merge patch simply cannot **set** one. The limitation is on
the write path only.

### RFC 6902 (JSON Patch) — Standards Track, April 2013, `application/json-patch+json`

Op/path/value model (§4): "Operation objects MUST have exactly one 'op'
member … one of 'add', 'remove', 'replace', 'move', 'copy', or 'test'" and
"exactly one 'path' member … a JSON-Pointer value [RFC6901]". Sequential
application (§3). The `test` op's typed equality includes null, so JSON
Patch **can** both set and test an explicit null. Atomicity (§5): a failed
operation means "application of the entire patch document SHALL NOT be
deemed successful"; "Note that the HTTP PATCH method is atomic, as per
[RFC5789]." JSON Pointer escaping (RFC 6901 §3–4): `~`→`~0`, `/`→`~1`,
decoded `~1`-first to avoid the `~01` corruption case.

`[COMPARATIVE]` Cost on the same edit: removing `familyName` is
`{"author":{"familyName":null}}` (merge) vs
`[{"op":"remove","path":"/author/familyName"}]` (JSON Patch) — comparable.
But merge patch replaces `"tags"` wholesale where JSON Patch can address
`/tags/1`.

## Q2 — What the field ships

| Vendor | PATCH? | Format | `Content-Type` accepted | Null handling |
|---|---|---|---|---|
| **Azure REST Guidelines** | Yes, mandated | RFC 7396 Merge Patch | `application/merge-patch+json` | null = delete field; **null ≡ absent in representations**; 400 if undeletable |
| **Microsoft Graph** | Yes | Proprietary merge-like | **`application/json`** | Per-field nullability documented; explicit deviations (e.g. `employeeOrgData`) |
| **Google AIP-134** | Yes | `update_mask` field mask (third way) | Not specified (plain JSON in REST mapping) | Mask sidesteps it — presence in mask, not body, is authoritative |
| **Kubernetes** | Yes | **Four**, negotiated by Content-Type | `json-patch+json`, `merge-patch+json`, `strategic-merge-patch+json`, `apply-patch+yaml` | Per-format; JSON Patch covers what merge cannot express |
| **GitHub REST** | Yes | Plain partial JSON | `application/json` | No general rule documented |
| **Stripe** | **No** — POST updates | Form-encoded partial body | `application/x-www-form-urlencoded` | **Empty-string sentinel**: "Individual keys can be unset by posting an empty value" |
| **Shopify Admin REST** | No — PUT | Plain JSON | `application/json` | Not documented |
| **Anthropic** | **No PATCH anywhere** | POST-for-update, plain JSON | `application/json` | n/a — see source-conflict note |
| **OpenAI** | **Not verified** (403 + search budget exhausted) | — | — | — |
| **Gemini** | **Not verified** (404s; AIP-134 makes PATCH+`updateMask` plausible) | — | — | — |

Key verbatim evidence:

- **Azure** (raw `Guidelines.md`): "**DO** create and update resources
  using PATCH [RFC 5789] with JSON Merge Patch [(RFC 7396)] request body."
  · "**DO** accept JSON fields with a null value **only for a PATCH
  operation with a JSON Merge Patch payload**. A field with a value of
  null instructs the service to delete the field. If the field cannot be
  deleted, then return 400-BadRequest…" · "**DO NOT** send JSON fields with
  a null value from the service to the client… **Semantically, Azure
  services treat a missing field and a field with a value of null as
  identical.**" · "Avoid arrays … especially with JSON Merge Patch where
  the entire array needs to be read prior to any operation being applied
  to it." · "**DO NOT** implement PATCH as an LRO."
- **Microsoft Graph**: request header table shows plain
  `Content-Type: application/json`; "supply *only* the values for
  properties to update"; nullability documented per field ("Can't be
  updated to `null`", "Allowed values: `null`, `Minor`, …"). The
  interesting divergence: same company, different surface, no merge-patch
  media type.
- **Kubernetes**: the only surveyed implementation supporting both RFCs
  simultaneously, negotiated by `Content-Type`; JSON Patch documented for
  conditional consistency ("allowing the operation to fail … to avoid a
  lost update"). Staleness note `[FACT]`: its docs cite **RFC 7386**,
  which RFC 7396 obsoletes.
- **Stripe**: POST with "Any parameters not provided are left unchanged";
  unset via **empty-string sentinel**, never null-as-verb.
- **⚠ Source conflict, surfaced per repo rules:** a cached `claude-api`
  skill endpoint table lists `PATCH` for Anthropic's UpdateMemory; the
  **live** docs (platform.claude.com, accessed 2026-08-09, newer beta
  header) show `POST`. **The live source governs.** Recorded as a caution
  about cached endpoint tables.

## Q3 — The AC-011 collision, precisely

**The collision is narrower than "Merge Patch breaks AC-011."** `[FACT]`
Merge Patch is the only standardized JSON patch format that *implements*
AC-011's required distinction at the wire level: omission means leave
alone, presence means change. The actual collision: Merge Patch overloads
`null` as the delete verb, so it **cannot express "set this field to
null" as distinct from "remove this key."** It bites only where stored
null is semantically different from absent.

Resolutions observed in the field:

| Resolution | Who | Mechanism |
|---|---|---|
| **Mandate null ≡ absent in representations** | Zalando rule 123; Azure | Dissolves the collision by rule |
| **Reject undeletable deletes** | Azure | `null` on non-deletable field → 400 |
| **Field mask** | Google AIP-134 | Mask presence, not body presence, decides |
| **Empty-string sentinel** | Stripe | `""` means unset |
| **Per-field nullability contract** | Microsoft Graph | Documented field by field |
| **JSON Patch fallback via Content-Type** | Kubernetes | For explicit-null / array-element / conditional edits |

`[INFERENCE]` Merge Patch + AC-011 forces one of two resolutions:
(1) constrain the resource model — null ≡ absent everywhere (Zalando/Azure;
cheap, but forbids tri-state fields); or (2) provide the JSON Patch escape
hatch (Kubernetes). Adopting Merge Patch **without** picking one leaves
AC-011 unsatisfiable on any nullable-meaningful field — silent data loss,
the exact failure AC-011 exists to prevent.

## Q4 — Tooling reality

`[FACT]` OpenAPI 3.1 media-type keying is native — an operation can declare
`application/merge-patch+json` and `application/json-patch+json` as sibling
keys under one `requestBody`; dual-format is directly expressible.

`[COMPARATIVE]` The all-fields-optional problem is policy at Azure, not
solved: "The PATCH request schema should contain all the same fields with
no required fields." `[INFERENCE]` A merge-patch body cannot validate
against the resource schema (every `required` stripped), so a second,
hand-maintained schema per resource results; JSON Patch has the mirror
problem (trivially valid ops array, resource-level validation only after
application).

`[COMPARATIVE, low confidence — not decision-relevant]` Both formats have
mature multi-language libraries; this axis does not discriminate.

## Q5 — Recommendation

**Recommended default: (a) Merge Patch with a mandatory companion null
rule, plus (c) as a bounded MAY.**

`[RECOMMENDATION]` Adopt RFC 7396 with `application/merge-patch+json` as
the required default; pair it with a Zalando/Azure-style null-equivalence
rule in representations (what makes it AC-011-safe); permit RFC 6902 at
`application/json-patch+json`, negotiated by `Content-Type`, for resources
merge patch demonstrably cannot serve.

- RFC 5789's "no single default format" makes `Content-Type` the
  sanctioned negotiation surface — dual support is standards-conformant.
- Merge Patch is the only standardized format implementing AC-011's
  omission-vs-presence distinction; plain JSON gets it only by convention.
- The companion rule is load-bearing, not garnish — both mandating vendors
  ship it.
- JSON Patch as MAY covers explicit-null, array-element, and
  `test`-conditioned updates without imposing its verbosity everywhere.

### Proposed rule wording

> **PATCH request bodies MUST be JSON Merge Patch (RFC 7396) sent with
> `Content-Type: application/merge-patch+json`; servers MUST reject other
> media types with `415 Unsupported Media Type` and advertise supported
> formats in `Accept-Patch`.** Because Merge Patch overloads `null` to mean
> "remove this member," resource representations MUST give `null` and an
> absent property the same meaning — Merge Patch delete semantics are the
> sole exception to this equivalence, and a `null` targeting a field that
> cannot be removed MUST return `400`. An API whose resources genuinely
> require a value-null distinct from absent, per-element array edits, or
> test-conditioned updates MAY additionally accept JSON Patch (RFC 6902)
> at `application/json-patch+json` on the same resource, and MUST document
> which format applies where.

### Confidence

**Moderate-high** on the format; **high** on the companion null rule
(load-bearing regardless of which format wins).

**A genuine fork.** Plain-JSON partial bodies are the field plurality
(GitHub, Graph, Shopify, Stripe-via-POST); only Azure mandates the media
type. Graph is the strongest counter-case — AC-011-compliant with plain
JSON plus per-field nullability docs, from the same vendor whose Azure
guidelines mandate merge patch. A reviewer weighting ecosystem
compatibility could reasonably choose plain JSON. The recommendation
matches the project's demonstrated posture: `AC-003` was ratified at
1-of-8 adoption; Merge Patch has strictly better adoption than that.

### What flips it

1. **Hard flip — resource model:** if stored-null must be distinct from
   absent, the companion rule is unadoptable → field masks (AIP-134) or
   JSON Patch as MUST.
2. **Soft flip — weighting:** ecosystem-over-standards inverts to plain
   JSON with per-field null docs (the Graph model).
3. **Scope flip:** if array-element mutation is common rather than
   exceptional, the JSON Patch MAY becomes MUST (Azure concedes the array
   weakness in its own guidance).

### Open items

- OpenAI PATCH surface unverified (403; search budget exhausted).
- Gemini PATCH surface unverified (404s; AIP-134 makes PATCH+`updateMask`
  likely — would add a second field-mask datapoint).
- The Anthropic cached-table vs live-docs method conflict, resolved for
  the live docs.

## Sources

RFC 5789 · RFC 7396 · RFC 6902 · RFC 6901 (rfc-editor.org raw text) ·
Azure API Guidelines (raw `Guidelines.md`) · Microsoft Graph user-update ·
Kubernetes `api-concepts.md` (raw) · Google AIP-134 · GitHub REST repos ·
Stripe update-customer · Shopify Admin REST · Anthropic managed-agents
memory (live) · OpenAPI 3.1.1 (raw) · Zalando RESTful API Guidelines.
