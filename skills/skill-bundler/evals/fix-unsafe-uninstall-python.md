I have a Python CLI (`envctl`) built with click. It already ships a `SKILL.md` at the repo root, but users have to manually copy it into their agent directories. The uninstall flow is also broken — it deletes the directory even if the user has modified files.

I want to:
1. Bundle the skill properly using `importlib.resources`
2. Add a `envctl skills install` command with `--user` / `--project` / `--path` flags
3. Fix uninstall to check the `.envctl-skill-source` marker before deleting
4. Add a `test_skill_drift` pytest test

Can you apply the skill-bundler skill to my Python project?
