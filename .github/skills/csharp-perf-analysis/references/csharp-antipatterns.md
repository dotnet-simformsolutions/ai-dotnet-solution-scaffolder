# C# Performance Anti-Patterns

This reference provides detailed examples of common C# performance anti-patterns with explanations and fixes.

---

## 1. String Concatenation in Loops

### ❌ Bad
```csharp
string result = "";
foreach (var item in items)
{
    result += item.Name + ", "; // Creates new string each iteration
}
```

### ✅ Good
```csharp
var sb = new StringBuilder();
foreach (var item in items)
{
    sb.Append(item.Name).Append(", ");
}
string result = sb.ToString();
```

**Why**: String concatenation with `+` creates a new string object each time (strings are immutable). With N items, this creates O(N²) allocations.

**Impact**: 100 items = ~5,000 allocations. 1,000 items = ~500,000 allocations.

---

## 2. LINQ ToList() Before Filtering

### ❌ Bad
```csharp
var activeUsers = users
    .ToList() // Materializes entire collection
    .Where(u => u.IsActive)
    .ToList();
```

### ✅ Good
```csharp
var activeUsers = users
    .Where(u => u.IsActive) // Deferred execution
    .ToList(); // Materialize only filtered results
```

**Why**: `.ToList()` forces immediate execution and allocates memory for entire collection before filtering.

**Impact**: 10,000 users → 10,000 allocations, even if only 100 are active. Fixed version → 100 allocations.

---

## 3. Synchronous Blocking in Async Code (Deadlock Risk)

### ❌ Bad
```csharp
public async Task<User> GetUserAsync(int id)
{
    var response = httpClient.GetAsync($"/users/{id}").Result; // DEADLOCK RISK!
    return await response.Content.ReadAsAsync<User>();
}
```

### ✅ Good
```csharp
public async Task<User> GetUserAsync(int id)
{
    var response = await httpClient.GetAsync($"/users/{id}");
    return await response.Content.ReadAsAsync<User>();
}
```

**Why**: `.Result` blocks the thread waiting for task completion. In ASP.NET, this can deadlock because the synchronization context is captured.

**Impact**: Deadlocks in production, thread pool starvation, reduced throughput.

---

## 4. Task.Wait() in Async Context

### ❌ Bad
```csharp
public async Task ProcessOrdersAsync()
{
    var orders = await GetOrdersAsync();
    foreach (var order in orders)
    {
        ProcessOrderAsync(order).Wait(); // Blocks thread!
    }
}
```

### ✅ Good
```csharp
public async Task ProcessOrdersAsync()
{
    var orders = await GetOrdersAsync();
    foreach (var order in orders)
    {
        await ProcessOrderAsync(order); // Properly async
    }
}
```

Or for parallel processing:
```csharp
public async Task ProcessOrdersAsync()
{
    var orders = await GetOrdersAsync();
    await Task.WhenAll(orders.Select(o => ProcessOrderAsync(o)));
}
```

---

## 5. Boxing in Hot Paths

### ❌ Bad
```csharp
ArrayList numbers = new ArrayList(); // Non-generic collection
for (int i = 0; i < 10000; i++)
{
    numbers.Add(i); // Boxing: int → object
}

int sum = 0;
foreach (object num in numbers)
{
    sum += (int)num; // Unboxing: object → int
}
```

### ✅ Good
```csharp
List<int> numbers = new List<int>();
for (int i = 0; i < 10000; i++)
{
    numbers.Add(i); // No boxing
}

int sum = 0;
foreach (int num in numbers)
{
    sum += num; // No unboxing
}
```

**Why**: Boxing allocates heap memory for value type. Each box/unbox is an allocation and dereference.

**Impact**: 10,000 items = 10,000 unnecessary allocations + GC pressure.

---

## 6. Missing IDisposable/Using

### ❌ Bad
```csharp
public void ProcessFile(string path)
{
    var stream = File.OpenRead(path);
    // ... process file
    // File handle leaked if exception occurs!
}
```

### ✅ Good
```csharp
public void ProcessFile(string path)
{
    using var stream = File.OpenRead(path);
    // ... process file
    // Automatically disposed even if exception
}
```

Or traditional:
```csharp
public void ProcessFile(string path)
{
    using (var stream = File.OpenRead(path))
    {
        // ... process file
    }
}
```

**Why**: Unmanaged resources (file handles, network connections, database connections) must be explicitly released.

**Impact**: Resource exhaustion, "too many open files" errors, database connection pool exhaustion.

---

## 7. Excessive Allocations in Loops

### ❌ Bad
```csharp
for (int i = 0; i < 1000000; i++)
{
    var temp = new SomeClass(); // 1 million allocations
    temp.DoWork();
}
```

### ✅ Good
```csharp
var reusable = new SomeClass();
for (int i = 0; i < 1000000; i++)
{
    reusable.Reset(); // Reuse instance
    reusable.DoWork();
}
```

Or use object pooling:
```csharp
var pool = new ObjectPool<SomeClass>(() => new SomeClass());
for (int i = 0; i < 1000000; i++)
{
    var temp = pool.Get();
    try
    {
        temp.DoWork();
    }
    finally
    {
        pool.Return(temp);
    }
}
```

---

## 8. DateTime.Now in Performance-Critical Code

### ❌ Bad
```csharp
public void ProcessRecords(List<Record> records)
{
    foreach (var record in records)
    {
        record.ProcessedAt = DateTime.Now; // Timezone conversion overhead
        // ... process record
    }
}
```

