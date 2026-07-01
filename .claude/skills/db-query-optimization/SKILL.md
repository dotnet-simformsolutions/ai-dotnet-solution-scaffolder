---
name: db-query-optimization
description: 'Analyze database queries via MCP, generate execution plans, detect table scans, recommend missing indexes for SQL Server and PostgreSQL. Examines DMVs, index usage stats, and query performance metrics. Use when optimizing database query performance.'
user-invocable: true
---

# Database Query Optimization

## When to Use

- Slow SQL queries identified in application logs
- High database CPU or I/O usage
- After detecting EF Core N+1 or suboptimal queries
- Database performance tuning and index optimization
- Query execution plan analysis

## Prerequisites

**SQL Server**:
- User needs `VIEW SERVER STATE` permission for DMV access
- Connection configured in `.vscode/mcp.json` as `mssql` server

**PostgreSQL**:
- `pg_stat_statements` extension recommended (gracefully handle if missing)
- Connection configured in `.vscode/mcp.json` as `postgres` server

**MCP Server Status**:
- Both `mssql-dotnet/*` and `postgres/*` tools must be available
- Connections are read-only (no DDL execution)

## Analysis Workflow

### 1. Identify Target Queries

Sources:
- EF Core generated SQL (from `/efcore-query-analysis`)
- Application logs with slow query warnings
- User-provided SQL query text
- Stored procedure calls (analyzed separately with `/stored-proc-analysis`)

### 2. Execute Diagnostic Queries via MCP

#### For SQL Server (use mssql/* tools)

**A. Get Execution Plan for Specific Query**

```sql
SET SHOWPLAN_TEXT ON;
GO
-- Your query here
SELECT o.OrderId, c.CustomerName
FROM Orders o
INNER JOIN Customers c ON o.CustomerId = c.Id
WHERE o.OrderDate > '2024-01-01';
GO
SET SHOWPLAN_TEXT OFF;
```

Or for actual execution plan with stats:
```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

-- Your query here

SET STATISTICS IO OFF;
SET STATISTICS TIME OFF;
```

**B. Find Top Expensive Queries (CPU/Reads/Duration)**

```sql
SELECT TOP 20
    qs.execution_count,
    qs.total_worker_time / qs.execution_count AS avg_cpu_time_microsec,
    qs.total_logical_reads / qs.execution_count AS avg_logical_reads,
    qs.total_elapsed_time / qs.execution_count AS avg_elapsed_time_microsec,
    SUBSTRING(qt.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(qt.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2) + 1) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
ORDER BY qs.total_worker_time DESC; -- Or total_logical_reads, total_elapsed_time
```

**C. Find Missing Indexes**

```sql
SELECT
    CONVERT(DECIMAL(18,2), migs.avg_total_user_cost * migs.avg_user_impact * (migs.user_seeks + migs.user_scans)) AS improvement_measure,
    'CREATE INDEX IX_' + OBJECT_NAME(mid.object_id, mid.database_id) + '_'
        + REPLACE(REPLACE(REPLACE(ISNULL(mid.equality_columns,''), ', ', '_'), '[', ''), ']', '') + '_'
        + REPLACE(REPLACE(REPLACE(ISNULL(mid.inequality_columns,''), ', ', '_'), '[', ''), ']', '')
        + ' ON ' + mid.statement + ' (' + ISNULL(mid.equality_columns,'')
        + CASE WHEN mid.equality_columns IS NOT NULL AND mid.inequality_columns IS NOT NULL THEN ',' ELSE '' END
        + ISNULL(mid.inequality_columns, '') + ')' 
        + ISNULL(' INCLUDE (' + mid.included_columns + ')', '') AS create_index_statement,
    migs.user_seeks,
    migs.user_scans,
    migs.avg_total_user_cost,
    migs.avg_user_impact
FROM sys.dm_db_missing_index_groups mig
INNER JOIN sys.dm_db_missing_index_group_stats migs ON migs.group_handle = mig.index_group_handle
INNER JOIN sys.dm_db_missing_index_details mid ON mig.index_handle = mid.index_handle
WHERE CONVERT(DECIMAL(18,2), migs.avg_total_user_cost * migs.avg_user_impact * (migs.user_seeks + migs.user_scans)) > 10
ORDER BY improvement_measure DESC;
```

