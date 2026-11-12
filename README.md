# Agents

Custom agents for GitHub Copilot.

| Agent | File | What it does |
|-------|------|--------------|
| `dotnet-lts-upgrade` | [.github/agents/dotnet-lts-upgrade.agent.md](.github/agents/dotnet-lts-upgrade.agent.md) | Upgrades a .NET solution to the latest LTS (e.g. 6 → 8, 8 → 10) on a new `upgrade-net<MAJOR>` branch: target frameworks, `global.json`, NuGet packages, breaking changes, pipeline YAML and Dockerfiles. Never upgrades packages into paid licenses (AutoMapper, MediatR, MassTransit, FluentAssertions …) — keeps the latest free version. Commits only after build, tests and a local smoke run (API starts and Swagger loads with no runtime errors) pass. |
| `automapper-removal` | [.github/agents/automapper-removal.agent.md](.github/agents/automapper-removal.agent.md) | Removes AutoMapper (now commercial) and replaces it with injectable hand-written mappers named after the mapped type (`LandedCost` → `LandedCostMapper`) in a `Mappers` folder (existing or created in the right class library), on a `remove-automapper` branch. Moves profile logic into the mappers, makes mapped objects immutable (get-only properties + constructor), replaces `ProjectTo` with EF-translatable projections, proves same behavior with characterization tests, fixes tests. Commits only after build, tests and a Swagger smoke run pass; pushes only after a full pre-push check (build → tests → run the APIs). |
| `service-documentation` | [.github/agents/service-documentation.agent.md](.github/agents/service-documentation.agent.md) | Documents a service for a handover, for developers, testers, architects, managers and POs: every endpoint and Azure Function (trigger → input → steps → output → failures) in plain language and technical detail, background jobs, messaging, data model, config, security, pipelines, AKS/Azure deployment, with Mermaid diagrams. Writes a questions list for the previous team and a health report first. Writes Markdown locally into `/docs` only (no code changes, no publishing), commits on a local `service-docs` branch. Overview mode combines several services. **Needs the `service-docs` skill.** |
| `tech-debt-analysis` | [.github/agents/tech-debt-analysis.agent.md](.github/agents/tech-debt-analysis.agent.md) | **Report only.** Finds technical debt and improvements in a .NET repo across 13 categories (platform & packages, Azure Functions, async/performance, reliability, security, architecture, code quality, tests, observability, API design, containers/AKS, pipelines, modernisation). Uses real tools (stricter analyzers via command-line flags, package audit, coverage, git hotspots, secret scan) plus code review, rates each finding by severity and effort, and writes Markdown into `/tech-debt`: executive summary with scorecard and trend, findings per category, priority matrix, backlog items ready for Azure DevOps, hotspots. Never changes code; commits on a local `tech-debt-report` branch. |
| `nuget-package-update` | [.github/agents/nuget-package-update.agent.md](.github/agents/nuget-package-update.agent.md) | Monthly package update without changing the .NET version: Microsoft runtime packages to the latest patch of the same major (`net10.0` → latest `10.0.x`, never 11), other packages to the latest stable version that supports the target framework. Paid packages stay on the latest free version; one-major jumps are done one by one with breaking-change fixes, two or more majors behind are reported for a decision; internal packages last (reverted if they break, auth problems reported). Commits per phase after build + tests (+ Swagger smoke run only if an API exists), then pushes branch `update-packages`. |
| `bug-investigator` | [.github/agents/bug-investigator.agent.md](.github/agents/bug-investigator.agent.md) | Investigates a concrete production bug like a senior: reads the bug / user story from Azure DevOps (MCP), traces the code path, finds the commit running in prod, checks git history, reads production logs via the Azure MCP (read only — or writes the KQL for you), ranks hypotheses, reproduces the bug with a local unit/integration test (with `git bisect` for regressions), finds root cause and blast radius, then fixes it on a `bugfix/<name>` branch with the reproducing test as regression test. **Never calls anything in prod, never writes to any database, never runs the app with real settings — tests only.** Pushes only when asked; writes an investigation report with a ready PR description. |

## Skills

| Skill | Folder | What it does |
|-------|--------|--------------|
| `service-docs` | [.github/skills/service-docs](.github/skills/service-docs) | Team templates and rules for documentation pages: endpoints, Azure Functions, background jobs, service overview, handover questions, health report, `service-summary.json`; "In short" sections for POs/managers; Mermaid conventions. Used by `service-documentation`, and usable on its own ("document this endpoint in our format"). |

## How to use an agent in a project

1. Copy the agent file into the project at the same path: `.github/agents/<name>.agent.md`.
   If the agent needs a skill, copy the whole skill folder too: `.github/skills/<skill-name>/`.
2. **VS Code (Copilot Chat):** select the agent from the agent dropdown and give it a task,
   e.g. *"Upgrade this solution to the latest LTS"*.
3. **Copilot coding agent on GitHub:** choose the custom agent when assigning an issue or starting a task.

If your Copilot version only supports the older chat-mode format, rename the file to
`.github/chatmodes/<name>.chatmode.md`.
