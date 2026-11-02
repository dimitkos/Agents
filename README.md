# Agents

Custom agents for GitHub Copilot.

| Agent | File | What it does |
|-------|------|--------------|
| `dotnet-lts-upgrade` | [.github/agents/dotnet-lts-upgrade.agent.md](.github/agents/dotnet-lts-upgrade.agent.md) | Upgrades a .NET solution to the latest LTS (e.g. 6 → 8, 8 → 10) on a new `upgrade-net<MAJOR>` branch: target frameworks, `global.json`, NuGet packages, breaking changes, pipeline YAML and Dockerfiles. Commits only after build and tests pass. |

## How to use an agent in a project

1. Copy the agent file into the project at the same path: `.github/agents/<name>.agent.md`.
2. **VS Code (Copilot Chat):** select the agent from the agent dropdown and give it a task,
   e.g. *"Upgrade this solution to the latest LTS"*.
3. **Copilot coding agent on GitHub:** choose the custom agent when assigning an issue or starting a task.

If your Copilot version only supports the older chat-mode format, rename the file to
`.github/chatmodes/<name>.chatmode.md`.
