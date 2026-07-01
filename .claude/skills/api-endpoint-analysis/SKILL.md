---
name: api-endpoint-analysis
description: 'Analyze ASP.NET Core API endpoints for performance issues including synchronous blocking in async methods, missing response caching, database calls in loops, missing CancellationToken propagation, over-fetching entities vs DTOs, missing pagination, and lack of compression. Use when optimizing API endpoint performance.'
user-invocable: true
---

# API Endpoint Performance Analysis

## When to Use

- Slow API endpoints identified in logs or monitoring
- High response times in production
- API performance review and optimization
- Reviewing controller or minimal API implementations
- Identifying bottlenecks in HTTP request handling

## Analysis Procedure

### 1. Locate API Endpoints

Find controller and minimal API endpoints:

```csharp
// Search for controllers
grep_search("Controller|ApiController", isRegexp: true, includePattern: "**/*Controller.cs")

// Search for minimal APIs
grep_search("MapGet|MapPost|MapPut|MapDelete", isRegexp: true, includePattern: "**/Program.cs")
```

Look for:
- Classes inheriting from `ControllerBase` or `Controller`
- Methods with `[HttpGet]`, `[HttpPost]`, `[HttpPut]`, `[HttpDelete]` attributes
- Minimal API mappings (`app.MapGet`, `app.MapPost`, etc.)

### 2. Pattern Detection

For each API endpoint, scan for these anti-patterns:

#### A. Synchronous Blocking in Async Methods

**Detection**:
- `async` method signature
- Contains `.Result`, `.Wait()`, or `.GetAwaiter().GetResult()`
- Pattern: `Task<IActionResult>` with blocking calls

**Example**:
```csharp
[HttpGet("{id}")]
public async Task<IActionResult> GetUser(int id)
{
    var user = _dbContext.Users.FindAsync(id).Result; // BLOCKING!
    return Ok(user);
}
```

**Red Flag**: Deadlock risk, thread pool starvation in ASP.NET.

#### B. Missing CancellationToken Propagation

**Detection**:
- `async` endpoint method
- Calls to async operations (EF Core, HttpClient, etc.)
- No `CancellationToken` parameter or not passed to async calls

**Example**:
```csharp
[HttpGet]
public async Task<IActionResult> GetOrders()
{
    // Missing CancellationToken parameter
    var orders = await _dbContext.Orders.ToListAsync(); // Should pass cancellationToken
    return Ok(orders);
}
```

**Impact**: Client disconnection doesn't cancel in-progress work. Wastes resources.

#### C. Database Calls Inside Loops (N+1)

**Detection**:
- Loop (`foreach`, `for`) over collection
- Database query inside loop body
- Pattern: `await _dbContext...` or `_repository...` inside `foreach`

**Example**:
```csharp
[HttpGet]
public async Task<IActionResult> GetOrdersWithCustomers()
{
    var orders = await _dbContext.Orders.ToListAsync();
    
    foreach (var order in orders)
    {
        // N+1: Separate query per order
        order.Customer = await _dbContext.Customers.FindAsync(order.CustomerId);
    }
    
    return Ok(orders);
}
```

**Impact**: 1 + N database queries. Should use `.Include()`.

#### D. Over-Fetching: Returning Full Entities Instead of DTOs

**Detection**:
- Endpoint returns EF Core entity directly (not DTO)
- `return Ok(entityFromDb)`
- Entity has navigation properties or unused fields

**Example**:
```csharp
[HttpGet("{id}")]
public async Task<IActionResult> GetUser(int id)
{
    var user = await _dbContext.Users.FindAsync(id);
    return Ok(user); // Returns ALL columns, including PasswordHash, SecurityStamp, etc.
}
```

**Impact**: 
- Exposes sensitive data
- Unnecessary bandwidth (100+ fields when only 5 needed)
- JSON serialization overhead
- Potential circular reference issues

#### E. Missing Pagination on List Endpoints

**Detection**:
- Endpoint returns `IEnumerable` or `List` without pagination
- `.ToListAsync()` without `.Take()` or `.Skip()`
- Large tables (Orders, Events, Logs) without limits

**Example**:
```csharp
[HttpGet]
public async Task<IActionResult> GetProducts()
{
    var products = await _dbContext.Products.ToListAsync(); // Returns ENTIRE table
    return Ok(products);
}
```

**Impact**: 
- OOM with large tables
- Slow response times
- High bandwidth consumption

#### F. Missing Response Caching

**Detection**:
- GET endpoints with stable data
- No `[ResponseCache]` attribute
- No cache headers in response
- Data that rarely changes (reference data, product catalogs)

**Example**:
```csharp
[HttpGet]
public async Task<IActionResult> GetCategories()
{
    // No caching: hits database every request
    var categories = await _dbContext.Categories.ToListAsync();
    return Ok(categories);
}
```

**Impact**: Unnecessary database load, slow responses for cacheable data.

#### G. Missing Output Caching (ASP.NET Core 7+)

**Detection**:
- Similar to response caching but server-side
- Stable GET endpoints without `[OutputCache]`

**Example**:
```csharp
[HttpGet]
public async Task<IActionResult> GetPublicData()
{
    // Should use output caching for stable public data
    var data = await _dbContext.PublicData.ToListAsync();
    return Ok(data);
}
```

#### H. Large Payloads Without Compression

**Detection**:
- Endpoints returning large JSON payloads (> 1MB)
- No response compression middleware
- Check `Program.cs` / `Startup.cs` for `AddResponseCompression()`

**Example**:
```csharp
// Missing in Program.cs
// builder.Services.AddResponseCompression(options => { ... });
// app.UseResponseCompression();
```

**Impact**: Slow responses over network, high bandwidth costs.

#### I. Synchronous I/O Operations

