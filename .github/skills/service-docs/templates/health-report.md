# Health report — <Service name>

What we inherit: risks and technical debt found in the code. Facts with sources — nothing has been changed.

## Summary

| Area | Status | Main finding |
|------|--------|--------------|
| .NET version | <OK / Warning / Risk> | <net6.0 — out of support since Nov 2024> |
| Packages | <…> | <3 vulnerable, AutoMapper 15 (paid)> |
| Tests | <…> | <2 test projects, ~140 tests, no tests for Functions> |
| Secrets in code | <…> | <1 connection string in appsettings.Development.json> |
| Observability | <…> | <Application Insights yes, no health checks> |
| Infrastructure as code | <…> | <none for the Function App> |
| Resilience | <…> | <no retry on Rates API calls> |

## Findings

| # | Severity | Area | Finding | Location | Suggested action |
|---|----------|------|---------|----------|------------------|
| 1 | High | .NET | Projects target `net6.0` (out of support) | `src/*/*.csproj` | Upgrade → `dotnet-lts-upgrade` agent |
| 2 | <High / Medium / Low> | <…> | <…> | `<file:line>` (never the secret value) | <…> |

## Numbers

- Projects: <n> · Endpoints: <n> · Functions: <n> · Background jobs/consumers: <n>
- Test projects: <n> · Tests: ~<n> · Integration tests: <yes/no>
- `TODO`/`FIXME`/`HACK`: <n>