**D. Check Index Usage**

```sql
SELECT
    OBJECT_NAME(s.object_id) AS table_name,
    i.name AS index_name,
    s.user_seeks,
    s.user_scans,
    s.user_lookups,
    s.user_updates,
    CASE 
        WHEN s.user_seeks + s.user_scans + s.user_lookups = 0 THEN 'UNUSED'
        WHEN s.user_updates > (s.user_seeks + s.user_scans + s.user_lookups) * 10 THEN 'OVER-MAINTAINED'
        ELSE 'ACTIVE'
    END AS index_status
FROM sys.dm_db_index_usage_stats s
INNER JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
WHERE OBJECTPROPERTY(s.object_id, 'IsUserTable') = 1
    AND i.type_desc <> 'HEAP'
ORDER BY s.user_updates DESC;
```

**E. Check Index Fragmentation**

```sql
SELECT
    OBJECT_NAME(ips.object_id) AS table_name,
    i.name AS index_name,
    ips.index_type_desc,
    ips.avg_fragmentation_in_percent,
    ips.page_count,
    CASE 
        WHEN ips.avg_fragmentation_in_percent > 30 THEN 'REBUILD'
        WHEN ips.avg_fragmentation_in_percent > 10 THEN 'REORGANIZE'
        ELSE 'OK'
    END AS recommended_action
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 10
    AND ips.page_count > 100 -- Ignore small indexes
ORDER BY ips.avg_fragmentation_in_percent DESC;
```

#### For PostgreSQL (use postgres/* tools)

**A. Get Execution Plan**

```sql
EXPLAIN ANALYZE
SELECT o.order_id, c.customer_name
FROM orders o
INNER JOIN customers c ON o.customer_id = c.id
WHERE o.order_date > '2024-01-01';
```

**B. Find Top Expensive Queries (requires pg_stat_statements)**

```sql
-- Check if extension is enabled
SELECT * FROM pg_extension WHERE extname = 'pg_stat_statements';

-- If enabled, get top queries
SELECT
    query,
    calls,
    total_exec_time / calls AS avg_time_ms,
    min_exec_time AS min_time_ms,
    max_exec_time AS max_time_ms,
    rows / calls AS avg_rows,
    100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0) AS cache_hit_ratio
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

If not enabled, note it as a recommendation.

**C. Check Index Usage**

```sql
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan AS index_scans,
    idx_tup_read AS tuples_read,
    idx_tup_fetch AS tuples_fetched,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    CASE 
        WHEN idx_scan = 0 THEN 'UNUSED'
        WHEN idx_scan < 100 THEN 'LOW_USAGE'
        ELSE 'ACTIVE'
    END AS usage_status
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC, pg_relation_size(indexrelid) DESC;
```

**D. Find Missing Indexes (Seq Scans on Large Tables)**

```sql
SELECT
    schemaname,
    tablename,
    seq_scan AS sequential_scans,
    seq_tup_read AS rows_read_sequentially,
    idx_scan AS index_scans,
    n_live_tup AS approx_row_count,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    ROUND(100.0 * seq_tup_read / NULLIF(seq_tup_read + idx_tup_fetch, 0), 2) AS seq_scan_ratio
FROM pg_stat_user_tables
WHERE seq_scan > 0
    AND n_live_tup > 10000 -- Large tables only
    AND seq_tup_read / NULLIF(seq_scan, 0) > 10000 -- Many rows per scan
ORDER BY seq_tup_read DESC
LIMIT 20;
```

**E. Check Table and Index Sizes**

```sql
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) AS indexes_size,
    n_live_tup AS row_count
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 20;
```

### 3. Analyze Execution Plans

Look for these red flags:

**SQL Server**:
- **Table Scan** / **Clustered Index Scan** on large tables → missing index
- **Key Lookup** with high cost → consider covering index
- **Sort** / **Hash Match** operators with high cost → missing index on ORDER BY/GROUP BY columns
- **Nested Loops** with large outer input → consider merge join or hash join via index
- **Missing Index** warnings in actual execution plan
- High **logical reads** in STATISTICS IO → table scans or poor indexing

**PostgreSQL**:
- **Seq Scan** on large tables (cost >> 1000) → missing index
- **Sort** with high cost → index on ORDER BY columns
- **Hash** or **Merge Join** with low buffer hit ratio → table too large or missing stats
- **Nested Loop** with large dataset → consider hash/merge join
- High `rows` removed by filter → predicate not selective enough

### 4. Generate Index Recommendations

For each problematic query:

1. **Identify filter columns** (WHERE clause) → equality columns first
2. **Identify join columns** (ON clause) → must be indexed
3. **Identify sort columns** (ORDER BY, GROUP BY)
4. **Identify covering columns** (SELECT columns not in index)

Use [index-strategies.md](./references/index-strategies.md) for detailed index design patterns.

**Recommendation format**:
```sql
-- Recommended index for query on Orders table
-- Filters: OrderDate > '2024-01-01' AND Status = 'Pending'
-- Join: CustomerId = c.Id
-- Sort: OrderDate DESC

