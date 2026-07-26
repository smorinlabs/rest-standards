Test the load-bearing inference behind principle `AC-003`, which recommends
mandating RFC 9457 `application/problem+json` despite only one of the eight
surveyed reference APIs doing so.

`baseline-02` §8.1 argues that the eight vendors diverge from RFC 9457 for
**historical** reasons — they shipped error formats before RFC 7807 existed, or
inherited a shape from an adjacent ecosystem — and asserts that "none of the
eight is a greenfield API that evaluated RFC 9457 and rejected it." That
assertion is an inference about other parties' motives. It is testable, and a
Gate C decision depends on it.

Exact scope: whether newly built APIs and the frameworks they are built on
adopt RFC 9457; and whether any greenfield API can be shown to have evaluated
and rejected it.

Research requirements:

1. Treat **server framework defaults** as the strongest available proxy for
   greenfield behavior: what a new API emits without its author making a
   deliberate choice is better evidence of the default path than any single
   vendor's decision.
2. For each framework, establish from the vendor's own documentation whether
   Problem Details is emitted **by default**, is **opt-in**, or is absent —
   and which RFC the vendor cites.
3. Look specifically for a vendor whose framework default diverges from its own
   flagship API's error shape. Such a case is strong evidence for the
   historical-inertia explanation and against a merits-based rejection.
4. Verify every material claim against at least two independent sources,
   preferring vendor-primary documentation over commentary.
5. Actively seek disconfirming evidence: any documented case of a new API
   evaluating RFC 9457 and choosing a proprietary shape on the merits. Report
   its absence explicitly if none is found, and state what that absence does
   and does not prove.
6. Note any inconsistency in the vendors' own RFC citation hygiene (7807 versus
   9457), which bears on `AC-005`.

Output: a framework-default matrix; an explicit verdict on whether the
`AC-003` inference holds, weakens, or fails; and a recommended confidence
level for `AC-003` going into Gate C.
