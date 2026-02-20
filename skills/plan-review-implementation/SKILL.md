---
name: plan-review-implementation
description: QA review of completed implementation using an agent team with 3 domain-specialized reviewers (security, reliability, quality). Use after plan-implement finishes, or when user says "review the implementation". Moves Linear issues Review→Merge. Creates new issues in Todo for bugs found. Falls back to single-agent mode if agent teams unavailable.
allowed-tools: Read, Edit, Write, Glob, Grep, Bash, Task, TeamCreate, TeamDelete, SendMessage, TaskCreate, TaskUpdate, TaskList, TaskGet, mcp__linear__list_teams, mcp__linear__list_issues, mcp__linear__get_issue, mcp__linear__create_issue, mcp__linear__update_issue, mcp__linear__list_issue_labels, mcp__linear__list_issue_statuses
disable-model-invocation: true
---

Review **ALL** implementation iterations that need review using an agent team with domain-specialized reviewers. You are the **team lead/coordinator**. You orchestrate 3 reviewer teammates who review changed files in parallel through different lenses, then you merge findings, document them, and handle Linear/git.

**If agent teams are unavailable** (TeamCreate fails), fall back to single-agent mode — see "Fallback: Single-Agent Mode" section.

**Reference:** See [references/code-review-checklist.md](references/code-review-checklist.md) for comprehensive checklist.

## Pre-flight Check

1. **Read PLANS.md** — Understand the full plan and iteration history
2. **Read CLAUDE.md** — Understand project standards and conventions
3. **Verify Linear MCP** — Call `mcp__linear__list_teams`. If unavailable, STOP and tell the user: "Linear MCP is not connected. Run `/mcp` to reconnect, then re-run this skill."
4. **Assess AI-generated code risk** — If implementation is large or shows AI patterns, apply extra scrutiny

## Linear State Management

This skill moves issues from **Review → Merge** (preparing for PR).

**If task passes review (no issues):**
- Move issue from "Review" to "Merge" using `mcp__linear__update_issue`

**If task needs fixes (Fix Plan path):**
- Move original issue from "Review" to "Merge" (the original task was completed)
- Create NEW Linear issue(s) in "Todo" for each bug/fix using `mcp__linear__create_issue`:
  - `team`: [discovered from CLAUDE.md or list_teams]
  - `state`: "Todo" (will enter PLANS.md via Fix Plan)
  - `labels`: Bug
- Add new issue links to the Fix Plan section in PLANS.md
- These new issues will go through the full cycle when plan-implement runs the Fix Plan

**If task needs fixes (inline fix path — ≤3 S-size bugs):**
- Move original issue from "Review" to "Merge" (the original task was completed)
- Fix bugs inline with TDD (see Inline Fix Assessment)
- Create NEW Linear issue(s) in "Merge" state for traceability using `mcp__linear__create_issue`:
  - `team`: [discovered from CLAUDE.md or list_teams]
  - `state`: "Merge" (already fixed)
  - `labels`: Bug
- No Fix Plan needed — bugs resolved in the same session

## Identify What to Review

**Detection logic:**

1. Search PLANS.md for `## Iteration N` sections
2. **If iterations exist:** Build list of iterations needing review:
   - Has "Tasks Completed This Iteration" or "### Completed" subsection
   - Does NOT contain `<!-- REVIEW COMPLETE -->` marker
   - Process in order (Iteration 1 first, then 2, etc.)
3. **If NO iterations exist:** Treat entire plan as single iteration:
   - Look for "Completed" or "### Completed" section at plan level
   - Check if plan already has `<!-- REVIEW COMPLETE -->` marker
   - If completed but not reviewed → review as "Iteration 1"

**Iteration detection:** A plan has iterations if it contains `## Iteration` (with or without number).

An iteration is **READY FOR REVIEW** when it has "Tasks Completed This Iteration" (or legacy "### Completed") section and does NOT have `<!-- REVIEW COMPLETE -->` marker yet. The presence of `### Tasks Remaining` does NOT affect review readiness — review covers only the **completed tasks**.

If no iteration/plan needs review → Inform user and stop.

