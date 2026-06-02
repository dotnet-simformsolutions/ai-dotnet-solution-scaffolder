# Stored Procedure Performance Anti-Patterns

Comprehensive guide to common stored procedure and database function performance issues for SQL Server and PostgreSQL.

---

## 1. Cursor Usage (Row-by-Row Processing)

### ❌ Bad: Cursor Processing

**SQL Server**:
```sql
CREATE PROCEDURE ProcessPendingOrders
AS
BEGIN
    DECLARE @OrderId INT, @CustomerId INT, @Total DECIMAL(10,2);
    
    DECLARE order_cursor CURSOR FOR
    SELECT OrderId, CustomerId, Total 
    FROM Orders 
    WHERE Status = 'Pending';
    
    OPEN order_cursor;
    FETCH NEXT FROM order_cursor INTO @OrderId, @CustomerId, @Total;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        -- Process each order individually
        UPDATE Orders SET ProcessedDate = GETDATE() WHERE OrderId = @OrderId;
        INSERT INTO AuditLog (OrderId, ProcessedDate) VALUES (@OrderId, GETDATE());
        
        FETCH NEXT FROM order_cursor INTO @OrderId, @CustomerId, @Total;
    END
    
    CLOSE order_cursor;
    DEALLOCATE order_cursor;
END
```

**PostgreSQL**:
```sql
CREATE OR REPLACE FUNCTION process_pending_orders()
RETURNS void AS $$
DECLARE
    order_rec RECORD;
    order_cursor CURSOR FOR
        SELECT order_id, customer_id, total
        FROM orders
        WHERE status = 'Pending';
BEGIN
    OPEN order_cursor;
    LOOP
        FETCH order_cursor INTO order_rec;
        EXIT WHEN NOT FOUND;
        
        -- Process each order
        UPDATE orders SET processed_date = NOW() WHERE order_id = order_rec.order_id;
        INSERT INTO audit_log (order_id, processed_date) VALUES (order_rec.order_id, NOW());
    END LOOP;
    CLOSE order_cursor;
END;
$$ LANGUAGE plpgsql;
```

**Impact**: 1000 orders → 2000+ database operations (1 UPDATE + 1 INSERT per order).

### ✅ Good: Set-Based Operation

**SQL Server**:
```sql
CREATE PROCEDURE ProcessPendingOrders
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Single UPDATE
    UPDATE Orders
    SET ProcessedDate = GETDATE()
    WHERE Status = 'Pending';
    
    -- Single INSERT with all rows
    INSERT INTO AuditLog (OrderId, ProcessedDate)
    SELECT OrderId, GETDATE()
    FROM Orders
    WHERE Status = 'Pending';
END
```

**PostgreSQL**:
```sql
CREATE OR REPLACE FUNCTION process_pending_orders()
RETURNS void AS $$
BEGIN
    -- Single UPDATE
    UPDATE orders
    SET processed_date = NOW()
    WHERE status = 'Pending';
    
    -- Single INSERT with all rows
    INSERT INTO audit_log (order_id, processed_date)
    SELECT order_id, NOW()
    FROM orders
    WHERE status = 'Pending';
END;
$$ LANGUAGE plpgsql;
```

**Impact**: 1000 orders → 2 database operations. **500x fewer operations**.

---

## 2. Missing SET NOCOUNT ON (SQL Server)

### ❌ Bad: Row Count Messages Returned

```sql
CREATE PROCEDURE GetActiveUsers
AS
BEGIN
    -- Missing SET NOCOUNT ON
    SELECT * FROM Users WHERE IsActive = 1; -- Returns "100 rows affected"
    SELECT * FROM UserRoles WHERE IsActive = 1; -- Returns "150 rows affected"
END
```

**Impact**: Application receives row count messages for each statement, increasing network traffic and processing time.

### ✅ Good: Suppress Row Count Messages

```sql
CREATE PROCEDURE GetActiveUsers
AS
BEGIN
    SET NOCOUNT ON; -- Suppress row counts
    
    SELECT * FROM Users WHERE IsActive = 1;
    SELECT * FROM UserRoles WHERE IsActive = 1;
END
```

**Impact**: 10-30% faster execution in tight loops due to reduced network traffic.

---

## 3. Scalar UDF in SELECT Statement

