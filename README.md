# Agents

Custom agents for GitHub Copilot.

| Agent | File | What it does |
|-------|------|--------------|
| `dotnet-lts-upgrade` | [.github/agents/dotnet-lts-upgrade.agent.md](.github/agents/dotnet-lts-upgrade.agent.md) | Upgrades a .NET solution to the latest LTS (e.g. 6 → 8, 8 → 10) on a new `upgrade-net<MAJOR>` branch: target frameworks, `global.json`, NuGet packages, breaking changes, pipeline YAML and Dockerfiles. Never upgrades packages into paid licenses (AutoMapper, MediatR, MassTransit, FluentAssertions …) — keeps the latest free version. Commits only after build, tests and a local smoke run (API starts and Swagger loads with no runtime errors) pass. |
| `automapper-removal` | [.github/agents/automapper-removal.agent.md](.github/agents/automapper-removal.agent.md) | Removes AutoMapper (now commercial) and replaces it with injectable hand-written mappers named after the mapped type (`LandedCost` → `LandedCostMapper`) in a `Mappers` folder (existing or created in the right class library), on a `remove-automapper` branch. Moves profile logic into the mappers, makes mapped objects immutable (get-only properties + constructor), replaces `ProjectTo` with EF-translatable projections, proves same behavior with characterization tests, fixes tests. Commits only after build, tests and a Swagger smoke run pass; pushes only after a full pre-push check (build → tests → run the APIs). |
| `service-documentation` | [.github/agents/service-documentation.agent.md](.github/agents/service-documentation.agent.md) | Documents a service for a handover, for developers, testers, architects, managers and POs: every endpoint and Azure Function (trigger → input → steps → output → failures) in plain language and technical detail, background jobs, messaging, data model, config, security, pipelines, AKS/Azure deployment, with Mermaid diagrams. Writes a questions list for the previous team and a health report first. Writes Markdown locally into `/docs` only (no code changes, no publishing), commits on a local `service-docs` branch. Overview mode combines several services. **Needs the `service-docs` skill.** |

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
