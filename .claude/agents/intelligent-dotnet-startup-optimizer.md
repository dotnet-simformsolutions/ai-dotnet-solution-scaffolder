# Intelligent .NET Startup Optimizer — Agent Instructions

Portable, self-sufficient instructions for an AI coding agent to analyze, generate an
optimized rewrite of, and report on a .NET application's startup configuration
(`Program.cs` / `Startup.cs`). Drop this file into the root of any .NET repo — it's
self-contained and doesn't depend on any specific project structure or external tool.
Every check in Step 2 is something you (the agent) perform directly by reading the code
with your own file-search/read tools — nothing needs to build or run for the checklist
itself. Building only comes in afterward: optionally, to cross-check findings against
compiler/analyzer output (Step 3), and later, to verify a change you've actually applied
(Step 9).

## When to use this

Apply these instructions when the user asks to review, audit, or optimize a .NET app's
startup/bootstrap code, or reports symptoms like slow startup, DI registration errors,
"service not found" exceptions that only show up at runtime, or middleware behaving in
an unexpected order.

## Ground rule: findings must come from this codebase, not from memory

This checklist names known .NET anti-patterns so you know what to look for — it is not a
list of things to report by default. Every finding you write down must be backed by
something you actually read in this repo: quote the exact line, method, or JSON key. If
a check doesn't apply — no `Startup.cs` exists, there's no `appsettings*.json`, no
blocking calls appear anywhere — say nothing about it, don't pad the report with generic
"industry best practice" advice for a problem that isn't present. Report uncertainty
through the Confidence field (Step 4), not prose hedging — e.g. a `.Result` property
access that might not be on a `Task` is a **Medium**-confidence finding, not an
asterisked claim.

## Step 1 — Locate the startup surface

Find, in order of relevance:

- The `.sln` file, if one exists, to see every project in the solution. Identify the
  startup/entry project(s) from it — usually the one(s) with `Sdk="Microsoft.NET.Sdk.Web"`
  or an executable output containing a `Program.cs` with `WebApplication.CreateBuilder`/
  `Host.CreateDefaultBuilder` — rather than assuming the first project alphabetically. A
  solution can have more than one startup project (e.g. an API and a worker service);
  analyze each one you find, and say which project each finding belongs to.
- `Program.cs` (minimal hosting model, .NET 6+) and/or `Startup.cs`
  (`ConfigureServices`/`Configure` split, older template) in each startup project.
- Each startup project's own `.csproj`, to read `<TargetFramework>` — recommendations
  should match the actual target (e.g. don't suggest `AddOpenApi()` on a project
  targeting `net6.0`; it ships in .NET 9+) — and other project-level settings checked in
  item 8 below.
- `appsettings.json` / `appsettings.{Environment}.json`, if startup reads configuration.
- The rest of the solution's `.cs` files — needed for the cross-file checks in items
  1 and 7 below (a project-wide search, e.g. grep, is enough; you don't need to open
  every file).

**Follow extension methods before judging DI/middleware.** Real apps rarely put every
`AddX()`/`UseX()` call directly in `Program.cs` — they're commonly grouped into custom
`IServiceCollection`/`IApplicationBuilder` extension methods (e.g.
`builder.Services.AddPersistence().AddMessaging()`, `app.UseCustomPipeline()`), often
defined in a different file, or even a different project in the solution. Before running
the checklist in Step 2, find and open the definition of every such extension method
reachable from `Program.cs`/`Startup.cs` — search the solution for
`static ... this IServiceCollection` / `static ... this IApplicationBuilder` method
signatures, since that's where the real registrations and middleware calls live. Items 1
through 4 need to see what's actually registered and in what order, not just the
top-level wrapper calls.

## Step 2 — Analysis checklist

Go through each of these against the startup surface (Step 1). For every issue found,
note: file, line, a one-sentence explanation of *why* it's a problem, and a concrete fix.

