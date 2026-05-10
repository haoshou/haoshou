# ArchitectureMap Workflow — JavaAnalyzer

## Purpose

Map the actual architecture of the project — package structure, module boundaries, layer dependencies — and surface violations of intended design.

## Invocation

- "map the architecture"
- "package structure"
- "how is this organized"
- "layering analysis"
- "dependency direction"
- "module boundaries"

## Procedure

### Step 1 — Package Tree

```bash
# Full package tree (exclude target/build/generated)
find src/main/java -type f -name "*.java" \
  | sed 's|src/main/java/||' \
  | sed 's|/[^/]*\.java$||' \
  | sort -u
```

Group packages into layers. Identify the architecture style:

| Pattern | Indicator |
|---------|-----------|
| **Standard Layered** | `controller/`, `service/`, `repository/`, `domain/` |
| **Hexagonal (Ports & Adapters)** | `application/`, `domain/`, `infrastructure/`, `adapter/` |
| **DDD** | Named aggregates: `order/`, `payment/`, `user/`, each with sub-packages |
| **Feature-based** | Feature folders: `login/`, `checkout/`, `reporting/` |
| **Mixed** | Combination — note explicitly |

### Step 2 — Module Map (multi-module projects)

```bash
# Maven multi-module
grep -A2 "<modules>" pom.xml 2>/dev/null

# Gradle
cat settings.gradle 2>/dev/null || cat settings.gradle.kts 2>/dev/null
```

For each module, identify its role and what it depends on.

### Step 3 — Layer Violation Detection

**Controller → Repository (skips Service):**
```bash
rg "@Autowired|private.*Repository" src/main/java/ -l \
  | xargs rg -l "@RestController|@Controller" 2>/dev/null
```
For each file found: read it and confirm if Repository is injected in Controller.

**Entity returned directly from Controller (DTO leak):**
```bash
# Find controllers
rg -l "@RestController" src/main/java/ | while read f; do
  # Check if it returns @Entity types
  rg "@Entity" src/main/java/ -l | xargs basename -s .java | \
  while read entity; do
    rg "$entity" "$f" && echo "POSSIBLE DTO LEAK: $f returns $entity"
  done
done 2>/dev/null | head -20
```

**Circular imports between layers:**
```bash
# Service importing from Controller package
rg "import.*controller" src/main/java/**/*Service*.java 2>/dev/null | head -10

# Repository importing from Service
rg "import.*service" src/main/java/**/*Repository*.java 2>/dev/null | head -10
```

### Step 4 — God Class Detection

```bash
# Classes with many lines (potential God classes)
find src/main/java -name "*.java" -not -path "*/test/*" | while read f; do
  lines=$(wc -l < "$f")
  if [ "$lines" -gt 300 ]; then
    echo "$lines $f"
  fi
done | sort -rn | head -15
```

For top 5 large files: read them briefly to understand if size is justified (e.g., generated code, large config class).

### Step 5 — Interface / Abstraction Coverage

```bash
# Interfaces defined
find src/main/java -name "*.java" | xargs rg "^public interface " 2>/dev/null | wc -l

# Abstract classes
find src/main/java -name "*.java" | xargs rg "^public abstract class " 2>/dev/null | wc -l

# Services with interfaces (good practice)
rg -l "interface.*Service" src/main/java/ 2>/dev/null | wc -l
rg -l "implements.*Service" src/main/java/ 2>/dev/null | wc -l
```

### Step 6 — Cross-Cutting Concerns

```bash
# Aspect-Oriented Programming usage
rg -l "@Aspect|@Around|@Before|@After" src/main/java/ 2>/dev/null

# Exception handling
rg -l "@ControllerAdvice|@ExceptionHandler" src/main/java/ 2>/dev/null

# Logging pattern
rg "LoggerFactory.getLogger|@Slf4j|@Log4j2" src/main/java/ | head -5
```

---

## ArchitectureMap Report Template

```
## JavaAnalyzer — ArchitectureMap Report

### Architecture Style
**Pattern**: [Standard Layered / Hexagonal / DDD / Feature-based / Mixed]
**Modules**: [N] — [list]

### Package Map
[Annotated tree showing actual layout and intended layer]

### Layer Health
| Layer | Package | Files | Notes |
|-------|---------|-------|-------|

### Findings

#### Layer Violations
| Severity | Violation | Location |
|----------|-----------|---------|
| 🔴/🟡 | [description] | [file:line] |

#### God Classes (>300 lines)
| File | Lines | Verdict |
|------|-------|---------|

#### Abstraction Health
- Service interfaces: [N defined / N implemented]
- Cross-cutting aspects: [list @Aspect classes]
- Exception handling: [@ControllerAdvice present / missing]

### Recommendations
1. [Highest priority structural fix]
2. [...]
```
