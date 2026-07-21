# PR Comments Resolution

## Purpose

This file instructs the GitHub Copilot agent to **automatically read, categorize, and resolve all open pull request comments** on this repository.

Before executing any resolution, load and strictly follow:
- `.github/copilot-instructions.md` — the primary project specification

All fixes must comply with those two files. Do not apply any pattern, package, or convention that contradicts them.

---

## Trigger

Run this instruction set when asked to:

> *"Resolve PR comments"* / *"Fix review feedback"* / *"Address PR review"*

---

## Step 1 — Load Project Context

Before touching any file, read the following in order:

1. `.github/copilot-instructions.md` — note the selected architecture, auth type, database provider, cloud provider, HTTP clients, multi-tenancy flag, and coding standards.

Do not proceed until the file is loaded and understood.

---

## Step 2 — Collect PR Comments (Mode Detection)

The agent must first detect which mode it is operating in and collect comments accordingly.

---

### Mode A — GitHub Copilot Agent (inside a GitHub PR)

This is the **primary mode**. When running as a GitHub Copilot agent with an active PR context:

- Use the native GitHub PR tool to fetch all unresolved review comments from the current pull request.
- Include both **inline comments** (on specific lines) and **general PR-level comments**.
- Filter to unresolved / non-outdated comments only.
- For each comment record:

| Field | Description |
|---|---|
| `file` | File path the comment targets (`null` for PR-level comments) |
| `line` | Line number or range (`null` for PR-level comments) |
| `author` | GitHub username of the reviewer |
| `body` | Full text of the comment |
| `category` | Assigned in Step 3 |

Do not begin applying fixes until ALL comments are collected.

---

### Mode B — No GitHub Integration Available

If the GitHub PR tools are **not available** in the current workspace (e.g., running inside Claude, a local IDE without GitHub MCP, or any environment without live PR access), do the following:

**Do not fail silently or stop.** Instead, switch to manual input mode:

1. Acknowledge the limitation clearly:

   > "GitHub PR integration is not available in this workspace. To proceed, please paste your PR comments below using the template provided."

2. Present the following **comment input template** to the user and ask them to fill it in:

```
## PR Comments Input

<!-- Paste one block per comment. Add more blocks as needed. -->

### Comment 1
- File: src/Project.Api/Controllers/UserController.cs
- Line: 42
- Author: reviewer-username
- Comment: "The cancellation token is not being passed to the service layer here."

### Comment 2
- File: src/Project.Infrastructure/Persistence/AppDbContext.cs
- Line: 18
- Author: reviewer-username
- Comment: "Entity configuration should not be inline — use IEntityTypeConfiguration<T>."

### Comment 3
- File: (general PR comment — no specific file)
- Line: N/A
- Author: reviewer-username
- Comment: "Integration tests are missing for the new endpoint."
```

3. Wait for the user to paste their comments before proceeding.
4. Once comments are received, parse them into the same internal structure as Mode A and continue from **Step 3** onward — the resolution process is identical regardless of input mode.

---

### Mode C — Comments Provided Inline in the Prompt

If the user pastes PR comments directly into the chat or prompt without using the template above, accept them as-is. Extract the file, line, author, and comment body from whatever format was provided. If any field is missing (e.g., no file path given), infer it from context or mark it as `unknown` and proceed — do not block on missing metadata.

---

> **In all three modes, the resolution logic from Step 3 onward is identical.**
> The only difference is how comments are collected. Once collected, proceed.

---

## Step 3 — Categorize Every Comment

Assign each comment exactly one category from the table below:

