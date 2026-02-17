---
name: plan-implement
description: Execute the pending plan in PLANS.md using an agent team for parallel implementation. Use when user says "implement the plan", "execute the plan", "team implement", or after any plan-* skill creates a plan. Spawns worker agents in isolated git worktrees for full code isolation. Updates Linear issues in real-time. Falls back to single-agent mode if agent teams unavailable.
allowed-tools: Read, Edit, Write, Glob, Grep, Task, Bash, TeamCreate, TeamDelete, SendMessage, TaskCreate, TaskUpdate, TaskList, TaskGet, mcp__linear__list_teams, mcp__linear__list_issues, mcp__linear__get_issue, mcp__linear__update_issue, mcp__linear__list_issue_statuses
disable-model-invocation: true
---

Execute the current pending work in PLANS.md using an agent team for parallel implementation. You are the **team lead/coordinator**. You break the plan into domain-based work units, create isolated git worktrees for each worker, spawn worker agents, coordinate their progress, merge their work, and handle verification and documentation.

Each worker operates in its own **git worktree** — a fully isolated working directory with its own branch, staging area, and `node_modules`. Workers cannot corrupt each other's files. Task assignment is **domain-based** — overlapping file edits are acceptable and resolved by the lead during the merge phase.

**If agent teams are unavailable** (TeamCreate fails), fall back to single-agent mode — see "Fallback: Single-Agent Mode" section.

## Pre-flight Check

1. **Read PLANS.md** — Understand the full context and history
2. **Read CLAUDE.md** — Understand TDD workflow and project rules
3. **Verify Linear MCP** — Call `mcp__linear__list_teams`. If unavailable, STOP and tell the user: "Linear MCP is not connected. Run `/mcp` to reconnect, then re-run this skill."
4. **Identify pending work** — Use this priority order:
   - Check latest Iteration block for "Tasks Remaining" section
   - Look for `## Fix Plan` (h2 level) with no iteration after it
   - Original Plan with no "Iteration 1" → Execute from Task 1
   - Nothing pending → Inform user "No pending work in PLANS.md"

## Work Partitioning

Group pending tasks into work units by **domain** — related areas of the codebase that form coherent implementation units. Workers MAY touch overlapping files; the lead resolves conflicts during the merge phase.

### Analyze Task Domains

For each pending task in PLANS.md:
1. Read the task description and files to understand its domain
2. Identify the primary layer: types/schema → service/business logic → API routes → UI components
3. Note cross-cutting concerns (shared types, utilities, config)

### Partition Into Work Units

Group tasks into work units where:
- Tasks in the same domain or tightly coupled belong together
- Cross-cutting tasks go with the domain they most closely relate to
- Work is spread roughly evenly across units

**Partitioning guidelines:**
1. Group by functional domain: "auth flow," "data pipeline," "notification system"
2. Prefer grouping tasks that depend on each other's output
3. When in doubt, group tasks together rather than splitting them

**Deciding the number of workers:**

| Work units | Workers |
|-----------|---------|
| 1 | 1 (still benefits from dedicated context) |
| 2 | 2 |
| 3 | 3 |
| 4+ | Cap at 4 (diminishing returns, coordination overhead) |

### Reserve Generated-File Tasks for the Lead

Tasks involving CLI tools that **generate** files (e.g., `npx drizzle-kit generate`) MUST NOT be assigned to workers. Workers hand-write generated files instead of running the command, producing corrupt output.

**How to handle:**
1. Identify any task whose steps include a generator command
2. Remove those tasks from worker assignments
3. The lead runs them after the merge phase
4. Note in partition log: "Task N: [title] — reserved for lead (generated files)"

### Verify Partition

Before proceeding, verify:
- [ ] Every pending task is assigned to exactly one work unit (or reserved for lead)
- [ ] Task ordering within each work unit respects dependencies
- [ ] Each work unit has a clear scope description
- [ ] No work unit contains tasks that run file-generation CLI tools

**Log the partition plan** — output to the user so they can see how work is divided.

## Worktree Setup

### Determine Feature Branch

If on `main`, create a feature branch:
```bash
git checkout -b feat/<plan-name>
```
If already on a feature branch, stay on it. Record the branch name as `FEATURE_BRANCH`.

### Clean Up Previous Runs

Remove any leftover worktrees and branches from a previous failed run:
```bash
git worktree prune
# For each worker N:
git branch -D <FEATURE_BRANCH>/worker-N 2>/dev/null || true
rm -rf _workers/
```

