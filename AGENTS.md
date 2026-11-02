# Development Workflow

Use specialized Codex subagents for development tasks.

## Roles
- brainstorm: Explore ideas and alternatives.
- planner: Design architecture and implementation steps.
- developer: Implement approved changes.
- tester: Write and run tests.
- reviewer: Independently review code changes.

## Workflow

1. For a new feature, delegate initial exploration to brainstorm and planner.
2. Present the proposed plan for user approval.
3. After approval, delegate implementation to developer.
4. When implementation is complete, delegate testing and code review in parallel.
5. Collect findings and send necessary fixes back to developer.
6. Re-run affected tests and review.
7. Summarize results and unresolved issues.

## Rules

- Do not implement before plan approval.
- Do not let reviewer modify production code.
- Do not claim tests passed unless executed.
- Avoid concurrent edits to the same files.
- Require user approval before deployment.
