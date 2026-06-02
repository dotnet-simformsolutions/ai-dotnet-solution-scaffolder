# EF Core Performance Anti-Patterns

This reference provides detailed examples of Entity Framework Core performance issues with SQL comparisons.

---

## 1. N+1 Query Problem

### ❌ Bad: Lazy Loading in Loop

```csharp
var orders = context.Orders
    .Where(o => o.Status == "Pending")
    .ToList(); // Query 1

foreach (var order in orders)
{
    Console.WriteLine($"Order for {order.Customer.Name}"); // Query 2, 3, 4...
}
```

**Generated SQL** (1 + N queries):
```sql
-- Query 1
SELECT Id, CustomerId, Status, Total FROM Orders WHERE Status = 'Pending'

-- Query 2
SELECT Id, Name, Email FROM Customers WHERE Id = 101

-- Query 3
SELECT Id, Name, Email FROM Customers WHERE Id = 102

-- ... N more queries
```

### ✅ Good: Eager Loading

```csharp
var orders = context.Orders
    .Include(o => o.Customer) // Eager load
    .Where(o => o.Status == "Pending")
    .ToList(); // Single query with JOIN

foreach (var order in orders)
{
    Console.WriteLine($"Order for {order.Customer.Name}"); // Already loaded
}
```

**Generated SQL** (1 query):
```sql
SELECT 
    o.Id, o.CustomerId, o.Status, o.Total,
    c.Id, c.Name, c.Email
FROM Orders o
INNER JOIN Customers c ON o.CustomerId = c.Id
WHERE o.Status = 'Pending'
```

**Impact**: 100 orders → 101 queries reduced to 1 query. ~99% fewer database round-trips.

---

## 2. Missing AsNoTracking for Read-Only Queries

### ❌ Bad: Unnecessary Change Tracking

```csharp
public async Task<List<ProductDto>> GetProductsForDisplay()
{
    var products = await context.Products
        .Include(p => p.Category)
        .Where(p => p.IsActive)
        .ToListAsync(); // Tracked by default
    
    return products.Select(p => new ProductDto 
    {
        Name = p.Name,
        CategoryName = p.Category.Name
    }).ToList();
}
```

**Overhead**:
- EF Core creates change tracking snapshots for all entities
- Memory overhead ~40% per entity
- Slower query execution

### ✅ Good: AsNoTracking for Read-Only

```csharp
public async Task<List<ProductDto>> GetProductsForDisplay()
{
    var products = await context.Products
        .AsNoTracking() // No change tracking
        .Include(p => p.Category)
        .Where(p => p.IsActive)
        .ToListAsync();
    
    return products.Select(p => new ProductDto 
    {
        Name = p.Name,
        CategoryName = p.Category.Name
    }).ToList();
}
```

**Impact**: 
- 30-50% faster query execution
- 40% less memory usage
- No change tracking overhead

**When to use**: API GET endpoints, reports, dashboards, any read-only scenario.

---

## 3. Client-Side Evaluation (Fetching Too Much Data)

### ❌ Bad: Filter After Materialization

```csharp
var recentOrders = context.Orders
    .ToList() // Fetches ALL orders to memory
    .Where(o => o.CreatedDate > DateTime.Now.AddDays(-30))
    .ToList();
```

**Generated SQL**:
```sql
-- Fetches entire table!
SELECT Id, CustomerId, CreatedDate, Total FROM Orders
```

Then filters in C# memory.

### ✅ Good: Filter in Database

```csharp
var recentOrders = context.Orders
    .Where(o => o.CreatedDate > DateTime.Now.AddDays(-30)) // SQL WHERE
    .ToList(); // Fetch only filtered results
```

**Generated SQL**:
```sql
SELECT Id, CustomerId, CreatedDate, Total 
FROM Orders
WHERE CreatedDate > @p0
```

**Impact**: 1,000,000 rows → 1,000 rows. 1000x less data transferred over network.

---

## 4. Select * When Projection Needed

### ❌ Bad: Fetch Entire Entity

```csharp
var userNames = context.Users
    .ToList() // Fetches ALL columns
    .Select(u => u.Name)
    .ToList();
```

**Generated SQL**:
```sql
SELECT Id, Name, Email, PasswordHash, CreatedDate, ModifiedDate, ... 
FROM Users
```

Fetches 20+ columns when only 1 is needed.

### ✅ Good: Project in Query

```csharp
var userNames = context.Users
    .Select(u => u.Name) // Project before materialization
    .ToList();
```

**Generated SQL**:
```sql
SELECT Name FROM Users
```

