# Drift Tests and `help all`

Two complementary mechanisms keep your bundled skill accurate without duplicating
CLI documentation:

1. **Drift test** — CI fails if you add a command without updating skill assets.
2. **`<tool> help all`** — agents call this at runtime instead of reading static
   command tables from the skill.

---

## `<tool> help all`

### Purpose

Your SKILL.md must not contain a full command synopsis — it will drift the moment
you rename a flag. Instead, implement `<tool> help all` (or `<tool> help --machine`)
that prints the live command tree.

### Output Shape (JSON)

```json
{
  "commands": [
    {
      "name": "sandbox",
      "short": "Manage sandboxes",
      "commands": [
        {
          "name": "create",
          "short": "Create a new sandbox",
          "flags": [
            { "name": "file",    "short": "f", "type": "string", "usage": "Spec file" },
            { "name": "no-wait", "short": "",  "type": "bool",   "usage": "Return immediately" }
          ]
        }
      ]
    }
  ]
}
```

### Reference in SKILL.md

Add a **Command Discovery** section near the top of your bundled SKILL.md:

```markdown
## Command Discovery

Do not guess command syntax.

Run `<tool> help all` to see the full current command surface before using any
command you have not run before.
Run `<tool> <command> --help` for detailed flag information on a specific command.
```

---

## Drift Test

### Convention: Uniform Test Names

| Language | Test name |
|---|---|
| Go | `TestSkillDrift` |
| Python | `test_skill_drift` |
| C#/.NET | `SkillDrift_AllNonHiddenCommandsAppearInSkillAssets` |

Using a consistent name lets CI scripts and code search find the drift test quickly
across any language.

### Escape Hatch

Not every command should be asserted. Mark commands as exempt:

| Language | How |
|---|---|
| Go + Cobra | `cmd.Hidden = true` — automatically excluded |
| Go + Cobra | Comment `// skill:exempt` on the command var — parse in test |
| Python click | `@click.command(hidden=True)` |
| .NET System.CommandLine | Annotate with `[SkipDriftCheck]` attribute; test reflects on this |

---

## Go Drift Test

```go
// internal/skills/drift_test.go
package skills_test

import (
    "io/fs"
    "strings"
    "testing"

    "github.com/spf13/cobra"

    "github.com/example/acme/internal/cli"
    acmeskills "github.com/example/acme/internal/skills"
)

func TestSkillDrift(t *testing.T) {
    root := cli.NewRootCommand()
    tokens := collectCommandTokens(root)

    // Concatenate all embedded skill text
    var sb strings.Builder
    for _, manifest := range acmeskills.Bundled() {
        _ = fs.WalkDir(manifest.FS, ".", func(path string, d fs.DirEntry, err error) error {
            if err != nil || d.IsDir() {
                return err
            }
            data, _ := fs.ReadFile(manifest.FS, path)
            sb.Write(data)
            return nil
        })
    }
    skillText := sb.String()

    for _, tok := range tokens {
        if !strings.Contains(skillText, tok) {
            t.Errorf("command token %q not found in any bundled skill asset", tok)
        }
    }
}

// collectCommandTokens walks the Cobra tree and returns the Use (first word)
// of every non-hidden command.
func collectCommandTokens(root *cobra.Command) []string {
    var tokens []string
    var walk func(*cobra.Command)
    walk = func(cmd *cobra.Command) {
        if cmd.Hidden {
            return
        }
        use := strings.Fields(cmd.Use)
        if len(use) > 0 {
            tokens = append(tokens, use[0])
        }
        for _, sub := range cmd.Commands() {
            walk(sub)
        }
    }
    for _, sub := range root.Commands() {
        walk(sub)
    }
    return tokens
}
```

### Go `help all` Implementation

```go
// internal/cli/help_all.go
package cli

import (
    "encoding/json"
    "fmt"

    "github.com/spf13/cobra"
)

type commandNode struct {
    Name     string        `json:"name"`
    Short    string        `json:"short"`
    Flags    []flagNode    `json:"flags,omitempty"`
    Commands []commandNode `json:"commands,omitempty"`
}

type flagNode struct {
    Name  string `json:"name"`
    Short string `json:"short,omitempty"`
    Type  string `json:"type"`
    Usage string `json:"usage"`
}

func buildTree(cmd *cobra.Command) commandNode {
    node := commandNode{
        Name:  strings.Fields(cmd.Use)[0],
        Short: cmd.Short,
    }
    cmd.Flags().VisitAll(func(f *pflag.Flag) {
        node.Flags = append(node.Flags, flagNode{
            Name:  f.Name,
            Short: f.Shorthand,
            Type:  f.Value.Type(),
            Usage: f.Usage,
        })
    })
    for _, sub := range cmd.Commands() {
        if !sub.Hidden {
            node.Commands = append(node.Commands, buildTree(sub))
        }
    }
    return node
}

func newHelpAllCommand(root *cobra.Command) *cobra.Command {
    return &cobra.Command{
        Use:    "all",
        Short:  "Print machine-readable command tree (JSON)",
        Hidden: false,
        RunE: func(cmd *cobra.Command, args []string) error {
            tree := map[string]any{"commands": func() []commandNode {
                var out []commandNode
                for _, sub := range root.Commands() {
                    if !sub.Hidden {
                        out = append(out, buildTree(sub))
                    }
                }
                return out
            }()}
            enc := json.NewEncoder(cmd.OutOrStdout())
            enc.SetIndent("", "  ")
            return enc.Encode(tree)
        },
    }
}
```