**Detection**:
- File operations without `Async` suffix
- `File.ReadAllText()`, `File.WriteAllText()`, `Stream.Read()`
- Should use `ReadAllTextAsync()`, `WriteAllTextAsync()`, `ReadAsync()`

**Example**:
```csharp
[HttpPost("upload")]
public IActionResult UploadFile(IFormFile file)
{
    var content = new StreamReader(file.OpenReadStream()).ReadToEnd(); // SYNC!
    // ... process
    return Ok();
}
```

#### J. Missing Model Validation

**Detection**:
- POST/PUT endpoints without `[ApiController]` attribute (auto-validates)
- Or missing `if (!ModelState.IsValid)` check
- Invalid data processed without validation

**Example**:
```csharp
[HttpPost]
public async Task<IActionResult> CreateUser(UserDto user)
{
    // Missing validation check
    await _dbContext.Users.AddAsync(new User { ... });
    await _dbContext.SaveChangesAsync();
    return Ok();
}
```

### 3. Performance Metrics Analysis

If available, review:
- Response time distribution (P50, P95, P99)
- Throughput (requests per second)
- Database query counts per endpoint
- Memory allocations per request
- Failed request rate

**Sources**:
- Application Insights telemetry
- Logs with execution times
- APM tools (New Relic, Datadog)
- Middleware timing logs

### 4. Severity Ranking

**Critical**:
- Synchronous blocking causing deadlocks
- Missing pagination on tables with millions of rows
- Security issues (returning sensitive data)

**High**:
- N+1 database queries in loops
- Missing cancellation token on long-running operations
- No caching on frequently-accessed stable data

**Medium**:
- Over-fetching (returning entities vs DTOs)
- Missing response compression for large payloads
- Synchronous file I/O

**Low**:
- Minor caching opportunities
- Missing model validation (if inputs are trusted)

### 5. Output Format

```markdown
### API Endpoint Analysis: GET /api/orders

**Current Performance**:
- Average response time: 3.5 seconds
- P95: 8.2 seconds
- Database queries per request: 101
- Payload size: 2.5 MB (uncompressed)

---

**Issue 1: N+1 Query Pattern [Critical]**

**Location**: [OrdersController.cs](src/Controllers/OrdersController.cs#L45-L55)

**Problem**: Lazy loading customer data inside loop.

\`\`\`csharp
var orders = await _dbContext.Orders.ToListAsync();

foreach (var order in orders)
{
    order.Customer = await _dbContext.Customers.FindAsync(order.CustomerId); // N+1!
}
\`\`\`

**Fix**:
\`\`\`csharp
var orders = await _dbContext.Orders
    .Include(o => o.Customer) // Eager load
    .ToListAsync();
\`\`\`

**Estimated Improvement**: 101 queries → 1 query. Response time: 3.5s → 0.3s (10x faster).

---

**Issue 2: Missing Pagination [Critical]**

**Problem**: Returns all orders without limit.

**Fix**:
\`\`\`csharp
[HttpGet]
public async Task<IActionResult> GetOrders(
    [FromQuery] int page = 1,
    [FromQuery] int pageSize = 50)
{
    var orders = await _dbContext.Orders
        .Include(o => o.Customer)
        .OrderByDescending(o => o.OrderDate)
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .ToListAsync();
    
    var total = await _dbContext.Orders.CountAsync();
    
    return Ok(new 
    {
        Items = orders,
        TotalCount = total,
        Page = page,
        PageSize = pageSize
    });
}
\`\`\`

---

**Issue 3: Over-Fetching Entities [High]**

**Problem**: Returns full Order entity with all properties.

**Fix**: Use DTO with projection:
\`\`\`csharp
var orders = await _dbContext.Orders
    .Include(o => o.Customer)
    .Select(o => new OrderDto
    {
        OrderId = o.OrderId,
        OrderDate = o.OrderDate,
        CustomerName = o.Customer.Name,
        Total = o.Total
    })
    .ToListAsync();
\`\`\`

**Impact**: Reduces payload size by 60%, faster serialization.

---

**Issue 4: Missing Response Caching [Medium]**

For stable data like categories:
\`\`\`csharp
[HttpGet("categories")]
[ResponseCache(Duration = 3600)] // Cache for 1 hour
public async Task<IActionResult> GetCategories()
{
    var categories = await _dbContext.Categories.ToListAsync();
    return Ok(categories);
}
\`\`\`

Or use output caching (ASP.NET Core 7+):
\`\`\`csharp
[HttpGet("categories")]
[OutputCache(Duration = 3600)]
public async Task<IActionResult> GetCategories() { ... }
\`\`\`
```

### 6. Consult Reference

Load [api-antipatterns.md](./references/api-antipatterns.md) for detailed examples and fixes for each pattern.

## Auto-Fix Capability

Can automatically fix:
- ✅ Add `.Include()` for navigation properties
- ✅ Replace `.Result` / `.Wait()` with `await`
- ✅ Add `CancellationToken` parameter to endpoints
- ✅ Add pagination (with configurable page size)
- ✅ Add `[ResponseCache]` or `[OutputCache]` attributes
- ✅ Replace sync file I/O with async equivalents
- ❌ Complete DTO mapping (requires domain knowledge)
- ❌ Complex caching strategies (requires business rules)

## Integration with Other Skills

- After `/efcore-query-analysis`: Many API issues stem from EF Core misuse
- Before `/db-query-optimization`: Fix application-level N+1 before DB-level optimization
- Feed findings into `/perf-report-generator`

## Notes

- Always measure before and after fixes (use logging/APM)
- Test pagination with realistic data volumes
- Consider rate limiting for public APIs
- DTOs are critical for API security and performance
- Caching must respect data freshness requirements