Or with DTO:
```csharp
var users = context.Users
    .Select(u => new UserDto 
    {
        Name = u.Name,
        Email = u.Email
    })
    .ToList();
```

**Impact**: Reduces network bandwidth, faster query execution, less memory.

---

## 5. Cartesian Explosion from Multiple Includes

### ❌ Bad: Multiple Includes at Same Level

```csharp
var user = context.Users
    .Include(u => u.Orders)      // 1-to-many
    .Include(u => u.Addresses)   // 1-to-many
    .FirstOrDefault(u => u.Id == userId);
```

If user has 100 orders and 3 addresses:
- Database returns 300 rows (100 × 3 Cartesian product)
- EF Core materializes 1 user, 100 orders, 3 addresses
- But 300 rows sent over network with massive duplication

**Generated SQL**:
```sql
SELECT 
    u.*, o.*, a.*
FROM Users u
LEFT JOIN Orders o ON u.Id = o.UserId
LEFT JOIN Addresses a ON u.Id = a.UserId
WHERE u.Id = @p0
```

### ✅ Good: Split into Multiple Queries or AsSplitQuery

**Option 1: Split Queries (EF Core 5+)**
```csharp
var user = context.Users
    .Include(u => u.Orders)
    .Include(u => u.Addresses)
    .AsSplitQuery() // Separate queries, no Cartesian product
    .FirstOrDefault(u => u.Id == userId);
```

Generates 3 queries:
```sql
SELECT * FROM Users WHERE Id = @p0
SELECT * FROM Orders WHERE UserId = @p0
SELECT * FROM Addresses WHERE UserId = @p0
```

**Option 2: Manual Loading**
```csharp
var user = context.Users.Find(userId);
context.Entry(user).Collection(u => u.Orders).Load();
context.Entry(user).Collection(u => u.Addresses).Load();
```

**Impact**: 300 duplicate rows → 104 rows (1 + 100 + 3). Significant bandwidth savings.

---

## 6. Unbounded Queries on Large Tables

### ❌ Bad: No Pagination or Limit

```csharp
public async Task<List<Order>> GetOrders()
{
    return await context.Orders
        .Include(o => o.Items)
        .ToListAsync(); // Returns ENTIRE table!
}
```

With 1 million orders → massive memory consumption, slow query, network timeout.

### ✅ Good: Pagination and Filtering

```csharp
public async Task<PagedResult<Order>> GetOrders(int page, int pageSize = 50)
{
    var query = context.Orders
        .Include(o => o.Items)
        .OrderByDescending(o => o.CreatedDate);
    
    var total = await query.CountAsync();
    var items = await query
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .ToListAsync();
    
    return new PagedResult<Order>
    {
        Items = items,
        TotalCount = total,
        Page = page,
        PageSize = pageSize
    };
}
```

Or with filtering:
```csharp
public async Task<List<Order>> GetRecentOrders()
{
    return await context.Orders
        .Where(o => o.CreatedDate > DateTime.UtcNow.AddDays(-7))
        .Take(100) // Safety limit
        .ToListAsync();
}
```

---

## 7. SQL Injection in FromSqlRaw

### ❌ Bad: String Concatenation/Interpolation

```csharp
// CRITICAL SECURITY VULNERABILITY!
var userName = GetUserInput(); // User enters: "admin' OR '1'='1"
var users = context.Users
    .FromSqlRaw($"SELECT * FROM Users WHERE Name = '{userName}'")
    .ToList();
```

**Executed SQL**:
```sql
SELECT * FROM Users WHERE Name = 'admin' OR '1'='1'
-- Returns ALL users!
```

### ✅ Good: Parameterized Query

```csharp
var userName = GetUserInput();
var users = context.Users
    .FromSqlRaw("SELECT * FROM Users WHERE Name = {0}", userName)
    .ToList();
```

Or better yet, use LINQ:
```csharp
var users = context.Users
    .Where(u => u.Name == userName)
    .ToList();
```

---

## 8. Inefficient Updates (Load-Modify-Save)

### ❌ Bad: Fetch Then Update

```csharp
public async Task UpdateUserEmail(int userId, string newEmail)
{
    var user = await context.Users.FindAsync(userId); // SELECT
    user.Email = newEmail;
    await context.SaveChangesAsync(); // UPDATE entire row
}
```

**Generated SQL**:
```sql
SELECT Id, Name, Email, ... FROM Users WHERE Id = @p0
UPDATE Users SET Id=@p0, Name=@p1, Email=@p2, ... WHERE Id = @p0
```

