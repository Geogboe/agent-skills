# Case Study: acme CLI

This document walks through adding a bundled agent skill to a fictional Go CLI called
`acme`. It illustrates every concept from the architecture in a concrete, linear order.

`acme` is a deployment tool with commands:
- `acme deploy`
- `acme rollback`
- `acme status`
- `acme config validate`
- `acme update` (self-update)
- `acme skills install` / `acme skills uninstall`
- `acme help all`

---

## Step 1: Design the Skill

Before writing any code, decide what the skill content will be.

**Goal:** AI agents using `acme` should know how to deploy safely, roll back when
something goes wrong, and diagnose failed deployments — without the agent needing to
guess command syntax.

**Skill name:** `acme-cli` (namespaced: `<tool>-<purpose>`).

**Reference workflows:**
- `deploy-workflow.md` — full deploy → verify → proceed flow
- `rollback-workflow.md` — when and how to roll back
- `diagnose.md` — interpreting `acme status` output

---

## Step 2: Create the Asset Tree

```
internal/
  skills/
    assets/
      acme-cli/
        SKILL.md
        VERSION
        references/
          deploy-workflow.md
          rollback-workflow.md
          diagnose.md
    skills.go
    link_unix.go
    link_windows.go
    skills_test.go
    drift_test.go
```

`VERSION` contents: `1.0.0`

`SKILL.md` frontmatter (targeting ≤ 1024 chars in description):

```yaml
---
name: acme-cli
version: "1.0.0"
description: >
  Use when working with the acme deployment CLI: deploying services, rolling back
  releases, diagnosing failures, validating config, and monitoring deployment status.
  Relevant for GitHub Copilot, Claude Code, Codex, and other AI agents operating
  acme safely. Trigger phrases: deploy with acme, acme rollback, acme status failed,
  acme config validate, acme update.
---
```

`SKILL.md` body opens with command discovery:

```markdown
## Command Discovery

Do not guess command syntax.

Run `acme help all` before using any command you haven't run in this session.
Run `acme <command> --help` for detailed flag information.
```

---

## Step 3: Embed the Assets

```go
// internal/skills/skills.go

package skills

import (
    "embed"
    "io/fs"
    "os"
    "path/filepath"
)

//go:embed assets/acme-cli
var embeddedAssets embed.FS

type SkillManifest struct {
    Name string
    FS   fs.FS
}

func Bundled() []SkillManifest {
    acmeCLI, _ := fs.Sub(embeddedAssets, "assets/acme-cli")
    return []SkillManifest{
        {Name: "acme-cli", FS: acmeCLI},
    }
}
```

The `evals/` directory lives at `internal/skills/assets/acme-cli/evals/` on disk
but is excluded from the embed glob. It is also listed in `.gitignore`:

```gitignore
internal/skills/assets/**/evals/
```

---

## Step 4: Implement `install` and `uninstall`

The install logic follows the canonical+symlinks model:

```go
// internal/skills/skills.go (continued)

func CanonicalDir(skillName string) (string, error) {
    base := os.Getenv("XDG_CONFIG_HOME")
    if base == "" {
        home, err := os.UserHomeDir()
        if err != nil {
            return "", err
        }
        base = filepath.Join(home, ".config")
    }
    return filepath.Join(base, "acme", "skills", skillName), nil
}

func InstallCanonical(skillFS fs.FS, canonical string, force bool) error {
    if _, err := os.Stat(canonical); err == nil && !force {
        return nil
    }
    if err := os.MkdirAll(canonical, 0o755); err != nil {
        return err
    }
    return fs.WalkDir(skillFS, ".", func(path string, d fs.DirEntry, err error) error {
        if err != nil {
            return err
        }
        dst := filepath.Join(canonical, filepath.FromSlash(path))
        if d.IsDir() {
            return os.MkdirAll(dst, 0o755)
        }
        data, _ := fs.ReadFile(skillFS, path)
        return os.WriteFile(dst, data, 0o644)
    })
}
```

