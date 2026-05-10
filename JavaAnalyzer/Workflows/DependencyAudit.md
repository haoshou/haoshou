# DependencyAudit Workflow — JavaAnalyzer

## Purpose

Audit Maven/Gradle dependencies for version conflicts, unused or redundant deps, outdated versions, and known CVE exposure.

## Invocation

- "dependency audit"
- "check dependencies"
- "pom.xml review"
- "version conflicts"
- "vulnerable dependencies"
- "outdated libraries"

## Procedure

### Step 1 — Read the Build File

```bash
cat pom.xml 2>/dev/null || cat build.gradle 2>/dev/null || cat build.gradle.kts 2>/dev/null
```

Extract: **all declared dependencies with versions**, **managed dependencies (BOM)**, **repositories used**.

### Step 2 — Dependency Count and Category

Count total dependencies and group by category:

```bash
# Maven: count direct dependencies
rg "<dependency>" pom.xml | wc -l

# Gradle
rg "implementation|testImplementation|compileOnly|runtimeOnly" build.gradle* | wc -l
```

Categorize:
- **Core framework** (Spring Boot BOM, Quarkus BOM)
- **Data access** (JPA, JDBC drivers, connection pools)
- **Web/API** (Jackson, OpenAPI, validation)
- **Testing** (JUnit, Mockito, Testcontainers)
- **Utilities** (Lombok, MapStruct, Apache Commons, Guava)
- **Observability** (Micrometer, Logback, Log4j2)
- **Security** (Spring Security, JWT libraries)

### Step 3 — Dependency Tree (if build tool available)

```bash
# Maven
mvn dependency:tree -q 2>&1 | head -100

# Gradle
./gradlew dependencies --configuration compileClasspath 2>&1 | head -100
```

Flag: **version conflicts** (same artifact resolved at different versions), **transitive deps being overridden**.

### Step 4 — Outdated Version Detection

Check the following common libraries against their latest stable versions:

| Library | Check for |
|---------|----------|
| Spring Boot | Is it current minor? (3.2.x as of 2026) |
| Jackson | `2.15+` recommended |
| Lombok | `1.18.30+` |
| MapStruct | `1.5+` |
| Testcontainers | `1.19+` |
| Log4j / Log4j2 | Must be `2.17.1+` (CVE-2021-44228) |
| Spring Security | Must be current patch |
| Flyway / Liquibase | Check for known migration issues |

```bash
# Find Log4j (critical CVE check)
rg "log4j" pom.xml build.gradle* 2>/dev/null

# Find specific versions
rg "<version>|:\"" pom.xml build.gradle* 2>/dev/null | head -40
```

### Step 5 — Redundant / Duplicate Libraries

Common redundancies to flag:

```bash
# Multiple logging implementations (should be only one)
rg "log4j|logback|slf4j|log4j2|commons-logging" pom.xml build.gradle* 2>/dev/null

# Multiple JSON libraries
rg "jackson|gson|fastjson|json-simple" pom.xml build.gradle* 2>/dev/null

# Multiple HTTP clients
rg "httpclient|okhttp|feign|webclient|resttemplate" pom.xml build.gradle* 2>/dev/null

# Both JUnit 4 and JUnit 5
rg "junit" pom.xml build.gradle* 2>/dev/null
```

### Step 6 — Security-Sensitive Dependencies

Flag libraries with known CVE history — verify version is patched:

```bash
# High-priority CVE-bearing libraries
rg -i "log4j|jackson-databind|spring-security|netty|commons-text|snakeyaml|\
  h2database|xstream|fastjson|shiro|struts" pom.xml build.gradle* 2>/dev/null
```

For each found: note the version and flag if it's below the known-safe threshold.

### Step 7 — Dependency Scope Hygiene

```bash
# Maven: test dependencies without test scope
rg "mockito|junit|testcontainers|assertj|hamcrest" pom.xml \
  | grep -v "<scope>test</scope>" | head -10

# Provided/optional used correctly?
rg "provided|optional" pom.xml | head -10
```

---

## DependencyAudit Report Template

```
## JavaAnalyzer — DependencyAudit Report
**Build**: Maven/Gradle | **Direct deps**: [N] | **Managed via BOM**: [yes/no]

### Dependency Overview
| Category | Count | Notable |
|---------|-------|--------|
| Framework | N | Spring Boot 3.2.x |
| Data | N | JPA + HikariCP |
| Testing | N | JUnit 5 + Mockito |
| ... | | |

### Findings

| Severity | Issue | Detail |
|----------|-------|--------|
| 🔴 HIGH  | CVE exposure | log4j-core 2.14.0 — vulnerable to Log4Shell (CVE-2021-44228), update to 2.17.1+ |
| 🟡 MED  | Version conflict | jackson-databind resolved as both 2.13.x and 2.15.x |
| 🟡 MED  | Redundant logging | Both Logback and Log4j2 on classpath |
| 🟢 LOW  | Test dep in compile scope | mockito-core missing `<scope>test</scope>` |
| ℹ️ INFO | Outdated | Lombok 1.18.20, current is 1.18.30 |

### CVE Risk Summary
[List all security-relevant libraries and their CVE exposure by version]

### Recommendations
1. [Urgent: CVE fix]
2. [Resolve version conflict in X]
3. [...]
```
