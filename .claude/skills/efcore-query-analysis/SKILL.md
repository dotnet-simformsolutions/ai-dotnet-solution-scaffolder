---
name: efcore-query-analysis
description: 'Detect EF Core N+1 query problems, missing eager loading, client-side evaluation, unbounded queries, missing AsNoTracking, cartesian explosions, and SQL injection risks. Use when analyzing Entity Framework Core performance issues and query optimization.'
user-invocable: true
---

# EF Core Query Analysis

## When to Use

- Analyzing DbContext classes and repository patterns
- Investigating slow database operations
- Detecting N+1 query patterns
- Reviewing LINQ queries that generate SQL
- Performance tuning Entity Framework Core applications

## Analysis Procedure

### 1. Locate EF Core Code

Find DbContext and query patterns:
```
grep_search("DbContext|IQueryable|DbSet", isRegexp: true, includePattern: "**/*.cs")
```

Key files to analyze:
- Classes inheriting from `DbContext`
- Repository classes
- Service classes with database queries
- Controllers with direct EF Core usage (anti-pattern itself)

### 2. Pattern Detection

Scan for these EF Core anti-patterns:

#### N+1 Query Pattern (Critical)

**Pattern**: Loop over collection + access navigation property inside loop

```csharp
// N+1: 1 query for orders + N queries for customers
var orders = context.Orders.ToList();
foreach (var order in orders)
{
    Console.WriteLine(order.Customer.Name); // Lazy loads Customer each iteration
}
```

**Detection signs**:
- `ToList()` or `ToArray()` materializing query
- Loop (`foreach`, `for`) over result
- Navigation property access inside loop (`.Customer`, `.Items`, etc.)
- No `.Include()` or `.ThenInclude()` in original query

#### Missing Eager Loading

**Pattern**: Navigation properties accessed without `.Include()`

```csharp
// Missing eager loading
var user = context.Users.FirstOrDefault(u => u.Id == id);
var orders = user.Orders; // Lazy load or separate query
```

Look for:
- Navigation properties accessed after materialization
- Multiple queries where one could suffice
- Repeated database calls in quick succession (check SQL logs if available)

#### Client-Side Evaluation

**Pattern**: `.ToList()` or `.ToArray()` before filtering/projection

```csharp
// Brings ALL users to memory, then filters client-side
var activeUsers = context.Users
    .ToList()
    .Where(u => u.IsActive && u.CreatedDate > threshold)
    .ToList();
```

**Detection**:
- `.ToList()` / `.ToArray()` early in query chain
- LINQ operations after materialization
- Non-translatable operations (custom methods, complex logic) before `.ToList()`

#### Unbounded Queries

**Pattern**: Queries without `.Take()`, pagination, or filtering on large tables

```csharp
// Returns entire table!
var allProducts = context.Products.ToList();
```

**Detection**:
- No `.Take()`, `.Skip()`, or `.FirstOrDefault()`
- No `WHERE` clause (no `.Where()` filter)
- Applied to tables known to grow large (Orders, Logs, Events)

#### Missing AsNoTracking

**Pattern**: Read-only queries using tracked entities

```csharp
// Tracking overhead for read-only data
var report = context.Orders
    .Include(o => o.Items)
    .Where(o => o.Date >= startDate)
    .ToList();
// No updates performed, tracking is waste
```

**Detection**:
- Query results used only for reading/display
- No calls to `context.SaveChanges()` after query
- Report generation, API GET endpoints, dashboards

#### Cartesian Explosion

**Pattern**: Multiple `.Include()` on separate navigation properties

```csharp
// Cartesian product: Orders × Items × Addresses
var user = context.Users
    .Include(u => u.Orders)
    .Include(u => u.Addresses)
    .FirstOrDefault(u => u.Id == id);
```

If user has 100 orders and 3 addresses → 300 rows returned from DB with duplication.

**Detection**:
- Multiple `.Include()` at same level (not chained with `.ThenInclude()`)
- Look for related entities that aren't hierarchical

#### SQL Injection Risk in FromSqlRaw

**Pattern**: String concatenation in raw SQL queries

```csharp
// SQL INJECTION RISK!
var userName = GetUserInput();
var users = context.Users
    .FromSqlRaw($"SELECT * FROM Users WHERE Name = '{userName}'")
    .ToList();
```

**Detection**:
- `FromSqlRaw()` with string interpolation `$"..."` or concatenation
- No parameterized queries
- User input directly in SQL string