### Create Worker Worktrees

For each worker:
```bash
git worktree add _workers/worker-N -b <FEATURE_BRANCH>/worker-N
```

Example: if `FEATURE_BRANCH` is `feat/proj-123-feature`, worker branches are:
- `feat/proj-123-feature/worker-1`
- `feat/proj-123-feature/worker-2`

### Bootstrap Worktree Environments

Each worktree needs dependencies and environment variables:
```bash
# For each worker N:
cp -r node_modules _workers/worker-N/node_modules
cp .env _workers/worker-N/.env 2>/dev/null || true
cp .env.local _workers/worker-N/.env.local 2>/dev/null || true
```

`cp -r node_modules` takes ~5-10s (faster than `npm ci`). Workers don't install new packages — the lead handles that after merge if needed.

### Worktree Setup Failure

If `git worktree add` fails:
1. Clean up: `git worktree prune && rm -rf _workers/`
2. Delete any created branches: `git branch -D <FEATURE_BRANCH>/worker-N 2>/dev/null || true`
3. Fall back to single-agent mode
4. Inform user: "Worktree setup failed. Falling back to single-agent mode."

## Team Setup

### Create the team

Use `TeamCreate`:
- `team_name`: "plan-implement"
- `description`: "Parallel plan implementation with worktree-isolated workers"

**If TeamCreate fails**, clean up worktrees and switch to Fallback: Single-Agent Mode.

### Create tasks

Use `TaskCreate` for each work unit:
- Subject: "Work Unit N: [brief scope/domain description]"
- Description: list of plan tasks assigned to this unit

### Spawn workers

Use `Task` tool with `team_name: "plan-implement"`, `subagent_type: "general-purpose"`, `model: "sonnet"`, and `mode: "bypassPermissions"` for each worker. Name them `worker-1`, `worker-2`, etc.

Spawn all workers in parallel (concurrent Task calls in one message).

### Worker Prompt Template

Each worker gets this prompt (substitute the specific values):

