---
name: agent-delegator
description: >
  Delegate implementation work to external coding agents (Codex CLI, Claude CLI, or Claude subagents)
  to save tokens while maintaining quality through supervised orchestration. Use this skill whenever
  the user asks to "delegate", "offload", "use codex", "save tokens", "have another agent do it",
  "kick it off in the background", or when there's a well-scoped implementation task that could be
  completed by an external agent. Also trigger when the user references reducing cost, token efficiency,
  or automating coding tasks through other AI tools.
---

# Agent Delegator

Orchestrate external coding agents to complete well-scoped implementation tasks while you supervise. This saves tokens in the main conversation by offloading the heavy file-reading and code-writing to a cheaper/separate context, while you handle planning, context gathering, prompt crafting, and quality review.

## When to use this

- Implementation tasks with a clear scope (not exploratory/research)
- Tasks that involve reading many files and writing code across them
- When the user explicitly asks to save tokens or delegate work
- Refactors, migrations, test writing, feature implementations with a known plan

## When NOT to use this

- Quick edits (a few lines) — just do it yourself
- Research or exploration tasks — agents need clear instructions
- Tasks requiring interactive back-and-forth with the user mid-implementation

## Workflow Overview

```
1. SCOPE    → Break the work into discrete stages
2. CONTEXT  → Read files, gather project conventions, build the prompt
3. DELEGATE → Launch the agent with a complete, self-contained prompt
4. MONITOR  → Periodic lightweight check-ins
5. REVIEW   → Code review the output, fix issues
6. VERIFY   → Run tests and lint
```

## Step 1: Scope and Stage the Work

Before delegating anything, break the task into stages that can be independently verified. Each stage should produce a testable/reviewable result.

Good staging example:
- Stage 1: Create shared helper functions
- Stage 2: Rewrite module A to use helpers
- Stage 3: Rewrite module B to use helpers
- Stage 4: Update tests

Between stages, review the output and course-correct before proceeding. This prevents compounding errors.

## Step 2: Gather Context and Build the Prompt

This is the most critical step. The agent operates in a separate context — it can only work with what you give it. A weak prompt produces garbage; a strong prompt produces nearly-finished work.

### What to include in the prompt

1. **Project conventions** — Read `AGENTS.md`, `CLAUDE.md`, or equivalent and include a truncated version with the sections relevant to the task (commands, structure, values, style).

2. **File contents** — Read every file the agent will need to modify or reference. Include the full content of files to modify, and relevant excerpts of files for reference.

3. **The plan** — Be extremely specific about what to do. Not "rewrite the tests" but "rewrite TestFoo in foo_test.go to use httptest.NewServer instead of the DiskStore mock. The API returns JSON matching the model.Sandbox struct."

4. **Constraints** — Mention linter rules, permission patterns, import conventions, error handling style.

5. **Verification command** — Tell the agent what command to run to verify its work (e.g., `task test`, `task lint`).

### Prompt template

```
You are working on [project name], a [brief description].

## Project Conventions
[Truncated AGENTS.md / CLAUDE.md relevant sections]

## Task
[Specific, actionable description of what to implement]

## Files to Modify
[List each file with its current content]

## Reference Files (read-only context)
[Files the agent needs to understand but shouldn't modify]

## Constraints
- [Linter rules, patterns, style requirements]
- [Import conventions]
- [Error handling patterns]

## Verification
Run these commands when done:
- [test command]
- [lint command]
```

## Step 3: Delegate to an Agent

Choose a backend based on availability and task characteristics.

### Backend: Codex CLI (`codex exec`)

Best for: Tasks where you want sandboxed execution with full disk access.

```bash
codex exec "YOUR PROMPT HERE" \
  --full-auto \
  -C /path/to/project \
  -m o3
```

Key flags:
- `--full-auto` — Auto-approve in a sandboxed workspace (safe default)
- `-C <dir>` — Set working directory
- `-m <model>` — Model selection (default: o3)
- `--json` — JSONL event stream for programmatic monitoring
- `-o <file>` — Write the agent's last message to a file
- `-s workspace-write` — Sandbox with workspace write access
- `--dangerously-bypass-approvals-and-sandbox` — Full access, no sandbox (use with caution)

### Backend: Claude CLI (`claude -p`)

Best for: Tasks requiring Claude's tool use and file editing capabilities.

