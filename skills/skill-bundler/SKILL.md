---
name: skill-bundler
description: >
  Embed an agent skill into a CLI project so it can be installed via a `skills install`
  subcommand (Go, Python, C#/.NET). Use when asked to: "bundle a skill into my CLI",
  "ship a skill with my tool", "add a skills install command", "embed agent skill",
  "make my CLI agent-aware", "let Copilot/Claude/Codex install context from my CLI".
  Covers canonical install layout (~/.config/<tool>/skills/), symlink fan-out, scoped
  install flags (--user / --project / --path), Windows symlink fallback, drift tests,
  `<tool> help all` machine-readable surface, versioning, safe uninstall, and multi-skill
  bundling.
---

# Skill Bundler

This skill guides you through embedding one or more agent skills directly into a CLI
binary so that users (and AI agents) can install structured knowledge about the tool
with a single command.

## What This Skill Produces

At the end of this workflow your CLI will have:

- **Embedded skill assets** (SKILL.md + optional `references/`) compiled into the binary.
- **`<tool> skills install`** command that extracts those assets to a canonical location
  and links them into agent directories (`~/.agents/skills/`, optionally `.agents/skills/`
  in the project, or any `--path` the user specifies).
- **`<tool> skills uninstall`** command with safe link/copy removal.
- **`<tool> help all`** command emitting a machine-readable command tree that your SKILL.md
  points agents at — so the skill never duplicates CLI syntax.
- **Drift test** that fails CI when commands are added without updating skill assets.
- **CLI Change Checklist** entry in AGENTS.md or CONTRIBUTING.md.

## When to Use

Trigger this skill when any of the following is true:

- The user wants an AI agent to know how to use their CLI tool.
- The user says "ship skill with my CLI", "bundle agent skill", "add skills install",
  or similar.
- A project already has a `SKILL.md` at the root and wants to turn it into a bundled
  installable skill.
- An existing `skills install` command exists but uninstall is unsafe, the drift test
  is missing, or `help all` is absent.

## Decision Tree

1. **Which language?** → [Go](./references/per-language/go.md) |
   [Python](./references/per-language/python.md) |
   [C#/.NET](./references/per-language/dotnet.md)
2. **Need drift test or `help all` guidance?** →
   [Drift and Help All](./references/drift-and-help-all.md)
3. **Want the full architecture rationale?** →
   [Architecture](./references/architecture.md)
4. **Want a concrete end-to-end example?** →
   [Case Study: acme CLI](./references/case-study-acme.md)

---

## Core Architecture

Full rationale lives in `references/architecture.md`. Summary:

```
Binary
  └── embedded assets: internal/skills/assets/<tool>-<purpose>/
        ├── SKILL.md
        └── references/
              ├── bootstrap.md
              └── ...

Install:
  ~/.config/<tool>/skills/<tool>-<purpose>/   ← canonical (single source of truth)
        ↑
        symlink (or copy fallback on Windows)
        ↓
  ~/.agents/skills/<tool>-<purpose>/          ← --user (default)
  .agents/skills/<tool>-<purpose>/            ← --project (opt-in)
  .claude/skills/<tool>-<purpose>/            ← --path .claude/skills (repeatable)
```

### Install Scopes

| Flag | Target | Default? |
|---|---|---|
| `--user` | `~/.agents/skills/<name>/` | Yes (implicit) |
| `--project` | `./.agents/skills/<name>/` | No |
| `--path <dir>` | `<dir>/<name>/` | No, repeatable |

Flags are **additive** — `--user --project --path .claude/skills` links to all three.
Omitting all flags is identical to `--user`.

### Cross-Platform Canonical Path

Use `~/.config/<tool>/skills/` on all platforms (Linux, macOS, Windows). This matches
the convention used by `gh`, `atuin`, and `starship` and avoids platform-branching in
tooling scripts. Users who prefer `%LOCALAPPDATA%` or `~/Library/Application Support/`
on Windows/macOS can override with an env var (e.g. `<TOOL>_SKILLS_HOME`).

### Windows Symlink with Copy Fallback

On Windows, creating a directory symlink requires either Developer Mode or elevated
privileges. Strategy:

1. Attempt `os.Symlink` (or language equivalent).
2. If it fails with `ERROR_PRIVILEGE_NOT_HELD` (Windows syscall error 1314), fall back
   to a full directory copy.
3. Write a marker file `.<tool>-skill-source` inside the copy containing the canonical
   path. The uninstall command reads this marker to verify ownership before removing.

### Naming Convention

Skill folder names **must** be namespaced: `<tool>-<purpose>`.

Examples: `acme-cli`, `acme-debug`, `myapp-ops`.

This prevents collisions when multiple CLIs install into the same shared agent directory
(e.g. `~/.agents/skills/`). A plain name like `cli` or `bootstrap` will clash.

---

## Versioning

Embed a `version` field in the skill's SKILL.md frontmatter or a sibling `VERSION`
plaintext file inside the asset tree. During install, write a `.installed-version`
marker in the canonical directory. The `skills install` command uses this to detect
upgrades and prompt the user when the embedded version is newer.

```yaml
# SKILL.md frontmatter
name: acme-cli
version: "1.4.0"
description: ...
```

---

## Uninstall Semantics

- A **symlink** target: verify it still resolves to the current canonical directory
  before removing. Never remove if the target has been modified by the user.
- A **managed copy** (Windows fallback): read the `.<tool>-skill-source` marker and
  confirm it points to the current canonical. Remove the whole copy directory only if
  the marker matches.
- If the user has modified files inside the skill directory, **do not delete** without
  `--force`. Print a warning listing the modified files.
- Always remove the canonical directory last (or only if `--force` is set), in case
  other agent dirs still link to it.

---

## Multi-Skill Bundling

One CLI may ship multiple skills (e.g. `acme-cli` for end-user workflows and
`acme-debug` for provider debugging).

- `<tool> skills install` with no arguments installs all bundled skills and prints a
  summary.
- `<tool> skills install <skill-name>` installs exactly one.
- Use a registry approach internally: a slice/list of embedded skill manifests, each
  with a name and an `fs.FS` (or language equivalent) pointing at the asset subtree.

---

## `<tool> help all` Convention

Your SKILL.md must **not** duplicate command tables. Instead:

1. Implement `<tool> help all` (or `<tool> help --machine-readable`) that emits a
   machine-readable JSON tree of every non-hidden command with its name, short
   description, and flags.
2. Reference it in your SKILL.md under a **Command Discovery** section:

   > Run `<tool> help all` before using any command you have not run before.

See [Drift and Help All](./references/drift-and-help-all.md) for the output shape and
how to consume it in agent prompts.

---

## Drift Test

Add a test (named `TestSkillDrift` / `test_skill_drift` / `SkillDrift_Test` per
language) that:

1. Walks the live command tree of the application.
2. Filters out **hidden** commands and commands annotated as **debug/internal**.
3. Asserts that every remaining command token (e.g. `"sandbox"`, `"create"`) appears
   at least once in the concatenated text of all embedded skill assets.
4. Fails with a clear message listing which tokens are missing.

See [Drift and Help All](./references/drift-and-help-all.md) for per-language snippets
and the escape-hatch annotation convention.

---

## Skill Content Rules

When writing the SKILL.md that gets bundled:

- **Task-oriented**, not command-reference. Write workflows ("Bootstrap a project"),
  not tables of flags.
- **Progressive loading**: keep the SKILL.md body < 5000 tokens; long procedure detail
  goes in `references/*.md` loaded on demand.
- **Point at `<tool> help all`** for current syntax. Never embed a full command
  synopsis — it will drift.
- **Discovery description** in frontmatter should be ≤ 1024 characters and
  keyword-rich (agent names, action verbs, tool name, major workflows).

---

## Self-Update Interplay (Optional)

If your CLI has a self-update mechanism (`<tool> update`), do **not** auto-refresh
skill assets silently on update. Instead, print a post-update note:

```
Skill assets may be out of date. Run '<tool> skills install --force' to refresh.
```

The agent can act on this message without any hidden side effects.

---

## Repo Hygiene: Excluding Evals

If you maintain an `evals/` folder alongside your skill assets for local testing,
**exclude it from both the embed manifest and git**:

- It may contain prompts or fixtures with local paths, credentials, or
  environment-specific data that must not ship in the binary.
- It is not useful to end users — only to skill authors running local evals.

See each language reference for the specific embed-exclusion syntax. Also add to
`.gitignore`:

```gitignore
# Skill eval fixtures — local only, not shipped
internal/skills/assets/**/evals/
```

---

## CLI Change Checklist

Add this block to your `AGENTS.md` or `CONTRIBUTING.md`:

```markdown
## CLI Change Checklist

Any CLI surface change (new command, renamed command, flag or output shape change)
MUST update the bundled skill assets in the same PR. The drift test enforces this:
a PR that adds a command without updating skill assets will fail CI.
```

---

## Acceptance Checklist

Before declaring this workflow done, verify:

- [ ] Skill assets compile into the binary (embed manifest verified by unit test).
- [ ] `skills install` writes a canonical directory and at least one symlink/copy.
- [ ] `skills uninstall` removes only what it owns; declines without `--force` on
       user-modified content.
- [ ] `<tool> help all` outputs a machine-readable command tree.
- [ ] SKILL.md references `<tool> help all` for syntax; contains no full command
       synopsis.
- [ ] Drift test exists, runs in CI, and covers non-hidden commands.
- [ ] Skill folder is named `<tool>-<purpose>` (namespaced).
- [ ] `evals/` (if present) is excluded from the embed manifest and `.gitignore`d.
- [ ] AGENTS.md / CONTRIBUTING.md contains the CLI Change Checklist entry.
- [ ] `version` field or `VERSION` file present in bundled skill assets.

---

## References

- [Architecture (long-form)](./references/architecture.md)
- [Go recipe](./references/per-language/go.md)
- [Python recipe](./references/per-language/python.md)
- [C#/.NET recipe](./references/per-language/dotnet.md)
- [Drift tests and `help all`](./references/drift-and-help-all.md)
- [Case study: acme CLI](./references/case-study-acme.md)