```
You are an implementation worker for this project. You implement plan tasks following strict TDD workflow in your own isolated git worktree.

WORKSPACE:
Your isolated workspace is at: {absolute_project_path}/_workers/worker-{N}
**FIRST ACTION:** Run: cd {absolute_project_path}/_workers/worker-{N}
All your work happens in this directory. You have a complete, independent copy of the repository with its own node_modules.

ASSIGNED TASKS:
{paste the full task descriptions from PLANS.md for this work unit}

RULES:
- **FIRST:** cd to your workspace directory before any other action
- Follow TDD strictly: write test → run test (expect fail) → implement → run test (expect pass)
- **E2E TEST EXCEPTION:** If writing Playwright E2E tests (`e2e/tests/*.spec.ts`), write the spec but SKIP the run steps. The lead runs E2E tests after merging.
- Read CLAUDE.md in your workspace for project conventions before starting
- Run tests via Bash: npx vitest run "pattern" (e.g., npx vitest run "session")
- Do NOT run the build (npm run build), full test suite (npm test), or E2E tests
- Report progress to the lead after completing each task
- Do NOT update Linear issues — the lead handles all Linear state transitions
- NEVER hand-write generated files (migrations, snapshots). Report as blocker — the lead handles it.
- NEVER use Bash for file operations (no sed, awk, cat >, echo >, tee, cp, mv, rm, mkdir). Use Edit/Write tools for ALL file modifications. Acceptable Bash uses: (1) cd to workspace, (2) npx vitest run, (3) git add/commit at the end.

WORKFLOW FOR EACH TASK:

For unit/integration test tasks (files in src/):
1. Send message to lead: "Starting Task N: [title] [PROJ-XXX]"
2. Read relevant existing source files to understand patterns
3. Write failing test(s) in the appropriate __tests__/ directory
4. Run: npx vitest run "test-pattern" — confirm test fails
5. Implement the minimal code to make test pass
6. Run: npx vitest run "test-pattern" — confirm test passes
7. Send message to lead: "Completed Task N: [title] [PROJ-XXX]"
8. Move to next task

For E2E test tasks (files in e2e/tests/*.spec.ts):
1. Send message to lead: "Starting Task N: [title] [PROJ-XXX]"
2. Read existing E2E specs and fixtures to understand patterns
3. Write the Playwright spec file — DO NOT run it
4. Send message to lead: "Completed Task N: [title] [PROJ-XXX] — E2E spec written"
5. Move to next task

WHEN ALL TASKS ARE DONE:
1. Commit all changes in your workspace:
   git add -A
   git commit -m "worker-{N}: [brief summary of all changes]"
   Do NOT push.
2. Send final summary to the lead:
---
WORKER: worker-{N}
STATUS: COMPLETE
TASKS COMPLETED:
- Task N: [title] (PROJ-XXX) - [what was done]
FILES MODIFIED:
- path/to/file.ts - [what changed]
COMMIT: [output of: git log --oneline -1]
---

If you encounter a blocker, send a message to the lead describing it. Do NOT guess or work around it.
```

### Assign tasks and label issues

After spawning, for each work unit:
1. `TaskUpdate` to assign each task to its worker by name
2. Label Linear issues with worker label using `mcp__linear__update_issue`:
   - Worker 1 → "Worker 1", Worker 2 → "Worker 2", etc.
   - Add label to existing labels (don't replace)

## Linear State Management

**CRITICAL:** Workers do NOT have access to Linear MCP tools. The lead handles ALL Linear state transitions.

**When a worker REPORTS starting a task:**
1. Parse the issue ID from the worker's message
2. IMMEDIATELY move the issue to "In Progress" using `mcp__linear__update_issue`

**When a worker REPORTS completing a task:**
1. Parse the issue ID from the worker's message
2. IMMEDIATELY move the issue to "Review" using `mcp__linear__update_issue`
3. Acknowledge the worker's completion

**If a task has no Linear issue link**, skip state updates for that task.

## Coordination (while workers work)

### Message Handling

1. Worker messages are **automatically delivered** — do NOT poll
2. Teammates go idle after each turn — normal and expected
3. Track progress via `TaskList`
4. Acknowledge each worker's task completion
5. If a worker reports a blocker, help resolve it

### Handling Blockers

| Blocker Type | Action |
|-------------|--------|
| Worker needs another worker's code | Tell them to proceed with their best assumption — conflicts resolved at merge |
| Test failure worker can't resolve | Read the failing test output, provide guidance |
| Unclear requirements | Re-read PLANS.md, provide clarification |
| Generated file needed | Acknowledge — lead handles it post-merge |

## Post-Worker Phase

Once ALL workers have reported completion and committed their changes:

### 1. Shutdown Workers

1. Send shutdown requests to all workers using `SendMessage` with `type: "shutdown_request"`
2. Wait for shutdown confirmations
3. Mark all work unit tasks as completed via `TaskUpdate`

### 2. Merge Worker Branches

Merge worker branches into the feature branch **one at a time, foundation-first**.

**Determine merge order:**
- Workers handling lower-level code merge first: types/schemas → services → API routes → UI
- If workers are at the same layer, merge by worker number
- The first merge is always a fast-forward (feature branch hasn't moved)

**For each worker branch (in order):**
```bash
git merge <FEATURE_BRANCH>/worker-N
```

**After each merge (starting from the second):**
```bash
npx tsc --noEmit
```
If type errors → fix them before merging the next worker. This catches integration issues early before they compound.

**If a merge has conflicts:**
1. Review the conflicting files — understand both workers' intent from the plan
2. Resolve conflicts, keeping correct logic from both sides
3. `git add` resolved files, then `git commit` (git's auto-generated merge message is fine)
4. Run `npx tsc --noEmit` before continuing to the next merge

### 3. Run Lead-Reserved Tasks (Generated Files)

If any tasks were reserved for the lead during partitioning:
1. Run the CLI command (e.g., `npx drizzle-kit generate`)
2. Verify output files are correct
3. If the generator produces no changes, investigate — workers may have missed a schema change

### 4. Install New Dependencies (if needed)

If the plan required new npm packages that workers couldn't install:
```bash
npm install <package-name>
```

### 5. Run E2E Tests (if workers wrote E2E specs)

Run the `verifier` agent in E2E mode:
```
Task tool with subagent_type "verifier" and prompt "e2e"
```
If E2E tests fail → fix the specs directly, then re-run.

### 6. Run Full Verification

**Bug hunter:**
```
Task tool with subagent_type "bug-hunter"
```

Fix ALL real bugs — pre-existing or new. Only skip verifiable false positives.

**Full test suite:**
```
Task tool with subagent_type "verifier"
```

If failures → fix directly (workers are shut down by this point).

## Document Results

After verification passes, append a new "Iteration N" section to PLANS.md:

```markdown
---

## Iteration N

**Implemented:** YYYY-MM-DD
**Method:** Agent team (N workers, worktree-isolated)
[OR: **Method:** Single-agent (team unavailable)]

### Tasks Completed This Iteration
- Task 3: Fix session validation - Updated middleware, added tests (worker-1)
- Task 4: Add health check - Created /api/health endpoint (worker-2)

### Tasks Remaining
- Task 5: Add token refresh
(omit this section if ALL tasks completed)

### Files Modified
- `src/lib/session.ts` - Updated session validation logic
- `src/app/api/health/route.ts` - Created health endpoint

### Linear Updates
- PROJ-9: Todo → In Progress → Review
- PROJ-10: Todo → In Progress → Review

### Pre-commit Verification
- bug-hunter: [Passed | Found N bugs, fixed before proceeding]
- verifier: All N tests pass, zero warnings

### Work Partition
- Worker 1: Tasks 3 (auth domain — session, middleware)
- Worker 2: Task 4 (API domain — health endpoint)
(omit in single-agent mode)

### Merge Summary
- Worker 1: fast-forward (no conflicts)
- Worker 2: merged, 1 conflict in src/types/index.ts (resolved)
(omit in single-agent mode)

### Continuation Status
[All tasks completed.]
OR
[Point budget reached. More tasks remain.]
```

**IMPORTANT:**
- Do NOT add "Review Findings" or "Notes" sections - reserved for `plan-review-implementation`
- Always list completed tasks in "Tasks Completed This Iteration"
- If stopping early, also list remaining tasks in "Tasks Remaining"
- If ALL tasks are complete, OMIT the "Tasks Remaining" section entirely

**Note:** The presence of "Tasks Remaining" does NOT prevent review. `plan-review-implementation` will review the completed tasks regardless.

## Cleanup

After documenting results (skip in single-agent fallback mode), the lead MUST clean up everything:

### 1. Remove Worktrees

```bash
# Remove each worktree (--force handles any uncommitted leftovers)
git worktree remove _workers/worker-1 --force
git worktree remove _workers/worker-2 --force
# ... repeat for each worker
```

### 2. Remove Worker Directory

```bash
# Safety net — remove the entire _workers/ directory
rm -rf _workers/

# Prune stale worktree metadata from .git/worktrees/
git worktree prune
```

### 3. Delete Worker Branches

```bash
# Worker branches are already merged — safe delete
git branch -d <FEATURE_BRANCH>/worker-1
git branch -d <FEATURE_BRANCH>/worker-2
# ... repeat for each worker
```

### 4. Sync Dependencies

```bash
# Ensure main project node_modules matches merged package.json/lock file
npm install
```

This catches any dependency changes from merged code (new imports, updated lock file entries). Fast no-op if nothing changed.

### 5. Verify Clean State

```bash
git worktree list
```

Should show only the main worktree. If stale entries remain, run `git worktree prune` again.

### 6. Delete Team Resources

Use `TeamDelete` to remove team resources.

## Fallback: Single-Agent Mode

If `TeamCreate` fails or worktree setup fails, implement the plan sequentially as a single agent:

1. **Inform user:** "Agent teams/worktrees unavailable. Implementing in single-agent mode."
2. **Clean up** any partially created worktrees: `git worktree prune && rm -rf _workers/`
3. **Follow TDD strictly** for each task:
   - Move Linear issue Todo → In Progress
   - Write failing test → run test (expect fail) → implement → run test (expect pass)
   - Move Linear issue In Progress → Review
4. **Track point budget** as a proxy for context usage:

   | Tool call type | Points |
   |----------------|--------|
   | Glob, Grep, Edit, MCP call (Linear etc.) | 1 |
   | Read, Write | 2 |
   | Bash (test run, build, git) | 3 |
   | Task subagent (verifier, bug-hunter) | 5 |

   | Cumulative points | Action |
   |-------------------|--------|
   | **< 200** | Continue to next task |
   | **200–230** | Continue only if next task is small (≤ 3 files) |
   | **> 230** | **STOP** — run pre-stop checklist immediately |

5. **Pre-stop checklist** (run when stopping, regardless of reason):
   - Run `bug-hunter` agent — fix ALL real bugs found
   - Run `verifier` agent — fix any failures or warnings
6. **Document results** — Same Iteration block format (omit Work Partition and Merge Summary)

## Termination: Commit and Push

**MANDATORY:** After cleanup (or after documenting results in single-agent mode), commit all changes and push.

**Steps:**
1. Stage modified files: `git status --porcelain=v1`, then `git add <file> ...` — **skip** files matching `.env*`, `*.key`, `*.pem`, `credentials*`, `secrets*`
2. Create commit (do **not** include `Co-Authored-By` tags):
   ```
   plan: implement iteration N - [brief summary]

   Tasks completed:
   - Task X: [title]
   - Task Y: [title]

   Method: agent team (N workers, worktree-isolated)
   ```
   (Use "Method: single-agent" in fallback mode)
3. Push to current branch: `git push`

**Branch handling:**
- If on `main`, create a feature branch first: `git checkout -b feat/[plan-name]`
- If already on a feature branch, push to that branch

## Error Handling

| Situation | Action |
|-----------|--------|
| PLANS.md doesn't exist or is empty | STOP — "No plan found. Run plan-backlog or plan-inline first." |
| PLANS.md has "Status: COMPLETE" | STOP — "Plan already complete. Create a new plan first." |
| `git worktree add` fails | Clean up, fall back to single-agent mode |
| TeamCreate fails | Clean up worktrees, switch to single-agent fallback |
| Worker branch already exists | Delete it first: `git branch -D <branch> 2>/dev/null` |
| All tasks in same domain (1 unit) | Use 1 worker — still benefits from isolated context |
| Worker stops without reporting | Enter their worktree, review state, commit if work is usable, fix directly |
| Merge conflict | Resolve in feature branch, run typecheck, continue merging |
| Type errors after merge | Fix before merging next worker |
| Integration failures after all merges | Fix directly in verification phase |
| Test won't fail in step 2 (single-agent) | Review test logic — ensure it tests new behavior |
| Test won't pass in step 4 (single-agent) | Debug implementation, do not skip |

## Scope Boundaries

**This skill implements plans. It does NOT:**
1. **NEVER create PRs** — PRs are created by plan-review-implementation
2. **NEVER skip failing tests** — Fix them
3. **NEVER modify PLANS.md sections above current iteration** — Append only
4. **NEVER proceed with warnings** — Fix all warnings first
5. **NEVER ask "should I continue?"** — Use context estimation to decide automatically (single-agent mode)

## Rules

- **Domain-based partitioning** — Group tasks by functional domain. Overlapping files are acceptable; the lead resolves conflicts at merge time.
- **Follow TDD strictly** — Test before implementation, always
- **Fix ALL real bugs** — Every bug found by bug-hunter must be fixed, whether pre-existing or new. Only skip verifiable false positives.
- **Fix failures immediately** — Do not proceed with failing tests or warnings
- **Never modify previous sections** — Only append new Iteration section to PLANS.md
- **Always commit and push at termination** — Never end without committing progress
- **Document completed AND remaining tasks** — So next iteration knows where to resume
- **Lead updates Linear in real-time** — Workers do NOT have MCP access
- **Cap at 4 workers** — More = more overhead, diminishing returns
- **Lead does NOT implement** — Delegate all implementation to workers. Lead only coordinates, merges, verifies, and documents. (Exception: single-agent fallback and post-merge fixes.)
- **Lead runs all CLI generators** — Drizzle-kit, prisma generate, etc. reserved for lead post-merge
- **Workers test via vitest only** — `npx vitest run "pattern"` in their worktree. No build, no full suite, no E2E.
- **E2E test tasks are write-only for workers** — Workers write specs but do NOT run them
- **Foundation-first merge order** — Merge lower-level workers first (types → services → routes → UI). Typecheck gate (`npx tsc --noEmit`) after each merge.
- **Workers commit, don't push** — Workers `git add -A && git commit` in their worktree. Lead merges locally via the shared git object database.
- **Always clean up worktrees** — Remove worktrees, prune metadata, delete worker branches after merge
- **No co-author attribution** — Commit messages must NOT include `Co-Authored-By` tags
- **Never stage sensitive files** — Skip `.env*`, `*.key`, `*.pem`, `credentials*`, `secrets*`
- **Log migrations in MIGRATIONS.md** — Workers report migration-relevant changes to lead; lead appends to MIGRATIONS.md