Updates all columns even though only Email changed.

### ✅ Good: ExecuteUpdate (EF Core 7+)

```csharp
public async Task UpdateUserEmail(int userId, string newEmail)
{
    await context.Users
        .Where(u => u.Id == userId)
        .ExecuteUpdateAsync(setters => setters
            .SetProperty(u => u.Email, newEmail));
}
```

**Generated SQL**:
```sql
UPDATE Users SET Email = @p0 WHERE Id = @p1
```

No SELECT, updates only changed column.

**Impact**: 2 round-trips → 1 round-trip. Faster, less bandwidth.

---

## 9. Missing Compiled Queries for Hot Paths

### ❌ Bad: Repeated Query Parsing

```csharp
public User GetUser(int id)
{
    // EF Core parses this LINQ expression every call
    return context.Users
        .Include(u => u.Profile)
        .FirstOrDefault(u => u.Id == id);
}
```

### ✅ Good: Compiled Query

```csharp
private static readonly Func<MyDbContext, int, User> _getUserQuery =
    EF.CompileQuery((MyDbContext context, int id) =>
        context.Users
            .Include(u => u.Profile)
            .FirstOrDefault(u => u.Id == id));

public User GetUser(int id)
{
    return _getUserQuery(context, id);
}
```

**Impact**: ~20-30% faster on frequently-executed queries (1000+ calls/sec).

---

## 10. Implicit vs Explicit Loading Confusion

### ❌ Bad: Implicit Lazy Loading Everywhere

```csharp
// DbContext configured with lazy loading proxies
var orders = context.Orders.ToList();

foreach (var order in orders)
{
    // Each access triggers separate query
    Console.WriteLine(order.Customer.Name);
    Console.WriteLine(order.Items.Count);
    Console.WriteLine(order.ShippingAddress.City);
}
```

100 orders → 301 queries (1 for orders, 100 for customers, 100 for items, 100 for addresses).

### ✅ Good: Explicit Eager Loading

```csharp
var orders = context.Orders
    .Include(o => o.Customer)
    .Include(o => o.Items)
    .Include(o => o.ShippingAddress)
    .ToList(); // Single query or split queries

foreach (var order in orders)
{
    // All data already loaded
    Console.WriteLine(order.Customer.Name);
    Console.WriteLine(order.Items.Count);
    Console.WriteLine(order.ShippingAddress.City);
}
```

---

## Summary Table

| Anti-Pattern | Severity | Typical Impact | Auto-Fix |
|--------------|----------|----------------|----------|
| N+1 queries | Critical | 1 query → 100+ queries | ✅ Add Include |
| Missing AsNoTracking | Medium | 30-50% slower reads | ✅ Add AsNoTracking |
| Client-side evaluation | High | 1000x more data | ✅ Move Where before ToList |
| Select * without projection | Medium | Unnecessary bandwidth | ✅ Add Select projection |
| Cartesian explosion | High | Huge result sets | ✅ Add AsSplitQuery |
| Unbounded queries | Critical | OOM / timeouts | ✅ Add Take + pagination |
| SQL injection | Critical | Security breach | ✅ Use parameters |
| Inefficient updates | Medium | 2x round-trips | ✅ Use ExecuteUpdate |

---

## Detection Checklist

When reviewing EF Core code, ask:

1. ✅ Are navigation properties loaded with `.Include()` before loops?
2. ✅ Is `.AsNoTracking()` used for read-only queries?
3. ✅ Are filters applied before `.ToList()` / `.ToArray()`?
4. ✅ Are projections (`.Select()`) used instead of full entities?
5. ✅ Do large queries have `.Take()` or pagination?
6. ✅ Is raw SQL parameterized, not concatenated?
7. ✅ Are hot-path queries compiled with `EF.CompileQuery()`?
8. ✅ Are multiple includes split with `.AsSplitQuery()` if needed?

---

## Pro Tips

### Enable SQL Logging in Development

```csharp
// appsettings.Development.json
{
  "Logging": {
    "LogLevel": {
      "Microsoft.EntityFrameworkCore.Database.Command": "Information"
    }
  }
}
```

Or in code:
```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder
        .UseSqlServer(connectionString)
        .LogTo(Console.WriteLine, LogLevel.Information)
        .EnableSensitiveDataLogging(); // Shows parameter values
}
```

### Use MiniProfiler or Application Insights

Track query counts and execution times in production to catch N+1 issues.

### Test with Realistic Data

Small test datasets hide performance problems. Load test with 10,000+ records.