---

## Python Drift Test {#python}

```python
# tests/skills/test_skill_drift.py
import click
import pytest
from importlib.resources import files
from acme.cli.root import cli_root        # your top-level click group
from acme.skills.skills import bundled


def _collect_command_tokens(group: click.Group) -> list[str]:
    tokens = []
    for name, cmd in group.commands.items():
        if getattr(cmd, "hidden", False):
            continue
        tokens.append(name)
        if isinstance(cmd, click.Group):
            tokens.extend(_collect_command_tokens(cmd))
    return tokens


def _all_skill_text() -> str:
    parts = []
    for manifest in bundled():
        root = files("acme.skills").joinpath(f"assets/{manifest.name}")
        for item in root.rglob("*"):
            if item.is_file():
                try:
                    parts.append(item.read_text(encoding="utf-8"))
                except Exception:
                    pass
    return "\n".join(parts)


def test_skill_drift():
    tokens = _collect_command_tokens(cli_root)
    skill_text = _all_skill_text()
    missing = [t for t in tokens if t not in skill_text]
    assert not missing, (
        f"Command tokens not found in any skill asset: {missing}\n"
        "Update the bundled skill or mark the command hidden=True."
    )
```

### Python `help all`

```python
@cli_root.command("all", hidden=False)
@click.pass_context
def help_all(ctx):
    """Print machine-readable command tree (JSON)."""
    import json

    def build_tree(group):
        out = {}
        for name, cmd in group.commands.items():
            if getattr(cmd, "hidden", False):
                continue
            entry = {"short": cmd.help or ""}
            if isinstance(cmd, click.Group):
                entry["commands"] = build_tree(cmd)
            params = [
                {"name": p.name, "type": p.type.name, "help": p.help or ""}
                for p in cmd.params if isinstance(p, click.Option)
            ]
            if params:
                entry["flags"] = params
            out[name] = entry
        return out

    click.echo(json.dumps({"commands": build_tree(cli_root)}, indent=2))
```

---

## C#/.NET Drift Test {#dotnet}

```csharp
// Tests/Skills/SkillDriftTests.cs
using System.CommandLine;
using System.Reflection;
using Xunit;

public class SkillDriftTests
{
    [Fact]
    public void SkillDrift_AllNonHiddenCommandsAppearInSkillAssets()
    {
        var root = CommandFactory.CreateRoot(); // your System.CommandLine root
        var tokens = CollectTokens(root);
        var skillText = GetAllSkillText();

        var missing = tokens.Where(t => !skillText.Contains(t)).ToList();
        Assert.True(missing.Count == 0,
            $"Command tokens missing from skill assets: {string.Join(", ", missing)}\n" +
            "Update the bundled skill or annotate the command with [SkipDriftCheck].");
    }

    private static List<string> CollectTokens(Command cmd)
    {
        var tokens = new List<string>();
        foreach (var sub in cmd.Subcommands)
        {
            if (sub.GetType().GetCustomAttribute<SkipDriftCheckAttribute>() is not null)
                continue;
            if (sub.IsHidden) continue;
            tokens.Add(sub.Name);
            tokens.AddRange(CollectTokens(sub));
        }
        return tokens;
    }

    private static string GetAllSkillText()
    {
        var sb = new System.Text.StringBuilder();
        foreach (var manifest in SkillBundler.Bundled())
        {
            foreach (var (_, stream) in SkillBundler.GetSkillFiles(manifest))
            {
                using var reader = new StreamReader(stream);
                sb.AppendLine(reader.ReadToEnd());
            }
        }
        return sb.ToString();
    }
}

/// <summary>Marks a command as exempt from the skill drift test.</summary>
[AttributeUsage(AttributeTargets.Class)]
public sealed class SkipDriftCheckAttribute : Attribute { }
```

### .NET `help all`

```csharp
var helpCmd    = new Command("help",    "Help and documentation");
var helpAllCmd = new Command("all",     "Print machine-readable command tree (JSON)");

helpAllCmd.SetHandler(() =>
{
    var root = CommandFactory.CreateRoot();
    var tree = BuildTree(root);
    Console.WriteLine(JsonSerializer.Serialize(
        new { commands = tree },
        new JsonSerializerOptions { WriteIndented = true }));
});

static List<object> BuildTree(Command cmd)
{
    var result = new List<object>();
    foreach (var sub in cmd.Subcommands)
    {
        if (sub.IsHidden) continue;
        result.Add(new
        {
            name     = sub.Name,
            @short   = sub.Description ?? "",
            flags    = sub.Options.Select(o => new {
                         name  = o.Name,
                         type  = o.ValueType.Name,
                         usage = o.Description ?? ""
                       }).ToList(),
            commands = BuildTree(sub),
        });
    }
    return result;
}
```
