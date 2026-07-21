---
name: full-audit
description: "Comprehensive performance audit of entire project covering C# code, EF Core queries, database optimization, stored procedures, and API endpoints. Generates detailed report with prioritized recommendations."
---

# Full Performance Audit

Perform a comprehensive performance analysis across your entire .NET project.

## What This Does

Executes a full-stack performance audit including:

1. **C# Code Analysis** - Scans all C# files for performance anti-patterns
2. **EF Core Query Analysis** - Identifies N+1 queries, missing eager loading, inefficient queries
3. **Database Optimization** - Analyzes queries via MCP, recommends indexes, detects table scans
4. **Stored Procedure Analysis** - Reviews SP definitions, execution stats, detects cursors and row-by-row operations
5. **API Endpoint Analysis** - Examines controller endpoints for sync blocking, missing pagination, over-fetching
6. **Report Generation** - Consolidates findings into prioritized recommendations

## Prerequisites

**Before running this audit:**

1. **MCP Database Connection**: Ensure `.vscode/mcp.json` is configured with database credentials
2. **Database Permissions**: 
   - SQL Server: User needs `VIEW SERVER STATE` permission
   - PostgreSQL: Read permissions on `pg_stat_*` views
3. **Project Structure**: .NET project with C# files in workspace

## Usage

Simply invoke this prompt. The audit will:
- Automatically discover all relevant files
- Analyze each layer systematically
- Connect to databases via MCP to gather metrics
- Generate a comprehensive report

**Estimated Duration**: 5-15 minutes depending on project size

## What You'll Get

A detailed performance audit report including:

### Executive Summary
- Total findings count by severity (Critical/High/Medium/Low)
- Top 5 quick wins
- Estimated overall impact

### Findings by Category
- **C# Code**: String concatenation, boxing, sync-over-async, etc.
- **EF Core**: N+1 queries, missing eager loading, client-side evaluation
- **Database**: Missing indexes, table scans, suboptimal query plans
- **Stored Procedures**: Cursors, parameter sniffing, missing error handling
- **API Endpoints**: Missing pagination, no caching, over-fetching

### Priority Recommendations
- **Phase 1: Quick Wins** (< 2 hours effort, high impact)
- **Phase 2: Critical Fixes** (< 1 day effort, addresses all critical issues)
- **Phase 3: High Priority** (< 1 week effort, remaining high-priority items)
- **Phase 4: Long-term** (architectural improvements)

### Detailed Metrics
- Before/after performance projections
- Estimated improvements (response times, throughput, memory)
- Database query reduction estimates

## Example Output

```markdown
# Performance Audit Report

**Project**: MyECommerceApp
**Date**: 2024-04-29
**Total Findings**: 47 (8 Critical, 15 High, 18 Medium, 6 Low)

## Executive Summary

Addressing critical and high-priority issues will result in:
- 10x faster API response times (P95: 8s → 0.8s)
- 90% reduction in database queries (500+ → 50 per request)
- 60% reduction in memory allocations

### Top 5 Quick Wins
1. Add eager loading to Orders API - 5 min, 10x gain
2. Enable response compression - 1 line, 10x bandwidth reduction
3. Add pagination to Products endpoint - 10 min, prevents OOM
...

[Full detailed report follows]
```

## After the Audit

1. **Review findings** with your team
2. **Prioritize** based on impact and effort
3. **Implement Phase 1** (quick wins) first
4. **Measure impact** of each change
5. **Iterate** through remaining phases

## Customization

To focus on specific areas:
- **C# only**: Use `/csharp-perf-analysis`
- **Database only**: Use `/db-query-optimization` and `/stored-proc-analysis`
- **API only**: Use `/api-endpoint-analysis`

---

Ready to start the audit? Just confirm and I'll begin the comprehensive analysis.
