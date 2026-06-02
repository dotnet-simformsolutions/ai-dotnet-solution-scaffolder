---
name: perf-report-generator
description: 'Generate comprehensive performance audit reports consolidating findings from C# code analysis, EF Core queries, database optimization, stored procedure analysis, and API endpoints. Creates structured Markdown reports with severity rankings, priority recommendations, and impact estimates. Use when completing performance audits.'
user-invocable: true
---

# Performance Report Generator

## When to Use

- After completing full performance analysis across multiple domains
- Consolidating findings from other skills into single report
- Providing client/team with actionable performance audit
- Documenting performance improvement roadmap
- Creating executive summary of performance issues

## Report Structure

Generate reports following this structure:

### 1. Executive Summary

- Total findings count by severity
- Estimated overall impact
- Top 3-5 quick wins
- High-level recommendations

### 2. Findings by Category

Group findings into categories:
- **C# Code Performance** (from `/csharp-perf-analysis`)
- **EF Core Queries** (from `/efcore-query-analysis`)
- **Database Optimization** (from `/db-query-optimization`)
- **Stored Procedures** (from `/stored-proc-analysis`)
- **API Endpoints** (from `/api-endpoint-analysis`)

### 3. Priority Recommendations

Rank actionable items by:
- Impact (high/medium/low)
- Effort (low/medium/high)
- Priority = Impact / Effort

Focus on high-impact, low-effort wins first.

### 4. Implementation Roadmap

Suggested phases:
- **Phase 1: Quick Wins** (low effort, high impact)
- **Phase 2: Critical Fixes** (high impact, any effort)
- **Phase 3: Medium Priority** (medium impact)
- **Phase 4: Long-term Improvements** (architectural changes)

## Report Generation Procedure

### 1. Collect All Findings

Gather findings from all analysis skills:

```markdown
// From each skill, extract:
- Severity (Critical/High/Medium/Low)
- Category (C#/EF Core/Database/SP/API)
- Location (file and line numbers)
- Problem description
- Current impact
- Recommended fix
- Estimated improvement
```

### 2. Deduplicate and Consolidate

Check for overlapping findings:
- N+1 query detected in both EF Core analysis AND API endpoint analysis → consolidate
- Missing index recommended in both DB optimization AND EF Core analysis → single recommendation

### 3. Calculate Aggregate Metrics

```markdown
Total Findings: 47
- Critical: 8
- High: 15
- Medium: 18
- Low: 6

Estimated Improvements:
- Database queries: 500+ queries → 50 queries (90% reduction)
- API response times: P95 of 8s → P95 of 0.8s (10x faster)
- Memory allocations: Reduced by ~60%
- Thread pool efficiency: +200% capacity
```

### 4. Prioritize Recommendations

Use this matrix:

| Impact | Effort Low | Effort Medium | Effort High |
|--------|-----------|---------------|-------------|
| **High** | P1 (DO NOW) | P2 (THIS SPRINT) | P3 (NEXT SPRINT) |
| **Medium** | P2 (THIS SPRINT) | P3 (NEXT SPRINT) | P4 (BACKLOG) |
| **Low** | P3 (NEXT SPRINT) | P4 (BACKLOG) | P5 (DEFER) |

### 5. Generate Report Using Template

Load [report-template.md](./references/report-template.md) and populate with findings.

## Output Format