```bash
claude -p "YOUR PROMPT HERE" \
  --dangerously-skip-permissions \
  --model sonnet \
  --output-format json \
  --max-budget-usd 5.00
```

Key flags:
- `-p` / `--print` — Non-interactive mode (required)
- `--dangerously-skip-permissions` — Allow autonomous file edits
- `--permission-mode acceptEdits` — Safer: auto-approve edits only
- `--model <model>` — Model selection (opus, sonnet, haiku)
- `--output-format json` — Structured output
- `--max-budget-usd <n>` — Cost cap
- `--system-prompt "..."` — Override system prompt
- `--append-system-prompt "..."` — Add to default system prompt
- `--bare` — Clean run: skip hooks, CLAUDE.md, LSP discovery
- `--effort high` — Reasoning depth (low/medium/high/max)
- `--allowedTools "Bash Edit Read Grep Glob"` — Restrict tool access
- `--no-session-persistence` — Don't save session to disk

### Backend: Claude Code Subagent (Agent tool)

Best for: When you're already in Claude Code and want to delegate within the same environment.

Use the `Agent` tool with `subagent_type: "general-purpose"` and `run_in_background: true`. The subagent inherits the project context and tool access.

This is the lightest-weight option but uses the same token pool. Use when the task needs Claude Code's tools but you want to keep your main context clean.

## Step 4: Monitor Progress

For background processes, check in periodically without burning tokens:

**For Bash background tasks** (`run_in_background: true`):
- Read the output file path periodically
- Look for error patterns or completion signals
- Don't read the entire output — just tail the last ~20 lines

**For subagents**:
- Wait for the completion notification (automatic)
- Don't poll or sleep — you'll be notified

**For long-running Codex/Claude CLI**:
- If launched with `run_in_background`, same as above — tail the output
- Check for the process completion

The goal is to avoid spending tokens on monitoring. A quick tail of the last few lines tells you if the agent is stuck, progressing, or done.

## Step 5: Code Review

After the agent completes, review its work before accepting it. This is non-negotiable — agents produce good-but-not-perfect code.

### Review checklist

1. **Correctness** — Does the code do what was asked?
2. **Conventions** — Does it follow the project's patterns (from AGENTS.md)?
3. **Imports** — Are imports correct and consistent with the codebase?
4. **Error handling** — Does it match the project's error handling style?
5. **Tests** — Are tests meaningful, not just passing?
6. **No regressions** — Did the agent accidentally modify unrelated code?
7. **Security** — No hardcoded secrets, proper permission handling?

Use `git diff` to review what changed. For large changes, use the code-reviewer agent.

### Common agent mistakes to watch for

- Adding unnecessary dependencies
- Using patterns inconsistent with the codebase
- Over-engineering (adding abstractions the codebase doesn't use)
- Missing error handling
- Wrong import paths
- Leaving debug/TODO comments

## Step 6: Verify

Run the project's test and lint commands. Always use Taskfile tasks if available.

```bash
task test
task lint
task fmt
```

Fix any issues the agent introduced. If the fixes are minor, do them yourself. If they're substantial, consider re-delegating with a more specific prompt that addresses the failures.

## Choosing a Backend

| Factor | Codex | Claude CLI | Subagent |
|--------|-------|-----------|----------|
| Token cost to main session | None | None | Shared pool |
| Sandbox support | Yes (default) | No | No |
| Working directory control | `-C` flag | Run from dir | Inherits |
| File editing | Shell-based | Tool-based | Tool-based |
| Progress monitoring | `--json` output | Output format | Auto-notify |
| Best for | Large refactors | Complex tool use | Quick delegation |

## Tips

- **Long prompts**: For Codex and Claude CLI, pipe the prompt via stdin or write to a temp file. Shell quoting breaks on multi-paragraph prompts.
- **Iterative stages**: After each stage, review and fix before delegating the next. Compounding errors across stages is expensive to fix.
- **When agents get stuck**: If an agent produces bad output, don't re-run with the same prompt. Analyze what went wrong, add more specific constraints, and include examples of the desired pattern.
- **Cost vs quality**: Use cheaper models (sonnet, o3-mini) for mechanical tasks (formatting, renaming, boilerplate). Use stronger models (opus, o3) for tasks requiring architectural judgment.