### ❌ Bad: Scalar Function Called Per Row

**SQL Server**:
```sql
-- Scalar function
CREATE FUNCTION dbo.CalculateDiscount(@Total DECIMAL(10,2))
RETURNS DECIMAL(10,2)
AS
BEGIN
    DECLARE @Discount DECIMAL(10,2);
    IF @Total > 1000 SET @Discount = @Total * 0.10;
    ELSE IF @Total > 500 SET @Discount = @Total * 0.05;
    ELSE SET @Discount = 0;
    RETURN @Discount;
END
GO

-- Used in query (called once per row)
SELECT 
    OrderId,
    Total,
    dbo.CalculateDiscount(Total) AS Discount -- Row-by-row execution!
FROM Orders;
```

**Impact**: 100,000 rows → 100,000 function calls. Forces row-by-row execution.

### ✅ Good: Inline Logic or Table-Valued Function

**Option 1: Inline in Query**:
```sql
SELECT 
    OrderId,
    Total,
    CASE 
        WHEN Total > 1000 THEN Total * 0.10
        WHEN Total > 500 THEN Total * 0.05
        ELSE 0
    END AS Discount -- Executes in single scan
FROM Orders;
```

**Option 2: Inline Table-Valued Function** (SQL Server 2019+):
```sql
CREATE FUNCTION dbo.CalculateDiscount_Inline(@Total DECIMAL(10,2))
RETURNS TABLE
AS RETURN
(
    SELECT CASE 
        WHEN @Total > 1000 THEN @Total * 0.10
        WHEN @Total > 500 THEN @Total * 0.05
        ELSE 0
    END AS Discount
);
GO

SELECT 
    OrderId,
    Total,
    d.Discount
FROM Orders
CROSS APPLY dbo.CalculateDiscount_Inline(Total) d;
```

**Impact**: 100,000 rows with inline logic → single table scan. **100x+ faster**.

---

## 4. SELECT * in Stored Procedures

### ❌ Bad: Fetching All Columns

```sql
CREATE PROCEDURE GetUserOrders
    @UserId INT
AS
BEGIN
    SELECT * FROM Orders WHERE UserId = @UserId;
    -- Breaks if schema changes
    -- Fetches unnecessary columns
END
```

**Impact**:
- Network bandwidth waste (unused columns)
- Breaks when columns added/removed
- Prevents covering indexes

### ✅ Good: Explicit Column List

```sql
CREATE PROCEDURE GetUserOrders
    @UserId INT
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT 
        OrderId,
        OrderDate,
        OrderTotal,
        Status
    FROM Orders 
    WHERE UserId = @UserId;
END
```

**Impact**: Faster execution, smaller result sets, covering indexes possible.

---

## 5. WHILE Loop Row-by-Row Processing

### ❌ Bad: RBAR (Row-By-Agonizing-Row)

**SQL Server**:
```sql
CREATE PROCEDURE UpdateOrderTotals
AS
BEGIN
    WHILE EXISTS (SELECT 1 FROM Orders WHERE TotalNeedsUpdate = 1)
    BEGIN
        DECLARE @OrderId INT;
        
        SELECT TOP 1 @OrderId = OrderId
        FROM Orders
        WHERE TotalNeedsUpdate = 1;
        
        -- Calculate and update
        UPDATE Orders
        SET OrderTotal = (SELECT SUM(LineTotal) FROM OrderLines WHERE OrderId = @OrderId),
            TotalNeedsUpdate = 0
        WHERE OrderId = @OrderId;
    END
END
```

**Impact**: 1000 orders → 1000 loops with SELECT TOP 1 + UPDATE. Very slow.

### ✅ Good: Single Set-Based UPDATE

```sql
CREATE PROCEDURE UpdateOrderTotals
AS
BEGIN
    SET NOCOUNT ON;
    
    UPDATE o
    SET o.OrderTotal = ol.LineTotal,
        o.TotalNeedsUpdate = 0
    FROM Orders o
    INNER JOIN (
        SELECT OrderId, SUM(LineTotal) AS LineTotal
        FROM OrderLines
        GROUP BY OrderId
    ) ol ON o.OrderId = ol.OrderId
    WHERE o.TotalNeedsUpdate = 1;
END
```