**Important:** Review ALL pending iterations in a single session, not just one.

## Collect Changed Files

From each iteration's "Tasks Completed This Iteration" (or legacy "Completed") section, list all files that were created, modified, or had tests added. This is the **review scope** — reviewers examine ONLY these files.

## Review Scope Assessment

Before spawning a review team, assess whether the overhead is justified. Three Sonnet reviewers each load full project context before reviewing — for small iterations, a single-agent sequential review is faster and cheaper.

### Count Changed Files

Count the unique files from "Collect Changed Files" above.

### Decision

| Changed files | Decision |
|---|---|
| ≤4 files | **Single-agent** — skip Team Setup, jump to Fallback: Single-Agent Mode |
| 5+ files | **3 reviewers** — proceed to Team Setup |

Announce: "N changed files — [single-agent/team] review mode."

**Override:** If the iteration includes security-sensitive changes (auth, session, tokens, API keys, crypto), always use the team regardless of file count — security review benefits from a dedicated specialist lens.

## Team Setup

**Skip this section if scope assessment chose single-agent mode** — jump to Fallback: Single-Agent Mode.

### Create the team

Use `TeamCreate`:
- `team_name`: "plan-review"
- `description`: "Parallel implementation review with domain-specialized reviewers"

**If TeamCreate fails**, switch to Fallback: Single-Agent Mode (see below).

### Create tasks

Use `TaskCreate` to create 3 review tasks:

1. **"Security review"** — Security & auth review of changed files
2. **"Reliability review"** — Bugs, async, resources, edge cases in changed files
3. **"Quality review"** — Type safety, conventions, test quality in changed files

### Spawn 3 reviewer teammates

Use the `Task` tool with `team_name: "plan-review"`, `subagent_type: "general-purpose"`, and `model: "sonnet"` to spawn each reviewer. Spawn all 3 in parallel (3 concurrent Task calls in one message).

Each reviewer prompt MUST include:
- The common preamble and their domain checklist from [references/reviewer-prompts.md](references/reviewer-prompts.md)
- The **exact list of changed files** to review (from Collect Changed Files step)
- Instructions to report findings as a structured message to the lead

### Assign tasks

After spawning, use `TaskUpdate` to assign each task to its reviewer by name.

## Coordination

While waiting for reviewer messages:
1. Reviewer messages are **automatically delivered** — do NOT poll or manually check inbox
2. Teammates go idle after each turn — this is normal, not an error. They're done when they send their findings message.
3. Track progress via `TaskList`
4. Acknowledge receipt as each reviewer reports
5. Wait until ALL 3 reviewers have reported before proceeding to merge

**If a reviewer gets stuck or stops without reporting:** Send them a message asking for their findings. If they don't respond, note that domain as "incomplete".

## Merge & Evaluate Findings

Once all reviewer findings are collected:

### Deduplicate
- Same code location reported by multiple reviewers → merge into the one with higher priority
- Same root cause in multiple locations → combine into one finding

### Classify Each Finding

Every finding MUST be classified as either **FIX** or **DISCARD**. There is no "document only" category — PLANS.md is overwritten by the next plan, so anything left there is lost.

| Classification | When to use | Action |
|----------------|-------------|--------|
| **FIX** | Real bug at ANY severity, regardless of origin | Create Linear issue + Fix Plan |
| **DISCARD** | Not a real bug — false positive, impossible, or not a bug | Report to user with reasoning |

**CRITICAL RULE: Pre-existing bugs are still bugs.** If a reviewer finds a real bug in a reviewed file, it MUST be classified as FIX — even if the bug existed before this iteration, even if it was introduced by a different PR, even if it's in code the current iteration didn't modify but is visible in a changed file. The review's job is to catch bugs, not to assign blame for when they were introduced. The only question is: "Is this a real bug?" If yes → FIX.

**FIX — real bugs at any severity:**
- Would cause runtime errors or crashes (any severity)
- Could corrupt or lose data
- Security vulnerability (OWASP categories)
- Resource leak affecting production
- Test doesn't actually test the behavior
- Violates CLAUDE.md critical rules
- Edge cases that could realistically occur
- Test flakiness from shared mutable state
- Convention violations that affect correctness
- Pre-existing bugs found in reviewed files (still real bugs)

