# Codex Multi-Agent Development Workflow

This project configures one primary Codex agent to orchestrate four specialized subagents:

- `planner` explores the problem, clarifies requirements, and produces a plan.
- `developer` implements an explicitly approved plan.
- `tester` derives and executes requirement-based tests.
- `reviewer` independently reviews the stable implementation and test changes.

The primary agent is the orchestrator because it owns the user conversation and approval record. There is intentionally no `orchestrator.toml`: another subagent would add a context handoff without making the approval boundary stronger.

`AGENTS.md` is the canonical workflow contract. The files under `.codex/agents/` define each specialist's responsibilities, and `.codex/config.toml` enables subagent execution.

## Architecture

```text
User <-> Primary Codex agent (orchestrator)
                    |
                    +-- planner
                    +-- developer
                    +-- tester
                    +-- reviewer
```

The orchestrator selects agents; specialists do not independently advance the workflow. A specialist receives a bounded handoff and returns evidence to the orchestrator.

## Lifecycle

The workflow has one canonical state:

```text
DISCOVERY
    -> PLANNING
    -> AWAITING_PLAN_APPROVAL
    -> IMPLEMENTING
    -> TESTING
    -> REVIEWING
    -> COMPLETE
```

Every development task begins in `DISCOVERY`. The primary agent announces each state transition and the evidence that opened the next gate. Agent completion by itself never advances the workflow, and transitions not defined by `AGENTS.md` are forbidden.

The gates are intentionally sequential:

1. During `DISCOVERY`, the primary agent discusses the desired outcome, scope, constraints, and success criteria with the user. It may ask `planner` to inspect the repository.
2. During `PLANNING`, only `planner` may be delegated. The primary agent relays questions and consolidates the answers into a versioned final plan.
3. During `AWAITING_PLAN_APPROVAL`, the plan is displayed and work stops until the user explicitly approves that exact version.
4. During `IMPLEMENTING`, `developer` receives the frozen approved baseline and owns production changes.
5. During `TESTING`, `tester` receives the same baseline, may add test-only artifacts, and executes the required checks after developer writes have stopped.
6. During `REVIEWING`, `reviewer` examines the stable combined production-and-test diff without modifying it.
7. `COMPLETE` is reached only after required tests pass and review returns `GO`.

A test failure or review finding returns the workflow to `IMPLEMENTING`. Every fix is tested and reviewed again:

```text
IMPLEMENTING -> TESTING -> REVIEWING
      ^            |           |
      +------------+-----------+
```

## Approval semantics

The final plan includes a plan identifier or version, scope and exclusions, verified codebase findings, the selected approach, ordered tasks, acceptance criteria, validation, risks, and assumptions.

Only an unambiguous user response approving that final displayed plan version opens the implementation gate. None of the following count as approval:

- the initial request to build or change something;
- answers to clarification questions;
- silence or lack of objection;
- permission to inspect or plan;
- approval of an older plan version.

A material change to scope, architecture, externally visible behavior, acceptance criteria, risk, migration, or validation invalidates approval. The workflow returns to `PLANNING`, creates a new version, and requests approval again.

This is an instruction-level gate. If an organization requires non-bypassable, persisted, or auditable approval across independent sessions, it should add an external workflow engine or approval store.

## Handoff contract

Every downstream handoff contains the same frozen baseline:

- the explicitly approved plan identifier/version;
- requirements and acceptance criteria;
- assumptions, exclusions, and constraints;
- the receiving agent's responsibility and file ownership;
- the repository baseline or changed-artifact scope;
- required checks and completion-report format.

Developer, tester, and reviewer must stop if approval evidence or the baseline is missing. They may not infer approval, redesign the solution, or silently change the plan.

If a new primary session is started after planning, persist or paste the full versioned plan and its explicit approval record into that session. A reference such as "use the previous plan" is not a sufficient frozen baseline when the previous conversation is unavailable.

## Stable verification

The shared workspace sequence is:

```text
developer -> tester -> reviewer
```

Testing and review are not run concurrently in one workspace because the tester may still write test artifacts while the reviewer is inventorying changes. They may run in parallel only against the same immutable snapshot in isolated worktrees or equivalent isolation.

Outcomes are handled as follows:

- tester `PASS` advances to review;
- tester `FAIL` returns to implementation;
- tester `INDETERMINATE` remains unresolved until evidence is supplied and testing is rerun;
- reviewer `GO` advances to completion;
- reviewer `NO-GO` returns to implementation;
- reviewer `INDETERMINATE` remains unresolved until evidence is supplied and review is rerun.

Missing evidence is never treated as success. Checks are reported as passed only when the exact command was executed successfully.

## Project configuration

```text
.
|-- AGENTS.md
|-- .codex/
|   |-- config.toml
|   `-- agents/
|       |-- planner.toml
|       |-- developer.toml
|       |-- tester.toml
|       `-- reviewer.toml
`-- src/
```

`.codex/config.toml` enables subagents and configures concurrency. Concurrency is capacity, not permission to bypass lifecycle gates.

Each role file is the source of truth for that specialist's detailed behavior. `AGENTS.md` remains the source of truth for ordering, approval, state transitions, and cross-agent handoffs.

## Example interaction

```text
User: Build a user authentication feature using the project workflow.

Primary agent:
1. Enters DISCOVERY and clarifies the outcome and constraints.
2. Delegates exploration and planning to planner.
3. Relays questions and produces Plan auth-v1.
4. Displays auth-v1 and waits without starting implementation.

User: I approve plan auth-v1.

Primary agent:
5. Delegates auth-v1 to developer.
6. After implementation stops, delegates auth-v1 to tester.
7. After test writes and runs stop, delegates the stable result to reviewer.
8. Routes any findings through the full fix-test-review loop.
9. Reports completion and residual risks.
```

## Deployment

Workflow completion authorizes only the approved workspace changes. Merging, deployment, publishing, releasing, or another external state change requires separate explicit user approval.