**Impact**: 1000 orders → 1 operation. **1000x faster**.

---

## 6. Temp Table Abuse

### ❌ Bad: Multiple Temp Tables for Small Data

```sql
CREATE PROCEDURE ProcessOrders
AS
BEGIN
    -- Creates 3 temp tables for small datasets
    SELECT OrderId, CustomerId INTO #TempOrders 
    FROM Orders WHERE Status = 'Pending';
    
    SELECT CustomerId, CustomerName INTO #TempCustomers
    FROM Customers WHERE CustomerId IN (SELECT CustomerId FROM #TempOrders);
    
    SELECT OrderId, ProductId INTO #TempOrderLines
    FROM OrderLines WHERE OrderId IN (SELECT OrderId FROM #TempOrders);
    
    -- ... processing
    
    DROP TABLE #TempOrders;
    DROP TABLE #TempCustomers;
    DROP TABLE #TempOrderLines;
END
```

**Impact**: tempdb contention, excessive I/O, locks on system tables.

### ✅ Good: CTEs or Table Variables

**Option 1: CTEs (Common Table Expressions)**:
```sql
CREATE PROCEDURE ProcessOrders
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH PendingOrders AS (
        SELECT OrderId, CustomerId FROM Orders WHERE Status = 'Pending'
    ),
    CustomerInfo AS (
        SELECT c.CustomerId, c.CustomerName
        FROM Customers c
        WHERE EXISTS (SELECT 1 FROM PendingOrders po WHERE po.CustomerId = c.CustomerId)
    )
    -- Single query using CTEs
    SELECT 
        po.OrderId,
        ci.CustomerName,
        ol.ProductId
    FROM PendingOrders po
    INNER JOIN CustomerInfo ci ON po.CustomerId = ci.CustomerId
    INNER JOIN OrderLines ol ON po.OrderId = ol.OrderId;
END
```

**Option 2: Table Variables (for < 1000 rows)**:
```sql
DECLARE @PendingOrders TABLE (OrderId INT, CustomerId INT);
INSERT INTO @PendingOrders
SELECT OrderId, CustomerId FROM Orders WHERE Status = 'Pending';
```

**Impact**: Reduced tempdb pressure, simpler code, often faster for small datasets.

---

## 7. Dynamic SQL Without Parameterization

### ❌ Bad: SQL Injection Risk + Plan Cache Pollution

```sql
CREATE PROCEDURE SearchUsers
    @SearchTerm NVARCHAR(100)
AS
BEGIN
    DECLARE @SQL NVARCHAR(MAX);
    
    -- CRITICAL VULNERABILITY!
    SET @SQL = 'SELECT * FROM Users WHERE UserName = ''' + @SearchTerm + '''';
    
    EXEC(@SQL);
END

-- If called with: EXEC SearchUsers 'admin'' OR ''1''=''1'
-- Executes: SELECT * FROM Users WHERE UserName = 'admin' OR '1'='1'
-- Returns ALL users!
```

**Impact**: SQL injection, plan cache pollution (new plan for each value).

### ✅ Good: Parameterized with sp_executesql

```sql
CREATE PROCEDURE SearchUsers
    @SearchTerm NVARCHAR(100)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @SQL NVARCHAR(MAX);
    
    SET @SQL = 'SELECT UserId, UserName, Email FROM Users WHERE UserName = @UserName';
    
    EXEC sp_executesql @SQL, N'@UserName NVARCHAR(100)', @UserName = @SearchTerm;
END
```

Or better yet, avoid dynamic SQL if possible:
```sql
CREATE PROCEDURE SearchUsers
    @SearchTerm NVARCHAR(100)
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT UserId, UserName, Email
    FROM Users
    WHERE UserName = @SearchTerm;
END
```

---

## 8. Parameter Sniffing Issues (SQL Server)

### Problem: Cached Plan Not Optimal for All Parameter Values

```sql
CREATE PROCEDURE GetOrdersByDate
    @StartDate DATE
AS
BEGIN
    SELECT OrderId, CustomerId, OrderTotal
    FROM Orders
    WHERE OrderDate >= @StartDate;
END

-- First call: @StartDate = '2020-01-01' (returns 1M rows)
--   → Plan cached: Table Scan
-- Second call: @StartDate = '2024-04-28' (returns 10 rows)
--   → Uses same plan: Table Scan (inefficient for 10 rows!)
```

