# Scoping — tier, switches, and evidence plane

Three dials, settled before plan, review, or audit. Two belong to the
standard and are read live; one belongs to this skill.

## 1. Tier (§1.7)

Read §1.7 for the current tier vocabulary — do not assume it. One
AskUserQuestion, recommendation first, derived from context: a public
developer portal or self-service signup ⇒ `public`; a partner integration
guide or contracted consumers ⇒ `partner`; consumed only by the providing
org's own services ⇒ `internal`.

Tier is an audience declaration. It does not reduce the rule set, and it never
waives a MUST. Honor notes ("internal, but it opens to partners next quarter"
⇒ scope at `internal`, flag the `partner` delta).

## 2. Switches (§1.8)

Read the live switch vocabulary from §1.8. Infer each switch's state from the
API's shape and confirm the inference inside the tier question's option
descriptions rather than asking one question per switch.

Every switch declared off MUST carry a one-line reason (R1.6). Record as
`N/A — <switch>: off, <reason>`. A switch wrongly marked off is itself a
finding.

Capability facts with no rule attached — tenancy model, PII handling, client
audience — go in the conformance note's free-text `Context` field, not here.

## 3. Evidence plane

Skip in plan mode — greenfield has nothing to survey.

Ask what exists. This is a factual question, not a judgment:

| Plane | Present when | Unlocks |
|---|---|---|
| contract | A documented interface contract exists — an OpenAPI or JSON Schema document, or, failing that, reference documentation or worked request/response exchanges | `conformance/spectral.yaml` on a machine-readable document; otherwise read the contract directly |
| source | The implementation is readable | Rules with no contract expression |
| runtime | A deployment exists **and** the user opts in (`references/audit.md` § gate) | Appendix G probes |

Planes overlap; more than one may be present. Runtime is never assumed
available — it requires the opt-in ladder, without exception.

## 4. Record it

Emit the scope record at the top of every deliverable, then proceed:

    Standard: rest-api-standard v<version>
    Tier: <internal|partner|public>
    Switches: <name>=<on|off> …  (each off switch carries a one-line reason)
    Evidence planes available: <contract|source|runtime> …
