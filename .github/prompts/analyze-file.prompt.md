---
name: analyze-file
description: "Quick performance analysis of a single C# file. Detects common performance anti-patterns and provides immediate feedback with fix suggestions."
---

# Quick File Performance Analysis

Analyze the currently open or selected C# file for performance issues.

## What This Does

Scans a single file for common C# performance anti-patterns:
- String concatenation in loops
- Synchronous blocking (`.Result`, `.Wait()`)
- Missing eager loading (EF Core N+1 patterns)
- Over-allocation in loops
- Missing `using` statements
- Inefficient LINQ usage
- Boxing/unboxing issues

## Usage

1. Open or select a C# file
2. Invoke this prompt
3. Receive immediate analysis with severity-ranked findings
4. Get specific fix recommendations with code examples

## Output

You'll receive:
- **List of issues** found in the file, ranked by severity
- **Line numbers** where issues occur
- **Explanation** of why each pattern is problematic
- **Fix recommendations** with before/after code examples
- **Estimated impact** of each fix

## Example

**Input**: Analyzing `OrderService.cs`

**Output**:
```
Found 3 performance issues in OrderService.cs:

🔴 CRITICAL: Synchronous blocking in async method (Line 45)
   Problem: Using .Result on Task
   Fix: Replace with await
   Impact: Deadlock risk, thread pool exhaustion

🟡 MEDIUM: Missing AsNoTracking on read-only query (Line 67)
   Problem: Unnecessary change tracking overhead
   Fix: Add .AsNoTracking()
   Impact: 30% faster queries, 40% less memory

🟢 LOW: DateTime.Now usage (Line 89)
   Problem: Timezone conversion overhead
   Fix: Use DateTime.UtcNow
   Impact: 2-3x faster
```

---

For full project analysis, use the **Performance Tuning Advisor** agent or `/full-audit` prompt.
