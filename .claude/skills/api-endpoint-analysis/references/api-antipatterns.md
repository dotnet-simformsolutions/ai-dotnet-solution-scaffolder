# API Endpoint Performance Anti-Patterns

Comprehensive guide to common ASP.NET Core API performance issues with fixes.

---

## 1. Synchronous Blocking in Async Methods

### ❌ Bad: Using .Result or .Wait()

```csharp
[HttpGet("{id}")]
public async Task<IActionResult> GetUser(int id)
{
    // DEADLOCK RISK in ASP.NET!
    var user = _dbContext.Users.FindAsync(id).Result;
    var orders = _httpClient.GetStringAsync($"/api/orders?userId={id}").Result;
    
    return Ok(new { user, orders });
}
```

**Impact**: Thread pool exhaustion, deadlocks, reduced scalability.

### ✅ Good: Proper Async/Await

```csharp
[HttpGet("{id}")]
public async Task<IActionResult> GetUser(int id)
{
    var user = await _dbContext.Users.FindAsync(id);
    var orders = await _httpClient.GetStringAsync($"/api/orders?userId={id}");
    
    return Ok(new { user, orders });
}
```

---

## 2. Missing Cancellation Token Propagation

### ❌ Bad: No Cancellation Support

```csharp
[HttpGet]
public async Task<IActionResult> GetLargeDataset()
{
    // Client disconnects but query continues!
    var data = await _dbContext.LargeTable
        .Where(x => x.IsActive)
        .ToListAsync(); // No cancellation token
    
    // Expensive processing continues even if client disconnected
    var processed = ProcessData(data);
    
    return Ok(processed);
}
```

### ✅ Good: Propagate Cancellation Token

```csharp
[HttpGet]
public async Task<IActionResult> GetLargeDataset(CancellationToken cancellationToken)
{
    var data = await _dbContext.LargeTable
        .Where(x => x.IsActive)
        .ToListAsync(cancellationToken); // Respects client cancellation
    
    cancellationToken.ThrowIfCancellationRequested();
    
    var processed = await ProcessDataAsync(data, cancellationToken);
    
    return Ok(processed);
}
```

**Impact**: Saves resources when client disconnects, improves scalability under load.

---

## 3. N+1 Query in API Endpoint

### ❌ Bad: Lazy Loading in Loop

```csharp
[HttpGet]
public async Task<IActionResult> GetOrders()
{
    var orders = await _dbContext.Orders.ToListAsync();
    
    foreach (var order in orders)
    {
        // N+1: Separate query for each order
        order.Customer = await _dbContext.Customers.FindAsync(order.CustomerId);
        order.Items = await _dbContext.OrderItems
            .Where(i => i.OrderId == order.OrderId)
            .ToListAsync();
    }
    
    return Ok(orders);
}
```

**Impact**: 100 orders → 201 database queries (1 + 100 customers + 100 item lists).

### ✅ Good: Eager Loading

```csharp
[HttpGet]
public async Task<IActionResult> GetOrders()
{
    var orders = await _dbContext.Orders
        .Include(o => o.Customer)
        .Include(o => o.Items)
        .ToListAsync(); // Single query (or 3 split queries with AsSplitQuery)
    
    return Ok(orders);
}
```

**Impact**: 201 queries → 1 query (or 3 with split). 50-100x faster.

---

## 4. Over-Fetching: Returning Entities Instead of DTOs

### ❌ Bad: Returning EF Core Entities

```csharp
[HttpGet("{id}")]
public async Task<IActionResult> GetUser(int id)
{
    var user = await _dbContext.Users
        .Include(u => u.Orders)
        .FirstOrDefaultAsync(u => u.Id == id);
    
    // Returns EVERYTHING: PasswordHash, SecurityStamp, all properties
    return Ok(user);
}
```

**Problems**:
- Exposes sensitive data (PasswordHash, internal IDs)
- Massive payloads (all columns, navigation properties)
- Circular reference risks
- JSON serialization overhead

### ✅ Good: Use DTOs with Projection

