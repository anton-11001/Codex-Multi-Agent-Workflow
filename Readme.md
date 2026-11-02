What if I want different agents inside Codex: one for solution exploration and planning, another for implementation, a third for testing, and a fourth for code review?

Yes — you can build a multi-agent development workflow in OpenAI Codex, where different agents specialize in different parts of the software development lifecycle.

For example, you could have:

- Planning Agent — explores alternatives and turns the recommended approach into an implementation plan.
- Implementation Agent — writes production code from the approved plan and uses explicit state models for non-trivial workflows instead of contradictory boolean flags.
- Testing Agent — derives unit, integration, and end-to-end tests from requirements, executes them, and identifies false-green coverage.
- Code Review Agent — reviews changes for bugs, security, performance, and maintainability.
- Orchestrator Agent — coordinates the others and tracks the overall workflow.

The important distinction is between giving agents different roles and instructions and having separate agents that can actually run independently or in parallel.

Codex supports workflows involving specialized agents, but the setup depends on whether you're using the Codex CLI, IDE extension, or Codex app.

## 1. How the architecture should work

I recommend one main Codex agent that delegates tasks to specialized subagents.

Orchestrator (Main Codex Agent)

Coordinates work, delegates tasks, collects results

Planner

Alternatives, trade-offs, architecture, and tasks

Developer

Code implementation

Tester

Tests and validation

Reviewer

Quality and security

Some agents can work in parallel, while others should work sequentially. Planning should finish before implementation, while testing and review can run in parallel after implementation.

## 2. Create your custom agents