#### Select * Equivalent

**Pattern**: Fetching entire entities when only few properties needed

```csharp
// Fetches all columns
var userNames = context.Users
    .ToList()
    .Select(u => u.Name);
```

**Should be**:
```csharp
var userNames = context.Users
    .Select(u => u.Name) // Projects before materialization
    .ToList();
```

### 3. Query Analysis Workflow

For each suspicious query:

1. **Identify the query** (LINQ expression)
2. **Trace execution** (where is it called? Hot path or cold path?)
3. **Estimate data volume** (how many records typically returned?)
4. **Check navigation properties** (accessed inside loops?)
5. **Look for related queries** (could be combined with Include?)
6. **Consult reference**: Load [efcore-antipatterns.md](./references/efcore-antipatterns.md) for detailed examples and SQL comparisons

### 4. Severity Ranking

- **Critical**: N+1 queries in loops processing 100+ items, SQL injection risks
- **High**: Missing eager loading causing multiple queries, unbounded queries on large tables
- **Medium**: Missing `AsNoTracking` on read-only, suboptimal projections
- **Low**: Minor inefficiencies on small tables or cold paths

### 5. Output Format

```markdown
**[Severity]** N+1 Query in [OrderService.cs](src/Services/OrderService.cs#L45-L50)

**Problem**: Navigation property `order.Customer` accessed inside loop without eager loading

**Impact**: Executes 1 + N queries where N = number of orders. With 100 orders → 101 database calls.

**Current SQL**:
\`\`\`sql
-- Query 1: Get all orders
SELECT * FROM Orders WHERE ...

-- Queries 2-101: For each order
SELECT * FROM Customers WHERE Id = @p0
SELECT * FROM Customers WHERE Id = @p1
...
\`\`\`

**Fix**:
\`\`\`csharp
// Add eager loading
var orders = context.Orders
    .Include(o => o.Customer)  // Load customers in single query
    .Where(o => o.Status == "Pending")
    .ToList();

foreach (var order in orders)
{
    Console.WriteLine(order.Customer.Name); // Already loaded
}
\`\`\`

**Optimized SQL**:
\`\`\`sql
-- Single query with JOIN
SELECT o.*, c.*
FROM Orders o
INNER JOIN Customers c ON o.CustomerId = c.Id
WHERE o.Status = 'Pending'
\`\`\`

**Estimated Improvement**: 101 queries → 1 query. ~95% reduction in database round-trips.
```

## Auto-Fix Capability

Can automatically fix:

- ✅ Add `.Include()` for navigation properties
- ✅ Add `.AsNoTracking()` for read-only queries
- ✅ Move `.ToList()` after filtering
- ✅ Add `.Select()` for projection instead of full entities
- ✅ Add `.Take()` with configurable limit for unbounded queries
- ✅ Convert `.ToList().Where()` to `.Where().ToList()`
- ❌ Complex query restructuring (requires domain knowledge)

## Common Patterns to Search

### N+1 Detection
```csharp
// Pattern: ToList() followed by loop with navigation access
\.ToList\(\).*\n.*foreach.*\{.*\n.*\.\w+\. // Regex approximation
```

### Missing AsNoTracking
```csharp
// Read-only endpoints without AsNoTracking
public.*Get.*\(.*\).*\n.*context\.\w+.*\.Include // In controllers/services
```

### Client-Side Evaluation
```csharp
// ToList() before Where/Select
\.ToList\(\).*\n.*\.Where\(
\.ToArray\(\).*\n.*\.Select\(
```

## Integration with Database Analysis

After identifying EF Core issues:

1. **Enable SQL logging** (if not already):
   ```csharp
   // In DbContext.OnConfiguring or appsettings.json
   .EnableSensitiveDataLogging()
   .LogTo(Console.WriteLine, LogLevel.Information)
   ```

2. **Capture actual SQL** generated by problematic queries
3. **Use `/db-query-optimization` skill** to analyze SQL via MCP:
   - Get execution plans
   - Check for missing indexes on join/filter columns
   - Verify query performance with `EXPLAIN ANALYZE` (PostgreSQL) or execution plan (SQL Server)

## Notes

- N+1 is acceptable in cold paths with small data (< 10 items)
- Always test fixes with realistic data volumes
- Monitor SQL query logs in development to catch issues early
- Consider compiled queries for frequently-executed LINQ
