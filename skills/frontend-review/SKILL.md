---
name: frontend-review
description: Reviews all frontend elements (UI, UX, accessibility, visual design, responsiveness, performance, visual QA from screenshots) using an agent team with 4 domain-specialized reviewers. Creates Linear issues in Backlog state for findings. Use when user says "review frontend", "check UI", "review UX", "audit accessibility", "check responsive", or "review screens". Falls back to single-agent mode if agent teams unavailable.
argument-hint: [optional: specific area like "settings page" or "photo capture"]
allowed-tools: Read, Glob, Grep, Task, Bash, TeamCreate, TeamDelete, SendMessage, TaskCreate, TaskUpdate, TaskList, TaskGet, mcp__linear__list_teams, mcp__linear__list_issues, mcp__linear__get_issue, mcp__linear__create_issue, mcp__linear__update_issue, mcp__linear__list_issue_labels, mcp__linear__list_issue_statuses
disable-model-invocation: true
---

Review all frontend elements using an agent team with domain-specialized reviewers. You are the **team lead/coordinator**. You orchestrate 4 reviewer teammates who examine the frontend through different lenses in parallel, then you merge findings, create Linear issues, and output a summary report.

**If agent teams are unavailable** (TeamCreate fails), fall back to single-agent mode — see "Fallback: Single-Agent Mode" section.

**Reference:** See [references/frontend-checklist.md](references/frontend-checklist.md) for the comprehensive checklist and [references/reviewer-prompts.md](references/reviewer-prompts.md) for domain-specific reviewer instructions.

## Pre-flight

1. **Verify Linear MCP** — Call `mcp__linear__list_teams`. If unavailable, STOP and tell the user: "Linear MCP is not connected. Run `/mcp` to reconnect, then re-run this skill."
2. **Discover project info from CLAUDE.md** — Read the LINEAR INTEGRATION section to find:
   - Team name (e.g., "ProjectName")
   - Issue prefix (e.g., PROJ-xxx)
   - If LINEAR INTEGRATION section not found, call `mcp__linear__list_teams` to discover the team
3. **Read CLAUDE.md** — Load project standards, tech stack, and conventions
4. **Discover frontend file patterns from CLAUDE.md** — Look for the STRUCTURE section to find frontend file patterns. Common patterns include:
   - `src/app/**/page.tsx` — Pages
   - `src/app/**/layout.tsx` — Layouts
   - `src/components/**/*.tsx` — Components
   - `src/hooks/**/*.ts` — Custom hooks
   - `src/app/globals.css` — Styles
   - `public/manifest.json` — PWA manifest
   - `middleware.ts` — Middleware
   - If STRUCTURE section not found, use common Next.js/React conventions
5. **Generate fresh screenshots** — Run E2E tests to produce up-to-date screenshots:
   ```
   Use Task tool with subagent_type "verifier" with prompt "e2e"
   ```
   - If E2E tests pass: screenshots are now available at `e2e/screenshots/*.png`
   - If E2E tests fail: warn the user, skip the visual-qa-reviewer (spawn only 3 code reviewers), note in the report that visual QA was skipped
6. **Discover frontend files** — Use Glob to find all frontend-related files using the patterns discovered in step 4
7. **Discover screenshots** — Use Glob to find `e2e/screenshots/*.png`. Build the screenshot list for the visual-qa-reviewer.
8. **Determine review scope:**
   - If `$ARGUMENTS` specifies an area → scope files and screenshots to that area only
   - If no arguments → review all frontend files and all screenshots
9. **Build the file list** — Create the exact list of files each code reviewer will examine, and the screenshot list for the visual-qa-reviewer

## Team Setup

### Create the team

Use `TeamCreate`:
- `team_name`: "frontend-review"
- `description`: "Parallel frontend review with domain-specialized reviewers"

**If TeamCreate fails**, switch to Fallback: Single-Agent Mode (see below).

### Create tasks

Use `TaskCreate` to create 4 review tasks (or 3 if screenshots unavailable):

1. **"Accessibility & semantics review"** — WCAG compliance, ARIA, semantic HTML, keyboard nav, screen readers
2. **"Visual design & UX review"** — Design consistency, layout, responsive design, mobile UX, user flows (code-level)
3. **"Performance & optimization review"** — Core Web Vitals, images, bundle, Server/Client components, PWA
4. **"Visual QA review"** — Screenshot-based analysis of rendered screens (skip if no screenshots)

### Spawn reviewer teammates

Use the `Task` tool with `team_name: "frontend-review"`, `subagent_type: "general-purpose"`, and `model: "sonnet"` to spawn each reviewer. Spawn all reviewers in parallel (concurrent Task calls in one message).

Each reviewer prompt MUST include:
- The common preamble and their domain checklist from [references/reviewer-prompts.md](references/reviewer-prompts.md)
- Code reviewers (1-3): The **exact list of files** to review
- Visual QA reviewer (4): The **exact list of screenshot paths** to read with the Read tool
- Instructions to report findings as a structured message to the lead

### Assign tasks

After spawning, use `TaskUpdate` to assign each task to its reviewer by name.

## Coordination

While waiting for reviewer messages:
1. Reviewer messages are **automatically delivered** — do NOT poll or manually check inbox
2. Teammates go idle after each turn — this is normal, not an error. They're done when they send their findings message.
3. Track progress via `TaskList`
4. Acknowledge receipt as each reviewer reports
5. Wait until ALL reviewers have reported before proceeding to merge

**If a reviewer gets stuck or stops without reporting:** Send them a message asking for their findings. If they don't respond, note that domain as "incomplete".

## Merge & Evaluate Findings

Once all reviewer findings are collected:

