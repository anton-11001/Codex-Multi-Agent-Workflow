What if I want to have a different agents inside codex, one for example for brainstorm and plan, second for code implementation, 3 for testing, 4 for code review and so on, is it possible to do and how I can do that

Yes — you can build a multi-agent development workflow in OpenAI Codex, where different agents specialize in different parts of the software development lifecycle.

For example, you could have:

- Brainstorm Agent — explores ideas, requirements, and architectural approaches.
- Planning Agent — turns ideas into implementation plans and tasks.
- Implementation Agent — writes code based on the approved plan.
- Testing Agent — creates tests, executes them, and identifies failures.
- Code Review Agent — reviews changes for bugs, security, performance, and maintainability.
- Orchestrator Agent — coordinates the others and tracks the overall workflow.

The important distinction is between giving agents different roles and instructions and having separate agents that can actually run independently or in parallel.

Codex supports workflows involving specialized agents, but the setup depends on whether you're using the Codex CLI, IDE extension, or Codex app.

## 1. How the architecture should work

I recommend one main Codex agent that delegates tasks to specialized subagents.

Orchestrator (Main Codex Agent)

Coordinates work, delegates tasks, collects results

Brainstorm

Ideas and trade-offs

Planner

Architecture and tasks

Developer

Code implementation

Tester

Tests and validation

Reviewer

Quality and security

Docs

Documentation

Some agents can work in parallel, while others should work sequentially. For example, brainstorming and codebase exploration can happen together, but implementation should generally wait for an approved plan.

## 2. Create your custom agents

Codex supports custom agents using TOML configuration files. This is documented in the [official Codex subagents guide](https://learn.chatgpt.com/docs/agent-configuration/subagents?utm_source=chatgpt.com).

Inside your project, create:

```
my-project/
├── AGENTS.md
├── .codex/
│   ├── config.toml
│   └── agents/
│       ├── brainstorm.toml
│       ├── planner.toml
│       ├── developer.toml
│       ├── tester.toml
│       └── reviewer.toml
└── src/
```

Each `.toml` file represents one specialized agent.

### Brainstorm agent

`.codex/agents/brainstorm.toml`

```

name = "brainstorm"
description = "Explores feature ideas, alternatives, and trade-offs."

model_reasoning_effort = "high"

developer_instructions = """
You are a product and technical brainstorming specialist.

Responsibilities:
- Explore multiple possible solutions.
- Identify technical trade-offs.
- Challenge assumptions.
- Consider scalability and maintainability.
- Recommend approaches with reasoning.

Rules:
- Do not modify code.
- Do not implement features.
- Present alternatives before recommending one.
"""

```

### Planning agent

`.codex/agents/planner.toml`

```

name = "planner"
description = "Creates technical implementation plans and architecture."

model_reasoning_effort = "high"

developer_instructions = """
You are a senior software architect.

Responsibilities:
- Analyze requirements and existing code.
- Design implementation architecture.
- Break work into small tasks.
- Identify dependencies and risks.
- Define acceptance criteria.

Rules:
- Do not implement code.
- Prefer existing project conventions.
- Produce actionable implementation plans.
- Identify decisions requiring user approval.
"""

```

### Developer agent

`.codex/agents/developer.toml`

```

name = "developer"
description = "Implements approved plans and fixes."

model_reasoning_effort = "medium"

developer_instructions = """
You are a senior software engineer.

Responsibilities:
- Implement approved technical plans.
- Follow existing architecture and conventions.
- Write clean, maintainable code.
- Make minimal necessary changes.

Rules:
- Do not introduce unapproved architecture changes.
- Avoid unrelated refactoring.
- Run relevant checks when possible.
- Report changed files and remaining risks.
"""

```

### Testing agent

`.codex/agents/tester.toml`

```

name = "tester"
description = "Creates and runs tests, validates implementation."

model_reasoning_effort = "high"

developer_instructions = """
You are a software QA and testing specialist.

Responsibilities:
- Analyze implemented functionality.
- Create unit and integration tests.
- Identify edge cases.
- Run relevant test suites.
- Report failures with reproduction steps.

Rules:
- Do not change production code.
- Do not hide or skip failing tests.
- Distinguish verified results from assumptions.
"""

```

### Code review agent

`.codex/agents/reviewer.toml`

```

name = "reviewer"
description = "Reviews changes for correctness, security, and quality."

model_reasoning_effort = "high"

developer_instructions = """
You are a strict senior code reviewer.

Responsibilities:
- Review changes against the approved plan.
- Identify bugs and security vulnerabilities.
- Find performance and concurrency issues.
- Check maintainability and test coverage.
- Report actionable findings with file references.

Rules:
- Do not modify files.
- Prioritize correctness over style.
- Classify findings by severity.
- Avoid speculative or unsupported findings.
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

Current Codex releases have subagents enabled by default, so this is primarily for explicitly setting your concurrency preference.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://learn.chatgpt.com&sz=32)

ChatGPT Learn+1

## 4. Define the workflow in AGENTS.md

The agent files define who does the work. Your `AGENTS.md` can define how they collaborate.

For example:

```

# Development Workflow

Use specialized Codex subagents for development tasks.

## Roles
- brainstorm: Explore ideas and alternatives.
- planner: Design architecture and implementation steps.
- developer: Implement approved changes.
- tester: Write and run tests.
- reviewer: Independently review code changes.

## Workflow

1. For a new feature, delegate initial exploration
   to brainstorm and planner.
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
- Do not let reviewer modify production code.
- Do not claim tests passed unless executed.
- Avoid concurrent edits to the same files.
- Require user approval before deployment.

```

Codex reads applicable `AGENTS.md` instructions, and these can direct it to use subagents. This is a useful way to make the workflow repeatable without writing a long prompt every time.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://learn.chatgpt.com&sz=32)

ChatGPT Learn+1

## 5. How you actually use it

Once configured, open Codex in your project and give the main agent a request such as:

Example prompt to Codex

Build a user authentication feature using our multi-agent workflow.

First, have brainstorm explore options and planner prepare an implementation plan. Wait for my approval before writing code.

Then have developer implement the approved solution. After implementation, run tester and reviewer in parallel. Fix critical findings and report the final results.

&#x20;Copy prompt

Codex can spawn the appropriate subagents, collect their results, and continue the workflow. In the CLI, you can use `/subagents` to inspect agent threads.

## 6. Important limitations

There are a few things worth knowing:

- Subagents aren't separate permanent services. They are spawned as needed and have their own working context.
- They can work in parallel, but concurrent code edits require coordination. Git worktrees can help isolate larger independent changes.
- They inherit the main session's permissions. Telling the reviewer not to edit files is an instruction, not a hard read-only security boundary.
- More agents use more tokens. Five agents won't necessarily outperform one agent for a small task.
- The workflow isn't a rigid pipeline engine. `AGENTS.md` guides behavior; if you need guaranteed stage transitions and approval gates, you would need additional orchestration or automation.

## My recommended starting setup

For most software projects, I would start with five specialized agents plus one orchestrator, as above. Add dedicated security, performance, documentation, or database agents only when your project warrants them.

One question: Are you using Codex in the VS Code extension, the terminal CLI, or the standalone Codex app?

I can tailor the setup to that environment, including how to select agents manually, run them in parallel, and make the workflow as automatic as possible.
