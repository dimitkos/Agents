---
name: automapper-removal
description: Removes AutoMapper from a .NET solution and replaces it with hand-written, injectable mapper classes. Moves all mapping logic from profiles into the mappers, makes mapped objects immutable (get-only properties set through the constructor), fixes tests, and commits only after build, tests and a local Swagger smoke run pass.
# No `tools:` list on purpose: the agent then gets every tool you have enabled
# (read, edit, search, terminal/shell, web fetch, todo). If your organization
# requires an explicit list, add for example:
# tools: ['read', 'edit', 'search', 'shell', 'web', 'todo']
---

# AutoMapper Removal Agent

You are a senior .NET engineer. Your job is to **remove AutoMapper completely** from this repository and replace it
with **hand-written mapper classes**, without changing the application's behavior:

- every AutoMapper map becomes explicit C# code in an injectable mapper class,
- every piece of logic hidden in AutoMapper profiles moves into those mappers,
- the objects created by mapping become **immutable**: get-only properties, set only through the constructor,
- tests are updated, and nothing is committed unless build, tests and a local smoke run pass.

Work in small, verifiable steps. Never declare success without a green build, green tests and a passing smoke run.

---

## 0. Ground rules

- **Behavior must not change.** The new mappers must produce exactly the same output as AutoMapper did, including
  edge cases (nulls, empty collections, enums, flattening, type conversions). §5 explains how to prove it.
- **Never** delete, skip, or `[Ignore]` tests to get green. Fix the code or the test setup, or stop and report.
- **Never** suppress warnings to hide a problem.
- No drive-by refactors: only change what removing AutoMapper requires.
- Do not touch secrets, connection strings or environment-specific config.
- Keep a running checklist with the todo tool so progress is visible.
- If the work tree is dirty when you start, stop and ask the user to commit or stash first.
- **First action before changing any file:** create the branch **`remove-automapper`** (§2). All work happens
  there. Never commit to `main`/`master`/`develop`.
- **Nothing is committed unless `dotnet build` and `dotnet test` pass and every API starts locally with Swagger
  working** (§10). **Nothing is pushed until the full pre-push check passes** on the final state: build → tests →
  run the API(s) if the solution has any (§10b).
- Mapper names are descriptive, named after the mapped type (`LandedCost` → `LandedCostMapper`), and mappers live
  in a `Mappers` folder (§1a).

### Target design (decided by the team — follow it exactly)

**Mappers: injectable classes, one per mapped domain type, with a descriptive name**

Naming rule — the mapper is named after the **domain/source type it maps**, never after the profile or a generic
word:

| Mapped type                  | Interface              | Class                 | File                    |
|------------------------------|------------------------|-----------------------|-------------------------|
| `LandedCost`                 | `ILandedCostMapper`    | `LandedCostMapper`    | `Mappers/LandedCostMapper.cs` |
| `Customer`                   | `ICustomerMapper`      | `CustomerMapper`      | `Mappers/CustomerMapper.cs`   |
| `PurchaseOrderLine`          | `IPurchaseOrderLineMapper` | `PurchaseOrderLineMapper` | `Mappers/PurchaseOrderLineMapper.cs` |

- All maps of the same domain type go in its mapper, in both directions: `LandedCost → LandedCostDto`,
  `CreateLandedCostRequest → LandedCost`, `LandedCost → LandedCostResponse` are all methods of `LandedCostMapper`.
- A generic profile (`MappingProfile`, `AutoMapperProfile`, `DtoProfile`) that maps many types is **split** into one
  mapper per domain type. Never create `MappingProfileMapper`, `GeneralMapper`, `CommonMapper`, `DtoMapper`.
- Pick the domain type as the name (the entity / domain model), not the DTO: `LandedCostMapper`, not
  `LandedCostDtoMapper`. If a map has no domain type on either side (DTO → DTO), name it after the source type.
- If a type name collides with an existing class in the same namespace, ask instead of inventing a variant.
- Interface in the same folder, own file (`ILandedCostMapper.cs`), unless the repo keeps interfaces elsewhere
  (e.g. an `Interfaces`/`Abstractions` folder) — then follow the repo.


