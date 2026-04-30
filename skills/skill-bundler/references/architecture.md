# Architecture: Canonical + Symlinks

This document explains the full rationale and mechanics of the skill install model
used by this skill.

---

## Why Embed Skills in the Binary?

A bundled skill is embedded in the CLI binary at compile time. This means:

- **Always in sync** — the skill ships alongside the code it describes. Users can't
  install the tool and forget to install the skill.
- **Single artifact** — no separate package or repository to maintain.
- **Versionable** — the embedded skill version matches the binary version.

The tradeoff is that the binary is slightly larger, but skill assets (markdown files)
are small — typically a few KB total.

---

## Canonical + Symlinks Model

The core insight is: **separate the source of truth from the installation targets**.

```
Binary (embedded assets)
        │
        │  skills install  (extract)
        ▼
~/.config/<tool>/skills/<tool>-<purpose>/   ← CANONICAL (source of truth)
        │
        ├──── symlink ──────────────────▶  ~/.agents/skills/<tool>-<purpose>/
        │                                        (--user, default)
        ├──── symlink ──────────────────▶  .agents/skills/<tool>-<purpose>/
        │                                        (--project)
        └──── symlink ──────────────────▶  .claude/skills/<tool>-<purpose>/
                                                 (--path .claude/skills)
```

### Why a canonical directory?

Without it, you'd need to copy the skill assets N times — once per agent directory.
With it:

- Update the canonical once (`skills install --force`) and all agent dirs pick up the
  change immediately (symlinks resolve to the updated canonical).
- Uninstall is atomic: remove the links, then optionally remove the canonical.
- Users can inspect or modify the canonical without touching agent configuration.

---

## Scope Flags in Detail

| Flag | Target directory | When to use |
|---|---|---|
| `--user` (default) | `~/.agents/skills/<name>/` | Personal machine, all projects |
| `--project` | `./.agents/skills/<name>/` | Project-scoped, checked into repo |
| `--path <dir>` | `<dir>/<name>/` | Agent-specific dirs (`.claude/skills/`, `.github/skills/`, etc.) |

`--path` is **repeatable**: `--path .claude/skills --path .codex/skills` links to both.
All scope flags are **additive**: specifying `--user --project` links to both.

If no scope flag is given, `--user` is implied.

---

## Canonical Path on Each Platform

Use `~/.config/<tool>/skills/` on all three platforms:

| Platform | Recommended path |
|---|---|
| Linux | `~/.config/<tool>/skills/` |
| macOS | `~/.config/<tool>/skills/` |
| Windows | `~/.config/<tool>/skills/` (or `%USERPROFILE%\.config\<tool>\skills\`) |

**Why not `%LOCALAPPDATA%` on Windows or `~/Library/Application Support` on macOS?**
Uniformity wins for CLI tools used in cross-platform scripting. The canonical path
should be predictable in shell scripts and agent prompts without branching on OS.
Users who want native conventions can set `<TOOL>_SKILLS_HOME` to override.

Respect `XDG_CONFIG_HOME` on Linux and macOS if set (i.e. replace `~/.config` with
`$XDG_CONFIG_HOME`).

---

## Symlink Mechanics

On Unix (Linux/macOS), directory symlinks work without elevated privileges:

```
os.Symlink(canonicalPath, targetPath)
```

On Windows, creating directory symlinks requires either Developer Mode enabled or the
`SeCreateSymbolicLinkPrivilege` privilege (typically requires elevation). Strategy:

1. Try `os.Symlink` (or equivalent).
2. Catch `ERROR_PRIVILEGE_NOT_HELD` (Win32 error 1314 / `syscall.Errno(1314)` in Go).
3. Fall back to a recursive directory copy.
4. Write a marker file `.<tool>-skill-source` inside the copy:

   ```
   ~/.agents/skills/acme-cli/
     ├── SKILL.md
     ├── references/
     └── .acme-skill-source    ← contains: /Users/alice/.config/acme/skills/acme-cli
   ```

5. During uninstall, read the marker to verify ownership before removing.

### Uninstall safety check

```
func safeRemove(linkOrCopyPath, canonicalPath string) error {
    target, err := os.Readlink(linkOrCopyPath)     // symlink?
    if err == nil {
        // It's a symlink — only remove if it points to our canonical
        if filepath.Clean(target) != filepath.Clean(canonicalPath) {
            return fmt.Errorf("symlink points elsewhere; use --force to override")
        }
        return os.Remove(linkOrCopyPath)
    }
    // It's a copy — check marker
    marker := filepath.Join(linkOrCopyPath, ".acme-skill-source")
    src, err := os.ReadFile(marker)
    if err != nil || filepath.Clean(string(src)) != filepath.Clean(canonicalPath) {
        return fmt.Errorf("managed-copy marker missing or mismatched; use --force")
    }
    return os.RemoveAll(linkOrCopyPath)
}
```

---

## Versioning and Upgrade Detection

Embed a `version` field in the skill's `SKILL.md` frontmatter (or a sibling `VERSION`
plaintext file):

```yaml
---
name: acme-cli
version: "2.1.0"
description: ...
---
```

During `skills install`:

1. Extract the embedded version.
2. Read `.installed-version` from the existing canonical directory (if present).
3. If embedded version > installed version (semver compare), proceed with extraction
   and update `.installed-version`.
4. If embedded version == installed version, skip extraction unless `--force` is set.

Write `.installed-version` to the canonical directory after a successful install:

```
~/.config/acme/skills/acme-cli/
  ├── SKILL.md
  ├── references/
  └── .installed-version    ← "2.1.0"
```

---

## Multi-Skill Bundling

A CLI may ship multiple skills. Recommended internal structure:

```
internal/
  skills/
    assets/
      acme-cli/           ← end-user workflows
        SKILL.md
        references/
      acme-debug/         ← provider/debugging workflows
        SKILL.md
        references/
    skills.go             ← registry: []SkillManifest
```

`skills.go` exports a slice of manifests, each pointing at its sub-filesystem:

```go
var bundled = []SkillManifest{
    {Name: "acme-cli",   FS: acmeCLIFS},
    {Name: "acme-debug", FS: acmeDebugFS},
}
```

`skills install` with no positional argument installs all; with a name it installs one.

---

## Repo Hygiene: What NOT to Embed

Never embed:

| Path | Why |
|---|---|
| `evals/` | May contain local paths, credentials, or environment fixtures |
| `.env` / `*.local` | Secrets |
| Test fixture data | Binary blobs, large generated files |

Each language recipe shows the exact syntax to exclude these paths from the embed
manifest without excluding the surrounding directory.

---

## Summary of Moving Parts

| Component | Description |
|---|---|
| Asset tree | Markdown files compiled into the binary |
| `skills install` | Extracts canonical, creates links/copies |
| `skills uninstall` | Removes links/copies safely |
| `.installed-version` | Upgrade-detection marker in canonical dir |
| `.<tool>-skill-source` | Ownership marker in managed copies (Windows) |
| Drift test | CI guard: command tokens must appear in skill text |
| `<tool> help all` | Machine-readable command surface; skill points agents at it |
