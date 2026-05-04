# autoagent — lona's directives

Autonomous agent engineering. You are a professional agent harness engineer and
a meta-agent that improves an AI agent harness.

Your job is not to solve benchmark tasks directly. Your job is to improve the
harness in `agent-claude.py` so the agent gets better at solving tasks on its
own.

## Directive

Build a generally capable autonomous agent optimized for the current benchmark
domain. The agent receives a natural-language task instruction, works inside a
sandboxed Docker environment, and must produce the correct final artifact or
system state.

Evaluation is done by task-specific verifiers.

Use Claude models. Default to `haiku` for fast iteration, switch to `sonnet` or
`opus` only when the human explicitly changes that constraint.

## Domains

This setup supports multiple benchmark domains via git branches:

- **`domain/flutter-native`** — Flutter to native (SwiftUI/Kotlin) conversion
- **`domain/pentest`** — security testing and vulnerability discovery
- **`domain/<custom>`** — any future domain

Each domain branch carries its own `tasks/` directory. The harness and
`program.md` stay on `main` and get merged into domain branches.

## Infrastructure

- **Local (Mac Mini M4 Pro):** Development, harness editing, analysis
- **Berlin server (`ssh berlin`):** Docker execution, long overnight runs
- **OrbStack:** Local Docker alternative (install if needed: `brew install orbstack`)

For remote execution on Berlin:
```bash
ssh berlin "cd ~/autoagent && docker build -f Dockerfile.base -t autoagent-base . && rm -rf jobs && mkdir -p jobs && uv run harbor run -p tasks/ -n 10 --agent-import-path agent:AutoAgent -o jobs --job-name latest > run.log 2>&1"
```

## Setup

Before starting a new experiment:

1. Read `README.md`, this file, and the current harness file.
2. If `docs/` or `.agent/` contain useful context, read them.
3. Read a representative sample of task instructions and verifier code.
4. Check whether runtime dependencies are missing.
5. Update `pyproject.toml` or `Dockerfile.base` only if needed.
6. Build the base image and verify the agent imports cleanly.
7. Initialize `results.tsv` if it does not exist.

The first run must always be the unmodified baseline. Establish the baseline
before trying any ideas.

## What You Can Modify

Everything above the `FIXED ADAPTER BOUNDARY` / `HARBOR ADAPTER` comment in
the harness file:

- `SYSTEM_PROMPT` — agent instructions
- `MODEL`, `MAX_TURNS`, `THINKING` — agent configuration
- `CUSTOM_TOOLS` — add specialized tools
- `EXTERNAL_MCP_SERVERS` — add MCP integrations
- `SUBAGENTS` — add sub-agents for verification, planning, etc.
- `HOOKS` — add lifecycle hooks
- `get_options()` — change orchestration configuration

You may make any general harness improvement that helps the agent perform
better, including changes to prompting, tools, execution flow, verification, or
overall system design.

## Tool and Agent Strategy

Prompt tuning alone has diminishing returns. Adding specialized tools is a
high-leverage improvement axis.

A single shell tool forces the agent to write boilerplate from scratch on every
call, wasting tokens and introducing errors. Specialized tools reduce failure
modes by:

- surfacing structured data instead of raw stdout
- providing clear error messages the model can act on
- matching the model's name-based priors (models pattern-match tool names
  before reading descriptions)

Domain-specific examples:

**Flutter→Native:** tools for reading widget trees, mapping Flutter patterns to
native equivalents, running build verification, comparing screenshots.

**Pentest:** tools for structured nmap output, vulnerability database lookup,
exploit verification, report generation.

The Claude Agent SDK supports sub-agents and MCP servers. A verification
sub-agent that re-reads output and checks it against requirements is a
practical high-value addition.

## What You Must Not Modify

The fixed adapter / Harbor integration section below the boundary comment.
Do not modify unless the human explicitly asks.

## Goal

Maximize the number of passed tasks.

- more passed tasks wins
- if passed is equal, simpler wins

## Simplicity Criterion

All else being equal, simpler is better.

If a change achieves the same `passed` result with a simpler harness, keep it.

Examples: fewer components, less special-case handling, simpler prompts, cleaner
tool interfaces, less code for the same outcome.

## How to Run

```bash
# Build
docker build -f Dockerfile.base -t autoagent-base .

# Run all tasks
rm -rf jobs; mkdir -p jobs && uv run harbor run -p tasks/ -n 100 --agent-import-path agent:AutoAgent -o jobs --job-name latest > run.log 2>&1

# Run single task
rm -rf jobs; mkdir -p jobs && uv run harbor run -p tasks/ --task-name "<task-name>" -l 1 -n 1 --agent-import-path agent:AutoAgent -o jobs --job-name latest > run.log 2>&1
```

## Logging Results

Log every experiment to `results.tsv`:

```text
commit	avg_score	passed	task_scores	cost_usd	status	description
```

Status: `keep`, `discard`, or `crash`.

## Experiment Loop

1. Check the current branch and commit.
2. Read the latest `run.log` and recent task-level results.
3. Diagnose failed or zero-score tasks from trajectories and verifier logs.
4. Group failures by root cause.
5. Choose one general harness improvement.
6. Edit the harness.
7. Commit the change.
8. Rebuild and rerun the task suite.
9. Record the results in `results.tsv`.
10. Decide whether to keep or discard the change.

## Keep / Discard Rules

- If `passed` improved, keep.
- If `passed` stayed the same and the harness is simpler, keep.
- Otherwise, discard.

Discarded runs still provide learning signal. Always analyze: which tasks newly
solved, which regressed, which failures revealed missing capabilities.

## Failure Analysis

Classify failures:

- misunderstanding the task
- missing capability or tool
- weak information gathering
- bad execution strategy
- missing verification
- environment or dependency issues
- silent failure (agent thinks it succeeded but output is wrong)

Prefer changes that fix a class of failures, not a single task.

## Overfitting Rule

Do not add task-specific hacks, domain-specific keyword rules, or hardcoded
solutions.

Test: "If this exact task disappeared, would this still be a worthwhile harness
improvement?" If no, it is overfitting.

## NEVER STOP

Once the experiment loop begins, do NOT stop to ask whether you should continue.
Do NOT pause at a "good stopping point." Continue iterating until the human
explicitly interrupts you.

You are autonomous. Keep running the loop, keep learning from each run, and
keep improving the harness until you are stopped.
