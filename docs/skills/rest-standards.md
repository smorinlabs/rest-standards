# rest-standards

An agent skill that applies this repo's [REST API Design Standard](../../rest-api-standard.md)
to any HTTP API work, at the depth the available evidence supports. Four modes:
**plan** (greenfield interview → an `openapi.yaml` skeleton and a seeded
`CONFORMANCE.md`), **check** (mid-build lookups answered with rule IDs, nothing
written), **review** (design review of a spec, OpenAPI document, or unshipped
diff), and **audit** (conformance sweep of an existing API across three evidence
planes, ending in a `<N> applicable MUSTs: <P> pass, <F> fail, <U> unverified`
summary). Depth scales on three dials: **tier** (§1.7 — `internal` / `partner` /
`public`, an audience declaration that never waives a MUST), **applicability
switches** (§1.8 — read live; every switch declared off carries a one-line
reason per R1.6), and the skill's own **evidence plane** (contract / source /
runtime), which decides which rules can be verified at all. A rule no available
plane reaches is reported `unverified`, never an inferred pass. Deviations are
never silent (R1.7): waived SHOULDs land in a conformance note rendered from the
§1.9 template, and a deviation that beats the rule becomes a proposed Part II
amendment to the standard itself. The skill reads `rest-api-standard.md` live
from this repo — no bundled copy, no drift — and pins the version it read into
every deliverable.

**Triggers on:** "new API", "design this endpoint", "review this OpenAPI spec",
"is this API conformant", "audit this API", "REST standards", "what does the
standard say about \<status codes / headers / pagination\>"
**Arguments:** none

## The runtime plane is gated

Appendix G probes hit a real deployment, and some of them are destructive
exactly when the API fails the check — an unguarded `DELETE` succeeds, an
"unimplemented" method turns out to be implemented, a `dry_run` parameter
executes for real. Those are examples, not the list: the skill reads Appendix G
live and classifies every row before running any of it, because an amendment
can add probes. So: the default is **no HTTP requests at all**; the user must
ask, then supply a base URL, an explicit statement that the deployment is
non-production, and which resources are disposable; read-only probes — those
that neither change state nor degrade service for other clients **and** return
on their own in bounded time — run first; mutating **and
disruptive** probes need a second confirmation naming the fixture;
**unbounded** probes — a stream consumed, a long poll held, a resumption — need
that confirmation plus an agreed wall-clock and frame/byte bound, and their
handed-off `curl` enforces both inline (`--max-time` and `--max-filesize`,
plus `-N`); nothing runs
against production or against an API the user does not own. Anything not run is
reported unverified with the exact `curl` for the user to run by hand. Reviewing
a third party's published contract needs no gate — that is the contract plane.

The contract plane runs [`conformance/spectral.yaml`](../../conformance/spectral.yaml)
when a machine-readable document exists. When one does not, reference docs and
worked request/response exchanges are still contract evidence: the skill reads
them directly and says so, rather than downgrading to source-only.

## Install

**In this repo — nothing to install.** Claude Code auto-discovers
`.claude/skills/rest-standards/`.

**Dev mode** (edits in the clone are live next session, and the skill keeps
reading the canonical standard):

    git clone https://github.com/smorinlabs/rest-standards
    cd rest-standards
    ln -s "$(pwd)/.claude/skills/rest-standards" ~/.claude/skills/rest-standards   # Claude Code
    ln -s "$(pwd)/.claude/skills/rest-standards" ~/.agents/skills/rest-standards   # Codex

The `cd` is load-bearing: `$(pwd)` is expanded from wherever you are, so
running the `ln -s` lines from the parent directory instead produces a link
into a path that does not exist.

Prefer the symbolic link over a copy. The skill locates the standard by
resolving its own real path and walking up three directories, so a copy placed
outside the repo cannot find `rest-api-standard.md` and the skill will stop
rather than answer from memory.

## Example session

> "We're standing up a new payments API — help me design it."
> → plan mode: interviews for resources, tier, switches, and versioning shape,
> then writes an `openapi.yaml` skeleton and a seeded `CONFORMANCE.md` into the
> new API's repo, both pinned to the standard version used, and lints the
> skeleton with the repo's Spectral ruleset before handing it over.

> "Is the orders API conformant?"
> → audit mode: settles tier, switches, and which planes exist, sweeps the
> contract and source planes, offers the gated runtime ladder for the rules
> neither plane can reach, and reports a findings table (blockers first) with
> the N/A list, the unverified rules and their missing planes, and the
> pass/fail/unverified count.
