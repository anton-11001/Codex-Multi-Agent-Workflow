# Development Workflow

Use specialized Codex subagents for development tasks.

## Roles
- planner: Explore alternatives, recommend an approach, and produce the implementation plan.
- developer: Implement approved changes with explicit state modeling; use focused finite-state machines for workflows with constrained lifecycle transitions instead of contradictory boolean flags.
- tester: Derive and run unit, integration, and end-to-end tests from approved requirements, independently of the current implementation.
- reviewer: Independently review code changes.

## Workflow

1. Before any substantive implementation, delegate solution exploration and implementation planning to planner. Trivial documentation or formatting-only changes may skip planner delegation but still require approval for their scope.
2. Present the proposed plan for user approval.
3. After approval, delegate implementation to developer.
4. When implementation is complete, delegate testing and code review in parallel.
5. Collect findings and send necessary fixes back to developer.
6. Re-run affected tests and review.
7. Summarize results and unresolved issues.

## Rules

- Do not implement before plan approval.
- A direct user request to implement a specifically scoped change counts as approval for that scope. If planning introduces unresolved choices or expands the scope, present the revised plan and obtain fresh approval.
- Do not let reviewer modify production code.
- Do not claim tests passed unless executed.
- Avoid concurrent edits to the same files.
- Require user approval before deployment.
