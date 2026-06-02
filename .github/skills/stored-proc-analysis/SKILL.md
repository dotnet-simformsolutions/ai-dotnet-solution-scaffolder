---
name: stored-proc-analysis
description: 'Analyze stored procedures and functions via MCP for both SQL Server and PostgreSQL. Retrieves definitions, execution statistics, execution plans, and detects anti-patterns like cursors, parameter sniffing, missing error handling, and row-by-row operations. Use when optimizing stored procedure performance.'
user-invocable: true
---

# Stored Procedure Analysis

## When to Use

- Identifying slow stored procedures or functions
- Database performance review focusing on stored code
- Detecting cursor usage and row-by-row operations
- Finding parameter sniffing issues
- Reviewing stored procedure best practices

## Prerequisites

**SQL Server**:
- `VIEW SERVER STATE` permission for `sys.dm_exec_procedure_stats`
- Read-only permissions on system views

**PostgreSQL**:
- Read-only permissions on `pg_proc`, `pg_stat_user_functions`
- `pg_stat_statements` extension helpful but not required

**MCP Servers**:
- `mssql-dotnet/*` tools for SQL Server
- `postgres/*` tools for PostgreSQL

## Analysis Workflow

### 1. Discovery - List Stored Procedures

#### SQL Server

```sql
-- List all stored procedures with metadata
SELECT
    SCHEMA_NAME(p.schema_id) AS schema_name,
    p.name AS procedure_name,
    p.create_date,
    p.modify_date,
    p.type_desc
FROM sys.procedures p
WHERE p.is_ms_shipped = 0 -- Exclude system procedures
ORDER BY p.modify_date DESC;
```

#### PostgreSQL

```sql
-- List all functions/procedures
SELECT
    n.nspname AS schema_name,
    p.proname AS function_name,
    pg_get_function_identity_arguments(p.oid) AS arguments,
    l.lanname AS language,
    CASE p.prokind
        WHEN 'f' THEN 'function'
        WHEN 'p' THEN 'procedure'
        WHEN 'a' THEN 'aggregate'
        WHEN 'w' THEN 'window'
    END AS routine_type
FROM pg_proc p
INNER JOIN pg_namespace n ON p.pronamespace = n.oid
INNER JOIN pg_language l ON p.prolang = l.oid
WHERE n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY p.proname;
```

### 2. Retrieve Stored Procedure Definitions

#### SQL Server

```sql
-- Get procedure definition
EXEC sp_helptext 'dbo.GetCustomerOrders';

-- Or using OBJECT_DEFINITION
SELECT OBJECT_DEFINITION(OBJECT_ID('dbo.GetCustomerOrders')) AS definition;

-- Get parameter information
SELECT
    p.name AS parameter_name,
    TYPE_NAME(p.user_type_id) AS data_type,
    p.max_length,
    p.is_output
FROM sys.parameters p
WHERE p.object_id = OBJECT_ID('dbo.GetCustomerOrders')
ORDER BY p.parameter_id;
```

#### PostgreSQL

```sql
-- Get function definition
SELECT pg_get_functiondef(oid)
FROM pg_proc
WHERE proname = 'get_customer_orders'
    AND pronamespace = (SELECT oid FROM pg_namespace WHERE nspname = 'public');

-- Or from information_schema
SELECT routine_definition
FROM information_schema.routines
WHERE routine_schema = 'public'
    AND routine_name = 'get_customer_orders';
```

### 3. Get Runtime Execution Statistics

#### SQL Server

```sql
-- Procedure execution stats from DMV
SELECT
    OBJECT_SCHEMA_NAME(ps.object_id) AS schema_name,
    OBJECT_NAME(ps.object_id) AS procedure_name,
    ps.execution_count,
    ps.total_elapsed_time / 1000000.0 AS total_elapsed_time_sec,
    ps.total_elapsed_time / ps.execution_count / 1000.0 AS avg_elapsed_time_ms,
    ps.total_worker_time / ps.execution_count / 1000.0 AS avg_cpu_time_ms,
    ps.total_logical_reads / ps.execution_count AS avg_logical_reads,
    ps.total_physical_reads / ps.execution_count AS avg_physical_reads,
    ps.last_execution_time,
    ps.min_elapsed_time / 1000.0 AS min_elapsed_time_ms,
    ps.max_elapsed_time / 1000.0 AS max_elapsed_time_ms
FROM sys.dm_exec_procedure_stats ps
WHERE OBJECT_SCHEMA_NAME(ps.object_id) NOT IN ('sys')
ORDER BY ps.total_elapsed_time DESC;
```

#### PostgreSQL