| Category | When to Apply |
|---|---|
| `bug` | Logic error, wrong return value, null reference, incorrect behavior, off-by-one |
| `style` | Naming, casing, formatting, C# idiomatic pattern violations |
| `architecture` | Code placed in the wrong layer, wrong project, or violates the selected architectural pattern |
| `performance` | Missing `async`/`await`, blocking call (`.Result`, `.Wait()`), in-memory sorting/pagination, missing `CancellationToken` propagation, N+1 query |
| `security` | Secret in config, missing auth check, insecure default, unvalidated input reaching infrastructure |
| `ef-core` | Inline `OnModelCreating` configuration instead of `IEntityTypeConfiguration<T>`, lazy loading used, `SaveChangesAsync` called outside `IUnitOfWork`, missing soft-delete filter |
| `validation` | Missing `FluentValidation` validator, validator placed in the wrong layer, not registered via `SharpGrip` AutoValidation |
| `mapping` | Mapster config missing, manual property mapping instead of Mapster, projection not used where required |
| `test` | Missing test, wrong assertion style (not `FluentAssertions`), `AutoFixture` not used, test coverage gap, wrong test type (unit vs integration), mock not verified |
| `api-contract` | Response not wrapped in `ApiResponse<T>`, missing XML doc comment, wrong HTTP status code, API versioning missing on controller |
| `logging` | Serilog not used, exception logged at wrong layer, sensitive data in log |
| `cloud` | Cloud service interface/implementation in wrong project folder, missing `CancellationToken` in cloud method, wrong DI registration |
| `http-client` | Polly pipeline missing or misconfigured, typed client not using resilience handler, `HttpClient` instantiated directly with `new` |
| `docs` | Missing or incorrect XML documentation, outdated `README.md` section, missing `appsettings.json` key reference |
| `question` | Reviewer asked a question — requires a response comment, not a code change |
| `wontfix` | Out of scope, intentional design decision, or deferred — requires a written explanation |

---

## Step 4 — Resolution Rules Per Category

Apply the rules below exactly. Do not change code outside the direct scope of each comment.

---

### `bug`
- Identify the root cause before changing anything.
- Apply the minimal fix — do not refactor surrounding code.
- Add or update the corresponding unit test to cover the fixed scenario.

---

### `style`
Follow the conventions from `.github/copilot-instructions.md` — Section: *Coding Standards & Conventions*. Key rules:
- `PascalCase` for types, methods, properties; `camelCase` for locals and parameters; `_camelCase` for private fields.
- `sealed` on all concrete service and handler classes.
- No `var` unless the right-hand side makes the type immediately obvious.
- XML documentation on all `public` members.
- Never use `// TODO` — resolve inline or create a follow-up issue.

---

### `architecture`

Refer to `.github/copilot-instructions.md` — Section: *Architecture Rules*. Apply the correct placement:

| Concern | Clean Architecture | Onion Architecture | Repository + UoW | Vertical Slice |
|---|---|---|---|---|
| Domain entities | `{Project}.Domain/Entities/` | `{Project}.Domain/Entities/` | `{Project}.Domain/Entities/` | `{Project}.Api/Common/Domain/` |
| Interfaces (application) | `{Project}.Application/Common/Interfaces/` | `{Project}.Application/Interfaces/` | `{Project}.Application/Interfaces/` | `{Project}.Api/Common/Interfaces/` |
| EF DbContext + Configurations | `{Project}.Infrastructure/Persistence/` | `{Project}.Infrastructure/Persistence/` | `{Project}.Infrastructure/Persistence/` | `{Project}.Infrastructure/Persistence/` |
| Cloud services | `{Project}.Infrastructure/CloudServices/` | `{Project}.Infrastructure/CloudServices/` | `{Project}.ThirdParty/CloudServices/` | `{Project}.Infrastructure/CloudServices/` |
| HTTP clients | `{Project}.Infrastructure/HttpClients/` | `{Project}.Infrastructure/HttpClients/` | `{Project}.ThirdParty/HttpClients/` | `{Project}.Infrastructure/HttpClients/` |

Move files to the correct location and update all `using` statements and project references.

---

### `performance`
- Replace all `.Result` / `.Wait()` with `await`.
- Propagate `CancellationToken` from controller through service layer down to every `async` call — never default it to `default` on parameters unless it is the top-level controller action.
- Replace any in-memory `.ToList()` + LINQ sort/page with EF Core `OrderBy` + `Skip` + `Take` applied before materialisation.
- Replace `Include` chains that load unnecessary data with Mapster projections (`ProjectToType<T>()`).
- Add `AsNoTracking()` on all read-only queries.

