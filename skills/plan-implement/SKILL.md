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

## Scope Assessment

Before partitioning into work units, assess whether workers are justified:

1. Count the pending tasks/fixes
2. Estimate total files modified (from the plan descriptions)

| Pending tasks | Est. files modified | Decision |
|---------------|---------------------|----------|
| 1–3 tasks OR ≤6 files total | Small batch | **Skip workers → single-agent mode** |
| 4+ tasks AND >6 files total | Medium+ batch | Proceed with workers |

For small batches, announce: "Small batch (N tasks, ~M files) — implementing in single-agent mode for efficiency." Then jump directly to "Fallback: Single-Agent Mode."

**Rationale:** Worker overhead (worktree setup, team creation, task assignment, merge, cleanup) exceeds implementation time for small batches.

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
git branch -D <FEATURE_BRANCH>-worker-N 2>/dev/null || true
rm -rf _workers/
```

### Create Worker Worktrees

For each worker:
```bash
git worktree add _workers/worker-N -b <FEATURE_BRANCH>-worker-N
```

**IMPORTANT:** Use a hyphen (`-worker-N`), NOT a slash (`/worker-N`). Git cannot create `refs/heads/feat/foo-123/worker-1` when `refs/heads/feat/foo-123` already exists as a branch ref.

Example: if `FEATURE_BRANCH` is `feat/proj-123-notifications`, worker branches are:
- `feat/proj-123-notifications-worker-1`
- `feat/proj-123-notifications-worker-2`

### Bootstrap Worktree Environments

**Pre-check:** Verify `.gitignore` covers symlinks before creating them. The `node_modules/` entry (with trailing slash) only matches directories — a symlink is a file and won't be excluded. Ensure a bare `node_modules` entry exists:
```bash
grep -q '^node_modules$' .gitignore || sed -i '' '/^node_modules\//i\
node_modules' .gitignore
```

Each worktree needs dependencies and environment variables:
```bash
# For each worker N:
ln -s "$(pwd)/node_modules" _workers/worker-N/node_modules
cp .env _workers/worker-N/.env 2>/dev/null || true
cp .env.local _workers/worker-N/.env.local 2>/dev/null || true
```

**Why symlink, not copy:** `cp -r node_modules` breaks `.bin/` symlinks on macOS — `cp -r` dereferences symlinks, turning `.bin/vitest -> ../vitest/vitest.mjs` into a regular file containing `import './dist/cli.js'` that can't resolve. Symlinking is instant and avoids the issue entirely. Workers don't install packages, so a shared read-only reference is safe.

### Worktree Setup Failure

If `git worktree add` fails:
1. Clean up: `git worktree prune && rm -rf _workers/`
2. Delete any created branches: `git branch -D <FEATURE_BRANCH>-worker-N 2>/dev/null || true`
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
You are worker-{N} for this project.

FIRST ACTION: Run via Bash: cd {absolute_project_path}/_workers/worker-{N}
Then read CLAUDE.md in your workspace. Follow its TDD workflow and conventions strictly.

ASSIGNED TASKS:
{paste the full task descriptions from PLANS.md for this work unit}

{TESTING_CONTEXT — optional, see "Lead Populates Testing Context" below}

TOOL USAGE (memorize — no exceptions):
| I want to...           | Use this tool                     | NEVER use               |
|------------------------|-----------------------------------|-------------------------|
| Read a file            | Read tool                         | cat, head, tail, less   |
| Find files by name     | Glob tool                         | find, ls                |
| Search file contents   | Grep tool                         | grep, rg, ag            |
| Edit an existing file  | Edit tool                         | sed, awk                |
| Create a new file      | Write tool                        | echo >, cat <<, tee     |
| Run tests              | Bash: npx vitest run "pattern"    |                         |
| Typecheck              | Bash: npm run typecheck           |                         |
| Commit at the end      | Bash: git add -A && git commit    |                         |
| Anything else via Bash | **STOP — ask the lead first**     |                         |

Using Bash for file operations (including reads like ls, find, grep) triggers
permission prompts on the lead's terminal. Use the dedicated tools above.

RULES:
- TDD: write failing test → run (expect fail) → implement → run (expect pass). See CLAUDE.md.
- Tests: `npx vitest run "pattern"` only. NEVER run npm test, npm run build, or E2E tests.
- **E2E specs** (`e2e/tests/*.spec.ts`): write the spec file but do NOT run it. The lead runs E2E after merging.
- Report "Starting Task N: [title] [PROJ-XXX]" and "Completed Task N: [title] [PROJ-XXX]" to the lead for each task.
- Do NOT update Linear issues — the lead handles all state transitions.
- NEVER hand-write generated files (migrations, snapshots). Report as blocker.

WHEN ALL TASKS DONE:
1. npm run typecheck — fix any type errors
2. Commit:
   git add -A -- ':!node_modules' ':!.env' ':!.env.local'
   git commit -m "worker-{N}: [summary]

   Tasks: Task X (PROJ-XXX), Task Y (PROJ-YYY)
   Files: path/to/file.ts, path/to/other.ts"
   Do NOT push.
3. Send final summary to the lead (MUST send before going idle):
   WORKER: worker-{N} | STATUS: COMPLETE
   TASKS: [list with PROJ-XXX ids and what was done]
   FILES: [list of modified files]
   COMMIT: [git log --oneline -1 output]

If blocked, message the lead. Do NOT guess or work around it.
```

### Lead Populates Testing Context

Before spawning workers, the lead reads 1-2 existing test files from the domains workers will touch. Extract testing gotchas that workers would otherwise discover by trial and error. Insert as a `TESTING NOTES` block where `{TESTING_CONTEXT}` appears. Omit if the tasks are straightforward.

**Example for React component tasks:**
```
TESTING NOTES:
- React 19 + testing-library v16: wrap async triggers in await act(async () => { ... })
- For tests with FileReader macrotasks (Blob conversion), waitFor is still needed for the fetch assertion
- Add mockFetch.mockReset() to beforeEach to prevent mock queue leakage
```

**Example for API route tasks:**
```
TESTING NOTES:
- Route tests mock @/lib/session and @/lib/claude at module level
- SSE route tests need a consumeSSEStream helper — check existing test files for the pattern
```

### Conditional Protocol Consistency Block

When tasks define or extend an **event protocol** (e.g., `StreamEvent`, WebSocket messages, API response shapes), append this to the worker prompt after the task descriptions. **Omit for all other tasks.**

```
PROTOCOL CONSISTENCY: These tasks define/extend a streaming event protocol.
Every code path must yield the SAME set of event types in consistent order:
- ALL exit paths yield at minimum: [usage] + [result event] + [done]
- Error paths yield either [error] OR [result + done], never both
- No path silently returns without a terminal event
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

### Worker Startup Grace Period

After spawning workers, wait at least **5 minutes** before taking any corrective action. Workers need 2–4 turns to: cd to workspace, validate environment, read CLAUDE.md, read source files, and send their first "Starting Task" message.

**During the grace period:**
- Idle notifications are EXPECTED and normal — do not react to them
- Do NOT send status check messages until 5 minutes have passed
- Do NOT delete worktrees, remove branches, or clean up
- You MAY acknowledge worker messages if they arrive

**After 5 minutes with no messages from a worker:**
1. Check the worktree for file modifications: `git -C _workers/worker-N status --short`
2. If files are modified → worker IS making progress silently. Wait 3 more minutes.
3. If NO files modified → send ONE status check message. Wait 2 more minutes.
4. If still no response and no file changes → the worker is stuck. Do NOT delete its worktree. Instead, fall back to single-agent mode for that worker's tasks (implement them yourself in the main workspace). Leave the worktree intact until the post-worker cleanup phase.

### Lead Non-Interference Rule

While workers are actively working (uncommitted changes visible in their worktree):
- Do NOT read or debug their source files from the main workspace
- Do NOT attempt to fix their tests or implementation
- DO check their worktree status to confirm activity: `git -C _workers/worker-N status --short`
- Only intervene if: (a) worker explicitly reports a blocker via message, OR (b) worker is idle with no file changes for 5+ minutes after the grace period

If the user reports a worker is struggling, check worktree status first. If changes exist, the worker is making progress — report this to the user and wait. Workers often hit temporary test failures and self-resolve within a few turns.

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

### 1. Pre-Shutdown Verification

Before sending any shutdown requests, verify each worker's state:
```bash
# For each worker N:
git -C _workers/worker-N log --oneline -1
git -C _workers/worker-N status --short
```

If a worker has uncommitted changes (files listed by `status --short`), salvage them:
```bash
git -C _workers/worker-N add -A -- ':!node_modules' ':!.env' ':!.env.local'
git -C _workers/worker-N commit -m "lead: salvage worker-N uncommitted progress"
```

### 2. Shutdown Workers

1. Send shutdown requests to all workers using `SendMessage` with `type: "shutdown_request"`
2. Wait for shutdown confirmations — timeout after 2 minutes per worker
3. Mark all work unit tasks as completed via `TaskUpdate`

**CRITICAL: Never delete worktrees while workers are alive.** Worktree deletion is IRREVERSIBLE and destroys all uncommitted worker progress. The sequence MUST be: shutdown all workers → verify all confirmed → THEN delete worktrees in the Cleanup phase.

### 3. Merge Worker Branches

Merge worker branches into the feature branch **one at a time, foundation-first**.

**Determine merge order:**
- Workers handling lower-level code merge first: types/schemas → services → API routes → UI
- If workers are at the same layer, merge by worker number
- The first merge is always a fast-forward (feature branch hasn't moved)

**For each worker branch (in order):**
```bash
git merge <FEATURE_BRANCH>-worker-N
```

**After each merge (starting from the second):**
```bash
npm run typecheck
```
If type errors → fix them before merging the next worker. This catches integration issues early before they compound.

**If a merge has conflicts:**
1. Review the conflicting files — understand both workers' intent from the plan
2. Resolve conflicts, keeping correct logic from both sides
3. `git add` resolved files, then `git commit` (git's auto-generated merge message is fine)
4. Run `npm run typecheck` before continuing to the next merge

**If `git merge` fails entirely** (e.g., worktree artifacts like committed symlinks):
1. Fall back to cherry-pick: `git cherry-pick <FEATURE_BRANCH>-worker-N --no-commit`
2. Unstage any worktree artifacts: `git reset HEAD node_modules 2>/dev/null`
3. Commit: `git commit -m "fix: [worker summary]"`
4. Verify `node_modules` is still a real directory (not a symlink): `ls -ld node_modules | head -1`
5. If it became a symlink: `rm -f node_modules && npm install`

### 4. Run Lead-Reserved Tasks (Generated Files)

If any tasks were reserved for the lead during partitioning:
1. Run the CLI command (e.g., `npx drizzle-kit generate`)
2. Verify output files are correct
3. If the generator produces no changes, investigate — workers may have missed a schema change

### 5. Install New Dependencies (if needed)

If the plan required new npm packages that workers couldn't install:
```bash
npm install <package-name>
```

### 6. Run Post-Merge Integration Tests

Run the full unit/integration test suite immediately after all merges:
```bash
npm test
```

**Why here (before bug-hunter):** Workers only run targeted tests (`npx vitest run "pattern"`) in their worktrees. Cross-domain integration bugs (missing events on certain paths, stale closures at boundaries, type mismatches between worker outputs) only surface when all code is merged and the full suite runs. Catching these before bug-hunter reduces the bug-hunter's job to logic issues that tests don't cover.

If failures → fix directly, then re-run until all tests pass.

### 7. Run E2E Tests (if workers wrote E2E specs)

Run the `verifier` agent in E2E mode:
```
Task tool with subagent_type "verifier" and prompt "e2e"
```
If E2E tests fail → fix the specs directly, then re-run.

### 8. Run Full Verification

**Bug hunter:**
```
Task tool with subagent_type "bug-hunter"
```

Fix ALL real bugs — pre-existing or new. Only skip verifiable false positives.

**Verifier (tests + lint + build):**
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
git branch -d <FEATURE_BRANCH>-worker-1
git branch -d <FEATURE_BRANCH>-worker-2
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
| Worker stops without reporting | Check worktree: `git -C _workers/worker-N status --short`. If changes exist, salvage and commit from lead. If empty, implement tasks in single-agent mode. Do NOT delete the worktree until shutdown is confirmed. |
| Worker reports workspace missing | Worktree was deleted prematurely. Shut down the worker. Implement its tasks in single-agent mode. |
| Worker's Bash environment breaks | Known bug (#17321) — worker used Bash for file ops. Shut down the worker. Implement its tasks in single-agent mode. |
| Small batch (≤3 tasks, ≤6 files) | Skip workers entirely — use single-agent mode from the start |
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
- **Foundation-first merge order** — Merge lower-level workers first (types → services → routes → UI). Typecheck gate (`npm run typecheck`) after each merge.
- **Workers commit, don't push** — Workers `git add -A && git commit` in their worktree. Lead merges locally via the shared git object database.
- **Never delete worktrees while workers are alive** — Worktree deletion is irreversible. Always shutdown workers first, then verify shutdown, then delete worktrees.
- **Respect the 5-minute grace period** — Workers need multiple turns to start. Do not send status checks or take corrective action before 5 minutes have passed.
- **Small batches skip workers** — ≤3 tasks or ≤6 files total → single-agent mode. Worker overhead exceeds implementation time for small batches.
- **Always clean up worktrees** — Remove worktrees, prune metadata, delete worker branches after merge
- **No co-author attribution** — Commit messages must NOT include `Co-Authored-By` tags
- **Never stage sensitive files** — Skip `.env*`, `*.key`, `*.pem`, `credentials*`, `secrets*`
- **Log migrations in MIGRATIONS.md** — Workers report migration-relevant changes to lead; lead appends to MIGRATIONS.md
