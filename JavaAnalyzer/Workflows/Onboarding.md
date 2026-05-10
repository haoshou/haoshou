# Onboarding Workflow — JavaAnalyzer

## Purpose

Build a complete mental model of an unfamiliar Java project in under 10 minutes. Answer the three questions every developer needs on day 1:
1. **What does this project do?**
2. **How is it structured?**
3. **Where do I start reading?**

## Invocation

- "understand this project"
- "what does this codebase do"
- "explain this project to me"
- "onboard to this Java repo"
- "I'm new to this project"

## Procedure

### Step 1 — Project Identity (parallel reads)

```bash
# Project name, version, description
head -20 pom.xml 2>/dev/null || head -20 build.gradle 2>/dev/null

# README if it exists
find . -maxdepth 2 -iname "README*" | head -3
```

Read the README (first 100 lines). Read pom.xml `<description>` and `<name>`.

Extract: **project name**, **stated purpose**, **owner/team** (if visible).

### Step 2 — Tech Stack Detection

```bash
# Framework and major dependencies
rg "spring-boot|quarkus|micronaut|jakarta" pom.xml build.gradle* 2>/dev/null | head -10

# Java version
rg "java.version|sourceCompatibility|jvmTarget|release" pom.xml build.gradle* 2>/dev/null | head -5

# Database
rg "mysql|postgresql|h2|mongodb|oracle|redis|elasticsearch" pom.xml build.gradle* 2>/dev/null | head -10

# Message broker
rg "kafka|rabbitmq|activemq|sqs" pom.xml build.gradle* 2>/dev/null | head -5

# Build: multi-module?
find . -name "pom.xml" -not -path "*/target/*" | wc -l
```

Summarize: **Java version**, **framework**, **databases**, **messaging**, **multi-module yes/no**.

### Step 3 — Entry Points

```bash
# Spring Boot main class
rg -l "@SpringBootApplication" src/ 2>/dev/null

# Other main methods
rg -l "public static void main" src/main/ 2>/dev/null | head -10

# Scheduled jobs
rg -l "@Scheduled|@EnableScheduling" src/ 2>/dev/null | head -5

# Message listeners
rg -l "@KafkaListener|@RabbitListener|@SqsListener|@MessageMapping" src/ 2>/dev/null | head -5
```

Read the main class. Identify: **single entrypoint** or **multi-entrypoint** (scheduled + REST + listener)?

### Step 4 — Package Map

```bash
# Top-level packages under main java source
find src/main/java -mindepth 1 -maxdepth 4 -type d | sed 's|src/main/java/||' | head -40
```

Map to architecture pattern (see Foundation.md). Note any packages that don't fit the standard layout.

### Step 5 — API Surface (if web project)

```bash
# REST endpoints
rg "@GetMapping|@PostMapping|@PutMapping|@DeleteMapping|@PatchMapping|@RequestMapping" \
  src/main/java/ -l | head -20

# Count of endpoints
rg "@GetMapping|@PostMapping|@PutMapping|@DeleteMapping|@PatchMapping" \
  src/main/java/ | wc -l

# List controller files
rg -l "@RestController|@Controller" src/main/java/ | head -20
```

Read 1-2 controller files to understand the API domain.

### Step 6 — Data Model

```bash
# Entity classes
rg -l "@Entity" src/main/java/ | head -20

# Count entities
rg -l "@Entity" src/main/java/ | wc -l
```

List entity names (filenames). This reveals what data the project manages.

### Step 7 — Infrastructure Config

```bash
# application properties/yaml
find src/main/resources -name "application*" | head -10
```

Read `application.properties` or `application.yml` (skip secret values). Note: active profiles, DB URL pattern, any feature flags.

```bash
# Docker/compose
find . -maxdepth 2 -name "docker-compose*" | head -5
```

### Step 8 — Test Coverage Signal

```bash
# Test files count
find src/test -name "*Test*.java" -o -name "*Spec*.java" 2>/dev/null | wc -l

# Test types
rg -l "@SpringBootTest" src/test/ 2>/dev/null | wc -l   # integration tests
rg -l "@ExtendWith(MockitoExtension" src/test/ 2>/dev/null | wc -l  # unit tests
```

Report: **test file count** and **integration vs unit ratio** (low coverage is ℹ️ INFO, not a finding).

### Step 9 — Compile & Build Health (optional, only if mvn/gradle available)

```bash
# Maven
mvn compile -q 2>&1 | tail -5

# Gradle
./gradlew compileJava 2>&1 | tail -5
```

Note any compile errors — these are 🔴 HIGH if present.

---

## Onboarding Report Template

```
## JavaAnalyzer — Onboarding Report
**Project**: [name] | **Version**: [x.x.x]
**Build**: Maven/Gradle | **Java**: [version] | **Framework**: [Spring Boot x.x / Quarkus / etc.]

### What This Project Does
[2-3 sentences from README + code observation]

### Tech Stack
- **Web**: [Spring MVC / WebFlux / none]
- **Database**: [PostgreSQL via JPA / MongoDB / etc.]
- **Messaging**: [Kafka / none / etc.]
- **Other**: [Redis cache / S3 / etc.]

### Structure
**Type**: [Monolith / Multi-module / Microservice]
**Modules**: [list if multi-module]
**Package layout**: [Standard Spring layered / Hexagonal / DDD / Custom]

### Entry Points
- **Main**: `com.example.ProjectApplication`
- **REST API**: [N] endpoints across [N] controllers — domain: [what they cover]
- **Scheduled**: [list @Scheduled jobs if any]
- **Listeners**: [Kafka/RabbitMQ listeners if any]

### Data Model
[N] JPA entities: [Entity1, Entity2, Entity3, ...]

### Start Reading Here
1. `[MainClass.java]` — entry point
2. `[PrimaryController.java]` — core API
3. `[CoreEntity.java]` — central domain object
4. `application.yml` — configuration

### Signals
| Area | Signal |
|------|--------|
| Tests | [N] test files ([N] unit, [N] integration) |
| Build | [Clean / N compile errors] |
| Docs | [README present / missing] |

### Suggested Next Step
[ArchitectureMap to understand layering | CodeQuality if smells visible | DependencyAudit if many deps]
```
