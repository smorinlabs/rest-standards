# Research Discipline

The inventory of what exists lives in [`README.md`](README.md). Read it before
starting new research. This file holds the rules; it deliberately does not
duplicate the table.

## Before running anything

1. Check `README.md` for an existing prompt covering the question. Reuse it.
2. Check `reports/` for a run of that prompt. A stem present in `prompts/` with
   no match in `reports/` is unrun; a stem with a run is answered.
3. Do not rerun a broad prompt to produce another synthesis. Rerun only to
   obtain independent evidence on a question where the existing run is thin,
   stale, or internally uncertain — and keep both runs.
4. Add a narrow follow-up leaf only after an existing report exposes an
   unresolved decision that needs new evidence. Give it the parent's stem with
   a letter suffix (`survey-05b-...`), so it sorts under its parent.

## Naming

Every file follows `<series>-<seq>-<topic>.<role>[.<date>[<letter>]].md`, fully
specified in [`README.md`](README.md). Two rules matter most:

- **Rename a report the moment it arrives.** Research tools emit opaque names.
  A report whose filename does not state its prompt is unpairable, and
  recovering the mapping means reading every file.
- **The topic slug never changes.** It is the pairing key across `prompts/`,
  `reports/`, and `decisions/`. Renaming it silently breaks every glob.

## Series boundaries

`survey` is descriptive and `baseline` is prescriptive. Keep the modes apart:

- A `survey` report that recommends a rule has exceeded its mandate.
- A `baseline` report proposes rules. Proposals are not decisions. Nothing in
  `reports/` is binding on the standard; only a ratified record in
  `decisions/` is.

## Evidence rules

Full detail is in [`CONSTRAINTS.md`](CONSTRAINTS.md). The load-bearing ones:

- Every material claim carries a direct URL, the source's authority class, and
  its publication or access date.
- Never present an Internet-Draft or a vendor header as a published standard.
  Record draft numbers and expiry.
- Surface source conflicts rather than averaging them. State which source
  should govern and why.
- Label sourced facts, inferences, recommendations, and project policy
  distinctly.
