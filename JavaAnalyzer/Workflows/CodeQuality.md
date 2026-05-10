# CodeQuality Workflow — JavaAnalyzer

## Purpose

Surface code smells, complexity problems, and maintainability issues — prioritized by actual impact, not style preference.

## Invocation

- "code quality"
- "code review"
- "code smells"
- "refactor candidates"
- "what needs to be improved"
- "technical debt"

## Procedure

### Step 1 — Long Methods

```bash
# Find Java files and count method length heuristically
# Flag methods >50 lines (between two method signatures)
find src/main/java -name "*.java" -not -path "*/test/*" | \
  xargs awk '
    /^\s+(public|private|protected|static).*\(.*\)/ { method=$0; start=NR }
    NR>start && /^\s+(public|private|protected|static).*\(.*\)/ {
      if (NR-start > 50) print FILENAME ":" start " — " NR-start " lines: " method
      method=$0; start=NR
    }
  ' 2>/dev/null | head -20
```

Or simpler approximation:
```bash
# Top largest files as proxy (read top 5 to inspect methods)
find src/main/java -name "*.java" -not -path "*/test/*" \
  | xargs wc -l | sort -rn | head -15
```

Read the largest files' method signatures to identify long methods manually.

### Step 2 — Too Many Parameters

```bash
# Methods with 5+ parameters
rg "^\s+(public|private|protected).*\(" src/main/java/ \
  | rg "(\w+\s+\w+,){4,}" | head -20
```

### Step 3 — Exception Handling Quality

```bash
# Swallowed exceptions (catch with no rethrow)
rg -A3 "catch\s*\(.*Exception" src/main/java/ \
  | rg -B1 "^\s*\}" | grep -v "throw\|log\|return" | head -20

# Catching raw Exception (too broad)
rg "catch\s*\(\s*Exception\s+\w+\s*\)" src/main/java/ | head -15

# printStackTrace (should use logger)
rg "\.printStackTrace\(\)" src/main/java/ | head -10
```

### Step 4 — Null Safety Issues

```bash
# Optional.get() without isPresent() check
rg -B5 "\.get\(\)" src/main/java/ \
  | grep -v "isPresent\|isEmpty\|ifPresent\|map\|orElse\|Optional" | head -15

# Returning null from methods that could return Optional
rg "return null;" src/main/java/ | head -20
```

### Step 5 — Magic Numbers and Strings

```bash
# Hardcoded numbers (not 0, 1, -1)
rg "= [2-9][0-9]+|= 1[0-9]+" src/main/java/ \
  | grep -v "final\|//\|test\|Test" | head -20

# Hardcoded string literals used in logic (not i18n/log)
rg '"[a-zA-Z]{4,}"' src/main/java/ \
  | grep -v "//\|log\.\|message\|description\|Test\|test" | head -20
```

### Step 6 — Static Mutable State

```bash
# Non-final static fields (shared mutable state = thread safety risk)
rg "^\s+private static (?!final)" src/main/java/ | head -15
rg "^\s+public static (?!final)" src/main/java/ | head -10
```

### Step 7 — Spring-Specific Pitfalls

```bash
# @Transactional on private methods (no-op with proxy AOP)
rg -B1 "private.*\(.*\)" src/main/java/ \
  | rg "@Transactional" | head -10

# Field injection (@Autowired on field — prefer constructor injection)
rg "@Autowired" src/main/java/ \
  | grep -v "//\|constructor\|setter" | head -20

# Missing @Valid on @RequestBody
rg "@PostMapping|@PutMapping|@PatchMapping" -A5 src/main/java/ \
  | rg "@RequestBody" | grep -v "@Valid" | head -10
```

### Step 8 — Code Duplication (manual signal)

```bash
# Files with same or similar names (potential duplication)
find src/main/java -name "*.java" | xargs basename | sort | uniq -d | head -10

# Large similar utility patterns
rg "private.*convert|private.*map|private.*transform" src/main/java/ | head -20
```

### Step 9 — Test Quality Signal

```bash
# Test files
find src/test -name "*.java" 2>/dev/null | wc -l

# Tests with no assertions (possible empty tests)
rg -l "void test" src/test/ 2>/dev/null | \
  xargs rg -L "assert\|Assert\|verify\|Assertions" 2>/dev/null | head -10

# @Disabled tests accumulation
rg "@Disabled|@Ignore" src/test/ 2>/dev/null | head -10
```

---

## CodeQuality Report Template

```
## JavaAnalyzer — CodeQuality Report

### Summary
[Key findings in 3-5 sentences. Lead with the most impactful issues.]

### Findings

| Severity | Category | Finding | Location |
|----------|---------|---------|---------|
| 🔴 HIGH  | Exception Handling | Swallowed exception in X | `ServiceA.java:42` |
| 🟡 MED   | Long Method | processOrder() is 87 lines | `OrderService.java:120` |
| 🟡 MED   | Spring Pitfall | @Transactional on private method | `PaymentService.java:55` |
| 🟢 LOW   | Magic Number | Hardcoded 3600 (seconds?) | `CacheConfig.java:18` |

### Top Refactor Candidates
1. **[ClassName]** — [reason, N lines, N issues]
2. **[ClassName]** — [reason]

### Spring Pitfalls
[List Spring-specific issues found]

### Test Coverage Signal
- Test files: [N]
- Disabled tests: [N]
- Tests without assertions: [N]

### Recommendations
1. [Highest value fix — what, where, why]
2. [...]
```
