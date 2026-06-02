---
description: "Use when editing C# files. Provides lightweight performance hints and best practices for C# code including async/await patterns, memory management, LINQ optimization, and common anti-patterns to avoid."
applyTo: "**/*.cs"
---

# C# Performance Guidelines

When editing C# code, keep these performance best practices in mind:

## Async/Await

- **Always use `async`/`await`** for I/O operations (database, HTTP, files)
- **Never block on async code** with `.Result`, `.Wait()`, or `.GetAwaiter().GetResult()`
- **Propagate `CancellationToken`** to all async methods for cancellation support
- **Use `ConfigureAwait(false)`** in library code to avoid context capture

## String Operations

- **Use `StringBuilder`** for concatenation in loops
- **Cache repeated string operations** (e.g., `ToLower()` results)
- **Prefer string interpolation** over concatenation for readability, but avoid in tight loops

## LINQ and Collections

- **Filter before materializing**: `.Where()` before `.ToList()` or `.ToArray()`
- **Use appropriate collection types**:
  - `HashSet<T>` for membership testing (not `List<T>.Contains()`)
  - `Dictionary<TKey, TValue>` for lookups (not `List<T>.Find()`)
- **Avoid multiple enumerations** - materialize once if needed multiple times
- **Consider `IAsyncEnumerable<T>`** for streaming large datasets

## Memory Management

- **Use `using` statements** for `IDisposable` types (files, connections, `HttpClient` requests)
- **Avoid allocations in hot paths**:
  - Don't create objects inside tight loops
  - Reuse instances when safe
  - Consider object pooling (`ObjectPool<T>`)
- **Prefer value types** for small, immutable data (but keep < 16 bytes)
- **Use `Span<T>` and `Memory<T>`** for performance-critical operations

## Entity Framework Core

- **Eager load navigation properties** with `.Include()` to avoid N+1 queries
- **Use `AsNoTracking()`** for read-only queries
- **Project to DTOs** with `.Select()` instead of fetching full entities
- **Add pagination** with `.Skip()` and `.Take()` on large queries
- **Avoid client-side evaluation** - filter in database, not in memory

## API Controllers

- **Always use `async Task<IActionResult>`** for endpoints
- **Add `CancellationToken` parameter** to long-running operations
- **Return DTOs, not entities** - never expose EF Core entities directly
- **Add pagination** to all list endpoints
- **Use response caching** for stable data with `[ResponseCache]` or `[OutputCache]`

## Common Anti-Patterns to Avoid

❌ String concatenation in loops  
❌ `.Result` or `.Wait()` on `Task`  
❌ `DateTime.Now` (use `DateTime.UtcNow`)  
❌ Reflection in hot paths  
❌ Boxing value types unnecessarily  
❌ `SELECT *` in queries  
❌ Missing `using` on `IDisposable`  
❌ Lazy loading in loops  

## Quick Wins

✅ Replace `+` in loops with `StringBuilder`  
✅ Add `.Include()` to avoid N+1 queries  
✅ Add `AsNoTracking()` to read-only EF queries  
✅ Use `async`/`await` instead of blocking  
✅ Add `using` for file/database/HTTP operations  
✅ Cache `DateTime.UtcNow` if called repeatedly  

---

For detailed analysis, invoke the **Performance Tuning Advisor** agent or use `/csharp-perf-analysis` skill.
