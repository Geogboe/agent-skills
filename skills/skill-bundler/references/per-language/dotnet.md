# C#/.NET Recipe: Embedding and Installing Skills

This recipe covers .NET 8+ CLIs using `System.CommandLine`. Assets are embedded using
MSBuild `<EmbeddedResource>` items and accessed at runtime via
`Assembly.GetManifestResourceStream`.

---

## 1. Asset Tree Layout

```
src/
  Acme.Cli/
    Skills/
      SkillBundler.cs
      SkillManifest.cs
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
```

---

## 2. Embed Assets in the Project File

In `Acme.Cli.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>

  <!-- Embed all skill assets -->
  <ItemGroup>
    <EmbeddedResource Include="Skills\assets\**\*" />
  </ItemGroup>

  <!-- Explicitly exclude evals from embedding -->
  <ItemGroup>
    <EmbeddedResource Remove="Skills\assets\**\evals\**" />
    <None Remove="Skills\assets\**\evals\**" />
  </ItemGroup>

</Project>
```

**Why the `<None Remove>` line?** Without it, MSBuild may still include `evals/`
files under the default `None` item group and copy them to the output directory.
Both lines are required to fully exclude.

Also add to `.gitignore`:

```gitignore
# Skill eval fixtures — local only
src/Acme.Cli/Skills/assets/**/evals/
```

---

## 3. Access Embedded Resources at Runtime

.NET embeds resources with a resource name derived from the path:
`<AssemblyName>.<Namespace>.<FolderPath>.<FileName>` where directory separators
are replaced with `.`.

For `Skills\assets\acme-cli\SKILL.md` in assembly `Acme.Cli`, the resource name
is `Acme.Cli.Skills.assets.acme-cli.SKILL.md`.

In `Skills/SkillBundler.cs`:

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Reflection;

namespace Acme.Cli.Skills;

public record SkillManifest(string Name, string ResourcePrefix);

public static class SkillBundler
{
    private static readonly Assembly _asm = Assembly.GetExecutingAssembly();
    private static readonly string _asmName = _asm.GetName().Name!;

    /// <summary>Returns all skills bundled into this binary.</summary>
    public static IReadOnlyList<SkillManifest> Bundled() =>
    [
        new("acme-cli",   $"{_asmName}.Skills.assets.acme-cli"),
        new("acme-debug", $"{_asmName}.Skills.assets.acme-debug"),
    ];

    /// <summary>
    /// Enumerates all embedded resource names belonging to a skill, returning
    /// (relativeFilePath, stream) pairs.
    /// </summary>
    public static IEnumerable<(string RelPath, Stream Content)> GetSkillFiles(
        SkillManifest manifest)
    {
        var prefix = manifest.ResourcePrefix + ".";
        foreach (var name in _asm.GetManifestResourceNames())
        {
            if (!name.StartsWith(prefix, StringComparison.Ordinal))
                continue;
            var rel = name[prefix.Length..]
                .Replace('.', Path.DirectorySeparatorChar);
            // Restore file extension: last segment needs the dot back
            // e.g. "SKILL_md" → "SKILL.md"  (handle via a naming convention below)
            var stream = _asm.GetManifestResourceStream(name)!;
            yield return (rel, stream);
        }
    }
}
```

> **Naming caveat:** .NET replaces `.` in file names with `_` in the resource name
> to avoid ambiguity with the namespace separator. `SKILL.md` becomes `SKILL_md`
> in the resource name.
>
> **Recommended workaround:** Use a `LogicalName` attribute in the `.csproj` to
> preserve the exact resource name:
>
> ```xml
> <EmbeddedResource Include="Skills\assets\**\*"
>                   LogicalName="%(RecursiveDir)%(Filename)%(Extension)" />
> ```
>
> This sets the resource name to the relative path (e.g. `acme-cli/SKILL.md`),
> making extraction straightforward:
>
> ```csharp
> // With LogicalName approach — resource names are plain relative paths
> foreach (var name in _asm.GetManifestResourceNames())
> {
>     if (!name.StartsWith("acme-cli/")) continue;
>     var rel = name; // already a relative path
>     using var stream = _asm.GetManifestResourceStream(name)!;
>     // write to disk...
> }
> ```

---

## 4. Install to Canonical Directory

```csharp
using System.IO;
using System.Runtime.InteropServices;

public static class SkillInstaller
{
    private const string MarkerFileName = ".acme-skill-source";

    /// <summary>Returns ~/.config/acme/skills/<skillName>.</summary>
    public static string CanonicalDir(string toolName, string skillName)
    {
        var configHome = Environment.GetEnvironmentVariable("XDG_CONFIG_HOME")
            ?? Path.Combine(Environment.GetFolderPath(
                Environment.SpecialFolder.UserProfile), ".config");
        return Path.Combine(configHome, toolName, "skills", skillName);
    }

    /// <summary>Extracts embedded skill assets to the canonical directory.</summary>
    public static void InstallCanonical(SkillManifest manifest, bool force = false)
    {
        var dest = CanonicalDir("acme", manifest.Name);
        if (Directory.Exists(dest) && !force) return;

        Directory.CreateDirectory(dest);
        foreach (var (rel, stream) in SkillBundler.GetSkillFiles(manifest))
        {
            var target = Path.Combine(dest, rel);
            Directory.CreateDirectory(Path.GetDirectoryName(target)!);
            using var fs = File.Create(target);
            stream.CopyTo(fs);
        }
    }