---

### `security`
- Move any secret value from `appsettings.json` to environment variables or the configured Key Vault / Secrets Manager.
- Add `[Authorize]` to endpoints that are missing it, or add the correct policy/role requirement.
- Ensure all user-supplied strings pass through the FluentValidation pipeline before reaching infrastructure.
- Never log passwords, tokens, connection strings, or PII via Serilog.

---

### `ef-core`
- Move any inline entity configuration from `OnModelCreating` into a dedicated `IEntityTypeConfiguration<T>` class under `Persistence/Configurations/`.
- Ensure `OnModelCreating` only calls `modelBuilder.ApplyConfigurationsFromAssembly(Assembly.GetExecutingAssembly())`.
- Remove all lazy loading — replace with explicit `.Include()`.
- Move any direct `SaveChangesAsync()` calls from repositories or services into `IUnitOfWork.CommitAsync()`.
- Verify the global soft-delete query filter (`HasQueryFilter(e => !e.IsDeleted)`) exists in the relevant configuration.
- Verify `CreatedAt` and `UpdatedAt` are set in `DbContext.SaveChangesAsync` override — not in entity constructors.

---

### `validation`
- Create a `FluentValidation` `AbstractValidator<T>` for the missing model.
- Place it in the `Validators/` folder co-located with the request model in the Application layer.
- Ensure it is picked up automatically via `AddValidatorsFromAssemblyContaining<Program>()` — no manual registration.
- Verify `AddFluentValidationAutoValidation()` is called in `Program.cs`.

---

### `mapping`
- Add a Mapster `TypeAdapterConfig` mapping in the dedicated Mapster class library (`{Project}.Mapster` or equivalent per architecture).
- Replace manual `new Response { Prop = entity.Prop }` assignments with `entity.Adapt<ResponseType>()`.
- Use `ProjectToType<T>()` on `IQueryable<T>` for DB-level projections — never `.ToList()` then `.Adapt()`.

---

### `test`
Apply based on test type:

**Unit tests (`{Project}.Tests.Unit`)**
- Use `Moq` to mock all dependencies — no real DB, no real HTTP.
- Use `AutoFixture` with `AutoMoqCustomization` for test data — no manually constructed objects.
- Use `FluentAssertions` — never `Assert.Equal` or `Assert.True`.
- Follow naming pattern: `MethodName_StateUnderTest_ExpectedBehavior`.
- One test class per production class.
- Verify mock interactions with `.Verify(...)` where behaviour matters.

**Integration tests (`{Project}.Tests.Integration`)**
- Use `WebApplicationFactory<Program>` for full pipeline tests.
- Replace `DbContext` with EF Core InMemory using a unique DB name per test class (`"TestDb_" + Guid.NewGuid()`).
- Seed test data per test — never share state across test classes.
- Assert full HTTP response: status code + `ApiResponse<T>` body structure.

---

### `api-contract`
- Wrap any unwrapped response in `ApiResponse<T>.Ok(data)` or `ApiResponse<T>.Fail(message)`.
- Add `[ApiVersion("1.0")]` and `[Route("api/v{version:apiVersion}/[controller]")]` to any controller missing versioning.
- Add XML `/// <summary>` to all public controller actions, request models, and response models.
- Return correct HTTP status codes: `201 Created` for POST, `200 OK` for GET/PUT, `204 No Content` for DELETE, `400 Bad Request` for validation errors, `404 Not Found` for missing resources, `401 Unauthorized` / `403 Forbidden` for auth failures.

---

### `logging`
- Move exception logging to the API boundary (global exception middleware or controller) — remove it from domain/application/infrastructure layers.
- Replace any `Console.WriteLine` or `Debug.WriteLine` with `ILogger<T>`.
- Ensure Serilog is used via `ILogger<T>` injection — not accessed as a static `Log.` call inside business logic.
- Remove any log statement that includes passwords, tokens, secrets, or full connection strings.

---

