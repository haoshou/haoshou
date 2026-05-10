---
name: JavaAnalyzer
description: "Structured analysis of Java project codebases across 5 workflows: Onboarding (rapid mental model of an unfamiliar project — entry points, purpose, tech stack, 10-min read), ArchitectureMap (package structure, module boundaries, layering health, dependency direction violations), CodeQuality (code smells, cyclomatic complexity, God classes, duplication, naming), DependencyAudit (Maven/Gradle dependency tree, version conflicts, unused deps, CVE exposure), SecurityReview (OWASP Top 10 aligned — SQL injection, XSS, auth gaps, secrets in code, input validation). Tools: rg/fd for fast search, Read for file inspection, Bash for mvn/gradle commands. Reads pom.xml or build.gradle first on every workflow. Output is structured report with findings, severity, and file:line references. USE WHEN: analyze java project, review java code, understand codebase, java architecture review, java code quality, java security audit, java dependencies, onboard to java project, understand this java repo. NOT FOR running tests or modifying code (analysis only)."
effort: high
context: fork
---

## Customization

**Before executing, check for user customizations at:**
`~/.claude/PAI/USER/SKILLCUSTOMIZATIONS/JavaAnalyzer/`

If this directory exists, load and apply any `PREFERENCES.md`, configurations, or resources found there (e.g., custom severity thresholds, company-specific patterns to check, excluded paths). If not found, proceed with skill defaults.

---

## MANDATORY: Voice Notification

**Send before any other action:**

```bash
curl -s -X POST http://localhost:31337/notify \
  -H "Content-Type: application/json" \
  -d '{"message": "Running the WORKFLOWNAME workflow in JavaAnalyzer to ACTION", "voice_id": "fTtv3eikoepIosk8dTZ5", "voice_enabled": true}' \
  > /dev/null 2>&1 &
```

**Text output:**
```
Running the **WorkflowName** workflow in the **JavaAnalyzer** skill to ACTION...
```

---

# JavaAnalyzer Skill

Structured analysis of Java codebases. The goal is to build an accurate mental model of a project and surface actionable findings — **read-only, never modifies code**.

Load `Foundation.md` before executing any workflow.

## Core Principle

> Understand before judging. Always establish what the project *intends* to do before evaluating how well it does it.

Every workflow starts with project detection (Maven vs Gradle, Spring Boot vs plain Java, monolith vs multi-module) because the same finding may be a problem or a deliberate design choice depending on context.

## Workflow Routing

| Workflow | Trigger | File |
|----------|---------|------|
| **Onboarding** | "understand this project", "what does this do", "explain this codebase", "new to this project", "onboard" | `Workflows/Onboarding.md` |
| **ArchitectureMap** | "architecture", "package structure", "module map", "layering", "how is this organized", "dependency direction" | `Workflows/ArchitectureMap.md` |
| **CodeQuality** | "code quality", "code smells", "code review", "refactor candidates", "complexity", "bad code", "improve" | `Workflows/CodeQuality.md` |
| **DependencyAudit** | "dependencies", "pom.xml", "build.gradle", "version conflicts", "unused deps", "CVE", "vulnerable" | `Workflows/DependencyAudit.md` |
| **SecurityReview** | "security", "security audit", "OWASP", "vulnerabilities", "SQL injection", "XSS", "secrets", "auth" | `Workflows/SecurityReview.md` |

**Ambiguous requests**: if the request doesn't clearly map to one workflow, run **Onboarding** first and then ask which specialized workflow to continue with.

**Chaining**: Onboarding → ArchitectureMap → (CodeQuality | DependencyAudit | SecurityReview) is the recommended full-analysis sequence.

## Output Format

Every workflow produces a structured report:

```
## JavaAnalyzer — [WorkflowName] Report
**Project**: [name] | **Build**: Maven/Gradle | **Framework**: Spring Boot x.x / Plain Java / Quarkus / etc.
**Analyzed**: [timestamp] | **Scope**: [path analyzed]

### Summary
[3-5 sentence overview of key findings]

### Findings
| Severity | Area | Finding | File:Line |
|----------|------|---------|-----------|
| 🔴 HIGH   | ...  | ...     | ...       |
| 🟡 MED    | ...  | ...     | ...       |
| 🟢 LOW    | ...  | ...     | ...       |

### Details
[Per-finding explanation with code context]

### Recommendations
[Prioritized action items]
```

Severity scale: 🔴 HIGH (fix now), 🟡 MED (fix soon), 🟢 LOW (technical debt), ℹ️ INFO (observation, no action needed).
