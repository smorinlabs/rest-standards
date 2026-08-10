# Changelog

Versioning follows the Part II amendment rule of
[`rest-api-standard.md`](rest-api-standard.md): editorial changes bump
patch; added rules, appendices, or relaxations bump minor; strengthened,
removed, or re-meant rules bump major. Every rule change is atomic across
the rule text, its decision record, its Part II row, its checklist row,
and the worked example.

## 1.0.0 — 2026-08-10

First release. 127 rules across twelve normative sections, each carrying
provenance (decision-layer key), classification, and confidence; Part II
Decision Log (provenance map + apparatus register); Appendices A–G;
executable conformance fixtures (`conformance/spectral.yaml`,
execution-verified against `conformance/fixture-violations.yaml`).

Produced by the phased process in [`PLAN.md`](PLAN.md):

- **Gate C (2026-08-09):** all 66 researched principles plus the
  structural locks and five addenda ratified into
  [`research/decisions/`](research/decisions/) (PRs #3, #4).
- **Gate D (2026-08-09):** draft approved for systematic review (PR #5);
  SDK-configuration conventions excluded, SSE/streaming deferred to a
  post-1.0 phase.
- **Phase 4 + Gate E (2026-08-10):** three review waves (internal
  three-agent, Codex second lens, raw-RFC classification sweep), four
  owner rulings (switch pruning, `cancel` scope, R4.16 path
  placeholders, R10.9 operation discovery via research leaf
  `baseline-02i`), and the Gate E CORS ruling (R4.17); approved as the
  1.0 candidate and landed via PR #6.
- **Phase 5 release (2026-08-10):** version flipped to 1.0.0, this
  changelog created, and `v1.0.0` tagged on the merge commit of release
  PR #7.