### ✅ Good
```csharp
public void ProcessRecords(List<Record> records)
{
    var now = DateTime.UtcNow; // Cache and use UTC
    foreach (var record in records)
    {
        record.ProcessedAt = now;
        // ... process record
    }
}
```

**Why**: `DateTime.Now` performs timezone conversions. `DateTime.UtcNow` is faster and avoids timezone issues.

**Impact**: In tight loops, 2-3x slower. Also prevents timezone-related bugs.

---

## 9. Reflection in Hot Paths

### ❌ Bad
```csharp
public void ProcessItems(List<object> items)
{
    foreach (var item in items)
    {
        var method = item.GetType().GetMethod("Process"); // Reflection every iteration
        method.Invoke(item, null);
    }
}
```

### ✅ Good (Compiled Expression)
```csharp
private static readonly ConcurrentDictionary<Type, Action<object>> _compiledMethods = new();

public void ProcessItems(List<object> items)
{
    foreach (var item in items)
    {
        var action = _compiledMethods.GetOrAdd(item.GetType(), type =>
        {
            var method = type.GetMethod("Process");
            var param = Expression.Parameter(typeof(object));
            var call = Expression.Call(
                Expression.Convert(param, type),
                method
            );
            return Expression.Lambda<Action<object>>(call, param).Compile();
        });
        
        action(item);
    }
}
```

Or use interface:
```csharp
public interface IProcessable
{
    void Process();
}

public void ProcessItems(List<IProcessable> items)
{
    foreach (var item in items)
    {
        item.Process(); // Direct call, no reflection
    }
}
```

---

## 10. Inefficient Collection Membership Testing

### ❌ Bad
```csharp
List<string> allowedUsers = GetAllowedUsers(); // 10,000 users

foreach (var request in requests)
{
    if (allowedUsers.Contains(request.UserId)) // O(N) linear search each time
    {
        ProcessRequest(request);
    }
}
```

### ✅ Good
```csharp
HashSet<string> allowedUsers = GetAllowedUsers().ToHashSet(); // O(1) lookups

foreach (var request in requests)
{
    if (allowedUsers.Contains(request.UserId)) // O(1) hash lookup
    {
        ProcessRequest(request);
    }
}
```

**Why**: `List<T>.Contains()` is O(N) linear search. `HashSet<T>.Contains()` is O(1) average case.

**Impact**: 10,000 users × 1,000 requests = 10 million comparisons vs 1,000 hash lookups.

---

## 11. Closures Capturing in Loops

### ❌ Bad
```csharp
var actions = new List<Action>();
for (int i = 0; i < 10; i++)
{
    actions.Add(() => Console.WriteLine(i)); // Closure captures loop variable
}

// All print "10" instead of 0-9
foreach (var action in actions)
{
    action();
}
```

### ✅ Good
```csharp
var actions = new List<Action>();
for (int i = 0; i < 10; i++)
{
    int captured = i; // Capture local copy
    actions.Add(() => Console.WriteLine(captured));
}

foreach (var action in actions)
{
    action(); // Prints 0-9 correctly
}
```

**Why**: Closures capture variables by reference. Loop variable is shared across iterations.

---

## 12. Large Value Type Copies

### ❌ Bad
```csharp
public struct LargeStruct // 128 bytes
{
    public long Field1, Field2, Field3, Field4;
    public long Field5, Field6, Field7, Field8;
    public long Field9, Field10, Field11, Field12;
    public long Field13, Field14, Field15, Field16;
}

public void Process(LargeStruct data) // Copied on every call
{
    // ... use data
}

foreach (var item in items)
{
    Process(item); // Copy 128 bytes each iteration
}
```

### ✅ Good
```csharp
// Option 1: Use readonly struct with ref
public void Process(in LargeStruct data) // Pass by reference
{
    // ... use data (read-only)
}

// Option 2: Make it a class if mutation needed
public class LargeData
{
    // ... same fields
}
```

**Why**: Value types are copied on method calls and assignments. Large structs cause excessive copying.

**Rule of Thumb**: Structs should be ≤ 16 bytes or passed by `ref`/`in`.

---

## Summary Table

| Anti-Pattern | Severity | Typical Impact | Auto-Fix |
|--------------|----------|----------------|----------|
| String concat in loops | High | O(N²) allocations | ✅ Yes |
| ToList before filter | Medium | 2-100x memory | ✅ Yes |
| .Result / .Wait() | Critical | Deadlocks | ✅ Yes |
| Boxing in collections | Medium | N allocations | ✅ Yes |
| Missing using/Dispose | Critical | Resource leaks | ✅ Yes |
| DateTime.Now | Low | 2-3x slower | ✅ Yes |
| Reflection in loops | High | 10-100x slower | ❌ Manual |
| List.Contains vs HashSet | High | O(N) → O(1) | ✅ Yes |
| Large struct copying | Medium | Cache misses | ❌ Manual |

---

## Hot Path vs Cold Path

Not all anti-patterns are equally critical:

**Hot Paths** (execute frequently):
- API request handlers
- Background job loops processing thousands of items
- Real-time data processing
- Inner loops with high iteration counts

→ Optimize aggressively

**Cold Paths** (execute rarely):
- Application startup
- One-time configuration
- Infrequent admin operations

→ Clarity over micro-optimization
