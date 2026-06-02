# Index Strategies and Optimization Patterns

Comprehensive guide to index design, covering both SQL Server and PostgreSQL.

---

## Index Design Principles

### 1. Column Order Matters

**Rule**: Equality columns first, then range columns, then sort columns.

**SQL Server Example**:
```sql
-- Query:
SELECT OrderId, CustomerName, OrderTotal
FROM Orders
WHERE Status = 'Pending'      -- Equality
    AND OrderDate > '2024-01-01'  -- Range
ORDER BY OrderDate DESC;      -- Sort

-- Optimal index:
CREATE NONCLUSTERED INDEX IX_Orders_Status_OrderDate
ON Orders (Status, OrderDate DESC) -- Equality first, then range with sort direction
INCLUDE (OrderId, CustomerName, OrderTotal); -- Covering columns
```

**Why**: 
- Equality columns narrow down the search space first (most selective)
- Range columns can use the sorted nature of the index
- Sort direction in index eliminates separate sort operation

### 2. Covering Indexes (Include Columns)

**SQL Server**:
```sql
-- Without covering index (Key Lookup required)
CREATE INDEX IX_Orders_Status ON Orders (Status);

-- With covering index (no Key Lookup)
CREATE INDEX IX_Orders_Status_Covering
ON Orders (Status)
INCLUDE (OrderDate, CustomerName, OrderTotal);
```

**PostgreSQL** (no INCLUDE clause, add to index):
```sql
-- Covering index (all columns in index)
CREATE INDEX ix_orders_status_covering
ON orders (status, order_date, customer_name, order_total);
```

**Trade-off**: Covering indexes are larger and slower to maintain but eliminate Key Lookups.

### 3. Selectivity

**High Selectivity** (good for indexes):
- Unique or near-unique columns (Email, OrderNumber)
- Status fields with many distinct values
- Foreign keys

**Low Selectivity** (poor for indexes):
- Boolean flags (IsActive, IsDeleted) with 50/50 distribution
- Gender fields with 2-3 values
- Status fields with only 2-3 states

**Exception**: Low selectivity can work if you're filtering for the rare case:
```sql
-- If only 1% of orders are "Cancelled", index on Status helps
WHERE Status = 'Cancelled'

-- But if 50% of orders are "Pending", index doesn't help much
WHERE Status = 'Pending'
```

---

## Common Index Patterns

### Pattern 1: Foreign Key Indexes

**Always index foreign keys** unless table is tiny (<1000 rows).

```sql
-- SQL Server
CREATE NONCLUSTERED INDEX IX_Orders_CustomerId
ON Orders (CustomerId);

-- PostgreSQL
CREATE INDEX ix_orders_customer_id
ON orders (customer_id);
```

**Why**: JOINs on foreign keys are extremely common. Without index, nested loop joins are slow.

### Pattern 2: Filtered Indexes (SQL Server) / Partial Indexes (PostgreSQL)

**Use Case**: Index only subset of rows that are frequently queried.

**SQL Server**:
```sql
-- Only index active orders
CREATE NONCLUSTERED INDEX IX_Orders_ActiveOnly
ON Orders (OrderDate DESC, CustomerId)
WHERE Status IN ('Pending', 'Processing');
```

**PostgreSQL**:
```sql
CREATE INDEX ix_orders_active_only
ON orders (order_date DESC, customer_id)
WHERE status IN ('Pending', 'Processing');
```