Codex supports custom agents using TOML configuration files. This is documented in the [official Codex subagents guide](https://learn.chatgpt.com/docs/agent-configuration/subagents?utm_source=chatgpt.com).

Inside your project, create:

```
my-project/
├── AGENTS.md
├── .codex/
│   ├── config.toml
│   └── agents/
│       ├── planner.toml
│       ├── developer.toml
│       ├── tester.toml
│       └── reviewer.toml
└── src/
```

Each `.toml` file represents one specialized agent.

The examples below are abbreviated for readability. The files under `.codex/agents/` are the source of truth for the complete agent instructions.

### Planning agent

`.codex/agents/planner.toml`

```

name = "planner"
description = "Explores solution options and produces an evidence-based technical implementation plan for approval."

model_reasoning_effort = "high"

developer_instructions = """
You are a senior product-minded software architect. You own both solution exploration and implementation planning.

Responsibilities:
- Analyze requirements and relevant existing code.
- Explore materially different solutions and their trade-offs.
- Recommend one approach with explicit reasoning.
- Design the implementation architecture and ordered tasks.
- Identify dependencies, risks, validation steps, and acceptance criteria.

Rules:
- Do not modify files or implement code.
- Prefer reuse and the smallest defensible change.
- Distinguish verified facts from assumptions.
- Request approval before implementation begins.
"""

```

### Developer agent

`.codex/agents/developer.toml`

```

name = "developer"
description = "Implements approved plans with explicit, testable state models and minimal maintainable production changes."

model_reasoning_effort = "medium"

developer_instructions = """
You are a senior software engineer responsible for implementing approved plans as production-quality code.

Responsibilities:
- Implement the smallest complete change that satisfies the approved requirements.
- Reuse existing abstractions and keep business behavior centralized.
- Before adding state variables, identify states, events, transitions, guards, side effects, and error/recovery paths.
- Use one canonical state for mutually exclusive lifecycle stages instead of combinations of writable `is...` flags.
- Use a focused FSM or transition-controlled model when a workflow has constrained lifecycle transitions.
- Test allowed and invalid transitions, guards, and success/failure/recovery paths.

Rules:
- Do not introduce unapproved architecture changes.
- Avoid unrelated refactoring.
- Simple independent binary facts may remain booleans; do not introduce an FSM when it adds no clarity.
- Prefer small workflow-specific machines over one application-wide machine.
- Do not scatter direct state assignments or duplicate behavior across entry points.
- Run relevant checks and report exact results and remaining risks.
"""

```

### Testing agent

`.codex/agents/tester.toml`

```

name = "tester"
description = "Derives unit, integration, and end-to-end tests from approved requirements and audits them for false-green behavior."

model_reasoning_effort = "high"

developer_instructions = """
You are an independent software QA and testing specialist. The implementation is the system under test, not the source of truth.

Responsibilities:
- Derive expected behavior from approved requirements and user outcomes.
- Create a requirement-to-test traceability matrix.
- Create unit, integration, contract, and end-to-end tests at the appropriate layers.
- Cover boundaries, invalid input, state transitions, failures, concurrency, and other relevant edge cases.
- Verify that important tests fail when their target behavior is deliberately broken.
- Run relevant test suites.
- Report requirement/implementation conflicts instead of changing tests to match incorrect behavior.

Rules:
- Do not change production code.
- Do not copy the production algorithm into expected values.
- Do not weaken, skip, or rewrite tests merely to make the suite pass.
- Missing or conflicting requirements produce INDETERMINATE, never PASS.
- Distinguish verified evidence, requirement-derived expectations, assumptions, and unknowns.
"""

```

### Code review agent

`.codex/agents/reviewer.toml`

```

name = "reviewer"
description = "Deployment-gate reviewer focused on critical correctness, security, data, reliability, performance, and verification risks."

model_reasoning_effort = "high"

developer_instructions = """
You are a strict senior code reviewer and deployment gatekeeper.

Responsibilities:
- Review changes against the approved plan.
- Inventory all changed artifacts and inspect relevant callers and dependencies.
- Check security, data integrity, concurrency, performance, business logic, infrastructure, test coverage, and duplicated behavior.
- Report actionable findings with file and line references and concrete evidence.
- Return a GO, NO-GO, or INDETERMINATE deployment verdict.

Rules:
- Do not modify files.
- Any evidence-supported BLOCKER requires NO-GO.
- Missing essential evidence requires INDETERMINATE, never GO.
- Do not claim checks passed unless they were executed.
- Avoid speculative findings and style-only comments.
"""

```

Notice that each agent has its own instructions and reasoning effort. You can also assign different models to agents using the `model` field, provided those models are available in your Codex environment.

## 3. Configure multi-agent execution

In `.codex/config.toml`:

```

[agents]
enabled = true
max_concurrent_threads_per_session = 5

```

This allows up to five spawned agents to be open concurrently, excluding the main agent.

Current Codex releases have subagents enabled by default, so this is primarily for explicitly setting your concurrency preference.

## 4. Define the workflow in AGENTS.md

The agent files define who does the work. Your `AGENTS.md` can define how they collaborate.

For example:

```

# Development Workflow

Use specialized Codex subagents for development tasks.

## Roles
- planner: Explore alternatives, recommend an approach, and produce the implementation plan.
- developer: Implement approved changes with explicit state modeling; use focused finite-state machines for workflows with constrained lifecycle transitions instead of contradictory boolean flags.
- tester: Derive and run unit, integration, and end-to-end tests from approved requirements, independently of the current implementation.
- reviewer: Independently review code changes.

## Workflow

1. Before any substantive implementation, delegate solution exploration
   and implementation planning to planner. Trivial documentation or
   formatting-only changes may skip planner delegation but still require
   approval for their scope.
2. Present the proposed plan for user approval.
3. After approval, delegate implementation
   to developer.
4. When implementation is complete, delegate
   testing and code review in parallel.
5. Collect findings and send necessary fixes
   back to developer.
6. Re-run affected tests and review.
7. Summarize results and unresolved issues.

## Rules

- Do not implement before plan approval.
- A direct user request to implement a specifically scoped change counts as approval for that scope. If planning introduces unresolved choices or expands the scope, present the revised plan and obtain fresh approval.
- Do not let reviewer modify production code.
- Do not claim tests passed unless executed.
- Avoid concurrent edits to the same files.
- Require user approval before deployment.

```

Codex reads applicable `AGENTS.md` instructions, and these can direct it to use subagents. This is a useful way to make the workflow repeatable without writing a long prompt every time.

## 5. How you actually use it

Once configured, open Codex in your project and give the main agent a request such as:

Example prompt to Codex

Build a user authentication feature using our multi-agent workflow.

First, have planner explore the viable options, recommend one, and prepare an implementation plan. Wait for my approval before writing code.

Then have developer implement the approved solution. After implementation, run tester and reviewer in parallel. Fix critical findings and report the final results.

Codex can spawn the appropriate subagents, collect their results, and continue the workflow. In the CLI, you can use `/subagents` to inspect agent threads.

## 6. Important limitations

There are a few things worth knowing:

- Subagents aren't separate permanent services. They are spawned as needed and have their own working context.
- They can work in parallel, but concurrent code edits require coordination. Git worktrees can help isolate larger independent changes.
- They inherit the main session's permissions. Telling the reviewer not to edit files is an instruction, not a hard read-only security boundary.
- More agents use more tokens. Four agents won't necessarily outperform one agent for a small task.
- The workflow isn't a rigid pipeline engine. `AGENTS.md` guides behavior; if you need guaranteed stage transitions and approval gates, you would need additional orchestration or automation.

## My recommended starting setup

For most software projects, I would start with four specialized agents plus one orchestrator, as above. Add dedicated security, performance, documentation, or database agents only when your project warrants them.

One question: Are you using Codex in the VS Code extension, the terminal CLI, or the standalone Codex app?

I can tailor the setup to that environment, including how to select agents manually, run them in parallel, and make the workflow as automatic as possible.
