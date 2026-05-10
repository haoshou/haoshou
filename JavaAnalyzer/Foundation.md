# JavaAnalyzer — Foundation

Background knowledge loaded at the start of every workflow.

---

## Project Detection Checklist

Run this at the start of every workflow to establish context:

```bash
# 1. Build tool
ls pom.xml build.gradle build.gradle.kts 2>/dev/null

# 2. Multi-module?
find . -name "pom.xml" -not -path "*/target/*" | head -20
# or
find . -name "build.gradle*" -not -path "*/build/*" | head -20

# 3. Source root
find . -type d -name "java" -path "*/main/*" | head -5

# 4. Spring Boot?
rg "spring-boot-starter" pom.xml build.gradle 2>/dev/null | head -5

# 5. Java version
rg "<java.version>|sourceCompatibility|jvmTarget" pom.xml build.gradle* 2>/dev/null | head -5
```

Record: **build tool**, **is multi-module**, **Java version**, **framework** — these shape every finding below.

---

## Java Ecosystem Map

### Build Tools

| Tool | Key Files | Dependency command |
|------|-----------|-------------------|
| **Maven** | `pom.xml`, `.mvn/` | `mvn dependency:tree` |
| **Gradle (Groovy)** | `build.gradle`, `settings.gradle` | `./gradlew dependencies` |
| **Gradle (Kotlin DSL)** | `build.gradle.kts`, `settings.gradle.kts` | `./gradlew dependencies` |

### Common Frameworks

| Framework | Marker | Architecture style |
|-----------|--------|-------------------|
| **Spring Boot** | `@SpringBootApplication`, `spring-boot-starter` | Layered / Hexagonal |
| **Quarkus** | `quarkus-bom`, `@QuarkusMain` | Cloud-native, reactive |
| **Micronaut** | `micronaut-parent`, `@MicronautApplication` | Compile-time DI |
| **Plain Java** | No framework marker | Library / CLI / batch |
| **Jakarta EE** | `jakarta.ee-api`, `javax.*` imports | Enterprise, app server |

### Standard Spring Boot Package Layout

```
com.company.project
├── controller/       # @RestController, @Controller — HTTP layer
├── service/          # @Service — business logic
├── repository/       # @Repository, extends JpaRepository — data access
├── domain/ (or model/)  # @Entity, POJOs — data model
├── config/           # @Configuration, @Bean — wiring
├── exception/        # Custom exceptions, @ControllerAdvice
├── dto/              # Request/Response objects (no JPA annotations)
└── util/             # Stateless utility classes
```

Violations of this layering (e.g., `@Repository` injected directly in `@Controller`, or `@Entity` returned from REST endpoints) are architecture findings.

---

## Common Anti-Patterns to Detect

### Architecture
- **Circular dependencies**: Module A imports Module B imports Module A
- **Layer skipping**: Controller → Repository (bypasses Service)
- **God class**: Single class with >500 lines or >20 public methods
- **Anemic domain model**: `@Entity` classes with only getters/setters, all logic in Services
- **Feature envy**: Class that mostly uses fields/methods of another class

### Code Quality
- **Long method**: >30 lines (flag), >50 lines (smell), >100 lines (refactor now)
- **Too many parameters**: >4 parameters in a method
- **Magic numbers/strings**: Hardcoded values not in constants or `@Value`
- **Catch-and-swallow**: `catch (Exception e) {}` or `catch (Exception e) { log.error(...); }` with no rethrow
- **Mutable static state**: `static` fields that are not `final`
- **Checked exceptions wrapped**: wrapping everything in `RuntimeException` without cause
- **`@Transactional` on private methods**: has no effect with Spring proxy AOP

### Performance
- **N+1 query**: Loop calling repository method inside, without `JOIN FETCH` or batch loading
- **Blocking call in reactive context**: `Thread.sleep()`, `Future.get()`, `block()` in reactive chain
- **Missing `@Index` on foreign key or search column**: JPA entities without DB indexes
- **`Optional.get()` without `isPresent()`**: guaranteed NPE path
- **String concatenation in loop**: use `StringBuilder`

### Security (quick flags — detail in SecurityReview)
- `String.format()` / concatenation used to build SQL query
- `@RequestParam` passed to repository without validation
- Passwords or API keys in `application.properties` (not using `${ENV_VAR}`)
- `@CrossOrigin("*")` on controllers
- Missing `@Valid` on `@RequestBody`

---

## Key Files to Read First

| File | What it tells you |
|------|------------------|
| `pom.xml` / `build.gradle` | Dependencies, Java version, plugins, project name |
| `application.properties` / `application.yml` | DB config, active profiles, feature flags |
| Main class (`@SpringBootApplication`) | Entry point, component scan root |
| `README.md` | Intended purpose (if exists) |
| `src/main/resources/db/migration/` | Flyway/Liquibase migrations = DB schema history |
| `docker-compose.yml` | Infrastructure: DB, cache, message broker |

---

## Severity Assignment Guide

| Severity | Criteria | Examples |
|----------|---------|---------|
| 🔴 HIGH | Security risk, data loss potential, production outage risk | SQL injection, hardcoded secret, missing transaction, blocking call in async context |
| 🟡 MED | Correctness risk, performance degradation under load, design violation | N+1 query, layer skipping, God class, missing validation |
| 🟢 LOW | Technical debt, readability, maintainability | Long method, magic number, swallowed exception |
| ℹ️ INFO | Observation worth noting, no action required | Framework version, pattern choice, test coverage gap |
