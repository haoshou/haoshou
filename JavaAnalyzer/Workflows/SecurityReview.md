# SecurityReview Workflow — JavaAnalyzer

## Purpose

Identify security vulnerabilities aligned with OWASP Top 10 (2021) in Java application source code. Read-only analysis — no fixes, just findings with file:line references.

## Invocation

- "security review"
- "security audit"
- "find vulnerabilities"
- "OWASP check"
- "SQL injection"
- "hardcoded secrets"
- "auth review"

## OWASP Top 10 (2021) Coverage

| OWASP | Category | Covered |
|-------|---------|---------|
| A01 | Broken Access Control | ✅ Step 4 |
| A02 | Cryptographic Failures | ✅ Step 5 |
| A03 | Injection (SQL, LDAP, OS command) | ✅ Step 1 |
| A04 | Insecure Design | ✅ Step 4 |
| A05 | Security Misconfiguration | ✅ Step 6 |
| A06 | Vulnerable Components | → DependencyAudit workflow |
| A07 | Auth & Session Mgmt Failures | ✅ Step 4 |
| A08 | Software/Data Integrity Failures | ✅ Step 7 |
| A09 | Logging & Monitoring Failures | ✅ Step 8 |
| A10 | SSRF | ✅ Step 3 |

## Procedure

### Step 1 — Injection (A03)

**SQL Injection:**
```bash
# String concatenation in queries (highest risk)
rg "\"SELECT|\"INSERT|\"UPDATE|\"DELETE|\"FROM" src/main/java/ \
  | grep "+" | head -20

# createNativeQuery / createQuery with string concat
rg "createNativeQuery|createQuery|nativeQuery" src/main/java/ -A2 \
  | grep '".*\+' | head -15

# JdbcTemplate with string-built SQL
rg "jdbcTemplate\.(query|update|execute)" src/main/java/ -A2 \
  | grep '".*\+' | head -10
```

**LDAP Injection:**
```bash
rg "LdapTemplate|DirContext|searchFilter" src/main/java/ | head -10
# If found, check if input is sanitized before use
```

**OS Command Injection:**
```bash
rg "Runtime.exec|ProcessBuilder|new Process" src/main/java/ | head -10
# Flag if user input flows to exec()
```

**Log Injection:**
```bash
# Logging user-controlled input without sanitization
rg "log\.(info|warn|error|debug)\(.*request|log\.(info|warn|error|debug)\(.*param" \
  src/main/java/ | head -15
```

### Step 2 — Input Validation (A03 / A04)

```bash
# @RequestBody without @Valid
rg "@RequestBody" src/main/java/ -B2 | grep -v "@Valid" | head -20

# @RequestParam without validation
rg "@RequestParam" src/main/java/ | head -20
# For each: check if validated downstream

# Missing @Validated on service layer
rg "class.*Service" src/main/java/ -l | \
  xargs rg -L "@Validated" 2>/dev/null | head -10
```

### Step 3 — SSRF (A10)

```bash
# URL construction from user input
rg "new URL\(|URI.create\|RestTemplate|WebClient|HttpURLConnection" src/main/java/ \
  | head -15
# Check if URL is validated against allowlist before use

rg "getParameter\|@RequestParam.*url\|@RequestParam.*host\|@PathVariable.*url" \
  src/main/java/ | head -10
```

### Step 4 — Broken Access Control / Auth (A01, A07)

```bash
# Spring Security config
rg -l "WebSecurityConfigurerAdapter|SecurityFilterChain|@EnableWebSecurity" \
  src/main/java/ | head -5
```

Read the security config file. Check:
- Is `permitAll()` used too broadly?
- Is `csrf().disable()` present? (flag — needs justification)
- Are method-level security annotations used?

```bash
# Method-level security
rg "@PreAuthorize|@PostAuthorize|@Secured|@RolesAllowed" src/main/java/ | head -20

# Missing auth on sensitive endpoints
rg "@DeleteMapping|@PutMapping|@PostMapping" src/main/java/ -l | \
  xargs rg -L "@PreAuthorize|@Secured|@RolesAllowed" 2>/dev/null | head -15

# CORS wildcard
rg "@CrossOrigin" src/main/java/ | head -10
# Flag any @CrossOrigin("*") or @CrossOrigin without origins
```

### Step 5 — Cryptographic Failures (A02)

```bash
# Weak hashing algorithms
rg "MD5|SHA-1|SHA1|\"MD5\"\|\"SHA1\"" src/main/java/ | head -10

# Hardcoded secrets, keys, passwords
rg -i "password\s*=\s*\"|apikey\s*=\s*\"|secret\s*=\s*\"|token\s*=\s*\"" \
  src/main/java/ | grep -v "//\|test\|Test\|\$\{" | head -15

rg -i "private.*key.*=.*\"[A-Za-z0-9+/]{20,}\"" src/main/java/ | head -10

# Secrets in application.properties (not using env var)
rg "password=(?!\$\{)|secret=(?!\$\{)|apikey=(?!\$\{)" \
  src/main/resources/application* 2>/dev/null | grep -v "#" | head -10

# Random not using SecureRandom for security context
rg "new Random()" src/main/java/ | head -10
# Flag if used in auth/token/OTP context
```

### Step 6 — Security Misconfiguration (A05)

```bash
# Actuator endpoints exposed
rg "management.endpoints.web.exposure.include" src/main/resources/ 2>/dev/null

# Error details exposed to clients
rg "server.error.include-stacktrace\|server.error.include-message" \
  src/main/resources/ 2>/dev/null

# H2 console enabled in non-dev profile
rg "h2-console.enabled=true" src/main/resources/application.properties 2>/dev/null

# Debug mode
rg "debug=true\|logging.level.root=DEBUG" src/main/resources/application.properties \
  2>/dev/null | head -5
```

### Step 7 — Deserialization / Integrity (A08)

```bash
# Java native deserialization (high risk)
rg "ObjectInputStream|readObject\(\)" src/main/java/ | head -10

# Jackson deserialization without type restrictions
rg "enableDefaultTyping\|@JsonTypeInfo" src/main/java/ | head -10

# XML external entity (XXE)
rg "DocumentBuilder|SAXParser|XMLReader" src/main/java/ | head -10
# Flag if not configured to disable external entities
```

### Step 8 — Logging & Monitoring (A09)

```bash
# Auth events logged?
rg "login\|authentication\|logout\|forbidden\|unauthorized" \
  src/main/java/ | grep "log\." | head -10

# Sensitive data logged (PII, tokens)
rg "log\.(info|debug|warn)\(.*password\|log\.(info|debug|warn)\(.*token\|\
log\.(info|debug|warn)\(.*credit" src/main/java/ | head -10
```

---

## SecurityReview Report Template

```
## JavaAnalyzer — SecurityReview Report
**OWASP Coverage**: A01–A10 (excl. A06 → see DependencyAudit)

### Risk Summary
| OWASP | Category | Risk Level | Findings |
|-------|---------|-----------|---------|
| A01 | Broken Access Control | 🔴/🟡/🟢/✅ | [N issues] |
| A02 | Cryptographic Failures | ... | |
| A03 | Injection | ... | |
...

### Critical Findings (🔴 HIGH)
[Detailed findings with file:line, what the vulnerability is, what the exploit path is]

### Medium Findings (🟡)
[...]

### Low / Info
[...]

### Auth & Session Architecture
[Summary of Spring Security config — what's protected, what's open, CSRF status]

### Immediate Action Items
1. [Fix by end of sprint]
2. [...]
```
