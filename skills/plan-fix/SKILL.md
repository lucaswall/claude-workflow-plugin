---
name: plan-fix
description: Investigates bugs AND creates actionable TDD fix plans. Creates Linear issues in Todo state. Use when you know you want to fix something - user reports errors, deployment failures, wrong data, or UI issues. Can be chained from investigate skill. Discovers MCPs from CLAUDE.md for debugging (logs, etc.).
argument-hint: <bug description>
allowed-tools: Read, Edit, Write, Glob, Grep, Task, Bash, mcp__linear__list_teams, mcp__linear__list_issues, mcp__linear__get_issue, mcp__linear__create_issue, mcp__linear__update_issue, mcp__linear__list_issue_labels, mcp__linear__list_issue_statuses
disable-model-invocation: true
---

Investigate bugs and create TDD fix plans in PLANS.md. Creates Linear issues in Todo state.

## 1. Git Pre-flight Check

Before starting any investigation, verify git status:

```bash
git branch --show-current
git status --porcelain
```

- **STOP if NOT on `main` branch.** Tell the user: "Not on main branch. Please switch to main before planning: `git checkout main`"
- **STOP if there are uncommitted changes.** Tell the user to commit or stash first.
- **Check if behind remote:** `git fetch origin && git status -uno` — STOP if behind.

## 2. PLANS.md Pre-flight

Check if `PLANS.md` already exists at the project root:

- If it does not exist: OK, you will create it when documenting findings.
- If it exists with `Status: COMPLETE`: OK, overwrite with new fix plan.
- If it exists with active (non-COMPLETE) content: **STOP.** Tell the user there is an active plan that must be completed or removed first.
- In all cases, check for an existing section about this bug to avoid duplicates.

## 3. Verify Linear MCP

Call `mcp__linear__list_teams`. If unavailable, **STOP** and tell the user: "Linear MCP is not connected. Run `/mcp` to reconnect, then re-run this skill."

## 4. Read Project Context

Read `CLAUDE.md` at the project root (if it exists) to understand:
- Project structure and conventions
- Available MCPs (Linear, Railway, etc.)
- Tech stack details
- Testing conventions
- Any project-specific debugging notes

**Discover team name:** Look for LINEAR INTEGRATION section in CLAUDE.md. If not found, use `mcp__linear__list_teams` to discover the team name dynamically. Store the discovered team name for use throughout the skill.

## 5. Classify Bug Type

Categorize the reported issue into one of these types:

| Category | Description | Key Investigation Areas |
|----------|-------------|------------------------|
| **API Error** | Backend route failures, 500s, bad responses | Route handlers, middleware, external API calls, error handling |
| **Auth Issue** | Login failures, session problems, unauthorized access | Auth configuration, session management, middleware, token handling |
| **Deployment Failure** | Build errors, runtime crashes on Railway | Build logs, environment variables, dependency issues |
| **Frontend Bug** | UI rendering issues, broken interactions, wrong data display | React components, state management, data fetching, hydration |
| **Data Issue** | Wrong data, missing data, data corruption | Database queries, API transformations, caching, race conditions |
| **Performance** | Slow responses, timeouts, memory issues | Query performance, bundle size, API response times |
| **Integration** | Third-party service failures (AI APIs, payment, etc.) | API keys, request/response formats, rate limits, error handling |

## 6. Gather Evidence

### 6.1 Codebase Investigation

Search the codebase for relevant code:

```bash
# Find related files
find src -name "*.ts" -o -name "*.tsx" | head -50

# Search for relevant patterns
grep -rn "relevant_pattern" src/
```

Use Glob and Grep tools to:
- Find the files involved in the bug
- Trace the code path from entry point to the error
- Look for recent changes that might have introduced the bug
- Check test files for related test coverage

### 6.2 Deployment Logs (if MCP available)

If CLAUDE.md lists deployment MCPs (e.g., Railway MCP) and the bug involves deployment or runtime errors, use the MCP to check logs:

- Check recent deployment status
- Look for error logs around the time of the reported issue
- Check environment variable configuration (without exposing values)
- Review build logs for warnings or errors

### 6.3 Linear Context

Search Linear for related issues:

- Use `mcp__linear__list_issues` to find existing issues about this bug
- Check if there are related issues that provide context
- Look for previously attempted fixes

### 6.4 Reproduce the Issue

When possible, try to reproduce:

```bash
# Check if tests exist and if they catch the issue
npm test 2>&1 | tail -50

# Check for TypeScript errors
npx tsc --noEmit 2>&1 | tail -50

# Check for lint errors
npm run lint 2>&1 | tail -50
```

## 7. Document Findings in PLANS.md

Write or append to `PLANS.md` at the project root with this structure:

```markdown
# Fix Plan: [Brief Bug Title]

**Issue:** PROJ-xxx (if Linear issue exists, otherwise "To be created")
**Date:** YYYY-MM-DD
**Status:** Planning
**Branch:** fix/PROJ-xxx-brief-description (proposed)

## Investigation

### Bug Report
[What was reported - user's description of the problem]

### Classification
- **Type:** [API Error | Auth Issue | Deployment Failure | Frontend Bug | Data Issue | Performance | Integration]
- **Severity:** [Critical | High | Medium | Low]
- **Affected Area:** [specific component/route/feature]

### Root Cause Analysis
[What you found during investigation]

#### Evidence
- **File:** `path/to/file.ts:lineNumber` - [what's wrong here]
- **File:** `path/to/another-file.ts:lineNumber` - [related issue]
- **Logs:** [relevant log output if any]

#### Related Code
- `path/to/file.ts:lineNumber` — [describe what this code does and why it's problematic]
- `path/to/other-file.ts:lineNumber` — [describe the related code]
(Reference files and line numbers. Do NOT paste code blocks — the implementer will read the files.)

### Impact
- [What breaks because of this bug]
- [Who is affected]
- [Any data implications]

## Fix Plan (TDD Approach)

### Step 1: [Short description of change]
**File:** `path/to/file.ts` (create | modify)
**Test:** `path/to/__tests__/file.test.ts` (create | modify)
**Pattern:** Follow `path/to/similar-existing-file.ts` structure

**Behavior:**
- [What this component/function should do — written as a behavioral spec]
- [State transitions, edge cases, error handling]
- [Reference existing patterns by file path]

**Tests:**
1. [Test assertion in plain English]
2. [Test assertion in plain English]
3. [Test assertion in plain English]

### Step 2: [Next change]
(Same structure — behavioral spec, not code)

### Step N: Verify
- [ ] All new tests pass
- [ ] All existing tests pass
- [ ] TypeScript compiles without errors
- [ ] Lint passes
- [ ] Build succeeds

## Notes
- [Any additional context, workarounds, or considerations]
```

## 8. Create Linear Issue

Create a Linear issue in the discovered team with status "Todo":

1. First, get the team statuses to find the "Todo" state ID:
   ```
   mcp__linear__list_issue_statuses for team [discovered team name]
   ```

2. Get available labels:
   ```
   mcp__linear__list_issue_labels for team [discovered team name]
   ```

3. Create the issue:
   ```
   mcp__linear__create_issue with:
   - team: [Discovered team name]
   - title: "[Bug Type] Brief description of the fix needed"
   - description: |
     ## Bug Report
     [Summary of the issue]

     ## Root Cause
     [What was found during investigation]

     ## Fix Plan
     See PLANS.md for detailed TDD fix plan.

     ## Files Affected
     - `path/to/file.ts`
     - `path/to/another-file.ts`

     ## Acceptance Criteria
     - [ ] Failing test written and passes after fix
     - [ ] All existing tests pass
     - [ ] No TypeScript errors
     - [ ] Deployed successfully
   - status: "Todo"
   - Apply relevant labels (bug, etc.)
   ```