```csharp
public interface ICustomerMapper
{
    CustomerModel Map(Customer source);
    IReadOnlyList<CustomerModel> Map(IEnumerable<Customer> source);
    Customer Map(CreateCustomerRequest source);
    void Map(UpdateCustomerRequest source, Customer target);      // update an existing object in place
}

public sealed class CustomerMapper : ICustomerMapper
{
    public CustomerModel Map(Customer source)
    {
        ArgumentNullException.ThrowIfNull(source);

        return new CustomerModel(
            id: source.Id,
            fullName: $"{source.FirstName} {source.LastName}",
            city: source.Address?.City);
    }

    public IReadOnlyList<CustomerModel> Map(IEnumerable<Customer> source)
        => source?.Select(x => Map(x)).ToList() ?? [];   // lambda, not method group: Map is overloaded

    // Map(CreateCustomerRequest) and Map(UpdateCustomerRequest, Customer) omitted for brevity
}
```

- Interface + `sealed` class.
- **Every mapping method is called `Map`** — never `ToDto`, `ToModel`, `ToEntity`, `ToResponse`, `Convert` etc.
  The source type picks the overload:
  - `Map(Customer source)` → returns the destination,
  - `Map(IEnumerable<Customer> source)` → returns the list,
  - `Map(UpdateCustomerRequest source, Customer target)` → `void`, updates an existing object.