1. **Duplicate / conflicting DI registrations** — before flagging any repeated
   `AddSingleton`/`AddScoped`/`AddTransient` call for the same service type, search the
   solution for a consumer that requests *all* registrations of that type rather than a
   single one — either by injecting `IEnumerable<TService>` (a constructor or method
   parameter of that shape), or by calling `serviceProvider.GetServices<TService>()` /
   `GetServices(typeof(TService))` (the common form in factory/strategy-dispatch code).
   If either exists, every registration is intentional and consumed by that consumer —
   do not flag it. Only flag repeated registrations of a type that's requested as a
   single instance somewhere (bare `TService` via constructor injection or
   `GetService<T>()`/`GetRequiredService<T>()`, never the plural form), since that's the
   case where the container silently returns only the last-registered implementation to
   that consumer. Flag the same type registered under *different* lifetimes as more
   serious — a likely captive-dependency bug — but apply the same
   `IEnumerable<T>`/`GetServices<T>()` check first before concluding it's unintentional.

   ```csharp
   // Counter-example — NOT a duplicate: every registration is consumed together.
   services.AddSingleton<INotifier, EmailNotifier>();
   services.AddSingleton<INotifier, SmsNotifier>();
   services.AddSingleton<Dispatcher>(); // Dispatcher(IEnumerable<INotifier> notifiers)
   ```

   - `AddHostedService<T>()` registers `IHostedService` under the hood on every call,
     but calling it once per distinct background-service type is normal, not a
     duplicate: N calls for N *different* `T`s are N intentionally-concurrent hosted
     services, not N-1 duplicates of the same registration. Only flag
     `AddHostedService<T>()` if the exact same `T` is registered more than once.

     ```csharp
     // Counter-example — NOT duplicates: two different hosted services.
     services.AddHostedService<EmailQueueWorker>();
     services.AddHostedService<MetricsFlushWorker>();
     ```
   - Repeated calls to `TryAddSingleton`/`TryAddScoped`/`TryAddTransient`/
     `TryAddEnumerable` are never duplicates — the entire point of the `Try*` family is
     to be safely callable more than once (e.g. from several extension methods) and
     register only if nothing is already registered for that type. Do not flag them
     regardless of how many times they appear.

     ```csharp
     // Counter-example — NOT a duplicate: TryAdd* is safe to call any number of times.
     services.TryAddSingleton<IClock, SystemClock>();
     services.TryAddSingleton<IClock, SystemClock>();
     ```
   - `services.BuildServiceProvider()` called anywhere during service registration
     (i.e. before the final `builder.Build()`/`host.Build()`) builds a *second*,
     throwaway container distinct from the one the app actually runs on — anything
     resolved from it can be a different instance than what the rest of the app
     receives, a classic source of "why do I have two instances of this singleton"
     bugs. **Warning.** This is exactly what the built-in analyzer rule `ASP0000`
     flags; if you're cross-checking build output per Step 3, note the match.

2. **Incorrect service lifetimes** — a singleton depending on (or capturing) a scoped or
   transient service; `HttpClient` registered as a bare singleton instead of via
   `AddHttpClient`; `DbContext` registered as singleton. Check constructors of registered
   types to see what they actually depend on, not just the registration line. If you find
   a genuine lifetime risk here, also look at the host's `ServiceProviderOptions`:
   `Host.CreateDefaultBuilder`/`WebApplicationBuilder` already set `ValidateScopes` and
   `ValidateOnBuild` to `true` by default whenever `IsDevelopment()` is true — so telling
   someone to "enable it for Development" is a no-op on a default-configured host; they'd
   implement it and see no change. What's actually worth recommending:
   - Enabling both **unconditionally**, in every environment — e.g.
     `builder.Host.UseDefaultServiceProvider(o => { o.ValidateScopes = true; o.ValidateOnBuild = true; })`
     with no environment check — so a captive dependency that only manifests under a
     staging/production configuration difference is caught there too, not only on a
     developer's machine. **Suggestion.**
   - Checking whether the project has explicitly *overridden* the default to turn
     validation off (e.g. an explicit `ValidateScopes = false`, or a custom
     `IServiceProviderFactory` that skips it). If so, flag it and ask whether that was
     intentional — it removes a safety net the framework otherwise gives for free.
     **Warning.**
   The lifetime issue itself keeps whatever severity you'd otherwise give it (typically
   Critical for a confirmed captive dependency).