**DISCARD — genuinely not bugs (the ONLY valid reasons):**
- False positive: reviewer misread the code or missed context — the code is actually correct
- Misdiagnosed: the flagged pattern is actually correct/intentional design
- Impossible in context: the scenario literally cannot occur given the app's constraints (e.g., "SQL injection" in a query that only accepts validated enum values)
- Style-only: cosmetic preference not enforced by CLAUDE.md, with zero correctness impact

**NOT valid reasons to DISCARD:**
- "Pre-existing" — a bug doesn't stop being a bug because it was there before
- "Already handled elsewhere" — if it's handled, prove it by citing the exact code; if you can't, it's a FIX
- "Low severity" — low severity bugs are still FIX, just with [low] tag
- "Out of scope for this iteration" — the review scope is the changed files, and any bug found in them is in scope

## Inline Fix Assessment

After evaluating findings, assess whether small bugs can be fixed inline instead of creating a full Fix Plan + another implementation cycle. This applies in both team and single-agent review modes.

### Step 1: Size each FIX finding

| Size | Description | Examples |
|---|---|---|
| **S** | Single-line or few-line surgical fix | Add missing try/catch, fix condition, add validation, fix off-by-one, add `await`, add log field |
| **M** | Moderate fix requiring investigation or multi-file changes | Refactor error handling, add timeout+retry, fix race condition |
| **L** | Substantial redesign or new abstraction | Redesign async flow, add middleware, restructure component |

### Step 2: Decision

| Condition | Action |
|---|---|
| ALL fixes S-size AND count ≤ 3 | **Fix inline** — proceed to Step 3 |
| Any M/L fix OR count > 3 | **Create Fix Plan** — skip to Document Findings |
| No FIX findings | Skip — proceed to Document Findings |

Announce: "N fix(es), all S-size — fixing inline." or "N fix(es) including M/L — creating Fix Plan."

### Step 3: Fix Inline (if eligible)

1. For each S-size fix, apply TDD:
   - Write failing test → run (`npx vitest run "pattern"`) → implement fix → run → verify pass
2. Run full test suite: `npm test`
3. Run bug-hunter:
   ```
   Task tool with subagent_type "bug-hunter"
   ```
4. **If clean:** Proceed to Document Findings with "fixed inline" format
5. **If bug-hunter finds new issues in the fixes:** Abandon inline approach — create Fix Plan as normal. Document the original findings plus new ones from bug-hunter.

### Linear Handling for Inline Fixes

Create Linear issues for traceability, but in "Merge" state (already fixed):
- `mcp__linear__create_issue` with `state: "Merge"`, `labels: ["Bug"]`

This preserves the audit trail without triggering the Todo → In Progress → Review workflow.

## Document Findings

### If Issues Found and Fixed Inline

When the inline fix assessment determined bugs could be fixed directly:

```markdown
### Review Findings

Summary: N issue(s) found, fixed inline ([single-agent review | Team: security, reliability, quality reviewers])
- FIXED INLINE: X issue(s) — verified via TDD + bug-hunter

**Issues fixed inline:**
- [MEDIUM] BUG: Missing try/catch in API handler (`src/lib/service.ts:142`) — added error handling + test
- [LOW] CONVENTION: Missing structured action field on log (`src/app/api/example/route.ts:55`) — added { action: "operation" }

**Discarded findings (not bugs):**
- [DISCARDED] ... (if any)

### Linear Updates
- PROJ-123: Review → Merge (original task)
- PROJ-130: Created in Merge (Fix: missing try/catch — fixed inline)
- PROJ-131: Created in Merge (Fix: missing log action — fixed inline)

### Inline Fix Verification
- Unit tests: all pass
- Bug-hunter: no new issues

<!-- REVIEW COMPLETE -->
```

No `## Fix Plan` is created — the bugs are already resolved. The iteration proceeds directly to the completion check.

