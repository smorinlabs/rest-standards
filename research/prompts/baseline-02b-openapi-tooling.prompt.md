Determine whether OpenAPI 3.2.0 tooling support is mature enough to justify
principle `AC-001`'s mandate that the contract be published as an OpenAPI
3.2.0 document, and confirm the current status of the JSON Schema dialect that
principle pins.

Exact scope: per-tool OpenAPI 3.2.0 support across parsers, validators,
linters, documentation renderers, and code generators; the authoritative
OpenAPI 3.2.0 release date; and the standardization status of JSON Schema
2020-12 including any active IETF work.

Do not re-decide which OpenAPI version is technically preferable; 3.2 is
strictly incremental over 3.1. This leaf answers an ecosystem-readiness
question only.

Research requirements:

1. `AC-001` currently rests on a secondary claim that "most major tools
   released 3.2 support in Q4 2025 or Q1 2026." **Test that claim directly
   against per-tool primary sources.** Do not repeat it as corroboration.
2. For each tool, consult the project's own issue tracker, release notes, or
   documentation. Record the issue number, its state, whether a maintainer has
   responded, and whether any milestone or pull request exists.
3. Distinguish three states: full support, partial or CLI-only support, and no
   support. Treat a tool that **silently ignores** 3.2 constructs as worse
   than one that rejects them outright, and say why.
4. Establish the OpenAPI 3.2.0 release date from at least two independent
   authoritative sources and reconcile any discrepancy rather than choosing
   one.
5. Confirm whether JSON Schema 2020-12 remains current and whether any IETF
   working-group draft now exists, including its intended RFC status.

Output: a per-tool support matrix with issue references and dates; a verdict
on whether `AC-001` should change; and any correction to the source-and-
currency matrix in `baseline-02`.