3. **Middleware order and pipeline issues** — the ASP.NET Core pipeline has a documented
   required order. **Exception that governs every check below involving `UseRouting`:**
   on .NET 6+, `WebApplication` automatically inserts `UseRouting` at the start of the
   pipeline if it's never called explicitly. So a pipeline with no explicit
   `UseRouting()` call anywhere — e.g.

   ```csharp
   // Counter-example — correct on .NET 6+: WebApplication inserts UseRouting
   // automatically since it's never called explicitly here.
   app.UseCors();
   app.UseAuthentication();
   app.UseAuthorization();
   app.MapControllers();
   ```

   — is correct on a .NET 6+ target, not a violation of any "before `UseRouting`" rule
   below. Only flag a routing-order problem on a .NET 6+ target when `UseRouting()` *is*
   called explicitly and still appears after routing-dependent middleware, or on a
   target older than .NET 6 (where there's no automatic insertion to rely on). Never
   report "missing `UseRouting()`" by itself as an issue on a .NET 6+ target.
   - `UseAuthentication` before `UseAuthorization` (reversed → every request looks
     unauthenticated). **Critical.**
   - `UseRouting` before `UseCors`/`UseAuthentication`/`UseAuthorization`/`UseEndpoints`/
     `Map*` (they depend on the endpoint match having already happened). **Critical.**
   - `UseCors` before `UseAuthentication` (so preflight requests aren't rejected by
     auth). **Critical.**
   - `UseExceptionHandler`/`UseHsts` registered first, before anything that could throw
     or that should be covered by HSTS. **Warning.**
   - `UseHttpsRedirection` and `UseStaticFiles` before `UseRouting` (the standard
     ASP.NET Core template order). **Warning.**
   - `UseResponseCompression` before `UseStaticFiles`/`UseRouting`, so responses are
     actually eligible for compression. **Warning.**
   - `UseSession` before any middleware/endpoint that reads session state. **Warning.**
   - `UseRateLimiter` / `UseOutputCache` (.NET 7+) after `UseRouting`, before `Map*`.
     **Warning.**
   - Any middleware or `Map*` call placed **after** `app.Run(context => ...)` — the
     terminal `IApplicationBuilder.Run` overload that takes a request delegate — is
     unreachable dead code, since `Run` always ends the pipeline right there. **Critical**
     (code that looks active but silently never executes — e.g. an auth check placed
     after it — is a real behavior bug, not just clutter). Don't confuse this with the
     parameterless `app.Run()` that starts the host, expected at the very end of
     `Program.cs`.
   - Exact duplicate middleware registrations (the same `UseX()` call appearing more
     than once with no apparent reason). **Suggestion.**
   - A CORS policy combining `AllowAnyOrigin()` with `AllowCredentials()` in the same
     policy — this throws an `InvalidOperationException` at runtime, it isn't just bad
     practice. **Critical.**

4. **Blocking and startup-stalling calls**:
   - Synchronous-over-async: `.Result`, `.Wait()`, `.GetAwaiter().GetResult()` on a
     `Task`/`Task<T>` anywhere in the startup path. **Warning** — recommend `await`
     (making the containing method async) or moving the work into an
     `IHostedService`/`BackgroundService` that runs after the app starts listening.
   - `context.Database.Migrate()` / `EnsureCreated()`, or a manual data-seeding loop,
     called directly in `Program.cs`/`Startup.cs` before `app.Run()`. **Warning.**
     **Default confidence: Medium** — whether this is actually a problem depends on
     deployment shape (single instance vs. many), which isn't visible from the code
     alone. This is a very common startup stall that is *not* sync-over-async — it's
     often correctly awaited — so the check above won't catch it. It blocks every
     instance from reaching "started" until it finishes, and running it unconditionally
     on every instance in a multi-instance deployment is a race. Recommend a dedicated
     startup hosted service that runs it once and exits, or moving it out of the app
     entirely into a deploy-pipeline step (e.g. `dotnet ef database update` in CI/CD).
   - A `BackgroundService.ExecuteAsync` override doing non-trivial synchronous work
     *before its first `await`*. **Warning.** That prefix runs inline as part of
     starting the service and can delay host startup. For ordered startup work
     specifically, mention `IHostedLifecycleService` (.NET 8+) as the modern
     alternative — it gives explicit `StartingAsync`/`StartedAsync` hooks instead of
     overloading `ExecuteAsync` for startup-time work.

5. **Legacy patterns worth modernizing**:
   - A `Startup.cs` with `ConfigureServices`/`Configure`, **on a project targeting .NET
     6+** → can collapse into a single top-level `Program.cs` using
     `WebApplication.CreateBuilder(args)`. On an older target (.NET Core 3.1, .NET 5),
     the minimal hosting model isn't available — note that upgrading the target
     framework is a prerequisite, don't suggest the code change alone. **Suggestion.**
   - `app.UseEndpoints(endpoints => endpoints.MapX(...))` → the `Map*` calls can go
     directly on `app`/`WebApplication` without the `UseEndpoints` wrapper.
     **Suggestion.**
   - `ILogger.LogXxx($"...")` with a string-interpolated message → recommend the
     message-template overload or a `[LoggerMessage]` source-generated method, so the
     string isn't built when the log level is filtered out. **Info.**
   - `AddSwaggerGen()` on a project targeting .NET 9+ → the built-in
     `Microsoft.AspNetCore.OpenApi` (`AddOpenApi()`/`MapOpenApi()`) covers most of the
     same ground without an extra dependency. **Info.**
   - `IConfiguration.Get<T>()`/`.Bind(...)` on a project targeting .NET 8+ → binds via
     reflection; recommend enabling the configuration binding source generator
     (`<EnableConfigurationBindingGenerator>true</EnableConfigurationBindingGenerator>`).
     **Info.**

6. **Configuration issues**:
   - Read every `appsettings*.json` in the project, then cross-reference
     `GetSection(...)`/`GetValue(...)`/`Configuration["x"]` key lookups in the startup
     code against it. Flag any key referenced in code that doesn't exist in the file
     (likely a typo or leftover reference — skip this check entirely if the project has
     no `appsettings*.json` at all, since config may come entirely from environment
     variables or a secret manager). **Warning.** **Default confidence: Medium** — a
     missing key is often a typo, but it may also be intentionally supplied by an
     environment variable or secret manager that isn't visible in this file.
   - Flag JSON values in `appsettings*.json` that look like committed secrets: key names
     like `Password`/`ApiKey`/`Secret`/`ConnectionString`, or a value embedding
     `Password=...`/`Pwd=...`. Recommend moving those to `dotnet user-secrets` (local) or
     environment variables/a secret manager (other environments) — and never repeat the
     actual secret value back in your report, only its location. **Warning.**
   - `builder.Configuration.AddUserSecrets<T>()` (or the non-generic overload) called
     without being gated by `if (builder.Environment.IsDevelopment())`. **Warning.** User
     secrets are meant to be a local-development-only provider; reachable in other
     environments it's at best dead weight, at worst a confusing source of
     configuration-precedence surprises.
   - Synchronous configuration/secret loading, spelled out concretely:
     `File.ReadAllText(...)` (or `File.ReadAllBytes`/`ReadAllLines`) used to manually
     parse a config file instead of the `IConfiguration` providers, or a synchronous
     (non-`Async`, unawaited) call to a remote config source or Key Vault SDK made
     directly in `Program.cs`. **Warning.** Recommend the async provider/SDK method, or
     moving the fetch into a hosted service.
   - Configuration bound to strongly-typed options without fail-fast validation, **on a
     project targeting .NET 6+** (`ValidateOnStart()` isn't available before that):
     `AddOptions<T>().Bind(section)` (or `services.Configure<T>(section)`) with no
     `.ValidateDataAnnotations().ValidateOnStart()` chained on. **Suggestion.** Without
     it, a missing or invalid setting isn't caught until the option is first *used* —
     potentially well after startup, mid-request. Recommend chaining it so invalid
     configuration fails fast at startup instead.

7. **Startup-time cost and reflection**:
   - Reflection-heavy calls: `Assembly.GetTypes()`/`GetExportedTypes()`-style member
     enumeration, `Activator.CreateInstance(...)`, `MethodInfo.Invoke(...)`.
     **Suggestion.**
   - Assembly-scanning registration libraries — in real apps, the reflection cost
     usually isn't raw `Assembly.GetTypes()`, it's a scanning library call: Scrutor's
     `services.Scan(s => s.FromAssemblyOf<T>().AddClasses(...))`, MediatR's
     `AddMediatR(cfg => cfg.RegisterServicesFromAssembly(...))` /
     `RegisterServicesFromAssemblyContaining<T>()`, `AddAutoMapper(...)`, and
     FluentValidation's `AddValidatorsFromAssembly(...)`. **Suggestion.** Flag these when
     the scanned assembly looks broader than necessary (e.g. scanning a large
     shared/vendor assembly for a handful of types) — **default confidence: Low** for
     this breadth judgment, since "broader than necessary" is a judgment call, not a
     fixed rule — and flag it when the same assembly is scanned by more than one such
     call, since each is a separate full scan.
   - Large inline collections/object graphs (rule of thumb: 30+ elements) built eagerly
     at startup instead of lazily on first use or loaded from a data source.
     **Suggestion.**
   - Possibly-unused service registrations: for each registered service type, search the
     rest of the solution (constructor parameters, method parameters, minimal-API lambda
     parameters — a project-wide grep for the type name is enough) for a place it's
     actually requested. If you find none, flag it as a candidate for removal.
     **Suggestion.** **Default confidence: Low** — this search can miss
     `IServiceProvider.GetService(typeof(...))` or `GetServices<T>()` (if you already
     found one of these while working through item 1's guard, that's your answer here
     too — don't flag the same type as unused in this item after already establishing
     it's consumed in that one), reflection-based resolution, or a consumer in a
     different referenced project — check those specifically before deleting rather than
     deleting outright; that's also what would raise this finding's confidence.
     **Exempt `AddHostedService<T>()`
     and `AddOptions<T>()`/`services.Configure<T>(...)` registrations from this check
     entirely** — for `AddHostedService<T>()`, the registered service type is
     `IHostedService`, which the host resolves and starts internally, never via a
     constructor/method parameter in application code. The same reasoning applies to
     options: `AddOptions<T>()`/`Configure<T>(...)` registers `IOptions<T>` (and
     `IOptionsSnapshot<T>`/`IOptionsMonitor<T>`), not `T` itself — application code
     injects one of those wrapper types, never bare `T`, so "no consumer found for `T`"
     is the normal, expected case here too, not a signal of anything wrong.

8. **Project-level startup levers** — read the `.csproj` (and `Directory.Build.props`,
   if present):
   - An out-of-support target framework — `net6.0` (end of support November 2024) or
     `net7.0` (end of support May 2024). **Warning** — this is a support/security
     posture issue, not just style. Recommend moving to `net8.0` (LTS) or later.
   - Missing `<InvariantGlobalization>true</InvariantGlobalization>` when the app shows
     no evident need for culture-specific behavior. **Suggestion.** **Default
     confidence: Medium** — "no visible use of `CultureInfo`, `.ToString(culture)`,
     resource-based localization" only rules out the obvious cases; drop to **Low** if
     you weren't able to search the whole solution for culture-sensitive code. It's
     usually a meaningful win on both startup time and image size by skipping ICU data
     loading, but invariant mode also changes culture-sensitive string comparison and
     sorting (e.g. `string.Compare`/case-insensitive comparisons of non-ASCII text behave
     differently) and can affect time-zone-name lookups on Linux. Say this explicitly in
     the recommendation rather than presenting it as risk-free, and skip the suggestion
     entirely if the app handles more than one culture's text or does any
     culture-sensitive sorting/comparison.
   - Missing `<PublishReadyToRun>true</PublishReadyToRun>` on an executable ASP.NET Core
     project (`Microsoft.NET.Sdk.Web`, or any other SDK producing a runnable service).
     **Suggestion.** **Default confidence: Low** — deployment intent isn't visible from
     source alone, so this is always somewhat speculative. It pre-JITs startup-path code
     instead of JITting it cold on first run — but only for *published* output; it has
     no effect on `dotnet run`/local debugging. Say that explicitly, but suggest it
     regardless of whether a publish profile is visible in the repo — the setting is
     harmless to add either way.
   - Native AOT (`<PublishAot>true</PublishAot>`) as a bigger lever for eligible apps
     (minimal APIs, no runtime reflection-based serialization, no unsupported
     dependencies). **Suggestion** — worth raising, not a default recommendation, since
     it requires the app to avoid the reflection-based patterns flagged in item 7.
   - `<TieredPGO>` explicitly disabled on a project targeting .NET 6/7. **Info** — it's
     on by default since .NET 8, so only worth mentioning for older targets that turned
     it off.
   All five are project-level checks, not code defects — only report the ones you can
   point to actual `.csproj` content for (present-and-wrong, or absent-and-would-help).

## Step 3 — Cross-check with build/analyzer output (optional)

If you can build the project, force a clean build — `dotnet build --no-incremental`, or
`dotnet clean` first — once before finalizing your report, and read the warnings. An
incremental `dotnet build` against an already-built tree recompiles nothing, so
analyzers don't run and it reports zero warnings no matter what's actually in the code —
that's evidence nothing was rebuilt, not evidence the project is clean; don't mistake a
warm-tree "0 warnings" for a clean bill of health. The compiler and the built-in
ASP.NET Core analyzers already catch part of this checklist deterministically and for
free:

- `ASP0014` ("Suggest using top level route registrations instead of `UseEndpoints`")
  is exactly item 5's `UseEndpoints` finding.
- `ASP0000` (calling `BuildServiceProvider()` during service registration) is exactly
  the second-container check in item 1.

Where a build warning matches a finding you already made by reading the code, say so
explicitly (e.g. "also reported as ASP0014") instead of presenting it as something only
this review caught — it's a stronger, deterministic confirmation, and it tells the user
this class of issue would surface on their next clean build regardless. If the project can't
be built in your environment (missing dependencies, no local SDK, a solution that's
intentionally not buildable outside CI, etc.), skip this step and say so — it's a
supplement to Step 2's checklist, not a prerequisite for it.

## Step 4 — Report

Summarize findings as a table — ID, Severity, Confidence, file, line, title — followed by
one section per finding, headed by that same ID, using the exact template below. Order
both the table and the sections by **severity first** (Critical, then Warning, then
Suggestion, then Info), and by **confidence within each severity band** (High before
Low) — the findings most worth acting on immediately come first. Assign IDs in this
final, sorted order — `F1`, `F2`, `F3`, ... — so every finding has a stable, short name a
human can approve or decline by referring to it, rather than by describing it in prose.

Every row must trace back to a specific file — and, for a finding about code that's
*present*, a specific line — or it doesn't belong in the report (see the ground rule
above). A finding about something **missing** (item 2's `ValidateScopes`/
`ValidateOnBuild`, item 8's project-level levers) is the deliberate exception: instead of
a line number, cite where the addition belongs — e.g. "host setup, in `Program.cs` near
`WebApplication.CreateBuilder`" or "project-level, in the `.csproj`'s `<PropertyGroup>`"
— but it still has to name a specific file. A finding that can't point to a file at all
doesn't belong in the report either. If a whole category (say,
middleware ordering) had nothing to flag, omit it from the report rather than writing "no
issues found" for every checklist item — a report of only real findings, even a short
one, is more useful than a checklist-shaped report padded with clean bills of health.

### Severity scale

- **Critical** — will cause a bug or wrong behavior (e.g. reversed auth middleware,
  conflicting lifetimes, a CORS policy that throws at runtime)
- **Warning** — real waste or risk but not incorrect today (e.g. duplicate registration,
  blocking call, a startup DB migration, a likely secret committed to config)
- **Suggestion** — modernization opportunity or heuristic finding worth a second look, no
  guaranteed behavior change (e.g. possibly-unused service, a project-level lever)
- **Info** — minor/stylistic (e.g. interpolated logging, reflective config binding)

### Confidence scale

Severity says how bad a finding is *if real*. Confidence says how sure you are that it's
real at all — the two axes are independent and never influence each other. A Critical
finding can be High confidence (a reversed auth pair is visible on the two lines cited)
just as easily as a Suggestion can be Low confidence (a possibly-unused service is a
guess, no matter how minor the fix would be).

- **High** — the defect is visible in the lines cited; no inference beyond reading the
  code. Reversed `UseAuthentication`/`UseAuthorization`, `AllowAnyOrigin()` +
  `AllowCredentials()`, code after a terminal `app.Run(...)`, or any pattern a built-in
  analyzer also reports (Step 3) — a deterministic second source outranks the agent's own
  reading.
- **Medium** — the code pattern itself is certain, but whether it's actually a *problem*
  depends on context not fully visible in the repo. A startup `Database.Migrate()` (fine
  for a single-instance deployment, a race for many), an ungated `AddUserSecrets`, a
  config key missing from `appsettings.json` (may legitimately come from an environment
  variable).
- **Low** — the finding rests on a search that can produce false negatives, or on
  deployment/runtime facts not present in the repo at all. Possibly-unused services,
  `PublishReadyToRun` (deployment intent is unknowable from source), an
  "this scan looks broader than necessary" judgment.

Two rules:
- **When torn between two confidence levels, pick the lower one** — same discipline as
  the ground rule above.
- **Every Low-confidence finding must say what would raise it** — the specific check a
  human could run to confirm or kill it (e.g. "search the solution for
  `GetService(typeof(IClock))` and any reflection-based resolution before deleting"). A
  Low finding with no path to resolution is just noise.

### Per-finding template

Use this exact structure for every finding — it's what lets a reader see, in one place,
what changes if they approve a given ID, without scrolling to the Step 5 rewrite:

    ### F<n> — <Severity> — <short title>

    **Where:** `<file>:<line or range>`
    **Confidence:** <High|Medium|Low> — <one clause saying what makes it that>
    **Problem:** One or two sentences on what the code does wrong. Quote the actual
    identifier or call from the file.
    **Impact:** What breaks at runtime, or what's wasted. Concrete, not generic.

    **Fix:**
    ```diff
    - <current line(s), verbatim from the file>
    + <corrected line(s)>
    ```

    **If you approve F<n>:** <what happens — see below>

Rules for each field:

- **Where** — a real line or range for present code. For an absence finding, keep the
  file-only convention above (no line number).
- **Confidence** — pick from the scale above; state the one clause that justifies it.
- **Fix** — always a fenced ```diff block, never prose-only. Three shapes:
  - change → `-` current line(s), `+` replacement
  - add (absence findings) → `+` only, with one or two unchanged lines of surrounding
    context above it so the insertion point is unambiguous
  - remove (duplicate registration, dead code) → `-` only
  Keep it minimal — the changed lines plus at most two lines of context. This is a
  preview, not the full Step 5 rewrite.
- **If you approve F\<n\>** — exactly one line, one of:
  - on Step 7's safe list → *"applied directly — <the mechanical change>, no decision
    needed."*
  - off the safe list, one obvious fix → *"applied after restating the behavior change:
    <the change>."*
  - off the safe list, several valid fixes → **state the question the user must answer**,
    e.g. *"needs your decision first: keep Singleton or Scoped for `IEmailSender`?"* Step
    7 already requires the user's stated choice rather than the agent picking one for
    them; surfacing the question here, at report time, is what makes that answerable
    instead of discovered only when the user tries to approve the ID.
- Where a built-in analyzer also catches the finding (per Step 3), append the rule ID to
  the title, e.g. `### F4 — Suggestion — UseEndpoints wrapper (ASP0014)`.

### Worked example

    ### F3 — Warning — `UseHttpsRedirection`/`UseStaticFiles` run after `UseRouting`

    **Where:** `Program.cs:41-45`
    **Confidence:** High — the call order is visible in the five lines cited; no
    inference needed.
    **Problem:** `app.UseRouting()` runs on line 41, before `app.UseHttpsRedirection()`
    (line 44) and `app.UseStaticFiles()` (line 45), reversing the standard ASP.NET Core
    template order.
    **Impact:** Not incorrect today, but static file requests and the HTTPS redirect are
    now evaluated *after* routing has already run — the standard order exists so those
    requests short-circuit before routing does any work.

    **Fix:**
    ```diff
     var app = builder.Build();

    -app.UseRouting();
    -
     app.UseHttpsRedirection();
     app.UseStaticFiles();
    +
    +app.UseRouting();
    +
     app.UseAuthentication();
    ```

    **If you approve F3:** applied directly — moves `UseRouting()` below
    `UseHttpsRedirection`/`UseStaticFiles`, no decision needed.

This report is followed immediately by the generated proposal in Step 5, which
references these same IDs — mention that it's coming so the two read as one deliverable,
not two disconnected outputs.

## Step 5 — Generate an optimized Program.cs

Every review produces a complete, ready-to-read proposed rewrite of the startup file —
not just a list of recommendations. This is the deliverable the brief actually asks for
("generate optimized startup code" / "produce optimized Program.cs suggestions").
Always include it, even when most findings are Suggestion/Info-level:

1. Start from the actual current file content (including folding in `Startup.cs`
   content, if you're proposing a migration to the minimal hosting model per item 5).
2. Apply exactly the mechanical fixes Step 7 lists as safe to apply — remove a
   confirmed-dead duplicate registration (not one guarded by an `IEnumerable<T>`/
   `GetServices<T>()` consumer — see item 1), reorder middleware into the correct
   sequence, flatten `UseEndpoints(...)` into top-level `Map*` calls, and convert
   interpolated log calls to message templates. Nothing beyond those four categories may
   be folded in here as a done deal, no matter how mechanical it looks — everything else
   goes through point 3 instead, and through the approval gate in Step 6 before it can
   ever reach disk.
3. For everything else — a possibly-unused service, a config key that might genuinely
   come from an environment variable, a lifetime change that could alter behavior, a
   project-level `.csproj` lever, or any other finding not in the point-2 list — do
   **not** silently remove or rewrite that line. Leave it in place and add a
   `// TODO (review, F<n>):` comment immediately above it, referencing that finding's ID,
   so the generated file stays runnable and behavior-preserving except where you've
   already confirmed a fix is safe.
4. Mark every changed line or block with a short comment naming the finding it resolves
   (e.g. `// removed (F1): duplicate registration`), so both the diff against the
   original and its link back to the findings table are self-explanatory.
5. Present this as a single fenced ```csharp code block. **Never write any of it to
   disk — not even the point-2 fixes — before Step 6.** Generating this proposal is not
   permission to apply it; it's the artifact the user reviews to decide what to approve.

## Step 6 — Stop and ask for approval

Present the Step 4 report and the Step 5 proposal together, then **stop**. Make no edit
to any file in the repository before this point — not even the findings on Step 7's safe
list. Generating the proposal is not permission to apply it.

Ask the user which findings to apply, by ID. State plainly that they can:

- approve all of them,
- approve a subset (e.g. "F1, F3, F5"),
- or decline everything,

and that anything not explicitly approved will be left untouched in the repository,
exactly as the Step 5 proposal shows it (with its `// TODO (review, F<n>):` marker, if it
has one).

Silence, ambiguity, or a partial answer is **not** approval. If the response doesn't
clearly identify which IDs to apply, ask again rather than guessing — applying an
unapproved change is the one failure mode this step exists to prevent.

## Step 7 — Applying approved fixes to disk

Approval from Step 6 is required before touching any file — nothing here happens without
it, regardless of which list a finding falls on. Once a specific ID is approved, which
list it's on determines *how* you apply it:

- **On the safe list** — duplicate-registration removal (once the `IEnumerable<T>`/
  `GetServices<T>()` check clears it), middleware reordering, `UseEndpoints` flattening,
  and interpolated-log-call conversion: apply the approved ID directly — it's exactly
  what Step 5 already showed, nothing new to decide.
- **Off the safe list** — lifetime changes, `Startup.cs` migration, blocking-call
  refactors, removing a possibly-unused service, any config or `.csproj` change:
  approving the ID means the user has made the judgment call this checklist can't make on
  its own, so it may now be applied. Before applying, restate the specific behavior
  change you're about to make, and where the fix has more than one option (e.g. *which*
  lifetime to keep for a conflicting registration, *which* config value is correct), use
  the choice the user stated when approving — don't pick for them. If they approved the
  ID without specifying which option, ask before applying rather than guessing.

Apply only the IDs that were approved. Leave every other finding's line exactly as
Step 5 proposed it — untouched, with its marker still in place.

## Step 8 — Report what was applied

After applying the approved changes, list, by ID, which were applied, and confirm each
one landed as shown in the Step 5 proposal. Then list every finding that was left
untouched and why — not approved, or approved but needing a decision the user didn't
provide. Then run the Step 9 build check and include its result in the same summary.
Never report the review as "done" without this — a user who approved 3 of 7 findings
needs to see, in one place, that the other 4 are still open, not discover it by noticing
the file wasn't fully changed.

## Step 9 — Verify

After applying any edit to a file (not just proposing one), run `dotnet build` on the
affected project (or solution) and report whether it succeeded. If you have a way to
start the app locally, note whether it still starts and reaches its "listening"/
"Application started" log line. If you can't run it locally (missing local dependencies,
a container-only environment, etc.), say so explicitly rather than claiming it works.
Never report a fix as "done" without at least the build check — an edit that doesn't
compile is worse than no edit.

## Step 10 — Measure (optional)

Where you can, make findings quantitative instead of purely qualitative:

- If you can run the app locally, capture a startup-time baseline before your changes
  (e.g. the wall-clock time to the `Application started.` log line, an
  `IHostApplicationLifetime.ApplicationStarted` callback timestamp, or a quick
  `dotnet-counters monitor --counters Microsoft.AspNetCore.Hosting` sample) and the same
  measurement after, and report the delta.
- Keep this lightweight — a couple of timed runs, not a formal benchmark suite. If the
  app can't be run locally (missing local dependencies/secrets, container-only
  environment, etc.), say so and report the findings qualitatively instead; don't block
  the rest of the report on being able to measure.
