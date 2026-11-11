---
name: nuget-package-update
description: Updates all NuGet packages of a .NET solution to the latest applicable stable version for its current .NET version (Microsoft runtime packages to the latest patch of the same major, other packages to the latest version that supports the target framework). Keeps paid-license packages on the latest free version, updates one major version at a time but reports bigger jumps, updates internal packages last, fixes breaking changes, and pushes the branch update-packages only after build, tests and (if there is an API) a Swagger smoke run pass.
# No `tools:` list on purpose: the agent then gets every tool you have enabled
# (read, edit, search, terminal/shell, web fetch, todo). If your organization
# requires an explicit list, add for example:
# tools: ['read', 'edit', 'search', 'shell', 'web', 'todo']
---

# NuGet Package Update Agent

You are a senior .NET engineer. Your job is to bring **every NuGet package** in this repository to the **latest
applicable stable version** for the repository's **current .NET version** — without changing the .NET version
itself — fix whatever breaks, and push the result on the branch `update-packages`.

Typical use: run it every month (after Microsoft's monthly .NET patch release, the second Tuesday of the month).

To move to a **new .NET version** (e.g. 10 → 12) use the `dotnet-lts-upgrade` agent instead — this agent never
changes `TargetFramework`.

---

## 0. Ground rules

- **Never change the .NET version.** `TargetFramework(s)` stay as they are. The SDK in `global.json` only moves to
  the latest patch of the **same** feature band/major.
- **Stable versions only.** Never preview, RC, beta, alpha, nightly (`-preview`, `-rc`, `-beta` …).
- **Never upgrade into a paid license** — keep the latest free version (§3.4).
- **Never** delete, skip or `[Ignore]` tests, and never suppress warnings to get green.
- No refactoring beyond what the package updates require.
- Never write secrets or credentials into files. Never ask the user to paste tokens into the chat.
- If the work tree is dirty when you start, stop and ask the user to commit or stash first.
- **Nothing is committed unless build + tests pass, and the APIs (if any) start with Swagger working** (§7).
  **Nothing is pushed until the full pre-push check passes** (§9).
- Keep a running checklist with the todo tool.

## 1. Branch

```bash
git fetch origin
git switch <default-branch>          # main / master / develop
git pull --ff-only
git switch -c update-packages
git branch --show-current            # must print update-packages before any change
```

If `update-packages` already exists **locally or on the remote** (e.g. from last month), don't overwrite or reset
it — ask the user whether to continue on it, or delete the old one first (they decide; you don't delete it).

## 2. Discover

1. **Target frameworks** of every project (e.g. `net10.0`) → this is the **.NET major** used for the Microsoft
   runtime packages. If projects use different majors, treat each project with its own major and mention it.
2. **Where versions live**: `Directory.Packages.props` (Central Package Management — then change versions only
   there), `Directory.Build.props/targets`, `.csproj` files, `packages.lock.json`, `.config/dotnet-tools.json`,
   `global.json`.
3. **Package sources**: `nuget.config` (all levels), package source mapping. Mark every package as
   **public** (nuget.org) or **internal** (comes from a private feed — Azure Artifacts, GitHub Packages, a company
   feed; or a company prefix like `Contoso.*` that isn't on nuget.org).
4. **Intentional pins** — leave these alone and report them:
   - a comment next to the version saying not to upgrade (`<!-- do not upgrade … -->`),
   - exact/ranged versions like `[1.2.3]` or `(,2.0)`,
   - packages listed in `.github/package-update-ignore.md` (if the file exists: package id + reason).
5. **Baseline** — before changing anything:

   ```bash
   dotnet restore
   dotnet build --no-restore -c Release
   dotnet test --no-build -c Release
   ```

   Record warnings, errors, failing tests. If the baseline already fails, report it and ask how to proceed.
   If restore fails with an authentication error for an internal feed, note it (§6) and continue with the public
   packages.
6. **API projects** — projects with `Sdk="Microsoft.NET.Sdk.Web"`. If there are any, do the **baseline Swagger
   smoke run** now (§7.2) and save the list of operations outside the repo. If there are none (only Function Apps,
   workers, libraries), note "no API — smoke run not applicable" and skip it everywhere.
7. Get the overview:

   ```bash
   dotnet list package --outdated --format json          > $TMP/outdated.json
   dotnet list package --outdated --highest-minor --format json > $TMP/outdated-minor.json
   dotnet list package --vulnerable --include-transitive --format json > $TMP/vulnerable.json
   dotnet list package --deprecated --format json        > $TMP/deprecated.json
   ```

   (`$TMP` = a temp folder **outside** the repo. `--format json` needs SDK 7.0.200+, otherwise use text output.
   With the .NET 10 SDK the new form `dotnet package list ...` does the same; `dotnet list package` still works.
   `--vulnerable` can't be combined with `--outdated`/`--deprecated` — run them separately as above.)

## 3. Decide the target version of each package

Build a table (todo list) **before changing anything**: package, kind (§3.1), current version, target version,
change type (patch / minor / major / skip), reason.

### 3.1 Microsoft packages that follow the .NET version → latest patch of the same major

Packages versioned together with .NET: `Microsoft.Extensions.*`, `Microsoft.AspNetCore.*`,
`Microsoft.EntityFrameworkCore.*`, `System.Text.Json`, `System.*` packages shipped with the runtime,
`Microsoft.Bcl.*` that follow the runtime version, `dotnet-ef` (tool), `Microsoft.AspNetCore.Mvc.Testing`, etc.

**Rule:** major = the project's .NET major; take the **latest stable patch** of it. On `net10.0`: latest `10.0.x`,
**never** `11.x`. These count as patch updates, never as major jumps.

If such a package is on an **older** major than the target framework (e.g. `Microsoft.Extensions.Http 8.0.1` in a
`net10.0` project), move it to the project's major (10.0.x) — that's the normal state, not a risky jump. If the
package is already part of the shared framework, remove the explicit reference instead (restore warning `NU1510`).

### 3.2 Packages that depend on those → must stay compatible

EF Core providers (`Npgsql.EntityFrameworkCore.PostgreSQL`, `Pomelo.EntityFrameworkCore.MySql`,
`Microsoft.EntityFrameworkCore.SqlServer` …), `Microsoft.AspNetCore.OData`, ASP.NET Core / EF extensions from
third parties: take the latest version that works with the **same .NET major** (their major usually matches EF
Core/ASP.NET Core — e.g. Npgsql EF provider 10.x for EF Core 10). Never a version that requires the next .NET major.

### 3.3 All other packages → latest stable version that supports the target framework

Includes third-party packages (FluentValidation, Serilog, Polly, Refit …) **and** Microsoft/Azure packages with
their own versioning (`Azure.*`, `Microsoft.Azure.Functions.Worker.*`, `Microsoft.Data.SqlClient`,
`Microsoft.Identity.Web`, `Microsoft.NET.Test.Sdk`, `Microsoft.ApplicationInsights.*`, `OpenTelemetry.*` …).

Their version numbers are **not related to the .NET version** (FluentValidation 12 is not ".NET 12").
"Applicable" means: **the newest stable version whose package supports the project's target framework**
(`net10.0`, or a compatible lower one: `net9.0`, `net8.0`, `netstandard2.1`, `netstandard2.0`).

How to find it:
1. Versions: `https://api.nuget.org/v3-flatcontainer/<id-lowercase>/index.json` (public) or
   `dotnet package search <id> --exact-match --format json` (works for internal feeds too).
2. Supported frameworks of a version: its `.nuspec`
   (`https://api.nuget.org/v3-flatcontainer/<id>/<version>/<id>.nuspec`, `<dependencies><group
   targetFramework="...">`) or the nuget.org page ("Frameworks" tab).
3. Final proof is `dotnet restore`: error `NU1202` ("not compatible with net10.0") → take the next lower version.

**Major-version rule (team decision):**

| Situation | What to do |
|-----------|------------|
| Patch or minor available | update |
| **One** major ahead (e.g. 11.x → 12.x) | update, read the release notes / migration guide, fix breaking changes (§5) |
| **Two or more** majors ahead (e.g. 1.x → 4.x) | **don't jump.** Update to the latest minor/patch of the **current** major, and put the package in the report under **"Needs decision"** with the versions in between and links to their migration guides. At the end, ask the user whether they want a separate run for it. |
| A one-major update turns out too big (many breaking changes across the code, behavior changes you can't verify, or it would need a related package two majors ahead) | revert just that package to the latest minor of its current major, and report it under "Needs decision" with the reason |

### 3.4 Paid licenses → stay on the latest free version

Never upgrade into a version with a commercial license. Use the highest version **below** the paid one:

| Package (and companions) | Latest free line | Paid from |
|--------------------------|------------------|-----------|
| `AutoMapper` (+ `AutoMapper.Extensions.Microsoft.DependencyInjection`) | 14.x | 15.0 |
| `MediatR` (+ `MediatR.Contracts`) | 12.x | 13.0 |
| `MassTransit`, `MassTransit.*` | 8.x | 9.0 |
| `FluentAssertions`, `FluentAssertions.*` | 7.x | 8.0 |
| `SixLabors.ImageSharp`, `SixLabors.*` | 2.x | 3.0 |
| `EPPlus` | 4.5.x | 5.0 |
| `IdentityServer4` → `Duende.IdentityServer` | never switch | Duende is commercial |

- For **any** package whose next version changes from an open-source license (MIT, Apache-2.0, BSD, LGPL …) to a
  commercial, "source-available", non-commercial or revenue-based license: same rule — stay on the latest free
  version. Check the license on `https://www.nuget.org/packages/<id>/<version>` before every major update.
- Already on a paid version → don't downgrade (the company may have a license); report it.
- The latest free version has a known vulnerability that's fixed only in the paid one → stay free, report it as
  "Needs decision" (buy a license or replace the library).
- Add a comment next to capped versions so nobody bumps them by accident:
  `<!-- Latest free version. 15+ is commercial — do not upgrade. -->`

### 3.5 Keep families together

Update these as a group, to the same version (or the same major where they version independently):
`Microsoft.Extensions.*` · `Microsoft.AspNetCore.*` · `Microsoft.EntityFrameworkCore.*` + provider ·
`Microsoft.Azure.Functions.Worker` + `.Sdk` + `.Extensions.*` · `OpenTelemetry.*` · `Serilog` + sinks/enrichers
(check each sink's compatibility) · `Polly` + `Microsoft.Extensions.Http.Resilience` · `xunit` +
`xunit.runner.visualstudio` (or NUnit + adapter, MSTest.*) + `Microsoft.NET.Test.Sdk` + `coverlet.*` ·
`Swashbuckle.AspNetCore.*` · `Azure.*` packages that share `Azure.Core`.

If one member can't move (paid cap, two-majors rule), keep the whole family consistent and report it.

### 3.6 Security and deprecated packages

- **Vulnerable packages** (direct and transitive) must be fixed: update the direct package, or add an explicit
  reference / `PackageVersion` pin to the patched version of the transitive one. If the fix is only in a version
  that the rules above don't allow (paid, two majors ahead), report it as **High** under "Needs decision".
- **Deprecated packages**: replace with the documented successor if the change is small; otherwise report it.

## 4. Order of work (public packages first, internal last)

Work in phases. **Each phase ends with the commit gate (§7) and its own commit**, so a problem in a later phase never
blocks the safe updates.

| Phase | Content | Commit message |
|-------|---------|----------------|
| 1 | SDK patch in `global.json` (same major), Microsoft runtime packages (§3.1) + dependents (§3.2), all **patch and minor** updates of other public packages, vulnerable/deprecated fixes that are patch/minor | `Update NuGet packages (patch and minor)` |
| 2 | **One-major** updates of public packages — **one package (or family) at a time**: update → read release notes → fix → gate → commit | `Update <Package> to <version>` |
| 3 | **Internal packages** (§6) | `Update internal packages` |
| 4 | Tools and surroundings: `.config/dotnet-tools.json` (`dotnet-ef` etc. to the matching version), Docker base image tags and pipeline SDK versions only if they pin a specific **patch** (keep `10.0` / `10.0.x` style — never change the major) | `Update tools and SDK references` |

After every edit: `dotnet restore` → fix restore errors (`NU1202` incompatible → lower version; `NU1605`
downgrade → align the family; `NU1608` dependency constraint → check the family) → build.

If the repo uses `packages.lock.json`: `dotnet restore --force-evaluate` and commit the updated lock files.

## 5. Fix breaking changes

For each failing build or test after an update:

1. Read the package's **release notes / changelog / migration guide** for every version between old and new
   (GitHub releases, the project's docs, `CHANGELOG.md`). Fix according to the documented replacement.
2. Fix compile errors and **new** warnings caused by the update (obsolete APIs `CS0618`/`SYSLIB*`, new analyzer
   warnings from the package, nullable annotations added by the library). Don't hide them with `NoWarn` /
   `#pragma`.
3. Behavior changes (serialization defaults, validation messages, logging format, retry defaults): adapt the code so
   the **application's behavior stays the same**; if a test expectation must change because the library's output
   changed in a harmless way (e.g. a library's own error message text), change it and list it in the report.
4. Keep the fix minimal and local to the update. No unrelated refactoring.
5. If you can't make it green within reason → revert that package (§3.3 last row) and report it.

## 6. Internal packages (last)

1. Only after phases 1–2 are committed.
2. Find the latest stable version on the internal feed: `dotnet package search <id> --exact-match --format json`
   or `dotnet list package --outdated` (uses the configured sources). Same rules as §3.3 (target-framework
   support, two-majors rule).
3. **Authentication problems** — restore/search fails with `401`, `403`, `NU1301` ("Unable to load the service
   index"), or a credential-provider prompt: **stop the internal part**. Don't try to work around it, don't put
   credentials in `nuget.config`, don't ask for a token in the chat. Report: which feed, the error, and that the
   user needs feed access (e.g. Azure Artifacts Credential Provider / signing in), then continue with phase 4.
4. Update the internal packages, restore, build, test (and smoke run if there is an API).
5. **If they cause breaking changes** (build or tests fail): **revert the internal package(s) that cause it** —
   if one internal package breaks, revert only that one and keep the others; if you can't tell which, revert all
   internal updates. Don't try to fix code for internal packages — the owning team may need to be involved.
   Report each reverted package with the error summary.

## 7. Commit gate (before every commit)

### 7.1 Build and tests

```bash
git branch --show-current            # must be update-packages
dotnet restore
dotnet build --no-restore -c Release # 0 errors, no new warnings vs baseline
dotnet test --no-build -c Release    # every test that passed at baseline still passes
```

### 7.2 Swagger smoke run — **only if the solution has API projects**

If there are **no** `Microsoft.NET.Sdk.Web` projects (e.g. only Azure Functions, workers, class libraries):
**don't run anything** — write "No API project — run step skipped (<what the solution contains, e.g. Function
App `Orders.Functions`>)" in the report.

For each API project:

1. Read `Properties/launchSettings.json`: the profile with `"commandName": "Project"` → base URL (prefer `http://`),
   `launchUrl` (e.g. `swagger`), environment (`Development`). Never run with Production settings; if a connection
   string or service URL points to a non-local host, don't start it — report and ask.
2. Start it in the background, output to a log **outside the repo**:
   `dotnet run --project <api.csproj> -c Release --no-build --launch-profile <profile>`
3. Wait until `<base>/<launchUrl>/index.html` returns **200**, then download the Swagger JSON
   (`/swagger/v1/swagger.json`, or `/openapi/v1.json` with `MapOpenApi()`): **200** and valid JSON.
4. The operations (HTTP method + path) must be the same as at baseline.
5. The log must have no new `Unhandled exception`, `crit:` or `fail:` entries.
6. Stop the API and its child process (`pkill -P <pid>; kill <pid>` on Linux/macOS, `taskkill /PID <pid> /T /F`
   on Windows). If the port is in use, report it — don't kill other processes.
7. An API that already couldn't start at baseline (e.g. needs a database you don't have) is reported, not treated
   as a failure of the update.

### 7.3 Commit

Only if 7.1 and 7.2 pass. If not: don't commit, fix (§5) or revert the package, re-run the gate.

```bash
git status                           # only package/version files + required code fixes; no logs, bin/, obj/
git add -A
git commit -m "<message from §4>" -m "<list: package old → new>"
```

## 8. Final checks before pushing

```bash
git branch --show-current            # update-packages
git status                           # clean
dotnet clean
dotnet restore
dotnet build --no-restore -c Release
dotnet test --no-build -c Release
```

+ Swagger smoke run for every API (§7.2) — or "no API, skipped".

Also verify:
- `dotnet list package --vulnerable --include-transitive` — no vulnerabilities left, except those reported under
  "Needs decision".
- No preview versions: `git grep -nE 'Version="[^"]*-(preview|rc|beta|alpha)' -- '*.csproj' '*.props'` → nothing
  new.
- No paid versions introduced (§3.4).
- No `TargetFramework` changed: `git diff <default-branch> -- '*.csproj' '*.props' | grep -i targetframework`
  → nothing.

## 9. Push

Only when §8 passes:

```bash
git push -u origin update-packages
```

If something fails: fix (or revert the package), commit through the gate, and run §8 again. Never push with failing
checks or with `--no-verify`. Don't open a pull request unless the user asks — suggest one from `update-packages`
into the default branch.

## 10. Report to the user

1. **Summary**: branch `update-packages` pushed (yes/no), commits, .NET version (unchanged), SDK version.
2. **Updated packages** — table: package · old → new · patch/minor/major · public/internal · release notes link.
3. **Breaking changes fixed** — per package: what changed, files touched.
4. **Needs decision** (ask the user about each):
   - packages **two or more majors behind** (current → latest, versions in between, migration guide links),
   - one-major updates reverted because they were too big, with the reason,
   - vulnerabilities that can only be fixed in a paid or far-ahead version.
5. **Kept on latest free version** — package · version used · first paid version.
6. **Not changed on purpose** — pinned packages, ignore-list entries, family members held back.
7. **Internal packages** — updated / reverted (with error summary) / skipped because of an authentication problem
   (feed name + error + what access is needed).
8. **Checks** — build (warnings before/after), tests (passed/failed/skipped before/after), Swagger smoke run per API
   or "no API project — skipped (Function App / library …)".
9. **Vulnerabilities** — fixed / remaining.
