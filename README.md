# Android AI Agent Skills

Reusable skills, workflows, engineering standards, templates, and checklists for Android development with AI coding assistants.

This repository is a **documentation and workflow library**. It helps a developer give an assistant focused context, review its changes, and validate the resulting Android implementation.

## Who this is for

- Android developers working with Kotlin, XML UI, or Jetpack Compose.
- Teams who want consistent planning, implementation, code review, and release checks.
- Developers building repeatable AI-assisted workflows around an existing codebase.

## Quick start


1. Clone or download the repository.
2. Read [Getting started](docs/getting-started.md) and [How to use skills](docs/how-to-use-skills.md).
3. Select the skill that matches your current task.
4. Add the relevant project context, standard, template, and checklist to your assistant workflow.
5. Review the proposed change and run the checks required by the Android project.



```sh
git clone https://github.com/Muhammad-muaz-dev/android-ai-agent-skills.git
cd android-ai-agent-skills
```

Each numbered directory under `skills/` contains its domain-specific `SKILL.md`. Copy or install the relevant skill using the conventions supported by your assistant; exact installation paths vary by tool.

## Skill catalog

| Domain | Skill directory |
| --- | --- |
| Product requirements and scope | [01-product-planning](skills/01-product-planning) |
| Application structure and responsibilities | [02-android-architecture](skills/02-android-architecture) |
| View/XML interfaces | [03-xml-ui-design](skills/03-xml-ui-design) |
| Compose interfaces | [04-jetpack-compose](skills/04-jetpack-compose) |
| Screen navigation and back behavior | [05-navigation](skills/05-navigation) |
| Networking and APIs | [06-networking-api](skills/06-networking-api) |
| Authentication flows | [07-authentication](skills/07-authentication) |
| Local persistence | [08-local-database](skills/08-local-database) |
| Firebase integrations | [09-firebase](skills/09-firebase) |
| Android permissions | [10-permissions](skills/10-permissions) |
| Media downloads | [11-media-downloads](skills/11-media-downloads) |
| Security review | [12-security](skills/12-security) |
| Performance investigation | [13-performance](skills/13-performance) |
| Test planning and coverage | [14-testing](skills/14-testing) |
| Bug diagnosis | [15-debugging](skills/15-debugging) |
| Refactoring | [16-refactoring](skills/16-refactoring) |
| Reviewing changes | [17-code-review](skills/17-code-review) |
| Release preparation | [18-play-store-release](skills/18-play-store-release) |

## Repository map

| Directory | Purpose |
| --- | --- |
| [skills/](skills) | Focused instructions for individual Android engineering domains |
| [workflows/](workflows) | Repeatable sequences for planning and delivering work |
| [standards/](standards) | Engineering conventions to apply to project changes |
| [templates/](templates) | Starting points for repeatable outputs |
| [checklists/](checklists) | Review and validation prompts |
| [examples/](examples) | Examples to adapt to a real project |
| [docs/](docs) | Getting started and workflow guidance |

## Example usage

For a Compose screen, start with the Compose, navigation, and testing skills. Supply the actual screen requirements, design references, existing ViewModels, and navigation conventions. Ask for a scoped implementation with loading, empty, error, and success states, then review the diff and run the project's relevant UI/build checks.

For a networking change, combine networking/API, local-database, and testing guidance. Supply the API contract and existing data layer, identify retry/error expectations, and verify the result against controlled responses.

For a bug, start with debugging. Provide reproduction steps, expected behavior, and relevant logs. Diagnose the cause before changing code, then use a focused regression check to verify the fix.

These are suggested combinations; read each skill before using it and adapt the workflow to the project's architecture.

## Recommended development cycle

1. Define the user-visible problem and acceptance criteria.
2. Inspect existing code and project conventions.
3. Plan a small, reviewable change.
4. Implement with the relevant domain guidance.
5. Run appropriate build, test, lint, and device checks.
6. Review the diff and document the result.

See [Agent workflow](docs/agent-workflow.md) and [Production-grade rules](docs/production-grade-rules.md) for supporting guidance.

## Validation and maintenance

This library has no Android app module to build. Validate documentation by checking referenced paths, Markdown rendering, examples, and consistency between related skills. When changing instructions, try them on a representative Android task and inspect the resulting code and validation evidence.

A checklist or assistant response is guidance; it does not establish that an app is tested, secure, or ready for release. Run the checks in the target project.

## Contributing

Read [CONTRIBUTING.md](CONTRIBUTING.md). Keep changes focused, state the problem the guidance solves, include examples when useful, and update [CHANGELOG.md](CHANGELOG.md) for meaningful changes. Preserve the numbered domain organization and avoid contradictory instructions across skills and standards.

## License

See [LICENSE](LICENSE) for the repository's MIT license terms.

## Maintainer

[Muhammad Muaz](https://github.com/Muhammad-muaz-dev) — Android developer working with Kotlin, Jetpack Compose, and practical app engineering workflows.