**Benefits**:
- Smaller index (faster seeks, less storage)
- Reduced maintenance overhead (updates to cancelled orders don't touch index)

### Pattern 3: Composite Indexes for Multi-Column Filters

**Query**:
```sql
WHERE Country = 'USA' AND State = 'CA' AND City = 'SF'
```

**Index Order** (most selective first):
```sql
-- Option 1: Selectivity order (if City is most selective)
CREATE INDEX IX_Users_City_State_Country
ON Users (City, State, Country);

-- Option 2: Query order (if all equally selective)
CREATE INDEX IX_Users_Country_State_City
ON Users (Country, State, City);
```

**Rule of Thumb**: Order by selectivity (most selective first) unless there's a range or sort involved.

### Pattern 4: Index for ORDER BY

**Query**:
```sql
SELECT * FROM Orders
WHERE CustomerId = 123
ORDER BY OrderDate DESC;
```

**Index**:
```sql
-- SQL Server
CREATE INDEX IX_Orders_CustomerId_OrderDate
ON Orders (CustomerId, OrderDate DESC);

-- PostgreSQL
CREATE INDEX ix_orders_customer_id_order_date
ON orders (customer_id, order_date DESC);
```

**Impact**: Eliminates sort operation (can be expensive for large result sets).

### Pattern 5: Index for GROUP BY / DISTINCT

**Query**:
```sql
SELECT CustomerId, COUNT(*)
FROM Orders
WHERE OrderDate > '2024-01-01'
GROUP BY CustomerId;
```

**Index**:
```sql
CREATE INDEX IX_Orders_OrderDate_CustomerId
ON Orders (OrderDate, CustomerId);
```

**Why**: Index supports both the filter (OrderDate) and the grouping (CustomerId).

---

## SQL Server Specific Patterns

### Clustered Index Selection

**Best Candidates** (in order):
1. **Primary Key** (if narrow and sequential)
2. **Date/Time column** for time-series data (Orders.OrderDate)
3. **Identity column** (auto-increment)

**Avoid**:
- Wide keys (multiple columns, > 16 bytes)
- Frequently updated columns (causes page splits)
- Random values (GUIDs) → heavy fragmentation

**Example**:
```sql
-- Good: Sequential clustered index
CREATE CLUSTERED INDEX IX_Orders_OrderDate
ON Orders (OrderDate);

-- Bad: Random GUID clustered index
CREATE CLUSTERED INDEX IX_Orders_OrderGuid
ON Orders (OrderGuid); -- Causes fragmentation
```

### Included Columns vs Index Columns

**Indexed Columns** (key columns):
- Used for seeks and range scans
- Used for sorting
- Stored in all index levels (leaf and intermediate)
- Count toward 16-column limit

**Included Columns** (INCLUDE):
- Only stored in leaf level
- Not used for seeks or sorts
- Can be wide (e.g., VARCHAR(MAX), large NVARCHAR)
- Don't count toward 16-column limit

**Guideline**:
- Filters, joins, sorts → **key columns**
- SELECT projections → **INCLUDE columns**

### Columnstore Indexes for Analytics

**Use Case**: Large fact tables (millions of rows) with analytical queries (aggregations, scans).

```sql
-- Clustered columnstore for data warehouse fact tables
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactSales
ON FactSales;

-- Nonclustered columnstore for reporting on OLTP table
CREATE NONCLUSTERED COLUMNSTORE INDEX NCI_Orders_Analytics
ON Orders (OrderDate, CustomerId, ProductId, OrderTotal, Quantity);
```

**Benefits**:
- 10x compression
- 10-100x faster for aggregations
- Batch mode execution

**Trade-offs**:
- Not for OLTP (row-level updates slow)
- Requires SQL Server 2016+ (Enterprise for full features)

---

## PostgreSQL Specific Patterns

### Index Types

**1. B-tree (default)** - Use for most cases:
```sql
CREATE INDEX ix_orders_order_date ON orders (order_date);
```

**2. Hash** - Equality only (=), faster than B-tree:
```sql
CREATE INDEX ix_users_email_hash ON users USING HASH (email);
```
Use case: Exact lookups, no range scans.

**3. GIN (Generalized Inverted Index)** - For array, JSONB, full-text:
```sql
-- JSONB column
CREATE INDEX ix_orders_metadata_gin ON orders USING GIN (metadata);

-- Array column
CREATE INDEX ix_posts_tags_gin ON posts USING GIN (tags);
```

**4. GiST (Generalized Search Tree)** - For geometric data, ranges:
```sql
-- Range types
CREATE INDEX ix_bookings_date_range_gist ON bookings USING GIST (date_range);
```

**5. BRIN (Block Range Index)** - For very large tables with natural order:
```sql
-- Time-series data (e.g., 100M rows)
CREATE INDEX ix_events_created_at_brin ON events USING BRIN (created_at);
```
Very small index, works well when data is naturally sorted (append-only logs).

### Expression Indexes

**Use Case**: Function calls in WHERE clause.

**Query**:
```sql
WHERE LOWER(email) = 'user@example.com'
```

**Index**:
```sql
CREATE INDEX ix_users_email_lower ON users (LOWER(email));
```

**Query**:
```sql
WHERE date_trunc('day', created_at) = '2024-01-01'
```

**Index**:
```sql
CREATE INDEX ix_orders_created_day ON orders (date_trunc('day', created_at));
```

### Operator Class (Text Search)

**Pattern Matching**:
```sql
-- For LIKE 'prefix%' queries
CREATE INDEX ix_users_name_text_pattern ON users (name text_pattern_ops);

-- For case-insensitive searches
CREATE INDEX ix_users_email_citext ON users ((email::citext));
```

---

## Index Maintenance

### SQL Server

**Fragmentation Management**:
```sql
-- Check fragmentation
SELECT
    OBJECT_NAME(ips.object_id) AS table_name,
    i.name AS index_name,
    ips.avg_fragmentation_in_percent
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 10;

-- Rebuild if > 30% fragmented
ALTER INDEX IX_Orders_OrderDate ON Orders REBUILD;

-- Reorganize if 10-30% fragmented
ALTER INDEX IX_Orders_OrderDate ON Orders REORGANIZE;
```

**Update Statistics**:
```sql
UPDATE STATISTICS Orders WITH FULLSCAN;
```

### PostgreSQL

**Vacuum and Analyze**:
```sql
-- Reclaim space and update statistics
VACUUM ANALYZE orders;

-- Rebuild index (if bloated)
REINDEX INDEX CONCURRENTLY ix_orders_order_date;
```

**Autovacuum** (usually sufficient):
PostgreSQL has automatic vacuum, but tune if needed:
```sql
ALTER TABLE orders SET (autovacuum_vacuum_scale_factor = 0.1);
```

---

## Anti-Patterns and Pitfalls

### 1. Over-Indexing

**Problem**: Every index slows down INSERT/UPDATE/DELETE.

**Rule of Thumb**:
- No more than 5-7 non-clustered indexes per table (SQL Server)
- No more than 8-10 indexes per table (PostgreSQL)

**Solution**: Consolidate overlapping indexes.

**Bad**:
```sql
CREATE INDEX IX_Orders_CustomerId ON Orders (CustomerId);
CREATE INDEX IX_Orders_CustomerId_OrderDate ON Orders (CustomerId, OrderDate);
-- First index is redundant (covered by second)
```

**Good**:
```sql
-- Single index covers both queries
CREATE INDEX IX_Orders_CustomerId_OrderDate ON Orders (CustomerId, OrderDate);
```

### 2. Indexing Low-Selectivity Columns

**Bad**:
```sql
-- Only 2 distinct values, 50/50 split
CREATE INDEX IX_Users_IsActive ON Users (IsActive);
```

Most queries return ~50% of table, so full table scan is faster than index seek + lookups.

**Better**: Filtered index for the rare case:
```sql
-- Only 5% of users are inactive
CREATE INDEX IX_Users_Inactive ON Users (IsActive)
WHERE IsActive = 0;
```

### 3. Functions on Indexed Columns

**Bad**:
```sql
-- Can't use index on OrderDate
WHERE YEAR(OrderDate) = 2024
```

**Good**:
```sql
-- Can use index
WHERE OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01'
```

Or use computed column (SQL Server):
```sql
ALTER TABLE Orders ADD OrderYear AS YEAR(OrderDate) PERSISTED;
CREATE INDEX IX_Orders_OrderYear ON Orders (OrderYear);
```

Or expression index (PostgreSQL):
```sql
CREATE INDEX ix_orders_order_year ON orders (EXTRACT(YEAR FROM order_date));
```

### 4. GUID Primary Keys without NEWSEQUENTIALID

**Bad** (SQL Server):
```sql
-- Random GUIDs cause page splits
CREATE TABLE Orders (
    OrderId UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID()
);
```

**Better**:
```sql
-- Sequential GUIDs reduce fragmentation
CREATE TABLE Orders (
    OrderId UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID()
);
```

**Best**: Use INT IDENTITY for clustered index, GUID as alternate key:
```sql
CREATE TABLE Orders (
    OrderId INT IDENTITY PRIMARY KEY,
    OrderGuid UNIQUEIDENTIFIER UNIQUE DEFAULT NEWSEQUENTIALID()
);
```

---

## Index Selection Decision Tree

```
Is it a foreign key?
├─ Yes → Create index (IX_Table_ForeignKeyColumn)
└─ No → Continue

Is it in WHERE clause?
├─ Yes → Check selectivity
│   ├─ High (> 5% distinct values) → Good candidate
│   └─ Low (< 5% distinct values) → Consider filtered index or skip
└─ No → Continue

Is it in JOIN ON clause?
├─ Yes → Create index (critical for joins)
└─ No → Continue

Is it in ORDER BY or GROUP BY?
├─ Yes → Include in index (after filter columns)
└─ No → Continue

Are there multiple filter columns?
├─ Yes → Composite index (equality first, range second)
└─ No → Single column index

Are there SELECT columns not in index?
├─ Yes → Add as INCLUDE columns (if < 5 columns)
└─ No → Index is complete
```

---

## Monitoring and Validation

### Verify Index Usage (SQL Server)

```sql
-- Check if index is used
SELECT
    OBJECT_NAME(s.object_id) AS table_name,
    i.name AS index_name,
    s.user_seeks + s.user_scans + s.user_lookups AS total_reads,
    s.user_updates AS writes,
    CASE
        WHEN s.user_seeks + s.user_scans + s.user_lookups = 0 THEN 'UNUSED'
        WHEN s.user_updates > (s.user_seeks + s.user_scans + s.user_lookups) * 10 THEN 'WRITE-HEAVY'
        ELSE 'ACTIVE'
    END AS status
FROM sys.dm_db_index_usage_stats s
INNER JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
WHERE OBJECT_NAME(s.object_id) = 'Orders';
```

### Verify Index Usage (PostgreSQL)

```sql
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE tablename = 'orders';
```

---

## Summary Checklist

Before creating an index:

✅ Check if existing index already covers the query  
✅ Verify column selectivity (> 5% distinct values)  
✅ Consider query frequency (hot path vs cold path)  
✅ Estimate index size and maintenance cost  
✅ Test with realistic data volume  
✅ Monitor index usage after creation  
✅ Remove unused indexes after 30 days  

Index design is iterative - start conservative, add indexes based on real usage patterns.
