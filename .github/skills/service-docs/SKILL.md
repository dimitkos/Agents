---
name: service-docs
description: Team templates and rules for documenting .NET services — page structure for endpoints, Azure Functions, background jobs, service overview, handover questions and health report; plain-language "In short" sections for POs/managers; Mermaid diagram conventions. Use whenever you write or update documentation in /docs, or when asked to explain an endpoint, function or flow "in our format".
---

# Service documentation — templates and rules

Used by the `service-documentation` agent, and by anyone asking Copilot for a doc page by hand.

## Audience: every page works for five readers

| Section | For | Rules |
|---------|-----|-------|
| **In short** | PO, manager | 1–3 sentences, business words, no class/method names, no abbreviations without explanation. Answer: *what does it do and why does it matter?* |
| **How it works** | developer, tester, architect | numbered steps in words, each with a source ref; diagram |
| **Technical details** | developer, architect | classes, config keys (names only), retries, scaling |
| **Testing notes** | tester | example request/message (made-up data), edge cases, expected errors, how to trigger |
| **Open questions** | everyone | what the code doesn't tell; also copied to `handover/questions.md` |

## Writing rules

- Present tense, active voice: "The function loads the order", not "The order is loaded".
- Explain *why* when the code shows it ("retries 5 times because the Rates API is often slow").
- One fact = one source ref: `` `src/Api/Controllers/OrdersController.cs:42` ``.
- Mark conclusions you drew indirectly with **(inferred)**. Don't invent business meaning — ask instead.
- Names only for config and secrets: `ServiceBus:Connection`, never its value.
- Examples use made-up data: `"orderId": "PO-1001"`, `"customer": "Contoso Ltd"`, emails `user@example.com`.
- Tables for lists of things (endpoints, keys, messages); prose for behavior.
- Use the exact names from the code for routes, topics, queues, functions, classes — in backticks.
- Time: say the time zone. Schedules: always cron **and** words ("`0 0 2 * * *` — every day at 02:00 UTC").

## Diagrams (Mermaid)

- Write diagrams in standard Markdown fences:
  ````
  ```mermaid
  sequenceDiagram
    ...
  ```
  ````
  They render in VS Code (Markdown preview with a Mermaid extension) and GitHub.
- **Later, for the Azure DevOps wiki** the syntax is different: `::: mermaid` … `:::` instead of the fences, and
  the wiki's Mermaid version is older. To stay compatible, use only: `flowchart`/`graph`, `sequenceDiagram`,
  `erDiagram`, `classDiagram`, `stateDiagram-v2`. No `C4Context`, `mindmap`, `timeline`, `block-beta`, no icons,
  no `%%{init}%%` themes.
- Keep diagrams small: max ~15 nodes. Split big ones (one per flow) instead of one giant picture.
- Node IDs: simple (`api`, `sb`, `db`); labels in quotes when they contain spaces or symbols:
  `api["Orders API (AKS)"]`. Never use `end` as a node ID.
- Shapes: services/functions `["..."]`, databases `[("...")]`, queues/topics `[["..."]]` or
  `>"topic: purchase-orders"]`, external systems `{{"..."}}`.
- Group by hosting with subgraphs: `subgraph aks["AKS"]`, `subgraph azure["Azure"]`.
- Sequence diagrams: participants in call order; show the failure path with `alt`/`opt`
  (e.g. `alt Rates API unavailable`).
- Under every diagram one sentence that says what it shows.

## Templates

| Page | Template |
|------|----------|
| `docs/README.md` (service overview) | [templates/service-overview.md](templates/service-overview.md) |
| `docs/endpoints/<group>.md` | [templates/endpoint-group.md](templates/endpoint-group.md) |
| `docs/functions/<Name>.md` and `docs/background-jobs/<Name>.md` | [templates/function.md](templates/function.md) |
| `docs/handover/questions.md` | [templates/questions.md](templates/questions.md) |
| `docs/handover/health-report.md` | [templates/health-report.md](templates/health-report.md) |
| `docs/service-summary.json` | [templates/service-summary.schema.json](templates/service-summary.schema.json) |

Placeholders are in `<angle brackets>`. Remove sections that don't apply (e.g. no request body for a GET), but keep
the section order.
