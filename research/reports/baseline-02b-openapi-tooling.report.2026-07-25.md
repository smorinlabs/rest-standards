# Baseline 02b — OpenAPI 3.2 Tooling Maturity

*Narrow leaf under `baseline-02`. All sources verified 2026-07-25. Answers
whether `AC-001` can stand as written.*

## Verdict

**`AC-001` must change.** The secondary claim it rested on — *"most major tools
released 3.2 support in Q4 2025 or Q1 2026"* — **is false when checked against
per-tool primary sources.** Four of the most widely deployed OpenAPI tools do
not support 3.2 today, ten months after release.

Recommended change: mandate an OpenAPI document as the contract source of
truth; **accept 3.1 or 3.2**, and do not require 3.2 until parser and generator
support lands.

## Per-tool support matrix

| Tool | Category | 3.2 support | Evidence |
| --- | --- | --- | --- |
| **Redocly CLI** | linter / docs | ✅ **Full** | Repo states support for "OpenAPI 3.2, 3.1, 3.0 and OpenAPI 2.0" |
| **swagger-parser** | parser / validator | ❌ **None** | Issue #2248 open; no maintainer response, no PR, no milestone |
| **Redoc** | docs renderer | ❌ **None** | Issue #2773 open; rejects documents with `Unsupported OpenAPI version: 3.2.0` |
| **openapi-generator** | code generator | ❌ **None** | Issue #22728 opened 2026-01-19, still open; no maintainer comment, no milestone, no PR |
| **Spectral** | linter | ⚠️ **Silently degraded** | Issue #2910 opened 2026-03-12. Does not fail on a 3.2 document but **silently ignores 3.2-specific constructs** |

### Why Spectral is the worst case

`[INFERENCE]` A tool that rejects 3.2 outright (Redoc) tells you it cannot help.
Spectral accepts the document and lints it **without warning that it is
skipping 3.2 constructs** — producing a green result that means less than it
appears to. For a standard whose whole premise is that machine-checkable
conventions do not drift (`baseline-02` §3), a linter giving false assurance is
worse than one that refuses the file.

Compounding this: reporting indicates Spectral has received minimal investment
since SmartBear's acquisition of Stoplight and is largely dormant. `[FACT]` The
3.2 issue has been open since 2026-03-12 with no resolution.

## Release date — discrepancy resolved

Two sources initially disagreed on the OpenAPI 3.2.0 release date. Resolved
against the GitHub API, which returns structured timestamps with no parsing
ambiguity:

```
3.2.0  published: 2025-09-19T16:20:24Z
3.1.2  published: 2025-09-19T15:45:02Z
3.1.1  published: 2024-10-24T17:37:48Z
```

**`[FACT]` OpenAPI 3.2.0 was released 2025-09-19**, confirming
`spec.openapis.org` and `baseline-02`'s original claim. The conflicting
"September 2024" reading was a page-parsing artifact, not a real disagreement.
3.2.0 remains the newest release; there is no 3.2.1 or 4.0.

## JSON Schema — a correction to `baseline-02`

`[FACT]` **JSON Schema is now on the IETF standards track.**
`draft-ietf-jsonschema-json-schema-02`, dated **2026-07-01**, is an **active**
Internet-Draft expiring 2027-01-02, owned by the `jsonschema` working group,
with **intended RFC status: Proposed Standard**. It corresponds to the
**2020-12** dialect.

`baseline-02`'s source matrix states JSON Schema is "**not an RFC** — no IETF
standards-track status." The first half remains true; the second is now wrong.
A working-group-adopted draft targeting Proposed Standard is standards-track
work, and it targets precisely the dialect `AC-001` pins.

**Net effect: the dialect choice is strengthened; the version mandate is
weakened.** These move in opposite directions and should not be conflated.

## Recommended changes

| Principle | Change | Reason |
| --- | --- | --- |
| `AC-001` | Accept **3.1 or 3.2**; drop the 3.2 mandate | Parser and generator support absent |
| `AC-001` | Keep the JSON Schema 2020-12 pin; **raise** its confidence | Now WG-adopted, targeting Proposed Standard |
| `baseline-02` matrix | Correct the JSON Schema authority class | No longer accurate |
| `AC-002` | Note that automated compatibility checking depends on parsers that do not yet handle 3.2 | The gate is unenforceable on 3.2 today |

**Re-check trigger:** swagger-parser #2248 or openapi-generator #22728 closing.
Either would materially change this verdict.

## Sources

- https://github.com/OAI/OpenAPI-Specification/releases (via GitHub API)
- https://spec.openapis.org/oas/latest.html
- https://github.com/swagger-api/swagger-parser/issues/2248
- https://github.com/Redocly/redoc/issues/2773
- https://github.com/OpenAPITools/openapi-generator/issues/22728
- https://github.com/stoplightio/spectral/issues/2910
- https://github.com/Redocly/redocly-cli
- https://datatracker.ietf.org/doc/draft-ietf-jsonschema-json-schema/
