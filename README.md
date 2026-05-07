---
name: onenote-work-notes
description: |
  A skill for processing and optimizing OneNote work notes. Triggers when the user
  pastes or describes content from OneNote, including: work item organization
  (action items / task lists), logic clarification (untangling complex relationships),
  text optimization (polishing, structuring, removing redundancy), and execution
  status tracking (progress updates, status attribution). Designed for SRE /
  engineering contexts: incident records, Runbooks, on-call logs, RCA drafts,
  weekly reports, and meeting notes. Always trigger this skill for requests like
  "help me organize this", "polish this text", "clarify the logic", or
  "turn this into action items".
---

# OneNote Work Notes Processing Skill

## Core Capabilities

This skill covers four primary modes. The appropriate mode is auto-detected from user
input, or can be explicitly specified by the user.

---

## Mode 1: Logic Clarification

**Trigger phrases**: clarify, untangle, make sense of, walk me through, explain the logic

### Processing Steps

1. **Identify the core problem**: Summarize in one sentence what is being solved
2. **Extract entities and relationships**: Key actors, systems, process nodes
3. **Build the causal chain**: Use `Cause → Symptom → Impact → Response` structure
4. **Output structured logic** (Markdown format):

```
## Problem Summary
[One sentence]

## Key Entities
- System/Component A: [Responsibility]
- System/Component B: [Responsibility]

## Logic Flow
[A] --calls--> [B] --depends on--> [C]

## Core Conflicts
- Conflict 1: ...
- Conflict 2: ...

## Conclusions / Open Questions
- ✅ Confirmed: ...
- ❓ To be verified: ...
```

### SRE Extension

When handling incident or failure descriptions, additionally output:

```
## Incident Timeline
- HH:MM  [Event]
- HH:MM  [Event]

## Blast Radius
- Services: ...
- Users affected: ...
- SLA impact: ...

## Root Cause Hypotheses
1. [Hypothesis 1] - Confidence: High / Medium / Low
2. [Hypothesis 2] - Confidence: High / Medium / Low
```

---

## Mode 2: Text Optimization

**Trigger phrases**: polish, rewrite, clean up, improve, simplify, translate, English version

### Optimization Rules

| Issue Type | Action |
|------------|--------|
| Informal / fragmented writing | Normalize expression, preserve intent |
| Redundancy and repetition | Merge duplicates, remove filler |
| Unclear logic | Reorder sentences, add connectors |
| Mixed language (e.g. Chinglish) | Unify language style (default: follow user style) |
| Inconsistent voice (active/passive) | Align to a single subject perspective |

### Output Format

```
## Before
[Original text]

## After
[Revised version]

## Key Changes
- [Change description 1]
- [Change description 2]
```

### Engineering Document Rules

- **Titles**: Start with a verb ("Fix X", "Optimize Y", "Investigate Z")
- **Descriptions**: Include Who + What + Why
- **Status tags**: Standardize as `[TODO]` `[WIP]` `[DONE]` `[BLOCKED]`
- **Metrics**: Always include units (ms, %, req/s, MB)

---

## Mode 3: Work Item Extraction

**Trigger phrases**: action items, tasks, TODO, next steps, follow-ups, work items

### Extraction Rules

Identify from unstructured text:
- Action verb phrases (need to, should, plan to, will, must)
- Owners (@mention, names, role titles)
- Deadlines (today, this week, ASAP, by Friday)
- Dependencies (waiting on, blocked by, requires X first)

### Output Format

```markdown
## Action Items

| # | Task | Owner | Priority | Due Date | Dependency | Status |
|---|------|-------|----------|----------|------------|--------|
| 1 | [Task] | [Person] | P0/P1/P2 | [Date] | [Dep] | TODO |

## Items to Clarify
- [ ] [Unclear item that needs confirmation]
```

### Priority Criteria (SRE Context)

- **P0**: Impacts production service / SLA / security
- **P1**: Must complete this sprint / has a hard deadline
- **P2**: Important but not urgent
- **P3**: Nice to have / technical debt

---

## Mode 4: Status Update

**Trigger phrases**: progress, status, weekly report, update, sync, summary

### Output Format (suitable for manager sync or weekly report)

```markdown
## Weekly Progress

### ✅ Completed
- [Item 1] (Impact: [quantified metric])
- [Item 2]

### 🔄 In Progress
- [Item 1] (Progress: X%, ETA: [date])
- [Item 2] (Blocked by: [reason], Needs: [support])

### 📋 Next Week Plan
- [Item 1]
- [Item 2]

### ⚠️ Risks & Blockers
- [Risk description] (Mitigation: [action])
```

---

## Usage

### Quick Invocation

```
# Logic clarification
"Help me clarify the logic here: [paste OneNote content]"

# Text optimization
"Polish this description: [paste text]"

# Work item extraction
"Extract action items from these meeting notes: [paste content]"

# Status update
"Format this progress into a weekly report: [paste notes]"

# Combined mode
"Clarify the logic first, then extract action items: [paste content]"
```

### Ground Rules

1. **Preserve technical terms**: Kafka, JVM, Financle, HikariCP — never translate proper nouns
2. **Preserve numeric precision**: Do not round or simplify key metrics (latency, error rate, etc.)
3. **Never guess owners**: Use `[TBD]` when the responsible party is unclear
4. **Language follows input**: Default Chinese input → Chinese output, unless user requests English

---

## Example

### Input (raw OneNote note)

```
Talked to Xiao Wang today, the Financle interface timeout issue seems to be
because the DB connection pool is running out, need him to check the pool config.
Also need to look at GC — there were full GC alerts recently.
Oh, and the P0 ticket — RCA needs to be delivered by next Friday.
```

### Output (Work Items + Logic Clarification)

```markdown
## Problem Summary
Financle API timeouts, likely caused by DB connection pool exhaustion compounded
by Full GC pressure.

## Logic Flow
[Traffic spike] → [DB connection pool exhausted] → [API requests timeout]
                                              ↘ [Full GC triggered] → [STW latency spike]

## Action Items

| # | Task | Owner | Priority | Due Date |
|---|------|-------|----------|----------|
| 1 | Investigate DB connection pool config (max size, timeout settings) | Xiao Wang | P0 | ASAP |
| 2 | Analyze Full GC root cause (heap dump / GC log review) | Xiao Wang | P0 | ASAP |
| 3 | Deliver Financle API timeout RCA document | [TBD] | P0 | Next Friday |

## Items to Clarify
- [ ] Which connection pool? HikariCP / DBCP / Druid? Current max pool size?
- [ ] Full GC frequency? When did it last occur?
- [ ] Who owns the RCA document?
```

---

*Place this file under `.claude/skills/` in your project root, or reference it as a rule in `CLAUDE.md`.*
