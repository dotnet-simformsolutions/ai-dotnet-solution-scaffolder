# Performance Audit Report Template

Use this template when generating performance audit reports.

---

# Performance Audit Report

**Project**: [Project Name]  
**Date**: [YYYY-MM-DD]  
**Analyzed By**: Performance Tuning Advisor  
**Scope**: [Describe scope - e.g., "Full-stack performance audit covering C#, EF Core, Database, and API layers"]

---

## Executive Summary

### Overview

This audit identified **[X] performance issues** across the application stack:

- **[N] Critical** issues [brief description]
- **[N] High** priority issues [brief description]
- **[N] Medium** priority issues [brief description]
- **[N] Low** priority issues [brief description]

### Estimated Impact

Addressing critical and high-priority issues will result in:

- **[Metric]**: [Before] → [After] ([X]x improvement)
- **[Metric]**: [Before] → [After] ([X]% reduction)
- **[Metric]**: [Improvement description]

### Top 5 Quick Wins

1. **[Issue Title]** - [Effort], [Impact]
2. **[Issue Title]** - [Effort], [Impact]
3. **[Issue Title]** - [Effort], [Impact]
4. **[Issue Title]** - [Effort], [Impact]
5. **[Issue Title]** - [Effort], [Impact]

---

## Findings by Category

### C# Code Performance ([N] findings)

#### Critical Issues ([N])

