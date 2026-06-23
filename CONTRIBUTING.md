# Contributing

## Guidelines

- Keep each skill focused on one responsibility.
- Put reusable agent instructions in `skills/<name>/SKILL.md`.
- Put process guidance in `workflows/`.
- Put engineering rules in `standards/`.
- Put reusable document formats in `templates/`.
- Put verification lists in `checklists/`.
- Prefer concise, actionable language over long explanations.

## Skill Format

Each `SKILL.md` should include YAML frontmatter:

```yaml
---
name: skill-name
description: Clear trigger description for when an agent should use this skill.
---
```

## Pull Requests

Include a short summary, changed folders, and validation performed.