### Deduplicate
- Same component/element reported by multiple reviewers → merge into the one with higher priority
- Same root cause across multiple locations → combine into one finding
- Visual QA findings that overlap with code-level visual-design findings → merge, keeping the screenshot evidence reference

### Evaluate Severity

| Severity | Criteria | Examples |
|----------|----------|---------|
| **CRITICAL** | Blocks usage for some users entirely | Missing keyboard navigation on core flow, zero-contrast text, broken responsive layout on mobile |
| **HIGH** | Significant UX degradation or accessibility barrier | Missing ARIA labels on interactive elements, no focus indicators, touch targets below 44px, missing loading states, LCP > 4s, visually broken layout |
| **MEDIUM** | Noticeable but not blocking | Inconsistent spacing, minor contrast issues on non-critical text, missing skip links, CLS > 0.1, visual misalignment |
| **LOW** | Polish and best-practice improvements | Inconsistent border radius, missing hover transitions, suboptimal image format, minor visual rhythm issues |

## Create Linear Issues

After merging and deduplicating, create a Linear issue for each finding using `mcp__linear__create_issue`:

```
team: [discovered team name]
state: "Backlog"
title: "[Brief description of the issue]"
description: (see Issue Description Format below)
priority: [1|2|3|4] (mapped from severity)
labels: [Mapped label(s)]
```

**Issue Description Format:**

```
**Problem:**
[Clear, specific problem statement — 1-2 sentences]

**Context:**
[Affected file paths with line numbers, e.g. `src/components/food-analyzer.tsx:45-60`]
[For visual QA findings: reference which screenshot(s) show the issue, e.g. "Visible in e2e/screenshots/dashboard.png"]

**Impact:**
[Who is affected and how — e.g. screen reader users, mobile users, slow connections]

**Fix:**
[Specific remediation steps — what needs to change]

**Acceptance Criteria:**
- [ ] [Specific, verifiable criterion — e.g. "All interactive elements have visible focus indicators"]
- [ ] [Another criterion]
```

**Severity → Priority Mapping:**
- CRITICAL → 1 (Urgent)
- HIGH → 2 (High)
- MEDIUM → 3 (Medium)
- LOW → 4 (Low)

**Label Mapping:**

| Domain | Linear Label |
|--------|-------------|
| Accessibility issues, semantic HTML, ARIA | Bug |
| Visual design, UX, responsive layout | Improvement |
| Visual QA (layout, composition, consistency) | Improvement |
| Performance, Core Web Vitals, bundle | Performance |
| Convention (CLAUDE.md compliance) | Convention |

**Rules:**
- Include file paths with line numbers in Context
- For visual QA findings, reference the screenshot filename(s) showing the issue
- Acceptance criteria define "done" — verifiable conditions
- One issue per distinct finding

## Shutdown Team

After all Linear issues are created:
1. Send shutdown requests to all reviewers using `SendMessage` with `type: "shutdown_request"`
2. Wait for shutdown confirmations
3. Use `TeamDelete` to remove team resources

## Fallback: Single-Agent Mode

If `TeamCreate` fails, perform the review as a single agent:

1. **Inform user:** "Agent teams unavailable. Running frontend review in single-agent mode."
2. Read each frontend file in the review scope
3. Apply all domain checks sequentially using [references/frontend-checklist.md](references/frontend-checklist.md):
   a. Accessibility & semantics checks
   b. Visual design & UX checks (code-level)
   c. Performance & optimization checks
   d. Visual QA checks — read each screenshot and apply the Visual QA checklist
4. Merge, deduplicate, and create Linear issues — same process as team mode

## Error Handling

| Situation | Action |
|-----------|--------|
| Linear MCP not connected | STOP — tell user to run `/mcp` |
| No frontend files found | Stop — "No frontend files found in scope." |
| CLAUDE.md doesn't exist | Use general best practices |
| TeamCreate fails | Switch to single-agent fallback mode |
| E2E tests fail | Warn user, skip visual-qa-reviewer, proceed with 3 code reviewers |
| No screenshots found | Skip visual-qa-reviewer, proceed with 3 code reviewers |
| Reviewer stops without reporting | Send follow-up message, note domain as incomplete |
| Focus area doesn't match any files | Stop — "No files match the specified area." |

## Rules

- **Analysis only** — Do NOT modify any source code
- **Be specific** — Include file paths and line numbers for every finding
- **Include remediation** — Every finding must have a concrete fix suggestion
- **Prioritize impact** — Focus on issues that affect real users
- **Test don't assume** — Read the actual code, don't guess about implementations
- **Lead handles all Linear writes** — Reviewers NEVER create issues directly
- **Deduplicate before creating** — No duplicate issues in Linear
- **Screenshots are transient** — They are gitignored and regenerated each run. Reference them by filename in issue descriptions for context, but don't depend on them persisting.

## Termination

Output this report and STOP:

```
## Frontend Review Report

**Team:** 4 reviewers (accessibility, visual-design, performance, visual-qa)
[OR: **Team:** 3 reviewers (accessibility, visual-design, performance) — visual QA skipped (no screenshots)]
[OR: **Mode:** single-agent (team unavailable)]
**Scope:** [all frontend files | specific area]
**Files reviewed:** N
**Screenshots analyzed:** M [or "0 (skipped)"]

### Issues (ordered by priority)

| # | ID | Priority | Label | Title |
|---|-----|----------|-------|-------|
| 1 | PROJ-N1 | High | Bug | Brief title |
| 2 | PROJ-N2 | Medium | Improvement | Brief title |
| ... | ... | ... | ... | ... |

X issues total | Duplicates merged: M

Next step: Review Backlog in Linear and use `plan-backlog` to create implementation plans.
```

Do not ask follow-up questions. Do not offer to fix issues.
