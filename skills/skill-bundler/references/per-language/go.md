# Go Recipe: Embedding and Installing Skills

This recipe covers Go + Cobra CLIs. All snippets use the standard library (`embed`,
`io/fs`, `os`, `path/filepath`) plus Cobra for the command tree.

---

## 1. Asset Tree Layout

```
internal/
  skills/
    assets/
      acme-cli/
        SKILL.md
        VERSION
        references/
          bootstrap.md
          diagnose.md
      acme-debug/
        SKILL.md
        VERSION
        references/
    skills.go
    link_unix.go
    link_windows.go
    skills_test.go
    drift_test.go
```

---

## 2. Embed the Assets

In `internal/skills/skills.go`:

```go
package skills

import (
    "embed"
    "io/fs"
)

//go:embed assets/acme-cli assets/acme-debug
var embeddedAssets embed.FS

// SkillManifest describes one bundled skill.
type SkillManifest struct {
    Name string
    FS   fs.FS
}

// Bundled returns all skills compiled into this binary.
func Bundled() []SkillManifest {
    acmeCLI, _ := fs.Sub(embeddedAssets, "assets/acme-cli")
    acmeDebug, _ := fs.Sub(embeddedAssets, "assets/acme-debug")
    return []SkillManifest{
        {Name: "acme-cli",   FS: acmeCLI},
        {Name: "acme-debug", FS: acmeDebug},
    }
}
```

**Excluding evals from the embed directive:**
The `//go:embed` glob applies only to the listed paths. To keep an `evals/` folder
in your source tree but out of the binary, do **not** include it in the glob:

```go
// Only embeds assets/acme-cli and assets/acme-debug — NOT assets/acme-cli/evals
//go:embed assets/acme-cli assets/acme-debug
var embeddedAssets embed.FS
```

If you use a wildcard for the whole `assets/` directory, exclude explicitly:

```go
//go:embed assets
//go:embed !assets/acme-cli/evals !assets/acme-debug/evals
var embeddedAssets embed.FS
```

Also add to `.gitignore`:

```gitignore
# Skill eval fixtures — local only, never embedded
internal/skills/assets/**/evals/
```

---

## 3. Canonical Install and Symlink

In `internal/skills/skills.go`:

```go
import (
    "io/fs"
    "os"
    "path/filepath"
)

// CanonicalDir returns ~/.config/<tool>/skills/<name>.
func CanonicalDir(toolName, skillName string) (string, error) {
    base := os.Getenv("XDG_CONFIG_HOME")
    if base == "" {
        home, err := os.UserHomeDir()
        if err != nil {
            return "", err
        }
        base = filepath.Join(home, ".config")
    }
    return filepath.Join(base, toolName, "skills", skillName), nil
}

// InstallCanonical extracts the embedded FS into the canonical directory.
func InstallCanonical(skillFS fs.FS, canonical string, force bool) error {
    if _, err := os.Stat(canonical); err == nil && !force {
        return nil // already installed
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
        data, err := fs.ReadFile(skillFS, path)
        if err != nil {
            return err
        }
        return os.WriteFile(dst, data, 0o644)
    })
}
```

Platform-split linking lives in `link_unix.go` and `link_windows.go`:

```go
// link_unix.go
//go:build !windows

package skills

import "os"

// LinkAt creates a directory symlink from target → canonical.
// Returns (targetPath, isCopy, error).
func LinkAt(canonical, parentDir string, force bool) (string, bool, error) {
    target := filepath.Join(parentDir, filepath.Base(canonical))
    if force {
        os.RemoveAll(target)
    }
    return target, false, os.Symlink(canonical, target)
}
```

```go
// link_windows.go
//go:build windows

package skills

import (
    "os"
    "path/filepath"
    "syscall"
)

const errPrivilegeNotHeld = syscall.Errno(1314)

func LinkAt(canonical, parentDir string, force bool) (string, bool, error) {
    target := filepath.Join(parentDir, filepath.Base(canonical))
    if force {
        os.RemoveAll(target)
    }
    err := os.Symlink(canonical, target)
    if err == nil {
        return target, false, nil
    }
    // Fall back to copy on privilege error
    if linkErr, ok := err.(*os.LinkError); ok {
        if linkErr.Err == errPrivilegeNotHeld {
            if copyErr := copyDir(canonical, target); copyErr != nil {
                return "", true, copyErr
            }
            // Write ownership marker
            marker := filepath.Join(target, ".acme-skill-source")
            _ = os.WriteFile(marker, []byte(canonical), 0o644)
            return target, true, nil
        }
    }
    return "", false, err
}
```

---

## 4. The `skills install` Cobra Command

```go
func newSkillsInstallCmd() *cobra.Command {
    var (
        userFlag    bool
        projectFlag bool
        pathFlags   []string
        forceFlag   bool
    )
    cmd := &cobra.Command{
        Use:   "install [skill-name]",
        Short: "Install bundled skills into agent directories",
        RunE: func(cmd *cobra.Command, args []string) error {
            manifests := skills.Bundled()
            if len(args) == 1 {
                // filter to named skill
            }
            for _, m := range manifests {
                canonical, _ := skills.CanonicalDir("acme", m.Name)
                if err := skills.InstallCanonical(m.FS, canonical, forceFlag); err != nil {
                    return err
                }
                parents := resolveTargetParents(userFlag, projectFlag, pathFlags)
                for _, p := range parents {
                    os.MkdirAll(p, 0o755)
                    skills.LinkAt(canonical, p, forceFlag)
                }
                fmt.Fprintf(cmd.OutOrStdout(), "installed %s → %s\n", m.Name, canonical)
            }
            return nil
        },
    }
    cmd.Flags().BoolVar(&userFlag,    "user",    false, "Install into ~/.agents/skills/ (default)")
    cmd.Flags().BoolVar(&projectFlag, "project", false, "Install into ./.agents/skills/")
    cmd.Flags().StringArrayVar(&pathFlags, "path", nil, "Install into additional directory (repeatable)")
    cmd.Flags().BoolVar(&forceFlag,   "force",   false, "Overwrite existing installation")
    return cmd
}

func resolveTargetParents(user, project bool, paths []string) []string {
    // If no explicit flags, default to --user
    if !user && !project && len(paths) == 0 {
        user = true
    }
    var out []string
    if user {
        home, _ := os.UserHomeDir()
        out = append(out, filepath.Join(home, ".agents", "skills"))
    }
    if project {
        out = append(out, filepath.Join(".", ".agents", "skills"))
    }
    out = append(out, paths...)
    return out
}
```

---

## 5. Verify the Embed (Unit Test)

In `internal/skills/skills_test.go`:

```go
func TestBundledSkillsHaveSKILLMD(t *testing.T) {
    for _, m := range skills.Bundled() {
        t.Run(m.Name, func(t *testing.T) {
            _, err := fs.Stat(m.FS, "SKILL.md")
            if err != nil {
                t.Fatalf("skill %q missing SKILL.md: %v", m.Name, err)
            }
        })
    }
}

func TestEvalsNotEmbedded(t *testing.T) {
    for _, m := range skills.Bundled() {
        t.Run(m.Name, func(t *testing.T) {
            _, err := fs.Stat(m.FS, "evals")
            if err == nil {
                t.Fatalf("skill %q: evals/ must not be embedded in the binary", m.Name)
            }
        })
    }
}
```

---

## 6. See Also

- [Drift test and `help all` for Go](../drift-and-help-all.md#go)
- [Architecture: canonical + symlinks](../architecture.md)
