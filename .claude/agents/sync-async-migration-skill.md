# Sync-to-Async Migration Agent

Analyze the codebase (or the specific files/classes passed as arguments) and perform a complete synchronous-to-asynchronous migration. Produce both the migrated code **and** the three required deliverables below.

Arguments (optional): `$ARGUMENTS`
- If a file path, class name, or directory is provided, scope the analysis to that target.
- If a language/framework is mentioned (e.g. "C#", "Java Spring", "Node.js"), apply its idiomatic async patterns.
- If no arguments are given, infer the language from the project and scan the entire working directory.

---

## Required deliverables (always produce all three)

After migration (or after analysis if no AUTO edits apply), the agent **must** produce all of the following:

### 1. Suggest CancellationToken where appropriate
- Add `CancellationToken` (C# / .NET: `CancellationToken cancellationToken = default`) to every converted async method signature.
- Thread the token into every awaitable API that accepts one (`Task.Delay`, `*Async` I/O, HTTP clients, etc.).
- Wire tokens to the correct lifetime at the edge:
  - ASP.NET / HTTP handlers → request `CancellationToken`
  - Background services / hosted jobs → stopping / job token
  - Library / domain methods → `= default` so callers decide
- In the migration report, list each method that received a token and how it is wired (see **CANCELLATION TOKEN GUIDANCE**).

### 2. Generate a migration report explaining every change
- After all edits, produce the structured **Migration Report** in Phase 3 format.
- Every AUTO-migrated method must appear under **MIGRATED METHODS** with Before / After / Notes.
- Print the report to the terminal when edits are complete.
- Optionally write the same report to a project docs file (e.g. `Docs/MigrationReport.md`) when useful for the user.

### 3. Highlight methods that cannot be safely converted automatically
- Classify every finding as `AUTO` or `MANUAL` in Phase 1.
- Do **not** silently force-convert `MANUAL` items; leave them (or add `// TODO: async-migrate`) and list each under **METHODS REQUIRING MANUAL REVIEW** with Reason + Suggestion.
- Typical MANUAL cases: locks over shared mutable state, sync-only third-party SDKs, constructor / static constructor / property-getter I/O, finalizers, void event handlers / delegates, callers out of requested scope.

---

## Phase 1 — Reconnaissance

Before writing a single line of code, complete every step below.

1. **Identify the language and framework** from file extensions, build files (`pom.xml`, `build.gradle`, `*.csproj`, `package.json`, etc.), and imports.

2. **Locate all synchronous entry points** — search for:
   - C# / .NET: `.Result`, `.Wait()`, `.GetAwaiter().GetResult()`, `Thread.Sleep()`, `Task.Run()` wrapping sync work, blocking stream reads, any method that *could* be `async` but is not.
   - Java: `Thread.sleep()`, `synchronized` blocks used for I/O, `Future.get()`, `CompletableFuture` chains replaced with blocking gets, RestTemplate calls, JDBC calls not wrapped in async.
   - Node.js / JS / TS: callback-based APIs, `fs.readFileSync`, `execSync`, any blocking loop over I/O.
   - Python: synchronous `requests`, `time.sleep` inside I/O code, missing `async def` / `await`.
   - Adapt for any other language found.

3. **Build the call graph** — for every sync method detected, trace callers up the stack to find the full propagation chain. Note which callers must also become async.

4. **Classify each finding** into one of:
   - `AUTO` — safe to convert automatically (clear async alternative exists, no side effects)
   - `MANUAL` — cannot be safely converted without human review (e.g. locks protecting shared mutable state, third-party libraries with no async API, static constructors, finalizers, event handlers with void return constraints)

5. **Read every file that will be changed** in full before proposing any edits — never guess at method signatures or class structure.

6. **Plan CancellationToken placement** — for each AUTO method, decide where the token parameter is added and which edge (HTTP, background job, or caller-default) will own the lifetime.

---

## Phase 2 — Migration

Apply changes **file by file**, in dependency order (leaf methods first, callers second).

For each `AUTO`-classified method:

### C# / .NET rules
- Add `async` modifier; change return type: `void` → `async void` only for event handlers (flag these), `T` → `Task<T>`, `void` (non-event) → `Task`.
- Replace `.Result` / `.Wait()` / `.GetAwaiter().GetResult()` with `await`.
- Replace `Task.Run(SyncWork)` misuse — if the work is truly CPU-bound keep `Task.Run` but document it; if it was wrapping I/O, replace with the native async API.
- **Suggest / add CancellationToken where appropriate:** add `CancellationToken cancellationToken = default` (or `ct = default`) to every converted method signature and thread it through to every awaitable call that accepts one.
- Preserve `try/catch` blocks — `async` methods surface exceptions through the returned `Task`; ensure callers that previously caught synchronous exceptions still catch them at the right `await` site.
- Replace `Thread.Sleep(ms)` with `await Task.Delay(ms, cancellationToken)`.
- Use `ConfigureAwait(false)` in library code; omit it in ASP.NET Core controllers/handlers (context is not needed).

### Java rules
- Prefer `CompletableFuture<T>` / reactive (`Mono<T>`, `Flux<T>`) depending on what the project already uses.
- Replace blocking `Future.get()` with `.thenApply()` / `.thenCompose()` chains or `await` equivalents.
- Replace `RestTemplate` with `WebClient` (Spring WebFlux) where applicable.
- Preserve `@Transactional` boundaries — note when an async boundary breaks transaction propagation and flag for manual review.
- Propagate cancellation via `CancellationToken` / reactor cancel signals where the stack already supports them; otherwise document as MANUAL.

### Node.js / JS / TS rules
- Convert callbacks to `Promise`-returning functions; then convert call sites to `async/await`.
- Replace `*Sync` fs/child_process APIs with their promise equivalents (`fs.promises.*`, `util.promisify`).
- Ensure `async` propagates up to Express/Fastify route handlers and that errors are passed to `next(err)` or caught properly.
- Suggest `AbortSignal` / `AbortController` where I/O APIs support cancellation.

### Python rules
- Add `async def`; replace `requests` with `httpx` or `aiohttp`; replace `time.sleep` with `asyncio.sleep`.
- Ensure the top-level caller uses `asyncio.run()` or an event loop.
- Suggest passing / checking cancellation (`asyncio.CancelledError`, timeout scopes) where appropriate.

For each `MANUAL`-classified method:
- Do not force an unsafe conversion.
- Leave the original implementation (or the minimum compile-safe shim).
- Add `// TODO: async-migrate` (or language-equivalent) with a short reason.
- Include the method in **METHODS REQUIRING MANUAL REVIEW** with Reason + Suggestion.

**Do not change:**
- Method names, public API contracts, or return semantics (beyond the wrapping type).
- Business logic inside the method body.
- Formatting or unrelated code in the same file.

---

## Phase 3 — Migration Report

After all edits are applied, produce a **Migration Report** in this exact format. The report must explain every AUTO change and highlight every MANUAL method.

```
=============================================================
  SYNC-TO-ASYNC MIGRATION REPORT
=============================================================

SUMMARY
-------
Files changed   : <n>
Methods migrated: <n>  (AUTO: <n> | MANUAL review needed: <n>)
CancellationToken added: <n> signatures

MIGRATED METHODS  (AUTO)
------------------------
<File> :: <ClassName.MethodName>
  Before : <one-line description of the sync pattern>
  After  : <one-line description of the async replacement>
  Notes  : <any noteworthy detail — e.g. ConfigureAwait(false) added>
  CancellationToken : <HTTP request | background job | default for caller | n/a>

... (repeat for each migrated method)

METHODS REQUIRING MANUAL REVIEW
--------------------------------
<File> :: <ClassName.MethodName>
  Reason : <exact reason it cannot be auto-converted>
  Suggestion : <what a developer should do manually>

... (repeat for each MANUAL item)

CANCELLATION TOKEN GUIDANCE
----------------------------
<List each method where a CancellationToken was threaded through,
 and note whether the token is wired to an HTTP request lifetime,
 a background job, or left as `default` for the caller to decide.>

EXCEPTION HANDLING NOTES
-------------------------
<List any places where the async conversion changed exception
 propagation semantics and what the caller must now do.>

WHAT WAS NOT CHANGED AND WHY
-----------------------------
<Briefly list any sync code that was intentionally left alone,
 e.g. CPU-bound work correctly using Task.Run, or sync utilities
 with no I/O.>

=============================================================
```

Print the report to the terminal after all file edits are complete.

---

## Constraints

- **Never modify production code that is out of scope** — if only a specific file was requested, do not touch callers in other files; instead list them in the report under "METHODS REQUIRING MANUAL REVIEW" with reason "caller not in scope."
- **Never break compilation** — if a change would cause a compile error you cannot resolve (e.g. interface you cannot modify), leave the original code, apply `// TODO: async-migrate` comment, and report it as MANUAL.
- **Preserve all existing tests** — if a test calls a method synchronously and the method signature changes, update the test call site too.
- **One commit-ready change** — all edits must leave the codebase in a buildable, logically consistent state when taken together.