    /// <summary>
    /// Creates a directory symlink (or managed copy fallback on Windows).
    /// Returns (targetPath, isCopy).
    /// </summary>
    public static (string Target, bool IsCopy) LinkAt(
        string canonical, string parentDir, bool force = false)
    {
        var target = Path.Combine(parentDir, Path.GetFileName(canonical));
        if (force && Directory.Exists(target))
            Directory.Delete(target, recursive: true);

        Directory.CreateDirectory(parentDir);

        try
        {
            Directory.CreateSymbolicLink(target, canonical);
            return (target, false);
        }
        catch (IOException ex) when (
            RuntimeInformation.IsOSPlatform(OSPlatform.Windows) &&
            ex.HResult == unchecked((int)0x80070522)) // ERROR_PRIVILEGE_NOT_HELD
        {
            // Copy fallback
            CopyDirectory(canonical, target);
            File.WriteAllText(Path.Combine(target, MarkerFileName), canonical);
            return (target, true);
        }
    }

    private static void CopyDirectory(string src, string dst)
    {
        Directory.CreateDirectory(dst);
        foreach (var file in Directory.GetFiles(src, "*", SearchOption.AllDirectories))
        {
            var rel = Path.GetRelativePath(src, file);
            var destFile = Path.Combine(dst, rel);
            Directory.CreateDirectory(Path.GetDirectoryName(destFile)!);
            File.Copy(file, destFile, overwrite: true);
        }
    }
}
```

---

## 5. System.CommandLine Install Command

```csharp
using System.CommandLine;

var skillsCmd = new Command("skills", "Manage bundled agent skills");
var installCmd = new Command("install", "Install bundled skills into agent directories");

var nameArg    = new Argument<string?>("skill-name", () => null, "Skill to install (default: all)");
var userOpt    = new Option<bool>("--user",    () => false, "Install into ~/.agents/skills/");
var projectOpt = new Option<bool>("--project", () => false, "Install into ./.agents/skills/");
var pathOpt    = new Option<string[]>("--path", "Additional target directory (repeatable)")
    { AllowMultipleArgumentsPerToken = false, Arity = ArgumentArity.ZeroOrMore };
var forceOpt   = new Option<bool>("--force",   () => false, "Overwrite existing installation");

installCmd.AddArgument(nameArg);
installCmd.AddOption(userOpt);
installCmd.AddOption(projectOpt);
installCmd.AddOption(pathOpt);
installCmd.AddOption(forceOpt);

installCmd.SetHandler((name, user, project, paths, force) =>
{
    var manifests = SkillBundler.Bundled();
    if (name is not null)
        manifests = manifests.Where(m => m.Name == name).ToList();

    foreach (var m in manifests)
    {
        SkillInstaller.InstallCanonical(m, force);
        var canonical = SkillInstaller.CanonicalDir("acme", m.Name);
        var parents = ResolveParents(user, project, paths);
        foreach (var p in parents)
        {
            var (target, isCopy) = SkillInstaller.LinkAt(canonical, p, force);
            Console.WriteLine($"{m.Name}: {(isCopy ? "copied" : "linked")} → {target}");
        }
    }
}, nameArg, userOpt, projectOpt, pathOpt, forceOpt);

skillsCmd.AddCommand(installCmd);

static IEnumerable<string> ResolveParents(bool user, bool project, string[] paths)
{
    if (!user && !project && paths.Length == 0) user = true;
    if (user)    yield return Path.Combine(Environment.GetFolderPath(
                     Environment.SpecialFolder.UserProfile), ".agents", "skills");
    if (project) yield return Path.Combine(".", ".agents", "skills");
    foreach (var p in paths) yield return p;
}
```

---

## 6. Tests (xUnit)

### Verify assets are embedded

```csharp
// Tests/Skills/SkillEmbedTests.cs
using Xunit;
using System.Reflection;

public class SkillEmbedTests
{
    [Theory]
    [InlineData("acme-cli")]
    [InlineData("acme-debug")]
    public void BundledSkill_HasSkillMd(string skillName)
    {
        var manifest = SkillBundler.Bundled().Single(m => m.Name == skillName);
        var files = SkillBundler.GetSkillFiles(manifest).Select(f => f.RelPath);
        Assert.Contains("SKILL.md", files);
    }

    [Theory]
    [InlineData("acme-cli")]
    [InlineData("acme-debug")]
    public void BundledSkill_DoesNotContainEvals(string skillName)
    {
        var manifest = SkillBundler.Bundled().Single(m => m.Name == skillName);
        var files = SkillBundler.GetSkillFiles(manifest).Select(f => f.RelPath);
        Assert.DoesNotContain(files, f => f.Contains("evals/") || f.Contains(@"evals\"));
    }
}
```

### Drift test → see [drift-and-help-all.md](../drift-and-help-all.md#dotnet)

---

## 7. See Also

- [Drift test and `help all` for .NET](../drift-and-help-all.md#dotnet)
- [Architecture: canonical + symlinks](../architecture.md)