### If Issues Found (creating Fix Plan)

Add Review Findings to the current Iteration section, then add Fix Plan at h2 level AFTER the iteration:

```markdown
### Review Findings

Summary: N issue(s) found (Team: security, reliability, quality reviewers)
- FIX: X issue(s) — Linear issues created
- DISCARDED: Y finding(s) — false positives / not applicable

**Issues requiring fix:**
- [CRITICAL] SECURITY: SQL injection in query builder (`src/db.ts:45`) - OWASP A03:2021
- [HIGH] BUG: Race condition in cache invalidation (`src/cache.ts:120`)
- [MEDIUM] TEST: Parallel test interference in api-keys assertions (`e2e/tests/api-keys.spec.ts:97`)

**Discarded findings (not bugs):**
- [DISCARDED] EDGE CASE: Unicode filenames not tested (`src/upload.ts:30`) — App only accepts JPEG/PNG/GIF/WebP/HEIC via validated content-type check; Unicode filenames are normalized by the browser before upload

### Linear Updates
- PROJ-123: Review → Merge (original task completed)
- PROJ-125: Created in Todo (Fix: SQL injection)
- PROJ-126: Created in Todo (Fix: Race condition)
- PROJ-127: Created in Todo (Fix: Parallel test interference)

<!-- REVIEW COMPLETE -->

---

## Fix Plan

**Source:** Review findings from Iteration N
**Linear Issues:** [PROJ-125](...), [PROJ-126](...), [PROJ-127](...)

### Fix 1: SQL injection in query builder
**Linear Issue:** [PROJ-125](...)

1. Write test in `src/db.test.ts` for malicious input escaping
2. Use parameterized query in `src/db.ts:45`

### Fix 2: Race condition in cache invalidation
**Linear Issue:** [PROJ-126](...)

1. Write test in `src/cache.test.ts` for concurrent invalidation
2. Add mutex/lock in `src/cache.ts:120`

### Fix 3: Parallel test interference
**Linear Issue:** [PROJ-127](...)

1. Remove fragile `No API keys` assertion after revoke
2. Verify revoked key absence without assuming empty state
```

**Note:** `<!-- REVIEW COMPLETE -->` is added even when issues are found — the review itself is complete. Fix Plan is at h2 level so `plan-implement` can find it.

### If No Issues Found (or all findings discarded)

```markdown
### Review Findings

Files reviewed: N
Reviewers: security, reliability, quality (agent team)
Checks applied: Security, Logic, Async, Resources, Type Safety, Conventions

No issues found - all implementations are correct and follow project conventions.

**Discarded findings (not bugs):**
- [DISCARDED] ... (if any findings were raised but classified as not-bugs)

### Linear Updates
- PROJ-123: Review → Merge
- PROJ-124: Review → Merge

<!-- REVIEW COMPLETE -->
```

**Then continue to the next iteration needing review.**

## Shutdown Team

After documenting findings for the current batch of iterations:
1. Send shutdown requests to all 3 reviewers using `SendMessage` with `type: "shutdown_request"`
2. Wait for shutdown confirmations
3. Use `TeamDelete` to remove team resources

## After ALL Iterations Reviewed

### Report Findings to User

**MANDATORY:** After all iterations are reviewed, present a direct summary to the user. PLANS.md is overwritten by the next plan, so the user must see all findings NOW.

**Report format (output directly to user as text):**

```
Review complete for [N] iteration(s). [M] files reviewed.

BUGS FIXED (Linear issues created):
- [CRITICAL] SECURITY: SQL injection in query builder (PROJ-125)
- [HIGH] BUG: Race condition in cache invalidation (PROJ-126)

DISCARDED FINDINGS (not bugs):
- EDGE CASE: Unicode filenames not tested — App only accepts validated content-types; filenames normalized by browser
- TYPE: Potential null in response — Actually guarded by early return on line 42

[or: No issues found — clean review.]
```

If there were no findings at all (clean review), a brief "No issues found" summary is sufficient.

### Determine Completion