CREATE NONCLUSTERED INDEX IX_Orders_Status_OrderDate_CustomerId
ON Orders (Status, OrderDate, CustomerId)
INCLUDE (OrderTotal, ShippingAddress);

-- Estimated Impact: 
--   Table Scan (cost 5000) → Index Seek (cost 50)
--   Reduces logical reads from 10,000 to 100
```

### 5. Detect Query Anti-Patterns

**Common Issues**:

- **SELECT * in application queries** → fetch only needed columns
- **Functions on indexed columns** (e.g., `WHERE YEAR(OrderDate) = 2024`) → prevents index usage
  - Fix: `WHERE OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01'`
- **LIKE with leading wildcard** (`WHERE Name LIKE '%son'`) → full table scan
  - Fix: Reverse index, full-text search, or redesign query
- **Implicit type conversions** (e.g., `WHERE VarcharColumn = 123`) → prevents index usage
  - Fix: Match data types in comparison
- **OR conditions on different columns** → can't use single index efficiently
  - Fix: UNION queries or composite index
- **Parameter sniffing** (SQL Server) → cached plan not optimal for current parameters
  - Fix: `OPTION (RECOMPILE)`, `OPTIMIZE FOR`, or plan guides

### 6. Output Format

```markdown
### Query Performance Analysis: [QueryName or Location]

**Current Performance**:
- Execution time: 2.5 seconds
- Logical reads: 50,000
- Execution plan: Table Scan on Orders (500,000 rows)

**Execution Plan Analysis**:
\`\`\`
|--Table Scan (Orders)
   Cost: 4876.23
   Rows: 500,000
   Predicate: OrderDate > '2024-01-01'
\`\`\`

**Problem**: Table scan on large Orders table due to missing index on OrderDate.

**Recommended Index**:
\`\`\`sql
CREATE NONCLUSTERED INDEX IX_Orders_OrderDate
ON Orders (OrderDate DESC)
INCLUDE (OrderId, CustomerId, OrderTotal);
\`\`\`

**Estimated Improvement**:
- Execution time: 2.5s → 0.05s (50x faster)
- Logical reads: 50,000 → 100 (500x reduction)
- Execution plan: Index Seek (cost ~10)

**Additional Recommendations**:
- Consider partitioning Orders table by OrderDate if > 10M rows
- Update statistics on Orders table weekly
- Review query: fetch only required columns instead of SELECT *
```

## Read-Only Enforcement

**YOU MUST NOT** execute DDL via MCP:
- ❌ CREATE INDEX
- ❌ ALTER TABLE
- ❌ DROP INDEX
- ❌ UPDATE STATISTICS

**Only recommend** these changes in your findings. Let the user execute them manually after review.

**You CAN execute** (read-only):
- ✅ EXPLAIN / SET SHOWPLAN_TEXT
- ✅ SELECT from DMVs (sys.dm_*)
- ✅ SELECT from pg_stat_* views
- ✅ SET STATISTICS IO/TIME (read-only)

## Integration with Other Skills

- After `/efcore-query-analysis`: Analyze the generated SQL queries
- Before recommending indexes: Check existing indexes aren't already covering the need
- After analysis: Feed findings into `/perf-report-generator`

## Notes

- Always check for existing indexes before recommending new ones
- Consider index maintenance overhead (writes become slower)
- Large batch operations may benefit from dropping/rebuilding indexes
- Test index recommendations in non-production environment first
- Monitor index usage after creation (unused indexes should be dropped)
