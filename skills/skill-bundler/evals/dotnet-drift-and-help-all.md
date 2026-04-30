I'm building a .NET 8 CLI (`infra`) using System.CommandLine. I want to bundle a skill so that Copilot and Codex can help users operate it. I need:

1. A `SkillDrift_AllNonHiddenCommandsAppearInSkillAssets` xUnit test that walks my System.CommandLine command tree
2. An `infra help all` command that outputs a JSON tree of all non-hidden commands
3. The SKILL.md to reference `infra help all` instead of duplicating command tables
4. The drift test to exclude any command annotated with `[SkipDriftCheck]`

My root command has: `deploy`, `destroy`, `plan`, `output`, `state`, `version`, and `debug` (hidden).

Can you apply the skill-bundler skill to set this up?
