---
name: csharp-perf-analysis
description: 'Detect inefficient C# code patterns including string concatenation in loops, boxing/unboxing, sync-over-async patterns, excessive allocations, missing IDisposable, reflection in hot paths, and inefficient collection usage. Use when analyzing C# code for performance bottlenecks.'
user-invocable: true
---

# C# Performance Analysis

## When to Use

- Analyzing C# files for performance issues
- Detecting memory allocation problems
- Finding synchronous blocking in async code
- Identifying reflection or boxing in hot paths
- Code review for performance anti-patterns

## Analysis Procedure

### 1. Scan C# Files

Use `grep_search` or `semantic_search` to find C# files in the project:
- Search for `*.cs` files
- Prioritize files in hot paths (controllers, repositories, services, background workers)

### 2. Pattern Detection

For each C# file, scan for these anti-patterns:

#### String Operations
- String concatenation in loops (`for`/`foreach`/`while` containing `+` operator on strings)
- Multiple string concatenations without `StringBuilder`
- Pattern: `str += something` inside loop body

#### LINQ Abuse
- `.ToList()` or `.ToArray()` before further filtering
- Example: `collection.ToList().Where(x => x.Condition)` → should be `collection.Where(x => x.Condition).ToList()`
- Multiple enumerations of IEnumerable without caching

#### Synchronous Over Async (Blocking)
- `.Result` or `.GetAwaiter().GetResult()` on Task
- `.Wait()` on Task in async contexts
- Look for these in `async` methods or methods called from async contexts
- Deadlock risk in ASP.NET contexts

#### Boxing/Unboxing
- Value types in collections that expect object (non-generic collections)
- `ArrayList`, `Hashtable` instead of `List<T>`, `Dictionary<TKey, TValue>`
- String interpolation with value types in hot paths (consider avoiding in tight loops)

#### Memory Allocations
- `new` keyword inside tight loops (millions of iterations)
- Closures capturing variables in hot paths
- Large value types being copied repeatedly
- Array reallocations (use `List<T>` with initial capacity)

#### Resource Management
- Classes implementing IDisposable without `using` statement or proper disposal
- File handles, database connections, HTTP clients not properly disposed
- Missing `ConfigureAwait(false)` in library code

#### Time Operations
- `DateTime.Now` in performance-critical code → use `DateTime.UtcNow` (faster, no timezone conversion)
- Repeated calls to `DateTime.Now` in same method → cache the value

#### Reflection
- `Type.GetType()`, `MethodInfo.Invoke()`, `Activator.CreateInstance()` in loops
- Reflection in hot paths without caching
- Consider compiled expressions or source generators

#### Collection Inefficiency
- Using `List<T>` with `.Contains()` for membership testing → use `HashSet<T>`
- `First()` or `Single()` without predicate followed by check → use `FirstOrDefault(predicate)`
- Linear search (`foreach` + `if`) when dictionary lookup available

### 3. Load Anti-Pattern Reference

When you find patterns, consult [csharp-antipatterns.md](./references/csharp-antipatterns.md) for:
- Detailed explanation of why the pattern is problematic
- Code examples showing bad vs good implementations
- Performance impact estimates

### 4. Rank Findings

Assign severity:
- **Critical**: Blocking operations in async code, resource leaks
- **High**: String concat in loops, N+1 database patterns via code, excessive allocations in tight loops
- **Medium**: Boxing, suboptimal LINQ, reflection in moderate-use paths
- **Low**: DateTime.Now instead of UtcNow, minor collection inefficiencies

### 5. Output Format

For each finding:

```markdown
**[Severity]** Issue in [file.cs](file.cs#L123-L125)

**Problem**: Brief description of the anti-pattern

**Impact**: Performance implication (e.g., "Allocates N string objects per loop iteration")

**Fix**: 
\`\`\`csharp
// Replace this:
string result = "";
foreach (var item in items)
{
    result += item.ToString();
}

// With this:
var sb = new StringBuilder();
foreach (var item in items)
{
    sb.Append(item.ToString());
}
string result = sb.ToString();
\`\`\`

**Estimated Improvement**: "Reduces allocations from O(N²) to O(N)"
```

## Auto-Fix Capability

You can automatically fix these patterns when requested:

- ✅ String concatenation → `StringBuilder`
- ✅ `.Result` / `.Wait()` → `await` (ensure method is async)
- ✅ Generic collections for non-generic ones
- ✅ Add `using` statements for IDisposable
- ✅ `DateTime.Now` → `DateTime.UtcNow`
- ✅ Add `ConfigureAwait(false)` in library code
- ❌ Architectural changes (require manual review)

## Examples

### Quick Scan Command

```csharp
// Search for sync-over-async anti-patterns
grep_search(".Result|.Wait\\(\\)|.GetAwaiter\\(\\).GetResult\\(\\)", isRegexp: true, includePattern: "**/*.cs")
```

### Pattern Recognition

Look for these code patterns:
1. Loop + string concatenation
2. `async` method with `.Result` or `.Wait()`
3. `IDisposable` type without `using`
4. `List<T>.Contains()` in a loop
5. `new` inside `for` loop with large iteration count

## Notes

- Always consider the execution frequency (hot path vs cold path)
- Profile before and after fixes when possible
- Some patterns are acceptable in cold paths (startup code, infrequent operations)
- Document assumptions about frequency if not obvious from code