```csharp
public class UserDto
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
    public List<OrderSummaryDto> RecentOrders { get; set; }
}

public class OrderSummaryDto
{
    public int OrderId { get; set; }
    public DateTime OrderDate { get; set; }
    public decimal Total { get; set; }
}

[HttpGet("{id}")]
public async Task<IActionResult> GetUser(int id)
{
    var user = await _dbContext.Users
        .Where(u => u.Id == id)
        .Select(u => new UserDto
        {
            Id = u.Id,
            Name = u.Name,
            Email = u.Email,
            RecentOrders = u.Orders
                .OrderByDescending(o => o.OrderDate)
                .Take(5)
                .Select(o => new OrderSummaryDto
                {
                    OrderId = o.OrderId,
                    OrderDate = o.OrderDate,
                    Total = o.Total
                })
                .ToList()
        })
        .FirstOrDefaultAsync();
    
    if (user == null)
        return NotFound();
    
    return Ok(user);
}
```

**Impact**: 
- Payload size reduced by 70-90%
- Faster JSON serialization
- Secure (no sensitive data leakage)
- Database fetches only needed columns

---

## 5. Missing Pagination on List Endpoints

### ❌ Bad: No Pagination

```csharp
[HttpGet]
public async Task<IActionResult> GetProducts()
{
    var products = await _dbContext.Products.ToListAsync();
    return Ok(products);
}
```

**Impact**: 1 million products → OOM, 10+ second response times.

### ✅ Good: Pagination with Metadata

```csharp
public class PagedResult<T>
{
    public List<T> Items { get; set; }
    public int TotalCount { get; set; }
    public int Page { get; set; }
    public int PageSize { get; set; }
    public int TotalPages => (int)Math.Ceiling((double)TotalCount / PageSize);
}

[HttpGet]
public async Task<IActionResult> GetProducts(
    [FromQuery] int page = 1,
    [FromQuery] int pageSize = 50,
    [FromQuery] string search = null)
{
    if (pageSize > 100)
        pageSize = 100; // Max page size limit
    
    var query = _dbContext.Products.AsQueryable();
    
    if (!string.IsNullOrEmpty(search))
        query = query.Where(p => p.Name.Contains(search));
    
    var total = await query.CountAsync();
    
    var items = await query
        .OrderBy(p => p.Name)
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .ToListAsync();
    
    return Ok(new PagedResult<Product>
    {
        Items = items,
        TotalCount = total,
        Page = page,
        PageSize = pageSize
    });
}
```

---

## 6. Missing Response Caching

### ❌ Bad: No Caching for Stable Data

```csharp
[HttpGet("categories")]
public async Task<IActionResult> GetCategories()
{
    // Hits database every single request
    var categories = await _dbContext.Categories
        .OrderBy(c => c.Name)
        .ToListAsync();
    
    return Ok(categories);
}
```

**Impact**: Unnecessary database load for data that changes once per month.

### ✅ Good: Response Caching

**Option 1: Response Cache (client + proxy caching)**:
```csharp
[HttpGet("categories")]
[ResponseCache(Duration = 3600, Location = ResponseCacheLocation.Any)]
public async Task<IActionResult> GetCategories()
{
    var categories = await _dbContext.Categories
        .OrderBy(c => c.Name)
        .ToListAsync();
    
    return Ok(categories);
}
```

**Option 2: Output Cache (ASP.NET Core 7+ server-side)**:
```csharp
// In Program.cs
builder.Services.AddOutputCache();
app.UseOutputCache();

// In controller
[HttpGet("categories")]
[OutputCache(Duration = 3600)]
public async Task<IActionResult> GetCategories()
{
    var categories = await _dbContext.Categories
        .OrderBy(c => c.Name)
        .ToListAsync();
    
    return Ok(categories);
}
```

**Option 3: In-Memory Cache with IMemoryCache**:
```csharp
private readonly IMemoryCache _cache;

[HttpGet("categories")]
public async Task<IActionResult> GetCategories()
{
    if (!_cache.TryGetValue("categories", out List<Category> categories))
    {
        categories = await _dbContext.Categories
            .OrderBy(c => c.Name)
            .ToListAsync();
        
        _cache.Set("categories", categories, TimeSpan.FromHours(1));
    }
    
    return Ok(categories);
}
```