```markdown
# Performance Audit Report

**Project**: [Project Name]  
**Date**: YYYY-MM-DD  
**Analyzed By**: Performance Tuning Advisor  
**Scope**: Full-stack performance audit (C#, EF Core, Database, API)

---

## Executive Summary

### Overview

This audit identified **47 performance issues** across the application stack:

- **8 Critical** issues causing significant performance degradation
- **15 High** priority issues with measurable impact
- **18 Medium** priority optimization opportunities
- **6 Low** priority minor improvements

### Estimated Impact

Addressing critical and high-priority issues will result in:

- **10x faster API response times** (P95: 8s → 0.8s)
- **90% reduction in database queries** (500+ → 50 per request)
- **60% reduction in memory allocations**
- **3x increase in throughput** capacity

### Top 5 Quick Wins

1. **Add eager loading to Orders API** - 5 min fix, 10x performance gain
2. **Enable response compression** - 1 line change, 10x bandwidth reduction
3. **Add pagination to Products endpoint** - 10 min fix, prevents OOM
4. **Replace string concatenation in report generator** - 5 min fix, 100x faster
5. **Add missing indexes on foreign keys** - 15 min DDL, 50x query speedup

---

## Findings by Category

### C# Code Performance (12 findings)

#### Critical Issues (2)

**C-1: String Concatenation in Loop** &#9650; Critical  
**Location**: [ReportGenerator.cs](src/Services/ReportGenerator.cs#L45-L52)  
**Impact**: O(N²) allocations, 100ms → 1ms with fix  
**Fix**: Replace with StringBuilder  
**Effort**: Low (5 minutes)  
**Priority**: P1

[...more findings...]

#### High Issues (4)

[...findings...]

### EF Core Queries (15 findings)

#### Critical Issues (3)

**EF-1: N+1 Query in GetOrders** &#9650; Critical  
**Location**: [OrdersController.cs](src/Controllers/OrdersController.cs#L45-L55)  
**Impact**: 101 queries per request → 1 query with fix  
**Fix**: Add .Include(o => o.Customer)  
**Effort**: Low (2 minutes)  
**Priority**: P1

[...more findings...]

### Database Optimization (8 findings)

#### Critical Issues (1)

**DB-1: Missing Index on Orders.OrderDate** &#9650; Critical  
**Impact**: Table scan on 500K rows, 5s query time  
**Fix**: CREATE INDEX IX_Orders_OrderDate ON Orders (OrderDate DESC)  
**Effort**: Low (1 minute)  
**Priority**: P1

[...more findings...]

### Stored Procedures (5 findings)

#### High Issues (3)

**SP-1: Cursor in ProcessPendingOrders** &#9650; High  
**Location**: dbo.ProcessPendingOrders, lines 15-45  
**Impact**: 1000 iterations, 450ms avg execution  
**Fix**: Replace with set-based UPDATE  
**Effort**: Medium (30 minutes)  
**Priority**: P2

[...more findings...]

### API Endpoints (7 findings)

#### Critical Issues (2)

**API-1: Missing Pagination on GET /api/products** &#9650; Critical  
**Location**: [ProductsController.cs](src/Controllers/ProductsController.cs#L28-L33)  
**Impact**: Returns 50K products, OOM risk, 10s response time  
**Fix**: Add Skip/Take with page parameters  
**Effort**: Low (10 minutes)  
**Priority**: P1

[...more findings...]

---

## Priority Recommendations

### Phase 1: Quick Wins (This Week)

**Estimated Total Effort**: 2 hours  
**Estimated Impact**: 5-10x performance improvement

1. &#9650; **C-1**: String concatenation → StringBuilder (5 min)
2. &#9650; **EF-1**: Add eager loading to GetOrders (2 min)
3. &#9650; **DB-1**: Add index on Orders.OrderDate (1 min)
4. &#9650; **API-1**: Add pagination to Products endpoint (10 min)
5. &#9650; **API-2**: Enable response compression globally (5 min)
6. **EF-3**: Add AsNoTracking to read-only queries (15 min)
7. **DB-2**: Add indexes on foreign keys (15 min)

### Phase 2: Critical Fixes (This Sprint)

**Estimated Total Effort**: 1 day  
**Estimated Impact**: Address all critical issues

1. &#9650; **C-2**: Fix sync-over-async in AuthService (30 min)
2. **SP-1**: Replace cursor in ProcessPendingOrders (30 min)
3. **EF-2**: Fix N+1 in customer dashboard (20 min)
4. **API-3**: Add CancellationToken to long-running endpoints (1 hour)
5. **DB-3**: Optimize GetReportData stored procedure (2 hours)

### Phase 3: High Priority (Next Sprint)

**Estimated Total Effort**: 3 days  
**Estimated Impact**: Eliminate remaining high-priority issues

[...15 items...]

### Phase 4: Long-term Improvements (Next Quarter)

**Estimated Total Effort**: 2 weeks  
**Estimated Impact**: Architectural improvements, scalability

[...items...]

---

## Detailed Metrics

### Before Optimization

- **Average API response time (P95)**: 8.2 seconds
- **Database queries per request**: 150-500
- **Memory allocations per request**: ~5 MB
- **Throughput capacity**: ~50 req/sec
- **Error rate**: 2.5% (timeouts)

### After Optimization (Projected)

- **Average API response time (P95)**: 0.8 seconds (10x improvement)
- **Database queries per request**: 5-10 (95% reduction)
- **Memory allocations per request**: ~2 MB (60% reduction)
- **Throughput capacity**: ~500 req/sec (10x improvement)
- **Error rate**: <0.1% (95% reduction)

---

## Implementation Notes

### Testing Strategy

1. **Measure baseline** before any changes (use Application Insights, logs)
2. **Implement one fix at a time** to isolate impact
3. **Load test** after each phase
4. **Monitor production** after deployment

### Risk Assessment

- **Low risk**: Code-level fixes (string builder, eager loading, async/await)
- **Medium risk**: Database indexes (test in staging, monitor disk space)
- **High risk**: Stored procedure rewrites (extensive testing required)

### Success Metrics

Track these KPIs weekly:

- API response time percentiles (P50, P95, P99)
- Database CPU utilization
- Application memory usage
- Error rate
- User-reported performance issues

---

## Appendix

### Tools and Resources

- **Profiling**: dotTrace, PerfView, Application Insights
- **Database**: SQL Server Profiler, pg_stat_statements
- **Load Testing**: k6, JMeter, Azure Load Testing
- **Monitoring**: Application Insights, Grafana, Datadog

### References

- [C# Performance Best Practices](https://learn.microsoft.com/en-us/dotnet/csharp/advanced-topics/performance/)
- [EF Core Performance](https://learn.microsoft.com/en-us/ef/core/performance/)
- [ASP.NET Core Performance Best Practices](https://learn.microsoft.com/en-us/aspnet/core/performance/performance-best-practices)

---

**Report Generated**: 2024-04-29  
**Next Review**: Recommended after Phase 2 completion
```

## Severity Legend

Use these markers in reports:

- &#9650; **Critical**: Production issues, deadlocks, OOM, security concerns
- **High**: Measurable performance impact, scalability bottlenecks
- **Medium**: Optimization opportunities, code quality
- **Low**: Minor improvements, code style

## Auto-Generation

When invoked, automatically:

1. Aggregate findings from current session
2. Deduplicate overlapping issues
3. Calculate metrics and estimates
4. Generate prioritized recommendations
5. Output structured Markdown report

## Integration

This skill is typically the **last step** in a full audit:

1. Run `/csharp-perf-analysis`
2. Run `/efcore-query-analysis`
3. Run `/db-query-optimization`
4. Run `/stored-proc-analysis`
5. Run `/api-endpoint-analysis`
6. **Run `/perf-report-generator`** → consolidates all findings

## Notes

- Keep executive summary concise (< 1 page)
- Link to specific files and line numbers
- Provide code examples for top fixes
- Include effort estimates to help planning
- Update report after each phase for progress tracking