```sql
-- Function execution stats
SELECT
    schemaname,
    funcname,
    calls,
    total_time / calls AS avg_time_ms,
    self_time / calls AS avg_self_time_ms,
    total_time,
    self_time
FROM pg_stat_user_functions
ORDER BY total_time DESC;
```

### 4. Get Execution Plans

#### SQL Server

```sql
-- Get execution plan for stored procedure
SET SHOWPLAN_TEXT ON;
GO
EXEC dbo.GetCustomerOrders @CustomerId = 123;
GO
SET SHOWPLAN_TEXT OFF;
GO

-- Or get actual execution plan with stats
SET STATISTICS TIME ON;
SET STATISTICS IO ON;
EXEC dbo.GetCustomerOrders @CustomerId = 123;
SET STATISTICS TIME OFF;
SET STATISTICS IO OFF;
```

#### PostgreSQL

```sql
-- Get execution plan for function
EXPLAIN ANALYZE
SELECT * FROM get_customer_orders(123);

-- Or if it's a procedural function
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT * FROM get_customer_orders(123);
```

### 5. Code Pattern Analysis

Analyze the stored procedure definition text for these anti-patterns:

#### A. Cursor Usage (Row-by-Row Processing)

**Detection Pattern** (SQL Server):
- `DECLARE` ... `CURSOR`
- `OPEN`, `FETCH`, `CLOSE`, `DEALLOCATE`

**Detection Pattern** (PostgreSQL):
- `DECLARE` ... `CURSOR FOR`
- `LOOP` ... `FETCH` ... `EXIT WHEN NOT FOUND`

**Red Flag**: Cursors in stored procedures processing > 100 rows.

#### B. Missing SET NOCOUNT ON (SQL Server Only)

**Detection**: SP definition doesn't start with `SET NOCOUNT ON`

**Impact**: Extra result sets returned with row count messages, slowing down applications.

#### C. Scalar UDFs in SELECT Statements

**SQL Server Detection**:
- `SELECT` statement calling a scalar function (not table-valued)
- Pattern: `dbo.FunctionName(column)` in SELECT list

**Impact**: Function called once per row (row-by-row execution).

#### D. SELECT * in Stored Procedures

**Detection**: `SELECT *` in queries inside SP

**Impact**: Fetches unnecessary columns, breaks when schema changes.

#### E. WHILE Loops Doing Row-by-Row Operations

**Detection**:
- `WHILE` loop with table variable/temp table
- Pattern: `SELECT TOP 1`, process, `DELETE TOP 1`, repeat

**Alternative**: Set-based operation.

#### F. Temp Table Abuse

**Detection**:
- Multiple `#TempTable` creations
- Temp tables with small row counts (< 1000 rows) → use table variables or CTEs

**Impact**: tempdb contention, excessive I/O.

#### G. Dynamic SQL Without Parameterization

**Detection**:
- `EXEC()` or `sp_executesql` with string concatenation
- Pattern: `EXEC('SELECT * FROM Users WHERE Id = ' + @UserId)`

**Impact**: SQL injection risk, plan cache pollution.

#### H. Parameter Sniffing Issues (SQL Server)

**Detection**:
- Procedures with highly variable result sets based on parameters
- No `OPTION (RECOMPILE)` or `OPTIMIZE FOR` hints
- Symptoms: inconsistent performance for different parameter values

**Recommendation**: Add `OPTION (RECOMPILE)`, use `OPTIMIZE FOR`, or split into multiple procedures.

#### I. Missing Error Handling

**SQL Server Detection**:
- No `BEGIN TRY` ... `END TRY` ... `BEGIN CATCH` ... `END CATCH`
- No `@@ERROR` checks

**PostgreSQL Detection**:
- No `EXCEPTION WHEN` block in PL/pgSQL

**Impact**: Silent failures, partial updates, data corruption.

#### J. Nested Stored Procedure Chains

**Detection**:
- SP calls another SP, which calls another SP (> 3 levels deep)

**Impact**: Complex debugging, hidden dependencies, performance overhead.

#### K. Excessive Recompilation Hints

**Detection** (SQL Server):
- `OPTION (RECOMPILE)` on every query in SP
- `sp_recompile` after every schema change

**Impact**: CPU overhead from constant recompilation.

### 6. Analyze and Rank Findings

For each stored procedure, prioritize issues:

**Critical**:
- Cursors processing 1000+ rows
- SQL injection vulnerabilities
- Missing error handling with data modifications
- Scalar UDFs in SELECT on large result sets

**High**:
- WHILE loops doing row-by-row operations
- Parameter sniffing causing 10x+ performance variance
- Missing SET NOCOUNT ON in frequently-called procedures
- Temp table abuse (excessive creation/dropping)