**Impact**: 1000 requests → 1 database call. ~99% reduction in database load.

---

## 7. Missing Response Compression

### ❌ Bad: No Compression Configuration

```csharp
// Program.cs - missing compression middleware
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
var app = builder.Build();
app.UseRouting();
app.MapControllers();
app.Run();
```

**Impact**: 2 MB JSON payload sent uncompressed over network.

### ✅ Good: Enable Response Compression

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true; // Important for HTTPS
    options.Providers.Add<GzipCompressionProvider>();
    options.Providers.Add<BrotliCompressionProvider>();
    options.MimeTypes = ResponseCompressionDefaults.MimeTypes.Concat(
        new[] { "application/json" });
});

builder.Services.Configure<GzipCompressionProviderOptions>(options =>
{
    options.Level = System.IO.Compression.CompressionLevel.Fastest;
});

builder.Services.Configure<BrotliCompressionProviderOptions>(options =>
{
    options.Level = System.IO.Compression.CompressionLevel.Fastest;
});

var app = builder.Build();
app.UseResponseCompression(); // Must be early in pipeline
app.UseRouting();
app.MapControllers();
app.Run();
```

**Impact**: 2 MB payload → 200 KB (10x reduction). Significantly faster responses over network.

---

## 8. Synchronous File I/O

### ❌ Bad: Blocking File Operations

```csharp
[HttpPost("upload")]
public IActionResult UploadDocument(IFormFile file)
{
    var path = Path.Combine("uploads", file.FileName);
    
    using (var stream = new FileStream(path, FileMode.Create))
    {
        file.CopyTo(stream); // BLOCKS thread!
    }
    
    var content = File.ReadAllText(path); // BLOCKS thread!
    
    return Ok(new { FileName = file.FileName, Size = content.Length });
}
```

### ✅ Good: Async File Operations

```csharp
[HttpPost("upload")]
public async Task<IActionResult> UploadDocument(
    IFormFile file,
    CancellationToken cancellationToken)
{
    var path = Path.Combine("uploads", file.FileName);
    
    await using (var stream = new FileStream(path, FileMode.Create))
    {
        await file.CopyToAsync(stream, cancellationToken);
    }
    
    var content = await File.ReadAllTextAsync(path, cancellationToken);
    
    return Ok(new { FileName = file.FileName, Size = content.Length });
}
```

---

## 9. Missing Model Validation

### ❌ Bad: No Validation

```csharp
public class CreateUserRequest
{
    public string Name { get; set; }
    public string Email { get; set; }
    public int Age { get; set; }
}

[HttpPost]
public async Task<IActionResult> CreateUser(CreateUserRequest request)
{
    // No validation - invalid data processed!
    var user = new User
    {
        Name = request.Name,
        Email = request.Email,
        Age = request.Age
    };
    
    await _dbContext.Users.AddAsync(user);
    await _dbContext.SaveChangesAsync();
    
    return Ok(user);
}
```

### ✅ Good: Data Annotations + ApiController Attribute

```csharp
public class CreateUserRequest
{
    [Required, MaxLength(100)]
    public string Name { get; set; }
    
    [Required, EmailAddress]
    public string Email { get; set; }
    
    [Range(18, 120)]
    public int Age { get; set; }
}

[ApiController] // Auto-validates and returns 400 BadRequest
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> CreateUser(CreateUserRequest request)
    {
        // Validation already done by [ApiController]
        var user = new User
        {
            Name = request.Name,
            Email = request.Email,
            Age = request.Age
        };
        
        await _dbContext.Users.AddAsync(user);
        await _dbContext.SaveChangesAsync();
        
        return CreatedAtAction(nameof(GetUser), new { id = user.Id }, user);
    }
}
```

Or manual validation:
```csharp
[HttpPost]
public async Task<IActionResult> CreateUser(CreateUserRequest request)
{
    if (!ModelState.IsValid)
    {
        return BadRequest(ModelState);
    }
    
    // ... process
}
```

---

## 10. Chatty APIs (Too Many Round-Trips)

### ❌ Bad: Multiple Requests Required

```csharp
// Client needs to make 3 separate requests
[HttpGet("{id}")]
public async Task<IActionResult> GetUser(int id) => Ok(await _dbContext.Users.FindAsync(id));

