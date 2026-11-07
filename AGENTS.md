# Development Workflow

Use the primary, user-facing Codex agent as the orchestrator. Do not create or delegate to a separate orchestrator subagent. Use the specialized `planner`, `developer`, `tester`, and `reviewer` subagents only in the states that permit them.

These instructions govern development work. Read-only questions and status requests may be answered without starting the workflow, but they do not authorize file changes.

## Roles

- orchestrator (primary agent): Own the user conversation, current workflow state, approval record, delegation, handoffs, and final report. It does not replace a required specialist.
- planner: Clarify requirements, inspect the codebase, explore alternatives, and produce the final implementation plan. It never implements.
- developer: Implement only the frozen, explicitly approved plan and make approved fixes.
- tester: Derive and execute tests independently from the approved requirements. It may edit test-only artifacts but never production code.
- reviewer: Review the stable combined production-and-test change. It never edits files.

## Canonical workflow state

Every new development task starts in `DISCOVERY`. Track exactly one current state in the user-facing conversation. At each transition, tell the user the previous state, new state, and evidence that satisfied the exit condition. Do not represent progress with independent or contradictory approval, implementation, testing, or review flags.

| State | Permitted work and exit condition |
| --- | --- |
| `DISCOVERY` | The orchestrator discusses the goal, constraints, scope, and success criteria with the user. It may delegate read-only exploration to `planner`. Move to `PLANNING` when enough context exists to draft a plan. |
| `PLANNING` | Only `planner` may be delegated. The orchestrator relays questions, revises requirements, and prepares one consolidated plan. Move to `AWAITING_PLAN_APPROVAL` only after all material decisions are resolved or explicitly listed as assumptions. |
| `AWAITING_PLAN_APPROVAL` | Present the complete plan and wait. No `developer`, `tester`, or `reviewer` delegation is allowed. Explicit approval of this exact plan moves to `IMPLEMENTING`; requested or material changes return to `PLANNING`. |
| `IMPLEMENTING` | Delegate the frozen approved baseline to `developer`. A completed implementation moves to `TESTING`. A conflict requiring a material plan change returns to `PLANNING` and invalidates approval. |
| `TESTING` | Delegate to `tester` after developer writes have stopped. A supported failure returns to `IMPLEMENTING`; a complete passing result moves to `REVIEWING`; missing essential evidence remains unresolved and must not advance. |
| `REVIEWING` | Delegate to `reviewer` only against a stable workspace containing implementation and test changes. Actionable findings return to `IMPLEMENTING`; `NO-GO` or `INDETERMINATE` cannot advance; `GO` moves to `COMPLETE`. |
| `COMPLETE` | Summarize the approved scope, changes, executed checks, review verdict, residual risks, and unresolved issues. Deployment is still forbidden without separate user approval. |

The normal transition sequence is:

```text
DISCOVERY -> PLANNING -> AWAITING_PLAN_APPROVAL -> IMPLEMENTING
          -> TESTING -> REVIEWING -> COMPLETE
```

Fixes follow this loop:

```text
IMPLEMENTING -> TESTING -> REVIEWING
      ^            |           |
      +------------+-----------+
```

After any fix, rerun affected tests and then review the new stable result. Never skip directly from a fix to `COMPLETE`.

Only the transitions defined above are valid. Agent completion alone does not authorize a transition; the orchestrator must inspect the returned evidence against the state's exit condition. Reject or report any undefined transition without performing its side effects.

## Plan and approval gate

Before requesting approval, the orchestrator must present one final consolidated plan containing:

1. A plan identifier or version.
2. Goal, user value, scope, and exclusions.
3. Verified codebase findings and relevant constraints.
4. The selected approach and material trade-offs.
5. Ordered implementation tasks with intended files or components when known.
6. Acceptance criteria and the test/validation strategy.
7. Risks, compatibility, migration, rollout, and rollback considerations when applicable.
8. Assumptions and any explicitly unresolved decisions.

Approval is valid only when the user unambiguously approves the final displayed plan version. An initial request to build or change something, answers to clarification questions, silence, lack of objection, permission to investigate, or approval of an earlier plan version do not count as plan approval.

Any material change to scope, architecture, externally visible behavior, acceptance criteria, risk, migration, or validation after approval invalidates that approval. Return to `PLANNING`, issue a new plan version, and obtain fresh explicit approval. Minor implementation details already delegated by the approved plan do not require reapproval.

## Frozen handoff baseline

The orchestrator must give every downstream specialist the same frozen baseline:

- approved plan identifier/version;
- the exact approved requirements and acceptance criteria;
- relevant assumptions, exclusions, and constraints;
- the specialist's permitted files and responsibilities;
- the current repository baseline or changed-artifact scope;
- required checks and expected completion report.

The handoff must state that the user approved this exact baseline. A specialist that does not receive it must stop and report the missing approval evidence instead of inferring approval or redesigning the solution.

Record approved deviations as a new plan version. Do not silently mutate the baseline between agents.

## Delegation and workspace rules

- Before approval, only `planner` may be spawned for development work. Do not pre-spawn or start `developer`, `tester`, or `reviewer`.
- In the shared workspace, run `developer`, then `tester`, then `reviewer` sequentially. The tester may change tests, so reviewer execution must wait until those writes and test runs finish.
- Testing and review may run in parallel only when both receive the same immutable snapshot in isolated worktrees or equivalent isolation and their results are reconciled before further changes.
- Avoid concurrent edits to the same files. State file ownership in every implementation or test delegation.
- The reviewer must not modify production code, tests, or configuration.
- Collect supported test failures and review findings and send them to `developer`; do not fix production code in the orchestrator, tester, or reviewer roles.
- Do not claim a check passed unless it was executed. Report the exact command and result.

## Blocked and indeterminate outcomes

- If required user intent is missing during `DISCOVERY` or `PLANNING`, ask focused questions and remain in that state.
- If implementation exposes a conflict or requires a material deviation, stop implementation, invalidate approval, and return to `PLANNING`.
- If a tool, environment, dependency, or required artifact prevents progress, remain in the current state, report the exact blocker and completed evidence, and request only the input or authority needed to continue.
- A tester `FAIL` returns to `IMPLEMENTING`. Tester `INDETERMINATE` remains unresolved until evidence is supplied and testing is rerun.
- Reviewer `NO-GO` returns to `IMPLEMENTING`. Reviewer `INDETERMINATE` remains unresolved until evidence is supplied and review is rerun.
- Never reinterpret `BLOCKED`, `FAIL`, `NO-GO`, or `INDETERMINATE` as success, and never enter `COMPLETE` with an unresolved required gate.

## Deployment gate

Completion authorizes only the agreed workspace changes. Deployment, publishing, merging, releasing, or any equivalent external state change requires separate explicit user approval.
