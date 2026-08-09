Execute the Gate C drafting consequence (b) recorded in `baseline-02d`:
"rule explicitly on `type`". The `type` member of RFC 9457 is known to be
overloaded — a stable identifier that consumers MUST dispatch on, and a
SHOULD-be-resolvable documentation locator — and every mandating adopter
examined so far has had to solve that tension, two by deviating from the
RFC. Before `AC-003` is ratified, establish what a greenfield standard
should mandate for `type`.

Questions:

1. The RFC text itself: what §3.1.1 and §4 exactly require, recommend, and
   permit — absolute vs relative, `about:blank` semantics, dereferenceability,
   stability, `tag:`/`urn:` schemes, and what the registry section asks of
   API authors.
2. The IETF debate: the httpapi WG mailing-list and `rfc7807bis` tracker
   history of proposals to split identifier from documentation link — who
   raised it, what alternatives were considered, why it was rejected.
3. Field practice: how real adopters populate `type` (Zalando, Belgif,
   ACME/Let's Encrypt, Cloudflare, ASP.NET Core defaults, Spring, others) —
   URL vs URN, resolvable or not, stable or not, documented policy or
   accident.
4. Failure modes: type-URL rot, environment leakage, client coupling,
   versioning of problem types — documented cases preferred.
5. Candidate policies for a greenfield standard, evidence for and against
   each: (i) resolvable https URLs under a controlled domain with a
   stability promise; (ii) non-resolvable stable URIs; (iii) a URN scheme
   formally; (iv) `about:blank` plus a `code` extension member; (v) a split:
   stable `type` plus a separate documentation member. Note interactions
   with `AC-004`, which already requires a stable machine-readable `code`
   extension on every problem document.

Evidence rules as in `CONSTRAINTS.md`: two or more sources per material
claim, primary preferred (RFC text, IETF mail archive, project repos);
label claims; date findings; quote load-bearing text verbatim; report
absences explicitly.

Output: findings per question; a comparison table of how each surveyed
adopter populates `type`; a recommendation naming the candidate policy the
evidence best supports, its exact interaction with the required `code`
member, and proposed normative wording.