4. Update PLANS.md with the created issue key (PROJ-xxx).

## 9. Error Handling

| Situation | Action |
|-----------|--------|
| Cannot reproduce the bug | Document what was tried, create issue with "needs-reproduction" label |
| Root cause unclear | Document hypotheses ranked by likelihood, create issue with investigation notes |
| Multiple bugs found | Create separate PLANS.md sections and Linear issues for each |
| Bug is in a dependency | Document the dependency issue, check for updates/workarounds, note in issue |
| Deployment MCP unavailable | Skip deployment log analysis, note in findings |
| Linear MCP unavailable | Document the issue details in PLANS.md only, tell user to create manually |
| CLAUDE.md not found | Proceed with standard Next.js conventions |
| Existing fix in progress | Check the existing Linear issue and PLANS.md entry, update rather than duplicate |
| Bug is actually a feature request | Reclassify and suggest using add-to-backlog skill instead |

## 10. Rules

- **NEVER modify application code.** This skill only investigates and plans.
- **NEVER run destructive commands** (no `rm`, no `git reset --hard`, no database mutations).
- **ALWAYS use TDD approach** in fix plans - tests first, then implementation.
- **ALWAYS check for existing Linear issues** before creating new ones to avoid duplicates.
- **ALWAYS include file paths and line numbers** in evidence and fix plans.
- **ALWAYS propose a branch name** following the pattern `fix/PROJ-xxx-brief-description`.
- **Keep fix plans actionable** - another developer (or AI agent) should be able to follow the plan without additional context.
- **Severity guidelines:**
  - **Critical:** Production down, data loss, security vulnerability
  - **High:** Feature broken for all users, significant data issues
  - **Medium:** Feature partially broken, workaround exists
  - **Low:** Minor UI issue, edge case, cosmetic problem
- **DO NOT expose secrets, API keys, or sensitive environment variable values** in PLANS.md or Linear issues.
- **DO NOT hallucinate code** - only reference code that actually exists in the codebase.
- **Plans describe WHAT and WHY, not HOW at the code level.** Include: file paths, function names, behavioral specs, test assertions, patterns to follow (reference existing files by path), state transitions. Do NOT include: implementation code blocks, ready-to-paste TypeScript/TSX, full function bodies. The implementer (plan-implement workers) writes all code — your job is architecture and specification. Exception: short one-liners for surgical changes (e.g., "add `if (!session.x)` check after the existing `!session.y` check") are fine.
- **Flag migration-relevant fixes** — If the fix changes DB schema, renames columns, changes identity models, renames env vars, or changes session/token formats, add a note in the fix plan: "**Migration note:** [what production data is affected]". The implementer will log this in `MIGRATIONS.md`.

## 11. Scope Boundaries

This skill is specifically for:
- Investigating reported bugs and errors
- Creating structured fix plans with TDD approach
- Creating Linear issues for tracking

This skill is NOT for:
- Actually implementing fixes (use plan-implement for that)
- Adding new features (use plan-backlog or add-to-backlog)
- Code reviews (use code-audit)
- General investigation without a fix intent (use investigate)
- Refactoring (create a separate task)

## 12. Termination and Git Workflow

When investigation and planning are complete:

1. **Summarize findings** to the user:
   - Bug classification and severity
   - Root cause (confirmed or hypothesized)
   - Files affected
   - Linear issue created (PROJ-xxx)

2. **Create branch, commit (no `Co-Authored-By` tags), and push:**
   ```bash
   git checkout -b fix/PROJ-xxx-brief-description && git add PLANS.md && git commit -m "plan(PROJ-xxx): add fix plan for brief description" && git push -u origin fix/PROJ-xxx-brief-description
   ```

3. **Suggest next steps:**
   - "Run `/plan-implement` to implement the fix plan"
   - If critical: "This is a critical issue - recommend implementing immediately"

4. **If chained from investigate skill:**
   - Reference the investigation findings
   - Note any additional evidence found during the fix planning phase
