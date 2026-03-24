# agentic-tools

This repository is a home for reusable [Agent Skills](https://agentskills.io/home): open, portable skill packages that give coding agents task-specific instructions and references.

The point of this repo is to keep those skills versioned, maintainable, and easy to install into compatible tooling without rewriting the same workflow guidance in prompts over and over.

## What Agent Skills Are

Agent Skills are an open standard for packaging agent expertise as files and folders an agent can discover and load on demand.

- Overview: <https://agentskills.io/home>
- Specification: <https://agentskills.io/specification>
- Open standard repository: <https://github.com/agentskills/agentskills>

If your agent or editor supports the Agent Skills format, the same skill can be reused across tools instead of being locked to one product.

## Install

Install this repository as a skill source with:

```bash
npx skills add https://github.com/Geogboe/agent-skills
```

After that, use your agent's normal skill discovery flow to browse or activate the installed skills.

## Repo Layout

This root README only explains the repository itself.

- Each skill should document its own purpose and usage in its own directory.
- The source of truth for a skill is the files stored in this repo.
- Skill-specific instructions live with the skill, not in this root README.

## Documentation Structure

Each skill directory should use this structure:

- SKILL.md: primary guidance, decision flow, and examples.
- references/: optional deep dives for complex workflows.
- evals/evals.json: optional acceptance-style eval scenarios for that skill.

Generated eval run artifacts belong under `.evals/*-workspace/` and are intentionally not committed.

## Why This Repo Exists

- Keep custom skills under version control.
- Make them easy to install and reuse.
- Share repeatable workflows in a tool-agnostic format.
- Maintain one repository instead of duplicating long prompt instructions across projects.

## Contributing

If you are updating or adding a skill, keep the detailed documentation with that skill itself so the root README can stay focused on what this repository is for and how to install it.

### Contribution Acceptance Rules

To keep this repository safe and reviewable, contributions are rejected if they include any of the following:

- Scripts of any kind (shell, PowerShell, Python, JS, batch, or other executable script files).
- Insecure commands or instructions (for example: disabling TLS validation, downloading and executing untrusted code without verification, or credential exposure patterns).
- Hidden or deceptive Unicode text (including zero-width/invisible characters used to obfuscate content).

Contributions should remain documentation-first, explicit, and security-conscious.