### `cloud`
- Verify the interface is in the correct location per the architecture table in the `architecture` section above.
- Verify the implementation is in `{Project}.Infrastructure/CloudServices/` (or `{Project}.ThirdParty/CloudServices/` for Repository+UoW).
- Ensure every cloud method accepts and passes `CancellationToken cancellationToken` — never `default`.
- Verify DI registration uses `AddScoped<IStorageService, ...>()` and `AddScoped<ISecretService, ...>()`.

---

### `http-client`
- Ensure the Polly resilience pipeline is registered once (`AddResiliencePipeline("default-http-pipeline", ...)`) and reused by all typed clients via `AddResilienceHandler`.
- Replace any `new HttpClient()` instantiation with a DI-injected typed client.
- Verify the Polly pipeline includes: retry (3 attempts, exponential back-off, jitter) + circuit-breaker (50% failure rate, 30s window, 15s break) + per-attempt timeout (10s).
- Verify the typed client follows the interface pattern: `I{ServiceName}Client` → `{ServiceName}Client`.

---

### `docs`
- Add missing XML `/// <summary>` documentation to the flagged member.
- Update `README.md` if the change affects a documented behaviour (endpoint, config key, migration command).
- Add any new `appsettings.json` key to the *Configuration Reference* table in `README.md`.

---

### `question`
- If the reviewer's question is answered by the existing code or a referenced decision in `copilot-instructions.md`, post a reply comment referencing the relevant section.
- If the question reveals an undocumented design decision, add a brief inline `// Reason: ...` comment to the code and reply to the PR comment pointing to it.
- Do not change code to answer a question unless the question implies the code is wrong.

---

### `wontfix`
- Do not change the code.
- Post a PR reply in the following format:

```
Marking as Won't Fix.

Reason: [one of the following]
  - Out of scope for this PR — tracked in Issue #[N]
  - Intentional design decision: [brief explanation referencing copilot-instructions.md section]
  - Deferred: [reason + link to follow-up]
```

---

## Step 5 — Post-Fix Verification

After all fixes are applied, run the following checks in order. Do not skip any.

```bash
# 1. Build entire solution
dotnet build

# 2. Run all tests
dotnet test

# 3. Check EF Core model (if ef-core comments were resolved)
dotnet ef migrations add VerifyModel --project {Project}.Infrastructure --startup-project {Project}.Api
# Review generated migration — if no changes are expected, delete it immediately
dotnet ef migrations remove --project {Project}.Infrastructure --startup-project {Project}.Api
```

If any check fails, fix the failure before posting the resolution summary. Do not leave the build or tests in a broken state.

---

## Step 6 — Resolution Summary Comment

After all fixes pass verification, post a single PR comment in the following format:

```markdown
## PR Comment Resolution Summary

### ✅ Resolved

| File | Line | Category | Change Applied |
|------|------|----------|----------------|
| `path/to/file.cs` | 42 | `bug` | Fixed null reference in UserService.CreateAsync |
| `path/to/file.cs` | 88 | `architecture` | Moved IStorageService interface to Application layer |

### ⏭️ Won't Fix

| File | Line | Reason |
|------|------|--------|
| `path/to/file.cs` | 15 | Intentional — Lazy<T> not applicable here; dependency is eagerly required on every request |

### 🔖 Follow-up Issues Created

| Title | Reason |
|-------|--------|
| Migrate from InMemory to Testcontainers for integration tests | Out of scope for this PR |
```

Then request re-review.

---

## Hard Constraints

These rules override everything else and must never be violated:

- **Never downgrade** to .NET 8 or .NET 9 patterns — all code targets .NET 10.
- **Never change code outside the scope** of a comment — no opportunistic refactoring.
- **Never remove code** to resolve a comment unless the reviewer explicitly asked for removal.
- **Never add a NuGet package** that is not already listed in `copilot-instructions.md` without explicitly noting it in the resolution summary.
- **Never leave partial fixes** — each comment is either fully resolved or marked `wontfix`.
- **Never use placeholder comments** (`// TODO`, `// ... rest of code`) — all generated code must be complete and compilable.
- **Never silence a compiler warning** with `#pragma warning disable` to make the build pass.