**Impact**: Inconsistent performance. Same query 10ms vs 10,000ms depending on cached plan.

### ✅ Solutions

**Option 1: OPTION (RECOMPILE)** - Recompile on each execution:
```sql
CREATE PROCEDURE GetOrdersByDate
    @StartDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT OrderId, CustomerId, OrderTotal
    FROM Orders
    WHERE OrderDate >= @StartDate
    OPTION (RECOMPILE); -- Fresh plan each time
END
```

**Option 2: OPTIMIZE FOR** - Optimize for specific value:
```sql
CREATE PROCEDURE GetOrdersByDate
    @StartDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT OrderId, CustomerId, OrderTotal
    FROM Orders
    WHERE OrderDate >= @StartDate
    OPTION (OPTIMIZE FOR (@StartDate = '2024-01-01')); -- Optimize for recent dates
END
```

**Option 3: Local Variable** - Prevents parameter sniffing:
```sql
CREATE PROCEDURE GetOrdersByDate
    @StartDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @LocalStartDate DATE = @StartDate; -- Copy to local variable
    
    SELECT OrderId, CustomerId, OrderTotal
    FROM Orders
    WHERE OrderDate >= @LocalStartDate; -- Optimizer uses statistics, not parameter value
END
```

**Option 4: Split into Multiple Procedures** - If workload clearly divided:
```sql
-- For historical queries (> 1 year old)
CREATE PROCEDURE GetHistoricalOrders ...

-- For recent queries (< 1 month)
CREATE PROCEDURE GetRecentOrders ...
```

---

## 9. Missing Error Handling

### ❌ Bad: No Error Handling

**SQL Server**:
```sql
CREATE PROCEDURE TransferFunds
    @FromAccount INT,
    @ToAccount INT,
    @Amount DECIMAL(10,2)
AS
BEGIN
    -- No error handling!
    UPDATE Accounts SET Balance = Balance - @Amount WHERE AccountId = @FromAccount;
    UPDATE Accounts SET Balance = Balance + @Amount WHERE AccountId = @ToAccount;
    -- If second UPDATE fails, first one commits → data corruption!
END
```

### ✅ Good: Transaction with Error Handling

**SQL Server**:
```sql
CREATE PROCEDURE TransferFunds
    @FromAccount INT,
    @ToAccount INT,
    @Amount DECIMAL(10,2)
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        UPDATE Accounts SET Balance = Balance - @Amount WHERE AccountId = @FromAccount;
        
        IF @@ROWCOUNT = 0
            THROW 50001, 'Source account not found', 1;
            
        UPDATE Accounts SET Balance = Balance + @Amount WHERE AccountId = @ToAccount;
        
        IF @@ROWCOUNT = 0
            THROW 50002, 'Destination account not found', 1;
        
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        
        -- Log error
        INSERT INTO ErrorLog (ErrorMessage, ErrorDate)
        VALUES (ERROR_MESSAGE(), GETDATE());
        
        -- Re-throw
        THROW;
    END CATCH
END
```

**PostgreSQL**:
```sql
CREATE OR REPLACE FUNCTION transfer_funds(
    from_account INT,
    to_account INT,
    amount DECIMAL(10,2)
) RETURNS void AS $$
BEGIN
    UPDATE accounts SET balance = balance - amount WHERE account_id = from_account;
    
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Source account not found';
    END IF;
    
    UPDATE accounts SET balance = balance + amount WHERE account_id = to_account;
    
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Destination account not found';
    END IF;
    
EXCEPTION
    WHEN OTHERS THEN
        -- Log error
        INSERT INTO error_log (error_message, error_date)
        VALUES (SQLERRM, NOW());
        
        RAISE; -- Re-raise
END;
$$ LANGUAGE plpgsql;
```

---

## 10. Nested Stored Procedure Chains

### ❌ Bad: Deep Nesting

```sql
CREATE PROCEDURE ProcessOrder
    @OrderId INT
AS
BEGIN
    EXEC ValidateOrder @OrderId; -- Level 2
    EXEC CalculateShipping @OrderId; -- Level 2
    EXEC ApplyDiscounts @OrderId; -- Level 2
    EXEC FinalizeOrder @OrderId; -- Level 2 (which calls SendEmail, which calls LogActivity...)
END
```