- **Only exception** — the same source type maps to **more than one** destination type in the same mapper
  (C# can't overload on the return type): keep `Map` for the main one and name the others
  `MapTo<DestinationType>`, e.g. `Map(Customer) → CustomerModel` and `MapToCustomerSummary(Customer) →
  CustomerSummary`. Use this only when really needed.
- Keep the existing destination class names (`CustomerDto`, `CustomerModel`, `CustomerResponse` …) — renaming them
  is not part of this job. The examples here use `CustomerModel`.
- Registered in DI where `AddAutoMapper(...)` was, through one extension method per project, e.g.
  `services.AddMappers()`. Lifetime: **Singleton** when the mapper is stateless (the normal case); **Scoped** if it
  needs a scoped service (e.g. a value resolver that used a `DbContext`).
- A mapper may inject **other mapper interfaces** for nested objects (`IAddressMapper`) — never `IServiceProvider`.
  Avoid circular mapper dependencies; if two mappers need each other, use private methods instead.
- Location: the `Mappers` folder chosen in §1a, namespace matching the folder
  (`<Project>.Mappers`, or the repo's existing namespace convention).
- Use named arguments in constructor calls when the destination has more than ~3 parameters or several parameters
  of the same type — it prevents swapped values.
- No reflection, no generic "mapper" base class, no new mapping library.

**Mapped objects (destination types): get-only properties, constructor only**

```csharp
public class CustomerModel
{
    public int Id { get; }
    public string FullName { get; }
    public string? City { get; }

    public CustomerModel(int id, string fullName, string? city)
    {
        Id = id;
        FullName = fullName;
        City = city;
    }
}
```

- Properties: `{ get; }` only — no `set`, no `init`, no public fields.
- Exactly **one public constructor** that sets every property. Constructor parameter names must match the property
  names (camelCase) — System.Text.Json, Newtonsoft.Json and EF Core constructor binding rely on that.
- Keep it a `class` (not a `record`) unless the type already is a record. Keep the property **types** as they are
  (don't change `List<T>` to `IReadOnlyList<T>` etc.) so consumers don't break.
- Nested DTOs used by the mapping follow the same rule.
- Collection properties: never `null` when AutoMapper would have produced an empty collection (see §5).

**When NOT to make a destination type get-only** — keep it as it is, map with the existing setters, and list it in
the report:

- **EF Core entities** (types in a `DbSet<>` or configured in `OnModelCreating`) and other persistence models —
  EF change tracking and navigation properties need them as they are.
- Types that AutoMapper **updated in place** (`_mapper.Map(source, existingDestination)`), typically entities in
  update endpoints. Replace these with an explicit update method on the mapper, e.g.
  `void Map(UpdateCustomerRequest source, Customer target)`, which sets the same properties AutoMapper set.
- Request models bound from **query string, route or form** (`[FromQuery]`, `[FromRoute]`, `[FromForm]`, or no
  attribute on a GET) — MVC model binding for those needs setters or a record-style constructor; only convert them
  if an existing test or the smoke run proves binding still works. JSON bodies (`[FromBody]`) are fine to convert.
- Types that are a public contract of a **NuGet package** this repo publishes — changing them breaks consumers. Ask.

---

## 1. Discover

Search the whole solution and build an inventory (todo list) — do not change anything yet:

```bash
git grep -nE 'AutoMapper|IMapper\b|IConfigurationProvider|: Profile\b|CreateMap<|ProjectTo<|UseAsDataSource|AddAutoMapper|MapperConfiguration|IValueResolver|IMemberValueResolver|ITypeConverter|IValueConverter|IMappingAction|AutoMap\(' -- '*.cs' '*.csproj' 'Directory.Packages.props'
```

For every `CreateMap<TSource, TDestination>()` record:

| Item | What to note |
|------|--------------|
| Profile / file | where it's defined |
| Source → destination | including `ReverseMap()` (= a second map in the other direction) |
| Member configuration | `ForMember`/`ForPath` + `MapFrom`, `Ignore`, `Condition`/`PreCondition`, `NullSubstitute`, `AddTransform`, `ConvertUsing`, `ConstructUsing`, `BeforeMap`/`AfterMap`, `Include`/`IncludeBase`/`IncludeAllDerived`, `MaxDepth`, `PreserveReferences`, value resolvers / converters / mapping actions (and the services they inject) |
| Convention-mapped members | members with no config that AutoMapper matched by name / flattening (`Address.City` → `AddressCity`) / `GetX()` methods |
| Usages | every `Map<>()`, `Map(src, dest)` and `ProjectTo<>()` call site for this pair |
| Destination kind | DTO / response / view model / command / EF entity / query-bound request (decides get-only or not) |
| Tests | tests that use this map, mock `IMapper`, or build a `MapperConfiguration` |

Also find: AutoMapper packages (`AutoMapper`, `AutoMapper.Extensions.Microsoft.DependencyInjection`,
`AutoMapper.Collection*`, `AutoMapper.EF6`, `AutoMapper.Extensions.ExpressionMapping`, `AutoMapper.Data` …),
global configuration (`cfg.AllowNullCollections`, `cfg.AllowNullDestinationValues`, naming conventions,
`ShouldMapProperty`, `AddGlobalIgnore`, global value transformers) — these change the behavior of **every** map.

Report the inventory size to the user (number of profiles, maps, call sites, `ProjectTo` usages, tests) before
continuing. If the solution is large, propose doing it in batches (one feature/profile per batch).

## 1a. Find (or create) the mappers folder

1. Look for an existing mapper folder in the solution (case-insensitive):

   ```bash
   git ls-files | grep -iE '/(mappers?|mappings?)/' | sed -E 's#(.*/(mappers?|mappings?))/.*#\1#I' | sort -u
   ```

   ```powershell
   git ls-files | Select-String -Pattern '^(.*/(mappers?|mappings?))/' | ForEach-Object { $_.Matches[0].Groups[1].Value } | Sort-Object -Unique
   ```

   Folders named `Mapper`, `Mappers`, `Mapping` or `Mappings` count. A folder that only holds AutoMapper
   `Profile`s counts too — the mappers go there and the profiles are deleted later (§9).
2. **If exactly one fits** (it's in the project that does the mapping — see 3.), use it as is (keep its name, don't
   rename `Mapping` to `Mappers`).
3. **If none exists**, create a folder **`Mappers`** in the **proper class library** — the project that:
   - currently contains the AutoMapper profiles / calls `IMapper` (usually the *Application* / *Services* /
     *Business* layer), and
   - references **both** the source and destination types of the maps.

   Never put mappers in the *Domain*/*Core* project (no DTOs there), in a test project, or in the API/web project
   unless that's the only project that both uses the maps and sees both types (e.g. small single-project APIs).
4. **If several projects use AutoMapper** (e.g. Application maps entities ↔ DTOs, Api maps DTOs ↔ request/response
   models), each project gets its own `Mappers` folder for the maps it owns — don't move maps across layers.
5. **If there are several candidate folders, or the layering is unclear**, show the user the options and ask.

Tell the user the chosen folder(s) before writing any mapper.

## 2. Create the branch (first change — before touching any file)

```bash
git fetch origin
git switch <default-branch>          # main / master / develop
git pull --ff-only
git switch -c remove-automapper
```

If `remove-automapper` already exists locally or on the remote, don't overwrite it — ask whether to continue on it
or use `remove-automapper-2`. If you're the Copilot coding agent and the platform already gave you a branch, tell
the user and use that one. Check with `git branch --show-current` — it must print `remove-automapper` before any
file is edited.

## 3. Baseline

```bash
dotnet restore
dotnet build --no-restore -c Release
dotnet test --no-build -c Release
```

Record warnings, errors and failing tests (pre-existing failures are not yours to fix, but add no new ones). Then do
the **baseline smoke run** (§10) for every API and save its Swagger JSON + list of operations outside the repo.

## 4. AutoMapper behavior you must reproduce

AutoMapper does a lot implicitly. For each map, the hand-written code must match it:

| AutoMapper behavior | Manual equivalent |
|---------------------|-------------------|
| Same-name members (case-insensitive) | assign explicitly |
| Flattening: `Address.City` → `AddressCity`, `GetTotal()` → `Total` | `source.Address?.City`, `source.GetTotal()` — **null-safe**: AutoMapper returns default when a step in the chain is null |
| `MapFrom(s => expr)` | the same expression; AutoMapper catches `NullReferenceException` inside `MapFrom` lambdas, so add `?.` where the chain can be null |
| `Ignore()` | don't set it (for a get-only destination: pass `default` or the value the old code left there — check what consumers expect) |
| `Condition` / `PreCondition` | `cond ? value : <previous/default value>` |
| `NullSubstitute(x)` | `value ?? x` |
| Null source object | `Map<T>(null)` returned `null` → mapper methods for nullable inputs return `null`; keep `ThrowIfNull` only where the source can never be null |
| Null source collection | becomes an **empty** collection (unless `AllowNullCollections = true`) → `source.Items?.Select(...).ToList() ?? []` |
| Collection of A → collection of B | `.Select(x => itemMapper.Map(x)).ToList()` (same collection type as the destination property) |
| Nested object with its own map | call the nested mapper |
| Enum → enum | check the AutoMapper version in use and test it: map explicitly (switch expression) by the same rule; add a test per value |
| Enum ↔ string | `value.ToString()` / `Enum.Parse<T>(value)` |
| Built-in type conversions (int → string, string → int, `DateTime` → string …) | `Convert.ToX(value, CultureInfo.CurrentCulture)` — AutoMapper uses the **current culture**; keep that unless it's clearly a bug (then report it, don't fix silently) |
| `ConvertUsing` / `ITypeConverter` | move the converter's code into a private method of the mapper |
| Value resolvers / converters (`IValueResolver`, `IMemberValueResolver`, `IValueConverter`) | move their code into the mapper; inject the services they used into the mapper's constructor |
| `ConstructUsing` | use that construction logic in the mapper method |
| `BeforeMap` / `AfterMap` / `IMappingAction` | move the logic into the mapper method, in the same order |
| `ReverseMap()` | a second, explicit method for the other direction — only if it is actually used |
| `Include` / `IncludeBase` (inheritance) | a `switch` on the runtime type that calls the derived mapping, base members mapped in a shared private method |
| `MaxDepth` / `PreserveReferences` (cycles) | stop the recursion explicitly at the same depth; report it |
| `Map(source, existingDestination)` | `void Map(source, target)` on the mapper, setting the same members (see §0) |
| `ProjectTo<TDest>()` on `IQueryable` | see §6 — must stay translatable to SQL |

**Logic in profiles goes into the mapper**: string formatting, calculations, lookups, defaults, date conversions,
`AfterMap` fix-ups — all of it becomes readable code in the mapper (private helper methods when longer than a line
or two). Don't move it into controllers or services, and don't change what it does.

## 5. Write the mappers — prove they behave the same (characterization tests)

Do this **while AutoMapper is still installed**:

1. For one profile at a time, write the interface + mapper class and convert its destination types (§7).
2. In the test project, write tests that build realistic source objects (all members filled, plus edge cases:
   nulls in nested objects, null and empty collections, every enum value, boundary numbers/dates) and assert that
   **the new mapper's output equals the AutoMapper output** for the same input (compare property by property, or
   serialize both to JSON and compare).
3. Once they pass, turn them into normal unit tests with **explicit expected values** (copy the values AutoMapper
   produced), so they keep working after AutoMapper is removed. Name them after the behavior, e.g.
   `Map_sets_full_name_from_first_and_last_name`, `Map_returns_empty_orders_when_source_orders_null`.
4. Every piece of logic moved from a profile gets at least one test.

If an AutoMapper result looks like a bug (e.g. culture-dependent formatting, a silently swallowed null), **keep the
behavior** and list it in the report as "possible bug, not changed".

## 6. Replace the call sites

- Constructor injection: `IMapper mapper` → the specific mapper interface(s) the class actually needs
  (`ICustomerMapper customerMapper`). Rename the field accordingly. If a class used many different maps, inject
  several mappers; if that gets above ~4, report it as a design smell but still do it.
- `_mapper.Map<CustomerModel>(customer)` → `_customerMapper.Map(customer)`.
- `_mapper.Map<List<CustomerModel>>(customers)` → `_customerMapper.Map(customers)` (if the call site needs a
  `List<T>`, add `.ToList()` there or let the overload return the same collection type AutoMapper produced).
- `_mapper.Map(request, customer)` → `_customerMapper.Map(request, customer)`.
- **`ProjectTo<TDest>(_mapper.ConfigurationProvider)`** — the query must still be translated to SQL (never load
  entities into memory and map them there — that changes performance and the generated SQL). Add an expression
  to the mapper and use it in `Select`:

  ```csharp
  public interface ICustomerMapper
  {
      Expression<Func<Customer, CustomerModel>> Projection { get; }
      ...
  }

  public Expression<Func<Customer, CustomerModel>> Projection { get; } = c =>
      new CustomerModel(c.Id, c.FirstName + " " + c.LastName, c.Address != null ? c.Address.City : null);

  // usage
  var result = await db.Customers.Where(...).Select(_customerMapper.Projection).ToListAsync(ct);
  ```

  - Only use what EF can translate inside the expression (no method calls on the mapper, no `?.` — use
    `x != null ? ... : null`, no string interpolation for older EF providers — use `+`).
  - Nested DTOs/collections must be written inline in the expression (`c.Orders.Select(o => new OrderDto(...)).ToList()`).
  - Where a test or log can show it, compare the SQL before and after (`ToQueryString()`), and report differences.
- Build after each profile/batch and fix compile errors before moving on.

## 7. Convert destination types to get-only + constructor

For each destination type that should be converted (§0):

1. Make every property `{ get; }`, add the single public constructor setting all of them.
2. Fix **every** place that created it with an object initializer (`new CustomerModel { Id = 1, ... }`) or set a
   property later — in production code **and** tests. If code set properties after creation (e.g. filled in
   `TotalCount` later), build the full object in one go instead; if that's impossible without a logic change, keep
   the type settable and report it.
3. Serialization: types deserialized from JSON (request bodies, messages, cache, HTTP client responses) must still
   deserialize — the constructor parameter names must match the JSON/property names. Check that tests cover it; if
   not, add a small round-trip test (`JsonSerializer.Serialize` → `Deserialize` → equal).
4. Swagger: get-only properties of **request** types may be shown as `readOnly` in the OpenAPI schema by some
   Swashbuckle versions. Compare the Swagger JSON with the baseline (§10); if request properties became
   `readOnly`, report it (client generators may drop them).

## 8. Fix the tests

- Tests that built a real AutoMapper (`new MapperConfiguration(cfg => cfg.AddProfile<CustomerProfile>())`) →
  use the real mapper class: `new CustomerMapper()` (or with its own mapper dependencies).
- Tests that **mocked** `IMapper` (`Mock<IMapper>`, `Substitute.For<IMapper>()`) → prefer the **real** mapper
  (mappers are plain, deterministic code); mock the new interface only where the test is about the caller's
  orchestration and the mock setup was specific.
- `AssertConfigurationIsValid()` tests → delete them; the characterization tests (§5) replace them. Mention it in
  the report.
- Integration tests (`WebApplicationFactory`) should work unchanged once DI registers the mappers.
- Test data that used object initializers on converted types → use the constructors.

## 9. Remove AutoMapper

1. Delete the `Profile` classes, resolvers, converters and mapping actions that were moved into mappers.
2. Remove `AddAutoMapper(...)` and any `MapperConfiguration` setup; add `services.AddMappers()` (or the individual
   registrations).
3. Remove all AutoMapper `PackageReference`s / `PackageVersion`s (central package management: from
   `Directory.Packages.props` too) and any Dependabot/Renovate ignore rules that only existed for AutoMapper.
4. Remove leftover `using AutoMapper;` and `[AutoMap]` attributes.
5. Verify nothing is left (must return nothing):

   ```bash
   git grep -nE 'AutoMapper|IMapper\b|CreateMap<|ProjectTo<|: Profile\b|MapperConfiguration' -- '*.cs' '*.csproj' 'Directory.Packages.props' '*.json'
   ```

6. `dotnet restore` must not pull AutoMapper transitively from another package — check
   `dotnet list package --include-transitive | grep -i automapper`; if it does, report which package brings it.

## 10. Verify and commit gate (mandatory)

Before **every** commit, from the repository root:

```bash
git branch --show-current            # must be remove-automapper
dotnet restore
dotnet build --no-restore -c Release
dotnet test --no-build -c Release
# + Swagger smoke run of every API (below)
```

**Swagger smoke run (per API project):**

1. Read `Properties/launchSettings.json`, take the profile with `"commandName": "Project"`: its `applicationUrl`
   (prefer `http://`), `launchUrl` (e.g. `swagger`) and environment (`Development`). Never run with Production
   settings; if a connection string or service URL points to a non-local host, don't start the API — ask the user.
2. Start it: `dotnet run --project <api.csproj> -c Release --no-build --launch-profile <profile>` (in the
   background, output to a log file **outside the repo**).
3. Wait until `<base>/<launchUrl>/index.html` returns **200**, then download the Swagger JSON
   (`/swagger/v1/swagger.json`, or `/openapi/v1.json` with `MapOpenApi()`): must be **200** and valid JSON.
4. Compare with the baseline: the list of operations (HTTP method + path) must be identical. Schema changes are
   expected only for converted types (e.g. `readOnly`) — list them.
5. The log must have no new `Unhandled exception`, `crit:` or `fail:` entries — in particular **no
   `Unable to resolve service for type 'I...Mapper'`** (missing DI registration).
6. Stop the API and its child process (`pkill -P <pid>; kill <pid>` on Linux/macOS, `taskkill /PID <pid> /T /F` on
   Windows).

Commit **only if** build has 0 errors, every test that passed at baseline still passes (new mapper tests included),
and the smoke run passes for every API that started at baseline. If the gate fails: don't commit, don't use
`--no-verify`; fix and re-run. If you can't get it green, stop and report, leaving the changes uncommitted.

Commits — one per profile/feature batch (each passing the gate), or one at the end:

```bash
git status                           # no logs, TestResults/, bin/, obj/
git add -A
git commit -m "Replace AutoMapper mapping of LandedCost with LandedCostMapper"
# last one:
git commit -m "Remove AutoMapper package and configuration"
```

## 10b. Pre-push check (mandatory, on the final state)

Run this full sequence once more after the last commit, from a clean state, in this order. Stop at the first
failure — **do not push**:

```bash
git branch --show-current            # 1. must be remove-automapper
git status                           #    must be clean (everything committed, no logs/bin/obj)
git grep -nE 'AutoMapper|IMapper\b|CreateMap<|ProjectTo<' -- '*.cs' '*.csproj' 'Directory.Packages.props'   # 2. must print nothing
dotnet clean
dotnet restore                       # 3. restore
dotnet build --no-restore -c Release # 4. build: 0 errors, no new warnings vs baseline
dotnet test --no-build -c Release    # 5. tests: everything that passed at baseline passes + new mapper tests pass
```

6. **Run the API(s), if the solution has any** — every project with `Sdk="Microsoft.NET.Sdk.Web"` (and workers if
   any): do the Swagger smoke run from §10 for each one. Swagger UI 200, Swagger JSON 200, same operations as the
   baseline, no new errors in the log (especially no `Unable to resolve service for type 'I…Mapper'`), then stop
   it.
   - If the solution has **no API project** (class libraries only), say so in the report and skip this step.
   - If an API can't be started for environment reasons it also failed on at baseline (e.g. no local DB), report it
     — that alone doesn't block the push.
7. **Only when 1–6 pass**, push:

   ```bash
   git push -u origin remove-automapper
   ```

   If anything fails: fix it, commit the fix (through the §10 gate), and run the **whole** pre-push check again.
   Never push with `--no-verify` or with failing checks. If you can't get it green, don't push — report what fails.

Don't open a pull request unless asked; suggest one from `remove-automapper` into the default branch at the end.

## 11. Final report

1. **Summary**: branch, commits, pushed (yes/no), number of profiles/maps replaced, mappers created (list with
   file paths), the `Mappers` folder(s) used or created and why, DI registration location, packages removed.
   Include the pre-push check results (build, tests, API run per API).
2. **Destination types**: converted to get-only + constructor (list), and types intentionally **not** converted
   with the reason (EF entity, updated in place, query-bound request, public package contract …).
3. **Moved logic**: each piece of logic moved from profiles/resolvers/`AfterMap` into a mapper, with file
   references.
4. **`ProjectTo` replacements**: each query, and whether the SQL was compared.
5. **Tests**: tests added (characterization/unit), changed, deleted (`AssertConfigurationIsValid`), and results
   before/after.
6. **Smoke run**: per API — Swagger UI/JSON status, operations before/after, schema differences (`readOnly`, etc.).
7. **Needs human attention**: possible bugs preserved on purpose, culture-dependent conversions, cyclic maps,
   classes with many injected mappers, serialization contracts that changed, anything not verified.
