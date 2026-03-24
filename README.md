# agent-skills

This repository is a home for reusable [Agent Skills](https://agentskills.io/home): open, portable skill packages that give coding agents task-specific instructions, references, and optional scripts.

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

## Why This Repo Exists

- Keep custom skills under version control.
- Make them easy to install and reuse.
- Share repeatable workflows in a tool-agnostic format.
- Maintain one repository instead of duplicating long prompt instructions across projects.

## Contributing

If you are updating or adding a skill, keep the detailed documentation with that skill itself so the root README can stay focused on what this repository is for and how to install it.
