I'm building a Go CLI with Cobra called `promptc`. I want AI agents like Copilot and Claude to know how to use it. Can you help me embed a skill into the binary so users can run `promptc skills install` to install structured knowledge for coding agents?

The CLI currently has these commands:
- `promptc run <template>` — render and execute a prompt template
- `promptc list` — list available templates
- `promptc add <name>` — add a new template
- `promptc config` — manage configuration
- `promptc version`

I'd like the skill to cover:
1. How to run templates
2. How to create and manage templates
3. Diagnosing when a template fails

Can you use the skill-bundler skill?