`LinkAt` in `link_unix.go` creates a symlink. On Windows (`link_windows.go`), it
catches `syscall.Errno(1314)` and falls back to a directory copy, writing a
`.acme-skill-source` marker so `RemoveLinkAt` can verify ownership:

```go
// link_windows.go (abbreviated)

func LinkAt(canonical, parentDir string, force bool) (string, bool, error) {
    target := filepath.Join(parentDir, filepath.Base(canonical))
    if force { os.RemoveAll(target) }

    if err := os.Symlink(canonical, target); err == nil {
        return target, false, nil
    } else if linkErr, ok := err.(*os.LinkError); ok && linkErr.Err == errPrivilegeNotHeld {
        if copyErr := copyDir(canonical, target); copyErr != nil {
            return "", true, copyErr
        }
        _ = os.WriteFile(filepath.Join(target, ".acme-skill-source"), []byte(canonical), 0o644)
        return target, true, nil
    } else {
        return "", false, err
    }
}
```

---

## Step 5: Wire the CLI Commands

Register `skills` under the root command. In Cobra:

```go
// cmd/acme/main.go or internal/cli/root.go

root := &cobra.Command{Use: "acme", Short: "Deployment tool"}
root.AddCommand(newSkillsCommand())
root.SetHelpCommand(newHelpCommand(root))   // replaces default help with help + help all
```

`newSkillsCommand` returns a parent command with `install` and `uninstall`
subcommands wired to the functions above.

`newHelpCommand` adds `acme help all` under `acme help`:

```go
func newHelpCommand(root *cobra.Command) *cobra.Command {
    helpCmd := &cobra.Command{
        Use:   "help [command]",
        Short: "Help about any command",
        // ... default help behavior ...
    }
    helpCmd.AddCommand(newHelpAllCommand(root))
    return helpCmd
}
```

---

## Step 6: Add the Drift Test

```go
// internal/skills/drift_test.go

package skills_test

func TestSkillDrift(t *testing.T) {
    root := cli.NewRootCommand()    // the live Cobra tree

    var sb strings.Builder
    for _, m := range skills.Bundled() {
        fs.WalkDir(m.FS, ".", func(path string, d fs.DirEntry, _ error) error {
            if !d.IsDir() {
                data, _ := fs.ReadFile(m.FS, path)
                sb.Write(data)
            }
            return nil
        })
    }
    text := sb.String()

    var check func(*cobra.Command)
    check = func(cmd *cobra.Command) {
        if cmd.Hidden { return }
        tok := strings.Fields(cmd.Use)[0]
        if !strings.Contains(text, tok) {
            t.Errorf("command %q not found in skill assets", tok)
        }
        for _, sub := range cmd.Commands() { check(sub) }
    }
    for _, sub := range root.Commands() { check(sub) }
}
```

When a developer adds `acme logs` without updating `SKILL.md`, this test produces:

```
--- FAIL: TestSkillDrift
    drift_test.go:42: command "logs" not found in skill assets
```

---

## Step 7: Add the CLI Change Checklist

In `AGENTS.md`:

```markdown
## CLI Change Checklist

Any CLI surface change (new command, renamed command, flag or output shape change)
MUST update the bundled skill assets in the same PR. The drift test enforces this:
a PR adding a command without updating skill assets will fail CI.
```

---

## Step 8: Run Task Verification

Add tasks to `Taskfile.yml`:

```yaml
skills:check:
  desc: Verify skill drift and embed tests
  cmds:
    - go test ./internal/skills/... -run TestSkillDrift
    - go test ./internal/skills/... -run TestBundled

skills:install:dev:
  desc: Install bundled skill into .tmp/skills for local inspection
  cmds:
    - mkdir -p .tmp/skills
    - go run ./cmd/acme skills install --path .tmp/skills
```

---

## Outcome

After these 8 steps, `acme` ships with:

- `acme skills install [--user] [--project] [--path <dir>] [--force]`
- `acme skills uninstall`
- `acme help all` → JSON command tree
- Drift test in CI
- CLI Change Checklist in AGENTS.md
- Evals excluded from binary and git