**C-[ID]: [Issue Title]** &#9650; Critical  
**Location**: [File.cs](path/to/file.cs#L[start]-L[end])  
**Problem**: [Description of anti-pattern]  
**Current Impact**: [Measurable impact - e.g., "Allocates 10,000 strings per request"]  
**Recommended Fix**:
```csharp
// Bad:
[code example]

// Good:
[code example]
```
**Estimated Improvement**: [Quantified improvement]  
**Effort**: [Low/Medium/High] ([time estimate])  
**Priority**: [P1/P2/P3/P4]

---

#### High Issues ([N])

[Repeat format above]

---

#### Medium Issues ([N])

[Repeat format above]

---

### EF Core Queries ([N] findings)

#### Critical Issues ([N])

**EF-[ID]: [Issue Title]** &#9650; Critical  
**Location**: [Controller.cs](path/to/controller.cs#L[start]-L[end])  
**Problem**: [Description of query issue]  
**Current SQL**:
```sql
[Generated SQL showing problem]
```
**Current Impact**: [Measurable impact - e.g., "Executes 101 queries per request"]  
**Recommended Fix**:
```csharp
[Fixed code with Include/AsNoTracking/etc.]
```
**Optimized SQL**:
```sql
[Improved SQL after fix]
```
**Estimated Improvement**: [Quantified improvement - e.g., "101 queries → 1 query"]  
**Effort**: [Low/Medium/High]  
**Priority**: [P1/P2/P3]

---

[Repeat for other severity levels]

---

### Database Optimization ([N] findings)

#### Critical Issues ([N])

**DB-[ID]: [Issue Title]** &#9650; Critical  
**Table/Query**: [Table name or query description]  
**Problem**: [Description - e.g., "Table scan on 500K rows"]  
**Execution Plan Analysis**:
```
[Relevant execution plan snippet showing problem]
```
**Current Performance**: [Metrics - e.g., "5 seconds, 50,000 logical reads"]  
**Recommended Index**:
```sql
CREATE [NONCLUSTERED] INDEX [Index_Name]
ON [Table] ([Columns])
[INCLUDE ([Covering_Columns])];
```
**Estimated Improvement**: [Quantified - e.g., "5s → 0.05s (100x faster)"]  
**Effort**: [Low/Medium/High]  
**Priority**: [P1/P2/P3]

---

[Repeat for other severity levels]

---

### Stored Procedures ([N] findings)

#### Critical Issues ([N])

**SP-[ID]: [Issue Title]** &#9650; Critical  
**Stored Procedure**: [Schema].[ProcedureName]  
**Location**: [Lines X-Y in procedure definition]  
**Problem**: [Description - e.g., "Cursor processing 1000+ rows"]  
**Current Code**:
```sql
[Problematic code snippet]
```
**Execution Statistics**:
- Execution count: [N]
- Average execution time: [X] ms
- Average logical reads: [N]

**Recommended Fix**:
```sql
[Set-based alternative or optimized code]
```
**Estimated Improvement**: [Quantified]  
**Effort**: [Low/Medium/High]  
**Priority**: [P1/P2/P3]

---

[Repeat for other severity levels]

---

### API Endpoints ([N] findings)

#### Critical Issues ([N])

**API-[ID]: [Issue Title]** &#9650; Critical  
**Endpoint**: [HTTP_METHOD /api/path]  
**Location**: [Controller.cs](path/to/controller.cs#L[start]-L[end])  
**Problem**: [Description]  
**Current Performance**:
- Average response time: [X] ms
- P95: [X] ms
- Database queries: [N]
- Payload size: [X] KB

**Recommended Fix**:
```csharp
[Fixed code]
```
**Estimated Improvement**: [Quantified - e.g., "3.5s → 0.3s (10x faster)"]  
**Effort**: [Low/Medium/High]  
**Priority**: [P1/P2/P3]

---

[Repeat for other severity levels]

---

## Priority Recommendations

### Phase 1: Quick Wins (This Week)

**Estimated Total Effort**: [X hours/days]  
**Estimated Impact**: [High-level impact description]

1. &#9650; **[ID]**: [Brief title and description] ([Effort])
2. &#9650; **[ID]**: [Brief title and description] ([Effort])
3. **[ID]**: [Brief title and description] ([Effort])
4. **[ID]**: [Brief title and description] ([Effort])
5. **[ID]**: [Brief title and description] ([Effort])

**Expected Outcome**: [What will be achieved after Phase 1]

---

### Phase 2: Critical Fixes (This Sprint)

**Estimated Total Effort**: [X hours/days]  
**Estimated Impact**: [Description]

1. &#9650; **[ID]**: [Title] ([Effort])
2. &#9650; **[ID]**: [Title] ([Effort])
[...]

**Expected Outcome**: [What will be achieved after Phase 2]

---

### Phase 3: High Priority (Next Sprint)

**Estimated Total Effort**: [X days]  
**Estimated Impact**: [Description]

[List of items]

**Expected Outcome**: [What will be achieved after Phase 3]

---

### Phase 4: Long-term Improvements (Next Quarter)

**Estimated Total Effort**: [X weeks]  
**Estimated Impact**: [Description]

[List of architectural improvements]

**Expected Outcome**: [Strategic improvements achieved]

---

## Detailed Metrics

### Before Optimization

| Metric | Current Value |
|--------|---------------|
| Average API response time (P50) | [X] ms |
| Average API response time (P95) | [X] ms |
| Average API response time (P99) | [X] ms |
| Database queries per request (avg) | [N] |
| Database queries per request (max) | [N] |
| Memory allocations per request | [X] MB |
| Throughput capacity | [N] req/sec |
| Error rate | [X]% |
| Database CPU utilization (avg) | [X]% |

### After Optimization (Projected)

| Metric | Target Value | Improvement |
|--------|--------------|-------------|
| Average API response time (P50) | [X] ms | [Y]x faster |
| Average API response time (P95) | [X] ms | [Y]x faster |
| Average API response time (P99) | [X] ms | [Y]x faster |
| Database queries per request (avg) | [N] | [Y]% reduction |
| Database queries per request (max) | [N] | [Y]% reduction |
| Memory allocations per request | [X] MB | [Y]% reduction |
| Throughput capacity | [N] req/sec | [Y]x increase |
| Error rate | [X]% | [Y]% reduction |
| Database CPU utilization (avg) | [X]% | [Y]% reduction |

---

## Implementation Notes

### Testing Strategy

1. **Establish Baseline**
   - Measure all key metrics before changes
   - Document current performance characteristics
   - Set up monitoring dashboards

2. **Incremental Implementation**
   - Implement fixes one at a time (or in small batches)
   - Measure impact of each change
   - Roll back if unexpected issues arise

3. **Load Testing**
   - Test after each phase completion
   - Validate improvements under realistic load
   - Identify any regressions

4. **Production Monitoring**
   - Deploy with feature flags if possible
   - Monitor key metrics closely
   - Have rollback plan ready

### Risk Assessment

| Change Type | Risk Level | Mitigation |
|-------------|------------|------------|
| Code-level fixes (async/await, StringBuilder) | Low | Code review + unit tests |
| EF Core query changes (Include, AsNoTracking) | Low-Medium | Integration tests + staging validation |
| Database indexes | Medium | Test in staging, monitor disk space, verify execution plans |
| Stored procedure rewrites | High | Extensive testing, gradual rollout, backup plan |
| API contract changes | High | Versioning, backward compatibility, communication |

### Success Metrics

Track these KPIs weekly during implementation:

- [ ] API response time percentiles (P50, P95, P99)
- [ ] Database CPU and I/O utilization
- [ ] Application memory usage
- [ ] Error rate and error types
- [ ] Throughput (requests per second)
- [ ] Database query counts (from EF Core logs)
- [ ] User-reported performance issues

**Success Criteria**:
- P95 response time < [target]
- Error rate < [target]%
- Database queries per request < [target]
- Memory allocations < [target] MB per request

---

## Appendix

### Tools and Resources

**Profiling**:
- [Tool names and links]

**Database Analysis**:
- [Tool names and links]

**Load Testing**:
- [Tool names and links]

**Monitoring**:
- [Tool names and links]

### Technical References

- [Link to C# Performance Best Practices]
- [Link to EF Core Performance Documentation]
- [Link to ASP.NET Core Performance Guide]
- [Link to Database-specific optimization guides]

### Team Contacts

- **Performance Lead**: [Name/Email]
- **Database Admin**: [Name/Email]
- **DevOps**: [Name/Email]

---

**Report Generated**: [Timestamp]  
**Report Version**: 1.0  
**Next Review**: [Date - recommend after Phase 2 completion]

---

## Change Log

| Date | Phase | Changes | Results |
|------|-------|---------|---------|
| [Date] | Baseline | Initial audit | [Findings count] |
| [Date] | Phase 1 | [Items completed] | [Improvements achieved] |
| [Date] | Phase 2 | [Items completed] | [Improvements achieved] |