- **If Fix Plan exists OR tasks remain unfinished** (and no inline fixes resolved all issues) → Do NOT mark complete. More implementation needed.
  1. **Commit and push** (see Termination section)
  2. Inform user: "Review complete. Changes committed and pushed. Run `/plan-implement` to continue implementation."

- **If all tasks complete and no fix plans needed** (includes iterations where all bugs were fixed inline) → Run E2E tests, update header status, append final status, then create PR:
  1. **Run E2E tests** using the verifier agent in E2E mode:
     ```
     Use Task tool with subagent_type "verifier" with prompt "e2e"
     ```
     If E2E tests fail, do NOT mark complete — create new Linear issues in Todo for the failures (same as review findings), add a Fix Plan, commit/push, and inform user to run `/plan-implement`.
  2. **Update the header** on line 3: change `**Status:** IN_PROGRESS` to `**Status:** COMPLETE`
  3. **Append** the final status section at the bottom of the file:

```markdown
---

## Status: COMPLETE

All tasks implemented and reviewed successfully. All Linear issues moved to Merge.
```

**Then create the PR:**
1. Commit any uncommitted changes
2. Push to remote
3. Create PR using the `pr-creator` subagent
4. Inform user with PR URL

## Fallback: Single-Agent Mode

If the scope assessment chose single-agent mode (≤4 changed files) OR `TeamCreate` fails, perform the review as a single agent:

1. **Inform user:** "N changed files — single-agent review mode." (or "Agent teams unavailable. Running review in single-agent mode." if TeamCreate failed)
2. For each iteration needing review:
   a. Identify changed files from "Tasks Completed This Iteration"
   b. Read each file and apply checks from [references/code-review-checklist.md](references/code-review-checklist.md)
   c. Apply all domain checks (security, reliability, quality) sequentially
   d. Classify findings (same Merge & Evaluate Findings rules)
   e. **Run Inline Fix Assessment** — same threshold applies (≤3 S-size fixes → fix inline)
   f. Document findings (same format as team mode, including "fixed inline" format if applicable)
   g. Handle Linear updates (same as team mode)
3. Continue with "After ALL Iterations Reviewed" section

## Context Management & Continuation

**Context is checked ONLY at iteration boundaries.** Never stop mid-iteration review.

**After completing each iteration review**, estimate remaining context:
- If **> 60%** → Continue to next pending iteration
- If **<= 60%** → Stop, commit/push, inform user to re-run

**Never stop mid-iteration** — once started, complete the full review for that iteration.

## Issue Categories Reference

| Tag | Description | Default Severity |
|-----|-------------|------------------|
| `SECURITY` | Injection, auth bypass, secrets exposure, IDOR | CRITICAL/HIGH |
| `BUG` | Logic errors, off-by-one, null handling | HIGH |
| `ASYNC` | Unhandled promises, race conditions | HIGH |
| `RESOURCE` | Memory/resource leaks, missing cleanup | HIGH |
| `TIMEOUT` | Missing timeouts, potential hangs | HIGH/MEDIUM |
| `EDGE CASE` | Unhandled scenarios, boundary conditions | MEDIUM |
| `TYPE` | Unsafe casts, missing type guards | MEDIUM |
| `ERROR` | Missing or incorrect error handling | MEDIUM |
| `CONVENTION` | CLAUDE.md violations | LOW-MEDIUM |

## Error Handling

| Situation | Action |
|-----------|--------|
| PLANS.md doesn't exist | Stop — "No plan found." |
| No iteration needs review | Stop — "No iteration to review. Run plan-implement first." |
| Plan has no iterations | Treat entire plan as single iteration |
| Files in iteration don't exist | Note as issue — implementation may have failed |
| CLAUDE.md doesn't exist | Use general coding best practices |
| TeamCreate fails | Switch to single-agent fallback mode |
| Reviewer stops without reporting | Send follow-up message, note domain as incomplete |
| Too many issues found | Create fix plan for all real bugs; discard false positives with reasoning |
| Inline fix test fails | Abandon inline approach — create Fix Plan for all findings (original + new) |
| Inline fix introduces new bugs | Abandon inline approach — create Fix Plan with combined findings |