[HttpGet("{id}/orders")]
public async Task<IActionResult> GetUserOrders(int id) => Ok(await _dbContext.Orders.Where(o => o.UserId == id).ToListAsync());

[HttpGet("{id}/preferences")]
public async Task<IActionResult> GetUserPreferences(int id) => Ok(await _dbContext.UserPreferences.FirstOrDefaultAsync(p => p.UserId == id));
```

**Impact**: 3 HTTP requests, 3× latency overhead, 3 database queries.

### ✅ Good: Aggregate Endpoint

```csharp
public class UserDetailsDto
{
    public UserDto User { get; set; }
    public List<OrderSummaryDto> RecentOrders { get; set; }
    public UserPreferencesDto Preferences { get; set; }
}

[HttpGet("{id}/details")]
public async Task<IActionResult> GetUserDetails(int id)
{
    var userDetails = await _dbContext.Users
        .Where(u => u.Id == id)
        .Select(u => new UserDetailsDto
        {
            User = new UserDto
            {
                Id = u.Id,
                Name = u.Name,
                Email = u.Email
            },
            RecentOrders = u.Orders
                .OrderByDescending(o => o.OrderDate)
                .Take(10)
                .Select(o => new OrderSummaryDto
                {
                    OrderId = o.OrderId,
                    OrderDate = o.OrderDate,
                    Total = o.Total
                })
                .ToList(),
            Preferences = u.Preferences != null ? new UserPreferencesDto
            {
                Theme = u.Preferences.Theme,
                Language = u.Preferences.Language
            } : null
        })
        .FirstOrDefaultAsync();
    
    if (userDetails == null)
        return NotFound();
    
    return Ok(userDetails);
}
```

**Impact**: 3 requests → 1 request. 3× faster overall response time.

---

## Summary Table

| Anti-Pattern | Severity | Typical Impact | Fix |
|--------------|----------|----------------|-----|
| Sync blocking (.Result) | Critical | Deadlocks, thread starvation | Use await |
| Missing CancellationToken | High | Wasted resources | Add CancellationToken parameter |
| N+1 in endpoint | Critical | 100+ DB queries | Eager loading with Include |
| Returning entities | High | Security + bandwidth | Use DTOs with projection |
| Missing pagination | Critical | OOM on large tables | Add Skip/Take with page params |
| No caching | Medium-High | Unnecessary DB load | ResponseCache or OutputCache |
| No compression | Medium | Large payloads | AddResponseCompression |
| Sync file I/O | Medium | Thread blocking | Use async file APIs |
| Missing validation | High | Invalid data processing | Data annotations + ApiController |
| Chatty APIs | Medium | Multiple round-trips | Aggregate endpoints |

---

## Best Practices Checklist

✅ All async endpoints use `async Task<IActionResult>`  
✅ No `.Result`, `.Wait()`, or `.GetAwaiter().GetResult()`  
✅ `CancellationToken` parameter on all long-running operations  
✅ Eager loading (`.Include()`) instead of lazy loading in loops  
✅ DTOs with projection instead of returning entities  
✅ Pagination on all list endpoints (max page size enforced)  
✅ Response caching on stable data (categories, reference tables)  
✅ Response compression enabled globally  
✅ Async file I/O (`ReadAllTextAsync`, `CopyToAsync`)  
✅ Model validation with `[ApiController]` or manual checks  
✅ Aggregate endpoints to reduce client round-trips  

---

## Performance Testing

Before and after:

```csharp
// Use BenchmarkDotNet or load testing tools
// Example with Application Insights query:

requests
| where timestamp > ago(1h)
| where name == "GET /api/orders"
| summarize 
    avg(duration), 
    percentile(duration, 50), 
    percentile(duration, 95), 
    percentile(duration, 99)
```

Measure:
- **Response time** (P50, P95, P99)
- **Throughput** (requests/second)
- **Database query count** (from EF Core logging)
- **Payload size** (Content-Length header)