**Medium**:
- SELECT * in procedures
- Minor cursor usage (< 100 rows)
- Deeply nested SP calls (> 3 levels)
- Missing indexes on temp tables

**Low**:
- Code style issues (naming, formatting)
- Minor optimization opportunities

### 7. Output Format

```markdown
### Stored Procedure Analysis: dbo.GetCustomerOrders

**Execution Statistics** (from `sys.dm_exec_procedure_stats`):
- Total executions: 125,000
- Average execution time: 450 ms
- Average CPU time: 320 ms
- Average logical reads: 15,000
- Last executed: 2024-04-29 10:15:33

**Performance Ranking**: High-frequency, moderate-duration (hot path)

---

**Issue 1: Cursor Usage [Critical]**

**Location**: Lines 15-45 in procedure definition

**Problem**: Cursor iterates over Orders table and processes row-by-row.

\`\`\`sql
DECLARE order_cursor CURSOR FOR
SELECT OrderId, CustomerId, Total FROM Orders WHERE CustomerId = @CustomerId;

OPEN order_cursor;
FETCH NEXT FROM order_cursor INTO @OrderId, @CustomerId, @Total;

WHILE @@FETCH_STATUS = 0
BEGIN
    -- Process each order
    EXEC ProcessOrder @OrderId;
    FETCH NEXT FROM order_cursor INTO @OrderId, @CustomerId, @Total;
END

CLOSE order_cursor;
DEALLOCATE order_cursor;
\`\`\`

**Impact**: 
- With 100 orders → 100 separate calls to ProcessOrder
- ~450ms per execution with cursor vs estimated ~50ms with set-based approach

**Recommended Fix**:
\`\`\`sql
-- Replace cursor with set-based operation
-- If ProcessOrder can be inlined:
UPDATE Orders
SET ProcessedDate = GETDATE(),
    Status = 'Processed'
WHERE CustomerId = @CustomerId
    AND Status = 'Pending';

-- Or use table-valued function if logic is complex:
INSERT INTO ProcessedOrders (OrderId, CustomerId, ProcessedData)
SELECT
    o.OrderId,
    o.CustomerId,
    dbo.ProcessOrderLogic(o.OrderId, o.Total)
FROM Orders o
WHERE o.CustomerId = @CustomerId;
\`\`\`

**Estimated Improvement**: 450ms → 50ms (9x faster), reduces logical reads significantly.

---

**Issue 2: Missing SET NOCOUNT ON [Medium]**

**Problem**: Procedure returns row count messages for each statement.

**Fix**:
\`\`\`sql
CREATE PROCEDURE dbo.GetCustomerOrders
    @CustomerId INT
AS
BEGIN
    SET NOCOUNT ON; -- Add this line
    
    -- ... rest of procedure
END
\`\`\`

**Impact**: Reduces network traffic and improves performance in tight loops.
```

### 8. Consult Reference

Load [sp-antipatterns.md](./references/sp-antipatterns.md) for detailed examples of each anti-pattern with fix strategies.

## Auto-Fix Capability

**Can automatically recommend** (but NOT execute):
- ✅ Set-based alternatives to cursors
- ✅ Add `SET NOCOUNT ON`
- ✅ Add `TRY/CATCH` error handling
- ✅ Parameterize dynamic SQL
- ✅ Replace `SELECT *` with explicit columns
- ❌ Rewriting complex business logic (requires domain knowledge)
- ❌ Executing ALTER PROCEDURE (read-only mode)

## Read-Only Enforcement

**YOU MUST NOT** execute via MCP:
- ❌ `ALTER PROCEDURE`
- ❌ `DROP PROCEDURE`
- ❌ `CREATE PROCEDURE`
- ❌ `sp_recompile`

**Only recommend** changes in findings.

**You CAN execute**:
- ✅ `SELECT` from system views (`sys.procedures`, `pg_proc`)
- ✅ `EXEC sp_helptext` (read-only)
- ✅ `EXPLAIN` / `SET SHOWPLAN_TEXT`
- ✅ `SELECT` from DMVs and `pg_stat_*`

## Integration with Other Skills

- After `/db-query-optimization`: Stored procedures may contain queries needing index optimization
- Related to `/efcore-query-analysis`: EF Core may call stored procedures
- Feed findings into `/perf-report-generator` for comprehensive report

## Notes

- Always consider execution frequency when prioritizing fixes
- Test fixes in non-production environment first
- Profile before and after changes to validate improvements
- Some cursor usage is acceptable for admin/maintenance procedures (cold paths)
- Parameter sniffing can be complex - consider workload and caching strategy