## Termination: Commit, Push, and PR

**MANDATORY:** Before ending, commit all local changes and push to remote.

### For Incomplete Plans
1. Stage modified files: `git status --porcelain=v1`, then `git add <file> ...` — **skip** `.env*`, `*.key`, `*.pem`, `credentials*`, `secrets*`
2. Commit (no `Co-Authored-By` tags): `plan: review iteration N - [issues found | no issues]`
3. `git push`
4. Inform user to run `/plan-implement`

### For Complete Plans
1. Stage modified files: `git status --porcelain=v1`, then `git add <file> ...` — **skip** `.env*`, `*.key`, `*.pem`, `credentials*`, `secrets*`
2. Commit (no `Co-Authored-By` tags): `plan: mark [plan-name] complete`
3. `git push`
4. **Collect ALL Linear issue identifiers** managed during this session:
   - Original plan issues (from PLANS.md header)
   - Issues moved Review → Merge during review
   - Inline-fix issues created in Merge state
   - Fix-plan issues that completed and were moved to Merge
   Deduplicate and sort numerically. Format as: `PROJ-123, PROJ-124, ...`
5. Create PR using the `pr-creator` subagent. **Include in the prompt:** `Linear issues to close: PROJ-123, PROJ-124, ...` with the full list from step 4. This overrides PLANS.md scanning and ensures no inline-fix issues are missed.
6. Inform user with PR URL

**Branch handling:** Assumes plan-implement already created a feature branch. If on `main`, create branch first.

## Rules

- **Review ALL pending iterations** — Don't stop after one
- **Do not modify source code during review** — Review only, document findings. Exception: when inline-fixing ≤3 S-size bugs (see Inline Fix Assessment), the lead applies TDD fixes directly after the review phase completes.
- **Be specific** — Include file paths and line numbers for every issue
- **One fix per issue** — Each finding must have a matching Fix task with Linear issue
- **Fix Plan follows TDD** — Test first for each fix
- **Never modify previous sections** — Only add to current iteration or append status
- **Mark COMPLETE only when ALL iterations pass** — No fix plans pending, all reviewed
- **Move issues to Merge** — All reviewed issues that pass go Review→Merge
- **Create bug issues** — Fix Plan bugs → new issues in Todo state. Inline-fixed bugs → new issues in Merge state.
- **Always commit and push at termination** — Never end without committing progress
- **Create PR when plan is complete** — Use pr-creator subagent for final PR
- **Lead handles all Linear/git writes** — Reviewers NEVER create issues or modify PLANS.md
- **No co-author attribution** — Commit messages must NOT include `Co-Authored-By` tags
- **Never stage sensitive files** — Skip `.env*`, `*.key`, `*.pem`, `credentials*`, `secrets*`
- **Check MIGRATIONS.md** — If implementation changed DB schema, column names, session/token formats, or env vars, verify that `MIGRATIONS.md` has a corresponding note. If missing, add it as a finding and create a Fix Plan task. The lead should append the missing note to `MIGRATIONS.md` before committing.
- **Every finding is FIX or DISCARD** — No "document only" category. Real bugs at any severity get Linear issues + Fix Plan. Non-bugs get discarded with reasoning.
- **Never discard real bugs** — Pre-existing, low-severity, or "already handled" are NOT valid reasons to discard. Only discard findings that are genuinely not bugs (false positive, impossible, style-only). When in doubt, FIX.
- **Always report findings to user** — PLANS.md is overwritten by the next plan. The user must see all findings (fixed and discarded) directly in the conversation output before the skill terminates.
- **Inline fix threshold: ≤3 S-size fixes** — When all FIX findings are S-size (surgical, single-line or few-line) and count ≤3, fix them inline with TDD instead of creating a Fix Plan. Abandon inline approach if tests fail or bug-hunter finds new issues.
- **Inline fixes still get Linear issues** — Create issues in "Merge" state for traceability. Never skip the audit trail.
- **Review scope assessment** — ≤4 changed files → single-agent review. 5+ files → 3 reviewers. Always use team for security-sensitive changes regardless of file count.