**Impact**:
- Hard to debug (stack traces span multiple procedures)
- Hidden dependencies
- Overhead of multiple EXEC calls
- Difficult to optimize (each procedure optimized separately)

### ✅ Good: Flatten or Use Inline Logic

**Option 1: Inline Logic**:
```sql
CREATE PROCEDURE ProcessOrder
    @OrderId INT
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- Validation
        IF NOT EXISTS (SELECT 1 FROM Orders WHERE OrderId = @OrderId AND Status = 'Pending')
            THROW 50001, 'Invalid order', 1;
        
        -- Calculate shipping (inline or single function call)
        UPDATE Orders
        SET ShippingCost = CASE
            WHEN Total > 100 THEN 0
            WHEN Total > 50 THEN 5.00
            ELSE 10.00
        END
        WHERE OrderId = @OrderId;
        
        -- Apply discounts (inline)
        UPDATE Orders
        SET Discount = Total * 0.10
        WHERE OrderId = @OrderId AND Total > 500;
        
        -- Finalize
        UPDATE Orders SET Status = 'Processed' WHERE OrderId = @OrderId;
        
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END
```

**Option 2: Table-Valued Functions for Reusable Logic**:
```sql
-- Reusable function for shipping calculation
CREATE FUNCTION dbo.CalculateShipping(@Total DECIMAL(10,2))
RETURNS TABLE
AS RETURN
(
    SELECT CASE
        WHEN @Total > 100 THEN 0
        WHEN @Total > 50 THEN 5.00
        ELSE 10.00
    END AS ShippingCost
);
GO

-- Use in procedure
UPDATE Orders
SET ShippingCost = s.ShippingCost
FROM Orders o
CROSS APPLY dbo.CalculateShipping(o.Total) s
WHERE o.OrderId = @OrderId;
```

---

## Summary Table

| Anti-Pattern | Severity | Typical Impact | Fix |
|--------------|----------|----------------|-----|
| Cursors on large datasets | Critical | 100-1000x slower | Set-based operations |
| Missing SET NOCOUNT ON | Medium | 10-30% overhead | Add SET NOCOUNT ON |
| Scalar UDF in SELECT | Critical | Row-by-row execution | Inline logic or iTVF |
| SELECT * | Medium | Bandwidth waste | Explicit columns |
| WHILE loops (RBAR) | Critical | 100-1000x slower | Set-based UPDATE |
| Temp table abuse | Medium | tempdb contention | CTEs or table variables |
| Dynamic SQL injection | Critical | Security breach | Parameterize with sp_executesql |
| Parameter sniffing | High | Inconsistent perf | RECOMPILE or OPTIMIZE FOR |
| Missing error handling | Critical | Data corruption | TRY/CATCH with transactions |
| Deep SP nesting | Medium | Debugging hell | Flatten or inline |

---

## Detection Checklist

When reviewing stored procedures:

1. ✅ Does it use cursors? (Search for `DECLARE...CURSOR`, `FETCH`)
2. ✅ Does it start with `SET NOCOUNT ON`? (SQL Server)
3. ✅ Does it call scalar functions in SELECT? (Search for `dbo.FunctionName(column)`)
4. ✅ Does it use `SELECT *`?
5. ✅ Does it have WHILE loops processing rows?
6. ✅ Does it create multiple temp tables for small data?
7. ✅ Does it use dynamic SQL? If yes, is it parameterized?
8. ✅ Are there parameter sniffing symptoms (inconsistent performance)?
9. ✅ Does it have TRY/CATCH error handling for modifications?
10. ✅ Does it nest > 3 levels deep?

---

## Performance Testing

Before and after fixes:

```sql
-- SQL Server
SET STATISTICS TIME ON;
SET STATISTICS IO ON;
EXEC dbo.YourProcedure @Param = 'Value';
SET STATISTICS TIME OFF;
SET STATISTICS IO OFF;

-- PostgreSQL
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM your_function('Value');
```

Measure:
- **Execution time** (milliseconds)
- **Logical reads** (SQL Server) or **Buffer hits** (PostgreSQL)
- **CPU time** (SQL Server worker time)
- **Row count** (ensure results match before/after)
