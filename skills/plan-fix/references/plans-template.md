# PLANS.md Template

All planning skills (plan-inline, plan-backlog, plan-fix) write PLANS.md using this structure.

```markdown
# Implementation Plan

**Created:** YYYY-MM-DD
**Source:** [see Source Format below]
**Linear Issues:** [PROJ-123](https://linear.app/...), [PROJ-124](https://linear.app/...)
**Sentry Issues:** [Sentry issue URLs] (omit if not from Sentry)
**Branch:** [type]/[description]

## Context Gathered

### Codebase Analysis
- **Related files:** [files found through exploration]
- **Existing patterns:** [patterns to follow]
- **Test conventions:** [how tests are structured in this area]

### MCP Context (if applicable)
- **MCPs used:** [which MCPs were consulted]
- **Findings:** [relevant information discovered]

### Investigation (plan-fix only)

**Bug report:** [what was reported]
**Classification:** [Type] / [Severity] / [Affected Area]
**Root cause:** [what's actually wrong and why]
**Evidence:**
- `path/to/file:line` — [what's wrong here]
**Impact:** [what breaks, who is affected]

### Triage Results (plan-backlog only)

**Planned:** PROJ-123, PROJ-456
**Canceled:** PROJ-789 (reason), PROJ-012 (reason)

## Tasks

### Task 1: [Name]
**Linear Issue:** [PROJ-123](url)
**Files:**
- `path/to/file` (create | modify)
- `path/to/test-file` (create | modify)

**Steps:**
1. Write test in [file] for [scenario]
2. Run verifier (expect fail)
3. Implement [function/fix] in [file]
4. Run verifier (expect pass)

**Notes:**
- Follow pattern in `path/to/existing-example`
- **Migration note:** [if applicable]

### Task N: [Name]
(same structure)

## Post-Implementation Checklist
1. Run `bug-hunter` agent — Review changes for bugs
2. Run `verifier` agent — Verify all tests pass and zero warnings

---

## Plan Summary

**Objective:** [One sentence — what and why]
**Linear Issues:** [PROJ-123, PROJ-124, ...]
**Approach:** [2-3 sentences]
**Scope:** X tasks, Y files, Z tests
**Key Decisions:** [list, if any]
**Risks:** [list, if any]
```

## Source Format

| Skill | Source Value |
|-------|-------------|
| plan-inline | `Inline request: [summary of request]` |
| plan-backlog | `Backlog: PROJ-123, PROJ-456` |
| plan-fix | `Bug report: [summary of bug]` |

## Termination

After writing PLANS.md and creating/moving Linear issues to Todo, output:

```
Plan created in PLANS.md
Linear issues in Todo: PROJ-123, PROJ-124, ...

## Plan Summary
**Objective:** [from PLANS.md]
**Linear Issues:** [from PLANS.md]
**Approach:** [from PLANS.md]
**Scope:** X tasks, Y files, Z tests
**Key Decisions:** [from PLANS.md, if any]
**Risks:** [from PLANS.md, if any]
```

Then create branch, commit (no `Co-Authored-By` tags), and push:
```bash
git checkout -b <type>/<description> && git add PLANS.md && git commit -m "plan: <description>" && git push -u origin <type>/<description>
```

Branch type: `feat/` for features, `fix/` for bugs, `refactor/` for refactoring.

Do not ask follow-up questions. Do not offer to implement. Output the summary and stop.
