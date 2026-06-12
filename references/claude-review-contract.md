# Claude Review Contract

Ask Claude to answer with one of these first-line status tokens. The first non-empty line must be exactly one token, with no markdown formatting, heading marker, prefix, or suffix:

Default review priorities:

1. Architecture design correctness and architecture option selection.
2. Execution reliability and verification sufficiency.

Before deep architecture review, Claude should classify the submitted plan or target file changes as `trivial`, `small`, `medium`, or `large` based on actual scope and risk, not line count.

Classification guide:

- `trivial`: localized text, comments, documentation wording, typo fixes, formatting-only changes, or one-line behavior-preserving edits with no interface or data-flow impact.
- `small`: localized implementation change inside one existing module or file, following established patterns, with no public API, dependency, state ownership, cross-module, or compatibility impact.
- `medium`: changes that touch multiple files or modules, modify internal interfaces, change non-trivial behavior, affect validation or testing strategy, or introduce meaningful maintenance tradeoffs.
- `large`: changes that affect public APIs, dependency direction, shared state ownership, persistence or wire formats, cross-module flow, major abstractions, rollout compatibility, or broad architectural direction.

For `trivial` or `small` changes, Claude should perform a lightweight architecture check only:

1. Confirm the change belongs in the touched module or file.
2. Confirm it follows existing local patterns.
3. Confirm it does not alter public APIs, dependency direction, state ownership, cross-module data flow, compatibility, or test isolation.
4. Do not require architecture alternatives unless one of those boundaries is touched.

For `medium` or `large` changes, Claude should review architecture in this order:

1. Scope and existing constraints: confirm the current task boundary, the architectural constraints already present in the workspace, and any compatibility limits that must not be broken.
2. Module responsibilities and boundaries: judge whether the work belongs in the proposed modules, keeps ownership clear, and preserves clean boundaries.
3. Data flow and state ownership: check where data comes from, where it goes, who owns state, and whether transformations and control flow stay understandable.
4. Interfaces, dependencies, and compatibility: check public API changes, dependency direction, coupling, compatibility risk, and test isolation risk.
5. Abstraction fit: judge whether the abstraction level matches the existing system style and avoids premature abstraction, duplicate abstraction, and public interface pollution.
6. Maintainability and extensibility consequences: assess whether the design creates avoidable long-term maintenance or extension risk.
7. In-scope architecture alternatives: actively check whether there is an architecture that still serves the current user request, remains inside the current task scope, and is better overall on complexity, consistency, maintenance cost, compatibility risk, verification difficulty, public API exposure, and dependency expansion. If a clearly better architecture exists and its benefit is enough to justify the change cost, Claude should require the plan or file changes to adopt it instead of approving a merely executable approach.

If a `trivial` or `small` change touches or risks touching module boundaries, public interfaces, dependencies, state ownership, data flow, compatibility, or test isolation, Claude should escalate to the full `medium` or `large` architecture review. Do not escalate a `trivial` or `small` change into full architecture review solely to find optional improvements.

Only after the appropriate lightweight or full architecture review is acceptable should Claude review execution steps, implementation detail, validation coverage, and rollout risk. Claude must not propose broad refactors unrelated to the current task. Architecture feedback must be concrete enough for the consensus subagent to convert into the current plan or target file changes.

After the status token, Claude must immediately output this minimal auditable classification block:

```text
Risk classification: <trivial|small|medium|large>
Classification reason: <one concise sentence>
Architecture review mode: <lightweight|full>
```

The `Classification reason` should briefly justify the chosen risk level and review mode. For `full` review, the `Classification reason` or the next brief sentence should explicitly name the main boundary risk source such as API, dependency, state ownership, data flow, compatibility, or test isolation. For `lightweight` review, the `Classification reason` should say the change is localized and does not touch architecture boundaries.

```text
APPROVED
Risk classification: <trivial|small|medium|large>
Classification reason: <one concise sentence>
Architecture review mode: <lightweight|full>
<brief rationale, optional>
```

Use `APPROVED` only when a `trivial` or `small` change passes the lightweight architecture check and important execution and verification risks are covered, or when a `medium` or `large` change has a sound architecture direction, no clearly better in-scope architecture alternative should be adopted, and the important execution and verification risks are covered. Keep `APPROVED` concise; for `trivial` or `small` changes, do not expand into long architecture analysis beyond the minimal auditable block and a brief rationale unless the change required `full` review.

```text
APPROVED_WITH_NOTES
Risk classification: <trivial|small|medium|large>
Classification reason: <one concise sentence>
Architecture review mode: <lightweight|full>
- <low-risk note or caveat that should be incorporated or explicitly deferred>
```

Use `APPROVED_WITH_NOTES` only for low-risk architecture caveats, architecture improvements that can be explicitly deferred, or execution-level cleanup that the consensus subagent should incorporate into the plan or target files when appropriate, or explicitly defer before another review round. Do not use `APPROVED_WITH_NOTES` for purely optional suggestions that require no follow-up. This is not a final approval state. For `plan`, the subagent sends the complete updated plan in the next round.

```text
REVISE
Risk classification: <trivial|small|medium|large>
Classification reason: <one concise sentence>
Architecture review mode: <lightweight|full>
- <required plan change, incorrect assumption, missing inspection, target file issue, or verification gap>
```

Use `REVISE` when a `trivial` or `small` change is in the wrong module, violates existing local patterns, actually touches architecture boundaries without accounting for them, or has an execution/verification issue that the consensus subagent can fix without asking the user. For `medium` or `large` changes, use `REVISE` when there is an architecture design problem, a clearly better in-scope architecture alternative that should be adopted, or an execution/verification issue that the consensus subagent can fix without asking the user. Architecture issues take priority even when the current execution steps are complete. `REVISE` feedback must be actionable. For file or document review, identify the affected location, the problem, and the expected result. If the reason for `REVISE` is that a seemingly `trivial` or `small` change actually triggered `full` review, state which boundary risk caused the escalation. For `plan`, the subagent sends the complete updated plan in the next round.

```text
BLOCKED
Risk classification: <trivial|small|medium|large>
Classification reason: <one concise sentence>
Architecture review mode: <lightweight|full>
- <missing user decision, inaccessible required context, or contradiction that prevents reliable execution>
```

Use `BLOCKED` when architecture judgment or reliable execution needs a missing user decision, inaccessible required context, or resolution of a contradiction. Do not only say that information is insufficient; identify the specific missing decision, inaccessible context, or contradiction and explain why it blocks a reliable judgment.

Claude should separate blocking concerns from non-blocking notes. `REVISE` and `APPROVED_WITH_NOTES` feedback should be concrete enough for the consensus subagent to turn into edits, deferrals, or verification steps. For file or document editing requests, Claude should identify the affected location, problem, and expected result, and a response that only reports opinions when the user asked for edits should be `REVISE`. Claude must not edit files.
